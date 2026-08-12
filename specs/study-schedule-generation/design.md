# Design: Sinh lịch học từ Lộ trình

- **Ngày:** 2026-08-04 · **Cập nhật:** 2026-08-05 (slice v2)
- **Tác giả:** Tech Lead (AI draft)
- **Trạng thái:** v1 đã ship; **v2 CHỜ Technical Review** (T5–T9 dưới)
- **Spec:** `specs/study-schedule-generation/spec.md`
- **Module:** `exe-api/services/api/src/modules/adaptive/` (chính) + `course-content/`, `user/`

## 1. Tổng quan kiến trúc

3 lớp tách bạch, lõi (2) **thuần**. v2 thêm 2 hàm thuần phụ (suggestion, projection) + mở rộng lõi (ngày ôn, chọn ngày).

```
POST /schedule/generate                         GET /schedule/suggest
   │ (controller: prefs, band, asOfDate)            │ (controller: tổng phút khóa, deadline)
   ▼                                                ▼
[1] resolveLessonList(userId)                    [S] suggestPlans(totalMin, asOfDate, deadline?)  ── THUẦN
      → LessonItem[] (kèm exercises[])                → { presets[] } | { requiredWeeklyMin, tooTight }
   │
   ▼
[2] generateSchedule(lessons, pref, bands, asOfDate)   ── THUẦN
      → { days: DayPlan[] (gồm ngày ôn), warning?, summary }
   │
   ▼
[3] persist(userId, schedule)                    ── I/O
```

- **[2] là trái tim** — deterministic. v2: [2] còn chèn **ngày ôn** + tính **summary**.
- **[S] độc lập**, thuần, chỉ cần tổng phút khóa (đọc từ resolver) — không persist.

## 2. Data model

### 2.1 `LessonSchema` (course-content.model.js) — v1, giữ nguyên

`estimatedDurationMin: { type: Number, min: 1, default: null }` (đã có). v2 **đọc thêm `exercises[]`** đã có sẵn (`[{ type, refId, order, label }]`) — **không sửa schema course-content**.

### 2.2 `StudyPlanSchema` (user.model.js) — v2 THÊM field

Thêm 2 field (giữ nguyên field cũ cho service coexist — **D8**):

```js
// v2 — người dùng chọn thứ học/ôn cụ thể (ISO dow: 1=Mon … 7=Sun).
learningDows: { type: [Number], default: null }, // các thứ HỌC; null → generator cũ suy từ daysPerWeek (back-compat)
reviewDow:    { type: Number, min: 1, max: 7, default: null }, // thứ ÔN, phải > max(learningDows) trong tuần
// GIỮ NGUYÊN (service study-schedule.service.js cũ còn dùng — KHÔNG xóa):
// daysPerWeek, weekendStudy, focusDurationMin, timeSlots, deadline, commitment
```

- `daysPerWeek` trở nên **suy được** = `learningDows.length` (giữ field để service cũ chạy; generator mới không đọc).
- **Không migration cứng** (default null); user chưa khai `learningDows` → controller trả `needsPrefs`.

### 2.3 Joi `study-schedule.schema.js` — v2 SỬA `savePrefs` + THÊM `suggest`

```js
// savePrefs.studyPlan — THÊM learningDows/reviewDow + ĐỔI sàn/trần (IS-8, D9)
learningDows: Joi.array().items(Joi.number().integer().min(1).max(7)).unique()
                 .min(2).max(7).required(),          // 2..7 thứ học, không trùng
reviewDow:    Joi.number().integer().min(1).max(7).required(),
focusDurationMin: Joi.number().integer().min(60).max(240).required(), // was 5..120
// ràng buộc reviewDow > max(learningDows): custom .custom() hoặc Joi.ref — AC-13
// daysPerWeek: giữ cho service cũ nhưng cho optional (suy từ learningDows.length nếu vắng)

// generate — giữ nguyên (asOfDate isoDate optional)
// suggest (mới): { deadline?: Joi.string().allow(null,'') }  // asOfDate optional
```

> **Lưu ý coexist:** `savePrefs` dùng CHUNG cho cả lịch LLM cũ. Đổi sàn/trần `focusDurationMin` (60–240) + thêm `learningDows` required áp cho **mọi** ghi prefs. Chấp nhận (1 studyPlan/user, thống nhất ràng buộc). User cũ có `focusDurationMin<60`/thiếu `learningDows` → lần sửa prefs kế phải khai lại. Ghi vào changelog.

### 2.4 `LessonSchedule` (adaptive/lesson-schedule.model.js) — v2 SỬA `days[]`

```js
days: [{
  dateLocal: String,       // 'YYYY-MM-DD'
  loadMin: Number,
  oversized: Boolean,      // chỉ cho ngày HỌC
  isReview: Boolean,       // v2 — true = ngày ôn tổng hợp (default false)
  items: [{ lessonKey|exerciseRef: String, title: String, type: String, durationMin: Number }]
}]
// v2 THÊM: summary { totalLearningSessions, totalReviewSessions, totalLessonMin, estimatedFinishDate } (embed hoặc tính khi đọc)
```

