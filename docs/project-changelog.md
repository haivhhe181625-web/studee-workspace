# Project Changelog

Ghi nhận các thay đổi đáng kể. Định dạng theo nhóm Added / Changed / Fixed / Docs. Ngày dạng YYYY-MM-DD.

---

## 2026-07-23 — Adaptive: Đồng bộ API (band-history UI, gỡ FE schedule, cập nhật doc)

Rà soát 3 luồng (Hồ sơ năng lực / Lộ trình / Khóa học): BE endpoint ↔ FE ↔ tài liệu API (phục vụ web + mobile), lấp gap. Scope: `exe-web` + docs `exe-api`.

### Added
- **UI Lịch sử trình độ** (`BandHistoryCard`) — timeline các mốc đổi band (assessment / checkpoint / ops), mới nhất trước; chip Trình độ tổng + kỹ năng đổi band; ẩn khi chưa có mốc. Chèn ở `/adaptive` sau `ProfileCard`.
- **FE nối `GET /adaptive/band-history`** — type `BandHistory`/`BandHistoryEntry` + `getBandHistory` service + `useBandHistory` hook + query-key (trước đó BE có nhưng web chưa gọi).

### Removed
- **Gỡ FE study-schedule** cho nhất quán với BE (route `/adaptive/schedule*` đang đóng 404): xóa `WeeklySchedule`/`StudyPrefsForm`/`schedule-activity`, khối "Lịch học hôm nay" ở Dashboard (`ScheduleStrip`), 3 hook + 3 service + types + query-key. Giữ `Recommendations.weekSchedule` (khác feature).

### Docs (BE — contract chung web + mobile)
- `competency-profile-api.md`: bổ sung `profile.readyForCheckpoint`/`checkpointDismissed`; `path.progress`(%/completedModules)·`skippedMasteredSkills`·`module.lessonKeys`/`lessonSummary`; thêm §9 `POST /adaptive/checkpoint` (gated); sửa roadmap (overlay tiến độ đã làm). `courses-api.md` đã đủ.

### Coverage sau rà soát
- **Khóa học** (6 endpoint): FE ✅ · Doc ✅. **Hồ sơ + Lộ trình**: mọi endpoint đang mở đều FE ✅ + Doc ✅. Còn `POST /checkpoint` (gated, chờ Assessment Engine) + `/schedule*` (đóng).

### Verify
- `tsc`/`eslint` sạch · `vitest adaptive` 5/5 · `next build` exit 0.

---

## 2026-07-23 — Adaptive Path: Đại tu UX (Bản đồ Lộ trình + Phòng học tập trung)

Tách không gian Lộ trình thành **Map View** (bản đồ kho báu kiểu Duolingo) + **Focus Study Room** (phòng học 1 chuyên đề), kèm overlay tiến độ backend. Scope: `exe-web` feature `learn`; `exe-api` module `adaptive`.

### Added — Backend (path progress overlay)
- **`attachPathProgress`** (`adaptive.service.js`) — `GET /adaptive/path` trả `courses[].progress = { percent, completedModules }`. `%` tính **chỉ trên bài custom-path gán** (đã cắt bài đã giỏi): 3/3 bài giao = 100%, không theo tổng bài gốc. Đọc `CourseEnrollment` xuyên khóa, key `phaseKey/moduleKey/lessonKey` (Map-safe cả `.get()` lẫn lean object). `completedModules` qualify `phaseKey/moduleKey` (moduleKey lặp giữa các chặng). 6 unit test.
- **Builder** (`adaptive-path-builder.js`) — custom module trả thêm `lessonKeys` + `lessonSummary` (key/title/type) cho sidebar/overlay.

### Added — Frontend (không gian mới)
- **Map View** `PathMapContainer` (`/learn/path`) — 1 cột; các **Chặng = Milestone Node** (cúp/mốc) zigzag nối bằng **đường nét đứt SVG uốn lượn**; accordion xổ lưới thẻ chuyên đề (`grid-rows` transition, tự mở chặng đang học); trạng thái gold/primary/xám; 1 thẻ → căn giữa, nhiều thẻ → lưới. Click thẻ → `/learn/path/study?slug=&phase=&module=`.
- **Focus Study Room** `PathFocusStudyContainer` (`/learn/path/study`) — sidebar gọn **chỉ 1 chuyên đề + bài của nó**, content panel nới rộng, nút "← Quay lại Bản đồ Lộ trình". Chọn module qua URL.
- **Chuyển tiếp thông minh** — "Bài tiếp" ở bài cuối chuyên đề → **"Chuyên đề tiếp"** đổi URL sang module kế (`path-module-sequence.findNextModule`), sidebar tự reload mượt (kể cả sang khóa mới, lazy-enroll).
- **Lesson stepper** trong content panel (`Bài x/n`, chấm tròn), **gating mở khóa tuần tự** (bài chưa mở → toast + mờ + icon Lock), **completion** thể hiện bằng **màu chữ xanh** (minimalist, bỏ icon tick).

