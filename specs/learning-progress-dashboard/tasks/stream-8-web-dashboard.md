---
stream: 8
feature: learning-progress-dashboard
repo: exe-web
status: done
---

# Stream 8 — Dashboard FE (fan-out 7 summary)

**Mục tiêu:** dashboard render "hôm nay học gì" + "% từng phần" + streak tổng + % tổng. Bắt đầu ngay với mock adapter theo DTO; nối API thật khi stream 1-7 xong.

## Đọc trước
- `exe-web/src/app/(main)/dashboard/page.tsx` — entry (hiện chỉ `<DashboardWelcome/>`).
- `exe-web/src/components/features/dashboard/DashboardWelcome.tsx` — dùng `useAdaptivePath, useCoach, useStudentModel, useFlashcardGamification`; có `CEFR_PERCENT, CORE_SKILLS`.
- `exe-web/src/components/features/dashboard/DashboardRail.tsx` — `DailyQuizCard` HARDCODE (thay bằng data thật), `ReviewMistakesCard`.
- `exe-web/src/components/features/dashboard/SkillProgress.tsx`, `StatTile.tsx` — tái dùng cho thanh %.
- `design.md §1` (DTO), `§5-§6` (streak + composition). Mockup tham chiếu: `plans/visuals/dashboard-hoc-don-le-mockup.html`.

## DTO (khớp BE)
Tạo type `FeatureProgress` (design.md §1). 7 feature: flashcards, ipa, grammar, listening, reading, exam, talk.

**7 endpoint THẬT (đã verify sau khi BE chạy xong — dùng đúng path này):**
- `GET /api/flashcards/me/summary`
- `GET /api/ipa/summary`
- `GET /api/grammar-practice/summary`
- `GET /api/listening/summary`
- `GET /api/reading/summary`
- `GET /api/test-papers/summary`
- `GET /api/talk/summary`

## Tasks (TDD nhẹ / component)
- [ ] Tạo type `FeatureProgress` + hằng map feature→{nhãn VI, màu, href, icon}.
- [ ] Hook `useFeatureProgress()`: React Query fan-out 7 `GET .../summary` song song; mỗi query độc lập (skeleton/error riêng). Mock adapter theo DTO để chạy trước khi có API.
- [ ] Util `computeTotalStreak(features)`: union `activeDates`, đếm ngày liên tiếp lùi từ hôm nay (hôm nay chưa học thì tính từ hôm qua). Unit test util. → verify: `npm test` (exe-web)
- [ ] Component "% từng phần": 7 card từ `FeatureProgress` (thanh %, `done/total unit`, secondary, "còn thiếu"=total-done). Tái dùng `SkillProgress`/`StatTile`.
- [ ] Component "Hôm nay cần học gì": gộp `todayTask` (bỏ rỗng), sort ưu tiên flashcards.due > ipa.review > exam.in-progress > còn lại.
- [ ] Overall: ring = trung bình 7 `completionPct`; streak chip từ util. Thay `DailyQuizCard` hardcode.
- [ ] Ráp vào `dashboard/page.tsx` (hoặc trong `DashboardWelcome`), giữ layout mockup.

## Acceptance
- [ ] 7 query song song, 1 lỗi không vỡ cả trang.
- [ ] Streak tổng đúng (test: activeDates liên tiếp/ngắt quãng).
- [ ] % tổng = trung bình 7 completionPct.
- [ ] Task rỗng bị ẩn; task đến-hạn lên đầu.

**Status khi xong:** DONE + tóm tắt. Nếu path/href tính năng khác thực tế → ghi lại trong PR.
