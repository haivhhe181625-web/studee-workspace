<!-- Tiếng Việt — tasks Giai đoạn B (API đọc learner). Giai đoạn C (exe-web) bổ sung sau. -->
# Tasks: Lesson Runtime — Giai đoạn B (API đọc learner)

**Design:** `design.md` · **Contract:** `contracts/web-course-consume.md`
**Nhánh:** tiếp tục `feature/course-management-admin` (course-content) hoặc nhánh mới `feature/lesson-runtime-api`.

---

## Task B1 — Learner read API (`/api/courses`) — ✅ Xong (`21f3a5b`)

> `course-content.public.api.test.js` **6/6**; full module course-content **10 suites / 77 test**.
> `course-content.public.service.js` (aggregate catalog + detail ẩn field nội bộ) + controller + routes +
> mount `app.use('/api/courses')`. Nhánh `feature/lesson-runtime-api` (từ HEAD Giai đoạn A).

**Files:**
- Create: `services/api/src/modules/course-content/course-content.public.service.js` — `listPublishedCourses`, `getPublishedCourseBySlug` (design §3).
- Create: `services/api/src/modules/course-content/course-content.controller.js` — envelope learner (design §4).
- Create: `services/api/src/modules/course-content/course-content.routes.js` — `GET /`, `GET /:slug` (`verifyToken`).
- Modify: `services/api/src/app.js` — mount `app.use('/api/courses', courseRoutes)`.
- Test: `services/api/src/__tests__/course-content.public.api.test.js`.

**Test (contract §B1/§B2):**
- `GET /api/courses` → 200, chỉ khóa `published` (seed thêm draft/ready/archived để chắc bị loại); item có
  `phaseCount`, không có `phases`; 401 khi thiếu token.
- `GET /api/courses/:slug` → 200 cả cây khi published; không có `importedBy`/`sourceMeta`/`centerId`; 404 khi khóa
  chưa publish; 404 khi slug sai; 401 thiếu token.

**Commit:** `feat(course-content): learner read API GET /api/courses (published catalog + detail)`

---

## Task B2 — Full suite + self-review — ✅ Xong
- `npx jest course-content -i` → **10 suites / 77 pass**. `npm test` (full) → **878 pass / 2 fail / 880**; 2 fail =
  `avatar-storage` EPERM (Windows), không hồi quy (như Phase 1 / Giai đoạn A).
- **Self-review:** §B1 (catalog chỉ published, `phaseCount`, không ship cây, 401) ✅ · §B2 (detail cả cây, ẩn
  `importedBy`/`sourceMeta`/`centerId`, 404 khi chưa publish/slug sai, 401) ✅. Learner surface tách hẳn admin-api
  (service riêng, envelope `{courses}`/`{course}`, chỉ đọc). Q-B1/B2/B3 đúng đề xuất.
- **Hợp đồng `web-course-consume.md` KÝ** → sẵn sàng cho Giai đoạn C (exe-web).
- **Bàn giao:** nhánh `exe-api` `feature/lesson-runtime-api` — 1 commit `21f3a5b` (base = HEAD Giai đoạn A). Chưa push/PR.

---

# Tasks: Giai đoạn C — exe-web hiển thị & học

**Design:** `design.md` §C · **Contract:** `contracts/web-course-consume.md` (§B1/§B2 + §B3 resolver).
**Nhánh:** `exe-api` tiếp `feature/lesson-runtime-api` (chỉ C1) · `exe-web` `feature/lesson-runtime-web` (base `develop`).

## Task C1 — Resolver ipa code→id (exe-api)
- `GET /api/ipa/lessons/by-code/:code → { id }` (published-only, 404 else, 401 no token).
- Files: `modules/ipa/ipa.service.js` (`resolveLessonIdByCode`), `ipa.controller.js` (`resolveByCode`),
  `ipa.routes.js` (route TRƯỚC `/lessons/:id`), test `__tests__/ipa.resolve.api.test.js`.
- Commit: `feat(ipa): resolve lesson id by code for course exercise launch`.

## Task C2 — Foundation exe-web
- `types/course.types.ts`, `services/course.service.ts`, `services/ipa.service.ts` (+resolve),
  `hooks/use-courses.ts`, `constants/query-keys.ts` (+courses), `constants/routes.ts` (+COURSES…),
  `components/layouts/nav-items.ts` (+"Khóa học").
- Commit: `feat(courses): service + hooks + routes foundation`.

## Task C3 — Catalog `/courses`
- `CourseCatalogContainer` + `CourseCard` + `app/(main)/courses/page.tsx`. Commit: `feat(courses): catalog page`.

## Task C4 — Chi tiết cây `/courses/[slug]`
- `CourseDetailContainer` + `app/(main)/courses/[slug]/page.tsx`. Commit: `feat(courses): course detail tree`.

## Task C5 — Lesson viewer + mở bài tập
- `+react-markdown +remark-gfm`; `MarkdownContent`, `LessonViewer`,
  `app/(main)/courses/[slug]/lessons/[lessonKey]/page.tsx`. ipa→resolve→push; talk→ConversationView inline.
- Commit: `feat(courses): lesson viewer with media + ipa/talk launch`.

## Task C6 — Verify + chốt
- exe-web `tsc`+`lint`+`build`; exe-api `jest ipa.resolve`. Cập nhật spec. Báo cáo bàn giao.

---

## Kết quả Giai đoạn C — ✅ Xong

| Task | Commit | Repo/Nhánh |
|---|---|---|
| C1 resolver ipa code→id | `7c8f734` | exe-api · `feature/lesson-runtime-api` |
| C2 foundation | `c992963` | exe-web · `feature/lesson-runtime-web` (base `develop`) |
| C3 catalog `/courses` | `061af89` | exe-web |
| C4 chi tiết cây `/courses/[slug]` | `e976805` | exe-web |
| C5 lesson viewer + mở ipa/talk | `a3f9364` | exe-web |

**Verify:**
- exe-api `jest ipa.resolve -i` → **4/4** (200 by-code, 404 chưa publish, 404 sai code, 401).
- exe-web `tsc --noEmit` → **0 lỗi**; `eslint` → **0 error** (chỉ warning có sẵn ở file khác); `next build` → **OK**,
  4 route mới: `/courses`, `/courses/[slug]`, `/courses/[slug]/lessons/[lessonKey]`, `/talk/[scenarioId]`.
- **Q-C1**: mở toàn bộ cây (chưa unlock tuần tự — Phase 3).
- **Convention**: page mỏng→container→AppShell (auth guard); service giải nén `{courses}`/`{course}`; hook TanStack
  `enabled: isAuthenticated`; `mediaUrl()` cho media; markdown qua `react-markdown`+`remark-gfm`.
- **Ranh giới features**: mở bài tập bằng **điều hướng route** (ipa→`/pronunciation/:id`, talk→`/talk/:scenarioId`),
  KHÔNG import chéo feature (tuân "features/ không import từ nhau"); thêm `TalkScenarioRunner` trong chính feature talk.
- **Chưa push/PR.** exe-web dùng lại cache `useCourseDetail` cho trang Bài (hợp đồng B không có endpoint per-lesson).
