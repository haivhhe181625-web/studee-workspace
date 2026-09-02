---
feature: placement-item-bank
phase: specification
status: draft
owner: BA
created: 2026-08-30
depends_on: luyen-de-khung-de-config-evolution/placement-decisions-handoff.md
---

# Spec — Placement Item Bank (Phần A: Câu hỏi/Item)

## 1. Mục tiêu & why
Dựng **pool item THẬT** cho bài kiểm tra đầu vào CEFR (`targetGoal='general'`) đủ để ráp bài **fixed-mode** **2 chiều điểm**: **Reading & Use of English** + **Listening** (chuẩn Cambridge — UoE gộp vào Reading, không tách chiều riêng). Grammar/word-form/collocation **chỉ nằm trong Reading** (skill=reading, tính vào θ_R); `vocab_in_context` nằm **cả Reading lẫn Listening** (là hiểu-nghĩa-trong-ngữ-cảnh). Hiện pool general = seed CI giả (`stem='[SEED]…'`, `stimulusId=null`, `contentSource=null`) → **fixed R/L không chạy được** (cần testlet theo `stimulusId`). Không có bank thật thì toàn bộ bài đầu vào vô nghĩa dù engine đã sẵn.

Nguồn quyết định: `../luyen-de-khung-de-config-evolution/placement-decisions-handoff.md` §Phần A, `item-types-catalog.md`, `bank-blueprint-cefr-placement.md`.

## 2. Phạm vi

### Trong scope (chỉ repo `exe-api`)
- **Model:** thêm `vocab_in_context` vào `SUBSKILLS` enum (`assessment-question.model.js`) — [G1].
- **Validation:** item placement bắt buộc `irt.b` non-null + `stimulusId` (R/L) / rubric-free UoE — [G2].
- **Stimulus:** tạo passage (Reading) + audio (Listening) `general` theo tier/CEFR (`AssessmentStimulus`).
- **Item:** sinh item thật gắn stimulus, đủ competency, dạng câu MVP lean, `contentSource≠null`, `targetGoal='general'`, `status=active`, `irt.b` provisional theo CEFR.
  - **Reading & UoE** (skill=reading): gist·detail·inference·attitude·vocab_in_context **+ grammar_tense·word_form·collocation** (phần UoE nhúng, dạng `cloze`/`fill_blank`/`word_form`). Tất cả `skill=reading` → tính vào θ_R.
  - **Listening** (skill=listening): gist·detail·inference·attitude·vocab_in_context (qua audio). **KHÔNG** câu grammar rời.
- **Gen:** nâng chất lượng pipeline có sẵn (model lớn qua `GEN_MODEL`, cross-model solver, siết prompt) — KHÔNG viết lại pipeline.
- **Coverage tooling:** report riêng cho placement `general × skill × CEFR × competency × irt.b tier` — [G3].

### Out of scope (lý do)
- **Chiều UoE riêng (θ_UoE)** → bỏ. Grammar/vocab gộp vào θ_R (Reading & UoE). Không còn skill=use_of_english cho placement → **không cần task route UoE→θ_UoE ở Phần B**.
- **Wiring engine / áp `CORE_PLACEMENT_PRESET` / admin** → Phần B, tài liệu riêng.
- **Writing/Speaking item** → tạm bỏ theo hướng đã chốt.
- **Adaptive mode** → chỉ fixed.
- **Calibrate thật (Rasch n≥100) + Angoff** → sau pilot; giai đoạn này chỉ `irt.b` provisional.
- **Dạng `sentence_ordering`, `key_word_transform`** → phase 2 (cần code type + luật chấm mới); dạng có-sẵn đã phủ đủ competency.
- **exe-web / exe-admin / mobile** → không đụng.

## 2b. Phân bổ competency — "đủ bao quát" (form ráp)
UoE chấm đủ nhờ phủ đúng competency, không nhờ chiều riêng. Form mẫu:

**Reading & Use of English — 16 câu** (rải A2→C1)
| Competency | # | Ghi chú |
|---|---|---|
| gist | 2 | B1, C1 |
| detail | 3 | A2, B1, B2 |
| inference | 3 | B1, B2, C1 |
| attitude | 2 | B2, C1 |
| vocab_in_context | 2 | B1, B2 |
| grammar_tense | 2 | A2, B1 — *UoE* |
| word_form | 1 | B2 — *UoE* |
| collocation | 1 | B1 — *UoE* |

→ **Đọc hiểu 10** (gist/detail/inference/attitude) + **vocab_in_context 2** + **UoE grammar 4** (grammar_tense/word_form/collocation) = 16. **UoE = 4/16 (25%)** — grammar chiếm ~25% "sức kéo" θ_R, reading vẫn chủ đạo (~75%), không thiên lệch. Overall = **(θ_R + θ_L)/2** (avg-renormalize 2 trục).

**Listening — 15 câu**
| Competency | # | Ghi chú |
|---|---|---|
| gist | 3 | A2, B1, B2 |
| detail | 5 | A2, B1, B2, C1 (+1) |
| inference | 3 | B1, B2, C1 |
| attitude | 2 | B2, C1 |
| vocab_in_context | 2 | B1, B2 — *vocab qua audio* |

Bank phải chứa **≥5 item / ô** (skill×CEFR×competency) để ráp được nhiều form từ phân bổ này.

## 3. Acceptance Criteria (checkable)
- [ ] `SUBSKILLS` enum chứa `vocab_in_context`; item gán được subskill này, lưu OK.
- [ ] Không tạo được item placement (`targetGoal='general'`, skill reading/listening) với `irt.b=null` hoặc `stimulusId=null` (validation chặn) — có test.
- [ ] Item grammar/word_form/collocation chỉ tạo được với `skill=reading` (không cho skill=listening) — có test.
- [ ] Pool `general` active đạt coverage target: mỗi ô `(skill×CEFR×competency)` ≥ **5** item. Reading cõng nhiều competency hơn (thêm grammar/word_form/collocation) → **Reading pool ≥ 70, Listening ≥ 50** — verify bằng coverage tooling.
- [ ] Ráp thử 1 form fixed **Reading & UoE 16 / Listening 15** từ pool `general` **thành công**, mỗi ô có item đúng dải `irt.b`, không thiếu ô nghiêm trọng.
- [ ] Coverage tooling báo đúng theo `general × skill × CEFR × competency × tier` (khác tool cũ chỉ skill×CEFR×itemType) — có test.
- [ ] Mọi item sinh ra có `contentSource≠null` và qua gate chất lượng (moderation + dedup + solver + expert-review mẫu).
- [ ] `mongodump` 3 collection chạy trước mọi script sửa bank (runbook, không phải test).

## 4. Quyết định đã chốt (phiên 2026-08-30)
- **2 chiều điểm:** Reading & Use of English + Listening (bỏ chiều UoE riêng).
- **Grammar/word-form/collocation → chỉ Reading** (skill=reading, tính vào θ_R). `vocab_in_context` → cả R và L.
- **Số câu:** Reading & UoE **16** (gồm ~5–6 grammar/vocab) · Listening **15** (gồm vocab_in_context qua audio; 5 audio × 3 item/audio).
- **Hoãn phase 2:** dạng `sentence_ordering`, `key_word_transform` (cần code type + grade mới).
- **Gen:** model lớn qua `GEN_MODEL` + cross-model solver; **gen 1 testlet mẫu để human chấm trước** khi chạy hàng loạt.
- **Pool depth:** ≥5/ô cho MVP.

## 5. Open Questions (còn lại)
- Chất lượng testlet mẫu có đạt để chốt nguồn gen không (chờ human chấm).
- (Phần B) route/serving khi grammar nhúng chung testlet với passage — không thuộc scope này.
