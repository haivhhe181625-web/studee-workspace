# Data Model: Learning Map Content Players

- **Spec:** `specs/learning-map-content-players/spec.md`
- **Ngày:** 2026-08-03
- **Trạng thái:** Đã build (hồi tố từ code)

## Tổng quan

Mô hình dữ liệu xoay quanh 3 thực thể chính:
1. **Node (Roadmap)** — lưu `config` (MCQ) hoặc `activityRef` (flashcard/video).
2. **Content** — AssessmentQuestion (MCQ), FlashcardDeck + FlashcardCard, Media (video).
3. **Runtime** — ActivityResult (idempotency), Binding (Redis, single-use).

---

## 1. Node Configuration

### NodeConfig (MCQ_QUIZ)

```javascript
// Trong document roadmap node (nested)
{
  nodeKey: "core-uoe-a2-01",
  kind: "MCQ_QUIZ",  // hoặc 'FLASHCARD_DECK', 'VIDEO_LESSON'
  config: {
    // ** MCQ_QUIZ specific **
    itemIds: ["ObjectId_q1", "ObjectId_q2", ..., "ObjectId_q5"],  // cố định 5 câu
    itemCount: 5,  // info-only (actual = itemIds.length)
    passRatio: 0.8,  // 4 câu đúng để pass
    mode: "single",  // hoặc "timed", "shuffle_options", ...
    // Deprecated/KHÔNG dùng ở node player:
    // answerKey: null,  // KHÔNG sử dụng ở MCQ player (curated dùng item.answerKey từ bank)
  },
  gatePolicy: "SOFT",  // or HARD / NONE
  status: "LOCKED",  // or IN_PROGRESS / COMPLETED / MASTERED
}
```

**Validate publish (MCQ):**
- `config.itemIds` phải tồn tại, NOT empty
- Tất cả item ID trong `itemIds` phải:
  - Tồn tại trong `AssessmentQuestion` collection
  - `status: 'active'`
  - `itemType: 'mcq'` (MCQ_QUIZ không chấp đa hình)
- Ít nhất 3 item hợp lệ để pass MIN_ITEMS check

### NodeConfig (FLASHCARD_DECK)

```javascript
{
  nodeKey: "vocab-a2-unit-1",
  kind: "FLASHCARD_DECK",
  activityRef: {
    refCollection: "flashcard_decks",
    refId: "ObjectId_deckXyz",  // trỏ FlashcardDeck._id
    slug: null,  // không dùng cho flashcard
  },
  gatePolicy: "SOFT",  // CẤM "HARD" (publish validation)
  status: "LOCKED",
}
```

**Validate publish (Flashcard):**
- `activityRef.refId` tồn tại trong `FlashcardDeck`
- Deck `ownerType: 'system'` (chỉ hệ thống)
- Deck `status: 'active'` hoặc tương tự
- `cardCount > 0` (ít nhất 1 thẻ)
- `gatePolicy !== 'HARD'` → reject 422 `FLASHCARD_HARD_GATE`

### NodeConfig (VIDEO_LESSON)

```javascript
{
  nodeKey: "lesson-grammar-01",
  kind: "VIDEO_LESSON",
  activityRef: {
    refCollection: null,
    refId: null,
    slug: "course-video/lesson-01.mp4",  // internal: course-video/<file>
                                         // hoặc external: https://youtube.com/...
  },
  config: {
    minWatchRatio: 0.8,  // 80% watched để pass
    popupQuestionCount: 0,  // hoãn: pop-up quiz giữa video
  },
  gatePolicy: "SOFT",  // CẤM "HARD"
  status: "LOCKED",
}
```

**Validate publish (Video):**
- `activityRef.slug` format hợp lệ:
  - Internal: match regex `^course-video/` → check file tồn tại
  - External: match regex `^https?://` → URL syntax check
- Nếu invalid → 422 `VIDEO_NO_SOURCE`
- Nếu internal + file missing → 422 `VIDEO_SOURCE_MISSING`
- `gatePolicy !== 'HARD'` → reject 422 `VIDEO_HARD_GATE`

---

## 2. Content Models

### AssessmentQuestion (MCQ items)

