---
feature: placement-item-bank
phase: C (remaining work — post Phần B wiring)
status: draft
owner: TechLead
created: 2026-08-31
---

# Phần C — Việc còn lại để placement chạy đúng end-to-end

> Nối tiếp `phase-b-tasks.md` (T-B1→B8 đã code + review). Phần C sinh ra từ **smoke thật** trên dev
> (2026-08-31): env chưa wire + 1 gap design↔impl lộ ra. Ghi kiểu SRS, concision.
> Nguồn sự thật: `design.md`, `placement-decisions-handoff.md`, `bank-blueprint-cefr-placement.md`.

## 0. Bối cảnh — phát hiện khi smoke (đã verify trên DB/code dev)
- **Cơ chế đúng, không đổi hướng:** placement = fixed-mode `mst-flow`, scope `targetGoal='general'` (`design.md:26-27`, `decisions:82`).
- **Serving fixed-mode dùng CHUNG mọi testType** (`assessment.service.js:109` fixed = default): diagnostic/mock_ielts/mock_toeic/placement đều đi `next-module`→`find_stimuli`. Blueprint engine (`assembleFromBlueprint`) **chưa tồn tại** (chỉ `item-type-blueprint.js` hằng số).
- **Gap thật (design ✗ impl):** `design.md:27` bắt buộc stimulus có `targetGoal`, nhưng `mapStimulus`/schema **không set**; và CAT `find_stimuli` (path MST) **không lọc targetGoal** (trong khi `find_candidates` path adaptive **có** lọc). → placement (và mọi fixed testType) serve **cả bank trộn pool**.

## 1. Trạng thái ENV sau smoke (đã sửa tạm — cần ratify)
| Việc đã làm phiên smoke | Trạng thái | Cần chốt |
|---|---|---|
| Retire 144 dummy general (`stimulusId:null`) | ✅ đúng (`decisions:54`) | — |
| Seed 52 item thật + 15 stimuli + 8 mp3 (→ remote `enova_dev`) | ✅ | mp3 ở `exe-api/storage`, ship dev/prod sau |
| Bật `ASSESSMENT_MST_ENABLED=true` (api `.env`) | ⚠️ tạm | **[Q1]** flag global — đúng ý? ảnh hưởng MỌI testType |
| Trỏ CAT `MONGO_URI` → remote `enova_dev` (trước trỏ local rỗng) | ✅ workaround LOCAL | Chỉ artifact máy này (api local trỏ remote, cat giữ internal). Dev/prod api+cat cùng DB sẵn → **không đổi gì**. |
| Thread `targetGoal` Node (adapter/mst-flow/attempt.service) + test | ✅ (prereq) | giữ; chưa đủ tác dụng tới khi làm T-C1..C3 |

## 2. Task workflow (ưu tiên → phụ thuộc)

### T-C1 — [CAT Python] `find_stimuli` scope theo targetGoal  ⚠️ RỦI RO CAO (đụng mọi fixed testType)
- **Repo:** `exe-api/services/cat`.
- `NextModuleRequest` (+ `next_module`) nhận `targetGoal`; forward `find_stimuli(..., target_goal=...)`.
- `find_stimuli` (store/mongo.py): `if target_goal: query["targetGoal"] = {"$in":[target_goal,"general"]}` — **gương `find_candidates`**.
- **TDD:** test find_stimuli lọc đúng khi có/không target_goal; test next_module truyền field.
- **Phụ thuộc:** T-C2 (stimulus phải có field targetGoal trước, nếu không filter loại sạch stimulus cũ → vỡ mock_ielts/toeic).
- **Verify:** probe placement→stimulus `6a955…` (general); probe mock_ielts→stimulus ielts; không testType nào `poolExhausted` sai.

### T-C2 — [Node] Thêm `targetGoal` vào stimulus + BACKFILL toàn bộ  ⚠️ đụng bank thật (backup trước — A5)
- `assessment-stimulus.model.js`: thêm field `targetGoal` (enum khớp question, default undefined additive).
- `mapStimulus` (`placement-content.mapper.js`): set `targetGoal:'general'` (`design.md:27`).
- **Backfill 116 stimulus cũ:** derive targetGoal từ question thuộc stimulus (mỗi stimulus các question chia sẻ 1 targetGoal — **verify giả định này trước** khi backfill; nếu lệch, quyết theo đa số + log).
- Re-seed/backfill 15 stimulus placement mới = `general` (loader idempotent — hoặc update trực tiếp).
- **TDD:** test mapStimulus set targetGoal; test backfill script trên fixture nhiều testType.
- **Verify:** mọi stimulus active có targetGoal ≠ null; 15 placement = general; ielts stimuli = ielts.
- **Backup bắt buộc** (`decisions:A5`): mongodump `assessment_stimuli` trước backfill.

