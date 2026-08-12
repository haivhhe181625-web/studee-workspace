# Design: Thông báo nhắc học (M4-5)

- **Ngày:** 2026-08-06
- **Tác giả:** Tech Lead (AI draft)
- **Trạng thái:** CHỜ Technical Review (OQ-1/2/3 spec §8) + owner xác nhận Prerequisites (spec §7).
- **Spec:** `specs/study-schedule-reminders/spec.md`
- **Module:** `exe-api/services/api/src/queues/` (cron mới) + `modules/adaptive/schedule/` (tái dùng M4-4) + `adapters/` (fcm/email) + `modules/user/`.

## 1. Nguyên tắc dẫn đường

1. **Tái dùng M4-4, không dựng lại.** "Hôm nay có gì / xong chưa / có dormant không" đều lấy từ `buildTodayView` (M4-4). Reminder chỉ là **tầng gửi** đặt trên đó.
2. **Không store nhiệm vụ.** Không `DailyQuest`. Store thêm là `NotificationLog` **dùng chung mọi feature** (owner chốt Hybrid 2026-08-07) — để **dedup (idempotent)** + **observability** + sau này in-app feed; không phải nguồn nhiệm vụ. M4-5 chỉ ghi `type:'study-reminder'`.
3. **Cron mỏng, quyết định thuần.** Cron chỉ quét + gọi hàm thuần `decideReminder(view, slot, nowLocalHour)` → trả `{ send, title, body } | null`. Logic "gửi hay không" test được không cần DB/queue.
4. **Idempotent theo (userId, ngày-local, slot).** Cron chạy mỗi giờ; dedup bằng `StudyReminderLog` unique index.

## 2. Kiến trúc

```
[cron mỗi giờ 0 * * * *]  reminder.worker
   │  for each active learner (có studyPlan, notifEnabled, có LessonSchedule)
   ▼
[A] tzLocal = todayLocalDate(plan.timezone, now); hourLocal = localHour(plan.timezone, now)
[B] slot = hourLocal===8 ? 'morning' : hourLocal===20 ? 'evening' : null    (ngoài 2 mốc → bỏ ngay)
   │  slot==null → skip
   ▼
[C] view = buildTodayView(sched, doneMaps, passedBoundaries, lastActiveDate, tzLocal)   ── tái dùng M4-4
   │  view.status==='dormant' → skip (loại ngủ đông)
   ▼
[D] decideReminder(view, slot) ── THUẦN
   │  morning: view.progress.totalCount>0 → {title,'Hôm nay bạn có N bài'} ; else null
   │  evening: doneCount<totalCount → {title,'Còn X bài chưa xong'} ; else null (đã xong → bỏ)
   │  null → skip
   ▼
[E] guard: quietHours(hourLocal) | notificationPrefs.study===false → skip
[F] dedup: NotificationLog.exists({userId, type:'study-reminder', dedupKey:`${dateLocal}:${slot}`}) → skip (idempotent)
   ▼
[G] send:  fcmTokens?.length ? addPushUserJob(...) : email.sendStudyReminder(...)   (fallback-only)
[H] NotificationLog.create({userId, type:'study-reminder', dedupKey, channel, sentAt})   (shared log + dedup marker)
```

- **[D] decideReminder thuần** — test không cần DB. **[A][B][E] biên thời gian** ở cron. **[C] tái dùng M4-4.** **[F][H] dedup/log.**
- **Thứ tự [G] gửi TRƯỚC [H] log (chủ đích):** unique index đảm bảo ≤1 *log*, không phải ≤1 *gửi*. Nếu worker crash sau khi push đã bắn nhưng trước khi ghi log, BullMQ retry job → learner có thể nhận tin lần 2 (cửa sổ ~1ms/user, payload nhẹ). Chọn gửi-trước-log = ưu tiên **giao được tin** hơn dedup tuyệt đối — hợp lý cho nhắc học low-stakes. Nếu sau này cần no-double-send tuyệt đối: đổi sang claim-then-send (create log trước, E11000 → skip, rồi mới gửi).

## 3. Data model

