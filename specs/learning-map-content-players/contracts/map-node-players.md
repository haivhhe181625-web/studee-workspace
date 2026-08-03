# Contract: Learning Map Node Player Endpoints

- **Loại:** Cross-repo (exe-api ↔ exe-web)
- **Bên cung cấp (provider):** `exe-api` (`/api/adaptive/map/node`)
- **Bên tiêu thụ (consumer):** `exe-web` (node player component)
- **Trạng thái:** Đã triển khai (spec hồi tố)

---

## Endpoint 1: Start Node (resolve + bind)

```
POST /api/adaptive/map/node/:nodeKey/start
```

### Auth & Permission

- **Auth:** JWT user (learner)
- **Permission:** nên có `learning-map:read` (hoặc implicit từ enrollment)

---

## Request

**URL Param:**
```
:nodeKey = "core-uoe-a2-01"  // Node identifier trong roadmap
```

**Body:** (empty hoặc minimal)
```json
{}
```

---

## Response — thành công (MCQ_QUIZ)

**HTTP 200 OK**

```jsonc
{
  "nodeKey": "core-uoe-a2-01",
  "status": "IN_PROGRESS",
  "activity": {
    "activityType": "MCQ_QUIZ",
    "activityRef": {
      "refCollection": null,
      "refId": null,
      "slug": null
    },
    "config": {
      "itemCount": 5,
      "passRatio": 0.8,
      "mode": "single"
      // ⛔ KHÔNG có: itemIds, answerKey
    }
  },
  "items": [
    {
      "questionId": "652f8ab1234567890abcdef0",
      "itemType": "mcq",
      "stem": "She ___ to school every day.",
      "options": ["go", "goes", "going", "went"],
      "media": {
        "audioUrl": null,
        "imageUrl": null
      },
      "answerSlots": null,
      "skill": "use_of_english",
      "subskill": "grammar_tense",
      "cefrLevel": "A2"
      // ⛔ KHÔNG có: answerKey, acceptedVariants
    },
    // ... (4 more items, curated order từ config.itemIds)
  ],
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",
  "expiry": 1785000000,
  "sig": "hmac-sha256-hex-string-64-chars"
}
```

**Response — thành công (FLASHCARD_DECK)**

**HTTP 200 OK**

```jsonc
{
  "nodeKey": "vocab-a2-unit-1",
  "status": "IN_PROGRESS",
  "activity": {
    "activityType": "FLASHCARD_DECK",
    "activityRef": {
      "refCollection": "flashcard_decks",
      "refId": "63f1a2b3c4d5e6f7g8h9i0j1",
      "slug": null
    },
    "config": {}  // flashcard không config-heavy
  },
  "cards": [
    {
      "cardId": "c001-persevere",
      "term": "persevere",
      "meaning": "to continue with something despite difficulty",
      "ipa": "/pɜːr'sɪvɪr/",
      "example": "She persevered through the tough times.",
      "imageUrl": "https://cdn.studee/images/persevere.jpg",
      "audioUrl": null
    },
    // ... (19 more cards)
  ],
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",
  "expiry": 1785000000,
  "sig": "hmac-sha256-hex-string-64-chars"
}
```

**Response — thành công (VIDEO_LESSON)**

**HTTP 200 OK**

```jsonc
{
  "nodeKey": "lesson-grammar-01",
  "status": "IN_PROGRESS",
  "activity": {
    "activityType": "VIDEO_LESSON",
    "activityRef": {
      "refCollection": null,
      "refId": null,
      "slug": "course-video/lesson-01.mp4"
    },
    "config": {
      "minWatchRatio": 0.8
    }
  },
  "video": {
    "url": "https://cdn.studee/v1/signed/video/course/lesson-01.mp4?exp=1785000000&sig=hmac-sha256-hex...",
    "minWatchRatio": 0.8
    // url có thể là external: https://youtube.com/watch?v=abc123
  },
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",
  "expiry": 1785000000,
  "sig": "hmac-sha256-hex-string-64-chars"
}
```

