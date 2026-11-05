# M2 IPA refactor — Tasks (TDD)

> Áp cho worktree: BE `.worktrees/ipa-m2-exe-api`, FE `.worktrees/ipa-m2-exe-web`.
> Mỗi task: viết test đỏ → code → verify xanh. Thứ tự có phụ thuộc; BE trước FE.
> Verify BE: `cd .worktrees/ipa-m2-exe-api && npm test -- <pattern>`.
> Verify FE: `cd .worktrees/ipa-m2-exe-web && npm test -- <pattern>`.

---

## Phase 0 — Chốt thiết kế (không code, gỡ 1 fork còn mở)

- [x] **T0.1** ✅ CHỐT: **mở rộng `IpaProgress` + tái dùng `IpaSrsQueue`**, không tạo `IpaMastery`. Ánh xạ: `state`→`IpaProgress.status` (learning≡completed), `needsReview`→field mới ở `IpaProgress`, `srsDueAt`→`IpaSrsQueue.nextReviewDate`. Chi tiết + máy trạng thái ở `design.md`.
- [ ] **T0.2** Chốt danh sách giá trị `IpaAttempt.kind`: `word|sentence|dialogue|perception|srs_review` (gộp 5 write cũ). → ghi `design.md`.

---

## Phase 1 — Data model + config

- [x] **T1.1** ✅ `ipa-lesson.model.js`: thêm `earTrainingTypes` (enum `['minimalPair']`, default `[]`) + export `EAR_TRAINING_TYPES`. `modelAudio.slow/superSlow` + `l1CompareAudio` giữ field, ngừng dùng (cắt ở Phase 6/FE, không xóa — surgical).
- [x] **T1.2** ✅ `ipa-attempt.model.js`: thêm `kind` (enum `word|sentence|dialogue|perception|srs_review`, default null, null-safe) + export `KINDS`. `itemType` giữ nguyên (migration sau).
- [x] **T1.3** ✅ `ipa-progress.model.js`: thêm `needsReview:bool=false`. `ipa-srs-queue.model.js`: thêm `reviewStage:Number=0`. `state`→dùng `status` sẵn có, `srsDueAt`→`nextReviewDate` sẵn có (theo design.md).
- [x] **T1.4** ✅ `app-setting.model.js`: thêm `ipaMasteredThreshold=80`, `ipaSentenceFloor=70`, `ipaPerceptionPassRatio=0.75`. `app-setting.service.js`: thêm `getIpaScoringConfig()` (fallback default vì `.lean()` bỏ default) + `invalidate()` xoá cache mới.
- **Verify Phase 1:** `npx jest --runInBand m2-schema.model ipa-scoring-config.service` → **11/11 pass**; `app-setting` 3/3 pass (không hồi quy).

---

## Phase 2 — Gộp endpoint (FR-006)

- [x] **T2.1** ✅ `POST /lessons/:id/attempts` dispatch theo `kind`: word/sentence/dialogue→`submitItem`, perception→`recordPractice`, srs_review→501 (Phase 5). `aiCallLimiter` có điều kiện (chỉ kind gọi engine). Schema `submitAttempt` (Joi switch) + code `IPA_ATTEMPT_KIND_NOT_IMPLEMENTED`.
- [x] **T2.2** ✅ Route cũ (`/submit`,`/practice`,`/mini-test`,`/warmup`) GIỮ NGUYÊN cho cửa sổ rollback — không đổi handler.
- **Verify Phase 2:** `submit-attempt.controller` 10/10; scoped `src/modules/ipa` + `src/__tests__/ipa` → 31/31 pass (không hồi quy).

---

## Phase 3 — Mastery mới (FR-007)

- [x] **T3.1** ✅ Tách hàm thuần `evaluatePass(template, screenScores, cfg)` = tổng ≥ `masteredThreshold` VÀ câu(m5) ≥ `sentenceFloor` VÀ cảm âm(m3) ≥ `perceptionPassRatio`. `computeLessonResult` dùng nó + `getIpaScoringConfig`; passed→`status=completed` (learning), bỏ streak 3×90. Side-effect (SRS/badge/emit) gated theo `passed`. AC-005/006 test xanh.
- [x] **T3.2** ✅ Template D → cổng cảm âm auto-pass (không minimal pair).
- **Verify Phase 3:** `evaluate-pass` 5/5 (AC-005/006 + cổng + skip D + dưới ngưỡng); scoped IPA 36/36 (không hồi quy). *Deep mastery (set `status=mastered`) + badge/emit chuyển sang Phase 5 SRS graduation — tạm gated theo `passed`.*

