# Contract: Learner Quiz Items & Submit

- **Loại:** Cross-repo (api ↔ web)
- **Bên cung cấp (provider):** exe-api (`services/api/src/modules/quiz/quiz.routes.js`)
- **Bên tiêu thụ (consumer):** exe-web (`src/services/quiz.service.ts`, learner quiz runner component)
- **Trạng thái:** Live (implemented Phase 2)

---

## Endpoint 1: Fetch Quiz Items (no answers)

```
POST /api/courses/:slug/quiz/items
```

**Auth:** JWT user (Bearer token)
**Permission required:** No specific permission; enrollment gate suffices
**Enrollment check:** User must have active CourseEnrollment to `:slug` + course must be published

---

## Request

```json
{
  "phaseKey": "string",
  "moduleKey": "string",
  "lessonKey": "string"
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| phaseKey | string | Có | Phase key (embedded in course structure, no _id) |
| moduleKey | string | Có | Module key |
| lessonKey | string | Có | Lesson key |

---

## Response — thành công

HTTP 200 OK

```json
{
  "quiz": {
    "quizConfigId": "665f1a2b3c4d5e6f7g8h9i0j",
    "code": "quiz_listening_01",
    "passThreshold": 0.8,
    "items": [
      {
        "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
        "stem": "What is the main purpose of the audio?",
        "options": [
          {
            "index": 0,
            "text": "To describe a new technology"
          },
          {
            "index": 1,
            "text": "To criticize an argument"
          }
        ],
        "itemType": "mcq",
        "skill": "listening",
        "cefrLevel": "B1",
        "stimulusId": "stim_001" | null,
        "answerSlots": {
          "prompts": ["Label 1", "Label 2"],
          "choices": ["Choice A", "Choice B", "Choice C"]
        } | undefined
      }
    ],
    "stimuli": [
      {
        "id": "stim_001",
        "kind": "audio",
        "skill": "listening",
        "body": null,
        "media": {
          "audioUrl": "https://cdn.example.com/audio.mp3?exp=1693123456&sig=abc123..."
        }
      },
      {
        "id": "stim_002",
        "kind": "passage",
        "skill": "reading",
        "body": "Lorem ipsum dolor sit amet, consectetur adipiscing elit...",
        "media": null
      }
    ]
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| quiz | object | Container |
| quiz.quizConfigId | string (ObjectId) | LessonQuiz._id |
| quiz.code | string | LessonQuiz.code (identifier) |
| quiz.passThreshold | number (0–1) | Pass bar (e.g., 0.8 = 80%) |
| quiz.items | array | Client-safe question list (NO answerKey/acceptedVariants/transcript) |
| quiz.items[].questionId | string (ObjectId) | AssessmentQuestion._id |
| quiz.items[].stem | string | Question text |
| quiz.items[].options | array | Multiple choice options (if applicable) |
| quiz.items[].options[].index | number | Option position (0-based) |
| quiz.items[].options[].text | string | Option text |
| quiz.items[].itemType | string | Question type (mcq, cloze, labeling, etc.) |
| quiz.items[].skill | string | Skill assessed (reading, listening, use_of_english, etc.) |
| quiz.items[].cefrLevel | string | CEFR difficulty (A1, B1, B2, etc.) |
| quiz.items[].stimulusId | string (ObjectId) \| null | Testlet ID (groups questions with shared audio/passage); null = standalone |
| quiz.items[].answerSlots | object \| undefined | LABELING ONLY: prompt list + choice set (no mapping) |
| quiz.items[].answerSlots.prompts | array | Prompt text per slot |
| quiz.items[].answerSlots.choices | array | Shuffled choice list |
| quiz.stimuli | array | Testlet metadata (audio/passage + media URLs) |
| quiz.stimuli[].id | string (ObjectId) | AssessmentStimulus._id |
| quiz.stimuli[].kind | string | 'audio' \| 'passage' |
| quiz.stimuli[].skill | string | Skill (listening for audio, reading for passage) |
| quiz.stimuli[].body | string \| null | Passage text (if kind='passage'); null if kind='audio' |
| quiz.stimuli[].media | object \| null | Media URLs |
| quiz.stimuli[].media.audioUrl | string \| null | Signed URL (if kind='audio'); null if kind='passage' |

**Notes:**
- `items` array preserves order of `LessonQuiz.resolvedItemIds` (frozen, reproducible)
- `items` array includes ONLY active, objective-type questions
- `answerKey`, `acceptedVariants`, and `transcript` are **NEVER** included
- `answerSlots` (labeling only) includes prompts + distinct choice set but **NOT the per-prompt mapping** (mapping is server-side only)
- `stimuli` includes ONLY active stimuli; if a stimulus is retired but items reference it, items render without stimulus context
- Audio URLs are signed with 3-hour TTL (HMAC); client must use within window

---

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 401 | UNAUTHORIZED | No/invalid JWT token |
| 409 | NOT_ENROLLED | User not enrolled in course, or course not published |
| 404 | QUIZ_CONFIG_NOT_FOUND | Lesson does not exist, or lesson has no quiz exercise, or LessonQuiz not found |
| 409 | QUIZ_CONFIG_NOT_PUBLISHED | LessonQuiz status='draft' (not published yet) |
| 404 | NOT_FOUND | Course not found |

**Example error response:**

```json
{
  "error": {
    "code": "QUIZ_CONFIG_NOT_PUBLISHED",
    "message": "Quiz chưa được publish",
    "statusCode": 409
  }
}
```

---

## Endpoint 2: Submit Quiz & Grade

```
POST /api/courses/:slug/quiz/submit
```

**Auth:** JWT user
**Permission:** Enrollment gate
**Enrollment check:** Same as /items

---

## Request

```json
{
  "phaseKey": "string",
  "moduleKey": "string",
  "lessonKey": "string",
  "answers": [
    {
      "questionId": "string",
      "response": "number|string|boolean|null"
    }
  ]
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| phaseKey | string | Có | Phase key |
| moduleKey | string | Có | Module key |
| lessonKey | string | Có | Lesson key |
| answers | array | Có | Question responses |
| answers[].questionId | string (ObjectId) | Có | AssessmentQuestion._id |
| answers[].response | number \| string \| boolean \| null | Có | Learner answer (shape depends on itemType). null = no answer. |

**Response shape per itemType:**
- `mcq`: response = number (option index, 0-based, e.g., 1 for 2nd option)
- `mcq_multi`: response = array of numbers
- `true_false_ng`: response = true \| false
- `cloze`, `fill_blank`: response = string (text)
- `labeling`: response = array of strings (one label per prompt, in order)
- Other types: depends on grading logic

---

## Response — thành công

HTTP 200 OK

```json
{
  "score": 0.8,
  "passThreshold": 0.8,
  "passed": true,
  "perItem": [
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
      "correct": 1
    },
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j1",
      "correct": 0
    }
  ],
  "progress": {
    "lessonKey": "l1",
    "status": "completed",
    "exercisesStatus": {
      "quiz": "completed",
      "flashcard": "not_attempted"
    }
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| score | number (0–1) | Ratio: correctCount / resolvedItemIds.length |
| passThreshold | number | Pass bar (echoed for clarity) |
| passed | boolean | score >= passThreshold |
| perItem | array | Per-question grading result |
| perItem[].questionId | string (ObjectId) | AssessmentQuestion._id |
| perItem[].correct | number (0 or 1) | 1 = correct, 0 = incorrect (or unanswered) |
| progress | object | Lesson progress update (from recordLesson) |
| progress.lessonKey | string | Lesson identifier |
| progress.status | string | Lesson status: 'not_started' \| 'in_progress' \| 'completed' \| 'mastered' |
| progress.exercisesStatus | object | Per-exercise status (quiz, flashcard, ipa, talk, etc.) |

**Notes:**
- `perItem` array length = `LessonQuiz.resolvedItemIds.length` (frozen denominator, not answer count)
- `perItem[i].correct = 1` only if gradeObjective returns 1; unanswered questions = 0
- `perItem` preserves resolved set order
- `progress` reflects lesson-level status (combines all exercises: quiz, flashcard, ipa, talk, etc.)

---

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | QUIZ_ANSWERS_INVALID | answers is not an array, or answer payload malformed |
| 409 | QUIZ_CONFIG_NOT_FOUND | Quiz config missing (race condition) |
| 409 | QUIZ_CONFIG_NOT_PUBLISHED | Quiz not published (race condition) |
| 409 | CONFLICT | Quiz resolved set empty (publish defect, should never happen) |
| 401 | UNAUTHORIZED | Invalid token |
| 409 | NOT_ENROLLED | User not enrolled |

---

## Versioning / Breaking Changes

N/A — contract is new (Phase 1).

---

## Sequencing (if cross-repo)

1. Deploy exe-api (routes + service + models)
2. Deploy exe-web (client service + types + runner component)

---

## Examples

### Example 1: MCQ question

**Request:**
```json
{
  "phaseKey": "p1",
  "moduleKey": "m1",
  "lessonKey": "l1",
  "answers": [
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
      "response": 1
    }
  ]
}
```

**Response (if correct):**
```json
{
  "score": 1.0,
  "passed": true,
  "passThreshold": 0.8,
  "perItem": [
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
      "correct": 1
    }
  ],
  "progress": { ... }
}
```

### Example 2: Fill blank + unanswered item

**Request:**
```json
{
  "phaseKey": "p1",
  "moduleKey": "m1",
  "lessonKey": "l1",
  "answers": [
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
      "response": "hello"
    },
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j1",
      "response": null
    }
  ]
}
```

**Response:**
```json
{
  "score": 0.5,
  "passed": false,
  "passThreshold": 0.8,
  "perItem": [
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
      "correct": 1
    },
    {
      "questionId": "6500a1b2c3d4e5f6g7h8i9j1",
      "correct": 0
    }
  ],
  "progress": { "status": "in_progress", ... }
}
```

### Example 3: Labeling question

**Fetch items:**
```json
{
  "items": [
    {
      "questionId": "abc123",
      "stem": "Label each part of the diagram",
      "itemType": "labeling",
      "answerSlots": {
        "prompts": ["Part A (top-left)", "Part B (center)", "Part C (bottom-right)"],
        "choices": ["Nucleus", "Mitochondria", "Ribosome"]
      }
    }
  ]
}
```

**Submit:**
```json
{
  "answers": [
    {
      "questionId": "abc123",
      "response": ["Nucleus", "Mitochondria", "Ribosome"]
    }
  ]
}
```

**Response:**
```json
{
  "perItem": [
    {
      "questionId": "abc123",
      "correct": 1
    }
  ]
}
```

