<!-- Sổ quyết định + handoff cho bài kiểm tra đầu vào CEFR. Ghi kiểu SRS, ưu tiên concision.
     Tách 2 phần: A = Câu hỏi/Item (LÀM TRƯỚC) · B = Bài kiểm tra đầu vào/wiring (LÀM SAU).
     Chốt của user là sticky — audit không tự lật. Trạng thái mỗi mục ghi rõ: CHỐT / đề xuất / mở. -->
# Placement Entry Test — Sổ quyết định & Handoff

> Nối tiếp: `demo-cefr-placement-entry-test-spec.md` (khung đề, bản 4 kỹ năng — **đã đổi hướng, xem dưới**), `assembly-and-difficulty-allocation-logic.md`, `bank-blueprint-cefr-placement.md`, `item-types-catalog.md`.
> Artifact review: placement demo `c15af819` · item-types `77f642a1`.

## 0. Đổi hướng lớn (phiên này) — override chốt cũ
| Chốt cũ (demo spec) | MỚI | Trạng thái |
|---|---|---|
| Đủ 4 kỹ năng, "không bỏ W/S" | **Tạm bỏ W/S**. Chỉ **Reading + Listening + Use of English (grammar/vocab)**. W/S optional, làm sau. | **CHỐT** |
| Overall = **min** | **avg-renormalize** (mỗi chiều đều nhau, thiếu chiều thì chia lại) | **CHỐT** |
| Ưu tiên tái dùng code | **Làm tốt nhất**, sửa code thoải mái | **CHỐT** |

## 1. Mô hình đo — 2 chiều (cập nhật 2026-08-30, chuẩn Cambridge)
- **Reading & Use of English (θ_R)** + **Listening (θ_L)** — 2 điểm. **KHÔNG** còn chiều UoE riêng (θ_UoE bỏ).
- **Grammar/word_form/collocation → chỉ Reading** (`skill=reading`, dạng đo-thuần cloze/fill_blank/word_form, gắn stimulus khối Reading) → tính thẳng vào θ_R. Vì grammar là task chữ viết, nhét Listening = "đọc trá hình"/bẩn θ_L. — **CHỐT**.
- **`vocab_in_context` = competency của cả Reading LẪN Listening** (hiểu nghĩa trong ngữ cảnh, skill theo bài). — **CHỐT**.
- Listening giữ **thuần comprehension** (gist/detail/inference/attitude/vocab_in_context qua audio), **không** grammar rời.
- **Đơn giản hoá:** grammar là skill=reading nên feed θ_R tự nhiên → **bỏ task Phần B "route UoE→θ_UoE"**.
- Competency Reading: gist·detail·inference·attitude·vocab_in_context **+ grammar_tense·word_form·collocation**. Hoãn `reference/cohesion`, `organization/discourse`. — **CHỐT MVP**.

## 2. Chấm điểm
- R/L/UoE: đúng/sai item → θ (IRT) → `thetaToBand` → `bandToCefr`.
- Overall = **avg-renormalize** trên **{R, L}** (mỗi chiều 1/2 — grammar/UoE nằm trong θ_R, KHÔNG là chiều riêng). Code `overall.js:110` renormalize trên skill CÓ điểm: `avg = Σ(num×w)/Σ(w)`; vì không sinh item `skill=use_of_english`, nó tự rớt → còn `(R+L)/2`. **Không cần sửa overall.js** (verified).
- Độ khó `irt.b` **toàn cục** (`relative_to_cefr:false`), tier `TIER_B_BAND` (easy <−0.5 · mid −0.5..0.5 · hard >0.5). Phân bổ %/level + fallback 4 bước có log (xem `assembly-and-difficulty-allocation-logic.md`).
- Output gắn nhãn **"ước tính"** tới khi calibrate (Rasch n≥100 + Angoff θ-cut).

---

# PHẦN A — CÂU HỎI / ITEM  (LÀM TRƯỚC)

## A1. Bộ dạng câu MVP (lean, đều CÓ SẴN engine — 0 code mới)
| Kỹ năng | Dạng MVP |
|---|---|
| Reading & UoE | `mcq` · `true_false_ng` · 1× matching (`matching_headings`/`matching_features`) · vocab_in_context (mcq) · **UoE nhúng:** `cloze` · `fill_blank`(banked) · `word_form` |
| Listening | `mcq` · `sentence_completion` · 1× `labeling` · `matching_features` · vocab_in_context (qua audio) |

