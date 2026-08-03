# Design: Learning Map Content Players

- **Spec:** `specs/learning-map-content-players/spec.md`
- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Trạng thái:** Đã build

## 1. Tóm tắt kiến trúc

3 content player (MCQ_QUIZ, FLASHCARD_DECK, VIDEO_LESSON) **tái sử dụng** cơ chế adapter pattern hiện tại (`activity/adapters/*.adapter.js`), mở rộng với:

- **Resolver layer** (`item-server.js`, `flashcard-public.service.js`, `course-content.public.service.js`): resolve + bind resource (items/cards/media) khi `/start`.
- **Anti-cheat binding (Redis):** GETDEL atomic, single-use per attempt (prevent recount + key reuse).
- **Server-keyed grade:** `/submit` fetch `serverAnswerKey` (MCQ) hoặc `bound cardIds` (flashcard) từ binding, chấm KHÔNG tin client.
- **Idempotency:** `ActivityResult` ledger track `(userId, attemptNonce)` → replay trả result cũ, KHÔNG double-grade.
- **Gating guardrail:** publish validation chặn `FLASHCARD_HARD_GATE` / `VIDEO_HARD_GATE` — flashcard/video KHÔNG làm HARD prerequisite.

Adapter độc lập (`mcq.adapter`, `flashcard.adapter`, `video.adapter`), dùng chung `NormalizedResult` schema từ adaptive module.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| **Curated vs Random (D2)** | CURATED `config.itemIds` cố định | Chặn reroll gian lận (F13 red-team), fair admin kiểm soát độ khó | Node chỉ 5 câu cố định, không variety cao (thiết kế tradeoff) |
| **answerKey type (D1)** | INDEX (0,1,2,3…) | Index-agnostic, dễ migrate reorder option, coerce String(answerKey)===String(choice) | Phải normalize seed toàn system → index |
| **Binding strategy (D3)** | Redis GETDEL atomic, `total=|bound|` | Single-use, prevent pick-twice; total neo cứng, không trust answers.length | TTL 15min → attempt phải submit trong window; GETDEL fail retry nhận misleading error |
| **Deck ownership (Anti-leak)** | Chỉ `ownerType:'system'` được resolve | Cấm learner deck private lộ global map; tránh data leak | Admin phải tạo deck hệ thống sẵn trước (KHÔNG on-demand) |
| **Video signed URL** | TTL 6h, KHÔNG bind userId | Consistent scheme nền tảng (course-content), simplify signing | URL share được 6h (tradeoff: platform policy, không siết TTL/user-bind ở slice này) |
| **Gating guardrail (A7)** | CẤM `FLASHCARD_HARD_GATE` / `VIDEO_HARD_GATE` | Tự đánh giá + pure self-report KHÔNG thể prove kiến thức cần HARD gate | Flashcard/video chỉ dùng CORE/PRACTICE/SOFT (giới hạn node chain) |

## 3. Data model

Chi tiết → `data-model.md`.

| Entity | Field đại diện | Ghi chú |
|---|---|---|
| **MCQNode** | `config: { itemIds: ObjectId[], itemCount, passRatio, mode }` | itemIds định sẽn curated; itemCount info-only (actual = itemIds.length) |
| **FlashcardNode** | `activityRef: { refCollection:'flashcard_decks', refId: ObjectId }` | refId trỏ FlashcardDeck (`ownerType:'system'` chỉ) |
| **VideoNode** | `activityRef: { slug: String }` | slug = `course-video/<file>` (internal) hoặc `http(s)://…` (external) |
| **NodeState** | `status: 'LOCKED'\|'IN_PROGRESS'\|'COMPLETED'\|'MASTERED'`, `gatePolicy: 'HARD'\|'SOFT'\|'NONE'` | gating validation: HARD + FLASHCARD/VIDEO → 422 |
| **AssessmentQuestion** | `itemType: 'mcq'\|'audio_mcq'\|...`, `answerKey: Index\|String\|Array`, `options: String[]\|Object[]` | Normalize answerKey → index; itemType filter MCQ chỉ |
| **FlashcardDeck** | `ownerType: 'user'\|'system'`, `cardCount, cards: [{cardId, term, meaning, ...}]` | Chỉ `ownerType:'system'` được node resolve |
| **ActivityResult** | `userId, attemptNonce, nodeKey, result: NormalizedResult, createdAt` | Ledger track (userId, attemptNonce) → idempotency |
| **Binding (Redis)** | `key: 'bind:{userId}:{attemptNonce}:{prefix}'`, value: `[itemIds\|cardIds]`, TTL: 15min | GETDEL atomic; prefix ∈ {nodeitems, nodecards, ...} |

