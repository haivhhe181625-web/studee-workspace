# Design: Lịch hôm nay + đẩy lùi lịch khi trễ (M4-4)

- **Ngày:** 2026-08-06
- **Tác giả:** Tech Lead (AI draft)
- **Trạng thái:** Ngã rẽ nghiệp vụ đã chốt (spec §2); chờ Technical Review OQ-1/OQ-2 (spec §8).
- **Spec:** `specs/study-schedule-catchup/spec.md`
- **Module:** `exe-api/services/api/src/modules/adaptive/schedule/` (chính) + `course-content/` (đọc hoàn-thành).

## 1. Nguyên tắc dẫn đường

1. **Không nhân đôi nguồn sự thật.** Hoàn-thành sống ở `CourseEnrollment` — M4-4 **đọc**. Lịch (`LessonSchedule`) là projection.
2. **Một thuật toán.** "Đẩy lùi" **không** phải thuật toán mới — nó là `generateSchedule(bài-chưa-done, prefs, bands, asOfDate=hôm-nay)`, **cường độ thường** (không ×1.5).
3. **Thủ công, không tự động.** `today` chỉ đọc + đánh dấu ngày lỡ; đẩy lùi chỉ chạy khi người học bấm.
4. **Không thêm field.** Cảnh báo hạn tái dùng `WarningSchema` (`{overdue, suggestions}`) sẵn có. Không `pendingAdjustment`/`droppedLessonKeys`/khóa idempotent (postpone tất định → bấm lại = y hệt).

## 2. Ba lớp

```
GET /schedule/today?tz=            POST /schedule/postpone
   │ (tz→todayLocal, biên)            │ (asOfDate=todayLocal, biên)
   ▼                                  ▼
[V] buildTodayView(sched, done, todayLocal)   ── THUẦN     [P] postponeSchedule(userId, todayLocal) ── I/O mỏng:
      → { date, items(done?), progress,                         1. đọc LessonSchedule + CourseEnrollment
          missed, warning }                                     2. remaining = bài học CHƯA done (theo thứ tự lộ trình)
                                                                3. { days, warning } = generateSchedule(remaining, prefs, bands, todayLocal)
                                                                4. sched.days = days; sched.warning = deadlineWarn(days, deadline)
                                                                5. sched.asOfDate/generatedAt = todayLocal; save
```

- **[V]** thuần (test không cần DB). **[P]** orchestrator mỏng gọi generator thuần.
- Không guard idempotent: postpone tất định; bấm 2 lần → cùng `days` (AC-7).

## 3. Data model

**KHÔNG đổi `LessonSchedule`, KHÔNG đụng `course-content`/`CourseEnrollment`.** Tái dùng:
- `days[].items[]` đã có `kind` + `courseSlug/phaseKey/moduleKey/lessonKey` (join done).
- `warning: { overdue, suggestions[] }` (WarningSchema) cho cảnh báo hạn.
- `asOfDate`/`generatedAt`/`summary` cập nhật khi postpone.

## 4. "done" = join lịch ↔ CourseEnrollment (READ-ONLY)

```
isDone(item, doneMaps, passedBoundaries):
  if item.kind === 'review':     return null           // chỉ hiển thị, không tính done (A1)
  if item.kind === 'checkpoint': return passedBoundaries.has(item.boundaryKey)
  key = `${item.phaseKey}/${item.moduleKey}/${item.lessonKey}`
  s = doneMaps.get(item.courseSlug)?.get(key)?.status
  return s === 'completed' || s === 'mastered'
```

- `doneMaps`: theo `courseSlug` → Map path→{status} (đọc `CourseEnrollment` theo `userId`).
- `passedBoundaries`: từ `getPassedBoundaries()` sẵn có (course-content.progress.service).

## 5. `buildTodayView` (IS-1/IS-5, thuần)

