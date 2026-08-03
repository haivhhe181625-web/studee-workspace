# Design: Khóa học điều phối Bản đồ game

- **Spec:** `specs/course-driven-map/spec.md`
- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Trạng thái:** Build complete (design hồi tố)
- **ADR liên quan:** `docs/adr/0001-course-as-source-of-truth.md` (proposed)

## 1. Tóm tắt kiến trúc

**Model-View pattern one-way**: CourseStructure (model — import Phase 1, lưu phân tầng complete + CEFR + content) → generator → CourseMapTemplate (view — flat nodes, SOFT gating, pin-version). LearnerPath = 1 document/user (segments[] append theo enroll; nodeStates[] track progress per node). Map-v2 engine reuse (submitNode + gating + HMAC-bind + rate-limit) — không tạo engine mới. /adaptive surface merge: game map primary; pre-placement branch = roadmap-preview explorer. Demo-foundation archive (KHÔNG resolver sửa) → learner mới empty-state + CTA.

Nguồn plans:
- `260728-0100-course-driven-gamified-map`: binding lesson→quiz, generator, learner-surface, E2E verify
- `260728-content-gamemap-unify-replace-path`: CONTENT node (theory), append-order, placement-gate
- `260728-1842-roadmap-explorer-course-maps-competency`: enrich seed + generate + explorer UI + competency cards
- `260728-1716-surface-unification`: /adaptive unified, demo-archive, getMap empty

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| CourseStructure = Source-of-Truth | Không generate map → reuse course; generate map ← course | ACID import (Phase 1); trẻ truy content live từ lesson (không copy stale); regenerate = pin-version, không rebase | Complexity generator; dual-storage (map cache vs course live) |
| Generator one-way | Course → Map (createNewVersion+publish); KHÔNG update khóa qua map | Simplicity; avoid sync bugs; learner mới lấy v-mới, cũ xong v-cũ | Không curation qua map UI (P6 riêng) |
| SOFT gating chỉ (MVP) | Lesson → node entry UNLOCKED; gating trong-khóa theo thứ tự linear | Đủ để play đơn; avoid hard-gate edge-case (STRUCTURAL_LOCKED 409) | Checkpoint/band-up deferred (thiếu metadata) |
| Pin-version | Enroll snapshot version; learner đang học KHÔNG vỡ khi khóa đổi | Stability (learner journey pin); fresh learner mới lấy fresh version | Re-import learner cũ là backlog (defer) |
| Lesson.uid bất biến | Surrogate id (NOT derive từ lesson.key mutable) | Key-change = uid-mới = lost nodeKey → completed mất; uid stable cross-import → carry cũ | Import-match logic (phase.key+module.key+lesson.key) phức tạp hơn slug |
| Theory off-map (interim) | Theory KHÔNG emit CONTENT v1; render inline trong study-space | MVP scope; CONTENT node cần activityType+adapter mới | Đẩy V3; learner thấy theory inline không trên map |
| Placement = cổng trước | StudentModel = precondition enroll/map; submitNode ném NEEDS_ASSESSMENT | Single error code (409); CTA clear (làm bài kiểm tra) | Query StudentModel 2x (getMap call); future skip-test cần handle |
| Demo archive-only | Archive record status; seed guard non-prod; resolver KHÔNG sửa | Test-safe (13 demo test PASS); no resolver hack | Cần migration script archive prod; learner cũ vẫn chạy (KHÔNG migrate) |

## 3. Data model

### Tầng Backend

#### CourseStructure (source-of-truth, course-content.model.js)

```
CourseStructure
├── _id: ObjectId
├── slug: String (unique)
├── title: String
├── program: String (enum PROGRAM_KEYS)
├── status: String enum ['draft','ready','published','archived']
├── thumbnail: String (URL, http(s):// only)
├── centerId: ObjectId | null (Phase 1 = null)
├── phases: [PhaseSchema]
│   ├── key: String
│   ├── title: String
│   ├── order: Number
│   ├── cefrFrom/cefrTo: String enum CEFR_LEVELS | null (band metadata)
│   ├── goalNote: String (IELTS 5.0→6.0, etc.)
│   └── modules: [ModuleSchema]
│       ├── key: String
│       ├── title: String
│       ├── order: Number
│       ├── category: String enum MODULE_CATEGORIES
│       └── lessons: [LessonSchema]
│           ├── key: String
│           ├── uid: String (surrogate, bất biến, phase.key:module.key:lesson.key)
│           ├── title: String
│           ├── order: Number
│           ├── type: String enum LESSON_TYPES
│           ├── theory: String (markdown)
│           ├── contentUrl: String (video/audio URL)
│           ├── media: [MediaRefSchema]
│           │   ├── type: String enum ['video','audio']
│           │   ├── url: String (http(s)://, KHÔNG Base64)
│           │   ├── label: String
│           │   └── order: Number
│           └── exercises: [ExerciseRefSchema]
│               ├── type: String enum ['ipa','talk','quiz','flashcard']
│               ├── refId: String (quiz.code | deck._id)
│               ├── order: Number
│               └── label: String
├── importedBy: ObjectId → User
├── createdAt/updatedAt: Date
└── sourceMeta: { sourceFile, importDate, ... }
```