### Changed
- Primitive học tách dùng chung: `lesson-flatten`, `use-lesson-navigation`, `lesson-content-panel`, `path-module-sequence`. `LessonViewer` prop `enrolled`→`available`.

### Removed
- Xóa dead code `PathStudyContainer.tsx` + `PathTimeline.tsx` (bị Map + Focus Room thay thế hoàn toàn).

### Verify
- Backend `jest adaptive` xanh (thêm `adaptive-path-progress` 6 test). FE `tsc`/`eslint` sạch; `next build` exit 0 — routes `○ /learn/path` + `○ /learn/path/study`. E2E do PO nghiệm thu trên trình duyệt.

---

## 2026-07-23 — Adaptive Path Phase 3: Không gian học Lộ trình (Frontend)

Dựng không gian học riêng cho Lộ trình thích nghi (`exe-web`), tái dùng tối đa component/hook của Khóa học. Scope: `exe-web` feature `learn` + `adaptive`.

### Added
- **Không gian học Lộ trình** `PathStudyContainer` (route `/learn/path`) — vỏ điều hướng riêng, orchestrate **nhiều khóa**: chọn chuyên đề → nạp nội dung khóa (`useCourseStudy`) → học bằng `LessonViewer`. Tiến độ ghi **chung** qua `useRecordLesson(slug)` (`POST /courses/:slug/lessons/progress`). Auto lazy-enroll khóa nguồn khi mở.
- **`PathTimeline`** — danh sách courses→phases→chuyên đề (chế độ custom, chỉ kỹ năng cần), highlight active, đánh dấu `fitsLevel`; overlay % chỉ khóa đang mở (MVP).
- **`CheckpointBanner`** "Sẵn sàng thi vượt cấp" — soft thông báo từ `profile.readyForCheckpoint` (không gọi API — endpoint checkpoint đang rào).
- **Primitive học dùng chung** (rút từ `CourseStudyContainer`): `use-lesson-navigation` (active/prev-next), `lesson-flatten` (`flattenCourseTree`/`flattenModuleLessons`), `lesson-study-layout` (khung 2 cột), `lesson-content-panel` (header + LessonViewer + prev/next).
- **CTA "Vào không gian học lộ trình"** ở `AdaptivePathCard` (`/adaptive`) → `/learn/path` (điểm vào duy nhất).
- **Route** `ROUTES.LEARN_PATH`; types `AdaptivePath.{readyForCheckpoint,skippedMasteredSkills}` + `CompetencyProfile.{readyForCheckpoint,checkpointDismissed}`.

### Changed
- **`LessonViewer`** prop `enrolled` → **`available`** (ngữ nghĩa "bài mở khóa/khả dụng", không phải "đã ghi danh").
- **`CourseStudyContainer`** refactor để compose primitive dùng chung — hành vi course-space giữ nguyên (deep-link, gate ghi danh, prev/next toàn khóa, sidebar).

### Fixed (từ code review)
- **Auto-enroll dead-end** — path space giờ hiện `ErrorState` + retry khi lazy-enroll lỗi (trước: kẹt spinner vô hạn).
- **`useLessonNavigation`** trả **resolved key** (nhất quán `activeKey` với `active`) — tránh stale khi đổi chuyên đề.
- **A11y** — `aria-current` trên item nav active (path + course); label loading đúng ngữ cảnh.
- **RSC boundary** — thêm `"use client"` vào `use-lesson-navigation` (dùng `useState`): barrel `index.ts` kéo hook vào Server Component (`learn/page.tsx`) gây lỗi build; phát hiện qua smoke-test `next build` (tsc/eslint/vitest không bắt).

### MVP có chủ đích (defer)
- Prev/next **trong 1 chuyên đề** (cross-course sau); overlay tiến độ timeline **chỉ khóa đang mở** (overlay đầy đủ chờ backend path-progress); banner checkpoint chỉ thông báo (nối endpoint khi Assessment Engine xong).

### Verify
- `tsc --noEmit` 0 lỗi · `eslint` sạch · `vitest` 16/16 pass. 8 file mới · 6 file sửa.
- Smoke-test: `next build` exit 0 (`○ /learn/path` trong route manifest); `GET /learn/path` → HTTP 200, title "Lộ trình học · Studee". Full E2E (login→học bài) cần backend+seed — chưa chạy.

