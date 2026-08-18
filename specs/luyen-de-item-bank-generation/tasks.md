---
feature: luyen-de-item-bank-generation
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Tasks — Sinh dữ liệu ngân hàng theo khung (Bước 2/3)

> Phụ thuộc **Bước 1** (khung ở DB). Không code trước khi Bước 1 xong. Tái dùng gen-testlet/bank-coverage/
> dedup nguyên trạng — chỉ thêm lớp điều phối.

## T1 — Coverage planner cho exam-practice (pure)

**File:** `src/modules/test-paper/exam-practice-coverage.js` (mới, pure) · test kèm

- [ ] `planExamPracticeCoverage(blueprints, bankRows, { surplus })`: gom section → cell
  `{skill, cefr, itemType, targetGoal}`, `need = Σ itemCount(section dùng cell) × surplus`; join với
  `bankRows` (active) → `have`, `gap = max(0, need - have)`. Reuse `bank-coverage.js` nếu khớp.
- [ ] Map cefr→tier cho gọi gen (A2→easy, B1→mid, C1→hard; B2→hard+cefrHint, ghi chú).
- [ ] Test: khung ielts-listening + bank rỗng → gap = need; bank đủ → gap 0; cell dùng chung (2 section) cộng dồn.

**Verify:** `npx jest exam-practice-coverage` → xanh.

## T2 — Batch runner (queue)

**File:** `src/modules/test-paper/gen-fill.service.js` (mới) · `src/queues/*` (job) · test kèm

- [ ] `runGenerationBatch(cells, { maxAttemptsPerCell, maxCost })`: mỗi cell gap>0 → loop
  `genTestlet.generateAndInsertTestlet({ skill, tier, itemTypes:[type], targetGoal, topic })` tới gap≤0 hoặc
  chạm trần; luôn truyền `targetGoal`.
- [ ] Dùng BullMQ job + throttle (LLM rate limit); tiến độ ghi lại đọc được.
- [ ] Chạm trần → dừng + report cell chưa đủ (AC2). Không loop vô hạn.
- [ ] Test (mock gen): cell gap=3 → gọi gen tới khi đủ; gen fail liên tục → dừng ở maxAttempts + report.

**Verify:** `npx jest gen-fill` → xanh.

## T3 — Admin endpoint + FE coverage

**File:** `src/admin-api/resources/test-paper.admin.js` (thêm route coverage + trigger batch) ·
exe-admin màn coverage

- [ ] `GET /test-papers/coverage` → planExamPracticeCoverage (ma trận cell + gap).
- [ ] `POST /test-papers/generate-fill` → enqueue batch job (perm `test-paper:write`).
- [ ] FE: bảng cell thiếu (skill×cefr×itemType×exam) + nút chạy + tiến độ job.
- [ ] Test: coverage 200; generate-fill enqueue 202/201; thiếu quyền 403.

**Verify:** `npx jest test-paper` + exe-admin build/lint.

## Definition of done

- [ ] Planner + runner test xanh; mồi thử 1 khung (vd ielts-listening) trên dev → bank đủ → assemble
  `got===requested` sau vet.
- [ ] 0 regression assessment/generation/test-paper.

## Câu hỏi mở

- surplus factor, queue vs script, no-human-review, prerequisite OPENAI_API_KEY cho dedup semantic (spec/design).
