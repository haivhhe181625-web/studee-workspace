# Data model: Khóa học điều phối Bản đồ game

- **Spec:** `specs/course-driven-map/spec.md`
- **Design:** `specs/course-driven-map/design.md`
- **Ngày:** 2026-08-03

## 1. Collection `CourseStructure` (Source of Truth)

**Mô tả**: Một CourseStructure = một khóa học; phân tầng đầy đủ (Lộ trình → Chặng → Chuyên đề → Bài học) nhúng trong 1 document (atomic import Phase 1).

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `slug` | String | ✓ | | unique index; kebab-case; khóa tự nhiên phát hiện trùng lúc import |
| `title` | String | ✓ | | tên lộ trình |
| `description` | String | | `''` | mô tả khóa |
| `program` | String enum | ✓ | | PROGRAM_KEYS: 'ielts' / 'toeic' / ... |
| `thumbnail` | String (URL) | | `null` | ảnh cover; bắt buộc http(s)://, **cấm Base64** (feature-spec §5.1) |
| `centerId` | ObjectId → Center | | `null` | **Phase 1 = null** (platform-shared); future tenant-scope |
| `status` | String enum | ✓ | `'ready'` | ['draft','ready','published','archived']; import Phase 1 tạo 'ready'; phase 2+ (manual state change) |
| `phases` | [PhaseSchema] | ✓ (≥1) | | Chặng — nhúng; validate: ≥1 không rỗng (ràng buộc BR "Ready cần ≥1 Chặng") |
| `importedBy` | ObjectId → User | ✓ | | người import (audit trail) |
| `sourceMeta` | Object | | `null` | {fileName, importDate, rowCount, ...} |
| `createdAt` / `updatedAt` | Date | auto | | |

**Indexes**:
- `{slug: 1}` unique
- `{status: 1, program: 1}` (roadmap explorer filter)
- `{centerId: 1}` (future tenant isolation)

**Ràng buộc**:
- `slug` unique per centerId (or global if centerId:null).
- `status='ready'` → phải ≥1 Chặng, Chặng ≥1 Chuyên đề, Chuyên đề ≥1 Bài học (tất cả non-empty).
- Import Phase 1 validate EMPTY_CHILDREN + EMPTY_LESSON trước insert.

---

## 2. Sub-document (nhúng) — Cây phân tầng

### PhaseSchema (Chặng — nấc nâng cấp trình độ) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | duy nhất trong khóa (AC-E3); không chứa `/` hoặc kí tự đặc biệt |
| `title` | String | ✓ | tên chặng (vd "A2 to B1", "Band 5-6") |
| `order` | Number | ✓ | thứ tự render; 0-indexed |
| `cefrFrom` | String enum | | CEFR_LEVELS: ['A1','A2','B1','B2','C1','C2'] hoặc null; trình độ bắt đầu |
| `cefrTo` | String enum | | CEFR_LEVELS hoặc null; trình độ đích. Validate: cefrTo ≥ cefrFrom (nếu cả có) |
| `goalNote` | String | | 'IELTS 5.0→6.0' hoặc mục tiêu tự do; optional |
| `modules` | [ModuleSchema] | ✓ (≥1) | Chuyên đề con; validate ≥1 không rỗng |

**Ngữ nghĩa**: Chặng mô tả một nấc trình độ của khóa. Từ A2 lên B1: cefrFrom='A2', cefrTo='B1'. Nếu là IELTS: cefrTo='B1' + goalNote='IELTS 5.0'. CEFR optional → linh hoạt.

---

### ModuleSchema (Chuyên đề — theo mảng kỹ năng) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | duy nhất trong Chặng cha; pattern tương tự phase.key |
| `title` | String | ✓ | vd "Listening for Detail", "Pronunciation: Vowels" |
| `order` | Number | ✓ | thứ tự trong chặng |
| `category` | String enum | | ['grammar','pronunciation','vocabulary','listening','reading','writing','speaking','other']; default 'other'; ngữ nghĩa kỹ năng |
| `lessons` | [LessonSchema] | ✓ (≥1) | Bài học con; validate ≥1 không rỗng |

**Ngữ nghĩa**: Chuyên đề = một mảng kỹ năng (pronunciation, từ vựng, ...) trong một chặng. Giúp cấu trúc bài học thành nhóm có ý nghĩa + enable future competency tracking.

---

