# Contract: Admin Group Items (Testlet Helper)

- **Loại:** Nội bộ module (question-bank helper action)
- **Bên cung cấp:** exe-api (`src/admin-api/resources/question-bank.admin.js`, action `/group-items`)
- **Bên tiêu thụ:** exe-admin UI (lesson-quiz curate flow)
- **Trạng thái:** Live (Phase 4, used during admin authoring)

---

## Purpose

Admin helper endpoint to **reconstruct testlet grouping** from a flat itemIds array. Given a list of question IDs, return grouping by `stimulusId` (which testlet/stimulus each item belongs to). Helps admin visualize how selectTestletAware will group items (whole-unit testlet selection).

---

## Endpoint

```
POST /api/admin/question-bank/group-items
```

**Auth:** JWT user
**Permission:** `question-bank:read` (readonly inspection, not mutation)

---

## Request

```json
{
  "itemIds": [
    "6500a1b2c3d4e5f6g7h8i9j0",
    "6500a1b2c3d4e5f6g7h8i9j1",
    "6500a1b2c3d4e5f6g7h8i9j2"
  ]
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| itemIds | array | Có | AssessmentQuestion._ids |

---

## Response — thành công

HTTP 200 OK

```json
{
  "groups": [
    {
      "stimulusId": "stim_listening_001",
      "kind": "audio",
      "skill": "listening",
      "itemCount": 2,
      "items": [
        {
          "questionId": "6500a1b2c3d4e5f6g7h8i9j0",
          "stem": "What is the main purpose?",
          "itemType": "mcq",
          "order": 0
        },
        {
          "questionId": "6500a1b2c3d4e5f6g7h8i9j1",
          "stem": "What does the speaker recommend?",
          "itemType": "mcq_multi",
          "order": 1
        }
      ]
    },
    {
      "stimulusId": null,
      "kind": "standalone",
      "skill": "use_of_english",
      "itemCount": 1,
      "items": [
        {
          "questionId": "6500a1b2c3d4e5f6g7h8i9j2",
          "stem": "Choose the correct form: ___ is better.",
          "itemType": "cloze",
          "order": null
        }
      ]
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| groups | array | Grouped items by stimulus |
| groups[].stimulusId | string (ObjectId) \| null | Stimulus ID (null = standalone) |
| groups[].kind | string | 'audio' \| 'passage' \| 'standalone' |
| groups[].skill | string | Skill category (reading, listening, use_of_english) |
| groups[].itemCount | number | Number of items in this group |
| groups[].items | array | Questions in group |
| groups[].items[].questionId | string (ObjectId) | AssessmentQuestion._id |
| groups[].items[].stem | string | Question text |
| groups[].items[].itemType | string | Question type (mcq, cloze, labeling, etc.) |
| groups[].items[].order | number \| null | Item order within testlet (null for standalone) |

---

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | BAD_REQUEST | itemIds is not an array, or missing |
| 404 | NOT_FOUND | One or more itemIds not found in question bank |
| 401 | UNAUTHORIZED | No/invalid JWT |
| 403 | FORBIDDEN | User lacks `question-bank:read` permission |

---

## Versioning

N/A — helper action, not a formal contract. Changes to grouping logic don't break API (response shape stable).

---

## Examples

### Example 1: Mix of testlet + standalone

**Request:**
```json
{
  "itemIds": ["q1", "q2", "q3"]
}
```

**Data (in question bank):**
- q1: listening, stimulusId=s_audio_001, order=0
- q2: listening, stimulusId=s_audio_001, order=1
- q3: use_of_english, stimulusId=null

**Response:**
```json
{
  "groups": [
    {
      "stimulusId": "s_audio_001",
      "kind": "audio",
      "skill": "listening",
      "itemCount": 2,
      "items": [
        {"questionId": "q1", "stem": "What is the main idea?", "itemType": "mcq", "order": 0},
        {"questionId": "q2", "stem": "What does the speaker recommend?", "itemType": "mcq", "order": 1}
      ]
    },
    {
      "stimulusId": null,
      "kind": "standalone",
      "skill": "use_of_english",
      "itemCount": 1,
      "items": [
        {"questionId": "q3", "stem": "Choose the word:", "itemType": "cloze", "order": null}
      ]
    }
  ]
}
```

**Admin visualization:**
- "Group 1: Listening testlet (audio) — 2 items"
- "Group 2: Grammar (standalone) — 1 item"
- Total budget: 3 items

### Example 2: Multiple testlets

**Request:**
```json
{
  "itemIds": ["a1", "a2", "b1", "b2", "b3", "c1"]
}
```

**Data:**
- a1, a2: stimulusId=s_passage_001 (reading testlet, 2 items)
- b1, b2, b3: stimulusId=s_audio_002 (listening testlet, 3 items)
- c1: stimulusId=null (standalone grammar)

**Response:**
```json
{
  "groups": [
    {
      "stimulusId": "s_passage_001",
      "kind": "passage",
      "skill": "reading",
      "itemCount": 2,
      "items": [
        {"questionId": "a1", ...},
        {"questionId": "a2", ...}
      ]
    },
    {
      "stimulusId": "s_audio_002",
      "kind": "audio",
      "skill": "listening",
      "itemCount": 3,
      "items": [
        {"questionId": "b1", ...},
        {"questionId": "b2", ...},
        {"questionId": "b3", ...}
      ]
    },
    {
      "stimulusId": null,
      "kind": "standalone",
      "skill": "use_of_english",
      "itemCount": 1,
      "items": [
        {"questionId": "c1", ...}
      ]
    }
  ]
}
```

**Admin insight:**
- If admin wants "up to 5 items" and uses selectTestletAware:
  - Reading testlet (2 items) ✓ taken whole (< 5)
  - Listening testlet (3 items) ✗ skipped (2 + 3 = 5, meets budget, but 2 + 3 + 1 would exceed)
  - Grammar (1 item) ✗ skipped
  - Result: 2 items (just reading testlet)
- Admin can adjust budget or manually pick specific testlets

---

## Usage in Admin UI Flow

1. **Admin curates items:** clicks "select items" → question-bank browser
2. **Admin picks items:** [q1, q2, q3, q4, q5, ...] (flat list)
3. **Admin clicks "Preview grouping"** → POST `/api/admin/question-bank/group-items` with selection
4. **UI shows testlet breakdown:**
   - Listening (audio) — q1, q2, q3
   - Grammar (standalone) — q4, q5
5. **Admin understands:** selectTestletAware will emit whole testlets (Listening 3 + Grammar 2 = 5 items)
6. **Admin can then:**
   - Accept (save to quiz config)
   - Adjust budget or selection
   - Remove items / add more

---

## Related Contract

- `admin-lesson-quiz-crud-and-publish.md` — after grouping, admin saves selection to LessonQuiz.itemIds via PUT /api/admin/lesson-quizzes/:id
- Learner-side: `learner-quiz-items-and-submit.md` — learner sees same testlet grouping via `stimuli[]` in response