- Chi tiết + ví dụ: `item-types-catalog.md` / artifact `77f642a1`.
- **Hoãn phase 2** (cần thêm type + luật chấm): `sentence_ordering` (sắp xếp thành câu), `key_word_transform` (biến đổi câu giữ nghĩa). Dạng có-sẵn đã phủ hết competency → 2 dạng này chỉ để phong phú/chống đoán mò. — **đề xuất hoãn, chờ chốt**.
- Điều tiết theo level (assembly, không phải thêm dạng): A1/A2 dùng dạng dễ (mcq, gap-fill đơn, TF); matching/headings dồn B1+.

## A2. Số câu — CHỐT (2026-08-30)
| Chiều | Item | Stimulus | ~phút |
|---|---|---|---|
| Reading & Use of English | **16** (gồm ~5–6 grammar/word_form/collocation + vocab_in_context) | 4 passage (A2/B1/B2/C1) | ~18 |
| Listening | **14** (gồm vocab_in_context qua audio) | 3–4 audio | ~15 |
| Tổng | **30 obj** | | ~33 |
- Reading dày hơn vì cõng phần UoE. Không còn chiều UoE riêng.

## A3. Ngân hàng item (bank) — thực trạng đã verify
- **Bank dùng CHUNG** collection `assessment_questions`, phân tách bằng `targetGoal`. Placement = `targetGoal:'general'`; luyện đề = `ielts/toeic`. → **giữ chung store**, chỉ thêm coverage-view riêng cho general. — **CHỐT**.
- Pool general hiện có = **seed CI giả** (`scripts/seed-assessment-questions.js`): `stem='[SEED]…'`, `stimulusId=null`, `contentSource=null` → **KHÔNG dùng được** cho fixed R/L (fixed cần testlet theo `stimulusId`). Item thật general ≈ 0.
- reading-bank-v1 = `targetGoal:'ielts'` + `status:'draft'` → không thuộc general, không active.
- **Gaps chặn** (xem `bank-blueprint-cefr-placement.md`): [G1] thêm `vocab_in_context` vào `SUBSKILLS` enum · [G2] `irt.b` bắt buộc · [G3] coverage tooling theo skill×CEFR×competency×tier (hiện chỉ skill×CEFR×itemType) · [G4] stimulus theo tier.
- Coverage target: mỗi ô (skill×CEFR×competency) ≥5 item; Reading ≥60, Listening ≥60.

## A4. Sinh item (gen) — đã verify pipeline
- **Cũ = `gpt-4o-mini`** (`GEN_MODEL` + `SOLVER_MODEL` trong `llm-generate-testlet.service.js`) → yếu ("lỏ"). Prompt generic, gate (moderation/dedup) fail-open, solver cùng model nhỏ.
- **Đề xuất cải thiện (không viết lại pipeline):** (1) đổi `GEN_MODEL` sang model lớn qua env — 0 code; (2) cross-model solver (gen model A, solve model B khác hãng); (3) siết prompt: ép competency từng câu + luật distractor + dạng đo-thuần cho UoE; giữ gate + thêm `expert-review` cho mẫu.
- **MỞ:** ai gen (model nào) — user muốn xem 1 testlet mẫu do Claude gen để chấm trước khi quyết.

## A5. Backup trước khi đụng bank (BẮT BUỘC)
`mongodump` 3 collection ở WSL/máy có DB **trước** mọi script sửa bank: `assessment_questions` · `assessment_stimuli` · `rubrics`. (Seed hiện chỉ xóa item prefix `[SEED]` nên an toàn với item thật, nhưng migration dựng-lại-bank sẽ đụng item thật.)

---

# PHẦN B — BÀI KIỂM TRA ĐẦU VÀO / WIRING  (LÀM SAU)

> Cập nhật 2026-08-31: scout exe-web xong → form-plan (skill nào chạy) do **CLIENT** quyết, server chỉ quyết số testlet/skill. Phần B = **backend + frontend**, không phải BE-only.

## B0. Chốt kích thước form (2026-08-31)
- **Reading & UoE = 16** câu (4 testlet × 4) · **Listening = 15** câu (5 testlet × 3) · tổng **31**, ~30 phút.
- Lý do 31 (không phải 30): reading testlet 4 câu (bội 4), listening testlet 3 câu (bội 3) → không có tổ hợp tổng 30 tròn testlet; 31 là gần nhất, hai phân môn đủ dày cho θ.
- Pool MVP đủ: reading 28 item / listening 24 item (52 tổng) — thừa cho 1 form.

