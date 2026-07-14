<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md -->
# Design: [VÍ DỤ] Nhắc học qua email hằng ngày

- **Spec:** `specs/example-feature/spec.md`
- **Ngày:** 2026-07-14
- **Tác giả:** Tech Lead Agent (ví dụ minh hoạ)
- **Trạng thái:** VÍ DỤ — không dùng để implement
- **ADR liên quan:** không có

## 1. Tóm tắt kiến trúc

Module mới `reminder` (không có model riêng — đọc trực tiếp `User` + 1 collection log nhỏ `ReminderLog` để
tránh gửi trùng). BullMQ cron job (đã có hạ tầng queue) chạy hằng ngày 20h VN, quét user `reminderEnabled: true`,
gọi adapter email hiện có.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Cơ chế lịch | BullMQ repeatable job | Hạ tầng BullMQ đã có sẵn, không thêm dependency | Phụ thuộc Redis còn sống lúc 20h |
| Chống gửi trùng | `ReminderLog { userId, date }` unique index | Đơn giản, idempotent nếu job chạy lại | Thêm 1 collection nhỏ |

## 3. Data model

`ReminderLog`: `{ userId: ObjectId, date: String (YYYY-MM-DD), sentAt: Date }`, unique index `(userId, date)`.
`User` thêm field `reminderEnabled: Boolean, default: false`.

## 4. Luồng dữ liệu

```
BullMQ repeatable job (20:00 Asia/Ho_Chi_Minh)
  -> query User where reminderEnabled = true
  -> for each: tính tổng phút học hôm nay, so với dailyGoalMinutes
  -> nếu < mục tiêu và chưa có ReminderLog hôm nay -> adapter email -> ghi ReminderLog
```

## 5. Contracts

- `contracts/toggle-reminder.md` — `PATCH /api/users/me/reminder` (exe-api ↔ exe-web).

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `<API_REPO>/services/api/src/modules/user/user.schema.js` | Thêm `reminderEnabled` vào `updateMe` | Sửa |
| `<API_REPO>/services/api/src/modules/reminder/reminder.model.js` | `ReminderLog` schema | Tạo |
| `<API_REPO>/services/api/src/modules/reminder/reminder.job.js` | BullMQ repeatable job + logic | Tạo |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| `dailyGoalMinutes` null khi job chạy | — (job bỏ qua, không phải API lỗi) | — |

## 8. Bảo mật & quyền

Không có permission mới — toggle chỉ áp dụng cho chính user (`req.user.id`), không có tenant scope liên quan.

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

Không có — ví dụ này đơn giản, không cần `research.md` riêng (file `research.md` trong ví dụ này để trống có chủ
đích, xem ghi chú trong file đó).

## 10. Testing strategy

Unit test cho job (mock adapter email + `ReminderLog`), API test cho `PATCH /me/reminder`.

## 11. Rollout / cross-repo sequencing

`exe-api` (endpoint + job) deploy trước; `exe-web` (toggle UI) deploy sau, có thể độc lập vì thiếu UI không làm
vỡ API.

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion | Được đáp ứng bởi |
|---|---|
| AC-1 | §4 luồng dữ liệu + adapter email |
| AC-2 | §4 — job filter `reminderEnabled: true` |
| AC-3 | §4 — so sánh tổng phút học với `dailyGoalMinutes` |
| AC-E1 | §4 — bỏ qua user không có mục tiêu |
