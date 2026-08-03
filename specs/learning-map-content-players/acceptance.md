# Acceptance Criteria: Learning Map Content Players

- **Spec:** `specs/learning-map-content-players/spec.md`
- **Ngày:** 2026-08-03
- **Trạng thái:** Đã build (spec hồi tố)

## Cách dùng file này

- Mỗi tiêu chí có ID (`AC-1`, `AC-2`, ...) để `design.md`, `contracts/`, và task tracking tham chiếu.
- Format Given/When/Then cho hành vi stateful; checklist cho structural requirement.
- Người implement đã đáp ứng; spec này ghi lại "phải đúng sao" để review + regression.

---

## Tiêu chí MCQ_QUIZ Player (IS-2, IS-5, IS-10)

### AC-1: Curated MCQ — resolve + bind + hide key

- **Given:** Node `MCQ_QUIZ` có `config.itemIds = ['id1','id2','id3','id4','id5']` (5 câu MCQ hợp lệ)
- **When:** Learner POST `/api/adaptive/map/node/:nodeKey/start`
- **Then:**
  - HTTP 200: `res.items[]` trả **chính xác 5 item, đúng thứ tự** config.itemIds
  - Mỗi item: `{ questionId, itemType:'mcq', stem, options[], media:{audioUrl,imageUrl}, skill, subskill, cefrLevel }`
  - **KHÔNG có** `answerKey`, `acceptedVariants`, `config.itemIds`, `config.answerKey` trong response
  - Binding Redis: set 5 item IDs, TTL 15 min (single-use)
  - Response.attemptNonce, expiry, sig: HMAC attempt token cho `/submit`

### AC-2: MCQ nhỏ/rỗng → NODE_NO_ITEMS (không set IN_PROGRESS)

- **Given:** Node `MCQ_QUIZ` có `config.itemIds=[]` (rỗng) HOẶC `config.itemIds=['invalid_id']` (không tồn tại)
- **When:** Learner POST `/api/adaptive/map/node/:nodeKey/start`
- **Then:**
  - HTTP 409: `{ "code": "NODE_NO_ITEMS", "message": "This node has no playable items" }`
  - Node status **KHÔNG** set IN_PROGRESS (vẫn LOCKED hoặc gì cũ)
  - FE nhận 409 → fallback SubmitSimulator Dialog

### AC-3: Index answerKey — grade on "choice"

- **Given:** Item: `{ questionId:'q1', options:['go','goes','going','went'], answerKey:1 }` (index dạng số)
- **When:** Learner POST `/api/adaptive/map/node/:nodeKey/submit` with `{ answers:[{questionId:'q1', choice:'1'}], ...hmac }`
- **Then:**
  - Grade via `gradeItem(1, '1')` → return `true` (coerce String(answerKey)===String(choice))
  - Score: 1/5 (1 câu đúng trong 5)
  - **Chỉ chấm trên binding set** (5 câu bound): `total=5`, không `answers.length`

### AC-4: Filter itemType — loại essay/cloze ra, chỉ MCQ

- **Given:** Node `config.itemIds = ['mcq1','essay1','mcq2']` (lẫn MCQ + Essay)
- **When:** `/start` resolve items
- **Then:**
  - Chỉ `res.items[]` gồm `itemType:'mcq'` (2 item)
  - Essay được skip (1 item)
  - Nếu còn ≥ 3 MCQ → 200 OK; nếu < 3 → 409 NODE_NO_ITEMS

### AC-5: Omission (câu thiếu đáp án) = sai

- **Given:** Bind 5 câu, learner submit 1 câu
- **When:** POST `/submit` với `answers:[{questionId:'q1',choice:'1'}]` (5-1=4 câu thiếu)
- **Then:**
  - Server key set = 5 keys; submitted answers = 1
  - `total=5, correct=1, score=0.2`
  - Result: `passed:false` (score < passRatio 0.8)
  - Các câu không submit trong binding **KHÔNG map nil** — chỉ count missing

### AC-6: Backdoor correct=true — bỏ qua

- **Given:** Learner submit `{ answers:[{questionId:'q1', choice:'0', correct:true}, ...], ...hmac }`
- **When:** `/submit` grade
- **Then:**
  - Server **HOÀN TOÀN BỎ QUA** field `correct` từ client
  - Chỉ dùng `choice` để compare với `serverAnswerKey[questionId]`
  - Nếu choice sai → score sai, không rescue bằng `correct:true`

### AC-7: Idempotency — replay cùng nonce

- **Given:** Learner submit 2 lần với cùng `attemptNonce`
- **When:** Lần 1: POST `/submit` → 200 `{ score:0.8, passed:true, node:{status:'COMPLETED'} }`
- **And:** Lần 2: POST `/submit` cùng nonce
- **Then:**
  - HTTP 200 `{ idempotent:true, result:{ score:0.8, passed:true }, ... }` (trả kết quả cũ)
  - Node status **KHÔNG tụt** (vẫn COMPLETED, không đổi về IN_PROGRESS)
  - Binding **KHÔNG** re-consume (GETDEL chỉ 1 lần)
  - Mastery signal **KHÔNG** double-count

