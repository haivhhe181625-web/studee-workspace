# Design: Quiz & Flashcard Graded Exercises

- **Spec:** `specs/course-graded-exercises/spec.md`
- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ code)
- **Trạng thái:** Đã build (live code)
- **ADR liên quan:** docs/adr/0001-admin-api-migration.md (AdminJS remove, REST replace)

---

## 1. Tóm tắt kiến trúc

Quiz & flashcard là **2 loại graded exercise tích hợp vào lesson** thông qua `lesson.exercises[]` entries (type='quiz'|'flashcard', refId trỏ tới LessonQuiz code hay FlashcardDeck id).

**Kiến trúc phân lớp:**
- **Learner surface:** `/api/courses/:slug/quiz|flashcard/*` routes, gated bởi enrollment + verifyToken
- **Grading domain:** `quiz.grading.js` (pure, side-effect-free, unit-testable) — cô lập logic chấm điểm từ route orchestration
- **Service layer:** `quiz.service.js` — load quiz/deck, resolve items, execute grading, persist attempt, feed mastery + progress
- **Admin surface:** `/api/admin/lesson-quizzes` (REST CRUD via factory) + group-items endpoint (question-bank helper)
- **Mastery integration:** Quiz attempts → LearningEvent rows → adaptive.ingestQuizAttempt → StudentModel update

**Điểm khác biệt exam-side (checkpoint):**
- Checkpoint = boundary-level (phase/course gating); course-graded-exercises = lesson-level (exercise binding)
- Checkpoint có proctoring logic; course-exercises là self-service learner
- Cả hai dùng chung `gradeObjective` + testlet stimuli

**Điểm khác biệt exercise-engines (phase 4):**
- Exercise-engines = QuizSet (new model) + WritingTask + SpeakingTask; course-graded = LessonQuiz (curated from bank)
- Exercise-engines cover new writing/speaking engines; course-graded cover reading/listening via existing bank
- Coexist: khác `lesson.exercises[]` type + refId model

---

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| **Quiz model** | LessonQuiz (new, curated itemIds binding) | Lightweight, audit trail (createdBy, timestamps), center-scoped ready | N/A |
| **Grading source** | AssessmentQuestion bank (reuse existing) | 558 active items, no duplicate content store | Must maintain question bank -> quiz contract |
| **Gradable types** | COURSE_GRADABLE_TYPES (OBJECTIVE_TYPES - matching) | matching FE response (object) ≠ grader expect (array) → would always grade 0 course-side; proctored exam handles matching separately | Exclude matching (not a loss for course-side) |
| **Denominator** | resolvedItemIds.length (frozen at publish) | Anti-cheat: learner can't inflate score by skipping hard questions; reproducible grading across retakes | Learner sees fewer items if one retires later → score unchanged (consistent) |
| **Passing threshold** | 0.8 fixed per LessonQuiz (settable by admin) | Simple, explicit, auditable; prevents edge case of different thresholds across attempts | Inflexible; all learners same bar |
| **Flashcard model** | FlashcardAttempt (self-contained, no SRS) | Lightweight, distinct from global clone-based SRS; completion = review all cards | No adaptive scheduling for flashcard; learner must review every card regardless of confidence |
| **Flashcard mastery** | Does NOT feed learning_events | Flashcard = practice/review, not assessment; would double-count if both quiz + flashcard on lesson | Flashcard progress not reflected in competency |
| **Re-attempt strategy** | Best-score-wins; prior attempt events deleted | Matches existing lesson progress monotonic rule; learner can retry without penalty to mastery inflate | Only latest event state in DB; prior attempts still queryable via QuizAttempt table |
| **Question mutation guard** | Block edit/retire/delete if referenced by published quiz (policy a from review) | Prevents silent grading corruption (wrong itemType, missing answerKey, retired item → denominator mismatch) | Admin must create new draft quiz if item changes; publish another quiz |
| **Testlet selection** | selectTestletAware (whole-unit only) | Audio/passage + questions must render as cohesive testlet; never split stimulus | Cannot hand-pick 2 out of 3 items from a 3-item testlet; may waste budget if testlet too big |
| **Admin authoring** | Curate-only (no spec-mode) | Phase 1 simplicity; admin hand-picks items/testlets via group-items helper | No dynamic sampling by skill/cefr; future extension to spec-mode possible |
| **Publish validation** | All-or-nothing (reject if any item invalid) | Safety: ensures quiz is grading-ready at publish; no silent defects surface at learner submit | Admin must fix all items before publish (no partial save) |
| **Soft delete** | status='draft' (soft-delete convention matches question-bank) | Aligns with existing audit/recovery model; hard-deleted LessonQuiz = undefined behavior | status='draft' ≠ truly deleted; soft-delete scan needed when querying active quizzes |
| **Testlet grouping endpoint** | POST group-items on question-bank resource | Single source of truth for stimulus membership; avoids duplication | Requires round-trip to group-items; admin UI must call it during curation flow |
| **Learning event scoping** | Per refId (QuizAttempt._id), latest-submission dedup | Clean separation: multiple quizzes referencing same question → each quiz gets own events; unlimited retakes don't multiply subskill sampleCount | Need careful handling of event deletion before new insert (soft-delete potential race) |

