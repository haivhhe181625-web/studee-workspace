# FE migrate `/attempts` — Tasks (TDD)

> Repo: **exe-web** (chính) · verify BE ở **exe-api**.
> Branch: `feat/ipa-m2-pronunciation-refactor`.
> Mỗi task: viết test đỏ → code → verify xanh. Thứ tự có phụ thuộc.
> Verify FE: `cd exe-web && npm test -- <pattern>` · `npx tsc --noEmit` · `npx eslint <file>`.
> Verify BE: `cd exe-api/services/api && npx jest --runInBand <pattern>`.
> Spec: `spec.md` (cùng thư mục). Contract `/attempts`: `spec.md` §2.

---

## Phase 0 — Guard (không code, chặn migrate sai)

- [ ] **T0.1** Grep mọi caller `submitIpaItem` — xác nhận chỉ đẩy `itemType` ∈
  `{word, sentence, dialogue}` (không `minimalPair`/`miniTest`, vốn KHÔNG phải `kind` hợp lệ).
  - verify: `grep -rn "submitIpaItem\|useSubmitIpaItem\|ITEM_TYPE" exe-web/src` → mọi itemType nằm trong 3 giá trị hợp lệ.
  - Nếu có caller đẩy giá trị khác → dừng, báo (ngoài giả định spec §2).
- [ ] **T0.2** Grep caller `recordIpaWarmup` + `submitIpaMiniTest` → lập danh sách để quyết
  xóa ở Phase 2 (dead nếu 0 caller component).
  - verify: `grep -rn "recordIpaWarmup\|recordIpaPractice\|submitIpaMiniTest\|MiniTestStep\|WarmUp" exe-web/src`.

---

## Phase 1 — Migrate write path sang `/attempts`

- [ ] **T1.1** Test đỏ: `src/services/ipa.service.test.ts` (mock `apiClient.post`) —
  `submitIpaItem(id, {itemType:"word", itemId, audioBase64})` phải gọi
  `POST /ipa/lessons/:id/attempts` với body `{kind:"word", payload:{itemId, audioBase64}}`
  + timeout 60s; trả nguyên response. Tương tự cho `sentence`, `dialogue`.
  - verify: `cd exe-web && npm test -- ipa.service` (đỏ trước khi sửa).
- [ ] **T1.2** Code: viết lại `submitIpaItem` (`ipa.service.ts:67`) → POST `/attempts`,
  bọc `{kind: payload.itemType, payload: {itemId, audioBase64, answer, tapPattern, attemptId}}`
  (bỏ field undefined). Giữ timeout 60s + comment *why* (cold model load). Không đổi
  signature/return type.
  - verify: `npm test -- ipa.service` xanh; `npx tsc --noEmit` exit 0.
- [ ] **T1.3** Test đỏ + code: `recordIpaPractice` (`ipa.service.ts:93`) →
  `POST /attempts {kind:"perception", payload:{screen, score}}`. Return `{screen,score}` không đổi.
  - verify: `npm test -- ipa.service` xanh.

---

## Phase 2 — Dọn dead code (chỉ khi grep sạch)

- [ ] **T2.1** Nếu T0.2 cho 0 caller component: xóa `recordIpaWarmup` (`ipa.service.ts:104`)
  + type/hook liên quan (`use-ipa.ts`) + import orphan. Màn warmup đã bỏ (v1.3).
  - verify: `grep -rn "recordIpaWarmup\|/warmup" exe-web/src` sạch; `tsc --noEmit` exit 0; `eslint` no unused.
- [ ] **T2.2** Nếu T0.2 cho 0 caller `submitIpaMiniTest`/`MiniTestStep`: xóa
  `submitIpaMiniTest` (`ipa.service.ts:81`) + type `IpaMiniTest*` orphan. Nếu CÒN caller →
  **giữ nguyên** (mini-test không nằm trong `/attempts`, ngoài scope) + ghi chú.
  - verify: `grep -rn "submitIpaMiniTest\|/mini-test" exe-web/src`; `tsc --noEmit` exit 0.

---

## Phase 3 — Chạy thật + regression (AC-4, AC-7)

- [ ] **T3.1** Chạy 1 bài S1–S4 trên web (FE `localhost:3000` → api `localhost:5050` →
  enova_dev): ghi âm 1 từ + 1 câu (chấm điểm), trả cảm âm 4 câu ≥3/4, mở Kết quả.
  - verify: DevTools Network — thấy `POST /ipa/lessons/:id/attempts` (không còn `/submit`,
    `/practice`); điểm + cổng cảm âm hoạt động như trước.
- [ ] **T3.2** Regression suite FE.
  - verify: `cd exe-web && npm test` (toàn bộ, 137+ ) xanh; `npx tsc --noEmit` exit 0.

---