---

## Phase 4 — Cảm âm có chấm (FR-005 + cổng)

- [x] **T4.1** ✅ `kind:"perception"` → `recordPractice` ghi điểm client-tính vào `screenScores.m3` (nối ở Phase 2). Cổng mastery = m3 ≥ `perceptionPassRatio` **và chỉ áp khi bài CÓ minimal pair** (`hasPerception` = countDocuments>0) — generalize D5, chống deadlock bài non-D thiếu pair. Test: 3/4 pass, 2/4 fail, no-pair (D & non-D) pass.
  - **Lưu ý:** chấm minimal-pair vẫn **client-side** (kiến trúc objective sẵn có) → cổng tin client. Chấp nhận cho app học self-paced; nếu cần chống gian lận, chuyển chấm server-side (ngoài scope M2).
- **Verify Phase 4:** `evaluate-pass` 7/7; scoped IPA 38/38 (không hồi quy).

---

## Phase 5 — SRS 2 nhịp (FR-008, D6)

- [x] **T5.1** ✅ `advanceReview(stage, correct)` state machine: 0→1 (7d)→2 (mastered); trượt giữ stage + ôn lại 1d + `needsReview`, không thu hồi (AC-008). `scheduleReview` khởi động đồng hồ khi pass (idempotent); `recordReviewResult` đẩy stage + set `IpaProgress.status=mastered`/`needsReview`. `recordReview` orchestrate srs_review attempt (score≥threshold=correct) + graduation badge/emit (dời từ pass). Wire `POST /attempts kind=srs_review`. Dọn SM-2 `intervalForScore` (orphan).
- **Verify Phase 5:** `advance-review` 4/4; controller/schema srs_review 15/15; scoped IPA 43/43 (không hồi quy). *AC-007 (ôn rút gọn không màn Học) là hành vi FE → Phase 7.*

---

## Phase 6 — Audio thu hẹp (FR-009/011, D2)

- [x] **T6.1** ✅ *Không cần đổi code* — `ipa-audio.service.js` **vốn chỉ** TTS word/sentence/dialogue/pair/intro/question, **không hề gen** model 3-tốc-độ/L1 (đó là upload tay). Scope FR-009 đã đúng sẵn.
- [x] **T6.2** ✅ Pure `evaluatePublishable({words,sentences,pairs})` = có từ + mọi audio word/sentence/pair `ready`. `checkLessonPublishable` query `audioStatus` → gọi pure. `getLessonPipeline.publishable` chuyển sang cùng cơ sở audio-ready (thêm aggregation). Bỏ gate theo câu mini-test (v1.3 đã bỏ). KHÔNG gate model 3-tốc-độ/L1.
- **Verify Phase 6:** `evaluate-publishable` 6/6; ipa module 36/36 (không hồi quy). Không test IPA cũ nào phụ thuộc gate câu-hỏi (an toàn đổi).

---

## Phase 7 — FE khớp hợp đồng mới

> **Bối cảnh:** FE M2 3-màn (Học/Luyện nói/Kết quả, `buildSteps`) + dệt cảm âm + types `earTrainingTypes/status/masteredStreak` **đã có sẵn** trên develop (dev khác code). Phase 7 = **hoà giải** phần lệch v1.3, không greenfield.

- [x] **T7.1** ✅ **BUG chặn (mình tự tạo Phase 1/4)**: `earTrainingTypes` KHÔNG được ghi ở đâu (default `[]`), nhưng cổng BE Phase 4 khoá theo **có cặp tối thiểu** → 29 bài có cặp bị khoá `m3≥75` mà FE (khoá theo `earTrainingTypes`) không hiện câu cảm âm nào → **không thể đạt bài**. Sửa: `buildWovenEarTraining` khoá theo `data.minimalPairs` (**cùng khoá với cổng BE**), dựng tới **4 câu** listen-choose. `ObjectiveChoiceStep` vốn đã POST `/practice {screen:m3, %}` khi trả hết → 3/4=75 khớp `perceptionPassRatio`. Cập nhật nhãn "không bắt buộc"→"cần ≥3/4 để đạt" + comment cũ sai ("KHÔNG tính Mastery"). `earTrainingTypes` **thành vestigial** (giữ như `masteredStreak`, dọn sau).
  - **Hoãn (chờ chốt):** submit per-item vẫn qua **legacy** `POST /submit` + `/practice` (Phase 2 giữ nguyên cho rollback). Chưa chuyển sang `POST /attempts {kind}` — xem "Quyết định mở" cuối file.
