---
stream: 1
feature: flashcard-guided-learning-flow
repo: exe-api
endpoint: GET /flashcards/decks/:deckId/today
status: todo
---

# Stream 1 — Today-session endpoint

**Mục tiêu:** 1 endpoint gói phiên hôm nay/bộ: từ mới (≤ limit − đã giới thiệu hôm nay) + thẻ due + counts.

## Đọc trước
- `exe-api/services/api/src/modules/flashcard/flashcard-srs.model.js` — fields: `state (new|learning|review|relearning)`, `reps`, `stability`, `due`, `deckId`, `cardId`.
- `flashcard.service.js`: `getDue(userId,{deckId,limit})` (~278) — thẻ due; `initSrsForCards` (state='new'). ⚠️ `getProgress` (~506) gộp `new` vào `learning` → **KHÔNG reuse** cho counts/progress; tự tính.
- `flashcard-review-log.model.js` — `createdAt` (server time, `timestamps:true`) cho introducedToday. **KHÔNG dùng `reviewedAt`** (client time, offline backdate).
- `flashcard-scheduler.service.js:88` — `isMastered = state==='review' && stability>=21`.
- `utils/vn-date.js` — `vnDayKey`.
- `design.md §1-§2` (counts + progress 2 lớp + công thức masteryScore ramp).

## Tasks (TDD)
- [ ] Test `getToday(userId, deckId)`: seed thẻ (new/learning/review<21/review≥21) + reviewLog; assert `newCards.length ≤ limit-introducedToday`, `dueCards`, `counts{new,learning,mastered,total}`, `progress{introduced,tiers,masteryScore}`, `introducedToday`. → `npx jest flashcard --runInBand`
- [ ] Service `getToday`:
  - `newLimit` = `User.profile.studyPlan.flashcardNewPerDay ?? 10`.
  - `introducedToday` = count thẻ của deck có review ĐẦU (min `createdAt` trong FlashcardReviewLog theo cardId) rơi vào `vnDayKey(min)===today`.
  - `newCards` = thẻ SRS state='new' của deck, limit = `max(0, newLimit - introducedToday)`.
  - `dueCards` = tái dùng `getDue(userId,{deckId})`, **map thêm `stability`** vào mỗi thẻ từ SRS rows đã query (join theo cardId) — FE cần cho mode theo độ chín. KHÔNG sửa `getDue`.
  - `counts` (1 vòng quét SRS rows, không reuse getProgress): new=state='new'; learning=`learning|relearning` **+ review&stability<21**; mastered=isMastered; total=deck.cardCount. Bất biến: `new+learning+mastered==total`.
  - `progress` (tiến độ 2 lớp — design §1):
    - `introduced` = count reps≥1, **clamp ≤ total** (orphan SRS row).
    - `tiers` = { remembering: `0<stab<7`, good: `7≤stab<21`, mastered: `stab≥21` } (để tô màu).
    - `masteryScore` = avg `depth(thẻ)/total` với ramp **liên tục** theo stability: `0` nếu new; `33+(min(stab,7)/7)·33` khi `<7`; `66+(clamp(stab-7,0,14)/14)·34` khi `≥7`. (KHÔNG trọng số bậc.)
- [ ] Controller + route `GET /decks/:deckId/today` (verifyToken) → envelope `{data}`.
- [ ] Smoke test route.

## Acceptance
- [ ] newCards giới hạn đúng theo ngày VN (đủ limit hôm nay → newCards rỗng); introducedToday theo `createdAt`.
- [ ] `counts` cộng đủ total; `masteryScore` ramp liên tục (thẻ mới học ~38%, không kẹt bậc); introduced clamp ≤ total.
- [ ] Không đổi getDue/scheduler/getProgress. Không lộ dữ liệu nhạy cảm.

**Status khi xong:** DONE + tóm tắt.