```
buildTodayView(sched, doneMaps, passedBoundaries, lastActiveDate, todayLocal):
  # IS-5 — ngủ đông: tính lúc đọc, không lưu, không ghi CourseEnrollment
  if daysBetween(lastActiveDate, todayLocal) >= DORMANT_INACTIVE_DAYS:
     return { status:'dormant', date: todayLocal, resume:true,
              message:'Đã tạm dừng theo dõi — quay lại bất cứ lúc nào' }   # KHÔNG trả danh sách đỏ

  today = sched.days.find(d => d.dateLocal === todayLocal)
  items = (today?.items ?? []).map(i => ({ ...i, done: isDone(i, doneMaps, passedBoundaries) }))
  counted = items.filter(i => i.done !== null)                 // bỏ ngày ôn khỏi tiến độ
  progress = { doneCount: counted.filter(i=>i.done).length,
               totalCount: counted.length,
               remainingMin: sum(counted.filter(i=>!i.done).durationMin) }
  # ngày lỡ = ngày < hôm-nay còn item HỌC chưa done (review/checkpoint không tính — AC-4)
  missedDays = sched.days.filter(d => d.dateLocal < todayLocal)
                 .map(d => ({ dateLocal:d.dateLocal, items: d.items.filter(i => isDone(i,…) === false) }))
                 .filter(d => d.items.length)
  missed = missedDays.length ? { count: sum(items), oldestDate: missedDays[0].dateLocal, days: missedDays } : null
  return { status:'active', date: todayLocal, items, progress, missed, warning: sched.warning ?? null }
```

- **`DORMANT_INACTIVE_DAYS = 14`** (hằng số, tinh chỉnh sau). `lastActiveDate` từ `User.profile` (đã có; bump bởi `touchStreak` khi hoàn thành bài). Về-active tự nhiên: postpone/hoàn-thành cập nhật `lastActiveDate` → lần đọc kế `daysBetween < ngưỡng`.
- **Không lưu cờ dormant, không ghi `CourseEnrollment`** — M4-5 cron tính cùng công thức để loại dormant khỏi vòng quét (nơi tốn chi phí thật).

## 6. `postponeSchedule` (IS-3)

```
postponeSchedule(userId, todayLocal):
  sched = LessonSchedule.findOne({ userId }); if !sched: return null
  doneMaps = loadDone(userId, sched)
  allItems = resolveLessonList(userId)                          # thứ tự lộ trình gốc (OQ-2)
  remaining = allItems.filter(l => !isDoneLesson(l, doneMaps))  # bỏ bài đã done; KHÔNG bỏ gì khác (no skip — AC-9)
  pref = readPrefs(userId)                                       # cường độ THƯỜNG (không ×1.5)
  { days, warning, summary } = generateSchedule(remaining, pref, bands, todayLocal)   # THUẦN
  # giữ lịch sử: ngày đã qua chỉ giữ item đã done; phần từ hôm nay = sinh lại
  pastKept = sched.days.filter(d => d.dateLocal < todayLocal)
               .map(d => ({ ...d, items: d.items.filter(i => isDoneLesson(i, doneMaps)) }))
               .filter(d => d.items.length)
  sched.days = pastKept.concat(days)                           # lịch sử (done) + kế hoạch mới
  sched.summary = summary
  sched.warning = deadlineWarn(summary.estimatedFinishDate, pref.deadline)  # IS-4
  sched.asOfDate = todayLocal; sched.generatedAt = todayLocal
  save(sched); return sched
```

- **Giữ lịch sử (AC-10):** ngày `< hôm-nay` giữ lại nhưng chỉ item đã done (lịch sử hoàn thành); bài chưa-done của ngày cũ đã nằm trong `remaining` → dời sang phần mới. Không trùng lặp bài chưa-done ở quá khứ.
- **Nghỉ 7 ngày (AC-6):** 7 ngày qua chưa done → toàn bộ nằm trong `remaining` → generator xếp lại từ hôm nay, cường độ giữ nguyên → ngày-xong lùi ~7 ngày.
- **Không nhồi (AC-5):** truyền `pref.minutesPerDay = focusDurationMin` **nguyên bản** → cap ngày như thường. Bài quá khổ vẫn độc chiếm ngày (đã có).
- **Không skip (AC-9):** `remaining` chỉ loại bài **đã done**; tổng bài chưa-done được giữ trọn.
- **Tất định (AC-7):** mọi input cố định (remaining theo thứ tự + prefs + todayLocal); không `now()`.

