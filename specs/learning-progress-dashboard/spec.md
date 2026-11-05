---
feature: learning-progress-dashboard
status: approved
tier: lite
owner: BA
created: 2026-08-25
affects: [exe-api, exe-web]
---

# Spec — Dashboard theo dõi tiến độ học đơn lẻ

## 1. Mục tiêu & lý do

Học viên đang học rải rác 7 tính năng đơn lẻ nhưng **không có chỗ nào tổng hợp tiến độ**. Dashboard hiện tại chỉ có welcome + vài card hardcode (`exe-web/src/components/features/dashboard/DashboardRail.tsx` — `DailyQuizCard` no-backend). Kết quả: học viên không biết "hôm nay cần học gì" và "đã đi được bao nhiêu %".

Feature này in ra 1 màn dashboard cho học viên tự theo dõi:
1. **Hôm nay cần học gì** — danh sách việc sinh riêng từ từng tính năng, xếp đến-hạn trước.
2. **% từng phần** — mỗi tính năng 1 thanh %, mỗi cái có "đường thẳng" (mẫu số) riêng phù hợp bản chất.
3. **1 streak tổng + % tổng** — 1 con số động viên duy nhất.

Nguyên tắc lõi: **mỗi tính năng phải có 1 "đường thẳng" (done/total) đo được**, kể cả tính năng mở.

## 2. Phạm vi — 7 tính năng

Gồm: Thẻ từ vựng, Luyện phát âm (IPA), Nói với AI (Talk), Thực hành Listening, Luyện tập Reading, Luyện đề thi thử, Ngữ pháp.

### Bảng "đường thẳng" mỗi tính năng (đã chốt)

| Tính năng | baselineType | done / total (đã verify path:line) | Chỉ số phụ | Task hôm nay |
|---|---|---|---|---|
| Thẻ từ vựng | mastery | `mastered / Σ deck.cardCount`, mastered = state='review' & stability≥21 (`flashcard-scheduler.service.js:88`) — gộp mọi deck | streak flashcard | số thẻ due hôm nay |
| Luyện phát âm | coverage | `count(status∈{completed,mastered}) / count(lesson published)` (`ipa-progress.model.js`) | avg overallScore | needsReview + bài not_started kế |
| Ngữ pháp | mastery | `itemsMastered / itemsPublished`, mastered = best===1.0 (`grammar-practice.service.js:58-72`) — dùng lại `/mastery/summary` | dueReview | câu đến hạn ôn |
| Listening | coverage | `passagesDone / total published`; **done = đủ item submit AND accuracy ≥ 50%** (`listening-item-attempt.model.js` + `listening-passage.model.js`) | avg accuracy | 1 passage mới / retry accuracy thấp |
| Reading | coverage | `distinct stimulusId đã submit / total reading-bank published` (`reading-attempt.model.js:26`) | avg best percent | passage kế / retry điểm thấp |
| Luyện đề | coverage | `distinct testPaperId status='submitted' / total published` (`practice-attempt.model.js`) | best percent + weakByItemType | resume in-progress / đề kế / drill weak |
| Nói với AI | coverage (scenario) | `scenariosTried / 4` (coffee, interview, travel, freetalk — `talk.scenarios.js:18`) | lượt hôm nay | thử kịch bản chưa luyện |

**IPA lưu ý:** `completed` = đạt điểm đủ 1 lần; `mastered` = đã ôn lại qua SRS vẫn đạt (nhớ lâu). Tử số coverage tính CẢ hai; số `mastered` hiện riêng làm chỉ số chất lượng.

### Streak & % tổng
- **Streak tổng** = số ngày liên tiếp có ≥1 hoạt động học bất kỳ. Suy ra ở dashboard từ hợp `activeDates` (30 ngày gần nhất) mà mỗi summary trả về — **không** tạo module aggregate/collection dùng chung (giữ 7 stream độc lập).
- **% tổng** = trung bình đơn giản `completionPct` của 7 tính năng.

## 3. Kiến trúc (đã chốt)

- Mỗi tính năng expose **1 endpoint summary riêng** trả về chung 1 DTO `FeatureProgress`. Dashboard FE gọi **fan-out song song** 7 endpoint (mẫu `useFlashcardGamification`).
- Khối "hôm nay học gì" **ghép ở FE** từ `todayTask` của mỗi summary — không thêm module aggregate ở backend.
- Chi tiết DTO, công thức, endpoint: `design.md`.