### T-C3 — [Verify] Regression serving đa testType sau T-C1+C2
- Probe/integration: placement→general pool (chỉ 52 item mình), mock_ielts→ielts pool, diagnostic→general, không cái nào rỗng.
- **Chốt:** T-C1+C2 là 1 cặp không tách (deploy cùng nhau, nếu không vỡ fixed testType khác).

### T-C4 — [Env/Infra] Ratify `ASSESSMENT_MST_ENABLED` (không code, quyết định)
- **[Q1]** flag global: giữ bật cho mọi testType (đã bật để gỡ 404). Prod deploy (T-B8) phải bật → kích hoạt fixed MST toàn hệ → cần team ký duyệt, không âm thầm.
- **Q2 (CAT MONGO_URI): ĐÃ ĐÓNG** — chỉ artifact máy local (api local trỏ remote). Dev/prod api+cat cùng DB sẵn, KHÔNG đổi cấu trúc env. `cat/.env` sửa tay chỉ để smoke máy này.
- Ghi vào `docs/system-architecture.md` khi chốt Q1.

### T-C5 — [Content] Bổ sung coverage bank (data, không code) ✅ DONE (2026-09-01)
- Thang **A2–C1** (chốt Q3) — KHÔNG author A1/C2 (ngoài scope MVP, kẹp biên).
- **Đã author + load** batch-05..13 (Claude/Opus soạn tay `placement-content-v1`, cùng pattern batch-01..04): **+36 testlet / +127 item** lên remote `enova_dev`. Bank: **15→51 stimuli · 52→179 item**. TTS 25 audio qua llm-proxy (0 fail). Plan: `plans/260901-0956-placement-testlet-expansion/`.
- **Coverage form:** **30/30 ô FORM (spec §2b) ≥5** trên DB thật. Kế hoạch/ý đồ: `plan.md` + `authoring-intent.md`.
- **[Decision 2026-09-01] 11 ô NGOÀI form giữ nguyên (mỏng 1–2 item):** `getPlacementCoverage` cờ mọi ô có item mà <5 → báo `ok:false` do 11 ô word_form/collocation/vocab/grammar ở CEFR mà **form không rút** (sót từ batch cũ 01–04 + 2 bonus vocab). Các ô này **không chặn ráp đề** (fixed MST serve testlet theo skill×tier, không theo competency-cell). Không lấp (tránh ~40 item ngoài form, YAGNI), không retire (item hợp lệ + DB dùng chung). `ok:false` của tool là kỳ vọng, không phải lỗi.
- **Fix khi load:** `word_form` là *subskill*, itemType phải `fill_blank` (không phải `word_form`) — batch-07.
- irt.b vẫn **provisional theo CEFR** (chưa calibrate) → calibrate sau pilot (Rasch n≥100), out-of-MVP.
- **Chưa commit** batch JSON (theo yêu cầu) — deliverable sẵn ở `services/api/scripts/data/placement-content/batch-05..13.json`.

### T-C6 — [FE] Restart dev + smoke cấu trúc
- Restart `next dev` (PID build cũ còn hiện W/S) — code T-B5 đã ở branch, chỉ cần compile lại.
- Smoke 3 mốc: intake không mic → 16R/15L không W/S → result 2 kỹ năng (`result.isPlacement`).
- **Phụ thuộc:** T-C1..C3 (nếu muốn smoke serve ĐÚNG pool placement). Không có T-C1..C3 vẫn smoke được **cấu trúc** nhưng serve item trộn.

### T-C8 — [FE/exe-web] Sửa default confidence `dont_know`→`null` (không phạt người bỏ qua widget)
- **Bug (smoke lộ):** `MstRunner.tsx:150-153` — câu đã trả lời nhưng không chọn tự tin → gửi `selfConfidence:'dont_know'` → BE `classifyResponse` xử admitted_guess → loại khỏi θ. Nếu người dùng không đụng widget tự tin → MỌI câu bị loại → "chưa đủ dữ liệu chấm điểm". Placement thiết kế chấm bằng đúng/sai (`decisions §2`), confidence không phải input bắt buộc.
- **Fix:** đổi default `"dont_know"` → `null` (BE xử `null → 'engaged' → COUNTS`). `selfConfidence: confidence[q.questionId] ?? null` (bỏ nhánh isAnswered→dont_know).
- **KHÔNG đụng** SkipReason 'dont_know' (skip có chủ đích, `MstRunner.tsx` handleSkip / test line 89) — khác hoàn toàn.
- Test: `MstRunner.test.tsx:183-193` "defaults confidence to 'dont_know'..." → sửa expect `null` (+ đổi tên test cho khớp).
- Repo `exe-web`, branch `feat/PRD-288-placement-wiring`. `SelfConfidence|null` đã hợp lệ ở type payload.
- **Verify:** `npx vitest run src/components/features/assessment/exam/MstRunner.test.tsx` pass; smoke lại (tốc độ thật >3s/câu, không cần bấm confidence) → ra CEFR.
- **Lưu ý còn lại:** rapid-guess (<3s) vẫn loại — đúng, giữ. Chỉ fix confidence default.