### AC-8: Re-start MASTERED → guard

- **Given:** Node status = `MASTERED` (học viên đã vượt qua)
- **When:** Learner POST `/start` lại
- **Then:**
  - HTTP 409: `{ "code": "NODE_ALREADY_MASTERED" }` (hoặc allow with note — owner decide)
  - Hoặc allow nhưng note là "re-practice" (không upgrade mastery lần 2)

### AC-9: Whitelist config — không lộ itemIds

- **Given:** GET `/api/adaptive/map/node/:nodeKey` (preview node)
- **When:** Response serialize node
- **Then:**
  - `node.activity.config` KHÔNG chứa `itemIds`, `answerKey`, `studentModel`, field nhạy
  - Chỉ `{ itemCount:5, passRatio:0.8, mode:... }`
  - `/start` response tương tự: `config` đã whitelist

---

## Tiêu chí Flashcard Player (IS-3, IS-5, IS-6, IS-7)

### AC-10: Flashcard deck hệ thống — resolve + bind CardIds

- **Given:** Node `FLASHCARD_DECK` có `activityRef={ refCollection:'flashcard_decks', refId:'deckId123' }`, deck: `{ ownerType:'system', cardCount:20, ... }`
- **When:** Learner POST `/api/adaptive/map/node/:nodeKey/start`
- **Then:**
  - HTTP 200: `res.cards[]` trả **tất cả 20 card** từ deck
  - Card DTO: `{ cardId, term, meaning, ipa, example, imageUrl, audioUrl }`
  - **KHÔNG có** deck `ownerType`, meta admin-only
  - Binding Redis: set 20 cardIds, TTL 15 min (single-use)
  - Response: attemptNonce, expiry, sig (HMAC attempt)

### AC-11: Deck user-owned — anti-leak 409/422

- **Given:** Node `FLASHCARD_DECK` có `activityRef.refId` trỏ deck `ownerType:'user'` (learner private deck)
- **When:** Learner POST `/start`
- **Then:**
  - HTTP 409: `{ "code": "FLASHCARD_DECK_NOT_FOUND", "message": "... deck không hệ thống" }`
  - Learner thấy lỗi, không leaked deck nội dung

- **And:** Admin publish node với deck `ownerType:'user'`
- **Then:**
  - HTTP 422: `{ "code": "FLASHCARD_DECK_NOT_FOUND", message: "... phải hệ thống" }`
  - Publish blocked; admin phải đổi deck

### AC-12: Self-rate rating — bound CardIds chốt

- **Given:** Bind 20 cardIds, learner rate 20 cards: `{ ratings:[{cardId:'c1', rating:'good'}, ..., {cardId:'c20', rating:'easy'}] }`
- **When:** POST `/submit` with `{ ratings:[], attemptNonce, sig, expiry }`
- **Then:**
  - Server validate rating enum ∈ {again, hard, good, easy}
  - Chấm FSRS: count rating>='good' → pass (simplistic, full FSRS = future)
  - Score: `correct/total = 18/20` (18 cards 'good'/'easy')
  - **Chỉ chấm trên bound set** (20 cards): `total=20`, submit rating cho card không bound → bỏ qua

### AC-13: Missing rating = miss

- **Given:** Bind 20 cardIds, learner rate 15 cards
- **When:** POST `/submit` with 15 ratings
- **Then:**
  - `total=20` (bound count), `correct=15` (rating count)
  - **5 cards thiếu rating = miss** → score = 15/20
  - Result: `passed:false` (score < 0.8)

### AC-14: Flashcard gating HARD → publish fail

- **Given:** Node `FLASHCARD_DECK` với `gatePolicy:'HARD'` (flashcard là prereq của node khác HARD-gated)
- **When:** Admin publish map
- **Then:**
  - HTTP 422: `{ "code": "FLASHCARD_HARD_GATE", message: "Flashcard không thể làm HARD prerequisite" }`
  - Publish blocked; admin phải đổi `gatePolicy` → SOFT/NONE hoặc bỏ node này khỏi HARD chain

### AC-15: Flashcard idempotency

- **Given:** Learner submit 2 lần cùng `attemptNonce`
- **When:** Lần 1: `/submit` → 200 `{ score:0.9, passed:true }`; Lần 2: `/submit` cùng nonce
- **Then:**
  - HTTP 200 `{ idempotent:true, result:{ score:0.9, passed:true }, ... }`
  - Binding **KHÔNG** re-consume (GETDEL 1 lần); status KHÔNG tụt

---

## Tiêu chí Video Player (IS-4, IS-5, IS-7, IS-9)

### AC-16: Video internal ref — signed URL