---

## 2026-07-23 — Adaptive Path: Band History, Checkpoint, Mastery-skip

Đồng bộ tài liệu Adaptive Path về code v3 + sửa lỗi logic backend (Lịch sử Band, Checkpoint, lọc bỏ bài đã giỏi). Scope: `exe-api` module `adaptive` + `assessment`; docs `studee-workspace/specs/adaptive-path` + `exe-api/docs`.

### Added
- **Band-history log** — mở rộng `LearnerProfile.history` thành log append-only ghi mọi lần đổi band (assessment / checkpoint / can thiệp vận hành): thêm `sourceGroup`, `bandSource`, `resultId`, `skillCefr` (snapshot band từng kỹ năng), `changedSkills`, và audit `actor`/`reason`/`notifiedLearner` (BR-33/34, BR-12).
- **Endpoint** `GET /api/adaptive/band-history` — timeline đổi band (mới nhất trước) cho "Nguồn cập nhật" (Block A) + đối sánh tiến bộ khi thi lại.
- **Cơ chế Checkpoint (core service)** — `applyCheckpointResult(userId, skill, passed, attemptId)`: đạt → nâng band kỹ năng +1 (cap ở band mục tiêu), tính lại overall level bằng công thức scorer (`computeOverall`), ghi band-history; chưa đạt → giữ band, không phạt (BR-31). `dismissCheckpoint` cho BR-30. HTTP endpoint `POST /api/adaptive/checkpoint` **tạm rào** chờ Assessment Engine phục vụ câu hỏi hiệu chuẩn.
- **Mastery-skip trong lộ trình** — `skillsMeetingMastery`/`masteredSkillSet` (pure); `buildAdaptivePath` nhận `masteredSkills`, bỏ qua module của kỹ năng đã giỏi (mastery ≥ 0.8, confirmed) thay vì bắt học lại (FR-B2). Giới hạn có chủ đích: skip ở **mức kỹ năng** (chưa mức từng-bài — cần gắn thẻ chủ điểm cho module).
- **Cờ checkpoint trên hồ sơ** — `profile.readyForCheckpoint` (gap≥1 + mastery ≥ 0.85 + chưa từ chối) và `profile.checkpointDismissed`; `StudentModel.checkpointDismissed`.

### Changed
- `profile.bandSource` suy động từ band-history entry mới nhất (thay vì hardcode `'assessment'`).
- `GET /api/adaptive/path` (builder v3): trả thêm `skippedMasteredSkills` (kỹ năng bỏ vì đã giỏi, ngưỡng 0.8); `readyForCheckpoint` lấy **cùng nguồn với `/profile`** (ngưỡng 0.85 + trừ dismissed) để 2 màn hình không mâu thuẫn.
- `seedFromResult` ghi band-history mỗi lần đo (Nhóm 1) kèm phát hiện kỹ năng đổi band.

### Fixed (từ code review)
- **Idempotency band-history** — `seedFromResult` dedup theo `resultId` (`bandHistoryHasResult`): queue retry cùng `attemptId` không còn nhân đôi entry / mất `changedSkills`.
- **Nhất quán `readyForCheckpoint`** — một nguồn duy nhất (`/profile`), tách khỏi `skippedMasteredSkills` (ngưỡng khác nhau, tên khác nhau).
- **Hiệu năng** — `getProfileView` chỉ lấy entry band-history cuối (`$slice: -1`) thay vì load cả log append-only.
- **Guard band tràn** — `applyCheckpointResult` chặn ghi `cefr` `undefined` khi band C2 + target bất thường.

### Docs
- Viết lại về v3 (hủy mô hình `adaptive_paths`/`stages` v1): `specs/adaptive-path/{data-model,design,plan}.md`, `contracts/adaptive-path.md`, `docs/prd-adaptive-path.md`, `exe-api/docs/api/integration/competency-profile-api.md`.

### Tests
- 94/94 pass (suite `adaptive`, 8 files). Thêm: mastery-skip (pure), checkpoint band-up + band-history + `checkpointReadySkills` (in-memory Mongo), helper `bandHistoryHasResult`/`skillCefrSnapshot`/`diffChangedSkills`.

### Chưa làm (ngoài phạm vi Phase 1/2)
- FE `PathStudyContainer` + tái dùng `LessonViewer` (Phase 3).
- Path progress overlay đọc `CourseEnrollment` (hook `completedModuleKeys` đã chừa chỗ trong builder).
- Mở `POST /api/adaptive/checkpoint` (thêm controller handler + Joi schema) khi Assessment Engine sẵn sàng.
