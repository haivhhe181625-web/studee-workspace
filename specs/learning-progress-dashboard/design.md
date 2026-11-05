---
feature: learning-progress-dashboard
status: approved
role: TechLead
created: 2026-08-25
approved: 2026-08-25
---

# Design — Dashboard tiến độ học đơn lẻ

Tài liệu này chốt **hợp đồng DTO dùng chung**, công thức đã verify path:line, và ranh giới 8 stream song song. Mỗi stream code theo DTO này mà không cần đọc code stream khác.

## 1. DTO dùng chung — `FeatureProgress`

Mọi endpoint summary trả về đúng shape này (BE build object, FE có type tương ứng):

```ts
type BaselineType = 'coverage' | 'mastery' | 'daily_activity';

interface FeatureProgress {
  feature: 'flashcards' | 'ipa' | 'grammar' | 'listening' | 'reading' | 'exam' | 'talk';
  baselineType: BaselineType;
  completionPct: number;              // 0..100, đã round
  done: number;                       // tử số
  total: number;                      // mẫu số ("đường thẳng"); 0 => completionPct=0
  unit: string;                       // 'thẻ' | 'bài' | 'điểm ngữ pháp' | 'passage' | 'đề' | 'kịch bản'
  secondary?: { label: string; value: string | number };
  todayTask?: { label: string; count: number; href: string };
  activeDates: string[];              // ISO 'YYYY-MM-DD', <=30 ngày gần nhất có hoạt động
}
```

Quy ước:
- **Envelope:** cả 7 endpoint trả `{ data: FeatureProgress }` (chuẩn chung, FE unwrap `data.data`).
- `total === 0` → `completionPct = 0` (không chia 0).
- `todayTask` bỏ qua (undefined) khi không có việc → FE ẩn.
- `activeDates`: mỗi feature tự suy từ collection của mình (distinct ngày có attempt/review 30 ngày gần nhất), **date-key theo giờ VN (UTC+7)** — cả 7 phải cùng múi giờ (dùng `utils/vn-date.js`) nếu không streak union bị lệch ngày. Dùng để dashboard tính **streak tổng** = số ngày liên tiếp tính từ hôm nay trong hợp các mảng (FE `utils/streak.ts` cũng theo UTC+7).

## 2. Công thức đã verify (nguồn code)

| feature | done | total | file:line nguồn |
|---|---|---|---|
| flashcards | Σ rows `state='review' && stability>=21` mọi deck user | Σ `deck.cardCount` các deck user có SRS | `flashcard.service.js:504-517`, `flashcard-scheduler.service.js:88` |
| ipa | count `ipa_progress{status∈completed,mastered}` | count IPA lesson `status='published'` | `ipa-progress.model.js:28`, `ipa.service.js:129` |
| grammar | Σ itemsMastered mọi grammarPoint | Σ itemsPublished | `grammar-practice.service.js:93-111` (`getMasterySummary`) |
| listening | distinct passageId done (đủ item submit && avg accuracy>=0.5) | count `ListeningPassage{status='published'}` | `listening-item-attempt.model.js`, `listening-passage.model.js:162` |
| reading | distinct stimulusId đã có attempt submit | count `AssessmentStimulus` reading-bank published (`READING_BANK_FILTER`) | `reading-attempt.model.js:26`, `reading.service.js:32` |
| exam | distinct testPaperId có PracticeAttempt `status∈{submitted, expired}` (expired = nộp quá giờ nhưng ĐÃ chấm) | count `TestPaper{status='published'}` | `practice-attempt.model.js:13-52`, `test-paper.model.js:19-35` |
| talk | distinct scenarioId đã có ≥1 lượt | 4 (`SCENARIO_IDS.length`) | `talk.scenarios.js:18` |

## 3. Endpoint list (8 stream)

| Stream | Endpoint (path THẬT sau mount `app.js`) | Loại | Ghi chú |
|---|---|---|---|
| 1 | `GET /api/flashcards/me/summary` | mới | gộp mọi deck; tái dùng logic `getProgress`, bỏ filter deckId |
| 2 | `GET /api/ipa/summary` | mới | aggregate `ipa_progress` theo status + count lesson published |
| 3 | `GET /api/grammar-practice/summary` | mới (wrap) | map `getMasterySummary` → FeatureProgress; không đổi logic tính |
| 4 | `GET /api/listening/summary` | mới | group `listening_item_attempts` theo passageId, áp rule done |
| 5 | `GET /api/reading/summary` | mới | distinct stimulusId + avg best percent |
| 6 | `GET /api/test-papers/summary` | mới | distinct testPaperId submitted + merge weakByItemType |
| 7 | `GET /api/talk/summary` + ghi activity ở `POST /api/talk/turn` | mới + sửa | model `talk-activity` mới |
| 8 | Dashboard FE (exe-web) | mới | fan-out 7, streak, % tổng, khối hôm nay |