---

## Response — lỗi

### 409 Conflict — Node Locked hoặc Missing Content

| HTTP | error code | Khi nào |
|---|---|---|
| 409 | `NODE_LOCKED` | Node status = LOCKED (chưa unlock điều kiện trước) |
| 409 | `NODE_NO_ITEMS` | MCQ: `config.itemIds` rỗng hoặc < 3 item hợp lệ |
| 409 | `FLASHCARD_DECK_NOT_FOUND` | Flashcard: deck trỏ `ownerType:'user'` (chỉ system allowed) |
| 404 | `FLASHCARD_DECK_NOT_FOUND` | Deck không tồn tại |
| 404 | `VIDEO_SOURCE_MISSING` | Video: internal ref + file không tồn tại |
| 400 | `VIDEO_NO_SOURCE` | Video: ref format invalid |

**Response body (lỗi):**
```jsonc
{
  "code": "NODE_NO_ITEMS",
  "message": "This node has no playable items",
  "nodeKey": "core-uoe-a2-01"
}
```

### 409 Conflict — Node Already Mastered

| HTTP | error code | Khi nào |
|---|---|---|
| 409 | `NODE_ALREADY_MASTERED` | Re-start node status = MASTERED (owner decide: allow hoặc block) |

---

## Endpoint 2: Submit Node (grade + unlock)

```
POST /api/adaptive/map/node/:nodeKey/submit
```

### Auth & Permission

- **Auth:** JWT user (learner)
- **Permission:** implicit (user xác định từ JWT)

---

## Request

**URL Param:**
```
:nodeKey = "core-uoe-a2-01"
```

**Body — MCQ_QUIZ:**
```jsonc
{
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",  // từ /start
  "expiry": 1785000000,                           // từ /start
  "sig": "hmac-sha256-hex-string-64-chars",       // từ /start
  "durationMs": 42000,                            // optional: thời gian làm bài
  "answers": [
    {
      "questionId": "652f8ab1234567890abcdef0",
      "choice": "1"  // ← STRING(optionIndex), NOT value "goes"
    },
    {
      "questionId": "652f8ab1234567890abcdef1",
      "choice": "2"
    },
    // ... (có thể < 5 câu; câu thiếu = miss)
  ]
  // ⛔ KHÔNG có field: "correct" (server bỏ qua hoàn toàn)
}
```

**Body — FLASHCARD_DECK:**
```jsonc
{
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",
  "expiry": 1785000000,
  "sig": "hmac-sha256-hex-string-64-chars",
  "ratings": [
    {
      "cardId": "c001-persevere",
      "rating": "good"  // enum: again | hard | good | easy
    },
    {
      "cardId": "c002-...",
      "rating": "easy"
    },
    // ... (có thể < 20 cards; câu thiếu = miss)
  ]
}
```

**Body — VIDEO_LESSON:**
```jsonc
{
  "attemptNonce": "b3f1a2c7d9e4f5g6h7i8j9k0l",
  "expiry": 1785000000,
  "sig": "hmac-sha256-hex-string-64-chars",
  "watchedRatio": 0.85,     // 0.0 to 1.0 (client self-report)
  "popupCorrect": null,     // hoãn: nếu có pop-up quiz ở video
  "durationMs": 600000      // optional
}
```

---

## Response — thành công

**HTTP 200 OK**

