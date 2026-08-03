# Contract: Admin Lesson Quiz CRUD & Publish

- **Loại:** Nội bộ module + cross-repo (api → admin)
- **Bên cung cấp:** exe-api (`src/admin-api/resources/lesson-quiz.admin.js`)
- **Bên tiêu thụ:** exe-admin UI (`src/pages/lesson-quizzes.tsx`)
- **Trạng thái:** Live (implemented Phase 4)

---

## Base Path

```
/api/admin/lesson-quizzes
```

All endpoints require:
- **Auth:** JWT user token (Bearer)
- **Permission:** `lesson-quiz:read` (for GET), `lesson-quiz:write` (for POST/PUT/DELETE/actions)

---

## 1. List Lesson Quizzes

```
GET /api/admin/lesson-quizzes
```

**Query parameters:**

| Param | Kiểu | Ghi chú |
|---|---|---|
| `page` | number | Page (1-based, default 1) |
| `limit` | number | Rows per page (default 20) |
| `sort` | string | Sort field + order (e.g., `createdAt:desc`) |
| `search` | string | Search code or title (partial match) |
| `status` | string | Filter by status ('draft', 'published') |

**Response:**

```json
{
  "data": [
    {
      "_id": "665f1a2b3c4d5e6f7g8h9i0j",
      "code": "quiz_listening_01",
      "title": "IELTS Listening Practice 1",
      "source": "curated",
      "status": "published",
      "passThreshold": 0.8,
      "centerId": null,
      "createdAt": "2026-07-15T10:30:00Z"
    }
  ],
  "pagination": {
    "total": 45,
    "page": 1,
    "limit": 20,
    "pages": 3
  }
}
```

**List fields returned:** code, title, source, status, passThreshold, centerId, createdAt

---

## 2. Get Single Lesson Quiz

```
GET /api/admin/lesson-quizzes/:id
```

**Response:**

```json
{
  "_id": "665f1a2b3c4d5e6f7g8h9i0j",
  "code": "quiz_listening_01",
  "title": "IELTS Listening Practice 1",
  "source": "curated",
  "itemIds": [
    "6500a1b2c3d4e5f6g7h8i9j0",
    "6500a1b2c3d4e5f6g7h8i9j1"
  ],
  "resolvedItemIds": [
    "6500a1b2c3d4e5f6g7h8i9j0",
    "6500a1b2c3d4e5f6g7h8i9j1"
  ],
  "passThreshold": 0.8,
  "status": "published",
  "centerId": null,
  "createdBy": "user_001",
  "createdAt": "2026-07-15T10:30:00Z",
  "updatedAt": "2026-07-15T11:00:00Z"
}
```

---

## 3. Create Lesson Quiz (Draft)

```
POST /api/admin/lesson-quizzes
```

**Request body:**

