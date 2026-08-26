---
stream: 2b
feature: flashcard-guided-learning-flow
repo: exe-api
status: todo
---

# Stream 2b — Bỏ thưởng gamification cho practice

**Mục tiêu:** XP/quest/streak CHỈ tính lượt thật (introduce + ôn due). Lượt `practice=true` không cộng gì → XP nói thật về trí nhớ, không đo âm lượng gõ.

## Đọc trước
- `flashcard-gamification.service.js` — `awardForReview(userId, {rating, correct}, now)`: hiện cộng XP + quest (`review_20`, `correct_10`) + streak cho MỌI lượt. dayKey đang là **UTC**.
- `flashcard.service.js:434` — nơi gọi `awardForReview` trong `applyReviewEvent`; cờ `evt.practice` / `isPractice` đã có (`schema.js:100`).
- `utils/vn-date.js` — `vnDayKey`.
- `design.md §1` (bỏ thưởng practice + lệch dayKey).

## Tasks (TDD)
- [ ] Test: review `practice=true` → XP/quest/streak KHÔNG đổi; `practice=false` → cộng bình thường. Streak/quest bucket theo ngày VN.
- [ ] `applyReviewEvent`: bỏ qua `awardForReview` khi `evt.practice` (hoặc `awardForReview` nhận cờ và no-op). Giữ ghi ReviewLog (`isPractice`) như cũ.
- [ ] Đổi dayKey của gamification (streak/quest) sang `vnDayKey` — nhất quán với coverage/introducedToday (stream 1). Kiểm streak cũ không vỡ (migration nhẹ: chấp nhận 1 ngày lệch khi chuyển, ghi rõ).

## Acceptance
- [ ] Luyện tự do / practice-new KHÔNG cộng XP/quest/streak.
- [ ] Introduce + ôn due vẫn thưởng đúng.
- [ ] Streak/quest dùng vnDayKey (không lệch với "hôm nay" của tiến độ).

## Rủi ro
- Đụng code gamification sẵn có (ngoài scope lite gốc). Surgical: chỉ 1 guard + đổi dayKey. Đổi dayKey UTC→VN dịch mốc "hôm nay" 7h → 1 lần chuyển có thể ảnh hưởng streak đang chạy; xác nhận với owner nếu nhạy cảm.

**Status khi xong:** DONE + tóm tắt.