### 3.1 Prefs — `timezone` (studyPlan) + `notificationPrefs` (profile, dùng chung)
```js
// StudyPlanSchema (user.model.js): THÊM 1 field (tz là thuộc tính lịch học)
timezone: { type: String, default: null },   // IANA, vd 'Asia/Saigon'; null → fallback ở biên
// timeSlots (đã có) có thể dùng suy giờ slot ở v2 (OQ-1); v1 dùng mốc cố định 8/20.

// profile (sibling studyPlan): notificationPrefs DÙNG CHUNG — mở rộng cho feature khác về sau
notificationPrefs: { study: { type: Boolean, default: true } },  // tắt study → không gửi nhắc học
// v.d tương lai: notificationPrefs.marketing, .streak … không phải migrate lại.
```
Joi (`study-schedule.schema.js` `saveStudyPlan`): thêm `timezone` (string optional), `notificationPrefs` (`{ study: bool }` optional). Reminder đọc `notificationPrefs?.study !== false`.

### 3.2 `NotificationLog` (mới, DÙNG CHUNG) — dedup + observability
```js
{ userId: ObjectId(ref User), type: String /*'study-reminder'|…*/, dedupKey: String,
  channel: 'push'|'email', sentAt: Date }
// study-reminder: dedupKey = `${dateLocal}:${slot}` (vd '2026-08-07:morning')
// unique index (userId, type, dedupKey) → idempotent: chèn trùng ném E11000 → coi như đã gửi.
// TTL index trên sentAt (vd 30 ngày) để không phình.  userId indexed → truy vấn feed/observability theo user.
```
> **OQ-3 → chốt:** collection (auditable, NFR-003) thay vì Redis TTL. Owner chốt Hybrid: tổng quát `type/dedupKey` để feature khác (streak, marketing) tái dùng cùng collection.

### 3.3 KHÔNG tạo `DailyQuest` (deviation SRS §10.1) — tiến độ hôm nay suy từ M4-4.

## 4. `decideReminder` (IS-3, thuần)
```
decideReminder(view, slot):
  if view.status !== 'active': return null           # dormant/không lịch → không nhắc
  const { doneCount, totalCount } = view.progress
  if slot === 'morning':
     return totalCount > 0 ? { slot, title:'Nhắc học hôm nay', body:`Hôm nay bạn có ${totalCount} bài. Hoàn thành để giữ nhịp!` } : null
  if slot === 'evening':
     const remaining = totalCount - doneCount
     return remaining > 0 ? { slot, title:'Chưa xong hôm nay', body:`Bạn còn ${remaining} bài chưa hoàn thành hôm nay.` } : null
  return null
```
- `view.progress` đã loại ngày ôn (M4-4). `totalCount`/`doneCount` là bài học hôm nay.
- Thuần: chỉ đọc `view`. Không `now()`.

## 5. Cron worker (IS-2)
- `queues/reminder.queue.js` + `reminder.worker.js` theo pattern `billing.*`: repeatable `{ pattern:'0 * * * *', jobId:'schedule:reminders:hourly' }` (upsert idempotent khi boot).
- Worker quét: `User.find({ 'profile.notificationPrefs.study': { $ne:false } })` — hoặc quét user có `LessonSchedule` rồi lọc. Với mỗi user: [A]→[H].
- **Loại dormant**: `buildTodayView` trả `status:'dormant'` khi ≥14 ngày → skip ([C]) → **không gửi + không quét sâu**. (Đây là chỗ M4-4 dormant tiết kiệm chi phí thật.)
- **Quiet hours**: mốc gửi 8/20 vốn nằm ngoài 22–7, nhưng vẫn kẹp guard `if hourLocal<7 || hourLocal>=22: skip` (an toàn nếu OQ-1 đổi sang timeSlots).
- Timezone: `localHour(tz, now)` + `todayLocalDate(tz, now)` (mở rộng `local-date.js` thêm `localHour`).

## 6. Kênh gửi (IS-4, fallback-only, đa thiết bị)
```
if user.fcmTokens?.length:                                   # có token ở BẤT KỲ nền tảng (web/android/ios)
   addPushUserJob(userId, {title, body}, { type:'study-reminder', slot, route:'/schedule' })  # multicast MỌI thiết bị = 1 tin
else:
   await email.sendStudyReminder({ to:user.email, name:user.name, title, body })              # fallback khi 0 token
channel = tokens ? 'push' : 'email'
```
- **Deep-link:** `data.route='/schedule'` để client mở màn "Hôm nay" khi chạm. Web: nếu muốn click-through chuẩn, **mở rộng `fcm.adapter` thêm `webpush.fcmOptions.link`** (hiện adapter chỉ gửi `{notification,data}`); mobile route bằng `data`. → 1 mục trong Task 4/Technical Review.
- **1 tin = 1 lần multicast** tới mọi `fcmTokens` (web+android+ios) → dedup theo `(userId,dateLocal,slot)` giữ ≤2/ngày/người.
- `email.adapter`: THÊM method `sendStudyReminder({to,name,title,body})` (Resend, mẫu `sendRenewalReminder`).
- **Không** gửi cả 2 (Email chỉ khi 0 token) — đúng FR-010.
- **iOS:** BE không đổi (FCM lo); delivery iOS phụ thuộc APNs key trong Firebase project (Prerequisite).