- [x] **T7.2** ✅ `IntroStep.tsx`: bỏ khối "Nghe chậm hơn" 3 tốc độ + orphan (`Gauge`, `hasSpeeds`) (D2, cũng là dead — `slowUrl/superSlowUrl` = null toàn DB). `LessonRunnerContainer` STEP_FLOW: bỏ chữ "3 tốc độ". `buildListenChoose` trả 1 câu/cặp → cắt `slice(0,4)` tại chỗ gọi (tới 4 câu).
- [x] **T7.3** ✅ `types/ipa.types.ts`: `earTrainingTypes` (L86), `status`, `masteredStreak`, mastery `state`≡`status` **đã có sẵn**. `kind` chưa cần (còn dùng legacy submit) — thêm khi chuyển `/attempts`.
- **Verify Phase 7:** `tsc --noEmit` **exit 0**; `eslint` 4 file đổi **exit 0**. Không có test component IPA (vitest) nào để chạy/hỏng. Commit `87553da` (+ hook fix `472ca18`). *Orphan `buildPatternMatch` (do đổi của mình) là hàm export cạnh `buildIpaRecognition` dead sẵn → giữ, nhắc dọn.*

---

## Phase 8 — Migration cutover (FR-012)

- [x] **T8.1** ✅ `src/scripts/migrate-ipa-mastery.js`: scope bài **published A–D** (35). Pure `planProgressMigration` (unit-test): cũ mastered (`status='mastered'` HOẶC `masteredStreak≥3`) → `status=mastered` + seed `IpaSrsQueue` graduated (`reviewStage=2, mastered, nextReviewDate=null`); dở (`in_progress`/`completed` cũ, **chưa có SRS clock**) → `not_started` + xoá score. Người học flow-mới (đã có SRS row) → **không đụng** (bảo vệ chạy sau cutover). Idempotent + DRY-RUN mặc định (`--apply`). Guard `require.main` để test import không auto-connect.
  - **Verify Phase 8:** `migrate-ipa-mastery` 7/7 (graduate qua status & qua streak, reset in_progress & completed, skip not_started, skip khi đã có SRS clock, idempotent rerun); ipa module 43/43 (không hồi quy). Dry-run trên bản sao là bước ops thủ công (`node src/scripts/migrate-ipa-mastery.js`).

---

## Phase 9 — Regression + acceptance

- [ ] **T9.1** Regression 50 audio mẫu: lệch điểm ≤ ±5 trước/sau (AC-014).
- [ ] **T9.2** E2E 3 màn + cảm âm 4 câu + SRS 2 nhịp; duyệt checklist AC-001…014.
  - verify: `npm test` (BE full) + `npm test` (FE full) xanh; không import step màn đã bỏ.

---

## Quyết định mở — FE chuyển `/attempts`?
FR-006 gộp 5 write → `POST /attempts {kind}`. Endpoint **đã có + test ở BE** (Phase 2). FE (dev khác code) đang chạy **legacy** `/submit` `/practice` `/warmup` `/mini-test` — Phase 2 **cố ý giữ** legacy cho rollback nên FE **không hỏng**.
- **Đề xuất: HOÃN** chuyển FE sang `/attempts` trong PR này. Lý do: (a) mọi thắng lợi chất-lượng-học (mastery mới, cổng cảm âm, SRS 2 nhịp) là **BE-driven, đã hiệu lực qua legacy**; (b) chuyển = churn plumbing thuần + rủi ro hồi quy, **0 lợi ích người dùng** ngay; (c) làm follow-up khi retire legacy. → cần user chốt: hoãn / làm luôn.

## Ghi chú
- **KHÔNG** đụng `modules/review/` (PRD-142) — disjoint, không hợp nhất SRS trong M2.
- **Dọn sau (cleanup PR):** `earTrainingTypes` (BE field + FE type) thành vestigial sau khi cổng cảm âm khoá theo cặp tối thiểu; `buildPatternMatch`/`buildIpaRecognition` (objective-adapters) dead. Gỡ cùng đợt retire legacy write endpoints + cột `masteredStreak/warmUpConfidence/easeFactor`.
- Comment code theo `.claude/rules/review-audit-self-decision.md` §5: giải thích *why*, không ghi mã phase/FR/backlog.
