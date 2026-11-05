---
status: draft
spec: ./spec.md
design: ./design.md
created: 2026-09-02
---

# Tasks — Surface LOFT khung + fixed-length progress

Cross-repo. BE trước (FE phụ thuộc field). Không đụng engine/scoring. No AI attribution.

---

## T1 — [BE exe-api] getPlacementCoverage trả `skeleton`
Repo exe-api. File: `services/api/src/modules/assessment/bank-stats.service.js`.
- [ ] Import `PLACEMENT_LINEAR_SKELETON` từ `../../constants/blueprint`.
- [ ] Thêm `skeleton: PLACEMENT_LINEAR_SKELETON` vào object `getPlacementCoverage()` trả về.
  Không đổi ok/cellGaps/clusteredCells/rows.
- **Verify:** unit test bank-stats.placement — return có `skeleton.reading.length===4`, `.listening.length===5`.
  `docker exec exe_api npx jest bank-stats.placement`. (Hoặc test coverage sẵn có + assert field.)

## T2 — [FE exe-admin] render dải skeleton trong "Khung đầu vào"
Repo exe-admin. Nhánh: feat/PRD-288-placement-admin-view.
- [ ] `src/services/admin.service.ts`: `PlacementCoverage` += `skeleton: { reading: string[]; listening: string[] }`.
- [ ] `src/components/features/question-bank/placement-framework.tsx`: dải nhỏ dưới header/trên 2 panel,
  map tier→CEFR `{easy:'A2',mid:'B1/B2',hard:'C1'}` từ `data.skeleton`. UI tiếng Việt: "Dải giao đề (linear): …".
  Ẩn dải nếu `skeleton` undefined (BE chưa lên). Không sửa components/ui.
- **Verify:** `npx tsc --noEmit` không lỗi mới (question-bank/admin.service); render dải đúng thứ tự A2→…→C1.

## T3 — [BE exe-api] expectedItemCount cho config fixed/linear
Repo exe-api. Files: `constants/blueprint.js`, `modules/assessment/assessment.service.js`.
- [ ] `blueprint.js`: `const PLACEMENT_ITEMS_PER_MODULE = Object.freeze({ reading:4, listening:3 });` + export.
- [ ] `assessment.service.js` (sau khi set assemblyMode + CORE_PLACEMENT_PRESET cho placement): nếu
  `config.assemblyMode ∈ {'fixed','linear'}` → `config.expectedItemCount = Σ config.fixedModuleCounts[skill] ×
  PLACEMENT_ITEMS_PER_MODULE[skill]` (placement → 31). Adaptive: không set. Chỉ cho goal placement.
- **Verify:** unit — intake placement (default linear) → config.expectedItemCount===31; betaAdaptive → undefined.
  `docker exec exe_api npx jest assessment.service` (hoặc suite createFromIntake defaults).

## T4 — [FE exe-web] tiến độ có mẫu số
Repo exe-web. Files: `types/assessment.ts`, `components/features/assessment/exam/MstExamFlow.tsx`.
- [ ] `AssessmentConfig` += `expectedItemCount?: number`.
- [ ] `MstExamFlow.tsx`: đọc `config.expectedItemCount`; progress hiện `câu {totalAnswered}/{expectedItemCount}`
  khi có; fallback tử-số-only khi không (adaptive/cache miss). Cập nhật comment cũ (~line 82).
- **Verify:** `npx tsc --noEmit`; test MstRunner/MstExamFlow — có expectedItemCount → hiện mẫu số; không có →
  tử-số-only (không regress). `npx jest MstExamFlow` / `MstRunner`.

---

## Done
- [ ] T1–T4 verify pass; không regress lưới coverage cũ + progress adaptive cũ.
- [ ] Diff + acceptance trình user trước merge.

## Đã chốt (design)
- Q1 skeleton ở getPlacementCoverage (service). · Q2 PLACEMENT_ITEMS_PER_MODULE hằng trong blueprint.js. ·
  Q3 config-cache lưu nguyên AssessmentConfig → chỉ thêm field.
