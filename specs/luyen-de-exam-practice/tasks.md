# Tasks: Luyện đề (IELTS/TOEIC Exam Practice) — Phase 1

- **Ngày:** 2026-08-14
- **Kèm:** `spec.md`, `design.md`, `data-model.md`
- **Repo chính:** `exe-api` (BE). `exe-web`/`exe-admin` = PR riêng (task FE liệt kê để tracking).
- Mỗi task: TDD-checkbox, 1 PR-worth, kèm verify command. Chạy test theo `exe-api/TESTING.md` (`npm test`).

## Thứ tự phụ thuộc

T1→T2 (enum trước blueprint). T3 (explanation) độc lập. T4 dựa T1-T2. T5 dựa T4. T6 dựa T3+T5. T7 dựa T5. T8 (review) dựa T7. T9 admin dựa T4. T10-T12 FE dựa API. T13 content song song.

---

### BE — exe-api

- [x] **T1. Mở rộng `ITEM_TYPES` enum (IELTS types)**
  - File: `modules/assessment/assessment-question.model.js`
  - Append: `matching_information`, `matching_features`, `yes_no_ng`, `matching_sentence_endings` (Technical Review 2026-08-17 ĐÃ APPROVE).
  - Trước sửa: `grep -r "ITEM_TYPES\|itemType" exe-api/services/api/src` → xác nhận append-only không vỡ chỗ dùng.
  - Verify: `npm test -- assessment-question` + test cũ xanh.

