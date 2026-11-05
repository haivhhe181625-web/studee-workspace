---
feature: lms-class-membership-api
role: tech-lead → developer
status: draft
date: 2026-08-22
test_runner: cd exe-api/services/api && npx jest <path>
---

# Tasks — class + membership (M10-11, M10-12)

> Nền tảng class module đã có (M10-4). Đa số là bổ sung + đổi tên route + bulk.
> Bulk-add cần transaction (replSet, M10-39).

## Nhóm A — Đổi tên & xác nhận đơn lẻ (M10-11, PRD-220)

- [ ] **A1. Route `/:id/students` → `/:id/members`** (khớp FE). Kiểm caller khác còn
  dùng `/students` không (grep exe-web/exe-admin); có → giữ alias tạm. Verify Q1 spec.
  · verify: `npx jest src/__tests__/class.test.js`

- [ ] **A2. Xác nhận rule thành viên đơn lẻ** (đã có phần lớn): teacher 403; ra-vào-lại
  2 bản ghi 1 active; tiến độ không ghi đè. Chỉ bổ sung test còn thiếu.
  · verify: `npx jest src/__tests__/class.test.js`

## Nhóm B — Ứng viên + hàng loạt (M10-12, PRD-221)

- [ ] **B1. `GET /classes/:id/eligible-students`.** Student tổ chức, `active`, chưa ở
  lớp; search + lọc cơ sở. Manager giới hạn scope. Test: không trả em đã active.
  · verify: `npx jest src/modules/class/__tests__/eligible-students.test.js`

- [ ] **B2. `bulkAdd` service — transaction + report idempotent.** Kiểm từng em; bỏ qua
  em đã active (không lỗi); membership + tiến độ tạo-nếu-chưa-có cùng session; giới hạn
  200. Test: nhầm em đã ở lớp → skip; manager ngoài scope → chặn; lỗi giữa chừng rollback.
  · verify: `npx jest src/modules/class/__tests__/bulk-add.test.js`

- [ ] **B3. `bulkRemove` service.** `status=left` + `leftAt` cho danh sách, không xoá.
  Test: các em thành `left`, bản ghi còn.
  · verify: `npx jest src/modules/class/__tests__/bulk-remove.test.js`

- [ ] **B4. Routes `POST /:id/members/bulk-add` + `/bulk-remove` + mount.** class:manage.
  Integration khớp FE contract (report shape). End-to-end.
  · verify: `npx jest src/modules/class/__tests__/bulk-routes.test.js`

## Kiểm tra tổng
- [ ] `npm test` xanh · [ ] không lọc organizationId thủ công · [ ] contract khớp FE `class-members.service.ts`.