```javascript
{
  _id: ObjectId("62e9b1..."),
  itemType: "mcq",  // enum: 'mcq' | 'audio_mcq' | 'mcq_multi' | ...
  
  // Stem (câu hỏi)
  stem: "She ___ to school every day.",
  options: [
    { text: "go", correctMarker: false },    // HOẶC string đơn "go"
    { text: "goes", correctMarker: true },   // (correct) marker để reference, KHÔNG phải actual key
    "going",
    "went"
  ],
  // **Hoặc** dạng mixed:
  options: ["go", "goes", "going", "went"],  // tập options dạng string
  
  // Answer key (server-side, select:false)
  answerKey: 1,  // INDEX (không value "goes") — locked decision D1
  
  // Metadata
  skill: "use_of_english",
  subskill: "grammar_tense",
  cefrLevel: "A2",
  
  // Media (nếu audio_mcq)
  media: {
    audioUrl: "https://cdn.studee/audio/q1.mp3",
    imageUrl: null,
  },
  
  // Admin + publisher
  status: "active",  // or 'archived' / 'draft'
  createdBy: ObjectId("user_admin"),
  createdAt: ISODate("2026-01-01"),
}
```

**Query chuẩn (node player):**
```javascript
AssessmentQuestion
  .find({ _id: { $in: [id1, id2, id3] } })
  .select('+answerKey +acceptedVariants itemType irt')  // +answerKey vì select:false
  .lean()
```

**publicItem (strip answerKey):**
```javascript
// Input từ DB
{ questionId:'q1', itemType:'mcq', stem:'...', options:[...], answerKey:1, media:{}, skill, subskill, cefrLevel }

// Output sang FE
{ questionId:'q1', itemType:'mcq', stem:'...', options:[...], media:{}, skill, subskill, cefrLevel }
// ⛔ Không chứa answerKey, acceptedVariants
```

### FlashcardDeck (deck container)

```javascript
{
  _id: ObjectId("63f1a2..."),
  name: "IELTS Vocabulary Unit 1",
  
  // Ownership
  ownerType: "system",  // enum: 'system' | 'user'
  ownerId: null,  // null if system; ObjectId if user
  
  // Content
  cardCount: 20,
  cards: [
    {
      cardId: "ObjectId_or_UUID",  // unique within deck
      term: "persevere",
      meaning: "to continue with something despite difficulty",
      ipa: "/pɜːr'sɪvɪr/",
      example: "She persevered through the tough times.",
      imageUrl: "https://cdn.studee/images/persevere.jpg",
      audioUrl: null,
      cefrLevel: "B2",
      topics: ["vocabulary", "character"],
      createdAt: ISODate("2025-12-01"),
    },
    // ... 19 more cards
  ],
  
  // Status
  status: "active",  // or 'archived'
  isArchived: false,
  
  // Audit
  createdBy: ObjectId("admin_user"),
  updatedBy: ObjectId("admin_user"),
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2026-07-27"),
}
```

**Validation rules:**
- `ownerType='system'` → must have publishing + review process
- `cardCount` vs `cards.length` must match
- Mỗi card `cardId` unique trong deck
- Rating enum for FSRS: `{again, hard, good, easy}` — validate ở adapter

### FlashcardCard (đơn vị thẻ, nested hoặc riêng)

Thường nested trong `FlashcardDeck.cards[]` (xem trên). Hoặc nếu riêng collection:

```javascript
{
  _id: ObjectId("63f1b3..."),
  deckId: ObjectId("63f1a2..."),  // back-ref
  term: "persevere",
  meaning: "to continue with something despite difficulty",
  // ... (tương tự nested)
  rating: {  // user progress (chỉ persist nếu user học từ deck riêng — NOT dùng trong map node)
    fsrs: { ease: 2.5, interval: 3, due: ISODate("2026-08-10") },
    lastRating: "good",
    lastReviewAt: ISODate("2026-08-03"),
  }
}
```

### Media (video + audio storage)

```javascript
// course-media collection (existing, reuse)
{
  _id: ObjectId("63f2c4..."),
  fileName: "lesson-01.mp4",
  category: "course-video",  // or 'course-audio', etc.
  size: 123456789,  // bytes
  mimeType: "video/mp4",
  
  // Storage reference (internal signRef)
  storageRef: {
    bucket: "studee-media",
    path: "video/course/lesson-01.mp4",
  },
  
  // Access control
  access: "signed",  // or 'public' / 'private'
  
  // Metadata
  uploadedBy: ObjectId("admin"),
  createdAt: ISODate("2026-07-01"),
}
```