## 4. Luồng dữ liệu

### MCQ_QUIZ flow

```
Client POST /start
  → verifyToken → findNodeInPath(nodeKey)
  → [new] loadCuratedItems(config.itemIds)
    → AssessmentQuestion.find({_id:{$in:itemIds}})
      .select('+answerKey').lean()
    → sort theo order itemIds
    → filter itemType='mcq'
    → nếu < MIN_ITEMS (3) → 409 NODE_NO_ITEMS
  → patchNodeStatesOcc(IN_PROGRESS) [chỉ sau load thành công]
  → signAttempt(nonce, expiry, sig)
  → bindItems(userId, nonce, itemIds, 900, 'nodeitems') [Redis SETEX]
  → map publicItem (strip answerKey, acceptedVariants)
  → whitelist config (strip itemIds, answerKey)
  → return { nodeKey, status, activity:{activityDescriptor}, items[], attemptNonce, expiry, sig }

Client POST /submit + {answers:[{questionId, choice}], attemptNonce, sig, expiry}
  → verifyHMAC
  → CHECK ActivityResult(userId, attemptNonce) → exists?
    → yes: return idempotent:true + cached result
    → no: continue
  → consumeBinding(userId, nonce, 'nodeitems') [Redis GETDEL]
    → itemIds (bound set)
  → AssessmentQuestion.find({_id:{$in:itemIds}})
      .select('+answerKey +acceptedVariants itemType irt')
    → build serverAnswerKey = {questionId:answerKey}
  → grade(answers, serverAnswerKey) via adapter.run()
    → lặp Object.keys(serverAnswerKey)
    → mỗi key: gradeItem(serverAnswerKey[key], answers.find(qId).choice)
    → coerce String(key)===String(choice)
    → total = |serverAnswerKey| (không answers.length)
    → score = correct/total
  → applyResult(score >= passRatio → passed)
  → return SubmitResult { idempotent:false, result:{...}, node:{status}, unlocked:[...], ... }
```

### FLASHCARD_DECK flow

```
Client POST /start
  → verifyToken → findNodeInPath
  → [new] resolveFlashcardDeck(activityRef.refId)
    → FlashcardDeck.findById(refId)
    → check ownerType==='system' → nếu 'user' → 409 FLASHCARD_DECK_NOT_FOUND
    → cards = deck.cards (20 cards, example)
  → patchNodeStatesOcc(IN_PROGRESS)
  → signAttempt
  → bindCards(userId, nonce, cardIds, 900, 'nodecards')
  → map cardDTO (term, meaning, ipa, example, imageUrl, audioUrl)
  → return { cards[], attemptNonce, expiry, sig }

Client POST /submit + {ratings:[{cardId, rating}], attemptNonce, ...hmac}
  → verifyHMAC
  → CHECK ActivityResult → cached?
    → yes: idempotent:true
    → no: continue
  → consumeBinding(userId, nonce, 'nodecards') → cardIds
  → grade(ratings, cardIds) via adapter.run()
    → validate rating ∈ {again, hard, good, easy}
    → count rating in {good, easy} → correct (simplistic FSRS)
    → total = |cardIds| (bound)
    → score = correct/total
  → applyResult → passed?
  → return SubmitResult
```

### VIDEO_LESSON flow

