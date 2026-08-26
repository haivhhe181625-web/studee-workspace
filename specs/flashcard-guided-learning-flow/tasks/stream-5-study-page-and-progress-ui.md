---
stream: 5
feature: flashcard-guided-learning-flow
repo: exe-web
status: todo
---

# Stream 5 — Trang study (guided default + Luyện tự do) + UI tiến độ

**Mục tiêu:** `/flashcards/[deckId]/study` mặc định luồng dẫn dắt; giữ "Luyện tự do"; hiển thị **thanh 2 lớp Độ phủ + Độ sâu** + mục tiêu ngày.

## Đọc trước
- `exe-web/src/app/(main)/flashcards/[deckId]/study/page.tsx`, `StudyRunnerContainer.tsx` (nhánh ModePicker cũ).
- `DecksContainer.tsx`, `DeckReviewRow.tsx` (chỗ hiện mastery% — thay bằng thanh 2 lớp).
- `design.md §1 (tiến độ 2 lớp), §3-§4`. Contract `progress`: `design.md §2` (từ stream 1 today-session).

## Tasks
- [ ] Trang study: mặc định render `GuidedStudyRunner` (stream 3) với nút **"Học hôm nay"**; thêm link/nút **"Luyện tự do"** → giữ `StudyRunnerContainer`+`ModePicker` cũ (không xoá).
- [ ] UI tiến độ per-bộ — **thanh 2 lớp** từ `progress` (stream 1): nền amber `coveragePct` (= introduced/total) + overlay teal `masteryScore` (ramp liên tục); chip 3 dải **Đang nhớ/Nhớ tốt/Thành thạo** từ `tiers`; dòng **"còn X từ mới · Y thẻ ôn hôm nay"**. Đặt đầu trang study và/hoặc trên `DeckReviewRow` ở `/flashcards`.
- [ ] Loading/empty: chưa tải → skeleton; hôm nay xong → "Hôm nay xong rồi" + Luyện tự do.

## Acceptance
- [ ] Guided là mặc định; Luyện tự do vẫn vào được (không mất chức năng cũ).
- [ ] Thanh 2 lớp đúng: Độ phủ (không lùi) + Độ sâu (masteryScore ramp); chip 3 dải đúng số; mục tiêu ngày đúng.
- [ ] Token app-*, không any/hex.

**Status khi xong:** DONE + tóm tắt.
