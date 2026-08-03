# Data Model: Quiz & Flashcard Graded Exercises

- **Spec:** `specs/course-graded-exercises/spec.md`
- **Design:** `specs/course-graded-exercises/design.md`
- **Ngày:** 2026-08-03

---

## Overview

Five entities:
1. **LessonQuiz** — admin-authored quiz config (curated item selection, pass threshold, freeze point)
2. **QuizAttempt** — learner quiz submission (score, passed, per-item results)
3. **FlashcardAttempt** — learner flashcard completion (reviewed cards)
4. **LearningEvent** — mastery-feeding event (quiz attempts → subskill learning record)
5. **AssessmentQuestion** + **AssessmentStimulus** — reused from existing question bank

---

## Entity: LessonQuiz

**Collection:** `lesson_quizzes`

**Purpose:** Represents a quiz bound to a lesson. Authored once (draft), published once (frozen). A lesson's `lesson.exercises[].type='quiz'` points to one LessonQuiz via `refId` (the LessonQuiz.code).

**Schema:**

```javascript
{
  _id: ObjectId,
  
  // Identifiers
  code: String (required, unique, trim, index),
    // e.g., "quiz_listening_01", "quiz_reading_unit_1"
    // Mirrors IPA.code convention; used as lesson.exercises[].refId
  
  title: String (required, trim),
    // e.g., "Listening Comprehension — IELTS Mock Test"
  
  // Source & itemization
  source: String (enum: ['curated'], default: 'curated'),
    // Only 'curated' supported in Phase 1 (hand-picked items)
    // Future: 'spec' for dynamic sampling (not Phase 1)
  
  itemIds: [ObjectId] (ref: 'AssessmentQuestion', default: []),
    // Admin-curated list of question _ids (can include testlet multiples)
    // MUTABLE when status='draft'
    // READ-ONLY copy made to resolvedItemIds at publish
  
  resolvedItemIds: [ObjectId] (ref: 'AssessmentQuestion', default: []),
    // FROZEN copy of itemIds at publish time
    // This is the ONLY set graded/displayed to learner
    // IMMUTABLE once published (ensures reproducible grading)
    // Denominator for score: correctCount / resolvedItemIds.length
  
  // Grading parameters
  passThreshold: Number (default: 0.8, min: 0, max: 1),
    // E.g., 0.8 means learner must score >= 0.8 to pass
    // Settable by admin (admin can adjust before publish)
    // IMMUTABLE once published
  
  // Lifecycle
  status: String (enum: ['draft', 'published'], default: 'draft'),
    // 'draft' — being authored, not graded
    // 'published' — frozen, learner can attempt
    // Soft delete: admin deletes → status set to 'draft' (not hard removed)
  
  // Tenant & audit
  centerId: ObjectId (ref: 'Center', default: null),
    // Platform-shared in Phase 1 (null for all)
    // Future: scoped per center
  
  createdBy: ObjectId (ref: 'User', required),
    // Admin who created this quiz
    // Auto-stamped by admin-api createDefaults
  
  createdAt: Date (timestamps),
  updatedAt: Date (timestamps)
}
```

**Indexes:**
- `{code: 1}` (unique)
- `{status: 1}`

**Business rules:**
- `resolvedItemIds` must not be empty (enforced at publish validation)
- All items in `resolvedItemIds` must be `status:'active'` + `itemType ∈ COURSE_GRADABLE_TYPES` + have `answerKey`
- Labeling items must have renderable `answerKey` (per-prompt string array)
- Once `status='published'`, `resolvedItemIds` is immutable (no re-publish edit)
- Soft delete via status (admin can always create new draft after deleting one)

---

## Entity: QuizAttempt

**Collection:** `quiz_attempts`

**Purpose:** Records each learner's quiz submission. One row per attempt. Enables best-score tracking and audit trail.

**Schema:**

