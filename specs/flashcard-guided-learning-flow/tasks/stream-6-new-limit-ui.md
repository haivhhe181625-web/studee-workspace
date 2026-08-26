---
stream: 6
feature: flashcard-guided-learning-flow
repo: exe-web
status: todo
---

# Stream 6 — Setting new-limit UI (5/10/20)

**Mục tiêu:** control cho học viên chỉnh số từ mới/ngày; lưu qua BE (stream 2).

## Đọc trước
- Endpoint setting của stream 2 (PATCH profile / flashcard settings) + shape.
- `design.md §1, §4`. Nơi đặt: trong trang study (gọn) — `design.md §7`.

## Tasks
- [ ] Service + hook `useUpdateNewLimit()` (mutation) gọi PATCH của stream 2; invalidate today-session query sau khi đổi.
- [ ] Control chọn **5 / 10 / 20** (segmented / select nhỏ), hiện giá trị hiện tại (đọc từ today-session `newLimit` hoặc profile).
- [ ] Đặt trong trang study (gần nút "Học hôm nay") — nhỏ gọn, không phá layout.
- [ ] Test: đổi limit → gọi mutation đúng giá trị; UI phản ánh.

## Acceptance
- [ ] Chỉ cho {5,10,20}; đổi xong today-session refetch (newCards cập nhật theo limit mới).
- [ ] Token app-*, không any/hex.

**Status khi xong:** DONE + tóm tắt.
