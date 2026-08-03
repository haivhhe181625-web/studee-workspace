# Contract: Checkpoint Items & Submit (Course-boundary test)

- **Loại:** Cross-repo (api ↔ web)
- **Bên cung cấp (provider):** `exe-api` (`modules/checkpoint`, reusable `modules/assessment/testlet-stimuli`)
- **Bên tiêu thụ (consumer):** `exe-web` (`components/checkpoint-inline-runner`)
- **Trạng thái:** Đã triển khai

## Endpoint 1: Get Items

```
POST /api/courses/:slug/checkpoint/items
```

- **Auth:** JWT user (verifyToken) → `withTenant`
- **Permission required:** `courses:read` (implicit via enrollment assertEnrolled)

### Request

`slug` ở URL path; `boundaryKey` ở **body** (controller đọc `req.body`).

```json
{ "boundaryKey": "phase-1" }
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `slug` | String | Yes | Course slug (URL path) |
| `boundaryKey` | String | Yes | Body: phaseKey (vd 'phase-1') hoặc literal 'course' |

### Response — thành công (200)

Controller trả `res.json({ checkpoint })` — toàn bộ object dưới đây nằm dưới key `checkpoint` (envelope), vd `{ "checkpoint": { "quizCode": …, "items": […], "stimuli": […] } }`.

```json
{
  "quizCode": "chkpt-intro-ielts-a2-b1-phase-1",
  "passThreshold": 0.8,
  "items": [
    {
      "id": "q1",
      "text": "Which phrase is correct? ...",
      "type": "mcq",
      "options": [
        { "id": "opt-a", "text": "A) ..." },
        { "id": "opt-b", "text": "B) ..." },
        { "id": "opt-c", "text": "C) ..." },
        { "id": "opt-d", "text": "D) ..." }
      ],
      "stimulusId": null,
      "skill": "use_of_english",
      "subskill": null,
      "difficulty": 2
    },
    {
      "id": "q2",
      "text": "What does the speaker say about...?",
      "type": "mcq",
      "options": [...],
      "stimulusId": "stim-audio-1",
      "skill": "listening",
      "subskill": "main_idea"
    }
  ],
  "stimuli": [
    {
      "id": "stim-audio-1",
      "kind": "audio",
      "skill": "listening",
      "media": {
        "audioUrl": "https://media.app.local/...?sig=...&expires=..."
      },
      "body": null
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `quizCode` | String | Mã quiz được config cho checkpoint này |
| `passThreshold` | Number (0–1) | Ngưỡng pass (default 0.8) |
| `items` | Array | Danh sách câu hỏi (flat, không nhóm stimulus) |
| `items[*].id` | String | Unique question ID |
| `items[*].type` | String | 'mcq' (checkpoint chỉ phục vụ MCQ) |
| `items[*].options` | Array | Lựa chọn trắc nghiệm |
| `items[*].stimulusId` | String\|null | Tham chiếu stimulus (listening audio / passage); null = standalone |
| `items[*].skill` | String | Kỹ năng được test (listening, reading, use_of_english, …) |
| `stimuli` | Array | Danh sách stimulus (audio/passage) được group |
| `stimuli[*].id` | String | Unique stimulus ID |
| `stimuli[*].kind` | String | 'audio' hoặc 'passage' |
| `stimuli[*].media.audioUrl` | String | Signed URL (TTL ~3h), hoặc null nếu không signable |
| `stimuli[*].body` | String\|null | Đoạn văn bản (reading passage); null nếu audio |

**Semantics:**
- `items[]` stays flat — grading contract không đổi.
- `stimuli[]` additive — FE render `{ stimulus, questions[] }` groups nếu có stimulusId, fallback flat nếu missing.
- Audio URL signed (TTL 10800 sec = 3h) — learner không thể download/leak trực tiếp; expiry rate low.
- Transcript NEVER returned (select:false on model).

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 404 | CHECKPOINT_NOT_FOUND | boundaryKey không match phase hoặc course.checkpoint null |
| 404 | CHECKPOINT_NOT_FOUND | quizCode referenced không tìm thấy |
| 409 | QUIZ_CONFIG_NOT_PUBLISHED | LessonQuiz status ≠ 'published' |
| 403 | NOT_ENROLLED | learner not enrolled in course |
| 401 | UNAUTHORIZED | token missing |

---

## Endpoint 2: Submit Checkpoint

```
POST /api/courses/:slug/checkpoint/submit
```

- **Auth:** JWT user
- **Permission required:** `courses:read` (implicit enrollment)

### Request

```json
{
  "boundaryKey": "phase-1",
  "answers": [
    { "questionId": "q1", "selectedOptionId": "opt-b" },
    { "questionId": "q2", "selectedOptionId": "opt-c" }
  ]
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `boundaryKey` | String | Yes | phaseKey hoặc 'course' |
| `answers` | Array | Yes | Câu trả lời learner |
| `answers[*].questionId` | String | Yes | ID câu hỏi |
| `answers[*].selectedOptionId` | String | Yes | ID lựa chọn (MCQ) |

### Response — thành công (200)

```json
{
  "score": 0.85,
  "passed": true,
  "skillScores": [
    {
      "skill": "listening",
      "score": 0.9,
      "passed": true,
      "raised": true
    },
    {
      "skill": "reading",
      "score": 0.8,
      "passed": true,
      "raised": false,
      "reason": "đã đạt target"
    },
    {
      "skill": "use_of_english",
      "score": 0.7,
      "passed": false,
      "raised": false
    }
  ],
  "raisedSkills": ["listening"],
  "suggestions": [
    {
      "skill": "use_of_english",
      "weakPoints": [
        {
          "subskill": "conditional",
          "score": 0.3,
          "hint": "Review conditional structure lesson"
        }
      ]
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `score` | Number (0–1) | Overall score (tất cả câu) |
| `passed` | Boolean | score ≥ passThreshold |
| `skillScores` | Array | Per-skill verdict |
| `skillScores[*].skill` | String | Kỹ năng |
| `skillScores[*].score` | Number (0–1) | Sub-score (câu của skill này) |
| `skillScores[*].passed` | Boolean | sub-score ≥ threshold |
| `skillScores[*].raised` | Boolean | Whether this skill got band-up (only if passed=true + not already credited) |
| `raisedSkills` | Array of String | Skills mà bandPoint tăng |
| `suggestions` | Array | Study hints cho weak skills |

**Semantics:**
- **`raised=true`:** sub-score ≥ threshold AND creditedBoundaries KHÔNG chứa boundaryKey lần này → `applyCheckpointResult` credit band.
- **`raised=false + passed=true`:** sub-score pass nhưng đã được credit trước (idempotent), hoặc đã đạt target (clamped).
- **`passed=false`:** KHÔNG credit band, KHÔNG deduct.

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | BAD_REQUEST | answers malformed (missing questionId, etc.) |
| 404 | CHECKPOINT_NOT_FOUND | checkpoint không tìm / boundary invalid |
| 409 | QUIZ_CONFIG_NOT_PUBLISHED | quiz chưa publish |
| 409 | NEEDS_ASSESSMENT | no StudentModel (pre-assessment) → applyCheckpointResult fail |
| 403 | NOT_ENROLLED | learner not enrolled |
| 403 | NOT_FOUND / 403 | answer to invalid questionId (question not in this quiz) |
| 401 | UNAUTHORIZED | token missing |

---

## Versioning / breaking change

**N/A — contract mới**, không supersede endpoint cũ. Checkpoint là feature mới trong course-boundaries.

**Future evolution:**
- `v2` có thể add `explanations` field sau khi pass (để learner học từ sai).
- `stimuli[*].transcript` có thể serve nếu learner có premium subscription (requires permission check).

---

## Sequencing triển khai (cross-repo)

1. **exe-api** POST /checkpoint/submit, POST /checkpoint/items — test, merge.
2. **exe-web** call endpoints, render items + stimuli, submit answers — test integration.
3. **exe-admin** (optional Phase 2) — authoring UI chọn quiz per phase.

**Dependency:** exe-api must deploy trước exe-web (web calls api).

---

## Ví dụ gọi thực tế

### Lấy items

```bash
curl -X POST "http://localhost:3000/api/courses/ielts-a2-b1/checkpoint/items" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{ "boundaryKey": "phase-1" }'
```

Response:
```json
{
  "quizCode": "chkpt-intro-phase-1",
  "passThreshold": 0.8,
  "items": [...],
  "stimuli": [...]
}
```

### Submit checkpoint

```bash
curl -X POST "http://localhost:3000/api/courses/ielts-a2-b1/checkpoint/submit" \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "boundaryKey": "phase-1",
    "answers": [
      { "questionId": "q1", "selectedOptionId": "opt-b" },
      { "questionId": "q2", "selectedOptionId": "opt-c" }
    ]
  }'
```

Response:
```json
{
  "score": 0.85,
  "passed": true,
  "skillScores": [...],
  "raisedSkills": ["listening"]
}
```

---

## Notes for Implementers

- **Answer decryption:** resolveQuizItems handles encrypted questions (if applicable).
- **Grading:** gradeItem registry maps item.type → grader function (MCQ → gradeObjective).
- **Per-skill score:** aggregate từ perItem grouped by skill; skip non-matched items.
- **Best-score-wins:** if learner retake, new attempt recorded separately; frontend/backend choose max score.
- **Testlet fallback:** if stimulusId in item but missing from stimuli[], FE should render item flat (no audio/passage wrapper).
