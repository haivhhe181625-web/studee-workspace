# Spec: Chống Spam & Bot ở tầng API — phần Backend (M5)

- **Ngày:** 2026-08-05
- **Tác giả:** AI Brainstorm (từ SRS M5, Plane page `0969376f`) + Vũ Hồng Hải
- **Repos/surfaces ảnh hưởng:** `api` (trọng tâm). `web` (nhúng widget Turnstile — repo `exe-web`, PR riêng) và hạ tầng Cloudflare/DevOps là **phụ thuộc ngoài phạm vi** spec này.
- **Module liên quan (`exe-api`):** `src/middlewares/rateLimit.js`, `src/modules/auth/`, `src/modules/user/`, `src/adapters/email.adapter.js`, `src/admin-api/`
- **Trạng thái:** Nháp — chờ design sign-off

## 1. Mục tiêu

Chỉ **tài khoản đã verify email + qua rate-limit + được xác thực là người thật** mới chạm được endpoint AI tốn tiền; đồng thời chặn đăng ký hàng loạt, dò mật khẩu, và cho admin công cụ dọn tài khoản ảo — **không làm phiền người dùng thật**.

## 2. Bối cảnh

Endpoint AI (chấm phát âm M2, chấm Viết/Nói M3, talk) là thứ thực sự đốt tiền. Bot có thể đăng ký hàng loạt, dò mật khẩu, và spam các endpoint này.

**Đã kiểm tra code — cái gì đã có:**
- `middlewares/rateLimit.js`: 7 limiter Redis (`generalLimiter`, `authLimiter` 20/15′ theo IP, `aiCallLimiter` 30/′/user, `submitLimiter` 6/′/user, `verifyLimiter`, `guestProvisionLimiter`, `firebaseLoginLimiter`). `aiCallLimiter` đã gắn ở `adaptive/assessment/ipa/talk.routes.js`. Có switch `TRIAL_LIMIT_MODE=unlimited` để tắt toàn bộ khi demo.
- `User.emailVerified` (mặc định `false`) đã tồn tại; luồng `modules/auth/email-otp.*` gửi & verify OTP → set `emailVerified=true`.
- Admin surface: `admin-api/_crud.factory.js` + audit hook + soft-delete (dùng lại cho công cụ dọn account).

**Cái gì chưa có (gap của M5):**
- Xác thực Turnstile phía server + honeypot/timing (0 dấu vết trong code).
- **Hard-gate** cấm endpoint AI khi `emailVerified=false` (hiện chỉ có rate-limit, không có gate).
- Chặn email rác: danh sách domain dùng-một-lần, kiểm tra MX, nhận diện biến thể (dấu `.`/`+`).
- Chống dò mật khẩu **per-account** (lockout tăng dần + khóa tạm) — hiện chỉ giới hạn theo IP.
- Chính sách **Redis chết**: đọc fail-open, ghi/AI fail-closed.
- Công cụ admin dọn tài khoản ảo + entity `SuspiciousAccountFlag`; dọn tài khoản chưa verify > 30 ngày.

## 3. Phạm vi

### Trong phạm vi (5 task BE của Vũ Hồng Hải)
- **M5-2 (BE):** endpoint verify token Turnstile phía server (one-time) + honeypot + kiểm tra thời gian submit; trả về pass/deny cho form nhạy cảm.
- **M5-3:** mở rộng rate-limit Redis hiện có — bổ sung **chính sách fail-open (đọc) / fail-closed (ghi + AI)** khi Redis lỗi; đảm bảo chạy đúng đa-instance; giữ ≤ 5ms.
- **M5-4:** chặn email rác (disposable domain + MX + biến thể) + **hard-gate**: `emailVerified=false` → cấm mọi endpoint AI (guest ẩn danh phải nâng cấp + verify mới gọi AI); job dọn tài khoản chưa verify > 30 ngày.
- **M5-5:** chống dò mật khẩu per-account: sai nhiều → tăng dần thời gian chờ → khóa tạm 15′; email cảnh báo đăng nhập bất thường; lỗi đăng nhập không lộ email tồn tại; đăng nhập đúng → reset bộ đếm.
- **M5-7 (phần admin BE):** công cụ admin lọc/khóa/xóa hàng loạt tài khoản nghi ngờ (xóa mềm, xác nhận 2 bước, whitelist, ghi log) qua CRUD factory.

### Ngoài phạm vi
- **M5-1, M5-6 (DevOps):** ngưỡng rate-limit từng endpoint (bảng số), WAF + Bot Fight + origin-lock, dashboard theo dõi + cảnh báo đột biến + runbook — **Lý do:** hạ tầng Cloudflare/observability, do Đỗ Tuấn Anh sở hữu; spec BE chỉ tiêu thụ cấu hình.
- **M5-2 (FE):** nhúng widget Turnstile ngầm vào form — **Lý do:** thuộc `exe-web`, Nguyễn Văn Nghĩa; BE chỉ cung cấp endpoint verify + hợp đồng honeypot/timing.
- **M5-8 (QA):** script giả lập bot + đo % chặn từng lớp — **Lý do:** kiểm thử, Ngô Bảo Linh.
- **Bảng ngưỡng cấu hình runtime (RateLimitConfig DB)** — **Lý do:** đã chốt KISS, mở rộng limiter env-driven sẵn có thay vì DB config (tránh query/cache mỗi request, giữ ≤5ms).
- **Risk scoring heuristic (User.riskScore)** — **Lý do:** đã chốt bỏ; chỉ làm lõi FR-007 (lockout), tránh dương tính giả và phụ thuộc dữ liệu vùng địa lý.
- **SMS verify** — **Lý do:** release này chỉ verify qua email (SRS OQ-3).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Hệ thống | Chỉ cho tài khoản đã verify gọi endpoint AI | 0% AI-call từ tài khoản chưa verify (không đốt tiền) |
| Người dùng thật | Đăng ký/đăng nhập không phải giải câu đố | Không bị rào chắn làm phiền |
| Hệ thống | Chặn gọi API quá nhiều theo IP/account/device | Ngăn lạm dụng, DoS-by-cost |
| Hệ thống | Không cho email rác/dùng-một-lần dùng AI | Ngăn tạo tài khoản ảo hàng loạt |
| Người dùng | Tài khoản an toàn trước dò mật khẩu | Không bị chiếm tài khoản |
| Admin | Dọn tài khoản ảo an toàn | Giữ dữ liệu sạch, có thể hoàn tác |

