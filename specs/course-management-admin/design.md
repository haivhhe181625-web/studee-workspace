<!-- Tiếng Việt — thiết kế kỹ thuật Giai đoạn A (Admin: xem/sửa/xuất bản khóa). Bám code Phase 1 đã có.
     Nghiệp vụ đã nắm rõ (xem docs/plan-course-to-web.md §4) — tài liệu này chỉ nêu phần cần để code. -->
# Thiết kế: Quản lý khóa (Giai đoạn A — Admin)

- **Ranh giới:** `exe-api` + `exe-admin`. Không đụng exe-web.
- **Quyết định đã chốt:** Q-A1 = không sửa khóa `published` (phải unpublish/clone); Q-A2 = ghi lại **cả cây +
  re-validate**; Q-A3 = cho `published→ready` (gỡ publish) kèm cảnh báo FE.
- **Nguyên tắc:** tái dùng tối đa pipeline validate của Phase 1; admin-api không chứa business logic (delegate service).

## 1. Tái dùng từ Phase 1

| Có sẵn | Dùng lại cho |
|---|---|
| `course-content.service.js` → `validateCourse(course)`, `buildDoc(course, user)` | Re-validate khi sửa/publish; dựng cây |
| `course-content.references.js` → `resolveReferences(course)` | Re-validate tham chiếu ipa/talk |
| `course-content.model.js` → `CourseStructure`, `COURSE_STATUSES` | Ghi/đọc, máy trạng thái |
| `admin-api/resources/course-content.admin.js` (hand-mount) | Thêm `PUT /:id`, `PATCH /:id/status` |
| Guard `status==='published'` (đang ở DELETE) | Cùng ý cho edit-lock |

## 2. Điểm mấu chốt: locator cho cây JSON (không có số dòng Excel)

`validateCourse`/`resolveReferences` chỉ **echo** locator của node vào `message` (trường `row`) — không so sánh số.
⇒ Khi sửa bằng JSON, ta **gắn `path` (chuỗi) vào chính trường `row`** của mỗi node trước khi validate; validator
chạy nguyên xi. Endpoint sau đó **đổi tên `row`→`path`** trong `errors[]` trả về.

```js
// course-content.tree.js (mới) — chuẩn hoá cây JSON → IR để validate/build.
// Gắn locator dạng path + suy `order` từ thứ tự mảng (giống import suy từ thứ tự dòng).
function attachLocators(course) {
  course.row = 'Khóa';
  course.phases.forEach((p, i) => {
    p.order = i + 1; p.row = `Chặng "${p.title}"`;
    p.modules.forEach((m, j) => {
      m.order = j + 1; m.row = `${p.row} › Chuyên đề "${m.title}"`;
      m.lessons.forEach((l, k) => {
        l.order = k + 1; l.row = `${m.row} › Bài "${l.title}"`;
        (l.media || []).forEach((md, x) => { md.order = x + 1; md.row = `${l.row} › media#${x + 1}`; });
        (l.exercises || []).forEach((ex, x) => { ex.order = x + 1; ex.row = `${l.row} › bài tập#${x + 1}`; });
      });
    });
  });
  return course;
}
```

## 3. Service mới (`course-content.service.js` — bổ sung)

Trích một `buildTree(course)` (phần map `phases[]` hiện nằm trong `buildDoc`) để dùng chung. Rồi:

```js
// updateCourse — sửa cả cây (Q-A2). Chỉ draft/ready. Giữ slug/status/importedBy.
async function updateCourse({ id, tree, user }) {
  const doc = await CourseStructure.findById(id);
  if (!doc) throw ApiError(404);
  if (doc.status !== 'draft' && doc.status !== 'ready')
    throw ApiError(409, …, CODES.COURSE_LOCKED);              // Q-A1
  attachLocators(tree);
  const errors = [...validateCourse(tree), ...(await resolveReferences(tree))];
  if (errors.length) throw ApiError(400, …, CODES.IMPORT_VALIDATION_FAILED, mapPath(errors));
  Object.assign(doc, {
    title: tree.title, description: tree.description || '', thumbnail: tree.thumbnail || null,
    phases: buildTree(tree),                                  // slug/status/importedBy KHÔNG đổi
  });
  await doc.save();
  return toSummary(doc);
}

