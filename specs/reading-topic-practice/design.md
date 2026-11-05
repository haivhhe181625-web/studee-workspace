# Design: Reading — import ngân hàng + Luyện tập theo chủ đề

- **Ngày:** 2026-08-20 · **Trạng thái:** Nháp — chờ Tech Lead review · **Spec:** `spec.md`

## 1. Quyết định kiến trúc

| # | Quyết định | Lý do |
|---|---|---|
| D1 | Nạp vào bank `assessment` sẵn có (`AssessmentStimulus`+`AssessmentQuestion`), KHÔNG collection reading mới | DRY: 1 bank, 1 luồng vet. Ranh giới với bank placement (`Question`) vốn đã tách. |
| D2 | Import bằng **seed script ETL**, KHÔNG dùng `test-paper/import.service.js` | Bank là pool để `assemble`/bốc câu, không phải 1 đề (`import.service` set `frozenItemIds` = tạo TestPaper). |
| D3 | Suy độ khó từ `cefr`: passage.cefr → `difficultyTier` (A2/B1→easy, B2→mid, C1→hard) + `cefrBandHint`; item `irt.b` tạm (easy=-1/mid=0/hard=+1) | JSON `difficulty`/`discrimination`=null; theo đúng công thức `gen-testlet`. Calibrate thật để sau (`item-quality.service`). |
| D4 | Bỏ dedup Qdrant + moderation khi import; giữ vet người (`draft`→`active`) | 2 gate đó cho AI-generated; đây là curated nội bộ. |
| D5 | Điều hướng learner = chủ đề (coarse `topic` enum) → CEFR → chọn tự do. KHÔNG gate/unlock/IRT | Owner chốt; `irt.b` chưa calibrate. |
| D6 | Không đổi schema model assessment | Field sẵn có đủ dùng (KISS). |

## 2. Mapping JSON → model (seed script)

**Passage → AssessmentStimulus** (upsert theo `provenance.sourceRef = passage_id`, idempotent)

| JSON | Model |
|---|---|
| ghép `paragraphs[].text` (giữ nhãn A/B/C) | `body` |
| `cefr` → tier (D3) | `difficultyTier` (required) |
| `cefr` | `cefrBandHint` |
| — | `skill:'reading'`, `kind:'passage'`, `contentSource:'self-authored'`, `status:'draft'` |
| `passage_id` | `provenance.sourceRef` |
| — | `provenance.importBatch:'reading-bank-v1'` |

**Item → AssessmentQuestion** (upsert theo `(importBatch, stimulusId, order)`)

| JSON | Model |
|---|---|
| link stimulus theo `passage_id` | `stimulusId` |
| `order` | `order` |
| `type` → `itemType` (bảng dưới) | `itemType` |
| passage.cefr | `cefrLevel` |
| raw topic → TOPICS enum coarse (bảng dưới) | `topic` |
| `prompt` | `stem` |
| `payload.options` (choice) | `options` |
| đáp án theo payload.kind (dưới) | `answerKey` (`select:false`) |
| `blanks[].accepted_variants` | `acceptedVariants` (`select:false`) |
| `explanation` | `explanation` |
| tier → b (D3) | `irt.b` |
| — | `skill:'reading'`, `targetGoal:'ielts'`, `contentSource:'self-authored'`, `status:'draft'`, `provenance.importBatch` |

**itemType map:** `mc_reading|main_idea|vocab_context|reference`→`mcq`; `tfng`→`true_false_ng`; `ynng`→`yes_no_ng`; `match_info`→`matching_information`; `match_heading`→`matching_headings`; `fill_completion`→`sentence_completion`; `synonym`→`fill_blank`; `short_answer`→`short_answer`. (Đối chiếu enum: `assessment-question.model.js:24-45`.)

**answerKey theo payload.kind** (đối chiếu `test-paper/import-validate.js` `ANSWER_KEY_SHAPES`):
- `choice`: single → string id; multi (`mcq_multi`) → array id.
- `fill`: 1 blank → string; nhiều blank → array theo index.
- `matching`: array `correct_pairs` theo shape `matching_*`.

**topic coarse map:** invitation/weekend/cafe/library→`daily`; food/coffee/sleep→`health`; work→`workplace`; technology→`tech`; environment/nature→`environment`; history/language/fashion/kindness→`culture`. Không phủ → fallback `daily` + log.

## 3. Surface learner (api)

Namespace học viên (KHÔNG `/api/admin`). Đề xuất: **mở rộng `test-paper` hoặc module `reading` mới** — chốt khi scout (xem §4).

| Endpoint | Trả | Ẩn |
|---|---|---|
| `GET reading/topics` | nhóm chủ đề có stimulus reading `active` + count | — |
| `GET reading/passages?topic=&cefr=` | list stimulus (`title`/genre/cefr/topic) | answerKey |
| `GET reading/passages/:id` | `body` + items (`.select('-answerKey -explanation')`) | answerKey, explanation |
| `POST reading/passages/:id/submit` | score + đáp án + explanation (chấm `grade.js`) | — |

## 4. Điểm cần scout trước khi code (fork thật)

- **Attempt model:** tái dùng `PracticeAttempt` (`modules/test-paper/practice-attempt.model.js`) hay tạo attempt reading tối giản? Nếu `PracticeAttempt.testPaperId` required (gắn chặt TestPaper) → làm attempt riêng nhẹ. Default đề xuất: attempt reading tối giản (userId, stimulusId, answers[], score) nếu không tái dùng sạch.
- **Nơi đặt endpoint:** module `reading` mới (5-file pattern) vs mở rộng `test-paper`. Default: module `reading` mới cho tách concern.

## 5. Bảo mật
- `answerKey`/`explanation` `select:false`; controller list/detail `.select('-answerKey -explanation')`; chỉ trả ở `submit`. Đúng pattern exam-practice.
- Permission learner-namespace (không dùng string staff). Theo HARD RULE `exe-api/CLAUDE.md`.

## 6. Rủi ro
- answerKey sai shape → chấm sai. Mitigation: 1 test grade mẫu / mỗi payload.kind.
- Trùng câu luyện tập ↔ luyện đề (cùng bank): chấp nhận (đều practice-grade); nếu lo, ưu tiên `exposureCount` thấp — tối ưu sau.
