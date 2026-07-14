<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md -->
# Data Model: [VÍ DỤ] Nhắc học qua email hằng ngày

- **Design:** `specs/example-feature/design.md` §3
- **Ngày:** 2026-07-14

## `User` (mở rộng model có sẵn)

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `reminderEnabled` | Boolean | Không | `false` | Bật/tắt nhắc học qua email |

## `ReminderLog` (model mới)

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `userId` | ObjectId (ref `User`) | Có | |
| `date` | String (`YYYY-MM-DD`, giờ VN) | Có | Cùng `userId` + `date` → unique index |
| `sentAt` | Date | Có | Thời điểm email thực sự được gửi |

**Index:** `{ userId: 1, date: 1 }` unique — đảm bảo idempotent nếu job bị chạy lại trong cùng ngày.

**Không cần migration** — cả hai đều là field/collection mới, không đổi field cũ.