```jsonc
{
  "idempotent": false,  // true nếu replay cùng nonce (cached result)
  "result": {
    "score": 0.8,                      // 4/5 = 0.8
    "passed": true,                    // score >= passRatio
    "isMeasurement": false,
    "masterySignals": [
      {
        "subskillTag": "grammar_tense",
        "delta": 0.1                   // mastery improvement
      }
    ],
    "errorTags": [],                   // từ adaptive engine
    "skill": "use_of_english"
  },
  "node": {
    "nodeKey": "core-uoe-a2-01",
    "status": "COMPLETED"              // hoặc MASTERED nếu score cao
  },
  "unlocked": [
    "core-uoe-a2-02",                  // node kế unlock
    "core-uoe-a2-03"
  ],
  "band": null,
  "remediation": null,
  "tutorResolved": null,
  "pathVersion": 7                     // increment khi roadmap thay đổi
}
```

**Response — lỗi**

| HTTP | error code | Khi nào |
|---|---|---|
| 400 | `INVALID_ATTEMPT_NONCE` | `attemptNonce` không match hoặc hết hạn |
| 401 | `ATTEMPT_SIGNATURE_INVALID` | `sig` verify fail (HMAC mismatch) |
| 409 | `NODE_STATE_ERROR` | Node không IN_PROGRESS (ngoài lỳ submit từ /start) |
| 409 | `ATTEMPT_SIGNATURE_INVALID` | GETDEL binding fail (binding hết hạn 15min) — transient retry case |
| 422 | `INVALID_RATING_VALUE` | Flashcard: rating không ∈ {again, hard, good, easy} |

```jsonc
{
  "code": "INVALID_ATTEMPT_NONCE",
  "message": "Attempt expired or signature invalid",
  "nodeKey": "core-uoe-a2-01"
}
```

---

## Idempotency (Replay Protection)

**Scenario:**
1. Learner `/submit` → binding GETDEL, grade, applyResult → 200 OK
2. Network fail hoặc learner reload
3. Learner `/submit` lại **cùng `attemptNonce`** → **KHÔNG double-score**

**Mechanism:**
- Server check `ActivityResult.findOne({userId, attemptNonce})`
- Nếu exist → return `{ idempotent: true, result: cached, ...}`
- KHÔNG consume binding lần 2; KHÔNG call applyResult (giữ state)

---

## Versioning / Breaking Change

**Current:** v1 (initial release, Aug 2026).

**Future changes tracked:**
- `items[]` field mở rộng (new media types, answerSlots essay)
- `config` mở rộng (new settings)
- Error code bổ sung
- **Non-breaking:** thêm field vào response (FE ignore unknown field)
- **Breaking:** xoá field, đổi meaning enum, đổi HTTP code → version route `/api/v2/adaptive/map/node`

---

## Sequencing triển khai (Backend → Frontend)

1. **Backend (exe-api):** `/start` + `/submit` endpoint live, binding Redis OK, adapter grade hardened.
2. **Frontend (exe-web):** Mock API contract → implement 3 player component → tích hợp real endpoint.
3. **Admin (exe-admin):** Node editor UI (reuse existing), thêm Verify button VIDEO_LESSON (optional).
4. **Integration:** Seed curated content → A→Z demo flow → publish validation → learner go-live.

---

## Ví dụ gọi thực tế

### Example 1: MCQ_QUIZ — Start

```bash
curl -X POST https://api.studee.com/api/adaptive/map/node/core-uoe-a2-01/start \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{}'

# Response 200:
{
  "nodeKey": "core-uoe-a2-01",
  "status": "IN_PROGRESS",
  "activity": { "activityType": "MCQ_QUIZ", ... },
  "items": [
    { "questionId": "652f8ab...", "itemType": "mcq", "stem": "She ___ to school...", ... },
    // ... 4 more items
  ],
  "attemptNonce": "b3f1a2c7d9e...",
  "expiry": 1785000000,
  "sig": "a1b2c3d4e5f6..."
}
```

### Example 2: MCQ_QUIZ — Submit

