# Spec: Thông báo nhắc học (Study Schedule Reminders · M4-5)

- **Ngày:** 2026-08-06
- **Tác giả:** BA Agent (từ SRS M4 FR-010, PRD-90, owner Vũ Hồng Hải)
- **Trạng thái:** CHỜ DUYỆT — 4 quyết định nghiệp vụ đã chốt (owner 2026-08-06, §2); còn Prerequisites môi trường cần owner xác nhận trước khi code (§7).
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — prefs (tz + bật/tắt nhắc), job scheduler nhắc học, gửi qua FCM/Email, log idempotent.
  - `web` (`exe-web`) — **ngoài phạm vi chính**; chỉ cần màn cài đặt "bật/tắt nhắc" + xin quyền Web Push (đã có `use-push-notifications`); in-app = widget "Hôm nay" của M4-4.
- **Module liên quan (exe-api):**
  - `src/modules/adaptive/schedule/` — **tái dùng M4-4**: `today-view.js` (`buildTodayView`, dormant, progress), `done-context.js` (`loadDoneContext`), `local-date.js` (`todayLocalDate`).
  - `src/modules/user/` — `studyPlan` (thêm `timezone`); `profile.notificationPrefs.study` (bật/tắt, **dùng chung**); `streak.lastActiveDate` (dormant); `fcmTokens`.
  - `src/adapters/` — `fcm.adapter` (push) + `email.adapter` (Resend, thêm method `sendStudyReminder`).
  - `src/queues/` — `notification.queue` (`addPushUserJob`) + **queue/worker cron mới cho nhắc học** (pattern `billing.worker` repeatable).
- **Backlog ref:** PRD-90 (M4-5) · SRS M4 FR-010 (§8.3) · AC-010→013 · EC-007→009 (Plane page 311545ba). Nối tiếp `specs/study-schedule-catchup` (M4-4).

## 1. Mục tiêu

Nhắc học **đúng giờ theo múi giờ học viên**, **không spam** (≤2 tin/ngày, quiet hours, không nhắc nếu đã học xong), qua **Web Push (chính) + Email (fallback)**. Tận dụng M4-4 để biết "hôm nay có gì / đã xong chưa" và để **loại người ngủ đông** khỏi vòng nhắc (tiết kiệm chi phí + đỡ phiền).

## 2. Quyết định nghiệp vụ (owner 2026-08-06)

| # | Quyết định |
|---|---|
| Nguồn "hôm nay xong chưa" | ✅ **KHÔNG tạo `DailyQuest` store** — suy từ M4-4 (`buildTodayView` + `CourseEnrollment`); chỉ done/not-done, không trạng thái "doing". *(Deviation SRS §10.1 có DailyQuest entity — owner chốt bỏ.)* |
| 2 tin/ngày | ✅ **Hệ thống định sẵn:** **sáng** "Hôm nay bạn có N bài" + **tối** "Còn X bài chưa xong". Không cho user tự đặt giờ ở v1. |
| Kênh in-app | ✅ **Tái dùng widget "Hôm nay" của M4-4** — v1 chỉ **Web Push + Email**, không xây notification center. |
| Email | ✅ **Fallback-only** — chỉ gửi khi Web Push bị từ chối/không hỗ trợ/không có token (đúng FR-010). |

## 3. Quy tắc bắt buộc (FR-010 + AC-010→013 + EC-007→009)

- **≤ 2 tin/ngày/học viên** (tối đa: sáng + tối).
- **Quiet hours 22h–7h** (giờ học viên) → không gửi.
- **Không gửi nếu đã học xong hôm nay** (tin tối chỉ gửi khi còn bài chưa xong).
- **Idempotent** — cron chạy lặp/2 lần cùng giờ → **không gửi trùng** (dedup theo `userId + type + dedupKey('${ngày-local}:${slot}')`).
- **`notificationPrefs.study=false` → không gửi gì.**
- **Web Push bị từ chối/không hỗ trợ/không có token → fallback Email.**
- **Gửi đúng giờ theo TZ học viên** (server-side, cron).
- **Loại người ngủ đông** (≥14 ngày bất hoạt — M4-4) khỏi vòng nhắc.

