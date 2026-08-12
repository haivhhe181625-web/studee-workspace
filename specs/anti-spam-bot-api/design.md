# Design: Chống Spam & Bot ở tầng API — Backend (M5)

- **Spec:** `specs/anti-spam-bot-api/spec.md`
- **Ngày:** 2026-08-05
- **Tác giả:** Tech Lead Agent + Vũ Hồng Hải
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** không có (không đổi kiến trúc nền; mở rộng middleware + module sẵn có)

## 1. Tóm tắt kiến trúc

M5-BE là **lớp cross-cutting** cắm vào chuỗi middleware sẵn có, không tạo module domain mới. Tất cả bám 5-file module pattern + adapter pattern của `exe-api`. Bốn nhóm thay đổi:

1. **Cửa form (M5-2):** thêm `turnstile.adapter.js` (gọi Cloudflare siteverify) + middleware `turnstile.js` (verify token one-time + honeypot + timing), cắm trước handler các form nhạy cảm.
2. **Rate-limit cứng hơn (M5-3):** mở rộng `rateLimit.js` — thêm chính sách Redis-down fail-open (đọc) / fail-closed (ghi+AI). Giữ nguyên 7 limiter hiện có.
3. **Gate + email sạch (M5-4):** middleware `require-verified-email.js` cắm cạnh `aiCallLimiter` trên route AI; util `email-validation.js` chặn disposable/MX/biến thể lúc đăng ký; job BullMQ dọn account chưa verify > 30 ngày.
4. **Chống dò mật khẩu + admin (M5-5, M5-7):** lockout per-account bằng Redis tích hợp vào `auth.service.login`; admin resource lọc + bulk soft-delete qua CRUD factory (delegate về service trong `modules/user/`).

Đối chiếu `ARCHITECTURE.md`: không mâu thuẫn — external call (Turnstile, MX-DNS) đi qua adapter/util; admin không chứa business logic.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Lưu trạng thái lockout dò mật khẩu | **Redis** (không Mongo) | Hot path, TTL auto-hết-hạn, đồng nhất với `email-otp.service` | Redis chết → không lock được (xem §9) |
| Bảng ngưỡng rate-limit | Giữ limiter env-driven sẵn có | KISS, ≤5ms, tránh query/cache mỗi request | Đổi ngưỡng phải deploy (chấp nhận — DevOps M5-1) |
| Turnstile fail (adblock) | Cho qua nếu honeypot sạch **VÀ** timing hợp lý | SRS OQ-2; không chặn oan người dùng adblock | Bot vượt được nếu giả cả honeypot + timing (lớp 2 chấp nhận) |
| Entity `SuspiciousAccountFlag` | **Bỏ** — admin lọc bằng query trên `User` | Đã bỏ risk scoring/auto-flag → không cần bảng cờ | Lệch data-model SRS (ghi rõ ở đây) |
| Lockout dò mật khẩu | Redis key theo **account+IP**, tăng dần → khóa 15′ | Đúng FR-007; account-key chặn dò 1 tài khoản, IP-key chặn spray | — |
| Hard-gate guest ẩn danh | Chặn AI, phân biệt thông điệp guest vs account chưa verify | Mục tiêu 0% AI-call chưa verify | Guest muốn thử AI phải đăng ký (chấp nhận) |

## 3. Data model

**Không thêm Mongoose model mới.** Sửa/dùng lại:

| Entity | Field | Kiểu | Ghi chú |
|---|---|---|---|
| `User` (có sẵn) | `emailVerified` | Boolean | Dùng lại cho hard-gate — không đổi |
| `User` (có sẵn) | `createdAt`, `lastLoginAt` | Date | Dùng cho job dọn + admin filter |
| Redis key | `loginguard:<emailHash>:<ip>` | JSON `{failCount, lockedUntil}` TTL 15′ | Lockout (M5-5) |
| Redis key | `turnstile:used:<tokenHash>` | flag TTL ~300s | Đánh dấu token one-time (M5-2) |

> Lệch SRS §10.1 có chủ đích: bỏ `RateLimitConfig`, `LoginAttempt` (→ Redis), `SuspiciousAccountFlag` (→ admin query). `User.riskScore` không thêm.