- Ngày ôn: `isReview:true`, `oversized:false`, `items` là **exercises tổng hợp** (mỗi item = 1 exercise ref với `type` ∈ {quiz,flashcard,ipa,talk}, `durationMin` = heuristic exercise).

### 2.5 Input types (DTO lớp thuần)

```
LessonItem = { lessonKey, title, type, durationMin, skill, phaseBand,
               exercises: [{ type, refId }] }        // v2 THÊM exercises (cho ngày ôn)
Preference = { learningDows:[Number], reviewDow:Number, minutesPerDay, deadline }  // v2: bỏ daysPerWeek/weekendStudy khỏi DTO generator
SkillBands = { reading:'B1', ... } | null
```

## 3. Resolver nguồn bài (IS-2 v1 + v2 surface exercises)

Giữ logic v1 (StudentPath → CourseStructure fallback, thứ tự `phase.order→module.order→lesson.order`). **v2 thêm 1 field khi emit:**

```
emit LessonItem{ ...(v1 fields), exercises: lesson.exercises.map(e => ({ type: e.type, refId: e.refId })) }
```

- Chỉ lấy `{type, refId}` (đủ để tính tải + tham chiếu; label/order không cần cho scheduler).
- Bài không có exercises → `exercises: []` (tuần toàn bài như vậy → không có ngày ôn — AC-10).

## 4. Thuật toán thuần `generateSchedule` (v2)

Pipeline: **lọc band → chunking bài học vào `learningDows` → chèn ngày ôn ở `reviewDow` cuối mỗi tuần → cảnh báo + summary.**

```
generateSchedule(lessons, pref, bands, asOfDate):
  list = filterByBand(lessons, bands)               # v1, giữ nguyên
  cap  = pref.minutesPerDay
  buckets = chunk(list, cap)                         # v1: no-split + oversized → [{items, loadMin, oversized}]

  # v2 — gán ngày + chèn ngày ôn, nhóm theo TUẦN LỊCH (ISO, Hai→CN — OQ-6)
  days = []
  cursor = asOfDate
  weekLessons = []            # tích luỹ bài học trong tuần hiện tại (để tổng hợp exercises)
  bi = 0                      # con trỏ bucket
  while bi < buckets.length or weekLessons has pending review:
     dow = isoDow(cursor)
     if dow in pref.learningDows and bi < buckets.length:
        b = buckets[bi++]; days.push({ dateLocal:cursor, ...b, isReview:false })
        weekLessons.push(...b.items)
     if dow == pref.reviewDow:
        rev = buildReviewDay(weekLessons)            # tổng hợp exercises tuần
        if rev.items.length: days.push({ dateLocal:cursor, loadMin:rev.loadMin,
                                         oversized:false, isReview:true, items:rev.items })
        weekLessons = []                             # reset tuần
     cursor = cursor + 1 day
  # (trailing week: nếu hết bài giữa tuần, buildReviewDay chạy ở reviewDow kế nếu weekLessons còn exercises)

  warning = overdueWarning(days, pref.deadline)      # v1 (AC-8) — dựa ngày HỌC cuối
  summary = buildSummary(days, asOfDate)             # v2 (IS-10)
  return { days, warning, summary }

buildReviewDay(weekLessons):
  exs = weekLessons.flatMap(l => l.exercises)        # {type, refId}[]
  items = exs.map(e => ({ exerciseRef:e.refId, type:e.type, durationMin: exerciseDurationOf(e.type) }))
  return { items, loadMin: sum(items.durationMin) }  # KHÔNG cắt theo cap (AC-11)
```

- **`isoDow`**: chuyển 0=CN của `getUTCDay()` sang ISO 1..7 (CN→7). Tuần bắt đầu Hai.
- **Ràng buộc `reviewDow > max(learningDows)`** đảm bảo ôn luôn sau buổi học cuối trong tuần (validate ở Joi — AC-13).
- **Terminate**: khi `bi==buckets.length` và weekLessons đã được review (hoặc rỗng exercises). Guard vòng lặp bằng số ngày tối đa (vd `buckets.length*14 + 7`) để không kẹt nếu prefs lỗi.
- **Determinism (NFR-1):** mọi thứ từ `asOfDate` + `learningDows`/`reviewDow` (số cố định); không `Date.now()`.

**Heuristic exercise** (`exerciseDurationOf`, thuần, bảng đề xuất — OQ-7): `quiz` 8 · `flashcard` 5 · `ipa` 5 · `talk` 10; type lạ → 5.

## 5. Suggestion engine `suggestPlans` (IS-9, thuần)