```json
{
  "code": "quiz_reading_02",
  "title": "Reading Comprehension — Unit 2",
  "source": "curated",
  "itemIds": ["qid1", "qid2", "qid3"],
  "passThreshold": 0.75
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| code | string | Có | Unique identifier (no spaces; kebab-case recommended) |
| title | string | Có | Display name |
| source | string | Không | Default 'curated' (only option in Phase 1) |
| itemIds | array | Không | Initial item selection (can be empty, added later) |
| passThreshold | number | Không | Default 0.8 (range 0–1) |

**Response:**

HTTP 201 Created

```json
{
  "_id": "665f1a2b3c4d5e6f7g8h9i0k",
  "code": "quiz_reading_02",
  "title": "Reading Comprehension — Unit 2",
  "source": "curated",
  "itemIds": ["qid1", "qid2", "qid3"],
  "resolvedItemIds": [],
  "passThreshold": 0.75,
  "status": "draft",
  "centerId": null,
  "createdBy": "user_001",
  "createdAt": "2026-08-03T12:00:00Z",
  "updatedAt": "2026-08-03T12:00:00Z"
}
```

**Notes:**
- `resolvedItemIds` starts empty (populated at publish)
- `status` defaults to 'draft'
- `createdBy` auto-stamped by admin-api (from request user)
- `centerId` auto-stamped (from user's center scope, or null if platform admin)

---

## 4. Update Lesson Quiz (Draft only)

```
PUT /api/admin/lesson-quizzes/:id
```

**Request body:**

```json
{
  "title": "Updated Title",
  "itemIds": ["qid1", "qid3", "qid5"],
  "passThreshold": 0.8
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| code | string | Read-only (cannot change) |
| title | string | Can update |
| itemIds | array | Can update (if status='draft') |
| passThreshold | number | Can update (if status='draft') |
| status | string | Cannot update directly (only via publish action) |

**Response:**

HTTP 200 OK (updated document)

```json
{
  "_id": "665f1a2b3c4d5e6f7g8h9i0k",
  "code": "quiz_reading_02",
  "title": "Updated Title",
  "itemIds": ["qid1", "qid3", "qid5"],
  "resolvedItemIds": [],
  "passThreshold": 0.8,
  "status": "draft",
  "updatedAt": "2026-08-03T12:30:00Z"
}
```

**Notes:**
- Cannot update if status='published' (freeze locks config)

---

## 5. Delete Lesson Quiz (Soft Delete → Draft)

```
DELETE /api/admin/lesson-quizzes/:id
```

**Response:**

HTTP 204 No Content (or 200 with updated doc)

**Side effect:**
- `status` set to 'draft' (soft delete convention)
- Document NOT hard-deleted (audit trail preserved)

---

## 6. Publish Lesson Quiz (Freeze & Validate)

```
POST /api/admin/lesson-quizzes/:id/publish
```

**Request body:** (empty)

```json
{}
```

**Response — success:**

HTTP 200 OK

```json
{
  "_id": "665f1a2b3c4d5e6f7g8h9i0k",
  "code": "quiz_reading_02",
  "title": "Reading Comprehension — Unit 2",
  "itemIds": ["qid1", "qid3", "qid5"],
  "resolvedItemIds": ["qid1", "qid3", "qid5"],
  "passThreshold": 0.8,
  "status": "published",
  "updatedAt": "2026-08-03T12:45:00Z"
}
```

**Notes:**
- `resolvedItemIds` copied from `itemIds` (frozen)
- `status` changed to 'published'
- All items in `itemIds` must pass validation (see error cases below)

**Response — validation error:**

HTTP 400 Bad Request

```json
{
  "error": {
    "code": "QUIZ_ANSWERS_INVALID",
    "message": "Quiz chỉ nhận câu tự chấm được (câu qid2 là matching)",
    "statusCode": 400,
    "details": {
      "failedItems": [
        {
          "itemId": "qid2",
          "reason": "unsupported_type",
          "expected": "one of [mcq, cloze, fill_blank, true_false_ng, matching_headings, sentence_completion, labeling]",
          "actual": "matching"
        }
      ]
    }
  }
}
```

**Validation rules (all-or-nothing):**
1. `itemIds` not empty
2. Every itemId must exist in AssessmentQuestion + status='active'
3. Every itemId.itemType ∈ COURSE_GRADABLE_TYPES (exclude matching)
4. Every itemId.answerKey not null
5. Labeling items: answerKey must be per-prompt string array (renderable)
6. IF any item fails: reject publish, return specific error (no partial publish)

**Error codes:**

| Mã HTTP | error code | Tình huống |
|---|---|---|
| 404 | QUIZ_CONFIG_NOT_FOUND | Quiz doc not found |
| 400 | QUIZ_ANSWERS_INVALID | Item not found \| not active \| wrong type \| missing key \| non-renderable labeling |
| 409 | CONFLICT | Unknown state error |

---

## Error Responses

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 401 | UNAUTHORIZED | No/invalid JWT |
| 403 | FORBIDDEN | User lacks `lesson-quiz:read` or `lesson-quiz:write` permission |
| 404 | NOT_FOUND | Quiz ID not found |
| 400 | BAD_REQUEST | Validation error (e.g., code already exists, invalid field type) |
| 409 | CONFLICT | State error (e.g., cannot update published quiz) |

---

## Versioning / Breaking Changes

N/A — new contract (Phase 1/4).

If future changes needed:
- Adding new question type to COURSE_GRADABLE_TYPES: existing quizzes unaffected (frozen)
- Changing passThreshold validation: only affects new publishes (published quizzes frozen)

---

## Examples

### Example 1: Create draft quiz

```bash
curl -X POST http://localhost:3000/api/admin/lesson-quizzes \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "quiz_unit1",
    "title": "Unit 1 Quiz",
    "itemIds": [],
    "passThreshold": 0.8
  }'
```

**Response:**
```json
{
  "_id": "abc123",
  "code": "quiz_unit1",
  "status": "draft",
  "itemIds": [],
  "resolvedItemIds": []
}
```

### Example 2: Update draft quiz with items

```bash
curl -X PUT http://localhost:3000/api/admin/lesson-quizzes/abc123 \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "itemIds": ["qid1", "qid2", "qid3", "qid4", "qid5"]
  }'
```

### Example 3: Publish quiz

```bash
curl -X POST http://localhost:3000/api/admin/lesson-quizzes/abc123/publish \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Success:**
```json
{
  "_id": "abc123",
  "status": "published",
  "resolvedItemIds": ["qid1", "qid2", "qid3", "qid4", "qid5"]
}
```

**Failure (non-gradable type):**
```json
{
  "error": {
    "code": "QUIZ_ANSWERS_INVALID",
    "message": "Quiz chỉ nhận câu tự chấm được (câu qid3 là essay)"
  }
}
```

### Example 4: Cannot edit published quiz

```bash
curl -X PUT http://localhost:3000/api/admin/lesson-quizzes/abc123 \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"itemIds": ["qid1", "qid2"]}'
```

**Response:**
```json
{
  "error": {
    "code": "CONFLICT",
    "message": "Không thể sửa quiz đã publish"
  }
}
```

