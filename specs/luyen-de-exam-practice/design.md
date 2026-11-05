# Design: Luyện đề (IELTS/TOEIC Exam Practice) — Phase 1

- **Ngày:** 2026-08-14
- **Tác giả:** Tech Lead Agent
- **Spec:** `spec.md` (scope chốt 2026-08-14)
- **Trạng thái:** Nháp — chờ Technical Review

## 1. Nguyên tắc thiết kế

- **Tái dùng, không xây lại.** `TestPaper` + `PracticeAttempt` là lớp mỏng trên `modules/assessment` (item bank, `grade.js`) và `modules/review` (SRS). KHÔNG đụng CAT/MST engine, KHÔNG sửa exam runner lõi.
- **Reference, không copy item** (DRY): `TestPaper.frozenItemIds` giữ ObjectId như `LessonQuiz.resolvedItemIds`.
- **Surgical**: thêm 1 field (`explanation`) vào `AssessmentQuestion`; 1 giá trị enum (`practice`) vào `learning_events.source`. Không refactor ngoài phạm vi.
- **Module 5-file** `modules/test-paper/`; admin qua factory; business logic ở service.

## 2. Kiến trúc tổng thể

```
[exe-admin]  đóng gói đề ──▶ POST /api/admin/test-papers ──▶ test-paper.service.assemble()
                                                                 │ query assessment_questions theo blueprint
                                                                 │ freeze frozenItemIds (atomic single-doc)
                                                                 ▼
                                                          TestPaper (status: draft→published)

[exe-web] Luyện đề:
  GET /api/test-papers            → list published papers (theo exam/skill/targetBand)
  POST /api/test-papers/:id/start → tạo PracticeAttempt (snapshot itemIds + shuffle) → trả items (KHÔNG answerKey/explanation)
  POST /api/practice-attempts/:id/submit
        → grade.gradeObjective() từng item (reuse)
        → tính score + breakdown theo itemType
        → emit learning_events (source:'practice') cho câu R/L
        → trả result: score, per-item {correct, answerKey, explanation}, weakByItemType

[modules/review] (không sửa): getReviewQueue/enqueueMistakes tự nạp câu sai practice
   ⚠️ chỉ re-drill item stimulusId==null; câu passage track-only (P1 chấp nhận)
```

## 3. Khung đề (blueprint) — cấu trúc mới

**KHÔNG dùng `constants/blueprint.js`** (đó là POOL-coverage cho CAT). Tạo registry mới: `modules/test-paper/paper-blueprint.js`.

Mỗi blueprint = template section-based:

```js
// paper-blueprint.js (shape)
{
  key: 'ielts-academic-reading',
  exam: 'ielts', skill: 'reading', variant: 'academic',
  totalItems: 40, timeLimitSec: 3600,
  sections: [
    { order: 1, difficultyTier: 'easy', cefr: 'B1',
      itemTypes: ['true_false_ng','matching_information','sentence_completion'], itemCount: 13 },
    { order: 2, difficultyTier: 'mid', cefr: 'B2',
      itemTypes: ['matching_headings','mcq','cloze','matching_features'], itemCount: 13 },
    { order: 3, difficultyTier: 'hard', cefr: 'C1',
      itemTypes: ['yes_no_ng','matching_sentence_endings','mcq','short_answer'], itemCount: 14 },
  ],
  targetBandRange: '5.5-7.5',
}
```

4 blueprint P1: `ielts-academic-reading`, `ielts-listening` (4 section S1-S4), `toeic-listening` (P1×6,P2×25,P3×39,P4×30), `toeic-reading` (P5×30,P6×16,P7×54).

**Hàm `validatePaper(blueprint, items[])`**: check số lượng + itemType per section khớp khung → dùng cả lúc assemble (admin) và test. Đây là AC#1.

