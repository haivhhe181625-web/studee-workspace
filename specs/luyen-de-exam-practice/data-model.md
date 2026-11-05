# Data Model: Luyện đề — Phase 1

- **Ngày:** 2026-08-14
- **Kèm:** `design.md`

## 1. TestPaper (mới) — collection `test_papers`

| Field | Type | Ghi chú |
|---|---|---|
| centerId | ObjectId ref Center, index, default null | multi-tenant; null = platform-wide |
| title | String, required | vd "IELTS Academic Reading — Test 01" |
| exam | String enum `['ielts','toeic']`, required | |
| variant | String enum `['academic','general']`, default null | ielts only |
| skill | String enum `['reading','listening']`, required | P1 R/L only |
| mode | String enum `['static','dynamic']`, default `'static'` | P1 chỉ static |
| blueprintKey | String, required | ref `paper-blueprint.js` key |
| frozenItemIds | [ObjectId ref AssessmentQuestion], default [] | đóng băng lúc publish (như LessonQuiz.resolvedItemIds) |
| targetBandRange | String, default null | vd "5.5-7.5" / "550-785" |
| timeLimitSec | Number, required | từ blueprint |
| status | String enum `['draft','published','archived']`, default `'draft'` | |
| createdBy | ObjectId ref User, default null | admin author |
| timestamps | | |

- Index: `{ exam:1, skill:1, status:1, centerId:1 }`.
- `toDto()`: trả nguyên (không có field nhạy cảm ở paper-level; item nhạy cảm nằm ở AssessmentQuestion).
- **Publish invariant**: chỉ set `status:'published'` khi `validatePaper(blueprint, frozenItems)` pass. Single-doc update (atomic).

## 2. PracticeAttempt (mới) — collection `practice_attempts`

| Field | Type | Ghi chú |
|---|---|---|
| userId | ObjectId ref User, required, index | |
| testPaperId | ObjectId ref TestPaper, required | |
| itemIds | [ObjectId], required | snapshot từ paper.frozenItemIds lúc start (đề static → = frozen; giữ snapshot để paper đổi không ảnh hưởng attempt đang chạy) |
| optionOrder | Map/Mixed, default null | shuffle order per item (nếu shuffle) — để chấm đúng |
| answers | [{ questionId, response (Mixed), correct (Bool), answeredAt }], default [] | |
| status | String enum `['in_progress','submitted','expired']`, default `'in_progress'` | |
| startedAt | Date, default now | |
| submittedAt | Date, default null | |
| score | { raw: Number, total: Number, percent: Number }, default null | |
| weakByItemType | [{ itemType, correct, total, percent }], default [] | breakdown |
| timestamps | | |

- Index: `{ userId:1, testPaperId:1, createdAt:-1 }`.
- `toDto()`: strip nothing đặc biệt ở attempt; nhưng khi trả lúc `in_progress` KHÔNG kèm answerKey/explanation của item (lấy item qua `.select('-explanation -answerKey')`).

## 3. AssessmentQuestion — thêm 1 field (existing model)

`modules/assessment/assessment-question.model.js`:

```js
// Sau field aiScoreVariance:
explanation: { type: String, default: null }, // rationale shown AFTER submit (review). NOT select:false — controller gates by phase.
```

- KHÔNG `select:false` (cần trả có điều kiện ở submit/review).
- `toDto()` giữ nguyên (explanation client-safe sau nộp; controller `start` phải `.select('-explanation')`).
- Migration: field optional default null → không cần backfill; item cũ explanation=null (admin bổ sung dần).

## 4. learning_events — thêm 1 enum value (existing model)

> **DEFER P2** (brainstorm 2026-08-17): P1 không tích hợp Sổ lỗi → không đụng model này. Giữ mô tả cho M7-6/M7-14.

`modules/adaptive/learning-event.model.js:13`:

```js
const SOURCES = ['assessment','lesson','quiz','speaking_drill','checkpoint','listening','flashcard','practice'];
//                                                                                                  ^^^^^^^^^^ thêm
```

- Chỉ append; không đổi giá trị cũ → không vỡ dữ liệu/validation hiện có.

## 5. review.service — mở rộng filter (existing, 1 dòng)

> **DEFER P2** (brainstorm 2026-08-17): P1 không đụng `modules/review`. Giữ mô tả cho M7-6.

`modules/review/review.service.js:33`:

```js
// từ:
{ userId, source: 'quiz', correct: false }
// thành:
{ userId, source: { $in: ['quiz', 'practice'] }, correct: false }
```

- Phần resolve `QuizAttempt` (refId) giữ nguyên: practice refId không khớp QuizAttempt → att=null → courseId/phaseKey null (code hiện tại đã handle null).

## 6. paper-blueprint.js (mới, không phải Mongo model) — `modules/test-paper/paper-blueprint.js`

Registry tĩnh (frozen object) + hàm:
- `PAPER_BLUEPRINTS`: map key → blueprint (4 key P1).
- `getBlueprint(key)`.
- `validatePaper(blueprint, items)`: trả `{ ok, errors[] }` — check tổng số item, số item + itemType per section.

## 7. Không đổi

- CAT/MST models, Assessment/AssessmentAttempt/Result, Stimulus — không sửa.
- review_schedule model, scheduler — không sửa.