```javascript
{
  _id: ObjectId,
  
  // Foreign keys & routing
  userId: ObjectId (ref: 'User', index),
    // Learner who submitted
  
  courseId: ObjectId (ref: 'CourseStructure', index),
    // Course containing the lesson
  
  lessonPathKey: String (index),
    // Embedded lesson's path: e.g., 'p1:m1:l1'
    // (phaseKey:moduleKey:lessonKey, no ObjectId, no _id on embedded nodes)
    // Used to locate lesson node within course.phases[].modules[].lessons[]
  
  quizConfigId: ObjectId (ref: 'LessonQuiz', index),
    // The LessonQuiz that was attempted
  
  // Grading result (immutable once created)
  score: Number (0 to 1),
    // Ratio: correctCount / resolvedItemIds.length
    // E.g., 0.8 = 4 correct out of 5 questions
  
  passed: Boolean,
    // score >= quizConfigId.passThreshold
    // Used by lesson progress evaluation
  
  perItem: [
    {
      questionId: String (ObjectId as string),
        // AssessmentQuestion._id
      correct: Number (0 or 1)
        // 1 = graded correct, 0 = graded incorrect (or unanswered)
    }
  ],
    // One entry per resolved item (even if not answered)
    // Preserved for audit trail & per-question feedback
    // Length = LessonQuiz.resolvedItemIds.length
  
  // Audit
  createdAt: Date (timestamps),
  updatedAt: Date (timestamps)
    // updatedAt = when record created (no mutation after creation)
}
```

**Indexes:**
- `{userId: 1, courseId: 1, lessonPathKey: 1}` (compound, find all attempts for a lesson)
- `{userId: 1}` (find all attempts by user)
- `{quizConfigId: 1}` (find all attempts on a quiz config)
- `{createdAt: -1}` (sort by submission time)

**Business rules:**
- Immutable once created (no update/patch)
- perItem.length = LessonQuiz.resolvedItemIds.length (denominator invariant)
- perItem[i].correct derived from gradeObjective(questionId, response, answerKey) — client cannot set it
- One attempt per submission (no upsert/replace; each submit = new row)
- Best-score tracking: lesson progress reads this table for `passed=true`; adaptive.ingestQuizAttempt uses latest events (not best-score) for mastery

---

## Entity: FlashcardAttempt

**Collection:** `flashcard_attempts`

**Purpose:** Tracks learner's flashcard completion state (review progress). Self-contained exercise, not SRS.

**Schema:**

```javascript
{
  _id: ObjectId,
  
  // Foreign keys
  userId: ObjectId (ref: 'User', index),
  courseId: ObjectId (ref: 'CourseStructure', index),
  lessonPathKey: String (index),
    // Same as QuizAttempt
  
  deckId: ObjectId (ref: 'FlashcardDeck', index),
    // The system deck bound to the lesson
  
  // Review state
  reviewedCardIds: [ObjectId] (ref: 'FlashcardCard', default: []),
    // Cards learner has reviewed in this session
    // Completion: reviewedCardIds.length >= deck.cardIds.length
    // Upserted each time learner completes (latest pass overwrites)
  
  // Audit
  createdAt: Date (timestamps),
  updatedAt: Date (timestamps)
    // updatedAt = most recent review completion
}
```

**Indexes:**
- `{userId: 1, courseId: 1, lessonPathKey: 1}` (unique, upsert key)
- `{deckId: 1}` (find all attempts on a deck)

