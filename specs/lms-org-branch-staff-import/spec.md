---
feature: lms-org-branch-staff-import
module: M10 · LMS Phase 1 — Vận hành & Quản lý
plane_epic: PRD-209
plane_tasks: [PRD-214 (M10-5), PRD-215 (M10-6), PRD-216 (M10-7), PRD-217 (M10-8)]
repo: exe-api (services/api)
status: draft
tier: lite
date: 2026-08-22
---

# Spec — Resource API tổ chức / cơ sở / nhân sự / nhập lô học viên

## Mục tiêu

Mở 4 nhóm REST resource đầu tiên của portal `exe-lms` trên bề mặt `/api/admin`, để
owner/manager vận hành được tổ chức mà không phá quy tắc phân quyền. Đây là phần
**mở khoá các màn FE đã hoàn thành** (M10-21/22/23/24 đều `done`, đang gọi API rỗng).

## Bối cảnh — cái gì đã có, cái gì thiếu

Nền tảng M10-1→4 **đã xong và test xanh** trên `develop` (tenant plugin, chuỗi
middleware, 3 chốt chặn, 5 bộ quyền + `ASSIGNABLE_ROLES`, models `Organization` /
`Branch` / `ClassGroup` / `ClassMembership`). Spec này **không đụng** nền tảng đó, chỉ
thêm tầng route/service resource lên trên.

| Resource | Model | Route/Service | Gap |
|---|---|---|---|
| Organization | ✅ có (`branding` embed) | ❌ chưa mount | Route + service + validate branding |
| Branch | ✅ có (`isDefault`, `isActive`, unique `(org,name)`) | ❌ chưa mount | Route + service + auto-tạo cơ sở chính + guard xoá |
| Staff | ✅ User fields | ⚠️ có `GET/POST/DELETE /admin/staff` + `staff.service` | Thiếu `PATCH`, `status=inactive`, gán `managedBranchIds`, phạm vi manager |
| Student import | ✅ User + ClassMembership | ❌ không có module | Toàn bộ: validate + commit transaction |

## Phạm vi (in scope)

**M10-5 — Organization API** (`PRD-214`)
- `GET /api/admin/organization` — mọi staff đọc hồ sơ tổ chức mình.
- `PATCH /api/admin/organization` — **chỉ owner**; sửa `name`, thông tin liên hệ,
  `branding.logoUrl`, `branding.primaryColor`, `branding.displayName`.
- Backend **validate trước khi lưu**: `logoUrl` phải là URL `https://`; `primaryColor`
  phải là mã hex hợp lệ (`#RGB` / `#RRGGBB`). Lỗi trả về **theo từng trường**.

**M10-6 — Branch API** (`PRD-215`)
- `GET/POST/PATCH/DELETE /api/admin/branches` — owner toàn quyền; manager chỉ
  thấy/thao tác cơ sở trong `managedBranchIds`.
- Tạo tổ chức → tự sinh **1 cơ sở mặc định** ("Cơ sở chính", `isDefault=true`).
- **Chặn xoá** cơ sở còn lớp đang hoạt động (`ClassGroup.status != 'draft'` thuộc
  branch), báo lý do rõ. Xoá = soft (`isActive=false`), không xoá cứng.

**M10-7 — Staff API** (`PRD-216`)
- `GET/POST/PATCH /api/admin/staff` (giữ `GET` sẵn có làm mẫu).
- Owner tạo **manager** (email bắt buộc, gán ≥1 cơ sở → `managedBranchIds`); manager
  **không** thấy chức năng tạo manager.
- Owner/manager tạo **teacher** (email bắt buộc); manager chỉ tạo trong cơ sở mình.
- `PATCH` sửa vai trò/phạm vi cơ sở + **vô hiệu hoá** (`status=inactive`), không xoá cứng.
- Đặt lại mật khẩu: dùng luồng `forgot/reset` email **sẵn có**, không làm mới.
- Teacher dạy nhiều lớp kể cả xuyên cơ sở (`teacherIds[]`) — phạm vi teacher = hợp
  các lớp có tên mình.

**M10-8 — Nhập lô học viên** (`PRD-217`)
- `POST /api/admin/students/import/validate` — kiểm tra toàn bộ, **không tạo gì**;
  trả `{ validCount, errors: [{ row, reason }] }`. Bắt: thiếu tên, mã trùng trong tổ
  chức, trùng trong lô, sai định dạng, mã lớp không tồn tại / không thuộc cơ sở mình.