- [x] **T2. `paper-blueprint.js` + `validatePaper`**
  - File mới: `modules/test-paper/paper-blueprint.js`
  - 4 blueprint (ielts-academic-reading, ielts-listening, toeic-listening, toeic-reading) + `getBlueprint`, `validatePaper`.
  - Test-first: `paper-blueprint.test.js` — validatePaper pass với tập item khớp, fail khi thiếu/sai itemType per section (AC#1).
  - Verify: `npm test -- paper-blueprint`.

- [x] **T3. Thêm field `explanation` vào AssessmentQuestion**
  - File: `assessment-question.model.js` (field), không `select:false`.
  - Test: model chấp nhận explanation; `toDto` giữ; item cũ default null.
  - Verify: `npm test -- assessment-question`.

- [x] **T4. `TestPaper` model + service.assemble/publish**
  - Files mới: `modules/test-paper/test-paper.model.js`, `test-paper.service.js`.
  - `assemble(blueprintKey, filter)`: query `assessment_questions` (status active, targetGoal, cefr/difficulty theo section) → build frozenItemIds. `publish(id)`: gọi `validatePaper` → set published (atomic single-doc). Reject publish nếu validate fail.
  - Test-first: assemble ra đúng số item; publish fail khi tập item không khớp khung (AC#2).
  - Verify: `npm test -- test-paper.service`.

- [x] **T5. `PracticeAttempt` model + start/submit service**
  - Files mới: `modules/test-paper/practice-attempt.model.js`, thêm vào `test-paper.service.js` (hoặc `practice.service.js` nếu >200 dòng).
  - `start(userId, paperId)`: snapshot itemIds (+shuffle optionOrder), tạo attempt in_progress, trả items **KHÔNG** answerKey/explanation.
  - `submit(userId, attemptId, answers)`: `grade.gradeObjective` từng câu → score {raw,total,percent} + `weakByItemType` → trả per-item {correct, answerKey, explanation}.
  - Test-first: start không leak explanation/answerKey (AC#3); submit chấm đúng + breakdown đúng (AC#4).
  - Verify: `npm test -- practice-attempt`.

- [x] **T6. Explanation leak-control ở controller/routes**
  - Files mới: `modules/test-paper/test-paper.controller.js`, `.routes.js`. Mount router (verifyToken + withTenant).
  - `GET /api/test-papers`, `POST /api/test-papers/:id/start`, `POST /api/practice-attempts/:id/submit`.
  - `start` dùng `.select('-explanation -answerKey')`; `submit` mới trả explanation.
  - Test-first (API): start response assert KHÔNG chứa `explanation`/`answerKey`; submit response CÓ (AC#3).
  - Verify: `npm test -- test-paper.api`.

- [~] **T7. Resume đề dở + auto-nộp hết giờ** (M7-12, brainstorm 2026-08-17 → P1) — BE xong (resume/getInProgress/expiry + routes); FE ở exe-web PR riêng
  - BE: `PracticeAttempt` tính deadline server-side (`startedAt`+`timeLimitSec`); endpoint resume trả attempt in_progress + remainingSec; submit quá hạn → `expired`/chấm phần đã làm.
  - FE: /luyen-de hiện "tiếp tục đề dở"; ExamTimer dựng deadline từ remainingSec+serverTime (như PRD-124); hết giờ tự nộp.
  - Test: deadline server-side; e2e rớt-mạng→resume→nộp; e2e hết giờ→auto-submit.

  > ~~T-cũ. Emit learning_events + enqueueMistakes~~ → **DEFER P2** (M7-6/M7-14). P1 dùng review-in-context, KHÔNG đụng `learning-event.model.js`/`modules/review`.

- [x] **T9. Admin CRUD `test-paper.admin.js`**
  - File mới: `admin-api/resources/test-paper.admin.js` (config `_crud.factory.js`). Action "publish" delegate `test-paper.service.publish` (KHÔNG business logic ở admin-api).
  - Permission: thêm `test_paper:<action>` vào `constants/permissions.js`.
  - Test: list/create/publish qua `/api/admin/test-papers`; publish gọi service.
  - Verify: `npm test -- admin` (hoặc test admin liên quan).

- [x] **T10. Regression gate** (dev-side verified 2026-08-17)
  - Full suite: **2983 pass**. Engine assessment/review/adaptive/grade/test-paper/admin đều xanh (0 regression từ thay đổi Luyện đề).
  - 3 fail PRE-EXISTING, ngoài bán kính: `avatar-storage` (EPERM Windows temp), `campaign.integration` (pass khi chạy riêng — timeout do parallel), `rate-limit.trial-mode` (1 test redis-mock có sẵn). QA (Linh) sign-off chính thức.
  - Verify: `npx jest --testTimeout=30000`.

### FE — exe-web (PR riêng)

- [ ] **T11. Danh sách đề + trang làm đề READING** (M7-9)
  - Reuse `ExamContainer`/`QuestionCard`/`ExamTimer` (`src/components/features/assessment/exam/`). Service `test-paper.service.ts`. Route `(main)/luyen-de`; list hiện `targetBandRange`+nhãn độ khó để chọn đề đúng trình độ. **Chỉ Reading** (Listening → T14).
  - Verify: Vitest component + e2e làm→nộp.

- [ ] **T12. Trang review-sau-nộp (review-in-context)** (M7-10)
  - Score raw+% + đối chiếu `targetBandRange` của đề (KHÔNG band cá nhân) + `weakByItemType`. Per-item đáp án + explanation **kèm passage** để ôn tại chỗ. KHÔNG link Sổ lỗi (defer P2).
  - Verify: Vitest; assert explanation chỉ hiện sau nộp.

- [ ] **T14. Listening mock runner** (M7-13, hạng mục lớn — KHÔNG có sẵn trong ExamContainer)
  - Audio phát 1 lần (chặn tua), đồng bộ câu hỏi theo timing, tự chuyển section, khoá điều hướng ngược. Reuse audio player M1 nhưng orchestration mới.
  - Verify: e2e phát 1 lần→chặn tua→auto chuyển section→nộp.

### Content (song song, không chặn code)

- [ ] **T13. Author item + seed 4 đề**
  - IELTS: dùng bank sẵn có, bổ sung explanation. TOEIC: AI-gen (`gen-testlet`) ~200 item P1-7 + examiner vet + explanation.
  - Verify: `node scripts/ops/bank-status.js` cho thấy đủ item per blueprint; assemble 4 đề publish thành công.

---

## Ghi chú deferred (documented, KHÔNG làm P1)
- **Sổ lỗi/SRS cho luyện đề (M7-6, P2):** emit learning_events(practice) + enqueueMistakes + re-drill item passage trong `modules/review`. P1 thay bằng review-in-context.
- **Band/CEFR ước lượng cá nhân (M7-14, P2):** cần calibration; P1 chỉ đối chiếu targetBandRange của đề.
- Micro-drill theo itemType (P2).
- IELTS Writing/Speaking chấm AI (P3).
- Dynamic/adaptive assembly `TestPaper.mode='dynamic'` (P3).
- General Training, TOEIC S/W, entitlement gating, band prediction chính thức.
