<!-- Tiếng Việt — thiết kế Giai đoạn B (API đọc learner). Giai đoạn C (exe-web) sẽ bổ sung sau vào cùng spec. -->
# Thiết kế: Lesson Runtime — Giai đoạn B (API đọc learner)

- **Ranh giới:** `exe-api` (module `course-content`, KHÔNG phải admin-api). Sinh hợp đồng cho exe-web (Giai đoạn C).
- **Quyết định (plan §9):** Q-B1 xem tự do khóa published (chưa enroll) · Q-B2 cần đăng nhập (`verifyToken`) ·
  Q-B3 `centerId` dùng chung (chỉ lọc `status`).
- **Contract:** `contracts/web-course-consume.md`.

## 1. Vị trí & khác biệt với admin-api

| | admin-api (Phase 1 + Giai đoạn A) | learner (Giai đoạn B) |
|---|---|---|
| Đường | `/api/admin/course-imports` | `/api/courses` |
| Auth | `verifyToken` + `verifyPermission(coursecontent:*)` | `verifyToken` (mọi user) |
| Lọc | mọi status | **chỉ `published`** |
| Envelope | `{ data: … }` | `{ courses }` / `{ course }` (khớp `modules/ipa`) |
| Field | đầy đủ | ẩn `importedBy`/`sourceMeta`/`centerId`/`__v` |

## 2. File (mới trong `modules/course-content/`)

```
course-content.public.service.js   listPublishedCourses(), getPublishedCourseBySlug(slug)
course-content.controller.js       req → service → res (envelope learner)
course-content.routes.js           GET / , GET /:slug  (verifyToken)
```
Mount trong `app.js`: `const courseRoutes = require('./modules/course-content/course-content.routes');`
→ `app.use('/api/courses', courseRoutes);` (cạnh `/api/ipa`, `/api/talk`).

## 3. Service (đọc thuần, không side-effect)

```js
// Catalog nhẹ — aggregation để KHÔNG ship cả cây; phaseCount = $size phases.
async function listPublishedCourses() {
  return CourseStructure.aggregate([
    { $match: { status: 'published' } },
    { $sort: { createdAt: -1 } },
    { $project: { _id: 0, id: { $toString: '$_id' }, slug: 1, title: 1, description: 1, thumbnail: 1, phaseCount: { $size: '$phases' } } },
  ]);
}

// Chi tiết cả cây, ẩn field nội bộ. 404 nếu thiếu hoặc chưa publish (không lộ tồn tại).
async function getPublishedCourseBySlug(slug) {
  const doc = await CourseStructure.findOne({ slug, status: 'published' })
    .select('-importedBy -sourceMeta -centerId -__v').lean();
  if (!doc) throw new ApiError(HTTP.NOT_FOUND, 'Không tìm thấy khóa học');
  doc.id = String(doc._id); delete doc._id;
  return doc;
}
```

## 4. Controller + Route

```js
// controller
const listCourses = async (req, res) => res.status(HTTP.OK).json({ courses: await svc.listPublishedCourses() });
const getCourse = async (req, res) => res.status(HTTP.OK).json({ course: await svc.getPublishedCourseBySlug(req.params.slug) });

// routes
router.get('/', verifyToken, asyncHandler(controller.listCourses));
router.get('/:slug', verifyToken, asyncHandler(controller.getCourse));
```

## 5. Kiểm thử (`course-content.public.api.test.js`, supertest + in-memory Mongo)
- `GET /api/courses`: 200 chỉ liệt kê khóa `published` (bỏ draft/ready/archived); có `phaseCount`, **không** có `phases`.
- `GET /api/courses/:slug`: 200 cả cây khi published; **ẩn** `importedBy`/`sourceMeta`/`centerId`. 404 khi chưa
  publish / slug sai. 401 khi thiếu token.

## 6. Ngoài phạm vi B (→ Phase 3): enroll, tiến độ, chấm điểm, unlock theo điểm.

---

# Thiết kế: Lesson Runtime — Giai đoạn C (exe-web hiển thị & học)

- **Ranh giới:** `exe-web` (tiêu thụ hợp đồng B) + **1 endpoint đọc nhỏ ở `exe-api` module ipa** (resolver §B3).
- **Quyết định (plan §9):** Q-C1 = **mở toàn bộ cây** (chưa unlock tuần tự; unlock gắn điểm → Phase 3).

## C1. Điểm tích hợp engine (rủi ro duy nhất) — ipa refId

`exercises[].refId` (ipa) = IpaLesson **`code`**; runner exe-web chạy theo **`_id`**. ⇒ thêm resolver
`GET /api/ipa/lessons/by-code/:code → { id }` (module ipa, read-only, published-only). Talk khớp sẵn:
`SCENARIO_IDS` = `TALK_SCENARIOS[].id` (`coffee/interview/travel/freetalk`) ⇒ mount `ConversationView` trực tiếp.

## C2. Convention exe-web (bám `docs/` + feature ipa làm mẫu)

- **page.tsx mỏng** → container; container bọc `<AppShell title=…>` (tự guard auth, redirect LOGIN nếu chưa đăng nhập).
- **service** (`services/course.service.ts`): axios thuần, giải nén key (`{courses}` / `{course}`) — KHÔNG envelope `{data}`.
- **hook** (`hooks/use-courses.ts`): TanStack Query, `QUERY_KEYS.courses.*`, `enabled: isAuthenticated`.
- **media**: `utils/media.mediaUrl()` cho mọi `media[].url`.
- **markdown lý thuyết**: thêm `react-markdown` + `remark-gfm` (chưa có sẵn) — wrapper `MarkdownContent` styled bằng Tailwind.

## C3. Cây file exe-web (mới, feature `courses`)

```
constants/routes.ts        + COURSES, COURSE_DETAIL(slug), COURSE_LESSON(slug,key)
constants/query-keys.ts    + courses: { all, list, detail(slug) }
types/course.types.ts      CourseSummary, CourseDetail, CoursePhase/Module/Lesson/Media/Exercise
services/course.service.ts getCourses(), getCourseBySlug(slug)
services/ipa.service.ts    + resolveIpaLessonIdByCode(code)            (§B3)
hooks/use-courses.ts       useCourses(), useCourseDetail(slug)
components/layouts/nav-items.ts  + "Khóa học" (group learn)
components/features/courses/
  CourseCatalogContainer.tsx  catalog grid (AppShell)
  CourseCard.tsx              thẻ khóa → /courses/:slug
  CourseDetailContainer.tsx   cây Chặng→Chuyên đề→Bài; link Bài
  LessonViewer.tsx            lý thuyết(md) + media + mở ipa/talk
  MarkdownContent.tsx         react-markdown wrapper
  index.ts
app/(main)/courses/page.tsx
app/(main)/courses/[slug]/page.tsx
app/(main)/courses/[slug]/lessons/[lessonKey]/page.tsx
```

- **Lesson viewer** không có endpoint per-lesson (hợp đồng B chỉ có catalog + cả cây) → trang Bài dùng
  `useCourseDetail(slug)` (cache dùng lại từ trang chi tiết) rồi tìm lesson theo `key`.
- **Mở bài tập:** `ipa` → resolve code→id → `router.push(ROUTES.PRONUNCIATION_LESSON(id))`; `talk` → state
  `activeTalk = TALK_SCENARIOS.find(id===refId)` → render `<ConversationView>` inline (`onBack` đóng).

## C4. Ngoài phạm vi C (→ Phase 3/4): tiến độ %, mastery, unlock theo điểm, engine Nghe/Đọc/Viết/quiz.