## 4. Luồng dữ liệu

**Form nhạy cảm (register/login/forgot-password):**
```
FE form --(POST + cf-turnstile-response + honeypot + renderedAt)--> route
  authLimiter (rate-limit IP)
  -> turnstile.js  (verify token qua adapter → one-time Redis; nếu no-token → check honeypot+timing)
  -> validate(schema)
  -> controller -> auth.service
```

**Login (thêm lockout M5-5):**
```
POST /auth/login
  authLimiter -> turnstile.js -> validate
  -> auth.service.login:
       loginGuard.check(email, ip)      // Redis: nếu lockedUntil > now → throw (delay/lock)
       user = User.findOne({email})
       comparePassword → sai:
            loginGuard.recordFail()      // tăng failCount, set lockedUntil khi vượt ngưỡng
            (nếu bất thường) emailAdapter.sendLoginAlert()
            throw SAME generic error     // không lộ email tồn tại
       đúng: loginGuard.reset(); sign tokens
```

**Endpoint AI (thêm gate M5-4):**
```
POST /ipa|assessment|talk|adaptive/... (AI)
  verifyToken -> requireVerifiedEmail  // !emailVerified → 403 (guest: "đăng ký"; account: "verify")
  -> aiCallLimiter -> controller
```

**Redis-down policy (M5-3):** limiter/guard bắt lỗi store → nếu limiter thuộc nhóm **đọc** → `next()` (fail-open); nhóm **ghi/AI** → `429/503` (fail-closed).

**Job dọn account (M5-4):** BullMQ repeatable (hằng ngày) → soft-delete `User` với `emailVerified=false AND createdAt < now-30d AND` không có giao dịch/mua.

## 5. Contracts

