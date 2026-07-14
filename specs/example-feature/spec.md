<!-- VÍ DỤ MINH HOẠ — không phải feature thật, không được implement. Mục đích: cho AI agent và người mới thấy
     đúng "hình dạng" của một spec hoàn chỉnh. Xem specs/README.md để biết cách dùng cấu trúc này cho feature thật. -->
# Spec: [VÍ DỤ] Nhắc học qua email hằng ngày

- **Ngày:** 2026-07-14
- **Tác giả:** BA Agent (ví dụ minh hoạ)
- **Repos/surfaces ảnh hưởng:** `exe-api` (endpoint bật/tắt + job gửi mail), `exe-web` (toggle trong Settings)
- **Module liên quan:** `<API_REPO>/services/api/src/modules/user`, module mới `<API_REPO>/services/api/src/modules/reminder`
- **Trạng thái:** VÍ DỤ — không dùng để implement

## 1. Mục tiêu

Cho phép người dùng bật/tắt nhận email nhắc học mỗi ngày nếu chưa hoàn thành mục tiêu học (`dailyGoalMinutes`)
trước 20h giờ Việt Nam.

## 2. Bối cảnh

`User` đã có field `dailyGoalMinutes` (xem `<API_REPO>/docs/superpowers/plans/2026-06-15-user-profile-edit-avatar.md`).
Chưa có cơ chế nhắc nhở nào. `Resend` đã được dùng để gửi email (reset password) — có thể tái dùng adapter email
hiện có thay vì tích hợp provider mới.

## 3. Phạm vi

### Trong phạm vi

- Toggle bật/tắt nhắc học trong Settings (`exe-web`).
- Job chạy hằng ngày, kiểm tra user có bật toggle + chưa đạt `dailyGoalMinutes` hôm đó → gửi email.

### Ngoài phạm vi

- Nhắc qua push notification (FCM) — **Lý do:** đã có luồng FCM riêng cho việc khác, tách feature để không phình
  phạm vi ví dụ này.
- Tuỳ chỉnh giờ nhắc theo từng user — **Lý do:** MVP dùng giờ cố định 20h, đo nhu cầu thực tế trước khi thêm cấu hình.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Bật nhắc học qua email | Không quên mục tiêu học hằng ngày |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| Có tính phí gửi email cho user free tier không? | Không giới hạn ở MVP | Product owner |

## 6. Acceptance criteria (tóm tắt)

1. User bật toggle → nhận email nhắc nếu chưa đạt mục tiêu lúc 20h.
2. User tắt toggle → không nhận email.
3. User đã đạt mục tiêu trước 20h → không nhận email dù toggle bật.

## 7. Câu hỏi mở

*(Không còn — đây là ví dụ, coi như đã được duyệt.)*

## 8. Ghi chú cho Tech Lead Design

Tái dùng adapter email hiện có (dùng cho reset password) thay vì thêm provider mới.