### T-C9 — [BE/exe-api] Exempt placement khỏi `assessment` quota
- **Lý do:** quota gate chi phí LLM (chấm W/S). Placement = R/L, KHÔNG LLM → không nên tiêu quota (`assessment.service.js:118-121` comment tự xác nhận).
- **Fix:** trong `createFromIntake`, khi `intake.goal === 'placement'` → `Assessment.create(...)` **KHÔNG qua `withQuota`** (bỏ tiêu quota). Các goal khác giữ nguyên `withQuota`.
- Giữ `Organization.retake.maxRetakes` (cap retake riêng — không phải quota). Bỏ quota ≠ bỏ mọi giới hạn.
- Test: intake goal=placement 2 lần liên tiếp không bị quota block; goal=ielts vẫn tiêu quota.
- Repo exe-api, branch `feat/PRD-288-placement-item-bank`.

### T-C10 — [FE/exe-web] "Đánh giá trình độ" → hiện result nếu đã có, không redo
- **Bug:** nav → `ROUTES.ASSESSMENT_INTAKE` (`nav-items.ts:95`) bọc `<QuotaGate>` → luôn vào intake; hết quota → hiện "limit", KHÔNG bao giờ xem được result cũ.
- **Fix:** nav → trang index `/assessment` mới: gọi `getHistory` (`assessment.service.ts:362`, BE `/assessments/history`); nếu có attempt scored → route/hiển thị **result mới nhất** (`/exam/<attemptId>/result`); nếu chưa có → `/assessment/intake`.
- Đổi `nav-items.ts` "Đánh giá trình độ" → route index mới (thêm `ROUTES.ASSESSMENT`).
- Repo exe-web, branch `feat/PRD-288-placement-wiring`.

### T-C11 — [FE/exe-web] Bỏ widget confidence khỏi bài thi
- **Lý do:** placement chấm bằng đúng/sai, confidence không dùng (T-C8 đã default null). Widget gây rối (đã khiến smoke "chưa đủ dữ liệu").
- **Fix:** bỏ `ConfidencePicker`/`CONFIDENCE_OPTIONS` render trong `TestletCard.tsx` (:39,:66,:341-342) + prop `onConfidence`/`confidence`; bỏ state `confidence` + `handleConfidence` trong `MstRunner.tsx`. Payload giữ `selfConfidence: null` (→ BE 'engaged' → counts). Dọn orphan do thay đổi tạo ra.
- **Giữ** rapid-guess (time-based) — không đụng.
- Test: cập nhật/bỏ test liên quan confidence UI; submit vẫn gửi selfConfidence null.
- Repo exe-web, branch `feat/PRD-288-placement-wiring`.

### T-C13 — [FE] Timer + auto-submit cho bài placement (GAP)
- **Hiện trạng:** `ExamTimer` (đếm ngược + auto-submit) CHỈ dùng ở `ExamContainer` (luồng adaptive `/do`). `MstExamFlow` (`/mst-run` = placement) tính `deadline` từ `remainingSec` (:112) nhưng **KHÔNG render timer, KHÔNG auto-submit**.
- **Fix:** wire `ExamTimer` vào `MstExamFlow` (tái dùng component sẵn có) + auto-submit khi hết giờ. BE đã gửi `remainingSec` (section timers CORE_PLACEMENT_PRESET R720/L600). Xác định timer theo section (skill) hay toàn bài — chốt với product.
- Repo exe-web.

### T-C14 — [FE] Hiển thị tiến độ tổng "câu X / tổng" (enhancement)
- **Hiện trạng:** có per-testlet (`{resolvedCount}/{totalQuestions} đã trả lời`, MstRunner:176) + per-skill (`Part {i}/{total}`, MstExamFlow). **Thiếu** tổng toàn bài (vd "câu 12/31").
- **Fix:** cộng dồn số câu đã trả lời across testlet/skill → hiện "X/31" ở progress header. Tổng = Σ fixedModuleCounts×items (R16+L15).
- Repo exe-web. Nhỏ.