### LessonSchema (Bài học — học đa phương thức) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | duy nhất trong Chuyên đề cha; pattern: 'lesson-001', 'reading-adv', ... |
| `uid` | String | ✓ | **surrogate id bất biến**; format: `${courseSlug}:${phaseKey}:${moduleKey}:${lessonKey}` hoặc hash; dùng làm nodeKey trong map (KHÔNG derive từ mutable lesson.key) |
| `title` | String | ✓ | "Reading Exercise 3", "IPA Lesson: /ə/" |
| `order` | Number | ✓ | thứ tự trong module |
| `type` | String enum | ✓ | LESSON_TYPES: ['theory','video_lesson','ipa_exercise','listening_quiz','reading_quiz','grammar_quiz','vocab_flashcard','speaking_task','writing_task']; default 'theory' |
| `theory` | String (markdown) | | nội dung lý thuyết; tối đa ~50KB; text thuần + markdown (KHÔNG HTML thô). Rỗng `''` nếu không có |
| `contentUrl` | String (URL) | | URL video/audio chính (nếu type='video_lesson'); http(s)://, KHÔNG Base64 |
| `media` | [MediaRefSchema] | | danh sách link media bổ trợ (video/audio); rỗng `[]` hợp lệ |
| `exercises` | [ExerciseRefSchema] | | tham chiếu bài tập (ipa/talk/quiz/flashcard); rỗng `[]` hợp lệ (lesson chỉ-lý-thuyết) |

**Ràng buộc "Bài học không rỗng" (EMPTY_LESSON)**:
- Phải có **ít nhất một** trong {theory (non-empty), ≥1 media, ≥1 exercises}.
- Rỗng hoàn toàn (theory='', media=[], exercises=[]) → lỗi EMPTY_LESSON (import reject).
- Giải thích: bài học chỉ-lý-thuyết (không media/exercise) hợp lệ; bài học RỖNG không.

**Ghi chú lesson.uid**:
- Bất biến cross-import (nếu re-import: match phase.key+module.key+lesson.key → reuse uid cũ).
- KHÔNG derive từ lesson.key (key có thể đổi, uid PHẢI ổn định → pin progress).
- Re-import khi lesson.key đổi: uid mới → tiến độ cũ mất (document warning cho academic team).

---

### MediaRefSchema (một link media bổ trợ) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | String enum | ✓ | ['video','audio']; type khác → UNSUPPORTED_MEDIA_TYPE |
| `url` | String (URL) | ✓ | http(s)://YouTube/CDN/R2/...; **cấm Base64/data:** (feature-spec §5.1) → MEDIA_BASE64_BLOCKED; không http(s) → INVALID_URL |
| `label` | String | | "Explanation", "Native Speaker Example"; tuỳ chọn (tối đa 100 ký tự) |
| `order` | Number | ✓ | thứ tự media trong bài học (0-indexed) |

**Ghi chú Base64/data:**
- Hệ thống **cấm** nhúng media Base64/`data:` thẳng vào DB.
- Lý do: document size limit (16MB); performance (parsing client-side); CDN cache benefits (URL dùng được nhiều learner).
- Giải pháp: content team host trên CDN (R2, YouTube, S3, ...); import chỉ lưu URL.

---

### ExerciseRefSchema (Tham chiếu bài tập) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | String enum | ✓ | ['ipa','talk','quiz','flashcard']; **Phase 1 CHỈ ipa+talk+quiz+flashcard**. (Nghe/Đọc/Viết/Ngữ pháp cần engine mới → Phase 2+) |
| `refId` | String | ✓ | type='ipa' → IPA lesson code; type='talk' → scenario id; type='quiz' → Quiz.code; type='flashcard' → FlashcardDeck._id string |
| `order` | Number | ✓ | thứ tự exercise trong bài học |
| `label` | String | | "IPA Warm-up", "Quiz 1"; tuỳ chọn (tối đa 100 ký tự) |

**Ghi chú EXERCISE_TYPES**:
- Phase 1 generator skip: type='ipa_exercise', 'speaking_task', 'writing_task' (không node được map).
- CHỈ quiz + flashcard + ipa + talk được bind vào lesson (trong exercises[]).
- Quiz: validate refId = LessonQuiz.code + status='published' (AS-Q3 chốt).

---

## 3. Collection `CourseMapTemplate` (Generated View)