**signMediaUrl logic (existing in course-content.public.service):**
```javascript
async function signMediaUrl(ref, ttlSec = 21600) {  // 6h
  // ref = 'course-video/lesson-01.mp4'
  // return signed URL: https://cdn.studee/v1/signed/video/...?exp=1785000000&sig=hmac...
}
```

---

## 3. Runtime / Transient Models

### ActivityResult (Idempotency ledger)

```javascript
// Collection: activity_results
{
  _id: ObjectId("63f3d5..."),
  userId: ObjectId("learner123"),
  attemptNonce: "b3f1a2c7d9e...",  // HMAC nonce từ /start
  nodeKey: "core-uoe-a2-01",
  nodeKind: "MCQ_QUIZ",
  
  result: {
    score: 0.8,
    passed: true,
    isMeasurement: false,
    masterySignals: [
      { subskillTag: "grammar_tense", delta: 0.1 }
    ],
    errorTags: [],
    skill: "use_of_english",
  },
  
  nodeAfterApply: {
    nodeKey: "core-uoe-a2-01",
    status: "COMPLETED",  // hoặc MASTERED
  },
  
  unlockedNodes: ["core-uoe-a2-02"],
  
  // Audit
  submittedAt: ISODate("2026-08-03T10:30:00Z"),
  createdAt: ISODate("2026-08-03T10:30:00Z"),
}
```

**Usage (idempotency check):**
```javascript
const existing = await ActivityResult.findOne({ userId, attemptNonce });
if (existing) {
  return {
    idempotent: true,
    result: existing.result,
    node: existing.nodeAfterApply,
    // ... cached result
  };
}
// else: continue with grade + applyResult
```

### Binding (Redis, ephemeral)

```javascript
// Redis key-value (KHÔNG persist vào MongoDB)
key: "bind:{userId}:{attemptNonce}:{prefix}"
value: itemIds | cardIds (JSON array hoặc set)
TTL: 900  // 15 minutes

// Examples:
key: "bind:learner123:b3f1a2c7d9e:nodeitems" → ["id1", "id2", "id3", "id4", "id5"]
key: "bind:learner123:b3f1a2c7d9e:nodecards" → ["c1", "c2", ..., "c20"]

// Operations:
SETEX(key, ttl, JSON.stringify(ids))  // bindItems
GETDEL(key)  // consumeBinding → atomic get + delete
```

**Prefix conventions:**
- `nodeitems` — MCQ_QUIZ item binding
- `nodecards` — FLASHCARD_DECK card binding
- (video KHÔNG cần binding — watchedRatio scalar)

---

## 4. API Request/Response DTOs

### StartNode Response (MCQ_QUIZ)

```javascript
{
  nodeKey: "core-uoe-a2-01",
  status: "IN_PROGRESS",
  activity: {
    activityType: "MCQ_QUIZ",
    activityRef: { /* ... */ },
    config: {
      itemCount: 5,
      passRatio: 0.8,
      mode: "single"
      // ⛔ NO: itemIds, answerKey
    }
  },
  items: [
    {
      questionId: "652f8ab...",
      itemType: "mcq",
      stem: "She ___ to school every day.",
      options: ["go", "goes", "going", "went"],
      media: { audioUrl: null, imageUrl: null },
      answerSlots: null,
      skill: "use_of_english",
      subskill: "grammar_tense",
      cefrLevel: "A2"
      // ⛔ NO: answerKey, acceptedVariants
    },
    // ... N items (curated order)
  ],
  attemptNonce: "b3f1...",
  expiry: 1785000000,
  sig: "hmac-hex...",
}
```

### SubmitNode Request (MCQ_QUIZ)

```javascript
{
  attemptNonce: "b3f1...",
  expiry: 1785000000,
  sig: "hmac-hex...",
  durationMs: 42000,  // optional
  answers: [
    { questionId: "652f8ab...", choice: "1" }  // choice = String(optionIndex)
    // ... 5 answers (có thể < 5, submission không complete)
  ]
  // ⛔ NO FIELD: correct
}
```