#### CourseMapTemplate (generated view, learning-map.model.js — proposed)

```
CourseMapTemplate
├── _id: ObjectId
├── courseId: ObjectId → CourseStructure
├── slug: String (= course.slug)
├── version: Number (1, 2, ... pin-version strategy)
├── status: String enum ['draft','published','archived']
├── nodes: [NodeSchema] (flat, dùng nodeKey + prereq để lock)
│   ├── nodeKey: String (lesson.uid-based, immutable)
│   ├── type: String enum ['CONTENT','EXERCISE','MCQ','VIDEO','FLASHCARD']
│   ├── title: String
│   ├── order: Number
│   ├── config: {
│   │   ├── theory: String (CONTENT type only; KHÔNG lộ qua getNode LOCKED)
│   │   ├── itemIds: [String] (MCQ type; KHÔNG lộ qua publicConfig)
│   │   ├── deckRefId: String (FLASHCARD type; KHÔNG lộ)
│   │   ├── contentUrl: String (VIDEO type)
│   │   ├── media: [MediaRefSchema] (VIDEO/AUDIO support)
│   │   └── exercises: [ExerciseRefSchema] (drill-down)
│   ├── gating: {
│   │   ├── type: String enum ['SOFT','HARD'] (MVP SOFT only)
│   │   ├── prereqs: [String] (nodeKey list; empty = entry node)
│   │   └── minMastery: Number | null
│   └── publishedAt: Date
├── generatedFrom: ObjectId (= courseId; track provenance)
├── generated: Boolean (true = auto via generator)
├── createdAt/updatedAt: Date
└── publishedAt: Date
```

#### LearnerPath (per-user roadmap + progress, adaptive/student-path.model.js)

```
LearnerPath
├── _id: ObjectId
├── userId: ObjectId → User
├── segments: [SegmentSchema]
│   ├── courseId: ObjectId (enrolled course)
│   ├── courseSlug: String
│   ├── courseMapVersion: Number (pin-version; regenerate không backtrack)
│   ├── appendedAt: Date
│   ├── status: String enum ['active','completed','archived']
│   └── nodeStates: [NodeStateSchema]
│       ├── nodeKey: String
│       ├── status: String enum ['unlocked','locked','in_progress','completed','failed']
│       ├── attempt: Number
│       ├── score: Number | null
│       ├── startedAt/completedAt: Date
│       └── mastery: Number | null
├── activeSegmentIndex: Number | null (multi-segment: track which to render)
├── createdAt/updatedAt: Date
└── lastSegmentAppendedAt: Date (track append order)
```

#### StudentModel (competency state, adaptive/student-model.model.js)

```
StudentModel
├── _id: ObjectId
├── userId: ObjectId → User
├── band: String enum CEFR_LEVELS
├── program: String (IELTS/TOEIC/etc.)
├── targetScore: Number | null
├── targetCefr: String enum CEFR_LEVELS | null
├── skills: {
│   "<skillName>": {
│   ├── current: String (CEFR)
│   ├── target: String (CEFR)
│   ├── mastery: Number (0-1)
│   ├── gap: Number (target-current)
│   └── lastAttemptAt: Date
│   }
│ }
├── createdAt/updatedAt: Date
└── startedAt: Date
```

### Tầng Frontend

#### Types (`types/adaptive-map.types.ts`)