## 7. `deadlineWarn` (IS-4, thuần)

```
deadlineWarn(estimatedFinishDate, deadline):
  if !deadline or estimatedFinishDate <= deadline: return { overdue:false, suggestions:[] }
  return { overdue:true, suggestions:[
    `Giãn hạn tới ${estimatedFinishDate}`,
    `Tăng lên ~${neededMinutesPerDay} phút/ngày để kịp ${deadline}` ] }   # neededMinutesPerDay xấp xỉ từ tổng phút còn lại / số ngày học tới deadline
```

- Chỉ **cảnh báo**, không chặn. Người học tự sửa `deadline`/`focusDurationMin` qua `PUT /schedule/prefs` sẵn có → lần postpone/generate kế phản ánh.

## 8. API contract

```
GET  /api/adaptive/schedule/today            (MỚI)
  query: { tz?: string }        // IANA (vd 'Asia/Saigon'); vắng → fallback 'Asia/Saigon'
  200 (active):   { status:'active', date, items:[{ ...item, done: boolean|null }],
                    progress: { doneCount, totalCount, remainingMin },
                    missed: { count, oldestDate, days:[{ dateLocal, items[] }] } | null,
                    warning: { overdue, suggestions[] } | null }
  200 (dormant):  { status:'dormant', date, resume:true, message }   // bất hoạt ≥ ngưỡng — không trả danh sách đỏ

POST /api/adaptive/schedule/postpone         (MỚI)
  body: {}                       // asOfDate = server today-local ở controller (dùng tz body/query nếu owner muốn; mặc định tz mặc định)
  200:  { schedule, warning }
  404:  chưa có lịch (SCHEDULE_NOT_FOUND)

(hoàn-thành: KHÔNG endpoint mới — dùng POST /course-content/:slug/lessons/progress)
```

- 2 route đặt trong `adaptive.routes.js` sau `schedule/*`, gate `verifyToken → withTenant`.

## 9. Kiểm thử (map AC → test)

| AC | Loại | Test |
|---|---|---|
| AC-1/2 | unit | `buildTodayView`: join done, ngày ôn không tính, progress/remainingMin đúng, ngày rỗng |
| AC-3 | integration | tick qua recordLesson → `today` phản ánh done |
| AC-4 | unit | `missed` chỉ gồm ngày quá hạn có item học chưa done; review/checkpoint không tính |
| AC-5 | unit | postpone: không ngày nào > focusDurationMin (trừ oversized); missed sạch sau đó |
| AC-6 | unit | nghỉ 7 ngày → dời từ hôm nay, cường độ giữ, lịch hợp lệ |
| AC-7 | unit | postpone 2× → `days` y hệt (tất định) |
| AC-8 | unit | deadline vượt → warning.overdue + suggestions; không deadline/kịp → không overdue |
| AC-9 | unit | tổng bài trước/sau postpone bằng nhau |
| AC-10 | unit | postpone giữ ngày quá khứ chỉ item done; bài chưa-done không trùng ở quá khứ |
| AC-11 | unit | lastActiveDate ≥ ngưỡng → status:dormant (không missed); < ngưỡng → active; không ghi CourseEnrollment |

## 10. Technical Review

- [ ] **OQ-1:** `remaining` = `resolveLessonList` (thứ tự lộ trình) lọc bỏ done — đúng thứ tự học?
- [ ] **OQ-2:** ngưỡng dormant 14 ngày; tín hiệu `lastActiveDate` vs `max(lastActiveDate,lastLoginAt)` — chốt.
- [ ] Checkpoint "done" = `getPassedBoundaries` (chỉ đọc) — xác nhận.

## 11. Không tạo (YAGNI)

- Không field mới trên `LessonSchedule`; không store done mới; không cờ dormant lưu sẵn; không ghi `CourseEnrollment` từ adaptive; không endpoint tick/adjust; không cron; không field timezone; không thuật toán đẩy-item riêng; không trần 150%; không logic skip; không auto-đuổi.