---

## 3. Data model

See `data-model.md` for detailed schema. Summary here:

| Entity | Collection | Key fields | Purpose |
|---|---|---|---|
| **LessonQuiz** | lesson_quizzes | code, status, itemIds, resolvedItemIds, passThreshold | Quiz config (draft/published) |
| **QuizAttempt** | quiz_attempts | userId, courseId, lessonPathKey, quizConfigId, score, passed | Learner attempt record |
| **FlashcardAttempt** | flashcard_attempts | userId, courseId, lessonPathKey, deckId, reviewedCardIds | Learner flashcard session (upsert) |
| **AssessmentQuestion** | assessment_questions | _id, itemType, skill, subskill, cefrLevel, answerKey (select:false) | Shared question bank (reused) |
| **AssessmentStimulus** | assessment_stimuli | _id, kind, skill, content, audioUrl | Testlet stimulus (audio/passage) (reused) |
| **LearningEvent** | learning_events | source='quiz', refId (QuizAttempt._id), questionId, skill, subskill, correct | Mastery feed (ingested by adaptive) |

---

## 4. Luồng dữ liệu

### Learner Quiz Flow

```
learner@web → POST /api/courses/:slug/quiz/items?phaseKey=p1&moduleKey=m1&lessonKey=l1
  (enrollment gate, verifyToken)
  ↓
quiz.service.getLessonQuizItems
  ├─ loadLesson (course + lesson node locate)
  ├─ loadPublishedQuiz (LessonQuiz fetch, status check)
  ├─ resolveQuizItems (fetch active/objective items from resolvedItemIds, build client-safe response)
  ├─ buildTestletStimuli (group items by stimulusId, fetch stimuli, HMAC audioUrl signing)
  ↓
response: {quizConfigId, code, passThreshold, items[], stimuli[]}
  (answerKey/transcript/acceptedVariants NEVER included)
```

```
learner@web → POST /api/courses/:slug/quiz/submit body={answers, phaseKey, ...}
  ↓
quiz.service.submitLessonQuiz
  ├─ loadLesson + loadPublishedQuiz
  ├─ fetchGradableItems (resolve frozen set, WITH answerKey for grading only)
  ├─ gradeQuiz (pure: perItem grading via gradeObjective, score = correctCount/resolvedItems.length)
  ├─ QuizAttempt.create (persist attempt)
  ├─ [Mastery pipeline — best-effort]
  │  ├─ LearningEvent.deleteMany (prior attempt events drop — latest-submission dedup)
  │  ├─ LearningEvent.insertMany (one per item, skill/subskill/correct from grading)
  │  └─ adaptive.ingestQuizAttempt (recompute StudentModel)
  └─ recordLesson (lesson progress update)
  ↓
response: {score, passThreshold, passed, perItem[], progress}
```