## 4. Ngoài phạm vi (out-of-scope)

- **Sổ lỗi sai & Phòng học chung** — user loại trừ rõ.
- **Đánh giá cuối hội thoại Talk** (chấm fluency/vocab/grammar) — hiện KHÔNG tồn tại (charter cấm grammar correction, `talk.scenarios.js:27`); là feature mới, tách spec riêng.
- **% tổng có trọng số** theo mục tiêu (IELTS/TOEIC) — dùng trung bình đơn giản trước; để ngỏ.
- **Đổi gamification/streak nội bộ Flashcard/IPA** — giữ nguyên, chỉ đọc.
- **Thay đổi bank nội dung** (thêm passage/đề) — không thuộc feature này.

## 5. Repo bị ảnh hưởng

| Repo | Thay đổi |
|---|---|
| `exe-api` | 6 endpoint summary mới (flashcard gộp deck, ipa, grammar-wrap, listening, reading, test-paper) + Talk activity tracking (model mới + ghi ở `POST /talk/turn`) + Talk summary |
| `exe-web` | Dashboard: fan-out 7 summary, 7 thanh %, khối "hôm nay", streak tổng, % tổng |

## 6. Acceptance Criteria

Backend (mỗi endpoint auth-required, trả `FeatureProgress` đúng DTO):
- [ ] `GET /flashcard/me/summary` trả `completionPct = round(Σmastered/Σcardcount*100)` gộp mọi deck của user; `todayTask.count` = số thẻ due ≤ now; `activeDates` = ngày có ReviewLog 30d.
- [ ] `GET /ipa/summary` trả `completionPct = round(count(completed|mastered)/count(published)*100)`; `secondary` = avg overallScore; `todayTask.count` = số bài needsReview.
- [ ] `GET /grammar/summary` map từ `getMasterySummary` hiện có → `completionPct = round(Σmastered/Σpublished*100)`; `todayTask.count` = dueReview.
- [ ] `GET /listening/summary`: passage done khi **mọi item của passage đã submit VÀ avg accuracy ≥ 0.5**; `completionPct = passagesDone/totalPublished`.
- [ ] `GET /reading/summary`: `completionPct = distinct stimulusId submitted / total reading-bank published`; `secondary` = avg best percent/passage.
- [ ] `GET /test-paper/summary`: `completionPct = distinct testPaperId submitted / total published`; `secondary.best` = max percent; `weakItemTypes` gộp từ `weakByItemType`.
- [ ] `POST /talk/turn` ghi lại `{userId, scenarioId, date}` (idempotent theo ngày+scenario cho activeDates); `GET /talk/summary` trả `scenariosTried/4` + `todayTurns` + `activeDates`.
- [ ] Mỗi summary trả `activeDates: string[]` (ISO date, ≤30 phần tử) để dashboard tính streak.

Frontend:
- [ ] Dashboard gọi 7 summary song song, render 7 thanh % với done/total + đơn vị + "còn thiếu".
- [ ] Khối "hôm nay cần học gì" liệt kê `todayTask` từng tính năng, ẩn task rỗng, xếp đến-hạn (flashcard due, ipa review, luyện đề in-progress) lên đầu.
- [ ] Streak tổng = ngày liên tiếp từ hợp `activeDates`; % tổng = trung bình 7 `completionPct`.
- [ ] Loading/skeleton mỗi card độc lập (1 endpoint lỗi không vỡ cả dashboard).

## 7. Quyết định & ghi chú

- **% tổng = trung bình đơn giản** 7 `completionPct` (đã chốt 25/08, không trọng số).
- **Luyện đề chỉ tính `PracticeAttempt`**, gồm cả `status='expired'` đã chấm (nộp quá giờ / auto-submit hết giờ vẫn là đề hoàn thành) — chốt 25/08.
- **Streak & activeDates theo giờ VN (UTC+7)** cho cả 7 tính năng (`utils/vn-date.js`) — chốt 25/08.
- **completionPct clamp ≤100** (nội dung bị gỡ sau khi học không đẩy % >100).
- Grammar: KHÔNG có `todayTask` due-review (module không có lịch SRS) — khác acceptance gốc §6, đúng thực tế.
- Talk activeDates: ≥1 lượt bất kỳ trong ngày = active.
- `total` (mẫu số coverage) chốt theo bank published tại thời điểm gọi → khi bank tăng, % loãng dần. Chấp nhận, luôn hiển thị done/total tuyệt đối kèm % để tránh hiểu lầm.
