# Data model: Checkpoint + Band-up (bandPoint)

- **Spec:** `specs/course-checkpoints-band-up/spec.md`
- **Ngày:** 2026-08-03
- **Grounded in:** checkpoint-attempt.model.js, student-model.model.js, course-content.model.js, adaptive-path-snapshot, learning-event.model.js

## M1 — CheckpointRef on Course Structure

Không collection mới; checkpoint config lưu trực tiếp trên `Phase` + `CourseStructure`.

```javascript
// Thêm vào PhaseSchema (course-content.model.js)
checkpoint: {
  type: new Schema({
    quizCode: { type: String, required: true, trim: true },
    passThreshold: { type: Number, min: 0, max: 1, default: null },  // null → use quiz default
    label: { type: String, default: null }
  }, { _id: false }),
  default: null
}

// Thêm vào CourseStructureSchema (whole-khóa checkpoint)
checkpoint: {
  type: new Schema({
    quizCode: { type: String, required: true, trim: true },
    passThreshold: { type: Number, min: 0, max: 1, default: null },
    label: { type: String, default: null }
  }, { _id: false }),
  default: null
}
```

**Semantics:**
- `quizCode` → tham chiếu LessonQuiz code (UNIQUE, published).
- `passThreshold` — score threshold tối thiểu để pass. Nếu null → fallback (xem checkpoint.service.js:77):
  - `LessonQuiz.passThreshold` (config mặc định của quiz)
  - `0.8` (hardcoded default)
- `label` — tên hiển thị (optional; fallback "Bài kiểm tra chặng X").

## M2 — CheckpointAttempt (collection mới)

One document per learner submission (tương tự QuizAttempt nhưng keyed by boundary, không lesson path).

```javascript
// Collection: checkpoint_attempts
{
  _id: ObjectId,
  userId: ObjectId,                    // ref User
  courseId: ObjectId,                  // ref CourseStructure
  boundaryKey: String,                 // phaseKey hoặc literal 'course'
  quizCode: String,                    // để trace → LessonQuiz
  score: Number,                       // 0–1 (overall)
  passed: Boolean,                     // score ≥ passThreshold
  perItem: [
    {
      questionId: ObjectId,            // ref Question
      skill: String,                   // skill tag (tách từ question.skill)
      correct: Boolean
    }
  ],
  skillScores: [
    {
      skill: String,
      score: Number (0–1),             // sub-score (questions for this skill)
      raised: Boolean                  // whether this skill was raised on this attempt
    }
  ],
  createdAt: Date (auto)
}

// Index
db.checkpoint_attempts.createIndex({ userId: 1, courseId: 1, boundaryKey: 1, passed: 1 })
db.checkpoint_attempts.createIndex({ userId: 1, courseId: 1, boundaryKey: 1, createdAt: -1 })  // best-score-wins sort
```

