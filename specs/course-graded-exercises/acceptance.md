# Acceptance Criteria: Quiz & Flashcard Graded Exercises

- **Spec:** `specs/course-graded-exercises/spec.md`
- **Ngày:** 2026-08-03

## Tiêu chí chức năng

### AC-1: Learner lấy quiz items (answers stripped)

- **Given:** Learner enrolled vào course published, course có lesson với quiz exercise (type='quiz', refId=LessonQuiz.code), LessonQuiz status='published', có resolvedItemIds
- **When:** POST `/courses/:slug/quiz/items` body `{phaseKey, moduleKey, lessonKey}`
- **Then:**
  - HTTP 200
  - Response: `{ quizConfigId, code, passThreshold, items[], stimuli[] }`
  - items[] chứa chỉ active + objective-type questions từ resolvedItemIds, KHÔNG chứa answerKey/acceptedVariants/transcript
  - stimuli[] chứa testlet mapping (id, kind='passage'|'audio', skill, body, media.audioUrl) cho mỗi unique stimulusId, CHỈ active stimuli
  - items[] giữ nguyên thứ tự resolvedItemIds (reproducible)
  - passThreshold = LessonQuiz.passThreshold (mặc định 0.8)

### AC-2: Learner submit quiz, graded per item, best-score tracking

- **Given:** Learner fetch quiz items (AC-1), học xong, submit
- **When:** POST `/courses/:slug/quiz/submit` body `{phaseKey, moduleKey, lessonKey, answers: [{questionId, response}, ...]}`
- **Then:**
  - HTTP 200
  - Response: `{ score, passThreshold, passed, perItem: [{questionId, correct: 0|1}, ...], progress }`
  - score = sum(correct) / resolvedItemIds.length (mẫu số cố định, KHÔNG answer.length)
  - passed = score >= passThreshold
  - perItem[i].correct = result of gradeObjective (item i graded against answerKey)
  - perItem INCLUDES all resolved items even if not answered (unanswered=incorrect)
  - QuizAttempt row created (userId, courseId, lessonPathKey, quizConfigId, score, passed, perItem, timestamp)
  - One learning_event per resolved item where skill is set (source='quiz', refId=attempt._id, questionId, skill, subskill, itemCefr, correct, countsTowardMastery=true)
  - Prior quiz attempts' learning_events DELETED (latest-submission dedup)
  - progress = recordLesson result (lesson marked as passed/mastered/in_progress based on all exercises)

### AC-3: Learner lấy flashcard cards

- **Given:** Learner enrolled, lesson có flashcard exercise (type='flashcard', refId=FlashcardDeck._id), deck là system deck, >= MIN_FLASHCARD_CARDS cards
- **When:** POST `/courses/:slug/flashcard/cards` body `{phaseKey, moduleKey, lessonKey}`
- **Then:**
  - HTTP 200
  - Response: `{ deck: {deckId}, cards: [{cardId, term, meaning, ipa?, example?, imageUrl?, audioUrl?}, ...] }`
  - cards = all FlashcardCard docs in deck (term, meaning, media stripped safe for client)
  - No answerKey, no SRS logic exposed

### AC-4: Learner complete flashcard

- **Given:** Learner review all cards in deck
- **When:** POST `/courses/:slug/flashcard/complete` body `{phaseKey, moduleKey, lessonKey, reviewedCardIds: [cardId1, cardId2, ...]}`
- **Then:**
  - HTTP 200
  - Response: `{ completed: true|false, progress }`
  - FlashcardAttempt upserted (userId, courseId, lessonPathKey, deckId, reviewedCardIds)
  - completed = true nếu reviewedCardIds.length >= deck.cardIds.length (all cards reviewed)
  - lesson progress updated via recordLesson
  - NO learning_events created (flashcard does NOT feed mastery)

### AC-5: Admin CRUD LessonQuiz

- **Given:** Admin user có permission 'lesson-quiz:write'
- **When:**
  - POST `/api/admin/lesson-quizzes` body `{code, title, source: 'curated', itemIds: [qid1, qid2, ...], passThreshold: 0.8}`
  - GET `/api/admin/lesson-quizzes` (list)
  - PUT `/api/admin/lesson-quizzes/:id` body `{itemIds: [new items], passThreshold: 0.75}`
  - DELETE `/api/admin/lesson-quizzes/:id` (soft delete → status: 'draft')
- **Then:**
  - CREATE: LessonQuiz created status='draft', createdBy=user.id, centerId=user.centerId (or null if platform admin)
  - UPDATE: itemIds/passThreshold updated, status remains 'draft' (cannot edit if published)
  - DELETE: status set to 'draft' (soft delete convention)
  - LIST shows code, title, source, status, passThreshold, createdAt

### AC-6: Admin publish quiz (freeze + validate)

