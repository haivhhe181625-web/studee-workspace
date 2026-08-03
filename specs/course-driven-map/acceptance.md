# Acceptance Criteria: Khóa học điều phối Bản đồ game

- **Spec:** `specs/course-driven-map/spec.md`
- **Ngày:** 2026-08-03

## Cách dùng file này

Mỗi tiêu chí có ID (`AC-1`, `AC-2`, ...) để `design.md` tham chiếu ngược. Given/When/Then cho hành vi phụ thuộc trạng thái; checklist đơn giản cho kiểm tra thẩm định.

## Tiêu chí chức năng

### AC-1: Generator sinh CourseMapTemplate từ published CourseStructure

- **Given** một CourseStructure `{status:'published', phases:[{modules:[{lessons:[{type:'theory', theory:'...'}, {exercises:[{type:'quiz', refId:'Q1'}]}]}]}]}`
- **When** tạo hoặc publish khóa học (trigger `generateMapFromCourse`)
- **Then** hệ thống tạo CourseMapTemplate với các node loại CONTENT (từ theory lessons) + MCQ (từ quiz-bound lessons) + FLASHCARD (từ deck-bound); `generated:true`; node SOFT-gated (entry mỗi khóa UNLOCKED, trong-khóa theo thứ tự); publish succeed (KHÔNG MAP_EMPTY error nếu ≥1 lesson mappable).

### AC-2: Theory lesson render thành CONTENT node + anti-leak

- **Given** lesson có `{type:'theory', theory:'Nội dung...'}` được map sinh
- **When** learner start node (POST /api/adaptive/map/node/:nodeKey/start) + getMap response
- **Then** CONTENT node trả lại trong getMap; theory chỉ qua startNode (sau LOCKED gate); publicConfig KHÔNG chứa `theory`/`itemIds`/`deckRefId` (strip SENSITIVE_CONFIG_KEYS). See `specs/learning-map-content-players/` for node grading detail.

### AC-3: Enroll khóa → appendCourseSegment idempotent

- **Given** learner ID có active LearnerPath; course slug "ielts-5-6"
- **When** POST /api/courses/ielts-5-6/enroll
- **Then** 
  - (lần 1) appendCourseSegment non-blocking; segment nối vào path.segments[], LearnerPath.nodeStates seeded (entry node UNLOCKED); HTTP 200 enroll-success
  - (lần 2) đế lại POST = idempotent, segment KHÔNG duplicate (E11000 guard → fall-through append winner)
  - Lỗi append KHÔNG rollback enroll (non-blocking)

### AC-4: Concurrent enroll 2 khóa → cả 2 segment order phân biệt

- **Given** learner ID, 2 course slug riêng biệt "ielts-5-6" + "toeic-700-800"
- **When** 2 POST /api/courses/:slug/enroll chạy concurrently (race condition)
- **Then** cả 2 segment có mặt trong path.segments[]; order phân biệt theo append-time (appendSegmentOcc guarded + recompute order).

### AC-5: Roadmap explorer (pre-placement): list courses by band range

- **Given** learner chưa StudentModel (needsAssessment = true)
- **When** GET /api/adaptive/map/roadmap-preview?program=ielts&fromCefr=A1&toCefr=B2
- **Then** hệ thống trả [courses] với cefrFrom≤A1, cefrTo≥B2 (band overlap); projection {slug, title, program, phases:[{title, modules:[{title, nodes:[{title, activityType}]}]}]}; KHÔNG theory/itemIds/deckRefId (anti-leak test strict).

### AC-6: RoadmapExplorer UI pre-placement branch

- **Given** learner chưa StudentModel (no StudentModel doc)
- **When** truy /adaptive
- **Then** AdaptiveMapContainer render `<RoadmapExplorer/>` (program tabs + start/target band picker + accordion courses/chặng/chuyên đề/nodes); map KHÔNG render; CTA "Làm bài kiểm tra năng lực" (link ROUTES.ASSESSMENT_INTAKE).

### AC-7: Game map render post-placement + competency cards

- **Given** learner đã StudentModel + enrolled course
- **When** truy /adaptive
- **Then** AdaptiveMapContainer render game map + competency panel (ProfileCard + StudentModelCard + TargetEditor); no RoadmapExplorer; modal + HUD + gameplay intact.

### AC-8: Quiz lesson chấm QUA map submitNode (HMAC)

- **Given** learner start node quiz (POST /api/adaptive/map/node/:nodeKey/start, HMAC signed)
- **When** POST /api/adaptive/map/node/:nodeKey/submit {answers, HMAC}
- **Then** submitNode (server-key + HMAC verify + bind) chấm quiz; grader KHÔNG được gọi song parallel (KHÔNG backdoor null-key); applyResult update node progress → COMPLETED / FAILED. See `specs/learning-map-content-players/contracts/map-node-players.md` for full node grading contract.

### AC-9: Placement precondition gate

- **Given** learner chưa StudentModel; enroll + tấn công vào game map (submit node, start node)
- **When** thực hành trên map node
- **Then** submitNode/startNode throw `NEEDS_ASSESSMENT` (409 CONFLICT); getMap catch + return `{empty:true, needsAssessment:true}`; không kẹt câm (CTA history rõ ràng).

### AC-10: /adaptive route unify + /adaptive/map redirect

- **Given** routing setup sau surface-unification
- **When** GET /adaptive hoặc GET /adaptive/map
- **Then** 
  - `/adaptive` → AdaptiveMapContainer (game map component)
  - `/adaptive/map` → HTTP 301 redirect `/adaptive`
  - KHÔNG còn `/adaptive/path` route hay separate dashboard

