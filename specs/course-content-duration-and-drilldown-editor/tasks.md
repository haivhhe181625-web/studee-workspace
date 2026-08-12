# Tasks: Thời lượng bài học + Editor nội dung drill-down

> **Spec:** `./spec.md` · **Design:** `./design.md` (đã duyệt 2026-08-06 — cột `DurationMin`=N, số thực → reject). **Ticket:** PRD-114.
> Chạy **exe-api (1→4) rồi exe-admin (5→7)**. FE (5→7) phụ thuộc contract BE (Task 4 `effectiveDurationMin`).
> **Repos:** exe-api `services/api` · exe-admin. Base nhánh: tách từ `develop` mỗi repo → nhánh `feat/PRD-114-<slug>`.

## Global Constraints
1. **exe-api:** English code/comment; Conventional Commits **kèm `PRD-114`** (vd `feat(course-content): PRD-114 …`); `git add <files>`; admin-api **chỉ delegate** service (no business logic); **KHÔNG đụng** `lesson-schedule.generator.js`. Jest `cd services/api && npx jest <p> -i`.
2. **exe-admin:** theo patterns; không sửa `components/ui/`; no `any`/`@ts-ignore`. Verify `npx tsc --noEmit` + `npm run lint` sạch; vitest. Commit kèm `PRD-114`.
3. Không nhắc plan-taxonomy (AC-n/IS-n) trong comment/tên test. Sửa nội dung chỉ khi status `draft/ready`.
4. Không mở scope: per-node endpoint · đổi contract PUT · generator lịch · trang lesson riêng · lưu `effectiveDurationMin` vào DB · migration.

---

## Task 1 (exe-api) — Vá `buildTree` giữ `estimatedDurationMin` (KEYSTONE)
**Files:** `modules/course-content/course-content.service.js`; test tương ứng.
- [ ] `buildTree()`: thêm `estimatedDurationMin` vào keep-list strip của lesson.
- [ ] Test: import (buildDoc) và updateCourse cho lesson có `estimatedDurationMin` → giá trị **được giữ**; lesson không có → `null` (không lỗi min:1).
- [ ] Verify `npx jest course-content -i` xanh (cập nhật assertion cũ nếu snapshot cây đổi). Commit `fix(course-content): PRD-114 preserve estimatedDurationMin through buildTree`.

## Task 2 (exe-api) — Parser cột `DurationMin` (N) + validate
**Files:** `modules/course-content/course-content.parser.js`, `course-content.service.js` (validateTree/validate); tests.
- [ ] Parser: đọc cột N (14) chỉ ở LESSON row → `estimatedDurationMin` (rỗng → null; else `Number`).
- [ ] Validate: nếu có mà **không integer > 0** (gồm số thực, ≤0, NaN) → push `INVALID_DURATION` kèm `row`; không chặn dòng khác.
- [ ] Tests: DurationMin=25 → field 25; blank → null; 0/"abc"/12.5 → đúng 1 lỗi INVALID_DURATION đúng dòng; file thiếu cột N → mọi lesson null (tương thích ngược).
- [ ] Verify `npx jest course-content parser -i` xanh. Commit `feat(course-content): PRD-114 parse optional DurationMin column from xlsx`.

## Task 3 (exe-api) — Template + Export round-trip `DurationMin`
**Files:** `admin-api/resources/course-content.admin.js` (§2 template), `modules/course-content/course-content.export.service.js`; tests.
- [ ] Template (§2): header cột N `DurationMin`; row LESSON mẫu để trống ô N (minh hoạ optional).
- [ ] Export `buildCourseXlsx`: ghi `estimatedDurationMin` vào cột N mỗi LESSON row; `null` → ô trống (không ghi 0).
- [ ] Test round-trip: export khóa có duration → parse lại file export = dữ liệu không đổi (gồm cả lesson null).
- [ ] Verify `npx jest course-content export -i` xanh. Commit `feat(course-content): PRD-114 round-trip DurationMin in template and export`.

## Task 4 (exe-api) — `effectiveDurationMin` ở detail (contract cho FE)
**Files:** `admin-api/resources/course-content.admin.js` (§4 shape) hoặc service `toDto`/shape; tests.
- [ ] `GET /:id`: mỗi lesson trả kèm `effectiveDurationMin = durationOf(lesson)` (read-only, không lưu DB); `estimatedDurationMin` thô vẫn trả.
- [ ] Không đưa `effectiveDurationMin` vào body PUT (client gửi lại bị bỏ qua — chỉ ghi `estimatedDurationMin`).
- [ ] Tests: lesson có field → effective = field; không có → effective = heuristic theo type; luôn > 0. PUT kèm effective → không persist.
- [ ] Verify `npx jest course-content -i` xanh. Commit `feat(course-content): PRD-114 expose effectiveDurationMin on course detail`.

## Task 5 (exe-admin) — Types mang `estimatedDurationMin`/`effectiveDurationMin`
**Files:** `services/course-content.service.ts`; tests nếu có.
- [ ] `LessonNode`: `estimatedDurationMin?: number | null` (editable) + `effectiveDurationMin?: number` (read-only).
- [ ] `updateCourse` payload gửi `estimatedDurationMin`, **không** gửi `effectiveDurationMin`.
- [ ] Verify `npx tsc --noEmit` + `npm run lint` sạch. Commit `feat(course-content): PRD-114 carry lesson duration in admin tree types`.

## Task 6 (exe-admin) — Editor drill-down (khóa→chặng→chuyên đề+lesson)
**Files:** `components/features/course-import/course-detail.tsx` (+ sub-view mới: nav/phase-editor/module-editor); tests.
- [ ] Container giữ **cả cây** + `dirty`/save/whole-tree PUT + publish-lock **như cũ**; thêm state điều hướng cấp (phase/module đang chọn).
- [ ] Render theo cấp: khóa (meta + list chặng) → chặng (meta + checkpoint + list chuyên đề) → chuyên đề (meta + **list lesson sửa inline**). Breadcrumb + lên cấp. Không có cấp lesson riêng.
- [ ] Chuyển cấp **không mất sửa dở**; khóa `published` → read-only.
- [ ] Tách sub-component < 200 dòng, DRY input helpers.
- [ ] Tests: điều hướng khóa→chặng→chuyên đề hiện đúng slice; sửa ở 1 cấp rồi qua cấp khác quay lại vẫn còn; Lưu gọi whole-tree PUT.
- [ ] Verify tsc+lint sạch; `npx vitest run course-detail` xanh. Commit `refactor(course-content): PRD-114 drill-down course editor navigation`.

## Task 7 (exe-admin) — Field duration per-lesson + rollup
**Files:** `course-detail.tsx`/module-editor, sub-component rollup; tests.
- [ ] Ô `estimatedDurationMin` mỗi lesson (number, trống = null = "auto theo loại"; hiện gợi ý effective).
- [ ] Rollup read-only tổng phút = Σ hiệu dụng client (`estimatedDurationMin ?? effectiveDurationMin`) ở chuyên đề/chặng/khóa.
- [ ] Tests: đổi duration 1 lesson → payload PUT mang giá trị mới; rollup chuyên đề/chặng/khóa cập nhật đúng; lesson trống → rollup dùng effective (heuristic) từ server.
- [ ] Verify tsc+lint; `npx vitest run course-detail` xanh. Commit `feat(course-content): PRD-114 per-lesson duration field with rollup totals`.

---
## Self-Review
AC-1→AC-7 map: T1 · T2 · T2 · T3 · T4 · T7 · T6. ⬜ Không mở scope ngoài spec §3.
