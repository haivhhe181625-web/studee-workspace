---
feature: placement-item-bank
phase: design
status: draft
owner: TechLead
created: 2026-08-30
---

# Design — Placement Item Bank (Phần A)

> Repo: `exe-api` (`services/api`). 5-file module pattern; item bank sống ở `src/modules/assessment/`.
> Nguyên tắc: **giữ chung collection** `assessment_questions` (phân tách bằng `targetGoal`), chỉ thêm view/coverage riêng cho `general`. KHÔNG tách store.

## 1. Data model — thay đổi tối thiểu

### 1.1 SUBSKILLS enum [G1]
`assessment-question.model.js` — thêm `'vocab_in_context'` vào `SUBSKILLS`. Đã có: gist, detail, inference, attitude, grammar_tense, collocation, word_form, pronunciation, fluency. Không đổi field khác.

### 1.2 Validation item placement [G2]
Item `targetGoal='general'` + skill ∈ {reading, listening}: `irt.b` **required non-null** + `stimulusId` **required** (mọi item R/L đều thuộc 1 testlet, kể cả câu grammar nhúng trong Reading). Thêm: item subskill ∈ {grammar_tense, word_form, collocation} **chỉ cho `skill=reading`** (không cho listening). Đặt ở schema `pre('validate')` hoặc guard trong service seed/gen — chỉ áp khi `targetGoal='general'` (không phá luyện đề).

### 1.3 irt.b provisional
Gán theo CEFR (cold-start): A1 −1.8 · A2 −1.2 · B1 −0.4 · B2 0.4 · C1 1.2. Rải trong ô ±0.24 để CAT/θ có phân biệt. `a=1.9, c=0.25` (MCQ). `se_b=null` (chưa calibrate) — đúng, output gắn nhãn "ước tính".

## 2. Stimulus (testlet) — điều kiện fixed R/L
`mst-flow` load câu theo `stimulusId` → **bắt buộc** có `AssessmentStimulus` cho R/L. Tạo:
- Reading: `kind='passage'`, `body`, `difficultyTier`, `cefrBandHint`, `targetGoal='general'`.
- Listening: `kind='audio'`, `maxPlays:1`, `transcript` (ẩn tới nộp), audio asset.
- Mỗi tier (easy/mid/hard) × skill ≥ vài stimulus để 4 câu/bài không phải lúc nào cũng cùng 1 bài (chống lộ).

## 3. Use of English — GỘP vào Reading (chuẩn Cambridge), KHÔNG chiều riêng
- **Chốt:** không có chiều/điểm UoE riêng. Câu grammar/word_form/collocation mang **`skill='reading'`** (subskill = grammar_tense/word_form/collocation), đính vào stimulus trong khối Reading → **tính thẳng vào θ_R** ("Reading & Use of English"). Listening giữ thuần comprehension.
- **vocab_in_context** = competency của **cả reading lẫn listening** (hiểu nghĩa trong ngữ cảnh) → skill theo bài nó nằm.
- **Đơn giản hoá lớn:** vì grammar là `skill=reading`, nó feed θ_reading tự nhiên — **KHÔNG cần** route sang θ_UoE. Bỏ luôn task wiring Phần B "UoE→θ_UoE" đã lo trước đó. `attempt.service.js:789` (lọc `q.skill !== skill`) giờ không còn liên quan (không còn item khác-skill trong testlet R).
- **Vì sao grammar chỉ Reading:** grammar là task chữ viết → nhét vào Listening thành "đọc trá hình" hoặc làm bẩn θ_L. Chuẩn Cambridge/Aptis cũng ghép UoE với Reading, Listening riêng.
- Grammar nhúng vẫn là **dạng đo-thuần** (cloze/fill_blank/word_form) — trả lời bằng kiến thức ngôn ngữ; đặt trong khối Reading nên gắn stimulus (passage hiện có hoặc short-text cloze riêng, đều `skill=reading`).

## 4. Gen pipeline — nâng chất, không viết lại
Tái dùng `gen-testlet.service.js` + `llm-generate-testlet.service.js` + gates (`gen-quality`, `dedup`, `expert-review`). Ba chỉnh:
1. **Model:** `GEN_MODEL` = model lớn (env, 0 code). Hiện default `gpt-4o-mini` → yếu.
2. **Cross-model solver:** `GEN_SOLVER_MODEL` khác hãng con gen → bất đồng đáp án có tín hiệu thật (giờ cùng model nhỏ → vô dụng).
3. **Prompt** (`llm-generate-prompts.js`): ép **competency từng câu** (gist/detail/inference/attitude/vocab_in_context), luật distractor, và **dạng đo-thuần** cho UoE. Reading/Listening prompt gắn subskill target.

Đầu ra: item `status='draft'`, `contentSource='ai-generated-reviewed'` → qua gate + expert-review mẫu → `activate-generated-items` để chuyển active.

## 5. Coverage tooling placement [G3]
Tool cũ (`bank-coverage.js`/`POOL_MATRIX`/`bank-stats`) group **skill×CEFR×itemType** → không đủ. Thêm **hàm/report mới** (không sửa đè tool luyện đề):
- Group `general` theo `skill × cefrLevel × subskill × tier(irt.b)`.
- Target: mỗi ô ≥ 5 active; Reading ≥60, Listening ≥60, UoE ≥30.
- Output: thiếu ô nào, tier nào rỗng (không route được) — để biết cần gen thêm gì.
- Đặt cạnh `bank-coverage.js` (pure, no DB) + wrapper aggregate trong `bank-stats.service.js` (thêm `subskill`+`irt.b` bucket vào `questionRows`, filter `targetGoal='general'`).

## 6. Backup (runbook, trước khi chạy script sửa bank)
`mongodump` ở WSL/máy có DB: `assessment_questions` · `assessment_stimuli` · `rubrics`. Seed hiện chỉ xóa prefix `[SEED]` (an toàn item thật) nhưng migration dựng-lại-bank đụng item thật → dump trước.

## 7. Quyết định & fork
- **CHỐT:** giữ chung collection · **2 chiều: Reading & UoE + Listening** (grammar gộp vào θ_R, không chiều UoE riêng, không task Phần B route UoE) · grammar chỉ Reading, vocab_in_context cả hai · số câu R16/L14 · gen model lớn + cross-model solver (mẫu trước) · pool ≥5/ô · coverage tool mới không đè tool cũ · irt.b provisional theo CEFR · hoãn 2 dạng phase-2.
- **Còn mở:** chất lượng testlet mẫu (human chấm) trước khi chốt nguồn gen.

## 8. Rủi ro
- **Gen chất lượng vẫn phụ thuộc model** — mitigate bằng cross-model solver + expert-review mẫu; nếu vẫn kém → soạn tay stimulus mỏ neo.
- **irt.b provisional lệch thực tế** — chấp nhận, gắn nhãn "ước tính", calibrate sau pilot (out of scope).
- **Coverage tooling đọc DB thật** — chỉ verify được ở WSL/CI có Mongo, không từ máy Windows hiện tại.