## 4. Phạm vi

### Trong phạm vi

- **IS-1 — Prefs mở rộng:** `studyPlan.timezone` (IANA, mặc định `'Asia/Saigon'` nếu thiếu) + `profile.notificationPrefs.study` (bool, mặc định `true`, **dùng chung — về sau thêm `.marketing`/`.streak` không migrate lại**). Joi + schema + endpoint prefs sẵn có (`PUT /adaptive/schedule/prefs`).
- **IS-2 — Scheduler nhắc học (cron):** BullMQ repeatable (mỗi giờ, `0 * * * *`, upsert theo `jobId`) quét học viên **active + có lịch + `notifEnabled` + không dormant**; theo `timezone` mỗi người, khi **giờ-local = slot sáng (8:00) / tối (20:00)** thì tính nội dung + gửi.
- **IS-3 — Quyết định gửi (per học viên, per slot):** tái dùng `buildTodayView` (M4-4) để lấy `{ todayCount, allDone, remaining }`:
  - **Sáng:** có bài hôm nay → "Hôm nay bạn có N bài"; không có bài → **bỏ**.
  - **Tối:** chưa xong hết → "Còn X bài chưa xong"; đã xong hết → **bỏ** (EC-007).
  - Quiet hours / `notificationPrefs.study=false` / dormant → **bỏ**.
- **IS-4 — Kênh + fallback (đa thiết bị):** có `fcmTokens` (bất kỳ nền tảng: `web`/`android`/`ios`) → Web/Mobile Push qua `addPushUserJob` — **1 lệnh multicast tới MỌI thiết bị của user, tính là 1 tin**; **không có token nào** → Email (`email.adapter.sendStudyReminder`). Payload kèm **deep-link** `data:{ type:'study-reminder', slot, route:'/schedule' }` để chạm tin mở đúng màn "Hôm nay" (web SW + app mobile tự route). `FcmTokenSchema.platform` đã có sẵn → không đổi schema.
- **IS-5 — Idempotent + observability (log dùng chung):** log mỗi lần gửi vào `NotificationLog` (`userId, type:'study-reminder', dedupKey:'${dateLocal}:${slot}', channel, sentAt`) — vừa để **dedup** (đã có (userId,type,dedupKey) → bỏ), vừa để **quan sát** (NFR-003). Collection **dùng chung mọi feature** (owner chốt Hybrid) — feature khác ghi `type` riêng.

### Đa thiết bị (web + mobile) — đã tính

