---
stream: 1
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /flashcard/me/summary
status: todo
---

# Stream 1 — Flashcard summary (gộp mọi deck)

**Mục tiêu:** endpoint trả `FeatureProgress` tổng hợp mastery flashcard trên TẤT CẢ deck user (hiện chỉ có per-deck `GET /decks/:deckId/progress`).

## Đọc trước (context)
- `exe-api/services/api/src/modules/flashcard/flashcard.service.js:504-518` — `getProgress` per-deck (công thức gốc: `mastered/total`, `total=deck.cardCount`).
- `exe-api/services/api/src/modules/flashcard/flashcard-scheduler.service.js:88` — `isMastered = state==='review' && stability>=21`.
- `exe-api/services/api/src/modules/flashcard/flashcard-srs.model.js` — fields `state, stability, due`.
- `exe-api/services/api/src/modules/flashcard/flashcard-review-log.model.js` — cho `activeDates`.
- `design.md §1` (DTO), `§2` (công thức).

## DTO trả về (feature='flashcards', baselineType='mastery')
- `done` = Σ SRS rows user có `isMastered`; `total` = Σ `deck.cardCount` các deck user có SRS row.
- `completionPct = total ? round(done/total*100) : 0`; `unit='thẻ'`.
- `secondary = {label:'streak', value: <flashcard gamification streak>}` (đọc `flashcard-gamification.model.js`).
- `todayTask` = nếu due>0: `{label:'Ôn thẻ đến hạn', count: dueCount, href:'/flashcards'}`; due = Σ rows `due<=now`.
- `activeDates` = distinct ngày (YYYY-MM-DD) có ReviewLog user, 30 ngày gần nhất.

## Tasks (TDD)
- [ ] Viết test service `getMeSummary(userId)`: seed 2 deck (cardCount 10 & 5), vài SRS rows (một số review+stability≥21, một số due), assert `done/total/completionPct/dueCount`. → verify: `npm test -- flashcard`
- [ ] Thêm `getMeSummary` vào `flashcard.service.js`: aggregate SRS rows user group theo deck, join `deck.cardCount`; đếm mastered/due; query ReviewLog distinct date 30d. Tái dùng `isMastered` từ scheduler (DRY).
- [ ] Thêm controller `getMeSummary` (asyncHandler) + `toDto` map sang `FeatureProgress`.
- [ ] Thêm route `GET /me/summary` vào `flashcard.routes.js` (verifyToken). Xác nhận prefix mount → path cuối `GET /flashcard/me/summary` (chỉnh label nếu mount khác).
- [ ] Smoke test route trả 200 + shape đúng. → verify: `npm test -- flashcard`

## Acceptance
- [ ] `completionPct` khớp `round(Σmastered/Σcardcount*100)`; deck không có card → không chia 0.
- [ ] `activeDates` ≤30, định dạng ISO date.
- [ ] Không sửa `app.js`, không đụng module khác.

**Status khi xong:** DONE + 1 câu tóm tắt.