## B1. Đã verify — engine + client
Backend:
- **Fixed mode = default**: `assessment.service.js:101` `assemblyMode = betaAdaptive===true ? 'adaptive' : 'fixed'`. Nhánh fixed: `mst-attempt.js` (`fixedStopDecision`), `mst-flow.service.js`.
- `CORE_PLACEMENT_PRESET` (`constants/blueprint.js:107`): `includeWritingSpeaking:false`, timer R720/L600/UoE480s. `getBlueprint` đọc `includeWritingSpeaking` (strip W/S) — có test. **CHƯA nơi nào spread** vào config.
- `goalToTargetGoal('placement') → 'general'` (`assessment.service.js:44`) — **đã đúng**, placement khớp pool general.
- `fixedModuleCount` = **1 số chung mọi skill, đếm testlet** (`assessment.model.js:42`, `attempt.service.js:1060`) → không ra R16/L15 khác nhau. Cần per-skill.
- `use_of_english` xuất hiện **19 file** → KHÔNG gỡ tận gốc; chỉ không đưa vào form-plan placement, `overall.js` tự renormalize (verified).

Client (exe-web) — verified:
- `TESTLET_SKILLS = ["reading","listening"]` hardcode (`MstExamFlow.tsx:39`) — vòng testlet **đã đúng R+L**.
- Trong 1 skill: **server quyết dừng** bằng cờ `done` (`MstRunner.tsx:49-66`) → số testlet/skill điều khiển từ BE.
- W/S là block phụ gọi `ws-tasks` sau R+L; **server trả 0 task → client tự nhảy qua submit** (`MstExamFlow.tsx:96-99`). → BE `includeWritingSpeaking:false` là W/S tự biến mất.
- `use_of_english` **không có UI bài thi** (chỉ nhãn ở result) → UoE nhúng reading không đụng client test flow.
- Client **bỏ qua `config.skills` server trả về**, chỉ dùng đếm hiển thị (`ExamConfirm.tsx:56`).

## B2. Việc wiring (backend + frontend)
| # | Nơi | Đổi gì | Độ lớn |
|---|---|---|---|
| B1 | BE `assessment.service.js:101` | goal=placement → spread `CORE_PLACEMENT_PRESET` → ws-tasks rỗng → client tự bỏ W/S | Nhỏ |
| B2 | BE form-plan | Số testlet/skill placement: reading 4 / listening 5 — hiện `fixedModuleCount` 1 số chung → cần per-skill | Vừa |
| B3 | FE `MstExamFlow.tsx` | **Dọn nợ (CHỐT):** client đọc skill-set từ `config.skills` server thay vì hardcode `TESTLET_SKILLS`/`EXAM_STEPS` → ẩn W/S stage tự động cho placement | Vừa |
| B4 | FE `SystemCheck.tsx`/`IntakeWizard.tsx` | Bỏ/rút gọn bước mic + recording-consent cho placement (R/L không cần) | Nhỏ |
| B5 | BE `failedSystemChecks:51` | Không bắt mic/consent cho placement (tránh 422) | Nhỏ |

- `overall.js` **KHÔNG sửa** — 2 chiều (R&UoE + L), avg-renormalize, skill vắng tự rớt (verified line 110).
- **Bỏ** task cũ "route UoE→θ_UoE" (grammar = skill=reading, feed θ_R tự nhiên).

## B3. Admin — HOÃN (2026-08-31)
- Coverage viewer read-only: **hoãn**, không làm đợt này. Hàm `getPlacementCoverage`/`placementCoverage` đã có sẵn (bank-stats + bank-coverage) — khi cần chỉ ráp UI. KHÔNG làm config-editor DB (YAGNI — placement 1 khung, đổi hiếm, để code + review + deploy).

## B4. Rủi ro cần rà khi vào code
- `use_of_english` hardcode ở `result.model`/band-scoring → khi vắng skill, trang kết quả có vỡ layout không.
- Per-skill `fixedModuleCount`: đổi schema (số → map skill×count) hay thêm field mới → ảnh hưởng adaptive attempt cũ không.
- FE đọc `config.skills`: đảm bảo server trả đúng `["reading","listening"]` cho placement.

---

## Câu hỏi mở
- ~~[Q-B2] áp `CORE_PLACEMENT_PRESET` khi nào~~ → **CHỐT (a): luôn khi `goal='placement'`** (2026-08-31).

## Thực thi Phần B
- Task breakdown turnkey: `specs/placement-item-bank/phase-b-tasks.md` (T-B1…B8, TDD + verify). 2 spike (fixed-mode selection · result page R+L) đã verify, kết quả ghi trong đó.