### T-C15 — [Data] "Khôi phục item cũ" — ĐÁNH GIÁ: phần lớn KHÔNG khả thi
- 144 dummy đã retire: `stimulusId=null` + irt **provisional (synthetic)** → **không serve được** fixed R/L vì thiếu passage/audio, KHÔNG phải vì IRT. IRT không cứu được item thiếu stimulus. → khôi phục = phải author passage cho chúng ⇒ tốn hơn gen mới. **Không đáng.**
- Hướng đúng để "không bỏ phí": **T-C5** — mở rộng pool bằng gen item mới (A2–C1, ≥5/ô) HOẶC reclassify item thật có sẵn (reading-bank-v1 ielts/draft) sang general NẾU khớp blueprint placement (content decision, cần review từng cái).
- irt.b thật chỉ có sau **calibrate** (Rasch n≥100) — hiện mọi item placement là provisional.

### T-C7 — [Deploy] T-B8 (giữ nguyên handoff)
- `plans/reports/deploy-handoff-260831-0823-placement-part-b-t-b8-report.md`: backup → loader TTS thật → dump `assessment_questions`+`assessment_stimuli` (giữ `_id`) + zip 8 mp3 → gửi lead nạp dev+prod → smoke dev.
- **Bổ sung:** dump phải gồm field `targetGoal` mới trên stimulus (sau T-C2).

## 3. Thứ tự thực thi đề xuất
```
T-C4 (ratify Q1/Q2)  ──►  T-C2 (stimulus.targetGoal + backfill, backup trước)
                              │
                              ▼
                          T-C1 (CAT find_stimuli filter)  ──►  T-C3 (regression đa testType)
                                                                   │
                          T-C6 (restart FE + smoke đúng pool) ◄────┘
                                                                   │
                          T-C5 (content A1/C2 + pool depth) ───────┤ (song song, data)
                                                                   ▼
                                                              T-C7 (deploy)
```

## 4. Rủi ro chính
- **T-C1+C2 đụng mọi fixed testType** — sai backfill → vỡ mock_ielts/toeic/diagnostic. Bắt buộc: backup + regression T-C3 + deploy cặp C1+C2 cùng lúc.
- **Bật MST global** đã đổi hành vi serving của các testType khác (trước 404). Rà completion/timeout của diagnostic/mock_* sau khi bật (dùng `mode-comparison.service.js`).
- **irt.b provisional** — spread/scoring chỉ "ước tính" tới khi calibrate.

## 4b. Quyết định sản phẩm (chốt 2026-08-31)
- **[D1] Guest data:** GIỮ "đăng ký TRƯỚC mới lưu data" (chuẩn ngành). KHÔNG làm guest→account migration (T-C12 huỷ). Guest = dùng thử; muốn lưu thì tạo tài khoản trước khi làm bài. FE có `linkWithCredential` path ở BE (`auth.service.js:263`) nhưng không dùng — chấp nhận.
- **[D2] KHÔNG cho xem lại item ở result placement.** Chỉ hiển thị result + competency-level (Gist 5/5...). Lý do: pool mỏng (52 item, phơi nhiễm ~57–63%/lượt) → mọi review mức item (kể cả chỉ hiện câu) đều lộ đề → thi lại ăn điểm giả → mất giá trị đo. Nhất quán với security posture (xáo option, ẩn transcript, maxPlays:1). Review-với-đáp-án (câu+đúng/sai+giải thích) chỉ ở **luyện tập** (Sổ lỗi sai / Reading-Listening practice / Luyện đề), KHÔNG ở placement. **Revisit khi pool đủ lớn** (T-C5) — phơi nhiễm/lượt thấp thì có thể mở review nhẹ.

## 5. Câu hỏi — trạng thái chốt
- **[Q1]** `ASSESSMENT_MST_ENABLED` giữ global → **giữ ON** (điều kiện cần của cả fixed-serving, mọi testType). Prod cần team ký duyệt khi T-B8.
- **[Q2]** ~~CAT MONGO_URI~~ → **ĐÓNG:** artifact local, dev/prod nhất quán sẵn, KHÔNG đổi cấu trúc env.
- **[Q3]** Thang CEFR → **A2–C1** (đúng `bank-blueprint:16` "mốc đề đo"; A1/C2 ngoài scope MVP, kẹp biên). Bỏ task gen A1/C2 khỏi T-C5.
- **[Q4]** Backfill → **VERIFIED an toàn:** 109 stimulus active đều đúng 1 targetGoal (ielts 428/toeic 210/general 52/null 2 câu). Derive 1-1 từ question; 2 câu null → gán 'general' hoặc bỏ.