**Mô tả**: CourseMapTemplate = snapshot dẫn xuất từ CourseStructure (map-v2 engine use). Một khóa = một map; một map = nhiều version (pin-version strategy).

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `courseId` | ObjectId → CourseStructure | ✓ | | FK: phát hiện khóa sinh từ map |
| `slug` | String | ✓ | | = course.slug (copy, denorm for quick lookup) |
| `version` | Number | ✓ | | 1, 2, 3, ...; regenerate = createNewVersion+publish |
| `status` | String enum | ✓ | `'draft'` | ['draft','published','archived']; publish = ready enroll |
| `nodes` | [NodeSchema] | ✓ | | flat danh sách node (không tree); generator sinh từ lesson (order maintained) |
| `generatedFrom` | ObjectId | ✓ | | = courseId (audit trail; track provenance) |
| `generated` | Boolean | ✓ | `true` | `true` = auto via generator; `false` = hand-authored (phase 6 future) |
| `publishedAt` | Date | | `null` | lúc publish (status='published') |
| `createdAt` / `updatedAt` | Date | auto | | |

**Indexes**:
- `{courseId: 1, version: 1}` compound unique (one map/course/version)
- `{slug: 1, status: 1}` (resolver: find active template by slug)
- `{status: 1}` (batch generate query)

**Ràng buộc**:
- Một courseId → một active version published (others archived).
- Regenerate: createNewVersion (draft) → copy nodes + gating → publish → old version archive (optional).

---

### NodeSchema (Node trong CourseMapTemplate) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `nodeKey` | String | ✓ | lesson.uid (surrogate, bất biến); dùng làm ID node trên map |
| `type` | String enum | ✓ | ['CONTENT','MCQ','FLASHCARD','VIDEO','EXERCISE']; CONTENT = theory lesson; MCQ = quiz-bound; FLASHCARD = deck-bound; VIDEO = contentUrl; EXERCISE = ipa/talk (skip map, off-map hiển thị) |
| `title` | String | ✓ | từ lesson.title |
| `order` | Number | ✓ | từ lesson.order (linear phục vụ gating) |
| `description` | String | | tối đa 500 ký tự; tuỳ chọn |
| `config` | Object | ✓ | loại-specific config (see below) |
| `gating` | Object | ✓ | gating metadata (see below) |
| `publishedAt` | Date | ✓ | lúc generator publish map |

**NodeSchema.config** (loại-specific):

```json
{
  "CONTENT": {
    "theory": "...(markdown)",    // KHÔNG lộ qua publicConfig khi LOCKED
    "media": [...],                // optional
    "contentUrl": "..."            // optional video
  },
  "MCQ": {
    "itemIds": [...],              // KHÔNG lộ khi LOCKED
    "passThreshold": 0.6,          // quizConfig.passingScore map
    "media": [...]
  },
  "FLASHCARD": {
    "deckRefId": "...",            // KHÔNG lộ khi LOCKED
    "cardCount": 20,               // thông tin
    "media": [...]
  },
  "VIDEO": {
    "contentUrl": "https://...",
    "media": [...],
    "duration": 300                // giây
  },
  "EXERCISE": {
    "type": "ipa" | "talk",
    "refId": "..."
  }
}
```

**NodeSchema.gating** (mapping thứ tự → prereq):

```json
{
  "type": "SOFT" | "HARD",         // MVP: SOFT only
  "prereqs": ["uid-node-1"],       // nodeKey list; empty = entry node
  "minMastery": 0.7                // optional, P5+ (checkpoint)
}
```

**Ghi chú config strip**:
- `getMap` response nodes' config **KHÔNG chứa**: theory, itemIds, deckRefId khi node.status='LOCKED'.
- Client side cần display activity-type icon (CONTENT/MCQ/VIDEO) từ node.type KHÔNG config.

---

## 4. Collection `LearnerPath` (Per-user Roadmap + Progress)

**Mô tả**: LearnerPath = 1 document/user; track segments enrolled + nodeStates progress.

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `userId` | ObjectId → User | ✓ | | unique (1 path/user) |
| `segments` | [SegmentSchema] | ✓ | [] | danh sách khóa học enrolled; mỗi segment = một course map version |
| `activeSegmentIndex` | Number | | `null` | index segment hiện render (multi-segment roadmap); null = chưa active |
| `createdAt` / `updatedAt` | Date | auto | | |

**Indexes**:
- `{userId: 1}` unique
- `{userId: 1, "segments.courseId": 1}` (detect duplicate enroll)

---

