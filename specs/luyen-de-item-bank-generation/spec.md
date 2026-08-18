---
feature: luyen-de-item-bank-generation
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
epic: PRD-151 (M7 Luyện đề)
step: 2/3 (blueprint storage → item generation → difficulty assembly)
---

# Spec — Sinh dữ liệu ngân hàng câu theo khung đề (Bước 2/3)

## 1. Mục tiêu & lý do

Ngân hàng câu exam-practice còn mỏng (TOEIC rỗng, IELTS ít) → `assemble` báo thiếu, không tạo đủ đề. Cần
làm dày bank **rẻ token, đủ số lượng, có kiểm soát chất lượng**.

**Không dựng generation mới** — hạ tầng đã có và production-grade (đối chiếu khi scout, xem `design.md §1`):
`gen-testlet.service.generateAndInsertTestlet()` gọi LLM sinh testlet (passage + câu) → quality gates
(moderation + dedup Qdrant) → insert `draft`/`ai-generated-reviewed` + metadata đầy đủ (cefr, targetGoal,
`irt.b` tạm theo tier). Đã expose admin `POST /stimuli/_/generate`. `bank-coverage.js` đã có gap analysis.

Bước 2 = **lớp điều phối blueprint-driven** trên hạ tầng đó: từ yêu cầu của các khung → tính cell còn
thiếu → chạy batch generation nhắm đúng `(skill, cefr/tier, itemType, targetGoal)` → item vào bank qua đúng
vet gate.

## 2. Phạm vi

- **BE (exe-api):**
  - **Coverage planner cho exam-practice**: gom yêu cầu từ tất cả khung `active` (mỗi section →
    cell `{skill, cefr, itemTypes, targetGoal}` × `itemCount` × hệ số dư an toàn) → so với bank hiện có
    (`bank-coverage.js` gap analysis) → ra danh sách cell thiếu + số cần sinh.
  - **Batch runner** dùng BullMQ (đã có trong stack) hoặc script ops: lặp cell thiếu → gọi
    `generateAndInsertTestlet({ skill, tier, itemTypes, targetGoal, topic })` cho tới khi đủ; tôn trọng
    quality/dedup gate sẵn có; item ra `draft`/`ai-generated-reviewed` (KHÔNG auto-active — xem §3).
  - **Đảm bảo `targetGoal`** ('ielts'|'toeic') được set trên mọi item sinh cho exam-practice (assemble lọc
    theo targetGoal — thiếu là assemble không thấy).
- **FE (exe-admin):** màn "Coverage & sinh câu theo khung" — bảng cell thiếu (skill×cefr×itemType×exam) +
  nút chạy batch, xem tiến độ. Tái dùng service generate sẵn có.
- **Vet**: item sinh nằm `draft` → examiner duyệt (hoặc bật no-human-review khi gate đủ tin, theo cấu hình
  hiện có của gen-testlet) → `active` mới vào được đề.

## 3. Ngoài phạm vi

- **Viết lại engine generation / quality gate / dedup** — tái dùng nguyên trạng.
- **Auto-publish item thẳng `active` không vet** — mặc định `draft`; chế độ no-human-review là tuỳ chọn
  vận hành sẵn có, không bật mặc định ở feature này.
- **Import dataset ngoài (CC BY)** — hướng bổ sung, tách spec riêng nếu cần (research đã có nguồn).
- **Calibrate IRT thật từ dữ liệu trả lời** — `irt.b` vẫn là giá trị tạm theo tier; calibrate là việc của
  `item-quality.service` sau khi có lượt làm.
- **Bật tier/IRT thành filter trong assemble** — Bước 3.

## 4. Acceptance Criteria

- [ ] AC1 — Planner đọc khung `active` → trả ma trận cell `{skill, cefr, itemType, targetGoal, need, have,
  gap}` đúng số học (need = Σ itemCount các section dùng cell đó × hệ số dư).
- [ ] AC2 — Batch runner sinh đủ tới khi `gap ≤ 0` cho các cell nhắm tới, HOẶC dừng khi chạm trần
  (max attempts / max cost) và **báo rõ** cell nào chưa đủ (không âm thầm bỏ).
- [ ] AC3 — Mọi item sinh cho exam-practice có `targetGoal` đúng ('ielts'|'toeic'), `contentSource =
  'ai-generated-reviewed'`, `status='draft'`, cefr khớp cell.
- [ ] AC4 — Item trùng bị chặn bởi dedup gate sẵn có (Qdrant); moderation FLAGGED bị loại.
- [ ] AC5 — Sau khi vet 1 batch → `active`, `assemble` khung tương ứng tạo được đề đủ câu (`got ===
  requested`).
- [ ] AC6 — Batch không chặn event loop / không nổ rate limit LLM: chạy qua queue hoặc có throttle; theo dõi
  được tiến độ.
- [ ] AC7 — Phân quyền `test-paper:write` (hoặc quyền gen sẵn có); FE hiển thị coverage + tiến độ batch.
- [ ] AC8 — Full suite exe-api xanh, 0 regression assessment/generation/test-paper.

## 5. Repos ảnh hưởng

- **exe-api** — planner + batch runner (tái dùng gen-testlet/bank-coverage/dedup).
- **exe-admin** — màn coverage + chạy batch.

## 6. Liên kết

- Research: `plans/reports/research-260818-ielts-toeic-paper-structure-for-blueprint-model.md` (§sinh dữ liệu,
  cửa chất lượng, provenance).
- Hạ tầng tái dùng: `assessment/gen-testlet.service.js`, `gen-quality.service.js`, `dedup.service.js`,
  `bank-coverage.js`, admin `assessment.admin.js` action `generate`.
- Phụ thuộc: **Bước 1** (`luyen-de-blueprint-storage`) — planner đọc khung từ DB. Làm sau Bước 1.
- Bước sau: `specs/luyen-de-difficulty-assembly/` (Bước 3) — hưởng lợi khi bank đã dày.

## Câu hỏi mở

- Hệ số dư an toàn (need = itemCount × ?) — 2×? 3×? Cần đủ để assemble có chỗ chọn ngẫu nhiên + Bước 3 lọc
  tier. Chốt khi review (research gợi ý bank dư ≥ 2× nhu cầu để random có tác dụng).
- Queue (BullMQ) vs script ops cho batch — design đề xuất queue (LLM chậm, cần throttle + resume); xác nhận.
- Bật no-human-review khi nào (tin gate) vs luôn vet tay — quyết định vận hành, hỏi owner.