```bash
curl -X POST https://api.studee.com/api/adaptive/map/node/core-uoe-a2-01/submit \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "attemptNonce": "b3f1a2c7d9e...",
    "expiry": 1785000000,
    "sig": "a1b2c3d4e5f6...",
    "durationMs": 45000,
    "answers": [
      { "questionId": "652f8ab...", "choice": "1" },
      { "questionId": "652f8ac...", "choice": "0" },
      { "questionId": "652f8ad...", "choice": "2" },
      { "questionId": "652f8ae...", "choice": "1" },
      { "questionId": "652f8af...", "choice": "3" }
    ]
  }'

# Response 200:
{
  "idempotent": false,
  "result": { "score": 0.8, "passed": true, "masterySignals": [{"subskillTag": "grammar_tense", "delta": 0.1}], ... },
  "node": { "nodeKey": "core-uoe-a2-01", "status": "COMPLETED" },
  "unlocked": ["core-uoe-a2-02"],
  "pathVersion": 7
}
```

### Example 3: FLASHCARD_DECK — Start

```bash
curl -X POST https://api.studee.com/api/adaptive/map/node/vocab-a2-unit-1/start \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{}'

# Response 200:
{
  "nodeKey": "vocab-a2-unit-1",
  "status": "IN_PROGRESS",
  "activity": { "activityType": "FLASHCARD_DECK", "activityRef": { "refId": "63f1a2..." }, ... },
  "cards": [
    { "cardId": "c001", "term": "persevere", "meaning": "...", ... },
    // ... 19 more cards
  ],
  "attemptNonce": "b3f1a2c7d9e...",
  "expiry": 1785000000,
  "sig": "a1b2c3d4e5f6..."
}
```

### Example 4: FLASHCARD_DECK — Submit

```bash
curl -X POST https://api.studee.com/api/adaptive/map/node/vocab-a2-unit-1/submit \
  -H "Authorization: Bearer eyJhbGc..." \
  -H "Content-Type: application/json" \
  -d '{
    "attemptNonce": "b3f1a2c7d9e...",
    "expiry": 1785000000,
    "sig": "a1b2c3d4e5f6...",
    "ratings": [
      { "cardId": "c001", "rating": "good" },
      { "cardId": "c002", "rating": "easy" },
      // ... 18 more cards (tất cả 20 hoặc thiếu = miss)
    ]
  }'

# Response 200:
{
  "idempotent": false,
  "result": { "score": 0.9, "passed": true, ... },
  "node": { "nodeKey": "vocab-a2-unit-1", "status": "COMPLETED" },
  "unlocked": ["vocab-a2-unit-2"],
  "pathVersion": 7
}
```

### Example 5: VIDEO_LESSON — Start + Submit

```bash
# Start
curl -X POST https://api.studee.com/api/adaptive/map/node/lesson-grammar-01/start \
  -H "Authorization: Bearer eyJhbGc..." \
  -d '{}'

# Response 200:
{
  "nodeKey": "lesson-grammar-01",
  "status": "IN_PROGRESS",
  "activity": { "activityType": "VIDEO_LESSON", "activityRef": { "slug": "course-video/lesson-01.mp4" }, ... },
  "video": {
    "url": "https://cdn.studee/v1/signed/...?exp=...&sig=...",
    "minWatchRatio": 0.8
  },
  "attemptNonce": "b3f1a2c7d9e...",
  "expiry": 1785000000,
  "sig": "a1b2c3d4e5f6..."
}

# Submit (learner watched 85%)
curl -X POST https://api.studee.com/api/adaptive/map/node/lesson-grammar-01/submit \
  -H "Authorization: Bearer eyJhbGc..." \
  -d '{
    "attemptNonce": "b3f1a2c7d9e...",
    "expiry": 1785000000,
    "sig": "a1b2c3d4e5f6...",
    "watchedRatio": 0.85,
    "durationMs": 595000
  }'

# Response 200:
{
  "idempotent": false,
  "result": { "score": 1.0, "passed": true, ... },
  "node": { "nodeKey": "lesson-grammar-01", "status": "COMPLETED" },
  "unlocked": ["lesson-grammar-02"],
  "pathVersion": 7
}
```
