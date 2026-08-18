---
feature: llm-direct-testlet-generation
status: draft
created: 2026-08-18
repos: [exe-api]
---

# Tasks — Direct llm-proxy testlet generation

Test-first. Verify từ `exe-api/services/api`. Không gọi LLM thật trong test (stub `deps.llmProxy`).

## T1 — Prompt templates (pure)

**File:** `src/modules/assessment/llm-generate-prompts.js` (mới) + test

- [ ] `buildTestletPrompt({ skill, itemTypes, cefr, tier, topic, count })` → mảng `messages`
  (system + user) ép JSON schema testlet (design §1), theo skill: reading passage / reading standalone /
  listening audio / standalone grammar.
- [ ] System prompt liệt kê rõ ràng buộc: mcq 4 lựa chọn + answerKey index; open-text bắt buộc
  acceptedVariants; explanation; CEFR đúng; trả JSON object thuần.
- [ ] Test (pure): prompt reading chứa 'passage'+CEFR+itemTypes; listening chứa 'transcript'; standalone
  không yêu cầu stimulus.

**Verify:** `npx jest llm-generate-prompts` → xanh.

## T2 — Generator + solver-check

**File:** `src/modules/assessment/llm-generate-testlet.service.js` (mới) + test

- [ ] `parseTestlet(raw)` — JSON.parse + chuẩn hoá (skill, tier, stimulus, questions[]); lỗi → throw có ý nghĩa.
- [ ] `solverCheck(testlet, { deps })` — gọi `llmProxy.chat` blind giải từng câu; so answerKey/acceptedVariants;
  ≥1 lệch tự tin → `{passed:false, errors}`.
- [ ] `generateTestletViaLLM({ skill, tier, itemTypes, topic, cefrHint, targetGoal, deps })` →
  buildPrompt → `deps.llmProxy.chat({responseFormat json})` → parseTestlet → solverCheck →
  `{ testlet, gates }` (shape = engine.generateTestlet).
- [ ] Model config: GEN_MODEL/SOLVER_MODEL (default gpt-4o-mini), override env.
- [ ] Test (stub `deps.llmProxy.chat`): trả JSON hợp lệ → testlet pass validateTestlet; đáp án khớp →
  gates.passed; solver trả khác → gates.passed=false; JSON hỏng → gates.passed=false.

**Verify:** `npx jest llm-generate-testlet` → xanh.

## T3 — Wire vào gen-testlet (engine off → llm thật)

**File:** `src/modules/assessment/gen-testlet.service.js` · test
`src/modules/assessment/__tests__/gen-testlet*.test.js`

- [ ] Import `generateTestletViaLLM`. Nhánh production:
  `engine.isEnabled() ? engine.generateTestlet(...) : generateTestletViaLLM({...deps})`.
- [ ] Truyền `deps` (llmProxy) xuống để test seam hoạt động; giữ `deps.llm` seam cũ (nếu còn dùng) không phá.
- [ ] Không đổi phần validate/quality/insert/TTS/embed.
- [ ] Test: engine disabled (mock `engine.isEnabled()=false`) + stub llmProxy → `generateAndInsertTestlet`
  chèn item `contentSource='ai-generated-reviewed'`, `status='draft'`, `mocked` không set; solver reject →
  throw, không chèn.

**Verify:** `npx jest gen-testlet` → xanh (0 regression).

## T4 — Smoke thật nhỏ (thủ công, có phí)

**File:** không code — dùng `scripts/ops/fill-exam-practice-bank.js` sẵn có.

- [ ] Chạy `node scripts/ops/fill-exam-practice-bank.js --max 3` với OPENAI_API_KEYS thật →
  xác nhận sinh item TOEIC reading THẬT (không mock), đọc log token → tính đơn giá thật.
- [ ] Soi 1-2 item vừa sinh (stem/options/answerKey/explanation) xem chất lượng.
- [ ] (Chỉ chạy khi user OK chi tiền; cap `--max` nhỏ.)

**Verify:** DB có item draft nội dung thật, đúng targetGoal/cefr/itemType.

## Definition of done

- [ ] exe-api `npx jest gen-testlet llm-generate` full xanh; 0 regression test-paper.
- [ ] Engine tắt → `fill-exam-practice-bank.js` sinh item THẬT (không mock).
- [ ] Đơn giá thật đo được từ mẻ `--max 3`.

## Câu hỏi mở
- TOEIC Part 1 (bỏ/khung 94) · solver model (mini/4.1) · use_of_english trong phase này (spec §Câu hỏi mở).
