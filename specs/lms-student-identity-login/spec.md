---
feature: lms-student-identity-login
module: M10 · LMS Phase 1
plane_tasks: [PRD-218 (M10-9), PRD-219 (M10-10)]
repo: exe-api (services/api)
status: draft
tier: lite
date: 2026-08-22
---

# Spec — Mã học viên, mật khẩu sinh sẵn & đăng nhập bằng mã

## Mục tiêu
Học sinh nhỏ tuổi **không có email** vẫn có tài khoản và tự đăng nhập ở `exe-lms`,
bằng **mã học viên + mật khẩu** do tổ chức cấp. Trung tâm phát tài khoản cả lớp một
lần mà không lộ mật khẩu về sau.

## Bối cảnh — trạng thái code
- `User` đã có `orgRole`, `organizationId` (M10-4). **Chưa** có field `studentCode`,
  `mustChangePassword`, `enrollmentBranchId` trên schema — cần thêm (khảo sát 22/08:
  `auth.schema.js` không có student-code login).
- Đăng nhập hiện chỉ nhận email (`auth.schema.login`). `User.email` đã `sparse` nên
  model không cản tài khoản không email.
- **Phụ thuộc ngược:** M10-8 (import commit) cần helper sinh mã/mật khẩu của M10-9 →
  nên làm **M10-9 trước hoặc song song** M10-8.

## Phạm vi

**M10-9 — Mã học viên + mật khẩu sinh sẵn + file tải-một-lần** (PRD-218)
- `studentCode` duy nhất **trong tổ chức** (không toàn hệ). Bỏ trống khi tạo → hệ
  **tự sinh tuần tự** (vd `HS001`, `HS002`).
- Mật khẩu **ngẫu nhiên** mỗi em khi tạo, đặt `mustChangePassword=true`.
- **File thông tin đăng nhập** chỉ sinh khi commit thành công, trả **tải-một-lần ngay
  trong response**, **không lưu đĩa** server.
- `POST /api/admin/students/:id/reset-password` — cấp lại **từng em** (mật khẩu mới +
  `mustChangePassword`), không tái tạo file lô.

**M10-10 — Đăng nhập bằng mã học viên + bắt đổi mật khẩu lần đầu** (PRD-219)
- Mở **kênh đăng nhập mới** ở tầng auth: `mã học viên + mật khẩu`, xác định trong
  **phạm vi một tổ chức** (mã chỉ unique theo org → request phải mang được tổ chức
  nào: qua org slug / chọn tổ chức / subdomain — chốt ở design).
- Đăng nhập tại **chính exe-lms**, không phụ thuộc exe-web.
- `mustChangePassword=true` → ép đổi mật khẩu lần đăng nhập đầu, **chặn mọi màn khác**
  cho tới khi đổi.
- **Không gộp** với tài khoản B2C sẵn có (AD-08) — chấp nhận 2 tài khoản.

## Ngoài phạm vi
- Nhập lô (M10-8, bộ spec `lms-org-branch-staff-import`) — chỉ tiêu thụ helper của M10-9.
- Parent OTP login (M10-14).

## Acceptance Criteria
**M10-9**
- [ ] Mã tự sinh không đụng mã đã có trong tổ chức (kể cả khi chạy đua 2 lô).
- [ ] Sau khi tải file đăng nhập, **không** còn file mật khẩu nào trên đĩa server.
- [ ] `reset-password` 1 em → chỉ em đó đổi, em khác không ảnh hưởng.
- [ ] `mustChangePassword` bật cho mọi tài khoản mới tạo.

**M10-10**
- [ ] Hai tổ chức cùng mã `HS001` **không** đăng nhập nhầm nhau.
- [ ] Chưa đổi mật khẩu lần đầu → mọi endpoint/màn khác bị chặn (chỉ cho đổi mật khẩu).
- [ ] Sai mã/mật khẩu → lỗi rõ ràng, rate-limited (authLimiter).

## Câu hỏi mở
1. **Giải tổ chức lúc login bằng mã:** cơ chế nào để biết mã `HS001` thuộc tổ chức
   nào? (a) người dùng chọn tổ chức từ dropdown, (b) org slug nhập kèm, (c) subdomain
   `abc.lms.studee.edu.vn`. → cần chốt UX với BA/FE (ảnh hưởng exe-lms login form).
2. **Sinh mã tuần tự an toàn concurrency:** counter theo tổ chức (collection riêng /
   `findOneAndUpdate $inc`) để 2 lô không cấp trùng — chốt cơ chế ở design.
3. `studentCode` unique index `(organizationId, studentCode)` — thêm ở M10-9 hay đã có
   ở M10-4? (kiểm tra; M10-8 D0 cũng đụng ràng buộc này — làm 1 lần).