> Lưu ý mapping enum: một số itemType trong blueprint (`matching_information`, `matching_features`, `yes_no_ng`, `matching_sentence_endings`) CHƯA có trong `ITEM_TYPES` hiện tại (`assessment-question.model.js:24`). Quyết định ở Technical Review: (a) thêm các enum này (surgical, cần cho phân loại đúng IELTS), hay (b) gộp tạm vào `matching`/`true_false_ng`/`mcq`. Khuyến nghị (a) — vì blueprint & phân loại điểm yếu theo dạng câu (AC#4) cần itemType chính xác.

## 4. Data model (chi tiết ở `data-model.md`)

- `TestPaper` (collection `test_papers`) — mới.
- `PracticeAttempt` (collection `practice_attempts`) — mới.
- `AssessmentQuestion.explanation` — thêm field (KHÔNG `select:false`; controller chặn trả lúc đang làm).
- `learning_events.source` — thêm enum value `'practice'`.

## 5. Chấm điểm & breakdown

- Reuse `grade.gradeObjective({itemType, answerKey, acceptedVariants, response})` — đã xử lý mcq/matching/cloze/tf_ng… (cùng hàm mà `review.service.submitReviewAnswer` dùng).
- Score P1 = raw + %correct. **KHÔNG quy band cá nhân** (cần calibration → P2, xem M7-14). Để định vị: đối chiếu `TestPaper.targetBandRange` của đề ("đề nhắm 6.0-7.0, bạn đúng 28/40") — rẻ, có sẵn, không cam kết band chính thức.
- `weakByItemType` = group per-item correctness theo `itemType` → % đúng mỗi dạng. Đây là giá trị học tập chính của trang review.

## 6. Explanation field — leak control

- `explanation` trả cho client **chỉ ở** response `submit` (sau nộp) và `list review`. TUYỆT ĐỐI không ở `start`/lúc đang làm.
- Cơ chế: controller `start` dùng `.select('-explanation -answerKey')` (answerKey vốn `select:false`); `submit` mới populate explanation. Không dựa `select:false` cho explanation vì cần trả có điều kiện.

## 7. Review sau nộp — review-in-context (P1); Sổ lỗi defer P2

**Quyết định brainstorm 2026-08-17:** P1 KHÔNG tích hợp Sổ lỗi/SRS. Lý do: ~100% câu đề R/L là item passage (`stimulusId!=null`) mà review session bỏ qua (`review.service.js:195,259`) → Sổ lỗi sẽ hiện gần trống ("hứa mà không giao").

- **P1 = review-in-context:** câu sai xem lại NGAY trong màn review-sau-nộp của attempt, kèm passage/ngữ cảnh + đáp án + giải thích (self-contained trong `PracticeAttempt.submit` response — không phụ thuộc `modules/review`). Không emit `learning_events`, không đụng `learning-event.model.js` / `enqueueMistakes` ở P1.
- **P2 (M7-14 backlog):** append `'practice'` vào SOURCES + emit event (`countsTowardMastery:false`) + mở `enqueueMistakes` `$in:['quiz','practice']` + mở re-drill item passage trong `modules/review`. Đụng engine review → cần test hồi quy kỹ.

## 8. Content authoring (TOEIC)

- Reuse `gen-testlet.service` + `question-bank.admin.js`. Author ~200 item TOEIC (P1-7), status draft→pilot→active, examiner vet.
- Nếu thêm TOEIC itemType enum (photograph/q-response) → làm TRƯỚC khi gen hàng loạt.
- Đây là workstream nội dung song song, không chặn code entity/UX (có thể seed IELTS trước để demo).

## 9. Điều KHÔNG làm (chống scope-creep)

- Không sửa CAT/MST, không đụng `assemblyMode` engine.
- Không mở testlet-review trong `modules/review`.
- Không quy band chính thức, không adaptive/dynamic assembly, không entitlement gating.

## 10. Rủi ro & mitigation

| Rủi ro | Mitigation |
|---|---|
| Thêm itemType enum làm vỡ code phụ thuộc `ITEM_TYPES` | Grep usage `ITEM_TYPES`/`itemType` trước; enum chỉ mở rộng (append), không đổi giá trị cũ |
| `explanation` leak lúc đang làm | Test riêng: `start` response không chứa explanation/answerKey |
| Emit learning_events sai lệch mastery của adaptive | `countsTowardMastery` set đúng; source `practice` tách biệt; test regression adaptive |
| TOEIC bank rỗng chặn demo | Seed IELTS trước, TOEIC song song |
| Freeze paper không atomic (Mongo standalone) | Single-doc insert như `LessonQuiz` publish |

## 11. Technical Review — đã chốt (2026-08-17)

- **Q1 · itemType IELTS:** ✅ Append `matching_information, matching_features, yes_no_ng, matching_sentence_endings` vào `ITEM_TYPES`. Verified append-safe: enum chỉ dùng làm mongoose validator (`assessment-question.model.js:24,103`), không có switch/exhaustive phụ thuộc; `grade.js` switch theo giá trị string trực tiếp.
- **Q2 · source emit:** ✅ Thêm enum `'practice'` (KHÔNG reuse `'quiz'`) — giữ analytics quiz sạch, filter được. Mở `enqueueMistakes` `$in:['quiz','practice']`. Refine: event practice đặt `countsTowardMastery:false` (xem §7) để cô lập adaptive.
- **Q3 · hiển thị band:** ❌ P1 KHÔNG hiện band. Chỉ raw + %correct + `weakByItemType`. Mapping (`constants/cefr-mapping.js`, `band-scoring.js`) có sẵn → phase sau có thể bật nhãn *khoảng* CEFR kèm disclaimer. Quyết định thiên product, dễ đảo.
