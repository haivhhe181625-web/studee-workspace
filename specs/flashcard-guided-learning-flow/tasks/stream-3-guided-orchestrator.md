---
stream: 3
feature: flashcard-guided-learning-flow
repo: exe-web
status: todo
---

# Stream 3 — Guided orchestrator (`GuidedStudyRunner`)

**Mục tiêu:** máy trạng thái điều phối introduce→practice-new→review-due, dùng lại 4 runner. Không đổi cơ chế từng lượt.

## Đọc trước
- `exe-web/src/components/features/flashcard/`: `StudySessionRunner.tsx`, `StudyRunnerContainer.tsx`, `ModePicker.tsx`, `FlipCardRunner/TypeRunner/QuizRunner/MatchRunner.tsx`, `SessionSummaryView.tsx`, hook `useStudyQueue`.
- `exe-web/docs/flashcard/fe-logic.md` (luồng phiên + 4 chế độ + luật MatchRunner).
- `design.md §3` (máy trạng thái). Contract today-session: `design.md §2`.

## Máy trạng thái (design §3)
LOAD(GET /decks/:id/today) → [newCards>0: INTRODUCE(practice=false) → PRACTICE_NEW(Match→Quiz→Type, practice=true)] → REVIEW_DUE(dueCards, mode theo độ chín, practice=false) → SUMMARY. Bỏ qua bước rỗng; cả hai rỗng → "Hôm nay xong" + link Luyện tự do.

## Cờ practice (design §1) — QUAN TRỌNG
- **INTRODUCE**: `POST /cards/:id/review` với `practice=false` (lượt thật duy nhất cho thẻ mới; đẩy Độ sâu).
- **PRACTICE_NEW** (Match→Quiz→Type): `practice=true` — CHỈ củng cố, KHÔNG advance FSRS, KHÔNG cộng XP. (Đừng gửi practice=false ở đây — thổi phồng stability.)
- **REVIEW_DUE**: `practice=false` (ôn thật).

## Mode theo độ chín (REVIEW_DUE, design §3)
Chọn mode theo `stability` của thẻ due (tái dùng 3 dải): `<7d`→Flip/Match · `7–21d`→Quiz · `≥21d`→Type. Fallback: Match cần ≥4–6 thẻ (thiếu→Flip/Quiz); Type cần đáp án gõ được (cụm dài→Quiz).

## Tasks (TDD nhẹ / component)
- [ ] Hook/service `useTodaySession(deckId)` gọi `GET /flashcards/decks/:deckId/today` (mock adapter theo contract để chạy trước).
- [ ] `GuidedStudyRunner`: quản lý phase + hàng đợi; INTRODUCE dùng `IntroduceCard` (stream 4, gửi practice=false); PRACTICE_NEW chạy thẻ mới qua Match→Quiz→Type với **practice=true**; REVIEW_DUE chọn mode theo độ chín, practice=false. Mỗi lượt gọi `POST /cards/:id/review`.
- [ ] Helper `pickModeByStability(stability)` (thuần hàm, test riêng) + fallback thiếu thẻ.
- [ ] Tổng kết dùng lại `SessionSummaryView` (X mới + Y ôn); thêm "Hoàn thành bộ N từ hôm nay" + "ngày mai ôn Y thẻ".
- [ ] Test: phase transitions (newCards rỗng bỏ introduce; dueCards rỗng bỏ review; cả hai rỗng → done); practice flag đúng theo phase; `pickModeByStability` + fallback. Giữ test 4 runner cũ XANH.

## Acceptance
- [ ] Không đổi contract `POST /cards/:id/review` + cơ chế 4 runner.
- [ ] Practice flag đúng: introduce/due=false, practice-new=true.
- [ ] Mode REVIEW_DUE khớp độ chín + fallback khi thiếu thẻ.
- [ ] Thứ tự phase đúng; bỏ qua phase rỗng; không double review 1 lượt.

**Status khi xong:** DONE + tóm tắt.