**Semantics:**
- `boundaryKey` — **phaseKey** (vd "phase-1") cho checkpoint chặng, hoặc literal **"course"** cho checkpoint khóa.
- `score` — (# correct questions) / (total questions) — tính từ perItem.
- `skillScores` — grouped từ perItem by skill (xem checkpoint.verdict.js:computeVerdict).
- `raised` — true nếu sub-score ≥ threshold AND creditedBoundaries không chứa boundaryKey (idempotent).
- **No retry cooldown, no penalty** — các attempt cộng vào history; UI show best-score-wins.

## M3 — StudentModel band fields (extend)

`modules/adaptive/student-model.model.js`:

```javascript
SkillAbilitySchema {
  // Existing
  theta: Number,
  se: Number,
  cefr: String,  // CEFR_LEVELS = [A1, A2, B1, B2, C1, C2]

  // NEW
  // Continuous CEFR position (1.0–6.0) = source of truth; cefr = floor.
  bandPoint: {
    type: Number,
    min: 1.0,
    max: 6.0,
    default: null    // null for legacy records (helper fallback to cefrToBandPoint(cefr))
  },

  // Track which boundaries already credited to this skill (idempotent credit)
  creditedBoundaries: {
    type: [String],  // array of boundaryKeys (phaseKeys or 'course')
    default: []
  }
}
```

**Semantics:**
- `bandPoint` — **authoritative** band state (1.0–6.0 continuous). `cefr` = `floor(bandPoint)` để hiển thị:
  - A1: 1.0–1.99
  - A2: 2.0–2.99
  - B1: 3.0–3.99
  - B2: 4.0–4.99
  - C1: 5.0–5.99
  - C2: 6.0–6.xx
- `creditedBoundaries` — prevents double-credit on retake. Check: `if (!creditedBoundaries.includes(boundaryKey))` before `applyCheckpointResult`.

**Migration:** Seed `bandPoint = CEFR_NUM[cefr]` (integer anchor) once for all existing records.

## M4 — StudentPath snapshot (extend)

`modules/adaptive/adaptive-path-snapshot` (per-learner assignment, calculated once + frozen):

```javascript
// Snapshot.courses[courseIndex]
{
  slug: String,
  // ... existing fields
  
  // NEW — skill budget + target (frozen at assign-time)
  skillBudget: {
    type: Map<String, Number>,       // skill → per-chặng phân bổ (bandPoint)
    default: {}
  },
  skillTarget: {
    type: Map<String, Number>,       // skill → target bandPoint (clamped)
    default: {}
  },

  // Extend phases
  phases: [
    {
      key: String,
      // ... existing
      
      // NEW — per-chặng budget (same as parent skillBudget, but repeated for clarity at phase level)
      skillBudget: {
        type: Map<String, Number>,
        default: {}
      }
    }
  ]
}
```

**Semantics:**
- `skillBudget[skill]` — frozen per-chặng phân bổ: `(target[skill] − current[skill]) / countPhasesTeachingSkill[skill]`.
- `skillTarget[skill]` — clamped target bandPoint (= `min(course.cefrTo_bandNum, learner.target.cefr_bandNum)`).
- **Frozen at assign-time** (lúc buildStudentPath) → live course edits KHÔNG ảnh hưởng active learners.
- Retake: learner replan → build new snapshot → applyCheckpointResult reads từ NEW snapshot.

**Calculation (adaptive.service.buildStudentPath):**
```javascript
const computeSkillBudgets = (studentModel, courseStructure) => {
  const budgets = {};
  const phaseCountBySkill = {};
  
  // Count phases teaching each skill
  courseStructure.phases.forEach(phase => {
    phase.modules?.forEach(mod => {
      const skill = mod.skill;
      phaseCountBySkill[skill] = (phaseCountBySkill[skill] || 0) + 1;
    });
  });

  // Allocate per-phase
  Object.keys(studentModel.skill).forEach(skill => {
    const current = studentModel.skill[skill].bandPoint || cefrToBandPoint(studentModel.skill[skill].cefr);
    const target = min(cefrToBandPoint(courseStructure.cefrTo), studentModel.target.cefr_bandNum);
    const gap = max(0, target - current);
    const count = phaseCountBySkill[skill] || 1;
    budgets[skill] = gap / count;
  });

  return budgets;
};
```

## M5 — LearnerProfile.history (extend)

`modules/adaptive/learning-event.model.js` → `HistoryEntrySchema`:

```javascript
HistoryEntrySchema {
  // Existing fields
  takenAt: Date,
  assessmentId?: ObjectId,
  attemptId?: ObjectId,
  overallCefr: String,
  theta?: Number,
  se?: Number,

  // NEW
  // Source category: distinguishes checkpoint from assessment from ops
  sourceGroup: {
    type: String,
    enum: ['assessment', 'checkpoint', 'skill_retest', 'ops_recompute', 'ops_revoke', 'ops_recalibrate'],
    default: 'assessment'
  },

  // Which source caused the band change (can differ from sourceGroup in future)
  bandSource: {
    type: String,
    enum: ['assessment', 'checkpoint', 'skill_retest', 'recompute'],
    default: 'assessment'
  },

  // Reference to the source (e.g. CheckpointAttemptId for checkpoint)
  resultId: {
    type: Schema.Types.ObjectId,
    default: null
  },

  // Per-skill snapshot at this point in time
  skillCefr: {
    type: Map,
    of: String,                      // skill → CEFR (not bandPoint; CEFR for display)
    default: undefined
  },

  // Which skills changed band since previous entry
  changedSkills: {
    type: [String],
    default: []
  },

  // Audit trail
  actor: {
    kind: { type: String, enum: ['learner', 'system', 'admin'], default: 'learner' },
    id: { type: Schema.Types.ObjectId, default: null }
  },
  reason: { type: String, default: null },

  // Notification state (BR-12)
  notifiedLearner: { type: Boolean, default: false }
}
```

**Semantics:**
- `sourceGroup='checkpoint'` → entry from checkpoint pass (per boundary).
- `takenAt` — when the test was taken (submissionTime).
- `skillCefr` — snapshot of all skills' CEFR at this moment (not bandPoint; for display on timeline).
- `changedSkills` — which skills' CEFR differed from previous entry (to highlight on UI).
- **Append-only** — no updates after creation. Each checkpoint pass → one new entry (KHÔNG rewrite).

**Write logic (adaptive.service.applyCheckpointResult):**
```javascript
const priorSkillCefr = {};
const priorEntry = lp.history[0];  // newest
if (priorEntry && priorEntry.skillCefr) {
  priorSkillCefr = priorEntry.skillCefr;
}

const currSkillCefr = {};
Object.keys(model.skill).forEach(s => {
  currSkillCefr[s] = model.skill[s].cefr;
});

const changedSkills = Object.keys(currSkillCefr).filter(
  s => currSkillCefr[s] !== priorSkillCefr[s]
);

lp.history.unshift({
  takenAt: new Date(),
  sourceGroup: 'checkpoint',
  bandSource: 'checkpoint',
  resultId: attemptId,
  skillCefr: currSkillCefr,
  changedSkills,
  notifiedLearner: false
});

lp.save();
```

## M6 — Learning Event (extend)

`modules/adaptive/learning-event.model.js`:

```javascript
LearningEventSchema {
  // Existing
  userId: ObjectId,
  type: String,
  source: String,  // enum + ADD 'checkpoint'

  // Source values
  // OLD: 'ipa_attempt', 'talk_attempt', 'lesson_quiz', 'flashcard', 'assessment'
  // NEW: 'checkpoint'
  //      'checkpoint:chapterName' (optional, for detail)
}
```

Add `'checkpoint'` to source enum. Used by ingestQuizAttempt to detect checkpoint events → mastery compute.

---

## Validation Rules

| Field | Validation |
|---|---|
| `boundaryKey` | Must be exact match to `phase.key` OR literal `'course'`. Otherwise 404 CHECKPOINT_NOT_FOUND. |
| `passThreshold` | 0–1 (inclusive). Recommend 0.75–0.95. Default 0.8. |
| `skillBudget[skill]` | > 0; sum ≤ gap (shouldn't exceed target − current). |
| `skillTarget[skill]` | ≥ current bandPoint. Usually ≤ 6.0. |
| `bandPoint` | 1.0–6.0 (hardcoded min/max). Cannot go below 1.0 (A1 floor). |
| `creditedBoundaries` | Array of unique valid boundary keys. |
| `score` | 0–1. Calculated: correct_count / total_count. |
| `skillScores[*].score` | 0–1. Per-skill: correct_of_skill / total_of_skill. |
| `sourceGroup` | Enum value only. |

---

## Backward Compatibility

- `bandPoint` NULL for legacy StudentModel records → helpers fallback to `cefrToBandPoint(cefr)`.
- `CheckpointAttempt` new collection → no migration needed (fresh data).
- `HistoryEntry` new fields optional (default empty) → existing history entries still queryable.
- `skillBudget`/`skillTarget` new on StudentPath → learners without snapshot run with empty maps → `resolveBandBudget` returns empty → applyCheckpointResult skips credit (safe no-op).

---

## Storage & Indexing

| Collection | Index | Purpose |
|---|---|---|
| `checkpoint_attempts` | `{userId, courseId, boundaryKey, passed}` | Progress gate (has passed this boundary?) |
| `checkpoint_attempts` | `{userId, courseId, createdAt}` | Best-score-wins sort |
| `student_models` | `{userId}` (unique) | Fetch skillBandPoint + creditedBoundaries |
| `learner_profiles` | `{userId}` (unique) | Append history entries |
| `adaptive_path_snapshots` | `{userId}` (unique?) | Fetch budget/target per assign |

---

## Migration Strategy

**One-time seed (run once on deploy):**
```bash
db.student_models.updateMany(
  { 'skill.bandPoint': null },
  [{ $set: { 'skill': { $arrayToObject: [{ $map: { input: { $objectToArray: '$skill' }, ... } }] } } }]
);
```

Maps `skill[S].cefr` → `skill[S].bandPoint = CEFR_NUM[cefr]` for all existing students.

**StudentPath snapshot versioning:**
- First assign → buildStudentPath calculates skillBudget/skillTarget → save snapshot.
- Replan → recalculate → save new snapshot.
- Active learner sees NEW budget after replan (checkpoint progress reset? or cumulative? TBD by ops, not data model).

---

## Comparison to QuizAttempt

| Aspect | QuizAttempt | CheckpointAttempt |
|---|---|---|
| **Key identifier** | lessonPathKey | boundaryKey (phaseKey or 'course') |
| **Scope** | One lesson → one quiz per lesson | One phase/course boundary |
| **per* structure** | perItem: [{ lessonModuleKey, … }] | perItem: [{ skill, … }] |
| **Score aggregation** | Single score (pass/fail per lesson) | Overall + per-skill subscores |
| **Use case** | In-lesson inline progress | Gating milestone + skill raise |
| **Band-up** | No (lesson quiz DOESN'T raise band) | Yes (checkpoint pass → per-skill credit) |

Both reuse `gradeItem`, `fetchGradableItems`, `resolveQuizItems` (frozen item set machinery).

---

## Summary of New Collections/Fields

| What | Type | Location | Note |
|---|---|---|---|
| `CheckpointAttempt` | Collection | `checkpoint_attempts` | New; mirrors QuizAttempt, keyed by boundary |
| `bandPoint` | Field | `StudentModel.skill[S]` | New; 1.0–6.0, source of truth |
| `creditedBoundaries` | Field | `StudentModel.skill[S]` | New; array of boundary keys (idempotent) |
| `skillBudget` | Field | `StudentPath.courses[*]` | New; frozen phased allocation |
| `skillTarget` | Field | `StudentPath.courses[*]` | New; frozen target (clamped) |
| `checkpoint` | Field | `PhaseSchema`, `CourseStructureSchema` | New; nested ref + threshold |
| `sourceGroup` | Field | `LearnerProfile.history[*]` | New; 'checkpoint' et al. |
| `skillCefr` | Field | `LearnerProfile.history[*]` | New; per-skill snapshot |
| `changedSkills` | Field | `LearnerProfile.history[*]` | New; which skills changed CEFR |