```typescript
interface RoadmapCourse {
  slug: string;
  title: string;
  program: string;
  phases: RoadmapPhase[];
  bandRange?: [string, string]; // [cefrFrom, cefrTo]
}

interface RoadmapPhase {
  title: string;
  modules: RoadmapModule[];
}

interface RoadmapModule {
  title: string;
  nodes: RoadmapNode[];
}

interface RoadmapNode {
  title: string;
  activityType: string; // 'CONTENT' | 'MCQ' | 'VIDEO' | 'FLASHCARD'
}

interface MapViewResult {
  empty?: boolean;
  needsAssessment?: boolean;
  nodes: PublicNode[];
  nodeStates: NodeState[];
  currentNode?: string;
}

interface PublicNode {
  nodeKey: string;
  title: string;
  type: string;
  order: number;
  config: {
    itemIds?: string[]; // strip if LOCKED
    deckRefId?: string; // strip if LOCKED
    media?: Media[];
    contentUrl?: string;
  };
  gating: GatingInfo;
  publishedAt: Date;
}
```

## 4. Luồng dữ liệu

```
### Phase 1: Import → Generate Map

Academic → POST /admin/courses/import
  ├─→ validation (parse + structure validate + exercise-ref validate)
  └─→ CourseStructure insert (atomicity all-or-nothing)

Academic → POST /admin/courses/:id/publish
  ├─→ trigger generateMapFromCourse(courseId)
  │   ├── read CourseStructure
  │   ├── projectLessonToNode (map lesson → node type)
  │   │   ├── lesson.type='theory' → CONTENT node
  │   │   ├── lesson.exercises[type='quiz'] → MCQ node
  │   │   ├── lesson.exercises[type='flashcard'] → FLASHCARD node
  │   │   ├── lesson.contentUrl → VIDEO node
  │   │   ├── lesson.exercises[type='ipa'|'talk'] → skip (not mapped)
  │   │   └── lesson.theory='' + exercises=[] + media=[] → skip (EMPTY_LESSON)
  │   ├── buildGating (linear: node-N prereq=[node-(N-1)]; entry=no-prereq)
  │   ├── CourseMapTemplate create ({version:1, status:'draft'})
  │   └── publish → status='published', publishedAt=now
  └─→ response {courseId, templateId, nodeCount}

### Phase 2: Enroll → Append Segment

Learner → POST /api/courses/:slug/enroll
  ├─→ auth (verifyToken) + tentant + enrollment-gate
  ├─→ progress.service.enroll(userId, slug)
  │   ├── createCourseEnrollment (CourseEnrollment doc track completion)
  │   └── appendCourseSegment(userId, slug) [NON-BLOCKING]
  │       ├── resolve CourseStructure by slug + publish → template version
  │       ├── LearnerPath.segments.find(slug) → if exist = skip (idempotent)
  │       ├── bootstrap segment {courseId, courseMapVersion:N, nodeStates:[]}
  │       ├── $push segment to path.segments
  │       ├── ensureActivePath set activeSegmentIndex
  │       └── reconcile enrollment↔segment mismatch
  ├─→ retry 3x non-blocking lỗi append (fall-through; KHÔNG fail enroll)
  └─→ response {enrollmentId, courseId} (200, KHÔNG wait append)

### Phase 3: Learner Journey

Learner → GET /adaptive
  ├─→ auth (verifyToken)
  ├─→ check StudentModel
  │   ├─ no StudentModel → render RoadmapExplorer (pre-placement)
  │   └─ StudentModel exist → render GameMap + competency cards (post-placement)
  └─→ response AdaptiveMapContainer state

  Case 1: Pre-placement (no StudentModel)
    ├─→ GET /api/adaptive/map/roadmap-preview?program=ielts&fromCefr=A1&toCefr=B2
    ├─→ service resolve: CourseStructure{program, cefrFrom≤A1, cefrTo≥B2}
    ├─→ map → {slug, title, phases:[{title, modules:[{title, nodes:[{title,activityType}]}]}]}
    └─→ FE: RoadmapExplorer browse courses/phases/modules/nodes; CTA "Làm bài kiểm tra"

  Case 2: Post-placement (has StudentModel)
    ├─→ GET /api/adaptive/map
    ├─→ service.getMap(userId)
    │   ├── ensureActivePath (create/reconcile LearnerPath)
    │   ├── resolver: active path → segments[] → course slug → template version
    │   ├── if no template: resolver fallback → empty (demo archive)
    │   ├── build nodeStates from path.segments[active].nodeStates
    │   └── return {nodes, nodeStates, empty?:false}
    ├─→ FE: AdaptiveMapContainer render GameMap + ProfileCard + StudentModelCard + TargetEditor
    └─→ response MapViewResult

### Phase 4: Node Start/Submit (Course-driven Map Grading)

**Note**: Full node grading detail (POST /api/adaptive/map/node/:nodeKey/start + POST /api/adaptive/map/node/:nodeKey/submit) documented in sibling spec `specs/learning-map-content-players/contracts/map-node-players.md`. Course-driven-map focuses on data-flow entry points (getMap) and course→map lifecycle. Node HMAC-bind, gating, type-specific grading (CONTENT/MCQ/FLASHCARD/VIDEO) are centralized player contracts.

### Phase 5: Regenerate (Course Change → New Version)

Academic → PATCH /admin/courses/:id (edit lesson/add/remove)
  ├─→ validation + update CourseStructure
  └─→ trigger (debounce) generateMapFromCourse
      ├── CourseMapTemplate.findOne(courseId, status='published')
      ├── createNewVersion {version:N+1, draft status}
      ├── generate nodes from updated course
      ├── publish (status='published')
      └── mark old version archived (?)

Learner → POST /api/courses/:slug/enroll (after regenerate)
  ├─→ appendCourseSegment resolve latest version N+1
  └─→ new learner segment pin version N+1; old learner segment pin N (no rebase)
```