### SegmentSchema (Segment CourseMapTemplate) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `courseId` | ObjectId | ✓ | FK CourseStructure |
| `courseSlug` | String | ✓ | = course.slug (denorm for quick lookup) |
| `courseMapId` | ObjectId | | FK CourseMapTemplate (optional; tùy caching strategy) |
| `courseMapVersion` | Number | ✓ | pin version lúc enroll; regenerate KHÔNG backtrack |
| `status` | String enum | | `'active'` | ['active','completed','archived'] |
| `nodeStates` | [NodeStateSchema] | ✓ | [] | progress per node; lazy-populate lúc enroll (append) |
| `appendedAt` | Date | ✓ | now | lúc enroll (track order) |
| `completedAt` | Date | | `null` | lúc segment mark complete |

**Ghi chú nodeStates**:
- Append lúc enroll: bootstrap từ CourseMapTemplate.nodes (copy order + type; state='unlocked' entry, locked sau).
- Mỗi node có NodeStateSchema track progress riêng.

---

### NodeStateSchema (Progress node trong segment) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `nodeKey` | String | ✓ | = CourseMapTemplate.node.nodeKey |
| `status` | String enum | ✓ | ['unlocked','locked','in_progress','completed','failed']; initial = unlocked (entry) hay locked (prereq chưa) |
| `attempt` | Number | | 0; increment sau mỗi try submit |
| `score` | Number | | [0,1]; null nếu chưa submit hay type=CONTENT |
| `passed` | Boolean | | true = score ≥ passThreshold; null nếu chưa submit |
| `startedAt` | Date | | lúc learner start node (POST /api/adaptive/map/node/:nodeKey/start) |
| `completedAt` | Date | | lúc pass (score ≥ threshold) hay CONTENT submitted |
| `mastery` | Number | | [0,1]; mastery score (P4+ feature; track skill progress). Null nếu chưa grade |
| `feedback` | String | | tuỳ chọn; personalized hint/explanation (P5+) |

**Ghi chú status flow**:
- Initial: entry-node = 'unlocked'; prereq-blocked = 'locked'.
- Start: → 'in_progress'.
- Submit (MCQ pass): → 'completed', score update, computeUnlocks(prereqs) → next node 'unlocked'.
- Submit (MCQ fail): → 'failed', attempt++; retry allowed (KHÔNG auto next).
- CONTENT submit: → 'completed' (no score, just ack).

---

## 5. Collection `StudentModel` (Competency State, Adaptive Phase)

**Mô tả**: StudentModel = năng lực profile sau placement; track CEFR band, target, per-skill mastery.

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `userId` | ObjectId → User | ✓ | | unique (1 model/user) |
| `band` | String enum CEFR | ✓ | | ['A1','A2','B1','B2','C1','C2']; từ placement result |
| `program` | String | ✓ | | 'ielts' / 'toeic' / ... |
| `targetScore` | Number | | `null` | TOEIC 700, IELTS 6.5, ... |
| `targetCefr` | String enum | | `null` | target band level |
| `targetSetManually` | Boolean | | `false` | true = learner edit via PUT /adaptive/target |
| `skills` | Object | | `{}` | per-skill CEFR + mastery (see below) |
| `createdAt` / `updatedAt` | Date | auto | | |
| `placementAttemptId` | ObjectId | | | FK Assessment attempt; audit trail |

**Indexes**:
- `{userId: 1}` unique
- `{band: 1, program: 1}` (roadmap resolver filter by target band)

**StudentModel.skills** (dynamic object):

```json
{
  "listening": {
    "current": "B1",
    "target": "B2",
    "mastery": 0.65,       // [0,1] từ quiz scores
    "gap": 1,              // target-current (semitone)
    "lastAttemptAt": "2026-08-03T..."
  },
  "reading": { ... },
  "writing": { ... },
  ...
}
```

**Ghi chú**:
- `skills` field tuỳ chọn; rỗng nếu placement generic (KHÔNG skill-breakdown).
- `mastery` update mỗi lần quiz grade; aggregate từ per-skill quiz items (future: per-item skill tag).
- `targetSetManually` track: learner edit → targetCefr thay đổi → trigger replan.

---

## 6. Constraints & Validation

### Import Validation (Phase 1, course-content.parser.js)