- **Given:** LessonQuiz status='draft', has itemIds
- **When:** POST `/api/admin/lesson-quizzes/:id/publish`
- **Then:**
  - **Success path:**
    - HTTP 200
    - LessonQuiz.resolvedItemIds = LessonQuiz.itemIds (frozen)
    - LessonQuiz.status = 'published'
    - Response: updated LessonQuiz doc
  - **Validation: all-or-nothing**
    - EVERY itemId must exist + status='active'
    - EVERY itemId.itemType ∈ COURSE_GRADABLE_TYPES (objective, NOT matching)
    - EVERY itemId.answerKey not null (must have answer key)
    - EVERY labeling item: answerKey must be array of strings (renderable per-prompt) — if not, reject with QUIZ_ANSWERS_INVALID
    - IF ANY item fails: HTTP 400, error message lists which itemId failed & why (specific)
    - Publish does NOT happen (all-or-nothing)
  - CenterId-scoped query if tenant-scoped

### AC-7: Admin guard: block edit of published-referenced questions

- **Given:** LessonQuiz status='published', resolvedItemIds=[qid1, qid2, ...]
- **When:** Admin attempt to edit/retire/delete qid1 via AssessmentQuestion admin endpoint
- **Then:**
  - HTTP 409 CONFLICT
  - error code: ITEM_REFERENCED_BY_PUBLISHED_QUIZ
  - message: "Câu hỏi đang được dùng trong một quiz đã publish — không thể sửa/xoá"
  - Item NOT modified
  - Draft quizzes referencing qid1 are NOT blocked (can still edit draft config)

### AC-8: Group-items endpoint reconstruct testlet mapping

- **Given:** Admin assembling quiz itemIds via question-bank (curating)
- **When:** POST `/api/admin/question-bank/group-items` body `{itemIds: [qid1, qid2, qid3, ...]}`
- **Then:**
  - HTTP 200
  - Response: `{ groups: [{stimulusId, items: [qid1, qid2]}, {stimulusId: null, items: [qid3]}] }`
  - groups map question-bank items back to their testlet (stimulus) memberships
  - Admin can see: "this set will render as 1 listening testlet (qid1, qid2) + 1 standalone grammar (qid3)"
  - Helps admin understand selectTestletAware impact

### AC-9: Testlet-whole selection

- **Given:** Question bank has items with stimulusId mapping (testlet = 1 stimulus + 3 items)
- **When:** selectTestletAware(rows, count=5, {overflow: false})
  - Input rows: [{_id: q1, stimulusId: s1}, {_id: q2, stimulusId: s1}, {_id: q3, stimulusId: s1}, {_id: q4, stimulusId: null}, {_id: q5, stimulusId: null}]
  - count=5 means "up to 5 items, but never split testlets"
- **Then:**
  - Output: [q1, q2, q3, q4, q5]
  - Explanation: testlet(s1) = 3 items taken whole (< 5), then q4 (< 5 total), then q5 (= 5, budget reached)
- **When:** selectTestletAware(rows, count=2, {overflow: false})
- **Then:**
  - Output: [q1, q2, q3]
  - Explanation: testlet(s1) is 3 items; even though 3 > 2 budget, whole unit is taken (must not split testlet); next unit q4 would exceed, so skip; q5 same
- **When:** selectTestletAware(rows, count=3.5, {overflow: true})
- **Then:**
  - Output: [q1, q2, q3, q4, q5] or [q1, q2, q3, q4] — at-least mode: keep adding whole units until >= count; stop once first unit crosses count line
  - (exact behavior depends on RowOrder, but never splits testlet)

### AC-10: Best-score monotonic progress

- **Given:** Learner attempt 1 quiz (score=0.5), then attempt 2 (score=0.6)
- **When:** Attempt 2 submitted successfully
- **Then:**
  - QuizAttempt records both (2 rows, same userId/courseId/lessonPathKey)
  - Learning_events from attempt 1 DELETED before attempt 2's events inserted
  - Latest mastery compute uses attempt 2's events (not attempt 1)
  - Lesson progress reads QuizAttempt.passed (true if score >= threshold) → remains consistent
  - **Learner never regresses:** if attempt 1 was passed, redefining to new events doesn't retrograde completion (recordLesson handles monotonic)

---

## Tiêu chí lỗi / edge case

### AC-E1: Not enrolled

- **Given:** User not enrolled in course, or course not published
- **When:** POST `/courses/:slug/quiz/items|submit` or `/flashcard/*`
- **Then:** HTTP 409, error code NOT_ENROLLED

### AC-E2: Quiz config not found

- **Given:** Lesson has no quiz exercise, or quiz exercise refId invalid/no LessonQuiz doc
- **When:** POST `/courses/:slug/quiz/items|submit`
- **Then:** HTTP 404, error code QUIZ_CONFIG_NOT_FOUND

### AC-E3: Quiz not published

- **Given:** LessonQuiz status='draft'
- **When:** POST `/courses/:slug/quiz/items|submit`
- **Then:** HTTP 409, error code QUIZ_CONFIG_NOT_PUBLISHED

### AC-E4: Empty resolved set (publish defect)