### CardDto (Flashcard response)

```javascript
{
  cardId: "ObjectId_or_UUID",
  term: "persevere",
  meaning: "to continue with something despite difficulty",
  ipa: "/pɜːr'sɪvɪr/",
  example: "She persevered through the tough times.",
  imageUrl: "https://cdn.studee/images/persevere.jpg",
  audioUrl: null,
  // ⛔ NO: answerKey, correctAnswer (flashcard không có key ẩn)
}
```

### FlashcardSubmit Request

```javascript
{
  attemptNonce: "b3f1...",
  expiry: 1785000000,
  sig: "hmac-hex...",
  ratings: [
    { cardId: "c1", rating: "good" },
    { cardId: "c2", rating: "again" },
    // ... 20 ratings (hoặc thiếu = miss)
  ]
  // rating ∈ {again, hard, good, easy}
}
```

### VideoSubmit Request

```javascript
{
  attemptNonce: "b3f1...",
  expiry: 1785000000,
  sig: "hmac-hex...",
  watchedRatio: 0.85,  // 0.0 to 1.0
  popupCorrect: null,  // hoãn: nếu có pop-up quiz
  durationMs: 600000,  // optional
}
```

---

## 5. Ràng buộc & Validation

| Field | Ràng buộc | Scope |
|---|---|---|
| `config.itemIds[]` | Không rỗng; ≥ 3 item hợp lệ; all active + itemType='mcq' | MCQ publish validate |
| `config.passRatio` | 0.0 to 1.0; thường 0.8 | MCQ grade |
| `answerKey` (MCQ items) | INDEX (int 0,1,2,3…) hoặc coerce-able string; KHÔNG value-type | MCQ grade via coerce |
| `options[]` (MCQ items) | String[] hoặc Object[]; phải match answerKey index range | MCQ grade |
| `activityRef.refId` (flashcard) | ObjectId hợp lệ; trỏ deck `ownerType:'system'` | Flashcard validate + resolve |
| `activityRef.slug` (video) | Format `course-video/*` (internal) hoặc `https?://…` (external) | Video validate + resolve |
| `minWatchRatio` (video) | 0.0 to 1.0; thường 0.7-0.9 | Video grade |
| `gatePolicy` | 'HARD' | 'SOFT' | 'NONE'; CẤM HARD cho FLASHCARD/VIDEO | Publish validate |
| `rating` (flashcard submit) | enum {again, hard, good, easy} | Flashcard grade validate |
| `watchedRatio` (video submit) | 0.0 to 1.0 | Video grade compare minWatchRatio |
| `attemptNonce` | HMAC token từ /start; phải resubmit ở /submit (không client chọn) | Idempotency + HMAC verify |

---

## 6. Indexing & Performance

| Collection/Field | Index | Lý do |
|---|---|---|
| `AssessmentQuestion._id` | default (primary) | Query by itemIds[] |
| `AssessmentQuestion.status` | simple | Filter 'active' |
| `AssessmentQuestion.itemType` | simple | Filter 'mcq' |
| `FlashcardDeck._id` | default | Lookup by refId |
| `FlashcardDeck.ownerType` | simple | Filter 'system' |
| `ActivityResult.{userId, attemptNonce}` | compound | Idempotency lookup (fast path) |
| Redis `bind:*` | N/A (TTL auto-expire) | Single-use, TTL 15min |

---

## 7. Schema validation (JSON schema / Joi example)

### Node MCQ_QUIZ publish validation

```javascript
const nodePublishSchema = Joi.object({
  kind: Joi.string().valid('MCQ_QUIZ').required(),
  config: Joi.object({
    itemIds: Joi.array()
      .items(Joi.string().regex(/^[0-9a-f]{24}$/))  // ObjectId format
      .min(3)  // MIN_ITEMS
      .required(),
    itemCount: Joi.number().positive(),
    passRatio: Joi.number().min(0).max(1),
    mode: Joi.string(),
  }).required(),
  gatePolicy: Joi.string().valid('SOFT', 'HARD', 'NONE'),
});

// Before publish: validate + check gatePolicy !== 'HARD' for FLASHCARD/VIDEO
```
