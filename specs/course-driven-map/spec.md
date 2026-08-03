# Spec: Khóa học điều phối Bản đồ game (Course-Driven Gamified Map)

- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Trạng thái:** Đã build (spec hồi tố)
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — xây xong: binding quiz vào lesson, generator khóa→map, append enroll, roadmap explorer, /adaptive surface unification
  - `web` (`exe-web`) — xây xong: /adaptive game map, roadmap explorer UI, competency cards
- **Module liên quan (exe-api):**
  - `src/modules/course-content/` — khóa học (CourseStructure) là source-of-truth
  - `src/modules/adaptive/` — LearnerPath (đường học), StudentModel (năng lực), adaptive controller

## 1. Mục tiêu

**Unify** luồng học sinh thành **1 bản đồ game điều phối bởi khóa học**: CourseStructure = nguồn sự thật; bản đồ game (map-v2) = VIEW dẫn xuất one-way qua generator; enroll tự append khóa vào roadmap duy nhất; pre-placement learner duyệt roadmap trước khi có placement; /adaptive = game map làm cơ chế học chính (retire demo-foundation).

## 2. Bối cảnh

### Hiện trạng đã kiểm tra trong code

- **CourseStructure** (course-content.model.js:1-112) — lưu phân tầng đầy đủ: Lộ trình → Chặng → Chuyên đề → Bài học (mỗi lesson lưu theory/media/exercises); trạng thái vòng đời (draft/ready/published/archived); CEFR band metadata trên Chặng.
- **Lesson content binding** — exercises[] chứa soft ref `[{type:'ipa'|'talk'|'quiz'|'flashcard', refId}]` (course-content.model.js:41-48); media[] lưu URL video/audio (MediaRefSchema, :54-62); theory lưu markdown (LessonSchema:74).
- **LearnerPath** (student-path.model.js trong adaptive) — 1 document/user: nodeStates (journey progress per node); segments[] (course segments enrolled).
- **Enroll** (course-content.routes.js:23) — endpoint POST /api/courses/:slug/enroll xác nhận người dùng.
- **StudentModel** (student-model.model.js) — trữ năng lực (CEFR band, target, skills mastery); tạo sau placement.
- **Learning-map routes** (learning-map.routes.js mounted at /api/adaptive/map) — GET /api/adaptive/map (getMap), GET /api/adaptive/map/roadmap-preview, POST /api/adaptive/map/course/:courseSlug/start (startCourseMap), POST /api/adaptive/map/node/:nodeKey/start, POST /api/adaptive/map/node/:nodeKey/submit (see sibling spec learning-map-content-players for node grading detail).

### Lý do chặp nhất

- CourseStructure sinh ra là để lưu cấu trúc khóa học import; nhưng nó chứa đủ metadata (theory, media, exercises) → dùng thẳng làm content view.
- Map-v2 engine (submitNode + gating + HMAC-bind-rate-limit) đã proven E2E (125 test); reuse nguyên instead of xây engine mới.
- Enroll tự động thêm khóa vào LearnerPath thay vì SoloGame-per-course → 1 roadmap unified (render theo thứ tự, placement là cổng).
- Demo-foundation (old fallback template) blocking learner mới → archive, empty-state + CTA.

## 3. Phạm vi

### Trong phạm vi

- **IS-1** — **Khóa học = Source of Truth**: CourseStructure (lộ trình + phân tầng) điều khiển sinh ra bản đồ game; content sống tại Lesson (theory/media/exercises).
- **IS-2** — **Generator khóa→map** (one-way): generator tạo CourseMapTemplate (VIEW) từ mỗi published CourseStructure; content-node từ theory; SOFT linear gating (không auto-checkpoint); pin-version + createNewVersion (enroll mới lấy version mới; learner đang học xong trên version cũ).
- **IS-3** — **Bind quiz vào lesson** (không song song grader): lesson.exercises[] chứa quiz refId; submitNode qua map (server-key + HMAC) chấm — KHÔNG reuse mcq.adapter (null-key backdoor).
- **IS-4** — **Enroll auto-append**: POST /api/courses/:slug/enroll → appendCourseSegment (non-blocking, idempotent) → segment nối vào LearnerPath.segments[]; reconcile (getMap/ensureActivePath tự bù segment thiếu).
- **IS-5** — **Single roadmap**: LearnerPath duy nhất/user; segments[] theo thứ tự enroll (render tuần tự, entry mỗi khóa UNLOCKED, gating trong-khóa SOFT).
- **IS-6** — **Roadmap explorer** (pre-placement): learner chưa StudentModel → duyệt roadmap (start→destination band, courses→chặng→chuyên đề→nodes) read-only, anti-leak (chỉ titles + activityType, KHÔNG theory/itemIds).
- **IS-7** — **Surface unification**: /adaptive render game map; /adaptive/map redirect /adaptive; retire demo-foundation (archive, không resolver sửa); getMap no-path → {empty:true} + CTA enroll.
- **IS-8** — **Competency cards** (/adaptive): ProfileCard + StudentModelCard + TargetEditor + RecommendationList (reuse từ adaptive dashboard).
- **IS-9** — **(DEFERRED)** Auto-checkpoint/band-up từ course (thiếu config.skill/item-source); theo dõi rebase in-flight learners; admin curation overlay.

### Ngoài phạm vi