```
Client POST /start
  → verifyToken → findNodeInPath
  → [new] resolveVideoSource(activityRef.slug)
    → if slug='course-video/*':
      → check file tồn tại (local.exists) → NO → 404 VIDEO_SOURCE_MISSING
      → signMediaUrl(ref, 6h) → signed URL
    → else if slug='http(s)://':
      → URL valid? → NO → 400 VIDEO_NO_SOURCE
      → pass-through URL
  → patchNodeStatesOcc(IN_PROGRESS)
  → signAttempt
  → [NO binding — watchedRatio là single scalar, không anti-swap per-item]
  → return { video:{url, minWatchRatio}, attemptNonce, expiry, sig }

Client POST /submit + {watchedRatio, attemptNonce, ...hmac}
  → verifyHMAC
  → CHECK ActivityResult → cached?
  → grade(watchedRatio >= minWatchRatio) → passed
  → applyResult
  → return SubmitResult
```

## 5. Contracts

Danh sách file trong `contracts/`:
- `map-node-players.md` — API `/start` + `/submit` bound (`MCQ_QUIZ`, `FLASHCARD_DECK`, `VIDEO_LESSON`)
- (Optional) `node-editor-verify.md` — Verify endpoint cho admin UI (VIDEO_LESSON slug)

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `exe-api/services/api/src/modules/learning-map/activity/item-server.js` | Helper: `loadCuratedItems, publicItem, gradeItem, bindItems, consumeBinding` | Tạo (Phase 1) |
| `exe-api/services/api/src/modules/learning-map/learning-map.service.js` | startNode/submitNode dispatch để MCQ_QUIZ curated; idempotency check | Sửa (Phase 2) |
| `exe-api/services/api/src/modules/learning-map/activity/adapters/mcq.adapter.js` | Grade MCQ từ serverAnswerKey (không client.correct); `total=|keys|` | Sửa (hardening) |
| `exe-api/services/api/src/modules/learning-map/activity/adapters/flashcard.adapter.js` | Grade flashcard từ bound cardIds; FSRS simplistic | Sửa |
| `exe-api/services/api/src/modules/learning-map/activity/adapters/video.adapter.js` | Grade watchedRatio >= minWatchRatio | Tạo hoặc Sửa |
| `exe-api/services/api/src/modules/flashcard/flashcard.service.js` | resolveFlashcardDeck(refId, ownerType filter) | Sửa |
| `exe-api/services/api/src/modules/course-content/course-content.public.service.js` | resolveVideoSource(slug), signMediaUrl | Sửa (reuse existing + thêm validate) |
| `exe-api/services/api/src/modules/learning-map/learning-map.publish.js` | Publish validation: node kind + gatePolicy check (HARD_GATE chặn) | Sửa |
| `exe-api/services/api/src/constants/error-codes.js` | Thêm: `NODE_NO_ITEMS, FLASHCARD_HARD_GATE, VIDEO_HARD_GATE, VIDEO_NO_SOURCE, VIDEO_SOURCE_MISSING` | Sửa |
| `exe-api/services/api/src/scripts/seed-map-template.js` | Seed node MCQ_QUIZ với `config.itemIds` curated; answer-neutral option | Sửa |
| `exe-web/components/node-player/MCQPlayer.tsx` | Render items[], submit form với choice index | Tạo hoặc Sửa |
| `exe-web/components/node-player/FlashcardPlayer.tsx` | Render cards[], self-rate, submit ratings | Tạo hoặc Sửa |
| `exe-web/components/node-player/VideoPlayer.tsx` | Video tag + watchedRatio report | Tạo hoặc Sửa |
| `exe-admin/pages/map-editor/NodeEditor.tsx` | Node field editor; thêm Verify button cho VIDEO_LESSON | Sửa |

## 7. Xử lý lỗi