// changeStatus — máy trạng thái (Q-A3). Re-validate khi đích ∈ {ready, published}.
const TRANSITIONS = {
  draft:     ['ready', 'archived'],
  ready:     ['draft', 'published', 'archived'],
  published: ['ready', 'archived'],
  archived:  ['ready'],
};
async function changeStatus({ id, to, user }) {
  const doc = await CourseStructure.findById(id);
  if (!doc) throw ApiError(404);
  if (!TRANSITIONS[doc.status]?.includes(to))
    throw ApiError(409, …, CODES.INVALID_STATUS_TRANSITION);
  if (to === 'ready' || to === 'published') {
    const tree = attachLocators(doc.toObject());
    const errors = [...validateCourse(tree), ...(await resolveReferences(tree))];
    if (errors.length) throw ApiError(400, …, CODES.IMPORT_VALIDATION_FAILED, mapPath(errors));
  }
  doc.status = to; await doc.save();
  return { id: String(doc._id), status: to };
}
```

`mapPath(errors)` = `errors.map(({ row, ...e }) => ({ ...e, path: row }))`.

## 4. Route (`course-content.admin.js` — bổ sung)

Thêm **sau** `DELETE /:id`, giữ HARD RULE (delegate + audit thủ công):

```js
router.put('/:id', verifyToken, verifyPermission(PERMISSIONS.COURSECONTENT_WRITE), asyncHandler(async (req, res) => {
  if (!Types.ObjectId.isValid(req.params.id)) throw new ApiError(HTTP.NOT_FOUND, 'Not found');
  const summary = await courseContentService.updateCourse({ id: req.params.id, tree: req.body, user: req.user });
  auditLog('admin.command.executed', { adminUser: req.user.id, command: 'coursecontent.update', target: summary.slug, ip: req.ip });
  res.status(HTTP.OK).json({ data: { summary } });
}));

router.patch('/:id/status', verifyToken, verifyPermission(PERMISSIONS.COURSECONTENT_WRITE), asyncHandler(async (req, res) => {
  if (!Types.ObjectId.isValid(req.params.id)) throw new ApiError(HTTP.NOT_FOUND, 'Not found');
  const out = await courseContentService.changeStatus({ id: req.params.id, to: req.body.status, user: req.user });
  auditLog('admin.command.executed', { adminUser: req.user.id, command: 'coursecontent.status', target: `${out.id}:${out.status}`, ip: req.ip });
  res.status(HTTP.OK).json({ data: out });
}));
```

> `updateCourse` phải ép `doc._id` không đổi khi `findById(...).save()` (không đổi slug ⇒ không đụng unique index).
> Nếu về sau cần đổi slug thì phải check trùng — **ngoài phạm vi Giai đoạn A**.

## 5. Constants (bổ sung `error-codes.js`)

```js
INVALID_STATUS_TRANSITION: 'INVALID_STATUS_TRANSITION',  // §7 chuyển trạng thái không hợp lệ
```
(`IMPORT_VALIDATION_FAILED`, `COURSE_LOCKED` đã có từ Phase 1.)

## 6. Frontend (`exe-admin`)

Theo convention repo (thin page → container; service + TanStack Query; RBAC display-only). Tái dùng
`course-content.service.ts` + mở rộng.

**Service bổ sung** (`services/course-content.service.ts`):
```ts
export interface CourseTree { title: string; description?: string; thumbnail?: string | null; phases: PhaseNode[]; }
export interface EditError { code: string; message: string; path?: string; }
export async function updateCourse(id: string, tree: CourseTree): Promise<ImportSummary> { /* PUT /:id */ }
export async function changeStatus(id: string, status: string): Promise<{ id: string; status: string }> { /* PATCH /:id/status */ }
// getCourse (đã có) → trả cả cây cho trang detail.
```

**Route + trang mới:**
- `app/(admin)/course-import/[id]/page.tsx` — thin wrapper → container.
- `components/features/course-import/course-detail.tsx` (container):
  - `useQuery(getCourse(id))` → render cây (Chặng→Chuyên đề→Bài học→media/bài tập).
  - **Chế độ sửa** (`coursecontent:write`): form sửa từng node (title/theory/media/exercise, thêm/xoá/di chuyển),
    state cục bộ → `useMutation(updateCourse)`; lỗi → bảng `errors[]` theo `path` (giống bảng lỗi import hiện có).
  - **Điều khiển trạng thái**: nút theo `TRANSITIONS` của `doc.status` hiện tại; `published→ready` hỏi xác nhận
    ("Gỡ xuất bản?"); `→published` disable nếu đang có lỗi.
- **Danh sách** (`course-import.tsx` hiện có): thêm cột/nút **"Xem / Sửa"** link tới `/course-import/[id]`.

**Query key:** thêm `["admin","course-imports", id]` cho detail (giữ style inline như hiện tại), invalidate sau update/status.

## 7. Kiểm thử & bàn giao

- API: unit `updateCourse`/`changeStatus` (validate reuse, guard lock, transition map) + supertest `PUT`/`PATCH`
  (200/400/403/404/409). exe-web KHÔNG bị ảnh hưởng.
- FE: `npm run lint` + `npx tsc --noEmit` (repo khác, không jest) — smoke thủ công khi có exe-api chạy.
- Deploy: exe-api trước exe-admin (như Phase 1).