## 7. API / prefs
- Không endpoint mới. Dùng `PUT /adaptive/schedule/prefs` sẵn có, nhưng cần chỉnh nhẹ cho prefs dùng-chung:
  - **`timezone`** nằm trong `studyPlan` → merge tự động qua path `$set 'profile.studyPlan'` hiện có (chỉ thêm field vào Joi `studyPlan`).
  - **`notificationPrefs`** ở cấp profile (sibling studyPlan) → controller `saveStudyPrefs` thêm **`$set 'profile.notificationPrefs'`** (merge, không ghi đè study bằng undefined) khi body có `notificationPrefs`.
  - Joi `saveStudyPlan`: **nới `studyPlan` từ `.required()` → optional** + thêm top-level `notificationPrefs: {study: bool}` optional + `.or('studyPlan','notificationPrefs')` (ít nhất 1). Lý do: toggle "tắt nhắc" không phải gửi lại toàn plan. `saveStudyPlan` chỉ dùng bởi endpoint này (routes:53) nên nới không ảnh hưởng generate flow.
- (Tùy chọn) `GET /adaptive/schedule/prefs` trả thêm `notificationPrefs` để FE prefill.

## 8. Kiểm thử (map AC → test)
| AC | Loại | Test |
|---|---|---|
| AC-1/2 | unit | `decideReminder`: morning có/không bài; evening còn/hết bài |
| AC-3 | unit | tối đa 2 slot/ngày |
| AC-4 | integration | dedup: chèn trùng (userId,type,dedupKey) → E11000/skip, không gửi lần 2 |
| AC-5 | unit | quietHours(23),(6) → skip; (8),(20) → cho phép |
| AC-6 | integration | notificationPrefs.study=false → không gửi |
| AC-7 | integration | có token → push (mock addPushUserJob); không token → email (mock adapter) |
| AC-8 | unit | view.status='dormant' → decideReminder=null |
| AC-9 | unit | localHour/todayLocalDate theo tz (mở rộng local-date test) |
| AC-10 | integration | mỗi lần gửi tạo NotificationLog đúng shape (type='study-reminder', dedupKey) |

## 9. Technical Review (OQ)
- [x] **OQ-1 → chốt (owner 2026-08-07):** slot **cố định 8:00 / 20:00**.
- [x] **OQ-2 → chốt:** `timezone` thiếu → **fallback `Asia/Saigon`**.
- [x] **OQ-3 → chốt:** dedup = **collection `NotificationLog`** (auditable, dùng chung — owner chốt Hybrid 2026-08-07).
- [ ] **Deep-link web:** mở rộng `fcm.adapter` thêm `webpush.fcmOptions.link='/schedule'` để chạm-tin mở đúng màn trên web (mobile route bằng `data`). Xác nhận khi code Task 4.
- [ ] Cron quét toàn user mỗi giờ — lọc `notifEnabled≠false` + có `LessonSchedule`; dormant tự loại ở [C]. Cần index thêm không?
- [ ] **Cross-repo (web+mobile):** client cập nhật `studyPlan.timezone` khi mở app/đăng nhập (BE chỉ đọc) — ngoài phạm vi BE M4-5, ghi backlog FE/mobile.

## 10. Không tạo (YAGNI — ranh giới Hybrid)
- Không `DailyQuest`; không in-app store; không user-set reminderTimes; không SMS/Zalo; không endpoint nhắc riêng; không đổi lõi M4-4.
- **Không** generic reminder-engine / notification-center API / in-app feed ngay bây giờ — chưa có consumer thứ 2 (rule of three). Chỉ tổng quát 2 seam rẻ: `NotificationLog{type,dedupKey}` + `notificationPrefs.study`. Cron/decideReminder vẫn riêng study-schedule.
