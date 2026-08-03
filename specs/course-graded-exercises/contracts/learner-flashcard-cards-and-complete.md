# Contract: Learner Flashcard Cards & Complete

- **Loại:** Cross-repo (api ↔ web)
- **Bên cung cấp:** exe-api (`services/api/src/modules/quiz/quiz.routes.js`)
- **Bên tiêu thụ:** exe-web (`src/services/quiz.service.ts`, flashcard runner component)
- **Trạng thái:** Live (implemented Phase 2)

---

## Endpoint 1: Fetch Flashcard Cards

```
GET /api/courses/:slug/flashcard/cards?phaseKey=<>&moduleKey=<>&lessonKey=<>
```

**Auth:** JWT user
**Permission:** Enrollment gate (same as quiz)
**Enrollment check:** User must have active CourseEnrollment + course published

---

## Request

Tham số truyền qua **query string** (GET — controller đọc `req.query`), KHÔNG có body.

| Param | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| phaseKey | string | Có | Phase key (query) |
| moduleKey | string | Có | Module key (query) |
| lessonKey | string | Có | Lesson key (query) |

---

## Response — thành công

HTTP 200 OK

```json
{
  "deck": {
    "deckId": "665f1a2b3c4d5e6f7g8h9i0j"
  },
  "cards": [
    {
      "cardId": "6500a1b2c3d4e5f6g7h8i9j0",
      "term": "apple",
      "meaning": "a fruit that grows on trees",
      "ipa": "/ˈæp.l̩/",
      "example": "I eat an apple every day.",
      "imageUrl": "https://cdn.example.com/apple.jpg",
      "audioUrl": "https://cdn.example.com/apple-audio.mp3"
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| deck | object | Deck metadata |
| deck.deckId | string (ObjectId) | FlashcardDeck._id |
| cards | array | Card list (all cards in deck, no SRS filtering) |
| cards[].cardId | string (ObjectId) | FlashcardCard._id |
| cards[].term | string | Word/term |
| cards[].meaning | string | Definition/meaning |
| cards[].ipa | string \| undefined | IPA pronunciation (if available) |
| cards[].example | string \| undefined | Example sentence (if available) |
| cards[].imageUrl | string \| undefined | Image URL (if available) |
| cards[].audioUrl | string \| undefined | Audio pronunciation URL (if available) |

**Notes:**
- `cards` array includes ALL cards in the deck (no SRS scheduling, adaptive filtering, or progress filtering)
- Minimum cards: >= 2 (enforced at lesson bind time)
- Card order: as stored in deck (no randomization at fetch)
- All URLs should be absolute (http/https)
- Optional fields (ipa, example, imageUrl, audioUrl) omitted if not available (undefined)

---

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 401 | UNAUTHORIZED | No/invalid JWT |
| 409 | NOT_ENROLLED | User not enrolled |
| 404 | FLASHCARD_DECK_NOT_FOUND | Lesson has no flashcard exercise, or deck not found, or deck not system-owned |
| 400 | FLASHCARD_QUIZ_TOO_SMALL | Deck has fewer than 2 cards |
| 404 | NOT_FOUND | Course or lesson not found |

---

## Endpoint 2: Complete Flashcard

```
POST /api/courses/:slug/flashcard/complete
```

**Auth:** JWT user
**Permission:** Enrollment gate
**Enrollment check:** Same as fetch

---

## Request

```json
{
  "phaseKey": "string",
  "moduleKey": "string",
  "lessonKey": "string",
  "reviewedCardIds": ["string"]
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| phaseKey | string | Có | Phase key |
| moduleKey | string | Có | Module key |
| lessonKey | string | Có | Lesson key |
| reviewedCardIds | array | Có | Array of cardId (ObjectId as string) learner reviewed |

**Notes:**
- `reviewedCardIds` should contain ALL cards in deck (completion = review entire deck)
- Learner can submit multiple times (latest pass overwrites prior)
- Invalid cardIds are silently filtered (only valid ObjectIds kept)

---

## Response — thành công

HTTP 200 OK

```json
{
  "completed": true,
  "progress": {
    "lessonKey": "l1",
    "status": "completed",
    "exercisesStatus": {
      "quiz": "completed",
      "flashcard": "completed"
    }
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| completed | boolean | true if reviewedCardIds.length >= deck.cardIds.length |
| progress | object | Lesson progress update (from recordLesson) |
| progress.lessonKey | string | Lesson identifier |
| progress.status | string | Lesson status: 'not_started' \| 'in_progress' \| 'completed' \| 'mastered' |
| progress.exercisesStatus | object | Per-exercise status (quiz, flashcard, ipa, talk) |

**Notes:**
- `completed = true` means learner reviewed all required cards
- `progress` reflects lesson-level status after marking flashcard complete
- Flashcard completion does NOT feed mastery (no learning_events) — only marks lesson as complete/mastered

---

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | QUIZ_ANSWERS_INVALID | reviewedCardIds is not an array |
| 409 | FLASHCARD_DECK_NOT_FOUND | Deck missing (race condition) |
| 400 | FLASHCARD_QUIZ_TOO_SMALL | Deck too small (race condition) |
| 401 | UNAUTHORIZED | Invalid token |
| 409 | NOT_ENROLLED | User not enrolled |

---

## Versioning / Breaking Changes

N/A — new contract (Phase 2).

---

## Sequencing

1. Deploy exe-api (routes + service)
2. Deploy exe-web (client service + types + runner component)

---

## Examples

### Example 1: Fetch cards

**Request:**
```json
{
  "phaseKey": "p1",
  "moduleKey": "m1",
  "lessonKey": "l1"
}
```

**Response (3 cards):**
```json
{
  "deck": {
    "deckId": "665f1a2b3c4d5e6f7g8h9i0j"
  },
  "cards": [
    {
      "cardId": "c1",
      "term": "apple",
      "meaning": "a red fruit",
      "imageUrl": "https://cdn.example.com/apple.jpg"
    },
    {
      "cardId": "c2",
      "term": "orange",
      "meaning": "a citrus fruit",
      "audioUrl": "https://cdn.example.com/orange.mp3"
    },
    {
      "cardId": "c3",
      "term": "banana",
      "meaning": "a yellow fruit",
      "ipa": "/bəˈnɑː.nə/"
    }
  ]
}
```

### Example 2: Complete flashcard (all cards reviewed)

**Request:**
```json
{
  "phaseKey": "p1",
  "moduleKey": "m1",
  "lessonKey": "l1",
  "reviewedCardIds": ["c1", "c2", "c3"]
}
```

**Response:**
```json
{
  "completed": true,
  "progress": {
    "lessonKey": "l1",
    "status": "completed",
    "exercisesStatus": {
      "flashcard": "completed"
    }
  }
}
```

### Example 3: Partial review (incomplete)

**Request:**
```json
{
  "reviewedCardIds": ["c1", "c2"]
}
```

**Response:**
```json
{
  "completed": false,
  "progress": {
    "lessonKey": "l1",
    "status": "in_progress",
    "exercisesStatus": {
      "flashcard": "in_progress"
    }
  }
}
```

### Example 4: Re-submit (latest pass overwrites)

**Learner reviews all 3 cards, submits same set again:**

**Request:**
```json
{
  "reviewedCardIds": ["c1", "c2", "c3"]
}
```

**Response:**
```json
{
  "completed": true,
  "progress": { "status": "completed", ... }
}
```

**Note:** FlashcardAttempt upserted (latest pass replaces prior); still only ONE session per user×lesson tracked.

