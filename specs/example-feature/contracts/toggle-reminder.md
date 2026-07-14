<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md -->
# Contract: Toggle nhắc học qua email

- **Loại:** Cross-repo (api↔web)
- **Bên cung cấp (provider):** `exe-api`
- **Bên tiêu thụ (consumer):** `exe-web` (Settings screen)
- **Trạng thái:** VÍ DỤ — không dùng để implement

## Endpoint

```
PATCH /api/users/me/reminder
```

- **Auth:** JWT user (`verifyToken`)
- **Permission required:** không (chỉ áp dụng cho chính user)

## Request

```json
{ "reminderEnabled": true }
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `reminderEnabled` | boolean | Có | |

## Response — thành công

```json
{ "user": { "id": "...", "reminderEnabled": true } }
```

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 401 | `NOT_AUTHENTICATED` | Thiếu/sai token |
| 400 | `VALIDATION_ERROR` | `reminderEnabled` không phải boolean |

## Versioning / breaking change

N/A — contract mới.

## Sequencing triển khai

`exe-api` deploy trước (endpoint hoạt động ngay cả khi chưa có UI). `exe-web` deploy toggle sau, độc lập.

## Ví dụ gọi thực tế

```bash
curl -X PATCH http://localhost:5050/api/users/me/reminder \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"reminderEnabled": true}'
```
