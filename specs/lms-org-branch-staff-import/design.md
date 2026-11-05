---
feature: lms-org-branch-staff-import
role: tech-lead
status: draft
date: 2026-08-22
---

# Design — Resource API org / branch / staff / import

> Chỉ ghi phần **không hiển nhiên**. CRUD thuần theo mẫu `staff` sẵn có không lặp lại ở đây.
> Mẫu chuẩn để nhân: hand-written router trong `admin-api/_router.js` (staff) +
> module service `modules/<feature>/`. KHÔNG dùng `_crud.factory` vì mỗi resource có
> rule nghiệp vụ (validate branding, guard xoá, transaction).

## 0. Nguyên tắc bám nền tảng đã có

- Route mount dưới `admin-api/_router.js` → tự thừa hưởng
  `verifyToken → loadCaller → tenantContext → denyLearnerTier`. **Không tự thêm lại.**
- Mọi query model đã áp `tenantPlugin` → **không tự lọc `organizationId` bằng tay**.
  Chỉ cần `verifyPermission` + rule branch-scope cho manager.
- Business logic ở `modules/<feature>/*.service.js`; `admin-api` chỉ gọi service
  (HARD RULE `exe-api/CLAUDE.md` §Admin Rules).

## 1. Branch (M10-6) — 2 điểm cần thiết kế

**Auto-tạo "Cơ sở chính".** Đặt ở **service tạo Organization**, không ở route branch.
Vì Organization được tạo ở luồng nào (staff onboarding / seed / admin), branch mặc định
phải luôn đi kèm. Dùng transaction org+branch nếu replica-set sẵn; nếu chưa, tạo tuần tự
org→branch và chấp nhận rủi ro nhỏ (org không branch) — thêm hàm dò lại (reconcile) tương
tự `class reconcileEnrollments`. **Chốt:** ai gọi tạo Organization hiện tại? (khảo sát:
chưa có route tạo org trong M10-5 scope — org tạo ở đâu? → xem Câu hỏi mở §Q).

**Guard xoá.** Trước khi soft-delete branch: `ClassGroup.countDocuments({ branchId,
status: { $ne: 'draft' } })`. > 0 → `ApiError(409, 'Cơ sở còn N lớp đang hoạt động…')`.
Đếm chạy qua tenant plugin nên tự giới hạn trong tổ chức.

**Manager scope.** Manager thao tác branch: chỉ khi `branchId ∈ caller.managedBranchIds`.
Viết 1 helper `assertBranchInScope(caller, branchId)` dùng lại cho cả branch/staff/class.

## 2. Organization (M10-5) — validate branding

Validate ở **schema Joi** (`organization.schema.js`), không rải trong controller:
- `logoUrl`: `Joi.string().uri({ scheme: ['https'] })` — chỉ https.
- `primaryColor`: `Joi.string().pattern(/^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$/)`.
- `PATCH` chỉ owner: `verifyPermission(PERMISSIONS.ORGANIZATION_WRITE)` (kiểm tra
  string này đã có trong `constants/permissions.js`; nếu thiếu → thêm ở M10-5, khai đủ
  4 action theo §4 architecture).
- Chỉ nhận whitelist field (`name`, `contact*`, `branding.*`) — không cho set
  `slug`/`proctorThresholds`/`retention` từ portal khách.

`GET` cho mọi staff (`ORGANIZATION_READ`). Manager/teacher đọc được, chỉ chặn `PATCH`.

## 3. Staff (M10-7) — mở rộng service sẵn có, không viết lại

`staff.service.js` đã có `assignStaff`/`listStaff`/`removeStaff` + `ASSIGNABLE_ROLES`.
Cần thêm:
- **`managedBranchIds`**: `assignStaff` nhận thêm `branchIds[]`; nếu `orgRole=manager`
  bắt buộc ≥1; nếu `teacher` gán `branchId` đơn (hoặc theo lớp). Set field, validate
  branch thuộc tổ chức + (nếu caller là manager) thuộc scope caller.
- **`updateStaff(caller, userId, {orgRole?, branchIds?, status?})`** mới → phục vụ `PATCH`.
  `status=inactive` set `user.isActive=false` (đăng nhập đã chặn `isActive` ở
  `loadCaller`) — **khác** `removeStaff` (xoá orgRole). Giữ orgRole để còn lịch sử.
- **Manager tạo manager** → chặn bằng `ASSIGNABLE_ROLES.manager = ['teacher','student',
  'parent']` (đã đúng sẵn). Không cần thêm code.

