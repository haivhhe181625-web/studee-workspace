---
feature: lms-org-branch-staff-import
role: tech-lead → developer
status: draft
date: 2026-08-22
test_runner: cd exe-api/services/api && npx jest <path>
---

# Tasks — org / branch / staff / import (M10-5→8)

Thứ tự: rẻ→đắt, độc lập test được. Mỗi task 1 PR. TDD: viết test đỏ → code → xanh.
Path gốc: `exe-api/services/api/src`. Mount route: `admin-api/_router.js`.

> **Trước khi bắt đầu:** chốt 3 câu hỏi mở (design §Q1/Q2/Q3). Q2 (nguồn tạo Organization)
> chặn phần auto-tạo branch mặc định của Task 5.

---

## Nhóm A — Organization API (M10-5, PRD-214)

- [ ] **A1. Permission strings tổ chức.** Xác nhận `ORGANIZATION_READ/WRITE/MANAGE` có
  trong `constants/permissions.js` + nằm trong bộ `owner` (write) và `manager/teacher`
  (read). Thiếu → thêm, khai đủ 4 action.
  · verify: `npx jest src/__tests__/permissions-tenant.test.js`

- [ ] **A2. `organization.schema.js` — validate branding.** Joi: `logoUrl` https-only,
  `primaryColor` hex, whitelist field PATCH. Test unit schema (accept/reject từng trường).
  · verify: `npx jest src/modules/organization/__tests__/organization.schema.test.js`

- [ ] **A3. `organization.service.js` — getOrganization / updateOrganization.** Chỉ
  whitelist field; đọc/ghi qua model (tenant plugin tự scope). Test: owner update ok,
  field ngoài whitelist bị bỏ.
  · verify: `npx jest src/modules/organization/__tests__/organization.service.test.js`

- [ ] **A4. Route + mount `GET/PATCH /api/admin/organization`.** `GET` = ORGANIZATION_READ,
  `PATCH` = ORGANIZATION_WRITE. Integration test qua chuỗi middleware thật: manager PATCH
  → 403; logoUrl non-https → 400 field-level; owner ok.
  · verify: `npx jest src/modules/organization/__tests__/organization.routes.test.js`

## Nhóm B — Branch API (M10-6, PRD-215)

- [ ] **B1. `assertBranchInScope(caller, branchId)` helper** (dùng chung branch/staff/class).
  owner: luôn true; manager: `branchId ∈ managedBranchIds`; else 403. Test unit.
  · verify: `npx jest src/modules/organization/__tests__/branch-scope.test.js`

- [ ] **B2. `branch.service.js` — list/create/update/softDelete + guard xoá.** Xoá:
  đếm `ClassGroup.status != draft` thuộc branch > 0 → 409. Soft delete `isActive=false`.
  Test: xoá cơ sở còn lớp → chặn; cơ sở trống → isActive=false, bản ghi còn.
  · verify: `npx jest src/modules/organization/__tests__/branch.service.test.js`

- [ ] **B3. Route + mount `/api/admin/branches`** (CRUD). Manager chỉ thấy/sửa branch
  trong scope. Integration: manager thấy branch tổ chức khác → không; xoá guard 409.
  · verify: `npx jest src/modules/organization/__tests__/branch.routes.test.js`

- [ ] **B4. Auto-tạo "Cơ sở chính" khi tạo Organization.** (Phụ thuộc Q2 — nguồn tạo org.)
  Đặt ở service tạo org; `isDefault=true`. Test: tạo org → đúng 1 branch default.
  Nếu Q2 chưa rõ → tách task riêng, đánh dấu blocked.
  · verify: `npx jest src/modules/organization/__tests__/org-default-branch.test.js`

## Nhóm C — Staff API (M10-7, PRD-216)

- [ ] **C1. Mở rộng `assignStaff` — `managedBranchIds`.** Nhận `branchIds[]`; manager
  bắt buộc ≥1; validate branch thuộc org + trong scope caller (B1). Test: tạo manager
  thiếu branch → 400; manager tạo teacher ngoài cơ sở → 403.
  · verify: `npx jest src/__tests__/staff.service.test.js`

- [ ] **C2. `updateStaff` + `PATCH /api/admin/staff/:id`.** Sửa orgRole/branchIds +
  `status=inactive` (set `isActive=false`, giữ orgRole/lịch sử). ASSIGNABLE_ROLES gác
  leo thang (đã có). Integration: manager PATCH tạo manager → 403; inactive giữ bản ghi.
  · verify: `npx jest src/modules/user/__tests__/staff.routes.test.js`

- [ ] **C3. Teacher xuyên cơ sở.** Test: teacher trong `teacherIds` của 2 lớp khác branch
  → "lớp của tôi" (`GET /classes/mine` đã có) trả đủ 2. Chỉ test, không sửa nếu đã đúng.
  · verify: `npx jest src/__tests__/class.test.js`

## Nhóm D — Nhập lô học viên (M10-8, PRD-217) — làm sau cùng

- [ ] **D0. Xác nhận ràng buộc DB `studentCode` unique `(organizationId, studentCode)`**
  (M10-4 §O). Thiếu index → thêm migration + index. Test ghi trùng bị DB chặn.
  · verify: `npx jest src/modules/user/__tests__/student-code-unique.test.js`

- [ ] **D1. Xác nhận hạ tầng transaction test** (replSet cho `mongodb-memory-server`).
  Chạy thử 1 test transaction sẵn có (payment/focus-room) trên máy. Đỏ → sửa cấu hình
  test trước khi làm D3.
  · verify: `npx jest src/modules/focus-room` (hoặc payment)

- [ ] **D2. `student-import.service.js` — validateBatch (không side-effect).** Gom lỗi
  `{row, reason}`: thiếu tên, mã trùng DB, trùng trong lô, mã lớp không tồn tại/không
  thuộc cơ sở. Test từng loại lỗi map đúng row.
  · verify: `npx jest src/modules/student-import/__tests__/validate.test.js`

- [ ] **D3. `commitBatch` — transaction nguyên tử.** `session.withTransaction`: tạo
  student (+ `mustChangePassword`, stub sinh mã/mật khẩu nếu M10-9 chưa xong) + membership
  cùng session (R9). Test: lỗi giữa chừng → rollback sạch, 0 student, 0 membership mồ côi.
  · verify: `npx jest src/modules/student-import/__tests__/commit.test.js`

- [ ] **D4. Routes + mount `/api/admin/students/import/{validate,commit}`.** commit chạy
  lại validate ở đầu, lô bẩn → 400 trước khi mở session. Integration end-to-end.
  · verify: `npx jest src/modules/student-import/__tests__/import.routes.test.js`

---

## Kiểm tra tổng (trước khi xin review)

- [ ] Toàn bộ suite xanh: `cd exe-api/services/api && npm test`
- [ ] Không rải `organizationId` filter thủ công (tenant plugin lo) — grep review.
- [ ] Không business logic trong `admin-api/` (chỉ gọi service) — HARD RULE.
- [ ] Contract path/shape khớp 4 service FE exe-lms (không sửa FE).
- [ ] Mỗi file < 200 dòng.

## Phụ thuộc / thứ tự
```
A (org)  →  độc lập, làm trước (dễ nhất)
B (branch) → B4 chặn bởi Q2
C (staff)  → C1/C2 chặn bởi Q1
D (import) → sau cùng; D0/D1 tiền đề, D chặn một phần bởi M10-9 (Q3) + M10-39 (replSet)
```