- `POST /api/admin/students/import/commit` — chỉ chạy khi lô sạch; tạo tài khoản
  **nguyên tử trong 1 transaction**; có cột `mã lớp` → tạo `ClassMembership` **cùng
  transaction** (R9). Đứt giữa chừng → rollback sạch, không học sinh mồ côi membership.
- Cột nhận: `họ tên` (bắt buộc), `mã học viên`, `ngày sinh`, `mã lớp`, `ghi chú`.

## Ngoài phạm vi (out of scope, có lý do)

- **Sinh `studentCode`/mật khẩu ngẫu nhiên + file tải-một-lần** → thuộc M10-9 (PRD-218).
  M10-8 chỉ lo validate→commit nguyên tử; cơ chế sinh mã/mật khẩu là dependency (xem
  design.md §Phụ thuộc). Nếu M10-9 chưa xong, commit dùng stub sinh mã tạm — chốt ở design.
- **Class API + thao tác thành viên hàng loạt** → M10-11/M10-12.
- **Parent, student self-service, OTP** → M10-13→16.
- **Rename `Center→Organization` phần còn sót ở module test-paper** → task riêng, không
  chặn M10.
- **MongoDB replica-set** cho transaction (M10-8) → DevOps M10-39, là tiền đề hạ tầng.

## Acceptance Criteria

**Organization (M10-5)**
- [ ] Manager/teacher gọi `PATCH /admin/organization` → `403`.
- [ ] Owner PATCH `logoUrl` không phải `https` → `400` với lỗi trỏ đúng trường `logoUrl`.
- [ ] Owner PATCH `primaryColor` sai mã hex → `400` với lỗi trỏ trường `primaryColor`.
- [ ] Owner PATCH hợp lệ → lưu, `GET` trả giá trị mới; tenant khác không đọc được (plugin).

**Branch (M10-6)**
- [ ] Tạo tổ chức mới → tồn tại đúng 1 branch `isDefault=true`.
- [ ] Xoá cơ sở còn lớp `status != draft` → `409`/`400` báo lý do, không xoá.
- [ ] Manager chỉ thấy/sửa branch trong `managedBranchIds`; branch tổ chức khác → không thấy.
- [ ] Xoá cơ sở trống → `isActive=false`, bản ghi vẫn còn.

**Staff (M10-7)**
- [ ] Manager `POST /admin/staff` tạo `manager` → `403`.
- [ ] Owner tạo manager không kèm cơ sở → `400`.
- [ ] Manager tạo teacher ngoài cơ sở mình → `403`.
- [ ] `PATCH status=inactive` → user không đăng nhập/không mất bản ghi (giữ lịch sử).
- [ ] Teacher trong `teacherIds` của 2 lớp khác cơ sở → truy vấn "lớp của tôi" thấy đủ 2.

**Import (M10-8)**
- [ ] Lô có ≥1 dòng lỗi → `validate` liệt kê đúng `row`+`reason`; `commit` từ chối.
- [ ] `commit` lô sạch có cột mã lớp → student + ClassMembership tạo cùng lúc.
- [ ] Mô phỏng lỗi giữa commit → rollback, không student nào tồn tại, không membership mồ côi.
- [ ] Mã học viên trùng trong tổ chức hoặc trùng trong lô → báo lỗi ở `validate`.

## Repo/surface bị ảnh hưởng

- `exe-api/services/api` — toàn bộ thay đổi backend.
- `exe-lms` — **không sửa**; FE đã gọi sẵn các endpoint này (chỉ cần BE khớp path/shape).
- Contract path/shape phải khớp service FE: `organization.service.ts`, `branch.service.ts`,
  `staff.service.ts`, `student-import.service.ts` (xem design.md §Contract).

## Câu hỏi mở

1. **M10-7 tạo staff:** `assignStaff` hiện **yêu cầu user đã đăng ký** (tìm theo email,
   404 nếu chưa có). M10-7 nói "email bắt buộc" + "reset qua email sẵn có" → ngụ ý
   **tạo tài khoản + mời qua email**. Chọn: (a) giữ yêu-cầu-đăng-ký-trước, hay (b) tạo
   tài khoản staff mới + gửi mail đặt mật khẩu? → chốt ở Design.
2. **M10-8 phụ thuộc M10-9:** commit cần `studentCode` + mật khẩu sinh sẵn. Làm M10-8
   trước M10-9 thì cần stub sinh mã/mật khẩu tạm. Có làm M10-9 gộp luôn không?
3. **Transaction:** local/dev Mongo đã là replica-set chưa (M10-39)? Nếu chưa,
   `commit` transaction sẽ lỗi khi chạy thật.