- **Given:** Node `VIDEO_LESSON` có `activityRef={ slug:'course-video/lesson-01.mp4' }` (internal ref, file tồn tại)
- **When:** Learner POST `/api/adaptive/map/node/:nodeKey/start`
- **Then:**
  - HTTP 200: `res.video={ url:'https://cdn.signed/v1/...?exp=...&sig=...', minWatchRatio:0.8 }`
  - URL: signed 6h TTL, không bind userId (scheme nền tảng course-content)
  - `minWatchRatio`: lấy từ node config (default 0.8)

### AC-17: Video external URL — pass-through

- **Given:** Node `VIDEO_LESSON` có `activityRef={ slug:'https://youtube.com/watch?v=abc123' }`
- **When:** Learner POST `/start`
- **Then:**
  - HTTP 200: `res.video={ url:'https://youtube.com/watch?v=abc123', minWatchRatio:0.8 }`
  - URL **KHÔNG signed** (external, pass-through)
  - Player phát trực tiếp

### AC-18: Video internal ref không tồn tại → VIDEO_SOURCE_MISSING

- **Given:** Node `VIDEO_LESSON` có `activityRef={ slug:'course-video/missing.mp4' }` (file không tồn tại trong storage)
- **When:** Learner POST `/start`
- **Then:**
  - HTTP 404: `{ "code": "VIDEO_SOURCE_MISSING", message: "Video file not found" }`
  - Hoặc 409 Conflict tùy implementation

### AC-19: Video ref format sai → VIDEO_NO_SOURCE

- **Given:** Node `VIDEO_LESSON` có `activityRef={ slug:'invalid-ref' }` (không match internal format `course-video/*`, không URL http(s))
- **When:** Learner POST `/start`
- **Then:**
  - HTTP 400/409: `{ "code": "VIDEO_NO_SOURCE", message: "Video reference format invalid" }`

### AC-20: Video watchedRatio >= minWatchRatio → passed

- **Given:** Node `minWatchRatio:0.8`, video 10 min
- **When:** Learner submit `{ watchedRatio:0.85, attemptNonce, sig, expiry }`
- **Then:**
  - HTTP 200: `{ score:1.0, passed:true, ... }` (watchedRatio ≥ 0.8)
  - Node status: COMPLETED → unlock kế

### AC-21: Video watchedRatio < minWatchRatio → failed

- **Given:** Node `minWatchRatio:0.8`
- **When:** Learner submit `{ watchedRatio:0.5 }`
- **Then:**
  - HTTP 200: `{ score:0.5, passed:false, ... }`
  - Node status: IN_PROGRESS (không unlock)

### AC-22: Video gating HARD → publish fail

- **Given:** Node `VIDEO_LESSON` với `gatePolicy:'HARD'`
- **When:** Admin publish map
- **Then:**
  - HTTP 422: `{ "code": "VIDEO_HARD_GATE", message: "Video không thể làm HARD prerequisite" }`
  - Publish blocked

### AC-23: Video idempotency

- **Given:** Learner submit 2 lần cùng nonce
- **When:** Lần 1: `/submit` → `{ score:1.0, passed:true }`; Lần 2 cùng nonce
- **Then:**
  - HTTP 200: `{ idempotent:true, result:{ score:1.0, passed:true }, ... }`
  - KHÔNG double-score; node status **KHÔNG tụt**

---

## Tiêu chí chế độ chỉnh sửa & Publish Validation (IS-7, IS-9)

### AC-24: Node editor — Verify button cho VIDEO_LESSON

- **Given:** Node editor admin UI, node kind = VIDEO_LESSON
- **When:** Admin nhập `activityRef.slug` rồi click "Verify"
- **Then:**
  - Nếu internal ref: check file tồn tại → ✅ "Source found"
  - Nếu external URL: check format http(s) → ✅ "URL valid"
  - Nếu sai: ❌ "Invalid source"

### AC-25: Publish validation — tất cả 3 player type

- **Given:** Admin publish map
- **When:** System validate tất cả node trong map
- **Then:**
  - **MCQ_QUIZ:** `config.itemIds` không rỗng, ≥ 3 item active, tất cả `itemType:'mcq'` → ✅
  - **FLASHCARD_DECK:** `activityRef.refId` trỏ deck hợp lệ `ownerType:'system'` → ✅
  - **VIDEO_LESSON:** `activityRef.slug` format đúng + (nếu internal) file tồn tại → ✅
  - Gating: node FLASHCARD_DECK / VIDEO_LESSON + `gatePolicy:'HARD'` → ❌ 422 HARD_GATE
  - Publish fail nếu có error; admin sửa để pass

---

## Ngoài phạm vi kiểm thử

- Sinh nội dung động: random MCQ từ bank, shuffle flashcard, các flow này không thuộc 3 player (xem spec §Ngoài phạm vi IS-1).
- Upload media: learner upload video/audio → chứa ở bên ngoài scope này.
- Advanced FSRS: map node không persist Anki state (throwaway grade).
- Time-gate / seek-protection: video pure self-report, KHÔNG fraud-proof.
