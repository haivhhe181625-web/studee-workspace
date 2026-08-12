# Tasks: Thông báo nhắc học (M4-5)

> 4 quyết định nghiệp vụ đã CHỐT (owner 2026-08-06, spec §2). Còn OQ-1/2/3 (spec §8) cho Technical Review + Prerequisites môi trường (spec §7) cần owner xác nhận **trước khi test thật** (không chặn viết code lõi).

**Spec:** `spec.md` · **Design:** `design.md`
**Repo/branch:** `exe-api`, branch `feat/PRD-90-study-schedule-reminders` **base lên `develop`** (owner đã merge M4-4 + web vào develop 2026-08-07).
**OQ đã chốt (owner):** slot cố định 8/20 · thiếu tz → Asia/Saigon · dedup = collection **`NotificationLog` dùng chung** (`type`/`dedupKey`) · prefs = `profile.notificationPrefs.study` (Hybrid 2026-08-07).
**Lưu ý chạy lệnh:** mọi `npx jest` trong `services/api`. Test ở `src/__tests__/`; helper `__tests__/helpers/{db,app}.js`.

---

## Global Constraints

1. **NFR-1 — lõi thuần xác định.** `decideReminder`, `quietHours`, `localHour`: KHÔNG `Date.now()`/`new Date()` không-tham-số; `now`/`view`/`hourLocal` là tham số. Thời gian chỉ ở **worker/controller (biên)**.
2. **Tái dùng M4-4 (READ-ONLY qua nó):** `buildTodayView`/`loadDoneContext`/`todayLocalDate` — KHÔNG viết lại logic today/done/dormant. KHÔNG ghi `CourseEnrollment` từ M4-5.
3. **KHÔNG tạo `DailyQuest`** (deviation SRS §10.1). Store mới = `NotificationLog` **dùng chung** (`{userId,type,dedupKey,channel,sentAt}`) — dedup + observability; M4-5 ghi `type:'study-reminder'`. KHÔNG generic reminder-engine/feed API (YAGNI).
4. **Idempotent:** dedup theo unique `(userId, type, dedupKey)` với `dedupKey='${dateLocal}:${slot}'`; cron chạy lặp không gửi trùng.
5. **Email = fallback-only:** chỉ gửi Email khi KHÔNG có `fcmTokens`; có token → chỉ Web Push.
6. **Quiet hours 22h–7h + `profile.notificationPrefs.study===false` + dormant → không gửi.**
7. **Adapters:** push qua `notification.queue.addPushUserJob`; email qua `email.adapter` (thêm method, KHÔNG import Resend trực tiếp ngoài adapter).
8. **Cron:** BullMQ repeatable upsert theo `jobId` (idempotent boot), pattern `billing.worker`.
9. **Commit:** Conventional Commits; `git add <files>`; KHÔNG `Co-Authored-By`/AI attribution. Code & comment English; user-facing string (tiêu đề/nội dung tin, message) **tiếng Việt**.
10. **KHÔNG tham chiếu plan-taxonomy trong code/test** (`AC-n`/`IS-n`/`FR-n`/PRD) — mô tả hành vi.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/user/user.model.js` | `StudyPlanSchema.timezone` + `ProfileSchema.notificationPrefs.study` | Sửa |
| `src/modules/adaptive/study-schedule.schema.js` | Joi `saveStudyPlan`: `studyPlan`→optional, +`timezone`, +top-level `notificationPrefs`, `.or(...)` | Sửa |
| `src/modules/adaptive/adaptive.controller.js` | `saveStudyPrefs`: thêm `$set 'profile.notificationPrefs'` khi body có; `getStudyPrefs` trả thêm | Sửa |
| `src/modules/adaptive/schedule/local-date.js` | thêm `localHour(tz, now)` | Sửa |
| `src/modules/adaptive/schedule/reminder-decide.js` | `decideReminder(view, slot)`, `quietHours(h)` — thuần | Tạo |
| `src/modules/notification/notification-log.model.js` | `NotificationLog` **dùng chung** (unique `(userId,type,dedupKey)` + TTL) | Tạo |
| `src/adapters/email.adapter.js` | method `sendStudyReminder` | Sửa |
| `src/modules/adaptive/schedule/reminder.service.js` | `remindOne(userId, now)` orchestrator | Tạo |
| `src/queues/reminder.queue.js` + `reminder.worker.js` | cron repeatable + scan | Tạo |
| `src/__tests__/reminder-decide.test.js` | unit decide/quietHours | Tạo |
| `src/__tests__/reminder-service.test.js` | integration dedup/channel/notifEnabled/dormant/log | Tạo |
| `src/__tests__/local-date.test.js` | thêm test `localHour` | Sửa |
| `src/__tests__/study-plan-prefs.schema.test.js` | thêm timezone/notifEnabled | Sửa |

---

## Task 1: Prefs `timezone` + `notificationPrefs.study` + `localHour` (AC-9 nền)

**Files:** Modify `user.model.js`, `study-schedule.schema.js`, `adaptive.controller.js`, `local-date.js`; Test `study-plan-prefs.schema.test.js`, `local-date.test.js`

- [ ] **Step 1: Test thất bại** — schema `saveStudyPlan`:
  - chấp nhận `studyPlan` có `timezone:'Asia/Saigon'`;
  - chấp nhận body **CHỈ** `{ notificationPrefs:{study:false} }` (KHÔNG kèm studyPlan) — vì `studyPlan` nới thành optional;
  - reject body rỗng `{}` (`.or('studyPlan','notificationPrefs')` → cần ≥1);
  - `localHour('Asia/Saigon', new Date('2026-08-06T01:00:00Z'))===8` (UTC+7); tz lỗi → fallback Asia/Saigon.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement**
  - `user.model.js`: `StudyPlanSchema` thêm `timezone:{type:String,default:null}`. `ProfileSchema` thêm `notificationPrefs:{ study:{type:Boolean,default:true} }` — **dùng chung, mở rộng sau (`.marketing`/`.streak`)**.
  - `study-schedule.schema.js` `saveStudyPlan`: đổi `studyPlan` từ `.required()`→optional; thêm `timezone: Joi.string().optional()` vào object `studyPlan`; thêm top-level `notificationPrefs: Joi.object({ study: Joi.boolean() }).optional()`; thêm `.or('studyPlan','notificationPrefs')`.
  - `adaptive.controller.js` `saveStudyPrefs`: giữ nguyên merge `profile.studyPlan` (chỉ khi `req.body.studyPlan`); thêm: nếu `req.body.notificationPrefs` → `$set 'profile.notificationPrefs'` = merge `{...existing, ...body.notificationPrefs}` (đọc `profile.notificationPrefs` cùng lúc). Trả cả `studyPlan` + `notificationPrefs`. `getStudyPrefs`: select + trả thêm `notificationPrefs`.
  - `local-date.js`: `localHour(tz, now)` = giờ (0–23) tại tz (`Intl.DateTimeFormat('en-GB',{hour:'2-digit',hour12:false,timeZone:tz})`; tz lỗi → fallback Asia/Saigon).
- [ ] **Step 4:** `npx jest study-plan-prefs.schema local-date -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): add timezone and shared notification prefs to study prefs`

---

## Task 2: `decideReminder` + `quietHours` (AC-1,2,3,5,8)

**Files:** Create `reminder-decide.js`; Test `reminder-decide.test.js`

> Thuần: `decideReminder(view, slot)` nhận output `buildTodayView`; `quietHours(hourLocal)`.

- [ ] **Step 1: Test thất bại**
  - morning + `progress.totalCount=3` → `{slot:'morning', body chứa '3 bài'}`; totalCount=0 → `null`.
  - evening + done=1/total=3 → `body` chứa `2` (còn 2); done=3/total=3 → `null` (đã xong → bỏ).
  - `view.status='dormant'` → `null` (mọi slot).
  - `quietHours(23)===true`, `quietHours(6)===true`, `quietHours(8)===false`, `quietHours(20)===false`.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** (design §4) — export `decideReminder`, `quietHours`. String tiếng Việt.
- [ ] **Step 4:** `npx jest reminder-decide -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): pure reminder decision for morning/evening study nudges`

---

## Task 3: `NotificationLog` model dùng chung — dedup + TTL (AC-4,10 nền)

**Files:** Create `src/modules/notification/notification-log.model.js`; test gộp ở Task 4

- [ ] **Step 1: Implement** — schema `{userId(ObjectId ref User), type(String), dedupKey(String), channel(enum push/email), sentAt(Date)}`; **unique index `(userId, type, dedupKey)`**; TTL index `sentAt` (30 ngày). Export model. Generic — không nhúng khái niệm slot/date vào schema (caller build `dedupKey`).
- [ ] **Step 2: Verify** — model test nhỏ: chèn 2 doc trùng `(userId,type,dedupKey)` → doc thứ 2 ném E11000.
- [ ] **Step 3: Commit** `feat(notification): shared notification log with idempotency and TTL`

---

## Task 4: `reminder.service.remindOne` — orchestrator (AC-1,2,4,6,7,8,10)

**Files:** Create `reminder.service.js`; Modify `email.adapter.js`; Test `reminder-service.test.js`

- [ ] **Step 1: Test thất bại** (seed DB qua helper `db`; mock `addPushUserJob` + `email.adapter.sendStudyReminder`):
  - **AC-7 channel:** user có `fcmTokens` → gọi `addPushUserJob`, KHÔNG gọi email; user không token → gọi email, KHÔNG push.
  - **AC-6:** `profile.notificationPrefs.study=false` → không gửi gì.
  - **AC-8:** dormant (lastActiveDate 20 ngày) → không gửi.
  - **AC-2:** evening đã xong hết hôm nay → không gửi.
  - **AC-4 dedup:** gọi `remindOne` 2 lần cùng (user, ngày-local, slot) → chỉ gửi **1** lần; lần 2 no-op (NotificationLog chặn).
  - **AC-10:** sau khi gửi → có `NotificationLog` đúng `{userId,type:'study-reminder',dedupKey,channel}`.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** (design §5/§6): `remindOne(userId, now)`:
  - đọc user (`studyPlan` + `profile.notificationPrefs`); skip nếu `notificationPrefs?.study===false`; `tz = studyPlan.timezone ?? 'Asia/Saigon'`; `dateLocal=todayLocalDate(tz,now)`; `hour=localHour(tz,now)`; `slot = hour===8?'morning':hour===20?'evening':null`; `if !slot || quietHours(hour): return`;
  - load `LessonSchedule` + `loadDoneContext` + `lastActiveDate` → `view = buildTodayView(...)`; `msg = decideReminder(view, slot)`; `if !msg: return`;
  - `dedupKey = `${dateLocal}:${slot}``; dedup: `NotificationLog.findOne({userId,type:'study-reminder',dedupKey})` tồn tại → return;
  - gửi: `fcmTokens?.length ? addPushUserJob(userId,{title,body},{type:'study-reminder',slot,route:'/schedule'}) : email.sendStudyReminder(...)`; channel tương ứng; **1 lệnh push = multicast mọi thiết bị** (web/android/ios). *(Deep-link web click-through: nếu cần chuẩn, mở rộng `fcm.adapter` thêm `webpush.fcmOptions.link` — xác nhận ở Technical Review.)*
  - `NotificationLog.create({userId,type:'study-reminder',dedupKey,channel,sentAt})` (bắt E11000 → coi như đã gửi).
  - `email.adapter.js`: thêm `async sendStudyReminder({to,name,title,body})` (mẫu `sendRenewalReminder`, Resend).
- [ ] **Step 4:** `npx jest reminder-service -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): send study reminder via push with email fallback, idempotent`

---

## Task 5: Cron queue + worker (AC-3, wiring)

**Files:** Create `queues/reminder.queue.js`, `reminder.worker.js`

- [ ] **Step 1: Implement** (pattern `billing.*`):
  - `reminder.queue.js`: Queue + `scheduleRepeatable()` upsert `{ repeat:{pattern:'0 * * * *'}, jobId:'schedule:reminders:hourly' }`.
  - `reminder.worker.js`: Worker xử lý job hourly → quét học viên `profile.notificationPrefs.study≠false` **có `LessonSchedule`** (join hoặc 2 truy vấn); mỗi user `await remindOne(userId, new Date())` (biên `new Date()` ở đây). Log tổng `{scanned, sent}` (NFR-003). Đăng ký repeatable khi boot.
  - Wire worker vào chỗ khởi động worker hiện có (như `billing.worker`).
- [ ] **Step 2: Verify** — worker load không lỗi; (nếu có harness) 1 test smoke gọi handler với DB seed 2 user (1 đủ điều kiện, 1 dormant) → chỉ 1 gửi. Nếu khó test cron trực tiếp, đã phủ ở `remindOne` (Task 4) — ghi rõ.
- [ ] **Step 3: Commit** `feat(adaptive): hourly repeatable job scanning learners for study reminders`

---

## Task 6: Full suite + purity + coexist

- [ ] **Step 1:** `npx jest reminder-decide reminder-service local-date study-plan-prefs.schema today-view schedule-today.api -i` → ALL PASS (gồm M4-4 không hồi quy).
- [ ] **Step 2:** `grep -RnE "Date\.now|new Date\(\)" src/modules/adaptive/schedule/reminder-decide.js` → không match (thuần).
- [ ] **Step 3:** `grep -Rn "CourseEnrollment" src/modules/adaptive/schedule/reminder*.js` → chỉ đọc (qua loadDoneContext), không ghi.
- [ ] **Step 4:** `grep -RnE "DailyQuest" src/modules/adaptive/` → không match (không tạo store nhiệm vụ).
- [ ] **Step 5:** Cập nhật `docs/api/integration/` nếu có — thêm prefs field `timezone`/`notificationPrefs.study`; không mở rộng ngoài feature.

---

## Self-Review

**Acceptance coverage:** AC-1/2 (decide morning/evening) → T2; AC-3 (≤2/ngày) → T2+T5; AC-4 (idempotent) → T3+T4; AC-5 (quiet hours) → T2; AC-6 (notificationPrefs.study) → T4; AC-7 (fallback email) → T4; AC-8 (dormant) → T2+T4; AC-9 (timezone) → T1; AC-10 (log) → T3+T4. ⬜
**DRY/scope:** không DailyQuest; không in-app store; tái dùng buildTodayView/loadDoneContext/todayLocalDate; email qua adapter; push qua addPushUserJob. ⬜
**Coexist:** M4-4 (today-view/schedule-today.api) + generator xanh. ⬜

---

## Câu hỏi mở (Technical Review — chốt khi implement)

1. **OQ-1** — slot cố định 8/20 vs suy từ `studyPlan.timeSlots`.
2. **OQ-2** — `timezone` thiếu → fallback Asia/Saigon (đề xuất) vs bắt buộc khai.
3. **OQ-3 → CHỐT** — dedup: `NotificationLog` collection dùng chung (`type`/`dedupKey`) — Hybrid 2026-08-07.
4. Cron quét toàn user mỗi giờ — cần index/giới hạn thêm không (dormant đã tự loại ở tầng view).

## Prerequisites môi trường (owner xác nhận trước khi test/deploy — spec §7)
RESEND_API_KEY+EMAIL_FROM · Firebase VAPID+web config · BullMQ worker chạy prod · 1 tài khoản test có push.