- **Retire /adaptive/path API** — GIỮ nguyên (coexist); retire plan riêng (3B) — **Lý do:** separation concerns; cá nhân hóa (mastery-skip/skill-gap) giữ track riêng.
- **Migrate learner cũ** (demo-path → course-map) — **Lý do:** archive demo chỉ chặn learner MỚI; cũ vẫn chạy nguyên (không hỏng data).
- **Teorý-as-CONTENT-node** (V3) — **Lý do:** cần activityType mới + adapter + player + validator; MVP interim render theory inline (off-map).
- **Dual-progress unify** — **Lý do:** 2 store coexist (map=LearnerPath, course-detail=CourseEnrollment); sync là refactor riêng (V4).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner (chưa placement) | Duyệt roadmap: chọn start→destination band, xem courses/chặng/chuyên đề/nodes | Hiểu khóa học trước khi thi placement; pick một khóa để thi placement + enroll |
| Learner (đã placement) | Enroll khóa học → bản đồ tự nối vào 1 roadmap | Học 1 luồng unified (không solo game per-course); play map, gặp CONTENT/MCQ/VIDEO/FLASHCARD cạnh nhau |
| Learner (đang học) | Xem bản đồ + năng lực (ProfileCard/StudentModelCard/TargetEditor) | Biết band hiện tại, target, kỳ vọng kế tiếp, skills yếu; chơi map với gating |
| Academic (admin) | Tạo/publish khóa học (CourseStructure) với theory + media + bind quiz | Phát hành khóa học (generator auto sinh map); nhất quán content vs gating |

## 5. Quyết định nghiệp vụ cần chốt

> **Đã chốt toàn bộ qua red-team + owner interview (2026-07-28).**

| Câu hỏi | Lựa chọn (chốt) | Người quyết |
|---|---|---|
| Khóa học = source-of-truth hay map = source? | CourseStructure = source; map = VIEW dẫn xuất → regenerable | Owner (red-team 27/07) |
| GIỮ hay retire /adaptive/path (cá nhân hóa)? | **GIỮ** — coexist; retire ở plan 3B | Owner (red-team reshape) |
| Chấm quiz qua map /submit (HMAC) hay grader song parallel? | /submit qua node (server-key + HMAC); KHÔNG backdoor grader | Red-team F1 (security) |
| Enroll khóa đã tồn tại: đè, version mới, hay từ chối? | Pin-version: enroll mới lấy version mới; học xong version cũ | Owner (V2 validation) |
| Placement = cổng trước enroll hay sau enroll? | **Cổng trước** — precondition; document CTA | Red-team C2 (failure-mode) |
| Thứ tự làm CONTENT node hay Append trước? | **CONTENT trước** (giá trị #1, self-contained) → Append sau | Red-team Sc-F5 (scope) |
| Retire demo-foundation: sửa resolver hay archive-only? | **Archive-only** — KHÔNG sửa resolver (phá test) | Verification V1 (28/07) |

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map về mục "Trong phạm vi".

1. Khóa học published → CourseMapTemplate sinh được với CONTENT node (theory) + MCQ/VIDEO/FLASHCARD (nếu bind); enroll lấy map đó (IS-2).
2. Learner enroll khóa → khóa tự append vào LearnerPath, render theo thứ tự (IS-4, IS-5).
3. Learner chưa placement → /adaptive hiển explorer (start→destination, courses/chặng/chuyên đề/nodes); với StudentModel → game map (IS-6).
4. Quiz lesson chấm QUA submitNode (HMAC server-side); không grader song song (IS-3).
5. Concurrent enroll 2 khóa → cả 2 segment present, order phân biệt (IS-5).
6. Learner mới (no-path) → /adaptive empty + CTA "Ghi danh khóa học" (IS-7).
7. /adaptive = game map; /adaptive/map redirect (IS-7).
8. Placement precondition: learner chưa StudentModel → nhánh needs-placement + CTA (IS-6).
9. Competency cards (/adaptive): ProfileCard + StudentModelCard + TargetEditor visible post-placement (IS-8).
10. Regression: BE 344 (learning-map + course-map + course-content + map-template) + FE 429 test xanh (IS-2, IS-4, IS-7).

## 7. Câu hỏi mở

- **Q1**: RoadmapExplorer component (FE) được tích hợp ở đâu — AdaptiveMapContainer pre-placement branch? → Cần confirm layout/routing.
- **Q2**: `enroll→appendCourseSegment` non-blocking: lỗi append không rollback enroll — learner thấy "enrolled" nhưng segment mất. Reconcile ở `ensureActivePath` giải quyết? → Test `concurrent E11000` → fall-through đã verify hay cần manual test?

## 8. Ghi chú cho Tech Lead Design

- **HARD RULE generator = one-way**: CourseStructure dùng làm input; không có "update map" back-sync → content sống ở Lesson, map là snapshot. Regenerate = createNewVersion + publish; learner đang học giữ pin-version cũ.
- **SOFT gating chỉ**: MVP KHÔNG auto-checkpoint/band-up (thiếu `config.skill`/`item-source`/StudentModel metadata) → CP1 + CP2 kế tiếp.
- **Lesson.uid bất biến**: surrogate id (KHÔNG derive từ lesson.key mutable) → pin nodeKey tới uid. Re-import: match (phase.key + module.key + lesson.key) → carry uid cũ (document trong import spec).
- **Theory-as-CONTENT interim**: theory không emit CONTENT node V1 (cần activityType mới); interim render theory inline trong study-space tới khi có.
- **Anti-leak**: getNode/publicConfig strip `theory`/`itemIds`/`deckRefId` khi status LOCKED; roadmap-preview chỉ titles + activityType.
- **Placement gate**: submitNode giữ ném NEEDS_ASSESSMENT; precondition document + CTA hướng + reconcile ở getMap/appendCourseSegment (defense-in-depth).
