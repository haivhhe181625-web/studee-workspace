---
feature: lms-class-membership-api
module: M10 · LMS Phase 1
plane_tasks: [PRD-220 (M10-11), PRD-221 (M10-12)]
repo: exe-api (services/api)
status: draft
tier: lite
date: 2026-08-22
---

# Spec — API lớp học + thành viên (đơn lẻ & hàng loạt)

## Mục tiêu
Owner/manager dựng lớp, gán giáo viên/giáo trình, và **xếp học sinh vào lớp** (đơn lẻ
+ hàng loạt) — luồng dùng thường xuyên nhất. Giữ đúng **lịch sử vào/ra lớp** để về sau
tính chuyên cần.

## Bối cảnh — trạng thái code (ĐÃ CÓ MỘT PHẦN)
- `class.model.js`: `ClassGroup` + `ClassMembership` đầy đủ, partial-unique "1 active
  membership / (student,class)" (R8) đã có (M10-4).
- `class.routes.js` (mount `/api/admin/classes`): đã có `GET/POST/PATCH/DELETE /`,
  `GET/POST/DELETE /:id/students`, `/:id/teachers`, `/:id/courses`, `reconcile-enrollments`.
- `class.controller.addStudents` đã nhận `studentIds[]` (đã hỗ trợ nhiều em).
- **Thiếu:** đổi tên `/students` → **`/members`** (khớp FE), `GET /:id/eligible-students`,
  `POST /:id/members/bulk-add` (report idempotent + transaction), `POST /:id/members/bulk-remove`.
  FE `class-members.service.ts` gọi `/members`, `/members/bulk-add`, `/members/bulk-remove`,
  `/eligible-students`.

## Phạm vi

**M10-11 — Class API + thành viên đơn lẻ** (PRD-220)
- `GET/POST/PATCH /api/admin/classes` — lớp **bắt buộc** thuộc 1 cơ sở, gán `teacherIds[]`,
  `courseStructureId`. (Phần lớn đã có — verify + khớp contract.)
- `GET /classes/:id/members` · `POST /classes/:id/members` (thêm 1 em) ·
  `DELETE /classes/:id/members/:studentId` — **chỉ owner/manager**, teacher **không**.
- Thêm em: `ClassMembership` (`active`, `joinedAt`) + bản ghi tiến độ mỗi giáo trình
  **tạo nếu chưa có**, giữ nguyên tiến độ cũ.
- Loại em: `status=left` + `leftAt`, **không xoá** membership/tiến độ.
- Quay lại: membership `active` **mới**, không ghi đè (R8).

**M10-12 — Gán/loại hàng loạt + ứng viên** (PRD-221)
- `GET /classes/:id/eligible-students` — student của tổ chức **chưa** ở lớp này, `active`,
  có search + lọc cơ sở.
- `POST /classes/:id/members/bulk-add` — kiểm từng em (cùng tổ chức, `student`, chưa
  active trong lớp — **bỏ qua em đã ở lớp, idempotent, không lỗi**; trong `managedBranchIds`
  nếu manager). Ghi **nguyên tử 1 transaction**: membership + tiến độ tạo-nếu-chưa-có.
  Trả **report**: thêm / bỏ qua / lỗi kèm lý do. Giới hạn **200 em/lần**.
- `POST /classes/:id/members/bulk-remove` — `status=left` + `leftAt`, không xoá.

## Ngoài phạm vi
- Điểm danh, buổi học, sổ điểm (Phase sau).
- Gán giáo trình catalogue read (`GET /admin/course-structures`, AD-05) — **chưa có task
  Plane**; đề xuất thêm hoặc gộp (spec §Q).

## Acceptance Criteria
**M10-11**
- [ ] Teacher gọi thêm/loại thành viên → 403.
- [ ] Ra rồi vào lại → 2 bản ghi lịch sử, đúng 1 `active`.
- [ ] Tiến độ tự học không bị ghi đè khi vào lớp.
- [ ] Lớp không kèm `branchId` → 400.

**M10-12**
- [ ] Chọn nhầm em đã ở lớp → bỏ qua, không lỗi (idempotent).
- [ ] Manager gán em ngoài cơ sở mình → chặn.
- [ ] Lô 200 em trong 1 transaction; lỗi giữa chừng → rollback sạch.
- [ ] `eligible-students` không trả em đã `active` trong lớp.

## Câu hỏi mở
1. Đổi route `/:id/students` → `/:id/members` (khớp FE) có phá caller nào khác đang dùng
   `/students` không? (grep exe-web/exe-admin). Nếu có → giữ cả 2 (alias) tạm thời.
2. `GET /admin/course-structures` (AD-05) — thêm task Plane riêng hay gộp M10-11?
