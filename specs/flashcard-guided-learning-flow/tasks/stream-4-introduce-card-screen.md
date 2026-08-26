---
stream: 4
feature: flashcard-guided-learning-flow
repo: exe-web
status: todo
---

# Stream 4 — Màn "Giới thiệu từ mới" (`IntroduceCard`)

**Mục tiêu:** component giới thiệu 1 từ mới: hiện đủ thông tin, không chấm, nút "Tôi đã nhớ" → ghi review đầu.

## Đọc trước
- `exe-web/src/components/features/flashcard/FlashcardSurface.tsx`, `FlipCardRunner.tsx`, `use-card-audio-player.ts` (phát audio).
- Card shape (front=từ, back=nghĩa, example, audio) — xem `types/flashcard.types.ts`.
- `design.md §3` (INTRODUCE), contract review `POST /cards/:id/review`.

## Tasks
- [ ] `IntroduceCard.tsx`: hiện **word + nghĩa + ví dụ + phát audio** cùng lúc (không ẩn/không chấm). Nút **"Tôi đã nhớ"** → callback `onIntroduced(cardId)` (orchestrator gọi review đầu **`practice=false`** — lượt thật đẩy new→learning + Độ sâu, rating "good" mặc định) → sang thẻ kế.
- [ ] Tái dùng `FlashcardSurface` + audio player, KHÔNG chép lại logic audio.
- [ ] A11y: nút rõ ràng, phát audio có nút bấm (không autoplay gây khó chịu — hoặc autoplay 1 lần tuỳ pattern hiện có).
- [ ] Test render: hiện đủ word/nghĩa/ví dụ; bấm "Tôi đã nhớ" gọi onIntroduced đúng cardId.

## Acceptance
- [ ] Không chấm điểm ở bước giới thiệu; chỉ hiển thị + xác nhận đã nhớ.
- [ ] Token app-*, không any/hex.

**Status khi xong:** DONE + tóm tắt (rating dùng cho review đầu).