## Phase 4 — Verify SRS 2 nhịp (AC-8, chỉ fix nếu lệch)

- [ ] **T4.1** Đọc `exe-api/.../ipa/*srs*` + hàm `advanceReview`/`recordReview`; chạy test
  `advance-review` xác nhận state machine **1 ngày → 7 ngày → mastered**, trượt không thu hồi.
  - verify: `cd exe-api/services/api && npx jest --runInBand advance-review` xanh + đọc khớp D6.
  - Nếu khớp → tick, không sửa. Nếu lệch → ghi rõ lệch + đề xuất fix (task fix riêng, cần user chốt).

---

---

# Phần B — SRS review flow (sau design sign-off)

> **GATE:** chỉ bắt đầu Phần B sau khi `design.md` §"Cần sign-off" được duyệt (F1–F5).
> BE đã sẵn (`kind:"srs_review"` → `recordReview`); Phần B là **FE UI + đường gọi**.

## Phase 5 — Verify tiền đề (không code)

- [ ] **T5.1** Đọc `startLesson` (`exe-api/.../ipa.service.js`) — xác nhận `/start` reset
  gì (screenScores/status). Chốt F5: mode review bỏ `/start` (mặc định) hay giữ được.
  - verify: đọc + ghi kết luận vào design.md (tick F5).

## Phase 6 — Runner hỗ trợ mode review (F2)

- [ ] **T6.1** Test đỏ: `buildSteps("review")` → `[{key:"luyennoi"},{key:"result"}]` (không
  `hoc`); `buildSteps()`/`buildSteps("lesson")` giữ 3 bước.
  - verify: `cd exe-web && npm test -- LessonRunnerContainer.buildSteps` đỏ→xanh.
- [ ] **T6.2** Code: `buildSteps(mode="lesson")` thêm nhánh review; `LessonRunnerContainer`
  nhận prop `mode`; mode review **bỏ `/start`** (theo T5.1). Page `/pronunciation/[lessonId]`
  đọc `searchParams.mode` truyền xuống.
  - verify: `npm test -- LessonRunnerContainer` xanh; `tsc --noEmit` exit 0.

## Phase 7 — Vòng ôn rút gọn + ghi kết quả (F3/F4/F6)

- [ ] **T7.1** `SpeakingLoopStep` thêm cờ `review`: ẩn khối cảm âm, cap tối đa 3 mục (con số
  từ design.md F3). Test: mode review không render cảm âm + ≤3 mục.
  - verify: `npm test -- SpeakingLoopStep` xanh.
- [ ] **T7.2** Tính `score` ôn = trung bình điểm mục đã chấm (F4 Option A); ở step `result`
  mode review fire **đúng 1 lần** `POST /attempts {kind:"srs_review", payload:{score}}` (guard
  double-post). Test: gọi 1 lần, body đúng shape.
  - verify: `npm test -- ipa.service SpeakingLoopStep` xanh.
- [ ] **T7.3** `ResultStep` biến thể review: thông điệp đạt/trượt ôn, ẩn CTA/Luyện-lại không
  hợp ngữ cảnh. Test render 2 nhánh.
  - verify: `npm test -- ResultStep` xanh.

## Phase 8 — Entry "Ôn tập hôm nay" (F1)

- [ ] **T8.1** Hook `useIpaDueReviews` (wrap `getIpaDueReviews` đã có) + khối UI trên
  `/pronunciation`: list bài tới hạn (empty state khi rỗng), bấm → `/pronunciation/:id?mode=review`.
  - verify: `npm test -- <due-reviews component>` xanh; `tsc --noEmit` exit 0.

## Phase 9 — E2E Phần B + regression toàn bộ

- [ ] **T9.1** Chạy thật: 1 bài `learning` tới hạn → vào ôn (bỏ Học), làm rút gọn, đạt ≥80 →
  Network thấy `srs_review`; `GET /progress` phản ánh nhịp mới; ôn trượt → giữ bài + `needsReview`.
- [ ] **T9.2** Regression: `cd exe-web && npm test` (toàn bộ) xanh; `tsc --noEmit` exit 0.

---

## Ghi chú

- **Surgical:** không đổi hook/component; chỉ đổi trong `ipa.service.ts` + test mới. Không
  đụng `modules/review/` (PRD-142). Không gỡ route legacy BE (PR khác).
- **Comment code** theo `.claude/rules/review-audit-self-decision.md` §5: giải thích *why*
  (contract gộp, timeout cold-load), không ghi mã phase/FR/backlog trong code/commit.
- **Commit** conventional, base `develop`, không AI-attribution. FE PR riêng (cross-repo,
  không chung commit với BE).
- Bàn giao code: `/superpowers execute studee-workspace/specs/m2-ipa-attempts-migration/tasks.md`.
