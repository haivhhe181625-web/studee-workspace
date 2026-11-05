---
feature: placement-item-bank
phase: tasks-phase-b
status: ready
owner: TechLead
created: 2026-08-31
repos: exe-api (services/api + services/cat) · exe-web
---

# Tasks — Phần B: Wiring bài kiểm tra đầu vào (fixed R/L)

> TDD-checkbox. Mỗi task ~1 PR, test được độc lập. **Backend + Frontend** (form-plan do client quyết — đã verify).
> Bản đồ đầy đủ + `file:line` ở `luyen-de-khung-de-config-evolution/placement-decisions-handoff.md` §B0–B4. Đọc trước.
> Phần A (item bank) đã xong: 52 item authored, mapper/loader/smoke pass. Chưa nạp DB (T7.b chạy **cùng đợt deploy B**).

## Chốt (khoá — đừng tự đảo)
- **Form: Reading & UoE 16 (4 testlet×4) · Listening 15 (5 testlet×3) = 31 câu**, ~30 phút.
- **Mô hình 2 chiều:** θ_R (cõng UoE) + θ_L. `overall.js` **KHÔNG sửa** (renormalize (R+L)/2, verified `overall.js:110`).
- **Q-B2 = (a):** áp `CORE_PLACEMENT_PRESET` **luôn khi `goal='placement'`** (mọi luồng, org-link lẫn vào thẳng). `betaAdaptive` vẫn cho opt-in adaptive.
- **`use_of_english` KHÔNG gỡ tận gốc** — chỉ không đưa vào form-plan placement; skill vắng tự rớt.
- **Admin coverage viewer: HOÃN** (hàm `getPlacementCoverage` đã có, chưa ráp UI).

## Kết quả 2 spike (verify 2026-08-31 — đừng verify lại, đã chốt)
- **Fixed-mode selection** (`services/cat/src/routes/cat.py:279-281` + `mst.py:26-54`): fixed **bỏ routing θ**, serve mọi stimulus active, rank theo coverage item-type → tier=mid, **dừng thuần bằng `fixedStopDecision(count>=fixedCount)`** (`mst-attempt.js:61`). ⇒ per-skill count là đủ để điều khiển độ dài; type-blueprint mismatch KHÔNG hang. ⚠️ Engine **không ép trải đều CEFR** → verify spread ở T-B4; ép tier chỉ khi spread xấu.
- **Result page** (`exe-web` `assessment.service.ts:74-84` `withCoreSkills`): pad `skillScores` lên `CORE_DISPLAY_SKILLS=[listening,reading,writing,speaking]` → R+L-only **không crash** nhưng hiện 2 dòng "Chưa đánh giá" W/S (`SkillBars`), radar méo (`SkillRadar`), CTA "hoàn thành Viết/Nói" sai (`MissingSkillCta`), copy "đủ 4 kỹ năng" (`CoreResultView.tsx:62`). ⇒ T-B7.

---

## T-B1 — [BE] Áp CORE_PLACEMENT_PRESET + set config.skills cho placement
- [ ] Trong `createFromIntake` (`assessment.service.js`), khi `intake.goal==='placement'`: spread `CORE_PLACEMENT_PRESET` vào `config` (`includeWritingSpeaking:false`, sectionTimers) **và** set `config.skills=['reading','listening']` (client sẽ đọc — xem T-B5). Giữ `assemblyMode` logic cũ.
- [ ] `goalToTargetGoal('placement')→'general'` đã đúng (verified `:44`) — không đổi.
- [ ] Test: intake goal=placement → `config.includeWritingSpeaking===false`, `config.skills` = [reading,listening], sectionTimers set; goal khác → không áp preset.
- verify: `npx jest src/modules/assessment/__tests__/assessment.service.placement.test.js`

## T-B2 — [BE] Nới system-check cho placement (không bắt mic/recording)
- [ ] `failedSystemChecks` (`assessment.service.js:49`) hiện luôn đòi mic+recordingConsent → placement (R/L, không thu âm) sẽ 422. Cho hàm biết goal: skip `mic`+`recordingConsent` khi `goal==='placement'` (vẫn giữ net check).
- [ ] Test: placement thiếu mic/consent → **pass** (không 422); ielts thiếu mic → vẫn fail.
- verify: `npx jest src/modules/assessment/__tests__/assessment.service.placement.test.js`

## T-B3 — [BE] Per-skill fixedModuleCount
- [ ] `fixedModuleCount` hiện **1 số chung** (`assessment.model.js:42`, đọc `attempt.service.js:1060`). Thêm **additive** `config.fixedModuleCounts` (map `{reading, listening}`) — KHÔNG bỏ field cũ (adaptive/attempt cũ fallback). `CORE_PLACEMENT_PRESET` set `fixedModuleCounts:{reading:4, listening:5}`.
- [ ] `nextModuleForAttempt`/`advanceMstModule`: `fixedCount = config.fixedModuleCounts?.[skill] ?? config.fixedModuleCount ?? 1`.
- [ ] Test: attempt placement fixed → skill reading dừng sau 4 testlet, listening sau 5; attempt cũ (chỉ `fixedModuleCount`) vẫn chạy như trước.
- verify: `npx jest src/modules/assessment/__tests__/mst-flow.fixed-per-skill.test.js`