```
suggestPlans(totalLessonMin, asOfDate, deadline?):
  PRESETS = {
    thong_tha: { learningDays:2, minutesPerDay:60 },
    can_bang:  { learningDays:4, minutesPerDay:60 },   # khuyến nghị
    cap_toc:   { learningDays:5, minutesPerDay:90 },
  }
  if deadline:
     weeks = weeksBetween(asOfDate, deadline)
     requiredWeeklyMin = ceil(totalLessonMin / weeks)          # chưa gồm ôn (xấp xỉ)
     healthyMaxWeekly = 5*90                                    # ~7.5h/tuần (OQ-8)
     return { requiredWeeklyMin, tooTight: requiredWeeklyMin > healthyMaxWeekly,
              fallbackPlan: tooTight ? planFor(healthyMaxWeekly) : null }
  else:
     return { presets: PRESETS.map(p => ({ ...p,
                estimatedFinishDate: projectFinish(totalLessonMin, p, asOfDate) })) }
```

- `projectFinish`: xấp xỉ `số tuần = totalLessonMin / (learningDays*minutesPerDay)` → cộng vào `asOfDate` (mỗi tuần +1 ngày ôn, không đổi tổng phút học). Thuần.
- **Không** cần lịch chi tiết để gợi ý → rẻ, gọi trước khi user khai.

## 6. Projection summary `buildSummary` (IS-10, thuần)

```
buildSummary(days, asOfDate):
  learning = days.filter(!isReview); review = days.filter(isReview)
  return { totalLearningSessions: learning.length,
           totalReviewSessions:  review.length,
           totalLessonMin:       sum(learning.loadMin),
           estimatedFinishDate:  days.at(-1)?.dateLocal ?? asOfDate }
```

Cảnh báo (gộp vào `warning`/`warnings[]`): quá hạn (v1) · có `oversized` · lịch quá dài (learning-only mốc tối thiểu) · tuần ôn nặng (`loadMin` review > 2×`minutesPerDay`).

## 7. API contract (v2)

```
POST /api/adaptive/schedule/generate      (v1, response mở rộng)
  body: { asOfDate?: 'YYYY-MM-DD' }
  200:  { schedule: LessonSchedule, warning?, summary }   # v2 thêm summary + days có isReview

GET  /api/adaptive/schedule/suggest        (v2, MỚI)
  query: { asOfDate?, deadline? }          # prefs KHÔNG bắt buộc (gợi ý trước khi khai)
  200:  { presets[] } | { requiredWeeklyMin, tooTight, fallbackPlan? }
  409:  { code: 'SCHEDULE_NO_SOURCE' }     # chưa có nguồn bài → không tính được tổng phút
```

- Prefs server-side (`User.profile.studyPlan`); generate cần `learningDows`/`reviewDow` → thiếu → `{ needsPrefs:true }`.
- `suggest` chỉ cần nguồn bài (tổng phút), **không** cần prefs.

## 8. Kiểm thử (map AC → test)

| AC | Loại | Test |
|---|---|---|
| AC-10 | unit | tuần có exercises → chèn ngày ôn `isReview`, items=exercises tuần, load=tổng; tuần rỗng → không chèn |
| AC-11 | unit | ngày ôn không bị cap, không oversized, không có lesson học |
| AC-12 | unit | learningDows=[2,4,6],reviewDow=7 → học đúng thứ, ôn CN; weekendStudy không ảnh hưởng |
| AC-13 | unit/api | reviewDow ≤ max(learningDows) → Joi reject 400 |
| AC-14 | api | learningDows<2/>7, focus<60/>240 → 400 |
| AC-15 | unit | không deadline → 3 preset kèm finishDate; có deadline → requiredWeeklyMin + tooTight |
| AC-16 | integration | generate trả summary + warnings; days có isReview |
| AC-17 | unit | suggest/projection cùng input → cùng output |

## 9. Quyết định kỹ thuật cần Technical Review

- [x] T1–T4 (v1) — chốt.
- [x] **T5 (D8/OQ-5):** ✅ chốt — giữ `weekendStudy`/`daysPerWeek` trong schema; generator mới ngừng dùng.
- [x] **T6 (OQ-6):** ✅ chốt — tuần = tuần lịch ISO (Hai→CN).
- [ ] **T7:** đổi sàn/trần `focusDurationMin` 60–240 trên `savePrefs` DÙNG CHUNG — ảnh hưởng cả lịch LLM cũ; chấp nhận? *(đề xuất: chấp nhận — 1 studyPlan/user; fixture test cũ cập nhật cho hợp lệ, không nới schema.)*
- [ ] **T8:** ràng buộc `reviewDow > max(learningDows)` đặt ở Joi custom hay ở service? (đề xuất Joi `.custom()`).
- [ ] **T9:** `summary` embed trong doc hay tính khi đọc? (đề xuất tính-khi-sinh, lưu trong doc để feature sau đọc rẻ).

## 10. Không tạo (YAGNI)

- Không `data-model.md`/`contracts/*.md` riêng — đủ gọn trong file này.
- Không engine ôn thông minh (chọn-lọc theo điểm/kỹ năng yếu) — v3, cần tầng tiến độ.