| Tình huống | HTTP | error code | Response |
|---|---|---|---|
| Node MCQ nhưng `config.itemIds` rỗng hoặc < 3 item active | 409 | `NODE_NO_ITEMS` | `{ code, message: "This node has no playable items" }` |
| Node flashcard trỏ deck `ownerType:'user'` | 409 | `FLASHCARD_DECK_NOT_FOUND` | `{ code, message: "Flashcard deck ... không phải hệ thống" }` |
| Deck không tồn tại | 404 | `FLASHCARD_DECK_NOT_FOUND` | `{ code, message: "Deck not found" }` |
| Video ref format `course-video/*` nhưng file không tồn tại | 404 | `VIDEO_SOURCE_MISSING` | `{ code, message: "Video file not found" }` |
| Video ref invalid (không `course-video/*`, không URL http(s)) | 400 | `VIDEO_NO_SOURCE` | `{ code, message: "Video reference invalid" }` |
| Publish node MCQ_QUIZ không curated (missing itemIds) | 422 | `NODE_NO_ITEMS` | Error detail: required config.itemIds |
| Publish node FLASHCARD_DECK + `gatePolicy:'HARD'` | 422 | `FLASHCARD_HARD_GATE` | `{ code, message: "Flashcard cannot be HARD prerequisite" }` |
| Publish node VIDEO_LESSON + `gatePolicy:'HARD'` | 422 | `VIDEO_HARD_GATE` | `{ code, message: "Video cannot be HARD prerequisite" }` |
| Re-start node MASTERED | 409 | `NODE_ALREADY_MASTERED` | `{ code, message: "Node already mastered" }` (hoặc allow with note) |

## 8. Bảo mật & quyền

| Yêu cầu | Cách thực hiện | Kiểm tra |
|---|---|---|
| answerKey KHÔNG lộ client (D4) | publicItem strip; whitelist config | assert `/start` response JSON NOT contain `answerKey`, `config.itemIds` |
| Chỉ deck `ownerType:'system'` được resolve | resolveFlashcardDeck() kiểm tra server-side | test: deck 'user' → 409 |
| Video media signed (nếu internal) | signMediaUrl(ref, 6h TTL) | verify URL format: `?exp=...&sig=...` |
| Server-keyed grade (không trust client.correct) | adapter.run() dùng serverAnswerKey, bỏ client.correct | test: submit `{correct:true}` → ignore |
| Binding single-use (GETDEL) | Redis GETDEL atomic | test: submit 2 lần cùng nonce → lần 2 idempotent |
| Publish guard gating (D7) | publish validation check HARD + FLASHCARD/VIDEO | test: publish node FLASHCARD + HARD → 422 |

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

**Giới hạn hiện tại (không blocking, out-of-scope):**
- **L1 (transient retry):** nếu `applyResult` throw CONCURRENT_WRITE **sau** binding consume → retry nhận `ATTEMPT_SIGNATURE_INVALID` (misleading). Workaround: node vẫn IN_PROGRESS → start lại cấp binding mới (acceptable).
- **L2 (video share):** signed URL 6h không bind userId → share được (platform policy tradeoff, scope không siết).
- **L3 (flashcard stateless):** adapter chỉ grade pass/fail, không persist FSRS user state → score throwaway (designed, KHÔNG persist SRS).

**Architectural assumptions:**
- `ActivityResult` ledger đã implement → idempotency check before grade.
- `applyResult` logic (mastery, unlock, state transition) — tái sử dụng hiện có, KHÔNG đổi.
- Redis ≥ 6.2 (`GETDEL` native) hoặc node-redis v4 mock (dev) → production: confirm version.

## 10. Testing strategy

| Loại test | Case chính | Scope |
|---|---|---|
| **Unit (item-server.js)** | `loadCuratedItems` order + filter mcq; `publicItem` strip key; `gradeItem` coerce index; `bindItems`/`consumeBinding` Redis | Binding + grade logic riêng |
| **Integration (MCQ flow)** | `/start` curated items → `/submit` server-grade → unlock | Full A→Z learner journey |
| **Integration (Flashcard flow)** | `/start` deck → `/submit` ratings → check bound | Flashcard resolution + grade |
| **Integration (Video flow)** | `/start` signed/external URL → `/submit` watchedRatio | Video resolution + simple grade |
| **Anti-cheat (red-team)** | F2 (coerce), F3 (omission), F4 (client.correct), F5 (filter), F7 (ordering), F8 (whitelist), F10 (idempotent) | 8 exploit → red before green |
| **Publish validation (gate)** | HARD_GATE chặn flashcard/video | Block risky node graph |
| **Regression (existing)** | test-out (SubmitSimulator) xanh y nguyên (cũ node không curated → giữ nhánh cũ) | 0 breaking change |

