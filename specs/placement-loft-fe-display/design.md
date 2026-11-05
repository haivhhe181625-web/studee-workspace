---
status: draft
spec: ./spec.md
created: 2026-09-02
---

# Design — Surface LOFT khung + fixed-length progress

Nguyên tắc: chỉ **expose data đã có** + render. Không đụng engine/scoring/selection.

## P1 — Admin skeleton

### BE (exe-api)
- `bank-stats.service.js` `getPlacementCoverage()`: import `PLACEMENT_LINEAR_SKELETON` từ
  `constants/blueprint.js`; thêm vào object trả về: `skeleton: PLACEMENT_LINEAR_SKELETON`.
  (Q1 chốt: đặt ở service, 1 nơi. `insights.admin.js` passthrough — không thêm logic.)
- Không đổi `ok/cellGaps/clusteredCells/rows`.

### FE (exe-admin)
- `services/admin.service.ts`: `PlacementCoverage` += `skeleton: { reading: string[]; listening: string[] }`.
- `placement-framework.tsx`: 1 dải nhỏ (dưới header, trên 2 panel). Map tier→CEFR label bằng hằng FE:
  `{ easy:'A2', mid:'B1/B2', hard:'C1' }`. Render:
  `Dải giao đề (linear): Reading A2 → B1/B2 → B1/B2 → C1 · Listening A2 → B1/B2 → B1/B2 → C1 → B1/B2`
  (map từ mảng skeleton, không hard-code — nếu skeleton đổi thì FE tự đổi). Style bằng class có sẵn,
  không sửa components/ui.

## P2 — Web fixed-length progress

### BE (exe-api)
- `constants/blueprint.js` (Q2 chốt): hằng nhỏ cạnh CORE_PLACEMENT_PRESET:
  `const PLACEMENT_ITEMS_PER_MODULE = Object.freeze({ reading: 4, listening: 3 });`
- `assessment.service.js` (chỗ build config placement, ~sau khi set assemblyMode + CORE_PLACEMENT_PRESET):
  nếu `assemblyMode ∈ {'fixed','linear'}` → thêm
  `config.expectedItemCount = Σ_skill config.fixedModuleCounts[skill] × PLACEMENT_ITEMS_PER_MODULE[skill]`
  (placement preset → 4×4 + 5×3 = 31). Adaptive: KHÔNG set (để undefined).
- Field này nằm trong `config` trả về ở intake response → IntakeWizard cache nguyên config → FE đọc được.
  (Q3 chốt: cache lưu nguyên `AssessmentConfig`, chỉ cần field có trong config.)

### FE (exe-web)
- `types/assessment.ts` (AssessmentConfig): thêm `expectedItemCount?: number`.
- `MstExamFlow.tsx`: đọc `config.expectedItemCount`. Progress hiện:
  - có `expectedItemCount` → `câu {totalAnswered}/{expectedItemCount}` (+ thanh % nếu đang có).
  - không có (adaptive / cache miss / attempt cũ) → **giữ nguyên** tử-số-only (hiện tại) — fallback an toàn.
- Cập nhật comment ~line 82 (giải thích cũ "adaptive không có mẫu số") cho khớp: giờ fixed/linear có mẫu số.

## Ràng buộc / rủi ro
- **Không regress adaptive**: expectedItemCount chỉ set cho fixed/linear → adaptive nhánh cũ nguyên.
- **expectedItemCount là ước tính danh nghĩa** (giả định 4 item/passage, 3/audio). Nếu 1 testlet thật có
  số câu lệch → mẫu số có thể lệch ±1–2. Chấp nhận cho thanh tiến độ (không phải con số chấm). Ghi chú ở code.
- **tier→CEFR thô** (mid=B1/B2) → nhãn hiển thị "B1/B2", không khẳng định 1 mức.
- Cross-repo: BE (exe-api) phải merge trước hoặc cùng đợt để FE có data; nếu FE lên trước, field thiếu →
  fallback (P1 skeleton undefined → ẩn dải; P2 → tử-số-only). Không vỡ.
