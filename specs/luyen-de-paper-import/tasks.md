---
feature: luyen-de-paper-import
status: implemented
created: 2026-08-17
role: tech-lead
---

# Tasks — Import đề luyện thi (TDD checkbox). Repo: exe-api. Test: `npm test` (cwd services/api).

Thứ tự: T1 (model) → T2 (validate) → T3 (service) → T3b (publish free-form) → T4 (CLI) → T5 (gate). Mỗi task 1 PR-worth, commit riêng.

**✅ HOÀN TẤT 2026-08-17** — 6 commit `e157850→4c2694d`: T1 `e157850` · T2 `4b5076e` · T3 `c24df53`(+`fb8e917`) · T3b `4782f8c` · T4 `4c2694d` · T5 full suite 3017 pass (2 fail pre-existing). CLI verify: import sample → 1 stimulus + 3 câu draft, frozenItemIds đúng thứ tự, 0 leak answerKey.

---

- [ ] **T1. Provenance subdoc trên model**
  - File: `modules/assessment/assessment-question.model.js`, `modules/assessment/assessment-stimulus.model.js`.
  - Thêm `ProvenanceSchema {sourceRef, importBatch, attribution}` (`_id:false`, default null) + `provenance: {default: () => ({})}` (data-model §3).
  - Test: model chấp nhận provenance; item cũ default `{}`; không đổi field khác.
  - Verify: `npm test -- assessment-question assessment-stimulus`.

- [ ] **T2. Validator thuần `import-validate.js`**
  - File mới: `modules/test-paper/import-validate.js`. Export `validateCanonical(canonical)` → ném lỗi hoặc trả `{contentSource}`.
  - Kiểm: schemaVersion=="1.0"; exam/skill enum; sections không rỗng; mỗi question itemType ∈ ITEM_TYPES + answerKey shape theo bảng (data-model §2); `provenance.license`→contentSource (reject unknown).
  - Test-first (pure, không DB): file hợp lệ pass; từng lỗi (itemType lạ; mcq answerKey không phải số/string; matching_headings không phải array; license unknown) → throw đúng (AC3, AC4).
  - Verify: `npm test -- import-validate`.

- [ ] **T3. `import.service.js` — importPaper(canonical)**
  - File mới: `modules/test-paper/import.service.js`. `importPaper(canonical)`:
    1. `validateCanonical`.
    2. Mỗi section: stimulus `ref`→dùng id (AC5, kiểm tồn tại **+ `skill===paper.skill`**, reject nếu khác — F3); inline→create draft (audioFile→copy `STORAGE_LOCAL_ROOT`→audioUrl, AC6). questions→create draft (cefrLevel/itemType/answerKey/... status draft, centerId null, contentSource, provenance).
    3. `TestPaper.create` draft + frozenItemIds theo thứ tự (AC1, AC2).
    4. return `{paperId, counts, warnings}`.
  - Test-first (dùng mongodb-memory-server): tạo đúng số doc + draft + centerId null (AC1); frozenItemIds đúng thứ tự (AC2); stimulus ref reuse (AC5); không leak answerKey ra ngoài select (AC7); blueprintKey vắng vẫn chạy (AC8); validate fail → không tạo doc nào (AC3).
  - Verify: `npm test -- import.service`.

- [ ] **T3b. `publish()` tolerate free-form (F1 — BLOCKING)**
  - File: `modules/test-paper/test-paper.service.js` (`publish`).
  - Nếu `paper.blueprintKey == null` → **bỏ `validatePaper`**, chỉ flip `status='published'`. Có blueprintKey → giữ validate cũ.
  - Test-first: publish đề free-form (blueprintKey null) → thành `published`, KHÔNG throw; publish đề có blueprintKey lệch khung → vẫn reject (regression cũ xanh).
  - Verify: `npm test -- test-paper.service`.

- [ ] **T4. CLI `scripts/ops/import-paper.js`**
  - File mới: `scripts/ops/import-paper.js <file.json>` → connect Mongo → đọc file → `importPaper` → in preview (paperId, counts, warnings, cảnh báo "draft — cần vet trước publish").
  - Verify: tạo 1 file JSON mẫu (reading testlet + câu độc lập) → `docker exec exe_api node scripts/ops/import-paper.js <file>` → in preview, DB có đề draft. (Manual — ops script không unit test.)

- [ ] **T5. Regression gate**
  - Full suite exe-api xanh; engine assessment/test-paper 0 regression (AC9).
  - Verify: `npx jest --testTimeout=30000` (chấp nhận 3 fail pre-existing đã biết: avatar EPERM, campaign parallel-timeout, rate-limit redis-mock).

---

## Ghi chú
- KHÔNG đụng assemble (import set frozenItemIds trực tiếp). KHÔNG center scoping (centerId=null). KHÔNG blueprint enforcement. KHÔNG auto-publish. English code/comment; không tham chiếu mã plan/finding trong code.
- Hand-off: `/superpowers execute studee-workspace/specs/luyen-de-paper-import/tasks.md` (sau khi owner duyệt spec).