**Quyết định Q1 (create-vs-invite).** Đề xuất: **giữ yêu-cầu-đăng-ký-trước** (assignStaff
hiện tại) cho Phase 1 — đơn giản, đúng KISS; owner nhập email người đã có tài khoản
Studee. "Tạo mới + mời email" là mở rộng sau nếu khách cần. → **Cần user chốt** (spec §Q1).

## 4. Import (M10-8) — validate/commit tách đôi, commit nguyên tử

Module mới `modules/student-import/` (5-file pattern). Không nhét vào `admin-api`.

**Validate (không side-effect):**
- Parse rows (FE gửi JSON array đã parse từ Excel — xlsx nằm ở FE `student-import.service.ts`;
  BE nhận `{ rows: [...], defaultBranchId? }`).
- Kiểm từng dòng, gom lỗi `{ row, reason }`. Các loại: thiếu `họ tên`; `mã học viên`
  trùng DB (`User.exists({ organizationId, studentCode })`) hoặc trùng trong lô (Set);
  `mã lớp` không tồn tại / không thuộc cơ sở caller (`ClassGroup` lookup by code + branch).
- Trả `{ validCount, errors }`. **Không tạo gì.**

**Commit (nguyên tử):**
```
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  for (row of rows) {
    user = createStudentUser(row, { organizationId, enrollmentBranchId }, session)
    if (row.classCode) createMembership(user, class, session)   // R9: cùng transaction
  }
});
```
- Chạy lại `validate` ở đầu commit (double-check) → lô bẩn thì abort trước khi mở session.
- `session.withTransaction` tự rollback khi throw → không student mồ côi (AC).
- **Phụ thuộc M10-9**: `studentCode` (nếu dòng bỏ trống → sinh tự động) + mật khẩu ngẫu
  nhiên + `mustChangePassword=true`. Nếu M10-9 chưa xong: viết helper `generateStudentCode`
  /`generateTempPassword` tối giản tại đây, M10-9 thay sau. **Không** để commit phụ thuộc
  code chưa tồn tại.
- **Ràng buộc DB đỡ lưng**: `studentCode` unique `(organizationId, studentCode)` phải có
  index (M10-4 §O — kiểm tra, thiếu thì thêm) → transaction vẫn an toàn khi 2 lô chạy đua.

**Tiền đề hạ tầng:** `withTransaction` cần MongoDB replica-set (M10-39). Test phải chạy
trên `mongodb-memory-server` cấu hình replSet (một số suite hiện có đã dùng transaction —
`payment`, `focus-room` — nên hạ tầng test có thể đã sẵn; xác nhận khi viết test).

## 5. Contract — khớp shape FE (không đổi FE)

Route đọc từ service FE exe-lms; giữ nguyên để FE không phải sửa:

| Endpoint | FE service | Response bọc |
|---|---|---|
| `GET/PATCH /admin/organization` | `organization.service.ts` | `{ data: {...} }` |
| `GET/POST/PATCH/DELETE /admin/branches` | `branch.service.ts` | `{ data: {...} }` / `{ data: { items } }` |
| `PATCH /admin/staff/:id` | `staff.service.ts` | `{ data: {...} }` |
| `POST /admin/students/import/{validate,commit}` | `student-import.service.ts` | `{ data: {...} }` |

Mọi response qua `toDto()`/transformer (không trả thẳng doc Mongoose).

## 6. Modularization

- `modules/organization/organization.service.js` + `.schema.js` + `.controller.js` + `.routes.js` (mount `/organization`).
- `modules/organization/branch.service.js` + branch routes (mount `/branches`) — dùng chung model dir organization.
- `modules/user/staff.service.js` — mở rộng, **không** file mới.
- `modules/student-import/` — module mới đủ 5 file.
- Mỗi file < 200 dòng (rule repo). Route mount thêm vào `admin-api/_router.js`.

## Câu hỏi mở (chuyển tiếp cho Task Gen / user)

- **Q1** create-vs-invite staff (đề xuất giữ đăng-ký-trước) — cần user chốt.
- **Q2** Org được tạo ở luồng nào hiện tại? M10-5 chỉ có GET/PATCH, **không có POST tạo
  org** → auto-tạo branch mặc định phải đặt đúng chỗ đó. Cần xác định nguồn tạo org
  (seed/admin-platform?) trước khi gắn hook. Nếu ngoài scope M10-5→8 → tách task.
- **Q3** M10-8 có gộp M10-9 (sinh mã/mật khẩu + file tải-một-lần) không, hay stub tạm?