> Path đã verify sau khi stream 1-7 chạy xong (mount prefix có sẵn: `flashcards`, `test-papers`, `grammar-practice` — số nhiều/tên module đầy đủ). FE (stream 8) PHẢI dùng đúng 7 path này.

Tất cả endpoint BE theo **5-file module pattern** (`project-context §3`), auth qua `verifyToken`, response qua `toDto()`, service throw `ApiError` + controller `asyncHandler`.

## 4. Talk activity tracking (stream 7 — chi tiết vì là phần mới nhất)

Talk hiện stateless (`talk.service.js:22` "No DB writes other than quota"). Thêm ghi nhận nhẹ để có `scenariosTried` + `activeDates` + `todayTurns`.

Model mới `talk-activity.model.js` (collection `talk_activity`), 1 doc / user×date:
```
{ userId, date: 'YYYY-MM-DD', scenarios: [scenarioId], turns: Number }
```
- Ghi ở `talk.service.turn()` sau khi turn thành công (không tính turn lỗi/refund): upsert `{userId,date}`, `$addToSet scenarios`, `$inc turns`. Idempotent theo turnId để retry không double-count (tái dùng ý tưởng quota idem — chỉ inc khi consumeQuota thực sự trừ).
- `scenariosTried` (mọi thời điểm) = distinct scenarios mọi doc user → không nằm trong 1 doc/ngày, cần aggregate `$addToSet` toàn bộ doc user.

`GET /talk/summary`:
- `done = scenariosTried.length`, `total = 4`, `unit='kịch bản'`.
- `secondary = {label:'lượt hôm nay', value: todayTurns}`.
- `todayTask` = kịch bản chưa luyện đầu tiên (nếu còn) → "Thử kịch bản <tên>".
- `activeDates` = distinct date 30 ngày gần nhất.

## 5. Streak tổng (stream 8 — FE)

```
allDates = union(feature.activeDates for 7 features)  // Set<'YYYY-MM-DD'>
streak = đếm ngày liên tiếp lùi từ hôm nay (hoặc hôm qua nếu hôm nay chưa học) có trong allDates
```
Thuần FE, không endpoint riêng. Không tạo collection activity dùng chung (giữ stream độc lập). Ghi chú tối ưu tương lai: nếu cần streak chính xác/nhẹ hơn có thể gom 1 `learning-activity` collection — ngoài scope.

## 6. Dashboard composition (stream 8)

- Fan-out 7 `GET .../summary` bằng React Query song song; mỗi card có skeleton/error riêng.
- Khối "% từng phần": 7 card, mỗi card = 1 `FeatureProgress` (thanh %, `done/total unit`, secondary, "còn thiếu" = `total-done`).
- Khối "hôm nay cần học gì": gộp `todayTask` (bỏ rỗng), sort ưu tiên: flashcards.due > ipa.needsReview > exam.in-progress > còn lại.
- Overall: ring = trung bình 7 `completionPct`; streak chip.
- Tái dùng component sẵn có: `SkillProgress.tsx`, `StatTile.tsx`, `DashboardRail.tsx` (thay `DailyQuizCard` hardcode).

## 7. Ranh giới file (đảm bảo song song không đụng)

Mỗi BE stream chỉ đụng thư mục module của nó `exe-api/services/api/src/modules/<feature>/` + đăng ký route trong file routes của chính module (đã mount sẵn ở `app.js`, không sửa `app.js`). Stream 8 chỉ đụng `exe-web/`. → 8 stream **không chia sẻ file**, chạy song song an toàn.

Phụ thuộc DUY NHẤT: cả 8 đọc DTO ở §1 tài liệu này (đã đóng băng). FE (stream 8) có thể bắt đầu ngay với mock adapter theo DTO, tích hợp thật khi endpoint sẵn sàng.

## 8. Quyết định đã chốt
- `test-paper`: **CHỈ tính `PracticeAttempt`** (đề thi thử). KHÔNG gộp `QuizAttempt` (quiz thuộc course). — chốt 25/08.
- **% tổng = trung bình đơn giản** 7 `completionPct` (không trọng số). — chốt 25/08.
- Listening ngưỡng accuracy 0.5: hằng số có tên trong code, chuyển app-setting sau nếu cần (không chặn).
