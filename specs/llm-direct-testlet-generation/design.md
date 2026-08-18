---
feature: llm-direct-testlet-generation
status: draft
created: 2026-08-18
repos: [exe-api]
---

# Design — Direct llm-proxy testlet generator

## 1. Testlet contract (shape `validateTestlet` sẵn có yêu cầu)

Generator PHẢI trả object này (không transform thêm — `validateTestlet` + `buildQuestionMeta` ăn trực tiếp):

```js
{
  skill: 'reading'|'listening'|'use_of_english'|...,
  tier: 'easy'|'mid'|'hard',
  stimulus: {                 // BẮT BUỘC cho reading/listening; BỎ cho standalone (Part 5, use_of_english)
    kind: 'passage'|'audio',
    body: '<passage text>',   // kind=passage (reading)
    transcript: '<script>',   // kind=audio (listening) — downstream gen-testlet gọi TTS tạo audioUrl
  },
  questions: [{
    itemType: 'mcq'|'true_false_ng'|'cloze'|...,
    stem: '<đề>',
    options: ['A','B','C','D'],        // mcq...; bỏ cho open-text
    answerKey: <index number | string>, // mcq=index; tf=chuỗi; open-text=chuỗi đáp án chuẩn
    acceptedVariants: ['...'],          // BẮT BUỘC cho open-text (cloze/fill_blank/short_answer/sentence_completion)
    explanation: '<giải thích, hiện sau submit>',
  }],
}
```

Ràng buộc `validateTestlet`: stimulus body/transcript phải khớp readability tier (`tierReadabilityOk`);
mọi itemType ∈ ITEM_TYPES; non-subjective phải có answerKey; open-text phải có acceptedVariants; mọi
requestedType phải được cover. → Prompt phải ép đúng các điều này.

## 2. Module mới `llm-generate-testlet.service.js`

```
generateTestletViaLLM({ skill, tier, itemTypes, topic, cefrHint, targetGoal, deps }):
  1. messages = buildPrompt(skill, itemTypes, cefrHint||tier, topic, count)
  2. raw = await llmProxy.chat({ messages, model: GEN_MODEL, responseFormat: {type:'json_object'}, temperature })
  3. testlet = parseTestlet(raw)                 // JSON.parse + chuẩn hoá field
  4. gates = await solverCheck(testlet, { deps }) // §4
  5. return { testlet, gates }
```
- `deps.llmProxy` inject để test (stub) — không gọi OpenAI thật trong unit test.
- `GEN_MODEL` = `gpt-4o-mini` (config, override env).
- Nếu parse lỗi (LLM trả JSON hỏng) → gates.passed=false, errors=['unparseable'] → gen-testlet retry.

### buildPrompt (theo skill)
- **reading (passage)**: "Viết 1 đoạn đọc CEFR <level> chủ đề <topic> (~<words> từ), rồi <count> câu
  <itemTypes>. Trả JSON đúng schema…". Ép: mcq 4 lựa chọn, answerKey là index (0-based), explanation ngắn.
- **reading standalone (Part 5, không passage)**: "Viết <count> câu mcq hoàn thành câu (ngữ pháp/từ vựng)
  CEFR <level>, KHÔNG stimulus". → stimulus bỏ.
- **listening (audio)**: "Viết transcript hội thoại/độc thoại CEFR <level> chủ đề <topic>, rồi <count> câu.
  stimulus.kind='audio', stimulus.transcript=<script>". Downstream gen-testlet tự TTS.
- **use_of_english / grammar**: standalone mcq, gắn grammarPoint (tuỳ chọn).

Prompt tách file `llm-generate-prompts.js` (per-skill templates) để dễ chỉnh, giữ service < 200 dòng.

## 3. Wire vào gen-testlet (surgical)

`gen-testlet.service.js`, nhánh production hiện tại:
```js
// TRƯỚC:
const res = await engine.generateTestlet({...});
// SAU:
const res = engine.isEnabled()
  ? await engine.generateTestlet({...})
  : await generateTestletViaLLM({ skill, tier, itemTypes, topic, cefrHint: cefrBandHint, targetGoal, deps });
gen = res.testlet;
engineGateErrors = res.gates && !res.gates.passed ? (res.gates.errors || []) : [];
```
- Khi engine bật lại (tương lai) → tự dùng engine, không đụng. Khi tắt (hiện tại) → llm thật (KHÔNG mock).
- Phần sau (validateTestlet, gen-quality moderate+dedup, buildQuestionMeta, insertMany, TTS listening,
  embedAndIndex) **giữ nguyên** → gen-fill/coverage Bước 2 chạy được ngay, không sửa.
- `gen.mocked` KHÔNG set (nội dung thật) → gen-quality gates + embed CHẠY (khác mock bị skip).

## 4. Solver-check gate (thay solver/judge engine, rút gọn)

`solverCheck(testlet, {deps})`:
- Gọi LLM lần 2 (blind): đưa stimulus + từng câu (stem+options) **KHÔNG kèm answerKey**, yêu cầu trả đáp án.
- So với `answerKey`: mcq so index; tf/yes_no so chuỗi; open-text coi đúng nếu trùng answerKey hoặc 1
  acceptedVariant (case-insensitive).
- Ngưỡng: nếu ≥1 câu solver-model tự tin giải KHÁC đáp án → `gates.passed=false` (đáp án nghi sai) → reject
  → gen-testlet retry (tối đa `attempts`).
- `SOLVER_MODEL` config (mặc định `gpt-4o-mini` — rẻ; có thể override `gpt-4.1` cho chắc). Câu hỏi mở.
- Tắt được qua `deps`/flag để test nhanh.

**Chi phí/testlet:** 1 gen + 1 solver = **2 lần gọi** (+ retry khi fail). Moderate (gen-quality) +1. ~$0.01–0.02.

## 5. Model & cost

| Vai trò | Model mặc định | Ghi chú |
|---|---|---|
| Generator | `gpt-4o-mini` | JSON mode |
| Solver-check | `gpt-4o-mini` (mặc định) | override `gpt-4.1` nếu cần chắc |
| Moderation | (llm /moderate) | có sẵn |
| Dedup embed | `text-embedding-3-small` | cần OPENAI_API_KEY (singular) — nếu placeholder thì dedup mock |
| TTS (listening) | `gpt-4o-mini-tts` | có sẵn |

## 6. Test (không gọi LLM thật)

- Unit `llm-generate-testlet`: stub `deps.llmProxy.chat` trả JSON cố định → assert parse + shape hợp lệ.
- Solver-check: stub chat trả đáp án khớp → pass; trả khác → reject.
- Wire: `gen-testlet` với engine disabled + stub llmProxy → dùng generator llm, insert draft thật (mocked
  không set). Reuse DB helper.
- gen-fill: đã có test mock generator (Bước 2) — không đổi.

## 7. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| LLM trả JSON hỏng | responseFormat json_object + parse guard → reject+retry |
| Đáp án sai | solver-check reject; vet người ở draft là lưới cuối |
| readability không khớp tier | validateTestlet đã gate; prompt ép CEFR |
| TOEIC Part 1 (ảnh) | ngoài phạm vi — bỏ Part 1 |
| Cost trôi | `--max` cap ở gen-fill + 2 call/testlet |

## Câu hỏi mở
Xem spec §Câu hỏi mở (Part 1, solver model, use_of_english).