- `contracts/turnstile-form-fields.md` — **cross-repo (api↔exe-web)**: FE phải gửi field `cf-turnstile-response` (token), field honeypot ẩn (tên chốt), và `renderedAt` (mốc render form để đo timing server-side). BE verify + trả mã lỗi khi fail. Quan trọng vì FE (M5-2) do repo khác (Nghĩa) làm.

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/adapters/turnstile.adapter.js` | Gọi Cloudflare siteverify (env secret) | Tạo |
| `src/middlewares/turnstile.js` | Verify token one-time + honeypot + timing | Tạo |
| `src/middlewares/require-verified-email.js` | Hard-gate AI theo `emailVerified` | Tạo |
| `src/utils/email-validation.js` | Disposable domain + MX check + normalize biến thể | Tạo |
| `src/modules/auth/login-guard.service.js` | Lockout Redis + gửi alert | Tạo |
| `src/middlewares/rateLimit.js` | Thêm fail-open/closed cho store lỗi | Sửa |
| `src/modules/auth/auth.service.js` | Tích hợp login-guard + lỗi non-disclosure | Sửa |
| `src/modules/auth/auth.routes.js` | Cắm `turnstile.js` vào form nhạy cảm | Sửa |
| `src/modules/{ipa,assessment,talk,adaptive}/*.routes.js` | Cắm `require-verified-email` cạnh `aiCallLimiter` | Sửa |
| `src/modules/auth/email-otp.service.js` / register | Gọi `email-validation` khi đăng ký/đổi email | Sửa |
| `src/queues/account-cleanup.queue.js` + `.worker.js` | Job dọn account chưa verify >30d | Tạo |
| `src/modules/user/suspicious-account.service.js` | Query lọc + bulk soft-delete (business logic) | Tạo |
| `src/admin-api/resources/suspicious-account.admin.js` | CRUD-factory config: filter + bulk action (delegate service) | Tạo |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| Token Turnstile thiếu/sai/dùng lại | 403 | `TURNSTILE_FAILED` |
| Honeypot bẩn / timing bất thường | 403 | `BOT_SUSPECTED` |
| Chưa verify email gọi AI (account) | 403 | `EMAIL_NOT_VERIFIED` |
| Guest ẩn danh gọi AI | 403 | `SIGNUP_REQUIRED` |
| Email disposable / MX fail | 422 | `EMAIL_REJECTED` |
| Vượt rate-limit | 429 | `TOO_MANY_REQUESTS` (đã có) |
| Redis down + endpoint ghi/AI | 503 | `SERVICE_UNAVAILABLE` |
| Account bị khóa tạm (dò mật khẩu) | 429 | `ACCOUNT_LOCKED` (kèm `Retry-After`) |
| Đăng nhập sai (email tồn tại hay không) | 401 | `INVALID_CREDENTIALS` (giống hệt nhau) |

## 8. Bảo mật & quyền

- Turnstile secret + Cloudflare qua **env** (không hardcode); token đánh dấu one-time trong Redis (chống replay).
- Lỗi login trả **cùng** `INVALID_CREDENTIALS` cho email-không-tồn-tại và sai-mật-khẩu → không rò rỉ tồn tại email (FR-007).
- Admin dọn account: dùng permission có sẵn `user:manage` (registry `constants/permissions.js`); bulk delete = **soft-delete** + audit hook (factory) + xác nhận 2 bước; whitelist trong service.
- Email hash trong Redis key lockout (không lưu email thô).
- Không phá `TRIAL_LIMIT_MODE=unlimited` — gate/turnstile cũng honor switch này ở môi trường demo.

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Redis chết ↔ lockout dò mật khẩu:** không đọc/ghi được counter → không lock được. Chọn **fail-open cho lockout** (vẫn cho đăng nhập, không khóa) để không chặn người thật; brute-force lúc đó vẫn bị `authLimiter` (nếu Redis store nó cũng fail-open thì degraded — chấp nhận, hiếm, có cảnh báo). Ghi log cảnh báo khi Redis down.
- **MX check latency:** chỉ chạy lúc đăng ký/đổi email, có timeout ngắn (~2s) + cache domain kết quả; MX-check lỗi mạng → cho qua (không chặn oan), vẫn giữ verify email làm chốt.
- **Turnstile one-time TTL:** đặt TTL key = tuổi thọ token CF (~300s) để không phình Redis.
- Không câu hỏi cần spike riêng → không tạo `research.md`.

## 10. Testing strategy

- **Unit:** `email-validation` (disposable list, biến thể dot/+, MX mock); `login-guard` (tăng dần → khóa → reset); `turnstile` middleware (no-token honeypot/timing branches).
- **Integration:** gate AI (unverified/guest → 403; verified → qua); rate-limit fail-open/closed khi mock Redis lỗi; login non-disclosure (2 email trả lỗi giống nhau); bulk soft-delete admin + audit.
- **Không** dùng mock giả để pass build (theo development-rules). Chi tiết case → `tasks.md` (test-first).

## 11. Rollout / cross-repo sequencing

- Từng phần có cờ bật/tắt độc lập (mỗi lớp tắt được mà không sập hệ) — theo SRS §16.2.
- Thứ tự: (1) rate-limit fail-open/closed + gate AI (BE thuần, deploy trước) → (2) email-validation + job dọn → (3) login-guard → (4) Turnstile BE endpoint **rồi** FE `exe-web` nhúng widget (contract `turnstile-form-fields.md` phải chốt trước khi Nghĩa làm FE) → (5) admin tool.
- DevOps (M5-1 ngưỡng, M5-6 WAF/dashboard) chạy song song, không chặn BE.

## 12. Đối chiếu Acceptance criteria

| AC (spec §6) | Đáp ứng bởi |
|---|---|
| AC-1 gate AI | `require-verified-email.js` §4, §6 |
| AC-2 email rác | `email-validation.js` §6 |
| AC-3 Turnstile one-time | `turnstile.js` + adapter §4 |
| AC-4 honeypot/timing | `turnstile.js` no-token branch §4 |
| AC-5 429 rate-limit | limiter sẵn có §3 |
| AC-6 fail-open/closed | `rateLimit.js` sửa §4 |
| AC-7 lockout+alert | `login-guard.service.js` §4 |
| AC-8 non-disclosure | `auth.service.login` §4, §8 |
| AC-9 dọn >30d | `account-cleanup.worker.js` §4 |
| AC-10 admin bulk | `suspicious-account.*` §6, §8 |
| AC-11 ≤5ms | không thêm I/O hot path §2 |
