<!-- Tiếng Việt — task breakdown Giai đoạn A (Admin: xem/sửa/xuất bản khóa). Thứ tự 1→5, mỗi task test được +
     commit riêng. Bám design.md. Lệnh API chạy trong exe-api/services/api (cd services/api). -->
# Tasks: Quản lý khóa (Giai đoạn A — Admin)

**Design:** `specs/course-management-admin/design.md` · **Contract:** `contracts/admin-course-management.md`
**Tiền đề:** Phase 1 (`course-content`) đã merge. Nhánh mới `feature/course-management-admin` (base `develop`).
**Quy ước:** API test đặt ở `src/__tests__/`, chạy `npx jest <pattern> -i`. `git add` từng file, KHÔNG `git add .`.

---

## Task 1 — Error code + helper chuẩn hoá cây (`attachLocators` + `buildTree`) — ✅ Xong (`e309962`)

> Test `course-content.tree.test.js` xanh; full suite course-content **50/50** (không hồi quy import).
> `buildTree` tách khỏi `buildDoc` (import không đổi hành vi). Nhánh `feature/course-management-admin` (từ HEAD Phase 1).

**Files:**
- Modify: `src/constants/error-codes.js` — thêm `INVALID_STATUS_TRANSITION: 'INVALID_STATUS_TRANSITION'`.
- Create: `src/modules/course-content/course-content.tree.js` — `attachLocators(course)` (design §2).
- Modify: `src/modules/course-content/course-content.service.js` — trích `buildTree(course)` từ `buildDoc` (phần
  map `phases[]`), export thêm `buildTree`; `buildDoc` gọi lại `buildTree` (không đổi hành vi import).
- Test: `src/__tests__/course-content.tree.test.js`

**Test (đỏ→xanh):**
- `attachLocators`: cây 2 Chặng → mỗi node có `row` dạng path (`Chặng "…" › Chuyên đề "…" › Bài "…"`) và `order`
  suy từ index (1-based).
- `buildTree`: cây IR → mảng phases nhúng đúng (giữ `buildDoc` cũ pass — chạy lại `course-content.service` cũ).

**Commit:** `chore(course-content): status transition error code + tree normalize/build helpers`

---

## Task 2 — Service `updateCourse` + `changeStatus` — ✅ Xong (`eba88ba`)

> `course-content.manage.service.test.js` **11/11**; full module **8 suites / 61 test** không hồi quy.
> `updateCourse` (guard lock + re-validate + slug/status/importedBy bất biến); `changeStatus` (TRANSITIONS +
> re-validate khi đích ∈ {ready,published}). exe-api không có lint script (jest là cổng, như Phase 1).

**Files:**
- Modify: `src/modules/course-content/course-content.service.js` — thêm `updateCourse`, `changeStatus`,
  `TRANSITIONS`, `mapPath`; export tất cả (design §3).
- Test: `src/__tests__/course-content.manage.service.test.js` (in-memory Mongo).

**Test `updateCourse`:**
- draft/ready → sửa title + thêm Bài học hợp lệ → doc cập nhật, `slug`/`status`/`importedBy` **không đổi**.
- cây lỗi (vd EMPTY_LESSON / IPA chưa publish) → throw `IMPORT_VALIDATION_FAILED`, `errors[].path` có chuỗi path
  (không có `row`), doc **không đổi**.
- status `published` → throw `COURSE_LOCKED`, không ghi.

**Test `changeStatus`:**
- `ready→published` khi cây sạch (IPA published) → `status='published'`.
- `ready→published` khi IPA ref bị unpublish → `IMPORT_VALIDATION_FAILED` (re-validate chặn), giữ `ready`.
- `draft→published` (không nằm trong TRANSITIONS) → `INVALID_STATUS_TRANSITION`.
- `published→ready` (gỡ publish) hợp lệ → `status='ready'`.
- `published→archived`, `archived→ready` hợp lệ.

**Commit:** `feat(course-content): updateCourse (edit whole tree) + changeStatus (lifecycle) services`

---

## Task 3 — Route `PUT /:id` + `PATCH /:id/status` + API test — ✅ Xong (`b641b3f`)

> `course-content.manage.api.test.js` **10/10** (PUT 5 + PATCH 5); full module **9 suites / 71 test**.
> Route thêm sau `DELETE /:id`; audit `coursecontent.update` / `coursecontent.status`. **Backend Giai đoạn A xong** — còn Task 4 (exe-admin UI) + Task 5 (self-review).

**Files:**
- Modify: `src/admin-api/resources/course-content.admin.js` — thêm 2 route sau `DELETE /:id` (design §4).
- Test: `src/__tests__/course-content.manage.api.test.js` (supertest, mock `audit`).

**Test (theo contract §6/§7):**
- `PUT /:id` — 200 + summary khi hợp lệ; 400 + `errors[].path` khi cây lỗi; 409 `COURSE_LOCKED` khi published;
  403 thiếu `coursecontent:write`; 404 id sai.
- `PATCH /:id/status` — 200 khi transition hợp lệ; 409 `INVALID_STATUS_TRANSITION` khi sai; 400 khi re-validate
  publish thất bại; 403/404.
- Xác nhận `auditLog` gọi với `command: 'coursecontent.update'` / `'coursecontent.status'`.

**Commit:** `feat(admin): PUT /course-imports/:id (edit) + PATCH /:id/status (lifecycle)`

---

## Task 4 — exe-admin: service + trang detail (xem/sửa/xuất bản) — ✅ Xong (`9541c19`, repo exe-admin)