**Business rules:**
- Upsert (no create separate rows; latest pass replaces prior)
- No SRS scheduling (self-contained, one-pass completion)
- No learning_events created (flashcard doesn't feed mastery)
- Completion = reviewedCardIds.length >= deck cardIds.length
- recordLesson marks lesson as completed/mastered when flashcard pass (not mastery-tracked)

---

## Entity: LearningEvent (existing, reused)

**Collection:** `learning_events`

**Purpose:** Mastery-feeding event stream. Created by quiz submit (and other features like IPA attempts). Consumed by adaptive.computeMastery.

**Schema (existing):**

```javascript
{
  _id: ObjectId,
  
  userId: ObjectId (ref: 'User', index),
  source: String (enum: ['ipa', 'talk', 'quiz', 'checkpoint', 'lesson', ...]),
    // 'quiz' = created by quiz.submitLessonQuiz
  refId: ObjectId (ref to QuizAttempt._id when source='quiz'),
    // Links event back to the originating attempt
    // Used for latest-submission dedup (delete events by refId when new attempt)
  
  questionId: String (ObjectId as string, ref: 'AssessmentQuestion'),
    // Question that contributed the event
  
  // Item metadata (copied at event time, not re-fetched)
  skill: String (e.g., 'reading', 'listening'),
  subskill: String | null (e.g., 'inference', 'vocabulary'),
    // Only subskill-tagged items contribute to mastery
    // null subskill items ignored by computeMastery
  
  itemCefr: String | null (e.g., 'B1', 'C1'),
    // CEFR level of the question
  
  correct: Boolean,
    // Item graded correctly (1) or not (0)
  
  countsTowardMastery: Boolean (default: true),
    // Flag to exclude certain events from mastery (though quiz always true)
  
  createdAt: Date (timestamps)
}
```

**Indexes:**
- `{userId: 1, source: 1, refId: 1}` (dedup by refId when new submit)
- `{userId: 1, skill: 1, subskill: 1}` (aggregate for competency compute)

**Integration with Quiz:**
- Quiz submit creates ONE event per resolved item (even if unanswered or retired)
- Filter before insert: only items where `skill` is set (null skill items dropped)
- Latest-submission dedup: new submit → delete events with source='quiz' and refId={prior attempt ids} → insert new events
- adaptive.ingestQuizAttempt triggers adaptive.computeMastery after events inserted

---

## Entity: AssessmentQuestion (existing, reused, NOT modified)

**Collection:** `assessment_questions`

**Purpose:** Shared question bank. Quiz reuses; does NOT duplicate content.

**Relevant fields for Quiz:**

```javascript
{
  _id: ObjectId,
  
  // Content
  stem: String,
    // Question text (e.g., "What is the main idea?")
  
  options: [
    { index: Number, text: String }
    // Multiple choice options (if itemType = mcq/mcq_multi/matching_headings)
  ],
  
  itemType: String (e.g., 'mcq', 'cloze', 'fill_blank', 'true_false_ng', 'labeling', ...),
    // Type determines grading logic
    // COURSE_GRADABLE_TYPES = all except 'matching' (FE response shape mismatch)
  
  // Grading (server-side, never sent to learner)
  answerKey: { select: false },
    // Shape depends on itemType:
    // - mcq/mcq_multi: number (option index) or [numbers]
    // - true_false_ng: true|false
    // - cloze/fill_blank: string or string[] (variants)
    // - labeling: string[] (per-prompt labels)
    // - Etc.
    // MUST be non-null to be published in a quiz
    // Fetched with +answerKey only for server-side grading or validation
  
  acceptedVariants: { select: false },
    // Normalization rules for text-based types (cloze, fill_blank)
  
  // Metadata
  skill: String (e.g., 'reading', 'listening', 'use_of_english'),
    // Skill assessed by this question
  
  subskill: String | null (e.g., 'inference', 'vocabulary_from_context'),
    // Finer skill classification (optional)
    // Only subskill-tagged items contribute to mastery competency
  
  cefrLevel: String | null (e.g., 'B1', 'B2'),
    // CEFR difficulty level
  
  stimulusId: ObjectId | null (ref: 'AssessmentStimulus'),
    // If set: part of a testlet (audio passage/reading passage + questions)
    // If null: standalone item (grammar/vocab)
    // Client groups items by stimulusId to render testlets
  
  status: String (enum: ['active', 'inactive', 'retired', ...], default: 'active'),
    // Only 'active' questions are fetchable/gradable
    // 'retired' or 'inactive' cannot be used in new quizzes
    // Retirement of an active item IN a published quiz is BLOCKED (H1 guard)
  
  // Etc. (other fields not relevant to quiz)
}
```

**Contract with LessonQuiz:**
- Quiz.itemIds[] = array of AssessmentQuestion._ids
- At publish: every itemId must be status='active' + itemType ∈ COURSE_GRADABLE_TYPES + answerKey not null
- At grading: QuizAttempt uses freezed resolvedItemIds → fetch via fetchGradableItems (WITH answerKey, server-side only)
- After publish: admin cannot edit/retire/delete any question in quiz.resolvedItemIds

---

## Entity: AssessmentStimulus (existing, reused, NOT modified)

**Collection:** `assessment_stimuli`

**Purpose:** Shared testlet stimulus (audio/passage). Stimuli group questions into testlets for coherent learner experience.

**Relevant fields for Quiz:**

```javascript
{
  _id: ObjectId,
  
  kind: String (enum: ['passage', 'audio', ...]),
    // 'passage' = reading text
    // 'audio' = listening audio
  
  skill: String (e.g., 'reading', 'listening'),
    // Skill this stimulus targets
  
  // Content
  content: String | null,
    // Passage text (if kind='passage')
    // null if kind='audio' (audio served via audioUrl)
  
  audioUrl: String | null,
    // Signed URL for audio file (if kind='audio')
    // Expires in ~3 hours (HMAC signature with TTL)
  
  transcript: String | null,
    // Transcription of audio (if kind='audio')
    // NEVER sent to learner (server-side only, kept secret)
  
  status: String (enum: ['active', 'inactive', 'retired', ...]),
    // Only 'active' stimuli are fetched & served to learner
  
  // Etc.
}
```

**Contract with LessonQuiz:**
- Quiz.items[] reference stimulusId (if question part of testlet)
- buildTestletStimuli fetches ALL active stimuli for resolved items, groups by stimulusId
- Learner receives stimuli[] with kind/content/audioUrl/media (NOT transcript)
- Server groups stimulus + items by stimulusId for testlet rendering

---

## Relationships

```
LessonQuiz
  ├─ itemIds[] → AssessmentQuestion._id
  ├─ resolvedItemIds[] → AssessmentQuestion._id (frozen copy)
  │
  └─ < bound to > lesson.exercises[].refId = LessonQuiz.code
  
QuizAttempt
  ├─ userId → User._id
  ├─ courseId → CourseStructure._id
  ├─ quizConfigId → LessonQuiz._id
  ├─ perItem[].questionId → AssessmentQuestion._id (as string)
  │
  └─ < spawns > LearningEvent[]
       ├─ refId → QuizAttempt._id
       └─ questionId → AssessmentQuestion._id

FlashcardAttempt
  ├─ userId → User._id
  ├─ courseId → CourseStructure._id
  ├─ deckId → FlashcardDeck._id
  └─ reviewedCardIds[] → FlashcardCard._id

LearningEvent (from quiz)
  ├─ userId → User._id
  ├─ refId → QuizAttempt._id
  ├─ questionId → AssessmentQuestion._id
  └─ [aggregated by skill/subskill] → StudentModel mastery update

AssessmentQuestion
  ├─ stimulusId → AssessmentStimulus._id (optional)
  └─ < reused by multiple Quizzes >

AssessmentStimulus
  └─ < groups AssessmentQuestion[] by stimulusId >
```

---

## Query Patterns

**Learner Quiz Flow:**

```javascript
// 1. Load lesson + quiz config
LessonQuiz.findOne({code: exerciseRefId}, {status: 1})

// 2. Resolve client-safe items
AssessmentQuestion.find(
  {_id: {$in: quizConfig.resolvedItemIds}, status: 'active', itemType: {$in: [...COURSE_GRADABLE_TYPES]}},
  {stem: 1, options: 1, itemType: 1, skill: 1, cefrLevel: 1, stimulusId: 1, +answerKey: 1}
)
// Note: +answerKey only to build labeling answerSlots (never emitted)

// 3. Build testlets (group by stimulusId)
AssessmentStimulus.find(
  {_id: {$in: stimulusIds}, status: 'active'},
  {kind: 1, skill: 1, content: 1, audioUrl: 1, media: 1}
)
// Note: transcript NEVER selected

// 4. Grade submission
AssessmentQuestion.find(
  {_id: {$in: quiz.resolvedItemIds}, status: 'active'},
  {+answerKey: 1, +acceptedVariants: 1, itemType: 1, skill: 1, subskill: 1, cefrLevel: 1}
)

// 5. Persist attempt + events
QuizAttempt.create({userId, courseId, lessonPathKey, quizConfigId, score, passed, perItem})
LearningEvent.deleteMany({source: 'quiz', refId: {$in: priorAttemptIds}})
LearningEvent.insertMany([{userId, source: 'quiz', refId: attempt._id, questionId, skill, subskill, correct, countsTowardMastery: true}, ...])
```

**Admin Publish:**

```javascript
// 1. Load quiz
LessonQuiz.findById(id)

// 2. Validate all items
AssessmentQuestion.find(
  {_id: {$in: quiz.itemIds}, status: 'active'},
  {+answerKey: 1, itemType: 1, options: 1}
)
// Check each: itemType ∈ COURSE_GRADABLE_TYPES, answerKey ≠ null, labeling renderable

// 3. Freeze & publish
quiz.resolvedItemIds = quiz.itemIds
quiz.status = 'published'
quiz.save()
```

**Admin Guard (H1):**

```javascript
// Block edit/retire/delete of published-referenced question
LessonQuiz.exists({status: 'published', resolvedItemIds: questionId})
```

---

## Index Strategy

| Entity | Index | Reason |
|---|---|---|
| LessonQuiz | {code: 1} | Unique lookup by code (lesson.exercises[] refId) |
| LessonQuiz | {status: 1} | Filter published vs draft |
| QuizAttempt | {userId: 1, courseId: 1, lessonPathKey: 1} | Find all attempts on a lesson |
| QuizAttempt | {userId: 1} | User's all attempts |
| QuizAttempt | {quizConfigId: 1} | All attempts on a config |
| QuizAttempt | {createdAt: -1} | Sort by recent |
| FlashcardAttempt | {userId: 1, courseId: 1, lessonPathKey: 1} | Unique lookup for upsert |
| FlashcardAttempt | {deckId: 1} | All attempts on a deck |
| LearningEvent | {userId: 1, source: 1, refId: 1} | Dedup by refId on new submit |
| LearningEvent | {userId: 1, skill: 1, subskill: 1} | Aggregate for mastery compute |

---

## Migration / Seed Notes

- **LessonQuiz** created fresh in Phase 4 (admin authoring) + Phase 5 (seeding sample lessons)
- **QuizAttempt** created by learner submits (Phase 2 onwards when learner routes live)
- **FlashcardAttempt** created by learner completes (Phase 2 onwards)
- **LearningEvent** (quiz source) created by quiz submit (Phase 2)
- **AssessmentQuestion** + **AssessmentStimulus** pre-existing; no migration needed, only reused

---

## Soft Delete Strategy

- **LessonQuiz deletion** → status set to 'draft' (not hard deleted)
  - Preserves audit trail
  - QuizAttempt rows still reference old quiz via quizConfigId (can still fetch for learner history)
  - When querying "active quizzes", filter `status='published'`
- **Learning_events** are kept (audit trail of all events ever created)
  - Deletion only happens on new quiz attempt (latest-submission dedup)