### Learner Flashcard Flow

```
learner@web → GET /api/courses/:slug/flashcard/cards?phaseKey&moduleKey&lessonKey
  ↓
quiz.service.getFlashcardCards
  ├─ loadLesson
  ├─ loadSystemDeck (fetch deck, validate ownerType='system')
  ├─ FlashcardCard.find (fetch all cards, strip media safe)
  ↓
response: {deck: {deckId}, cards: [{cardId, term, meaning, ...}]}
```

```
learner@web → POST /api/courses/:slug/flashcard/complete body={reviewedCardIds}
  ↓
quiz.service.completeFlashcard
  ├─ loadLesson + loadSystemDeck
  ├─ FlashcardAttempt.updateOne (upsert latest pass)
  ├─ recordLesson (mark lesson completed)
  ↓
response: {completed: boolean, progress}
  (NO learning_events created; flashcard is self-contained exercise, no mastery feed)
```

### Admin Quiz Authoring Flow

```
admin@portal → POST /api/admin/lesson-quizzes body={code, title, itemIds}
  ↓
_crud.factory (REST CRUD config-based)
  ├─ Validation (no custom logic, just data)
  ├─ LessonQuiz.create (status='draft', createdBy=user.id, centerId from scope)
  ↓
response: created doc
```

```
admin@portal → POST /api/admin/lesson-quizzes/:id/publish
  ↓
quiz.service.publishLessonQuiz (action handler)
  ├─ LessonQuiz fetch
  ├─ AssessmentQuestion.find (itemIds, status=active, fetch WITH answerKey for validation)
  ├─ FOR EACH item:
  │  ├─ Check itemType ∈ COURSE_GRADABLE_TYPES
  │  ├─ Check answerKey !== null
  │  └─ If labeling, check renderable (per-prompt string array)
  ├─ IF any fail: throw 400 (all-or-nothing, no publish)
  ├─ Else: LessonQuiz.resolvedItemIds = itemIds (freeze)
  ├─ LessonQuiz.status = 'published'
  ├─ LessonQuiz.save()
  ↓
response: updated doc
  (freeze is idempotent; re-publish same quiz = no-op)
```

### Admin Guard: Question Mutation

```
admin@portal → PATCH /api/admin/assessment-questions/:qid body={...}
  ↓
quiz.service.assertItemNotInPublishedQuiz (called from admin-api question route before delete/edit)
  ├─ LessonQuiz.findOne({status: 'published', resolvedItemIds: {$in: [qid]}})
  ├─ IF found: throw 409 ITEM_REFERENCED_BY_PUBLISHED_QUIZ
  └─ Else: allow mutation
```

---

## 5. Contracts

Listed in `contracts/` subdirectory:

- `contracts/learner-quiz-items-and-submit.md` — POST /quiz/items, POST /quiz/submit
- `contracts/learner-flashcard-cards-and-complete.md` — GET /flashcard/cards, POST /flashcard/complete
- `contracts/admin-lesson-quiz-crud-and-publish.md` — REST CRUD + publish action
- `contracts/admin-group-items-testlet-helper.md` — POST /question-bank/group-items (reconstruct stimulus mapping)