- **Gửi:** `sendMulticast` platform-agnostic → 1 tin tới cả web + Android + iOS của user; dedup theo `(userId,dateLocal,slot)` ⇒ ≤2/ngày/**người** (không nhân theo thiết bị).
- **Deep-link:** payload có `route:'/schedule'` để mọi client mở màn "Hôm nay" khi chạm.
- **Timezone:** 1 `studyPlan.timezone`/người (server tính giờ gửi). **Client (web + mobile) phải cập nhật `timezone` khi mở app/đăng nhập** (PATCH prefs) để nhắc đúng giờ khi người học đổi múi giờ — nếu không, dùng giá trị đã lưu / fallback Asia/Saigon.
- **iOS:** FCM→iOS cần **APNs Auth Key** đã nạp vào Firebase project (xem Prerequisites). Web push trên **Safari/iOS** chỉ chạy khi cài PWA (iOS ≥16.4) — hạn chế của nền tảng, không phải bug.

### Ngoài phạm vi

- **Render in-app trên mobile** — là việc của app mobile; BE chỉ gửi push, mỗi client tự hiển thị. In-app trên web = widget M4-4.
- **In-app notification center/feed** — tái dùng widget M4-4 (v1).
- **User tự đặt `reminderTimes`** — slot cố định sáng/tối cho v1.
- **`DailyQuest` store** — suy từ M4-4.
- **Prompt "resolve-overdue" qua thông báo** — M4-4 đã xử lý bằng banner "Dời lịch" (cảnh báo mềm).
- **Gamification (streak/XP)**, **lịch theo giờ-phút cụ thể**, **SMS/Zalo**.
- **Đổi lõi M4-4** — chỉ *gọi lại* `buildTodayView`/`loadDoneContext`.

## 5. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Được nhắc học đúng giờ, không spam | Không quên học, không phiền |
| Learner (đã học xong) | Không bị nhắc nữa hôm đó | Đỡ khó chịu |
| Learner (từ chối push) | Vẫn nhận nhắc qua Email | Không bỏ lỡ |
| Learner (tắt nhắc) | Không nhận gì | Tôn trọng lựa chọn |
| Hệ thống | Bỏ qua người ngủ đông | Tiết kiệm chi phí + đỡ spam |

## 6. Acceptance criteria

1. **AC-1** (IS-2/3): học viên `notifEnabled` + không dormant + có bài hôm nay, tại **giờ-local 8:00** → nhận **1** tin sáng "Hôm nay bạn có N bài"; không có bài → **không** gửi.
2. **AC-2** (IS-3, EC-007/AC-010): tại **giờ-local 20:00**, còn bài chưa xong → tin tối "Còn X bài chưa xong"; **đã xong hết → KHÔNG gửi**.
3. **AC-3** (≤2/ngày): tối đa **2 tin/ngày/học viên** (sáng + tối).
4. **AC-4** (IS-5 idempotent/AC-008-analog): cron chạy lại cùng giờ/ngày → **không gửi trùng** (dedup theo `userId+dateLocal+slot`).
5. **AC-5** (quiet hours/AC-011): trong 22h–7h local → **không** gửi.
6. **AC-6** (AC-013): `notificationPrefs.study=false` → **không** nhận tin nào.
7. **AC-7** (fallback/AC-012): không có `fcmTokens` (push bị từ chối/không hỗ trợ) → gửi **Email**; có token → Web Push, **không** gửi Email song song.
8. **AC-8** (dormant): học viên bất hoạt ≥14 ngày → **bị loại** khỏi vòng quét (không tin nào).
9. **AC-9** (timezone): thời điểm gửi tính theo `studyPlan.timezone` (mặc định `Asia/Saigon`); đổi tz trong prefs → nhắc kế theo tz mới.
10. **AC-10** (observability/NFR-003): mỗi lần gửi ghi log `NotificationLog{userId, type:'study-reminder', dedupKey, channel, sentAt}`.

## 7. Prerequisites môi trường (owner xác nhận TRƯỚC khi code/test)

- [ ] **Resend**: `RESEND_API_KEY` + `EMAIL_FROM` (domain đã verify) set ở dev/prod. *(Thiếu key → adapter chỉ log, không gửi thật — test được nhưng không ra mail.)*
- [ ] **BE Firebase Admin**: 3 env `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY` (service-account key của project) — gửi được cho cả web + mobile qua `sendMulticast`.
- [ ] **Web Push**: Firebase VAPID key + web config (exe-web) → trình duyệt lấy `fcmToken` (SW `firebase-messaging-sw.js` đã có).
- [ ] **iOS mobile**: **APNs Auth Key** đã nạp vào Firebase project (Console → Cloud Messaging → Apple app configuration) — bắt buộc để FCM đẩy tới iOS. Android không cần.
- [ ] **BullMQ worker** chạy trong prod (để cron nhắc chạy thật).
- [ ] **1 tài khoản test** có push (web + 1 thiết bị mobile nếu test mobile) → test e2e nhắc học.

## 8. Câu hỏi mở (chốt trong Technical Review)

- [ ] **OQ-1:** Slot sáng/tối cố định **8:00 / 20:00** hay suy từ `studyPlan.timeSlots` (morning/noon/evening đã có)? (đề xuất: cố định 8/20 cho v1, đơn giản.)
- [ ] **OQ-2:** `timezone` thiếu → mặc định `Asia/Saigon` (đề xuất) hay bắt buộc khai? (nối OQ-2 của M4-4.)
- [x] **OQ-3 → chốt:** Dedup store = collection `NotificationLog` dùng chung (`type`/`dedupKey`) — owner chốt Hybrid 2026-08-07 (auditable + tái dùng cho feature khác).