## 11. Rollout / cross-repo sequencing

Parallel 3 track (2026-07-26 onwards):
1. **Backend (exe-api):** Phase 1 extract helper → Phase 2 curated MCQ + grade hardening → ready contract DS-6.
2. **Frontend-web (exe-web):** mock API contract → implement 3 player component → build xanh.
3. **Frontend-admin (exe-admin):** reuse node editor + thêm Verify button VIDEO_LESSON.
4. **Integration:** seed curated items → A→Z demo → publish validation → live.

**Deploy order:**
- api deploy trước (contract DS-6 live);
- web/admin sau (mock → real API);
- admin UI live (node editor + Verify);
- learner surface go-live.

## 12. Đối chiếu Acceptance criteria

| AC | Được đáp ứng bởi |
|---|---|
| AC-1 (curated MCQ) | `loadCuratedItems` (item-server.js §3b), binding Redis, publicItem whitelist |
| AC-2 (NODE_NO_ITEMS) | loadCuratedItems guard < MIN_ITEMS, chưa patchNodeStatesOcc |
| AC-3 (index coerce) | `gradeItem(answerKey, choice)` — coerce String(answerKey)===String(choice) |
| AC-4 (filter MCQ) | loadCuratedItems `.find().filter(itemType==='mcq')` |
| AC-5 (omission) | adapter grade `total=|bound|`, câu thiếu bỏ (không map nil) → score < total |
| AC-6 (ignore correct) | adapter.run() cấm rơi về `Boolean(a.correct)`, chỉ dùng serverAnswerKey |
| AC-7 (idempotency) | `ActivityResult` ledger check (userId, attemptNonce) → trả cached result |
| AC-8 (re-start guard) | startNode guard MASTERED → 409 NODE_ALREADY_MASTERED (hoặc allow) |
| AC-9 (whitelist config) | publicItem + activityDescriptor strip sensitive field |
| AC-10 (flashcard resolve) | resolveFlashcardDeck() + bindCards + cardDTO map |
| AC-11 (deck anti-leak) | resolveFlashcardDeck() check ownerType==='system' → 409/422 |
| AC-12 (rating bound) | adapter grade flashcard từ bound cardIds, `total=|bound|` |
| AC-13 (missing rating) | rating < bound → miss count, score = correct/total |
| AC-14 (HARD gate) | publish validate `gatePolicy:'HARD'` + nodeKind ∈ {FLASHCARD, VIDEO} → 422 HARD_GATE |
| AC-15 (idempotency) | idempotent check → cached |
| AC-16 (signed URL) | signMediaUrl(ref, 6h) → URL format `?exp=...&sig=...` |
| AC-17 (external URL) | parse slug URL → pass-through |
| AC-18 (missing file) | local.exists(category, file) → false → 404 VIDEO_SOURCE_MISSING |
| AC-19 (invalid ref) | ref format validate → 400 VIDEO_NO_SOURCE |
| AC-20 (watchedRatio >=) | adapter grade watchedRatio >= minWatchRatio → passed |
| AC-21 (watchedRatio <) | adapter grade watchedRatio < minWatchRatio → failed |
| AC-22 (VIDEO HARD) | publish validate → 422 VIDEO_HARD_GATE |
| AC-23 (video idempotent) | idempotent check |
| AC-24 (Verify button) | admin UI node editor + Verify endpoint |
| AC-25 (publish validate) | publish check itemIds/deck/video source + HARD gate |