### AC-11: getMap no-path empty-state

- **Given** learner mới, chưa enroll khóa nào
- **When** GET /api/adaptive/map (hay map-fetch API internal)
- **Then** ensureActivePath catch LEARNER_PATH_NOT_FOUND → createNewPath; appendCourseSegment empty → resolver `[]` (demo archive); getMap return `{empty:true}` (200 OK, KHÔNG 404).

### AC-12: Demo-foundation archive-only (KHÔNG sửa resolver)

- **Given** prod environment migration
- **When** chạy migration archive demo-foundation; seed code guard non-prod
- **Then** 
  - Template demo-foundation record có `status:'archived'`
  - template-resolver.js KHÔNG sửa (KHÔNG xóa fallback)
  - 13 test seed demo published vẫn PASS (resolver auto-pick demo khi band match)
  - learner mới band>A2 → resolver `[]` → empty-state (demo excluded by archive)

## Tiêu chí lỗi / edge case

### AC-E1: Theory lesson RỖNG → skip (không emit CONTENT)

- **Given** lesson `{type:'theory', theory:null}` bound trong course
- **When** generateMapFromCourse
- **Then** skip lesson (fallback skip); publish PASS (KHÔNG CONTENT_NO_SOURCE error).

### AC-E2: Lesson RỖNG toàn bộ (no theory/media/exercise) → fail validate

- **Given** lesson `{type:'vocab_flashcard', exercises:[], media:[], theory:''}`
- **When** import / save lesson
- **Then** validate fail (EMPTY_LESSON error); import không tạo lesson.

### AC-E3: Quiz lesson bind sai refId → fail generate

- **Given** lesson `{exercises:[{type:'quiz', refId:'INVALID_Q999'}]}`
- **When** generateMapFromCourse
- **Then** generator skip lesson (EXERCISE_NOT_FOUND) hoặc fail import (resolve tùy schema strict); publish KHÔNG throw generic 500.

### AC-E4: Concurrent append race E11000 → fallthrough

- **Given** 2 concurrent POST /courses/:slug/enroll, 2 appendCourseSegment race $push
- **When** MongoDB E11000 duplicate key (index trên segment key)
- **Then** winner append thành công; loser E11000 catch → fall-through append vào path winner (idempotent, KHÔNG exception throw).

### AC-E5: Re-enroll (route chưa match uid) → KHÔNG duplicate

- **Given** learner enroll, course lesson.key đổi → new uid → re-enroll
- **When** POST /api/courses/:slug/enroll lần 2
- **Then** reconcile match (phase.key + module.key + lesson.key) → carry uid cũ; appendSegmentOcc detect uid cũ đã có → skip (KHÔNG duplicate segment).

### AC-E6: Roadmap explorer anti-leak: theory KHÔNG lộ

- **Given** lesson có sensitive `theory:'Đáp án: A là đúng...'` + `exercises:[{refId:'Q1'}]`
- **When** GET /api/adaptive/map/roadmap-preview
- **Then** response KHÔNG chứa `theory` field; chỉ `{title, activityType:'QUIZ'}`; content đầy đủ chỉ qua getMap (learner enrolled + placement).

### AC-E7: Regenerate (course edit) → enroll mới lấy version mới, cũ xong trên version cũ

- **Given** CourseMapTemplate v1 published + learner1 enroll (v1) + learner2 enroll sau (= v1)
- **When** course edit (lesson thêm/xóa) → trigger generateMapFromCourse → v2 publish
- **Then** learner1 path segment pin v1; learner2 enroll mới → nhận v2; regression: learner1 xong học v1 KHÔNG vỡ.

### AC-E8: Submit node LOCKED (gating chưa unlock) → 403

- **Given** node LOCKED (prereq chưa pass)
- **When** POST /api/adaptive/map/node/:nodeKey/submit
- **Then** submitNode throw 403 FORBIDDEN (LOCKED_NODE hoặc tương đương); KHÔNG 409 NEEDS_ASSESSMENT.

## Tiêu chí phi chức năng

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| Bảo mật | submitNode HMAC verify ≥ 1 key mismatch reject; itemIds KHÔNG leak qua publicConfig khi LOCKED | test {HMAC sai → 403}, {theory field strip}, {config trả tối thiểu SENSITIVE_CONFIG_KEYS set} |
| Bảo mật | appendCourseSegment assertEnrolled gate + defense-in-depth | test direct appendCourseSegment call (bypass controller) → no insert; enroll endpoint → OK |
| Hiệu năng | getMap (cached LearnerPath) ≤ p95 500ms; append non-blocking (≤ 3s timeout) KHÔNG block enroll | load-test 100 concurrent enrolls; measure append latency vs enroll response |
| Độ tin cậy | Dual-progress (map vs course-detail): document limitation (2 store coexist); rebase sync = V4 | document in design.md; unit test courseEnrollment↔LearnerPath mismatch scenario |

## Ngoài phạm vi kiểm thử

- Retire /adaptive/path API (GIỮ nguyên coexist) → kiểm sau ở plan 3B.
- Migrate learner cũ demo-path → course-map (KHÔNG làm v1).
- Auto-checkpoint/band-up từ course metadata (deferred P5).
- Dual-progress unify + sync (deferred V4).
- Theory-as-CONTENT-node V3 (cần activityType + adapter mới).