- **Given:** LessonQuiz published but resolvedItemIds=[] (should never happen, publish validation guards, but defensive check)
- **When:** POST `/courses/:slug/quiz/submit`
- **Then:** HTTP 409, error code QUIZ_CONFIG_NOT_PUBLISHED (same as draft guard)

### AC-E5: Invalid quiz answers payload

- **Given:** POST `/courses/:slug/quiz/submit` body `{answers: "not an array"}`
- **When:** submit
- **Then:** HTTP 400, error code QUIZ_ANSWERS_INVALID

### AC-E6: Flashcard deck not found / too small

- **Given:** Lesson flashcard refId=invalid, or deck.ownerType != 'system', or cards.length < MIN_FLASHCARD_CARDS (2)
- **When:** POST `/courses/:slug/flashcard/cards|complete`
- **Then:** HTTP 404 or 400, error code FLASHCARD_DECK_NOT_FOUND or FLASHCARD_QUIZ_TOO_SMALL

### AC-E7: Invalid reviewedCardIds

- **Given:** POST `/courses/:slug/flashcard/complete` body `{reviewedCardIds: "not an array"}`
- **When:** complete
- **Then:** HTTP 400, error code QUIZ_ANSWERS_INVALID

### AC-E8: Publish with non-gradable item

- **Given:** LessonQuiz itemIds=[qid1 (matching), qid2 (mcq)]
- **When:** POST `/api/admin/lesson-quizzes/:id/publish`
- **Then:** HTTP 400, specific error message: "Quiz chỉ nhận câu tự chấm được (câu [qid1] là matching)"

### AC-E9: Publish with keyless item

- **Given:** LessonQuiz itemIds=[qid1 (mcq, answerKey=null)]
- **When:** POST `/api/admin/lesson-quizzes/:id/publish`
- **Then:** HTTP 400: "Câu [qid1] chưa có đáp án (answerKey)"

### AC-E10: Publish with non-renderable labeling key

- **Given:** LessonQuiz itemIds=[qid1 (labeling, answerKey=[0, 1, "wrong"] — not per-prompt strings)]
- **When:** POST `/api/admin/lesson-quizzes/:id/publish`
- **Then:** HTTP 400: "Câu labeling [qid1] có đáp án không hợp lệ (cần mảng nhãn dạng chuỗi, đủ mỗi prompt)"

### AC-E11: Unanswered questions grade as incorrect

- **Given:** Quiz has 5 items, learner submits answers for only 3 items
- **When:** POST `/courses/:slug/quiz/submit` body `{answers: [{questionId: q1, response: 0}, {q2: 1}, {q3: 0}]}`
- **Then:**
  - perItem includes all 5 items
  - Items q4, q5 have correct: 0 (unanswered → incorrect)
  - score = correctCount / 5 (denominator = 5, not 3)

---

## Tiêu chí phi chức năng

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| **Bảo mật** | Endpoint /quiz/*, /flashcard/* require verifyToken + enrollment gate | Check middleware order in routes |
| **Bảo mật** | answerKey NEVER sent to learner (select: false on fetch) | Read quiz.service.resolveQuizItems & fetchGradableItems — keys only fetched when needed by gradeQuiz (server-side) |
| **Bảo mật** | Transcript (stimulus audio) not emitted in /items response | Verify buildTestletStimuli does NOT select transcript field |
| **Bảo mật** | Admin permission 'lesson-quiz:write' required for publish/edit | Check _crud.factory permission check on PATCH/POST/DELETE |
| **Bảo mật** | Admin guard blocks question mutation if referenced by published quiz | Read assertItemNotInPublishedQuiz logic; test: publish quiz, try to edit question, expect 409 |
| **Hiệu năng** | resolveQuizItems query is indexed on {_id, status, itemType} | Check AssessmentQuestion.find() + indexes on model |
| **Hiệu năng** | Learning events batch-insert (no loop insert) | Read adaptive.ingestQuizAttempt → should use insertMany |

---

## Ngoài phạm vi kiểm thử

- **Checkpoint grading**: Checkpoint dùng chung gradeObjective nhưng là boundary-level (phase/course), không lesson-level; test riêng ở specs/course-checkpoints-band-up/
- **Writing/Speaking engines**: Quản lý riêng ở specs/exercise-engines/; không overlap với course-graded-exercises
- **Learner unlock / gating progression**: Phase 1 không gate bài học tiếp theo dựa trên quiz pass; cơ chế unlock thuộc specs/lesson-runtime/
- **Dynamic spec-mode**: Plan gốc đề cập sampling qua `spec: {skill, cefr, count}` — **đã xoá**, curate-only ở phase 1
- **Re-publish / versioning**: Chỉ có create-new-draft-or-soft-delete; edit resolved set của published quiz ngoài phạm vi
- **Global SRS for flashcard**: Flashcard course-side standalone; SRS clone-based ngoài scope
- **Upload media / flashcard source**: LessonQuiz + FlashcardDeck bound tới existing question bank + flashcard deck; ingestion là ngoài scope (handled by course-content-import, flashcard-import, etc.)

