---
feature: lms-parent-student-portal-api
module: M10 · LMS Phase 1
plane_tasks: [PRD-222 (M10-13), PRD-223 (M10-14), PRD-224 (M10-15), PRD-225 (M10-16), PRD-226 (M10-17)]
repo: exe-api (services/api)
status: draft
tier: lite
date: 2026-08-22
---

# Spec — Tài khoản phụ huynh, OTP, portal phụ huynh & khu học viên + audit

## Mục tiêu
Phụ huynh xem được **con mình và chỉ con mình** (không mật khẩu, đăng nhập OTP email);
học sinh xem lớp của mình (không xem của bạn); và ghi nhật ký các thao tác nhạy cảm.

## Bối cảnh — trạng thái code (ĐÃ CÓ MỘT PHẦN)
- Bề mặt `/api/lms/parent` + `/api/lms/classes` đã tồn tại (M10-2/16 nền):
  `GET /parent/children`, `GET /parent/children/:id/classes`, `GET /classes/mine`.
- Guard `denyGuardianWrites` (parent chỉ đọc) đã áp ở `lms-api/_router`.
- `email-otp` module đã có (`/api/auth/email-otp`) — M10-14 **tái dùng/mở rộng**.
- Bộ quyền `parent`/`student` (`MY_*`) đã khai (M10-3).
- **Lệch prefix cần chốt:** FE exe-lms gọi `/api/parent/*` và `/api/classes/mine`,
  backend đặt dưới `/api/lms/*`. Phải thống nhất (sửa FE theo `/lms` **hoặc** BE thêm
  alias) — xem §Q1.
- **Thiếu:** `POST /admin/parents` (+ link children), `/parent/otp/*`, `/parent/children/:id/overview`
  (hiện chỉ có `/classes`), audit log.

## Phạm vi

**M10-13 — Tạo & liên kết tài khoản phụ huynh** (PRD-222)
- `POST /api/admin/parents` (tạo bằng email) · `POST /api/admin/parents/:id/children`
  (liên kết `guardianOf[]`). Owner/manager, **từng ca**, không nhập lô Phase 1.
- Ràng buộc: con cùng tổ chức + vai trò `student`; **1 con ↔ ≤1 phụ huynh** (O4) →
  con đã có phụ huynh khác thì **409**; 1 phụ huynh ↔ **≤20 con**.

**M10-14 — Đăng nhập OTP email phụ huynh** (PRD-223)
- `POST /api/parent/otp/request` · `POST /api/parent/otp/verify`. Không mật khẩu.
  Cổng **trung lập kênh** để cắm phone-OTP (Firebase) sau **không đổi mô hình**.
- Chốt khi làm: thời hạn OTP, số lần sai, chống gửi dồn (liên quan M5 anti-spam).

**M10-15 — API portal phụ huynh (chỉ đọc)** (PRD-224)
- `GET /api/parent/children` · `GET /api/parent/children/:id/overview`. Lọc theo người
  gọi — không đọc ngoài `guardianOf` (R3), kể cả truyền `studentId` lạ. `/api/parent/*`
  **chỉ đọc**. Phase 1 trả: hồ sơ con + lớp đang học (results/attendance = khung, treo).

**M10-16 — API khu học viên (my-\*)** (PRD-225)
- `/api/classes/*` với `my-class:read`. **Ép `studentId` từ token**, không từ request.
  Phase 1: danh sách lớp active + chi tiết lớp (giáo viên, giáo trình). Làm/nộp bài,
  xem điểm = treo phase sau.

**M10-17 — Audit log thao tác nhạy cảm** (PRD-226, ưu tiên thấp nhất)
- Ghi log: tạo/vô hiệu hoá tài khoản · đổi vai trò · liên kết/gỡ phụ huynh · reset mật
  khẩu học viên · truy vấn xuyên tổ chức (`bypassTenant`). Mỗi log: ai, khi nào, tác
  động ai, tổ chức nào. (Hàm `auditLog` đã tồn tại — mở rộng phạm vi.)

## Ngoài phạm vi
- Gỡ liên kết & vô hiệu hoá tài khoản phụ huynh (FR-PAR-05 — treo phase sau).
- Kết quả/chuyên cần/bài tập/nhận xét của con (khung sẵn, dữ liệu ở phase sau).

## Acceptance Criteria
- [ ] **(M10-13)** Liên kết con đã có phụ huynh → 409; con thứ 21 → chặn; con tổ chức khác → chặn.
- [ ] **(M10-14)** OTP hết hạn/sai quá số lần → từ chối rõ; thêm kênh phone sau không đổi model; parent không có đường đặt mật khẩu.
- [ ] **(M10-15)** Truyền id con nhà khác → 403/404 không rò dữ liệu; không endpoint ghi nào dưới `/api/parent`.
- [ ] **(M10-16)** Truyền `studentId` khác trong body/query → bỏ qua, luôn dùng token; không đọc được bạn cùng lớp.
- [ ] **(M10-17)** 4 nhóm thao tác đều có log; truy được theo tổ chức + người thực hiện.

## Câu hỏi mở
1. **Prefix `/api/parent` & `/api/classes/mine` (FE) vs `/api/lms/*` (BE)** — sửa FE hay
   BE thêm alias? (khuyến nghị: sửa FE theo `/lms` — đúng tách namespace BE). Chặn M10-15/16.
2. `/parent/children/:id/overview` shape: hiện BE có `/classes`; overview cần gộp
   hồ sơ + classes — chốt field trả về.
3. M10-14 tham số OTP (TTL, max attempts, resend cooldown) — chốt số cụ thể.