| Error Code | Tình huống | HTTP Code |
|---|---|---|
| `EMPTY_COURSE` | khóa 0 chặng | 400 |
| `EMPTY_PHASE` | chặng 0 chuyên đề | 400 |
| `EMPTY_MODULE` | chuyên đề 0 bài | 400 |
| `EMPTY_LESSON` | bài học (theory='' + media=[] + exercises=[]) | 400 |
| `DUPLICATE_KEY` | phase.key / module.key / lesson.key trùng trong scope | 400 |
| `INVALID_CEFR` | cefrFrom/cefrTo ∉ CEFR_LEVELS | 400 |
| `INVALID_CEFR_ORDER` | cefrTo < cefrFrom | 400 |
| `MEDIA_BASE64_BLOCKED` | media.url dạng Base64/data: | 400 |
| `INVALID_URL` | media.url không http(s):// | 400 |
| `EXERCISE_NOT_FOUND` | quiz.code / deck._id không tồn tại | 400 |
| `EXERCISE_NOT_PUBLISHED` | quiz/deck status ≠ 'published' | 400 |
| `UNSUPPORTED_MEDIA_TYPE` | media.type ∉ ['video','audio'] | 400 |
| `COURSE_ALREADY_EXISTS` | slug trùng + status='ready' | 409 |
| `INVALID_FILE_FORMAT` | file không .xlsx | 400 |

### Data Integrity (MongoDB Unique Indexes)

| Index | Collision → Error |
|---|---|
| `CourseStructure.slug` | re-import same slug → 409 COURSE_ALREADY_EXISTS (handled: check before insert) |
| `LearnerPath.userId` | E11000 → fallthrough (1 path guaranteed per user) |
| `StudentModel.userId` | E11000 → replace existing (placement retake) |
| `segment on courseId+version` (compound) | E11000 if append twice same version (handled: idempotent skip) |

### Access Control (within learner self-only)

- **GET /adaptive**: verifyToken + withTenant (userId from JWT)
- **GET /api/adaptive/map**: verifyToken (learner can only see own path)
- **POST /api/courses/:slug/enroll**: verifyToken (learner can only enroll self)
- **GET /api/adaptive/map/roadmap-preview**: verifyToken (no StudentModel = pre-placement)
- Admin admin-api routes: verifyPermission `course:write` + `learning-map:write`

---

## 7. Migration Path (if needed)

### Scenario: Regenerate Course (Lesson Edit)

```
1. Learner1 enroll ielts-5-6 → courseMapVersion=1 → segment pin v1
2. Academic edit lesson (add media) → trigger generateMapFromCourse
3. New CourseMapTemplate v2 created (draft) → publish
4. Learner2 enroll ielts-5-6 → resolver find v2 → segment pin v2
5. Learner1 path.segments[0].courseMapVersion=1 (KHÔNG change → pin-version safety)
```

### Scenario: Re-import Same Course (Key Match)

```
1. Initial import: phase.key='P1' + module.key='M1' + lesson.key='L1' → uid='slug:P1:M1:L1'
2. Re-import same course + lesson.key='L1' still → match uid cũ → reuse (KHÔNG new uid)
3. BUT: lesson.key='L1' → 'L1-v2' → NEW uid → tiến độ cũ KHÔNG carry (document warning)
```

---

## 8. Denormalization (Performance)

| Field | Nơi copy | Lý do |
|---|---|---|
| `segment.courseSlug` (from course) | LearnerPath.segments | quick filter enroll duplicate; không query course mỗi lần |
| `node.title` (from lesson.title) | CourseMapTemplate.nodes | avoid lesson lookup lors render map |
| `CourseMapTemplate.slug` (from course.slug) | template | resolver query by slug, không FK lookup |

---

## 9. Size Estimation & Limits

| Entity | Giới hạn | Lý do |
|---|---|---|
| CourseStructure document size | ≤ 16MB | MongoDB document limit |
| ~200 chặng × 10 chuyên đề × 5 bài = 10k bài học | ~8MB (embeds) | safe margin |
| LessonSchema.theory | ≤ 50KB | markdown; avoid bloat |
| media[] per lesson | ≤ 10 links | reasonable media support |
| exercises[] per lesson | ≤ 20 refs | not expected many |
| LearnerPath document | ≤ 100MB | millions of nodeStates (N learners × M nodes) |
| NodeState per learner | ~1KB × N nodes | if 10k nodes → 10MB per learner (manageable) |

**Monitoring**: if `LearnerPath.size()` → >100MB, shard by userId.

---

## 10. Backward Compatibility

- CourseStructure v1 = Phase 1 (lock schema; no breaking changes).
- Lessons chỉ-theory (exercises=[]) = valid; no migration risk.
- StudentModel optional (chưa placement → no doc).
- LearnerPath lazy-create on first enroll.
- Demo-foundation archive (old learner xong v1 vẫn OK; new learner chưa demo).