## 5. Contracts

Liệt kê contracts rõ ràng ở thư mục `contracts/`:

- `contracts/get-map.md` — GET /api/adaptive/map (learner-facing game map)
- `contracts/roadmap-preview.md` — GET /api/adaptive/map/roadmap-preview (pre-placement explorer)
- `contracts/enroll-course.md` — POST /api/courses/:slug/enroll (append-trigger)
- See sibling spec `specs/learning-map-content-players/contracts/map-node-players.md` — POST /api/adaptive/map/node/:nodeKey/start + /submit (HMAC grading)

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/course-content/course-content.model.js` | CourseStructure schema + validation | Sửa (thêm uid + media validation) |
| `src/modules/course-content/course-content.service.js` | Import + publish + validation logic | Sửa (generator call on publish) |
| `src/modules/course-content/course-content.parser.js` | Excel/JSON parser | Sửa (nếu cần format update) |
| `src/modules/learning-map/course-map-generator.js` | NEW — generator lesson→node + gating + version | Tạo |
| `src/modules/learning-map/learning-map.service.js` | NEW — getMap + ensureActivePath + reconcile | Tạo |
| `src/modules/learning-map/learning-map.routes.js` | NEW — GET /api/adaptive/map + /api/adaptive/map/roadmap-preview + POST /api/adaptive/map/course/:courseSlug/start | Tạo |
| `src/modules/adaptive/student-path.service.js` | appendCourseSegment + bootstrap | Sửa (append logic) |
| `src/modules/adaptive/adaptive.controller.js` | /adaptive routes handler | Sửa (wire map endpoint) |
| `src/modules/adaptive/adaptive.service.js` | getStudentModel + profile + competency | Sửa (thêm map entry point) |
| `scripts/seed-course-structures.js` | Seed dev courses (theory + media + exercise bind) | Sửa (enrich) |
| `scripts/generate-maps-for-published-courses.js` | NEW — batch generate maps | Tạo |
| `scripts/archive-demo-foundation.js` | NEW — migration archive demo | Tạo |
| `scripts/migrate-demo-archive-guard.js` | Seed guard: non-prod = skip seed demo | Sửa seed-map-template |
| `exe-web/src/components/features/adaptive/AdaptiveMapContainer.tsx` | Merge /adaptive route + explorer branch | Sửa |
| `exe-web/src/components/features/adaptive/RoadmapExplorer.tsx` | NEW — pre-placement explorer UI | Tạo |
| `exe-web/src/components/features/adaptive/RoadmapCourseCard.tsx` | NEW — course card component | Tạo |
| `exe-web/src/services/adaptive-map.service.ts` | Fetch getMap + roadmap-preview | Tạo |
| `exe-web/src/hooks/use-adaptive-map.ts` | Hook: useGetMap + useRoadmapPreview | Tạo |
| `exe-web/src/types/adaptive-map.types.ts` | TS types (RoadmapCourse, MapViewResult, etc.) | Tạo |
| `docs/adr/0001-course-as-source-of-truth.md` | ADR: decision course=source, map=view | Tạo |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| Learner chưa placement, start/submit node | 409 | NEEDS_ASSESSMENT |
| Node LOCKED, try submit | 403 | FORBIDDEN (hay LOCKED_NODE) |
| Quiz HMAC invalid | 403 | INVALID_HMAC |
| Exercise refId KHÔNG tồn tại (import) | 400 | EXERCISE_NOT_FOUND |
| Lesson RỖNG (EMPTY_LESSON) | 400 | EMPTY_LESSON |
| Khóa học duplicate (re-import same slug) | 409 | COURSE_ALREADY_EXISTS |
| Curriculum learner chưa enroll, getStudyContent | 403 | NOT_ENROLLED |
| /adaptive no map (empty) | 200 | {empty:true, needsAssessment?:bool} |
| Roadmap explorer (no StudentModel) | 200 | [courses] or {empty:true} |

## 8. Bảo mật & quyền

- **Permission string (mới)**: `course:generate-map` (system actor only); `learning-map:write` (admin); `learning-map:read` (learner self-only).
- **Tenant scope**: course `centerId:null` (platform-shared, Phase 1); future tenant-scoped = Phase 2 + migration.
- **Sensitive fields**:
  - `theory` — chỉ via startNode AFTER LOCKED gate; KHÔNG trong publicConfig hay getNode.
  - `lesson.exercises[].refId` — chỉ server-side resolve; KHÔNG lộ item ID qua publicConfig khi LOCKED.
  - `deckRefId` — chỉ server-side; client không biết deck-id nào.
  - `StudentModel.skills[].gap` — private (KHÔNG share qua recommend nếu learner B compare A).
- **HMAC bind**: submitNode server-key (secret stored env); verify before grade.
- **Anti-leak test (code-reviewer verified)**: roadmap-preview chỉ titles + activityType; getNode strip config khi LOCKED.

## 9. Rủi ro / đánh đổi / câu hỏi mở

### Rủi ro

1. **Dual-progress (map vs course-detail)**: LearnerPath (map progress) ≠ CourseEnrollment (course completion). DOCUMENT limitation; sync = V4. Rollback tức: 0-byte (no back-rebase); learner thấy khác nhau (notification cần hướng dẫn).
2. **Regenerate learner-old pin-version**: khi course edit, learner cũ còn node cũ deleted → getNode 404 (handle: fallback skip hay graceful?). Verify end-to-end.
3. **Append non-blocking lỗi kín**: appendCourseSegment timeout/fail KHÔNG roll back enroll → learner thấy "enrolled" nhưng mất segment. Reconcile ở getMap/ensureActivePath bù, nhưng cần retry + monitoring.

### Đánh đổi

1. **PIN-VERSION không rebase**: learner cũ học version cũ xong (simplicity); rebase = future. Cost: không live-update course xong mid-learn (document timing).
2. **SOFT gating chỉ (MVP)**: hard-gate skip → khóa-cứng (checkpoint) defer P5. Cost: learner có thể skip tùy (trade competency-tracking).
3. **Theory off-map (interim)**: theory không CONTENT node V1 → learner thấy inline off-map (không phần thưởng "bài học đọc xong"). Cost: engagement (document; V3 priority).

### Câu hỏi kỹ thuật mở (research nếu cần)

- **Q1**: Enrich seed "rich bindings" — bao nhiêu MCQ pool / system deck size / video/course ? → Confirm ở script seed-course-structures.js.
- **Q2**: Roadmap explorer pre-placement — filter nào exactly: `cefrFrom ≤ fromCefr` AND `cefrTo ≥ toCefr`? → Verify logic ở getRecommendedCourses.
- **Q3**: Concurrent E11000 fallthrough — MongoDB E11000 re-throw hay catch+skip? → Verify driver behavior.

## 10. Testing strategy

### Unit Tests

- **course-map-generator.test.js**: projectLessonToNode (tất cả LESSON_TYPES); buildGating (linear prereq); skip-reasons (type='ipa', empty-lesson); CONTENT-no-config.
- **student-path.service.test.js**: appendCourseSegment (idempotent); concurrent append (E11000 guard); reconcile (first-build, missing segment).
- **roadmap-preview.service.test.js**: band-filter (cefrFrom/cefrTo overlap); anti-leak (no theory/itemIds); empty course list.
- **submitNode.test.js**: HMAC verify (invalid→403); LOCKED gate (prereq check); applyResult (score/status update); type-specific (CONTENT/MCQ/FLASHCARD).

### Integration Tests

- **getMap (learner-surface).test.js**: no StudentModel → empty; StudentModel + enrolled → map nodes; multiple segments → order preserved; no-template → fallback empty.
- **enroll-append.test.js**: concurrent enroll 2 courses → both segment + order distinct; enroll ≥2x → idempotent.
- **course-generate.test.js**: publish trigger generate → template v1; regenerate → v2; learner1 pin v1, learner2 get v2.
- **roadmap-explorer.test.js** (FE): pre-placement render explorer; IELTS A1→B2 → courses with band-overlap; no StudentModel → empty → CTA visible.
- **adaptive-container.test.js** (FE): no StudentModel → explorer; post-placement → map + cards; TargetEditor edit → refetch.

### E2E / Real-stack (manual browser + docker)

- **Seed**: `npm run seed:all` → 8 published courses + 8 non-empty generated maps (DB assert).
- **No-StudentModel → placement → enroll → play**:
  - Browser JWT-inject (no StudentModel).
  - GET /adaptive → explorer render (A1→B2 courses).
  - POST /onboarding/placement-result (place to B1).
  - POST /courses/ielts-5-6/enroll.
  - GET /adaptive → game map + competency cards.
  - Submit MCQ/CONTENT node → score/complete.
- **Concurrent enroll**: 2 parallel POST enrolls → both segment saved (DB verify).
- **Anti-leak verify**: no-StudentModel roadmap response payload {no theory/itemIds}; LOCKED node config {no itemIds/deckRefId}.

## 11. Rollout / cross-repo sequencing

### BE release (exe-api)

1. **Merge + test**:
   - course-map-generator.js + service logic (generateMapFromCourse, appendCourseSegment, reconcile).
   - course-content: publish trigger + seed enrich.
   - learning-map routes + controller (getMap, roadmap-preview).
   - adaptive: wire getMap entry point, remove/disable old path endpoints (GIỮ API, route defer P3B).
   - archive-demo-foundation migration.
2. **Deploy** (feature-branch, test-verified):
   - Verify 344 tests pass (learning-map + course-map + course-content + map-template + adaptive).
   - Run seed:all + generate-maps-for-published-courses.js (non-prod).
   - Run migration archive-demo-foundation (prod).

### FE release (exe-web)

1. **Merge + build**:
   - RoadmapExplorer + RoadmapCourseCard components.
   - AdaptiveMapContainer refactor (pre-placement branch).
   - use-adaptive-map hook + service + types.
   - Route /adaptive → AdaptiveMapContainer; /adaptive/map → redirect (tùy router implementation).
2. **Deploy** (after BE live):
   - 429 vitest + tsc + eslint + next build green.
   - Browser JWT-harness real-stack verify (explorer + map + cards).
   - Verify no regression: /learn/path, /profile, dashboard.

### Sequencing lock

- **BE first**: generate-maps + archive-demo-foundation live → FE ready di route.
- **FE follow**: no delay necessary (FE getMap call → BE endpoint live).
- **Monitoring**: watch demo-archive fallback (resolver `[]`); ERR_LEARNER_PATH_NOT_FOUND → expected (empty-state); spike investigate.

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion (acceptance.md) | Được đáp ứng bởi |
|---|---|
| AC-1: Generator CourseMapTemplate | course-map-generator.js::generateMapFromCourse |
| AC-2: Theory CONTENT + anti-leak | LessonSchema.theory + publicConfig strip + startNode gate |
| AC-3: Enroll appendCourseSegment | student-path.service.appendCourseSegment (idempotent) |
| AC-4: Concurrent segment order | appendSegmentOcc guarded + recompute order |
| AC-5: Roadmap explorer list | roadmap-preview.service::resolveByBandRange + FE RoadmapExplorer |
| AC-6: Explorer pre-placement UI | AdaptiveMapContainer pre-placement branch + RoadmapExplorer render |
| AC-7: Map + cards post-placement | AdaptiveMapContainer post-placement branch + ProfileCard/StudentModelCard/TargetEditor compose |
| AC-8: Quiz HMAC chấm | submitNode HMAC verify + grade qua map (no parallel grader) |
| AC-9: Placement gate | submitNode throw NEEDS_ASSESSMENT (409); getMap catch + empty |
| AC-10: /adaptive route unify | adaptive.routes.js: GET /adaptive → AdaptiveMapContainer; redirect /adaptive/map |
| AC-11: getMap empty-state | getMap catch LEARNER_PATH_NOT_FOUND → ensureActivePath bootstrap + resolver `[]` → {empty:true} |
| AC-12: Demo archive | migration archive-demo-foundation + seed guard non-prod |
