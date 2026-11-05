---
feature: lms-student-identity-login
role: tech-lead → developer
status: draft
date: 2026-08-22
test_runner: cd exe-api/services/api && npx jest <path>
---

# Tasks — mã học viên & đăng nhập bằng mã (M10-9, M10-10)

> Làm trước/song song M10-8 (import cần helper sinh mã/mật khẩu). Chốt 3 câu hỏi mở
> (spec §Q) trước — đặc biệt Q1 (giải tổ chức lúc login) chặn Task 5.

## Nhóm A — Định danh & mật khẩu (M10-9, PRD-218)

- [ ] **A1. Thêm field User:** `studentCode`, `mustChangePassword`, `enrollmentBranchId`
  + index unique `(organizationId, studentCode)` (partial: chỉ khi studentCode tồn tại).
  Test: ghi trùng mã trong 1 tổ chức bị DB chặn; 2 tổ chức cùng mã OK.
  · verify: `npx jest src/modules/user/__tests__/student-code-unique.test.js`

- [ ] **A2. `generateStudentCode(organizationId)` — tuần tự, an toàn concurrency.**
  Counter theo tổ chức (`findOneAndUpdate $inc upsert`). Test: 100 lần gọi song song →
  100 mã khác nhau, không nhảy cóc trùng.
  · verify: `npx jest src/modules/user/__tests__/generate-student-code.test.js`

- [ ] **A3. `generateTempPassword` + tạo student account helper.** Mật khẩu ngẫu nhiên
  đủ mạnh, hash, `mustChangePassword=true`. Trả plaintext **chỉ 1 lần** cho caller.
  · verify: `npx jest src/modules/user/__tests__/create-student.test.js`

- [ ] **A4. File đăng nhập tải-một-lần.** commit import (M10-8) trả CSV/xlsx trong
  response body (stream/buffer), **không ghi đĩa**. Test: sau request không có file tạm.
  · verify: `npx jest src/modules/student-import/__tests__/login-file.test.js`

- [ ] **A5. `POST /api/admin/students/:id/reset-password`.** Sinh mật khẩu mới +
  `mustChangePassword`, trả plaintext 1 lần. Chỉ owner/manager trong scope. Test:
  reset 1 em không đổi mật khẩu em khác.
  · verify: `npx jest src/modules/student/__tests__/reset-password.test.js`

## Nhóm B — Đăng nhập bằng mã (M10-10, PRD-219)

- [ ] **B1. Kênh auth mới `loginByStudentCode`.** Input: `{ organizationId|slug, studentCode,
  password }` (giải tổ chức theo Q1). Xác thực trong phạm vi org. authLimiter. Test:
  2 tổ chức cùng `HS001` không nhầm; sai mật khẩu → 401.
  · verify: `npx jest src/modules/auth/__tests__/student-code-login.test.js`

- [ ] **B2. Ép đổi mật khẩu lần đầu.** Khi `mustChangePassword=true`: token/response gắn
  cờ; middleware chặn mọi route ngoài `change-password`. Sau đổi → tắt cờ. Test:
  chưa đổi → gọi route khác 403; đổi xong → truy cập bình thường.
  · verify: `npx jest src/modules/auth/__tests__/force-password-change.test.js`

- [ ] **B3. Contract khớp FE exe-lms.** `exe-lms` đã có màn student-code + change-password
  login (commit #16, #a3c6c70). Khớp path/shape, không sửa FE. Integration end-to-end.
  · verify: `npx jest src/modules/auth/__tests__/student-login.routes.test.js`

## Kiểm tra tổng
- [ ] `npm test` xanh · [ ] không file mật khẩu tồn trên đĩa · [ ] rate-limit login mã.