## T-B4 — [BE] Integration: ráp 1 attempt placement thật từ pool → đúng 16R/15L
- [ ] Nạp 52 item (loader, TTS stub) → tạo attempt placement fixed → chạy `nextModuleForAttempt` lặp cho reading rồi listening tới `done` (stub `catAdapter.nextModule` phục vụ từ pool DB, hoặc dùng cat thật nếu chạy được).
- [ ] Assert: reading tổng **16** item (4 testlet), listening **15** (5 testlet); mỗi skill span **≥3 CEFR**; không skill nào `pool_exhausted` sớm. Log phân bố tier (chỉ cảnh báo nếu cụm 1 tier).
- [ ] (nối tiếp T8 smoke Phần A — smoke test chỉ ráp tay; task này chạy qua flow engine thật.)
- verify: `npx jest src/modules/assessment/__tests__/placement-attempt.integration.test.js`

## T-B5 — [FE] Client đọc config.skills thay vì hardcode (dọn nợ)
- [ ] `MstExamFlow.tsx`: thay `TESTLET_SKILLS`/`EXAM_STEPS` hardcode (`:39,:42`) bằng đọc `config.skills` server trả (hiện bỏ qua, chỉ đếm ở `ExamConfirm.tsx:56`). W/S stage chỉ hiện khi skills chứa writing/speaking.
- [ ] Giữ vòng trong-skill nguyên (server `done` — `MstRunner.tsx:49-66`). W/S vẫn auto-skip khi ws-tasks rỗng (đã có `:96-99`).
- [ ] Test: config.skills=[reading,listening] → chỉ chạy R+L, progress 2 stage, không gọi/hiện W/S; config đủ 5 skill → như cũ.
- verify: `pnpm test MstExamFlow` (hoặc runner test tương ứng exe-web)

## T-B6 — [FE] Bỏ bước mic/recording-consent cho placement
- [ ] `IntakeWizard.tsx:50-72` / `SystemCheck.tsx`: bước SystemCheck mic+consent điều kiện theo goal — placement chỉ giữ net check (hoặc bỏ hẳn bước nếu không cần net).
- [ ] Test: intake goal=placement → không render mic/consent gate; goal ielts → vẫn render.
- verify: `pnpm test IntakeWizard` / `SystemCheck`

## T-B7 — [FE] Result page placement-aware (chỉ R+L, không dụ W/S)
- [ ] `assessment.service.ts` `withCoreSkills` (`:74-84`): khi result là placement (mode/type), **không pad** lên CORE_DISPLAY_SKILLS — render đúng skill có (`[reading,listening]`).
- [ ] Ẩn `MissingSkillCta` + copy "đủ 4 kỹ năng" (`CoreResultView.tsx:62`) cho placement. `SkillRadar`/`SkillBars` chỉ vẽ skill có điểm.
- [ ] Test: result 2 skill (R,L) placement → không có dòng "Chưa đánh giá", không CTA Viết/Nói, radar 2 trục.
- verify: `pnpm test CoreResultView` / `SkillBars`

## T-B8 — [Deploy] Nạp DB thật + đóng gói (= Phần A T7.b, chạy cùng đợt)
- [ ] Chạy loader 4 batch ở **dev/WSL** (TTS thật) → item active → `getPlacementCoverage` report.
- [ ] Dump `assessment_questions`+`assessment_stimuli` (giữ `_id`) + zip `audio/assessment/*.mp3` → gửi lead add **dev+prod**.
- [ ] Smoke thủ công 1 lượt: tạo attempt placement trên dev → làm hết → xem result.
- verify: attempt placement trên dev ra CEFR (R+L), result page sạch.

---

## Thứ tự chạy
BE trước (T-B1→B4) để `config.skills`/preset/ counts sẵn sàng → FE (T-B5→B7) bám config đó → T-B8 deploy cuối (gộp nạp DB).

## Rủi ro / theo dõi
- **CEFR spread ở fixed mode** (T-B4): nếu engine cụm tier mid → cân nhắc ép phân bố tier trong `mst.py` (Python, thêm code) HOẶC chọn testlet theo CEFR ở gateway. Chỉ làm nếu spread xấu.
- **Per-skill count schema** (T-B3): làm additive, giữ field cũ — đừng migrate.
- **FE đọc config.skills**: đảm bảo server luôn trả `['reading','listening']` cho placement (T-B1).

## Câu hỏi mở
- Không còn blocker. (Q-B2 đã chốt (a); 2 spike đã verify.)