## 5. Quyết định nghiệp vụ cần chốt

Đã chốt qua hỏi-đáp (2026-08-05):

| Câu hỏi | Đã chốt | Người quyết |
|---|---|---|
| Phạm vi spec | Chỉ BE của Vũ Hồng Hải (5 task) | Vũ Hồng Hải |
| Rate-limit config | Mở rộng limiter sẵn có (không DB config) | Vũ Hồng Hải |
| Hard-gate AI với guest ẩn danh | Guest phải nâng cấp + verify email mới gọi AI | Vũ Hồng Hải |
| Độ sâu chống dò mật khẩu | Lõi FR-007, bỏ risk scoring | Vũ Hồng Hải |

## 6. Acceptance criteria (tóm tắt)

1. **AC-1 (M5-4/gate):** tài khoản `emailVerified=false` (kể cả guest ẩn danh) gọi endpoint AI → bị từ chối kèm hướng dẫn verify; vẫn xem được nội dung miễn phí.
2. **AC-2 (M5-4):** đăng ký bằng email domain dùng-một-lần hoặc MX không hợp lệ → bị từ chối, báo rõ; biến thể (`a.b+x@`) nhận diện trùng tài khoản đã có.
3. **AC-3 (M5-2):** submit form nhạy cảm không kèm token Turnstile hợp lệ → bị chặn; token dùng lại lần 2 → bị từ chối (one-time).
4. **AC-4 (M5-2):** Turnstile lỗi/adblock (không có token) → chỉ cho submit khi honeypot sạch **VÀ** thời gian submit hợp lý (đo phía server); không đạt → chặn.
5. **AC-5 (M5-3):** vượt ngưỡng rate-limit → `429` + `Retry-After`; hết thời gian chờ gọi lại được.
6. **AC-6 (M5-3):** Redis chết → endpoint **đọc** vẫn cho qua (fail-open); endpoint **ghi/AI** bị chặn (fail-closed).
7. **AC-7 (M5-5):** sai mật khẩu nhiều lần cùng account → thời gian chờ tăng dần → khóa tạm 15′ + email cảnh báo; đăng nhập đúng reset bộ đếm.
8. **AC-8 (M5-5):** lỗi đăng nhập với email tồn tại và không tồn tại → thông báo **giống hệt nhau** (không lộ tồn tại email).
9. **AC-9 (M5-4/job):** tài khoản chưa verify quá 30 ngày → bị dọn (xóa mềm) bởi job định kỳ.
10. **AC-10 (M5-7):** admin chọn nhiều tài khoản nghi ngờ → khóa/xóa hàng loạt = xóa mềm + xác nhận 2 bước + ghi log; tài khoản whitelist không bị đụng.
11. **AC-11 (NFR):** rate-limit không làm chậm request > 5ms (đo trên staging).

## 7. Câu hỏi mở

- [ ] Không còn câu hỏi chặn approval. (Ngưỡng số cụ thể từng endpoint do DevOps điền ở M5-1 — không chặn spec BE này.)

## 8. Ghi chú cho Tech Lead Design

- **Ràng buộc hiệu năng:** rate-limit ≤ 5ms/request → không thêm I/O đồng bộ nặng vào hot path; MX-check email chỉ chạy lúc đăng ký (không phải mọi request).
- **Bảo mật:** token Turnstile verify phía server + đánh dấu one-time (chống replay); thông báo lỗi đăng nhập không được rò rỉ tồn tại email; không hardcode secret (dùng env).
- **Tương thích ngược:** không phá `TRIAL_LIMIT_MODE=unlimited` (đang dùng để demo cả team); giữ nguyên hành vi 7 limiter hiện có, chỉ **thêm** chính sách Redis-down + gate.
- **Điểm chạm với M4 (đang chạy song song):** chỉ `email.adapter.js` (M5 thêm mail verify/cảnh báo, M4 thêm mail nhắc học) — tách hàm gửi để tránh xung đột merge. `rateLimit.js` M5 sở hữu, M4 không đụng.
- **Admin:** mọi hành động dọn account phải delegate về service trong `modules/user/` (không nhét business logic vào `admin-api/` — theo `exe-api/CLAUDE.md` §Admin Rules).
- **Guest ẩn danh:** hard-gate cần phân biệt guest Firebase (không email) vs account có email chưa verify — cả hai đều bị chặn AI, nhưng thông điệp hướng dẫn khác nhau (guest → đăng ký; account → verify).