> Nhánh `feature/course-management-admin` (exe-admin). `npx tsc --noEmit` exit 0 + `npm run lint` clean.
> Service +`updateCourse`/`changeStatus` + types cây; trang `course-import/[id]` (page async await params →
> container `course-detail.tsx`: xem cây, sửa từng node + thêm/xoá/di chuyển, cụm nút trạng thái, bảng lỗi theo
> `path`); danh sách thêm link "Xem / Sửa". **Smoke tương tác chưa chạy** (cần exe-api + DB + tài khoản
> `coursecontent:*`) — dev chạy tay trước merge.

> Repo `exe-admin` (Next.js, không jest). Nhánh `feature/course-management-admin` (base `develop`).
> Verify: `npm run lint` + `npx tsc --noEmit`; smoke thủ công khi có exe-api.

**Files:**
- Modify: `src/services/course-content.service.ts` — thêm `updateCourse`, `changeStatus`, types `CourseTree`/`EditError`.
- Create: `src/app/(admin)/course-import/[id]/page.tsx` — thin wrapper → container.
- Create: `src/components/features/course-import/course-detail.tsx` — container (design §6):
  - `getCourse(id)` → render cây read-only.
  - Chế độ sửa (`RequirePermission perm="coursecontent:write"`): form từng node + thêm/xoá/di chuyển; lưu bằng
    `updateCourse`; bảng lỗi theo `path`.
  - Cụm nút trạng thái theo `TRANSITIONS[status]`; `published→ready` confirm; `→published` chặn nếu đang lỗi.
- Modify: `src/components/features/course-import/course-import.tsx` — thêm link **"Xem / Sửa"** mỗi hàng →
  `/course-import/[id]`.

**Verify:** `cd exe-admin && npm run lint && npx tsc --noEmit` (đều clean/exit 0).

**Commit (repo exe-admin):** `feat(course-import): course detail page — view, edit tree, publish lifecycle`

---

## Task 5 — Full suite + Self-Review — ✅ Xong

- `npx jest course-content -i` → **9 suites / 71 test pass**.
- `npm test` (full) → **872 pass / 2 fail / 874 total** (184 s). 2 fail = `avatar-storage.test.js` (`EPERM` dọn
  temp Windows) — **y hệt Phase 1, không phải hồi quy** (diff không đụng `modules/media`/avatar). Course-content: 0 fail.

### Self-Review — đối chiếu contract

**§6 `PUT /:id`** (sửa cả cây): 200 + summary · 400 `IMPORT_VALIDATION_FAILED` + `errors[].path` (không `row`) ·
409 `COURSE_LOCKED` (published/archived) · 403 thiếu `coursecontent:write` · 404. → Task 2 service + Task 3 API test. ✅
**§7 `PATCH /:id/status`** (vòng đời): 200 · 409 `INVALID_STATUS_TRANSITION` (ngoài bảng) · 400 khi re-validate
publish thất bại (IPA unpublish) · 403 · 404; audit `coursecontent.status`. → Task 2 + Task 3. ✅

**Quyết định đã chốt:** Q-A1 (published không sửa trực tiếp → `COURSE_LOCKED`; FE khoá input) ✅ · Q-A2 (PUT cả
cây, tái dùng `validateCourse`+`resolveReferences` qua `attachLocators`, `buildTree`, ghi atomic `doc.save`) ✅ ·
Q-A3 (`published→ready` hợp lệ + FE confirm; `draft→published` chặn) ✅.

**Bất biến:** `slug`/`status`/`importedBy` không đổi khi sửa (test khẳng định). **Không thêm permission** mới
(`coursecontent:*` từ Phase 1 đủ). **Không đổi data-model** (chỉ hành vi).

**Deviation:** không có ngoài dự kiến. FE smoke tương tác **chưa chạy** (cần exe-api + DB + tài khoản
`coursecontent:*`) — thay bằng `tsc --noEmit` (exit 0) + `lint` (clean); dev chạy checklist tay trước merge.

### Bàn giao / Deploy
- **Nhánh:** `exe-api` `feature/course-management-admin` — 3 commit (`e309962`→`eba88ba`→`b641b3f`, base = HEAD
  Phase 1 vì Phase 1 chưa lên `develop`). `exe-admin` `feature/course-management-admin` — 1 commit (`9541c19`).
  Chưa push/PR. `studee-workspace` (spec + progress) chưa commit — chờ dev.
- **Thứ tự deploy:** `exe-api` trước `exe-admin`. Không cần seed permission mới.
- **Smoke checklist (dev):** mở `/course-import` → "Xem / Sửa" 1 khóa `ready` → sửa tên/lý thuyết, thêm/xoá node →
  Lưu (thấy summary) → "Xuất bản" (status→published, input khoá) → "Gỡ xuất bản" (confirm) → sửa lại. Thử lưu cây
  lỗi (Bài học rỗng / IPA sai) → bảng lỗi theo `path`, không ghi.

---

## Ghi chú phạm vi (khỏi làm ở Giai đoạn A)
- **Không** đổi `slug` khi sửa (slug bất biến = định danh); đổi slug ⇒ Phase 5.
- **Không** sửa khóa `published` trực tiếp (phải `published→ready` trước) — Q-A1.
- **Không** versioning/lịch sử sửa — Phase 5.
- **Không** đụng exe-web (đó là Giai đoạn B/C — xem `docs/plan-course-to-web.md`).