---

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/src/modules/quiz/lesson-quiz.model.js` | LessonQuiz schema (code, title, itemIds, resolvedItemIds, passThreshold, status) | Tạo (Phase 1) |
| `services/api/src/modules/quiz/quiz-attempt.model.js` | QuizAttempt schema (userId, courseId, lessonPathKey, quizConfigId, score, passed, perItem) | Tạo (Phase 2) |
| `services/api/src/modules/quiz/flashcard-attempt.model.js` | FlashcardAttempt schema (userId, courseId, lessonPathKey, deckId, reviewedCardIds) | Tạo (Phase 2) |
| `services/api/src/modules/quiz/quiz.grading.js` | Pure grading logic (gradeQuiz, resolveQuizItems, fetchGradableItems, COURSE_GRADABLE_TYPES) | Tạo (Phase 1) |
| `services/api/src/modules/quiz/quiz.service.js` | Orchestration (getLessonQuizItems, submitLessonQuiz, getFlashcardCards, completeFlashcard, publishLessonQuiz, assertItemNotInPublishedQuiz, selectTestletAware) | Tạo (Phase 2) |
| `services/api/src/modules/quiz/quiz.controller.js` | Route handlers (light, delegate to service) | Tạo (Phase 1) |
| `services/api/src/modules/quiz/quiz.routes.js` | Route mount (`/courses/:slug/quiz/*`, `/courses/:slug/flashcard/*`) | Tạo (Phase 1) |
| `services/api/src/admin-api/resources/lesson-quiz.admin.js` | Admin resource (CRUD config + publish action) | Tạo (Phase 4) |
| `services/api/src/modules/adaptive/adaptive.service.js` | ingestQuizAttempt (stub → fill Phase 2) | Sửa (fill method) |
| `services/api/src/modules/course-content/course-content.progress.service.js` | evaluateStatus (expand to handle quiz/flashcard exercises) | Sửa (xử lý quiz+flashcard status + mastery) |
| `services/api/src/__tests__/quiz*.test.js` | Unit + API tests for quiz grading, publish guard, routes | Tạo |
| `web/src/components/features/learn/quiz-inline-runner.tsx` | Quiz player (question card, submit, retry) | Tạo (Phase 3) |
| `web/src/components/features/learn/flashcard-inline-runner.tsx` | Flashcard player (card flip, review, complete) | Tạo (Phase 3) |
| `web/src/hooks/use-lesson-quiz.ts` | React hook (fetch quiz items, submit, track state) | Tạo (Phase 3) |
| `web/src/services/quiz.service.ts` | Quiz API client (methods for /items, /submit, /flashcard/*) | Tạo (Phase 3) |
| `web/src/types/course.types.ts` | TypeScript types (Quiz, QuizAttempt, Flashcard, etc.) | Tạo (Phase 1) |
| `admin/src/pages/lesson-quizzes.tsx` | Admin UI (CRUD, curate, publish) | Tạo (Phase 4) |

---

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code | Xử lý |
|---|---|---|---|
| User không enrolled | 409 | NOT_ENROLLED | assertEnrolled middleware throw |
| Bài học không gắn quiz | 404 | QUIZ_CONFIG_NOT_FOUND | findExercise returns null |
| LessonQuiz không tìm thấy | 404 | QUIZ_CONFIG_NOT_FOUND | LessonQuiz.findOne miss |
| Quiz draft (chưa publish) | 409 | QUIZ_CONFIG_NOT_PUBLISHED | Check lq.status !== 'published' |
| Quiz published nhưng resolvedItemIds empty | 409 | QUIZ_CONFIG_NOT_PUBLISHED | Defensive check (should never happen due to publish validation) |
| Answers không phải array | 400 | QUIZ_ANSWERS_INVALID | typeof check: !Array.isArray |
| Publish: item not found / not active | 400 | QUIZ_ANSWERS_INVALID | Specific error message: "Câu hỏi XYZ không khả dụng" |
| Publish: item không objective type | 400 | QUIZ_ANSWERS_INVALID | Specific: "Quiz chỉ nhận câu tự chấm được (câu XYZ là TYPE)" |
| Publish: item no answerKey | 400 | QUIZ_ANSWERS_INVALID | Specific: "Câu XYZ chưa có đáp án" |
| Publish: labeling non-renderable key | 400 | QUIZ_ANSWERS_INVALID | Specific: "Câu labeling XYZ có đáp án không hợp lệ" |
| Edit question referenced by published quiz | 409 | ITEM_REFERENCED_BY_PUBLISHED_QUIZ | assertItemNotInPublishedQuiz throw |
| Flashcard deck not found / not system | 404 | FLASHCARD_DECK_NOT_FOUND | loadSystemDeck miss or ownerType check |
| Flashcard too few cards | 400 | FLASHCARD_QUIZ_TOO_SMALL | cards.length < MIN_FLASHCARD_CARDS (2) |
| reviewedCardIds not array | 400 | QUIZ_ANSWERS_INVALID | typeof check |

---

## 8. Bảo mật & quyền

| Nhu cầu | Giải pháp |
|---|---|
| Learner endpoints require auth + enrollment | `verifyToken` + `assertEnrolled` middleware |
| answerKey never sent to learner | `AssessmentQuestion.select('-answerKey')` on client-safe fetch; grading fetch uses `+answerKey` server-side only |
| Transcript audio never sent to learner | buildTestletStimuli does NOT select transcript field |
| Admin publish requires permission | `'lesson-quiz:write'` checked by _crud.factory action handler |
| Admin cannot edit/delete questions in published quiz | `assertItemNotInPublishedQuiz` called from admin-api question route before mutation |
| Quiz config createdBy stamping | `createDefaults: (req) => ({createdBy: req.user.id})` in lesson-quiz.admin.js |
| Tenant scoping (future, Phase 1 = null) | `tenantScoped: true` on _crud.factory; centerId auto-stamped from req.user.centerId |
| Learning events source tracking | source='quiz' (vs 'ipa', 'talk', 'lesson', etc.) → audit trail of which feature contributed to mastery |

---

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

| Risk | Mitigation |
|---|---|
| **Frozen denominator vs retired question** — if question retires after publish, resolvedItemIds still refs it → fetchGradableItems returns found:false → graded as incorrect → denominator unchanged → learner may see fewer items but score stays consistent. Is this OK? | **Mitigated by guard:** H1 policy (a) blocks question retire if it's in published quiz. If policy changes to (b) drop found:false from denominator, need careful recompute. Policy (a) is safer: freeze is the source of truth. |
| **Best-score-wins + mastery inflation** — if learner retries quiz, prior events deleted, latest inserted. What if latest attempt is worse? | **Accepted trade-off:** monotonic lesson progress (recordLesson) + best-score-tracking in QuizAttempt table ensure learner never regresses. Mastery events reflect latest submission (one event set per lesson-quiz). |
| **Race: event deletion before insert** — atomic guarantee? | **Mitigated:** LearningEvent.deleteMany + insertMany not in transaction (MongoDB would need multi-doc ACID). If failure between delete and insert, events lost (soft loss). Mastery recompute is idempotent (aggregation re-runs). Acceptable risk: use best-effort try/catch in submitLessonQuiz. |
| **Testlet split budget** — selectTestletAware may skip a small unit to avoid exceeding budget. Admin sees "5 items selected" but gets 3 testlet (larger). Is UX confusing? | **Mitigated:** group-items endpoint shows testlet mapping → admin can adjust budget knowing testlet sizes. UI should show "3 items from 1 testlet + 2 standalone". |
| **Flashcard vs global SRS confusion** — learner learns there's a global flashcard app but lesson flashcard is different (course-side, no SRS). | **Mitigation:** clear naming in UI ("Lesson Vocabulary Practice" vs "Flashcard App" elsewhere); no cross-linking. Separate data store (FlashcardAttempt vs global SRS). |
| **Admin can't re-publish** — once published, resolvedItemIds frozen. If admin wants to add items, must soft-delete + create new quiz. Workflow OK? | **Accepted:** keeps freeze simple, reproducible. Versioning/re-publish is Phase 2+. Current workflow: create new config, publish new. |

---

## 10. Testing strategy

### Unit tests

- **quiz.grading.js** — pure functions, no DB/HTTP
  - gradeQuiz: correctness per itemType (mcq, cloze, labeling, etc.), denominator invariant, perItem accuracy
  - resolveQuizItems: client-safe response (no keys), testlet grouping (stimulusId), active/objective filter
  - fetchGradableItems: answerKey inclusion, found:false handling, skill/subskill extraction
  - selectTestletAware: testlet whole-unit, budget handling, overflow mode
- **quiz.service publishLessonQuiz** — publish validation, error messages
  - Publish success: freeze, status flip
  - Reject non-gradable itemType, missing answerKey, non-renderable labeling
  - All-or-nothing (no partial publish)
- **quiz.service assertItemNotInPublishedQuiz** — guard logic
  - Block if item in published quiz.resolvedItemIds
  - Allow if quiz draft
  - Allow if item in unreferenced quiz

### API integration tests

- **Quiz items fetch:** enrollment gate, quiz not found, quiz not published, items response structure
- **Quiz submit:** grading correctness, perItem size = resolvedItemIds.length, learning events creation, prior event deletion (latest-submission dedup), progress update
- **Flashcard cards fetch:** deck not found, too small, cards response
- **Flashcard complete:** upsert, completion logic (reviewedCardIds >= deck), progress update
- **Admin publish:** validation errors, success case, publish idempotency
- **Admin guard:** block mutation of published-referenced question

### E2E tests

- Full learner flow: enroll → fetch items → submit → progress updates
- Best-score-wins: attempt 1 (0.5) → attempt 2 (0.8) → best tracked, prior events dropped
- Testlet coherence: audio passage + questions render as one block

---

## 11. Rollout / cross-repo sequencing

**Multi-repo execution order (Phase-based):**

1. **Phase 1 (Contract):** exe-api (models, enums, error codes, route stubs) + exe-web (types, service stubs)
   - Merge to base on both repos BEFORE branching tracks
2. **Phase 2 (Track A: grading):** exe-api only
   - Branches off base → implement grading, service, routes, tests
   - Merges back to base when done
3. **Phase 3 (Track B: runners):** exe-web only
   - Branches off Phase 2 base → implement quiz/flashcard players, hooks
   - Merges back to base when done
4. **Phase 4 (Track C: admin):** exe-api + exe-admin
   - api: lesson-quiz.admin.js resource
   - admin: CRUD UI pages
   - Both branch off base → merge back
5. **Phase 5 (Track D: seed):** exe-api only
   - Populate sample course → bind LessonQuiz + system deck
6. **Phase 6 (Integration E2E):** All repos
   - Merge all tracks into one integrated branch
   - E2E tests wire learner → grading → mastery

**Deployment sequence (if staged):**
1. Deploy exe-api (grading + admin routes)
2. Deploy exe-admin (authoring UI)
3. Deploy exe-web (learner UI)

(All must be present for E2E to work.)

---

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion | Được đáp ứng bởi |
|---|---|
| AC-1: Learner fetch quiz items | quiz.service.getLessonQuizItems + resolveQuizItems (FE route POST /quiz/items) |
| AC-2: Submit & grade | quiz.service.submitLessonQuiz + gradeQuiz + QuizAttempt.create + learning_events + adaptive.ingestQuizAttempt |
| AC-3: Flashcard fetch & complete | quiz.service.getFlashcardCards + completeFlashcard + FlashcardAttempt.updateOne |
| AC-4: Admin CRUD | _crud.factory (lesson-quiz.admin.js config) |
| AC-5: Admin publish | quiz.service.publishLessonQuiz (validation + freeze) |
| AC-6: Admin guard published items | quiz.service.assertItemNotInPublishedQuiz (block mutation) |
| AC-7: Group-items testlet helper | POST /admin/question-bank/group-items (endpoint on question-bank resource) |
| AC-8: Testlet-whole selection | quiz.service.selectTestletAware (never split testlet) |
| AC-9: Best-score tracking | Quiz attempt re-do → prior events deleted, latest inserted (latest-submission dedup) |
| AC-10: Mastery ingestion | adaptive.ingestQuizAttempt (learning_events → StudentModel) |

