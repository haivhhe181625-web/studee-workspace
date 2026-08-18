---
feature: llm-direct-testlet-generation
status: draft
created: 2026-08-18
repos: [exe-api]
epic: PRD-151 (M7 Luyện đề)
relates: luyen-de-item-bank-generation (Bước 2 — cung cấp coverage + gap-fill runner)
---

# Spec — Sinh testlet trực tiếp qua llm-proxy (không cần Scoring Engine)

## 1. Mục tiêu & lý do

Đường sinh câu hiện có (`gen-testlet.service`) route qua **Scoring Engine** (service standalone,
`SCORING_ENGINE_URL`). Engine đó **không được cấu hình** (URL trống) → `engine.generateTestlet` trả
**mock** (nội dung giả) → chạy gap-fill (Bước 2) hiện sinh ra rác, không dùng được.

Ngân hàng TOEIC rỗng, không có nguồn đề để import → **phải tự sinh**. OpenAI key đã có (trong llm service).
`llmProxy.chat({responseFormat:json})` gọi thẳng OpenAI được → **sinh trực tiếp khả thi, không cần engine**.

Kiến trúc: engine chỉ được gom vào vì **gate chất lượng "blind solver + judge"** (một AI tự giải câu để
kiểm đáp án) — không phải vì sinh cần chấm điểm. Feature này thay engine bằng: (1) generator gọi llm-proxy,
(2) **solver-check nhẹ** tự làm lại gate đó.

## 2. Phạm vi

- **BE (exe-api) — module mới `llm-generate-testlet.service.js`:**
  - `generateTestletViaLLM({ skill, tier, itemTypes, topic, cefrHint, targetGoal })` → build prompt theo
    skill/itemTypes/cefr → `llmProxy.chat` (JSON) → parse thành testlet shape mà `validateTestlet` sẵn có
    hiểu (`{ stimulus?, questions:[{itemType, stem, options, answerKey, acceptedVariants?, explanation}] }`).
  - **Solver-check gate** (mặc định BẬT): gọi LLM lần 2 tự giải các câu (blind), so `answerKey`; lệch →
    reject → retry. Bản rút gọn của solver/judge engine.
  - Trả `{ testlet, gates:{passed, errors} }` — **cùng hình dạng** `engine.generateTestlet` để cắm vào chỗ cũ.
- **Wire vào `gen-testlet.service.js` (surgical):** nhánh production hiện gọi engine → đổi thành:
  `engine.isEnabled() ? engine.generateTestlet(...) : generateTestletViaLLM(...)`. Khi engine tắt (hiện tại)
  → dùng generator llm thật thay vì mock. Phần còn lại (validateTestlet, gen-quality moderate+dedup,
  buildQuestionMeta, insert draft, TTS cho listening) **giữ nguyên**.
- **Prompt templates:** reading (passage + câu), listening (transcript + câu → downstream TTS), standalone
  (grammar/use_of_english). Skill-agnostic.
- **Sinh cho luyện đề:** reading + listening (targetGoal ielts/toeic), qua `gen-fill` (Bước 2) không đổi.

## 3. Ngoài phạm vi

- **TOEIC Part 1 (Photographs)** — cần ảnh; `AssessmentStimulus.kind` chỉ passage/audio (không image). Bỏ
  qua ở P1 (biến thể khung TOEIC Listening không Part 1, hoặc import ảnh riêng sau).
- **use_of_english cho luyện đề** — generator hỗ trợ được nhưng không thuộc khung luyện đề (reading/listening).
  On-demand cho placement là feature khác.
- **Sinh ảnh (image gen)** — không làm.
- **Bật lại Scoring Engine** — không (không có engine); feature này là đường thay thế.
- **Đổi schema bank** — không cần (bank đã chứa đủ; chỉ thiếu image-stimulus, cố ý bỏ qua).
- **BullMQ/queue trigger** — vẫn chạy qua ops script `fill-exam-practice-bank.js` như Bước 2.

## 4. Acceptance Criteria

- [ ] AC1 — `generateTestletViaLLM({skill:'reading', itemTypes:['mcq'], cefrHint:'B1', targetGoal:'toeic'})`
  → trả testlet hợp lệ (`validateTestlet` pass): có stimulus (reading) + ≥1 câu mcq, mỗi câu có
  stem/options/answerKey đúng shape theo itemType.
- [ ] AC2 — Solver-check: câu có `answerKey` mà solver giải ra khác → testlet bị reject (gates.passed=false),
  generateAndInsertTestlet retry; hết attempts → throw (không insert câu sai đáp án).
- [ ] AC3 — Khi engine tắt (`SCORING_ENGINE_URL` trống), `generateAndInsertTestlet` (không deps) dùng
  generator llm thật → item chèn có `contentSource='ai-generated-reviewed'`, `status='draft'`, `mocked` KHÔNG
  set (nội dung thật, không phải mock).
- [ ] AC4 — Reading (standalone Part 5: stimulusId null; passage Part 6/7: có stimulus) + listening
  (stimulus kind='audio' + transcript → TTS tạo audioUrl) đều insert đúng bank.
- [ ] AC5 — Item sinh có `targetGoal`, `cefrLevel` (=cefrHint), `itemType`, `explanation`, và
  `acceptedVariants` cho item open-text (short_answer/sentence_completion/fill_blank).
- [ ] AC6 — Chạy `fill-exam-practice-bank.js --max N` (mock LLM trong test) → gọi generator llm, KHÔNG gọi
  engine mock; report đúng.
- [ ] AC7 — Test không gọi LLM thật (DI `llmProxy` stub); full suite exe-api xanh, 0 regression gen-testlet/
  test-paper.
- [ ] AC8 — Chi phí có kiểm soát: generator + solver-check ≤ 2–3 lần gọi/testlet; `--max` cap tổng calls.

## 5. Repos ảnh hưởng

- **exe-api** — module generator mới + wire vào gen-testlet + prompt templates + test. gen-fill/coverage
  (Bước 2) không đổi. exe-web/admin không đụng.

## 6. Liên kết

- Bước 2 (coverage + gap-fill runner): `specs/luyen-de-item-bank-generation/`.
- Hạ tầng tái dùng: `assessment/gen-testlet.service.js` (validate/insert/TTS), `gen-quality.service.js`
  (moderate+dedup), `adapters/llm-proxy.adapter.js` (`chat`, `tts`, `moderate`).
- Model: `assessment-question.model.js`, `assessment-stimulus.model.js` (kind passage/audio — no image).

## Câu hỏi mở

- TOEIC Part 1: bỏ hẳn (khung 94 câu) hay giữ khung 100 + để trống Part 1 chờ import ảnh? Chốt khi review.
- Solver-check dùng model nào: cùng generator (`gpt-4o-mini`, rẻ) hay mạnh hơn (`gpt-4.1`, chắc hơn, đắt hơn)?
- use_of_english: có muốn sinh cho placement luôn trong phase này không, hay để riêng?
