---
feature: placement-item-bank
phase: tasks
status: draft
owner: TechLead
created: 2026-08-30
repo: exe-api (services/api)
---

# Tasks — Placement Item Bank (Phần A)

> TDD-checkbox. Mỗi task = ~1 PR, độc lập test được. Repo: `exe-api/services/api`.
> **PRE-REQ (runbook, làm trước T5+):** `mongodump` `assessment_questions` · `assessment_stimuli` · `rubrics` ở WSL/máy có DB.
> Test: jest. Task thuần logic chạy `npx jest`; task đụng Mongo cần DB (mongodb-memory-server hoặc `docker exec exe_api`).

## T1 — [G1] Thêm `vocab_in_context` vào SUBSKILLS
- [x] Thêm `'vocab_in_context'` vào `SUBSKILLS` (`src/modules/assessment/assessment-question.model.js`).
- [x] Test: tạo item `subskill='vocab_in_context'` validate pass; giá trị lạ vẫn fail.
- verify: `npx jest src/modules/assessment/__tests__/assessment-question.model.test.js`

## T2 — [G2] Validation item placement (general R/L)
- [x] Guard (chỉ khi `targetGoal='general'`): skill∈{reading,listening} ⇒ `irt.b` non-null **và** `stimulusId` non-null. Subskill∈{grammar_tense,word_form,collocation} ⇒ **chỉ cho skill=reading** (grammar không vào listening).
- [x] Test: general reading `irt.b=null` → reject; `stimulusId=null` → reject; grammar item skill=listening → reject; hợp lệ → pass; ielts item bỏ qua guard.
- verify: `npx jest src/modules/assessment/__tests__/placement-item-validation.test.js`

## T3 — [G3a] Coverage pure function (general × skill × CEFR × competency × tier)
- [x] Hàm pure trong `bank-coverage.js`: nhận rows (đã filter general), group theo `skill×cefrLevel×subskill×tier(irt.b)`, so target (≥5/ô; Reading pool ≥70 vì cõng grammar/vocab, Listening ≥50; chỉ 2 skill reading+listening), trả ô thiếu + tier rỗng. KHÔNG sửa `coverageGaps`/`POOL_MATRIX` cũ.
- [x] Test: fixture rows → phát hiện đúng ô thiếu + tier rỗng.
- verify: `npx jest src/modules/assessment/__tests__/placement-coverage.test.js`

## T4 — [G3b] Aggregate + report cho placement coverage
- [x] `bank-stats.service.js`: `placementRows()` aggregate group `subskill` + bucket `irt.b` tier ($switch), filter `targetGoal='general'`; `getPlacementCoverage()` nối hàm T3 ra report.
- [x] Test (DB): seed vài general item → group đúng tier; loại non-general/non-R-L; báo ô thiếu.
- verify: `docker exec exe_api npx jest src/modules/assessment/__tests__/bank-stats.placement.test.js`

> **Hướng nội dung = (A) tác giả tự viết (Claude), KHÔNG auto-gen LLM.** Item do người/Claude soạn thành JSON → loader chèn DB + TTS. Chất lượng = mẫu đã chấm. Text không tốn LLM; chỉ TTS cho audio.

## T5 — Content format + mapper (pure)
- [x] Định nghĩa JSON contract `placement-content-v1`: `{ passages:[{sourceRef,cefr,tier,body,items:[{competency,cefr,irtB,type,stem,options,answerKey,explanation}]}], audios:[{sourceRef,cefr,tier,transcript,items:[...]}] }`.
- [x] Mapper pure (mirror `reading-bank.mapper.js`): JSON → `AssessmentStimulus` (skill reading/listening, general, active, kind passage/audio) + `AssessmentQuestion` (subskill=competency, irt.b, answerKey, targetGoal='general', contentSource='self-authored', status active, order). Grammar subskill chỉ ở reading.
- [x] Test (pure): map 1 passage + 1 audio → field đúng; qua `validatePlacementItem`.
- verify: `npx jest src/modules/assessment/__tests__/placement-content-mapper.test.js`

## T6 — Loader (chèn DB idempotent + TTS listening)
- [x] `placement-content-loader.service.js` (mirror `reading-bank-seed`): additive/idempotent theo `provenance.importBatch`+`sourceRef`; validate mỗi item (T2) trước khi tạo. Listening: `llmProxy.tts` → `saveAssessmentAudio` → set `media.audioUrl` (rollback nếu TTS fail). Reading không TTS. `deps.llmProxy`/`deps.saveAudio` injectable.
- [x] Test (DB, stub `llmProxy.tts`+`saveAudio`): load 2 lần không nhân đôi; audio có `media.audioUrl`; TTS fail → xóa stimulus + skip; reading vẫn load.
- verify: `docker exec exe_api npx jest src/modules/assessment/__tests__/placement-content-loader.test.js`

## T7 — Tác giả viết content + chạy loader (dev) + activate + coverage
- [ ] **MVP pool trước** (~50 item, **≥2/ô** cho các ô 1 form cần → đủ ráp 1–2 form, chạy T8 + test đầu-cuối). Batch 01 done (14). Mở rộng lên **≥5/ô** (R≥70, L≥50) sau.
- [ ] **Claude soạn** passages + audios theo lô, đủ phân bổ competency; grammar/word_form/collocation chỉ ở Reading.
- [ ] Chạy loader ở **dev/WSL** (TTS thật) → item active. **KHÔNG chạy prod.**
- [ ] Verify (DB): report T4 đạt target. Đóng gói: dump `assessment_questions`+`assessment_stimuli` (giữ `_id`) + zip `audio/assessment/*.mp3` → gửi lead add vào dev+prod.
- verify: `docker exec exe_api node scripts/ops/placement-bank-status.js` → mọi ô đạt.

## T8 — Smoke: ráp thử 1 form fixed từ pool general
- [x] Ráp form **Reading & UoE 16 / Listening 14** từ pool general active (stratified theo CEFR); mỗi item có irt.b + stimulusId + subskill; form span ≥3 level mỗi skill.
- [x] Test (memory-server, load 4 batch thật qua loader, TTS stub): pool load đủ (52 item/15 stimulus); ráp đủ câu; grammar chỉ ở reading; answerKey chấm đúng (mcq + cloze/fill_blank); overall renormalize = (R+L)/2 khi chỉ R&L có điểm.
- verify: `npx jest src/modules/assessment/__tests__/placement-assemble-smoke.test.js` (5/5 pass — chạy được không cần Mongo ngoài)

---

## Trạng thái chốt
- ✅ T1–T4 done (enum · validate · coverage · aggregate). 2 chiều R&UoE + L · grammar chỉ Reading · R16/L14 · UoE 4/16 · pool ≥5/ô.
- ✅ Nội dung = **(A) Claude tự viết** (không auto-gen). TTS OpenAI giữ nguyên. Không chặn-prod (gen chỉ chạy dev, lead add tay cả 2 server).
- ⏳ T5/T6 code ở đây (mapper + loader, test memory-server/stub) · T7 chạy dev/WSL (TTS thật).
