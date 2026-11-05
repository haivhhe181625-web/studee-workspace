---
status: draft
owner: BA
created: 2026-09-02
affects: [exe-api, exe-admin, exe-web]
depends_on: placement-linear-loft-mode (assemblyMode 'linear' + PLACEMENT_LINEAR_SKELETON)
---

# Spec — Surface LOFT khung + fixed-length progress (FE)

## 1. Mục tiêu & why
Vừa thêm mode `linear` (LOFT): khung giao đề placement giờ là **dải slot cố định**
(`PLACEMENT_LINEAR_SKELETON`). 2 enhancement FE (tuỳ chọn, user duyệt) để bề mặt khớp code mới:
- **P1:** admin thấy **cấu trúc giao đề thật** (skeleton) cạnh lưới coverage — "hiển thị khung" đúng nghĩa mới.
- **P2:** bài thi hiện **tiến độ có mẫu số** ("câu N/31") cho fixed/linear (độ dài cố định, biết trước),
  thay vì chỉ tử số như hiện tại (vốn giả định adaptive biến độ dài).

FE hiện đã tương thích code LOFT (không bắt buộc sửa) — đây là 2 việc **thêm giá trị hiển thị**.

## 2. Phạm vi

### P1 — Admin hiển thị skeleton (exe-api BE + exe-admin FE)
- **BE (exe-api):** `getPlacementCoverage()` (bank-stats.service) trả kèm field
  `skeleton: { reading:[...], listening:[...] }` từ `PLACEMENT_LINEAR_SKELETON` (import blueprint.js).
  Endpoint admin `/insights/placement-coverage` passthrough — không thêm logic ở admin-api.
- **FE (exe-admin):** `placement-framework.tsx` render 1 dải nhỏ (trên/dưới 2 panel):
  "Dải giao đề (linear): Reading A2→B1→B2→C1 · Listening A2→B1→B2→C1→B1" (map tier→CEFR).
  Thêm `skeleton` vào type `PlacementCoverage` (admin.service.ts).

### P2 — Web tiến độ có mẫu số (exe-api BE + exe-web FE)
- **BE (exe-api):** config placement (assessment.service) mang thêm `expectedItemCount` cho mode
  ∈ {fixed, linear} (null/omit cho adaptive). Giá trị = Σ `fixedModuleCounts[skill] × itemsPerModule[skill]`
  (reading 4, listening 3 → 31). Field phải nằm trong config được cache client (readIntakeConfig).
- **FE (exe-web):** `MstExamFlow.tsx` đọc `config.expectedItemCount`; nếu có → hiện "câu N/{total}";
  nếu không (adaptive) → giữ nguyên chỉ tử số.

### Out of scope
- KHÔNG đụng engine LOFT/scoring/selection.
- KHÔNG expose config/chọn assemblyMode ở admin (v1 nội bộ — spec linear-loft §5 Q3).
- Map tier→CEFR ở FE là **thô** (mid=B1|B2) — hiển thị "B1/B2" hoặc theo skeleton design; không cố pin mịn.
- Đổi cách chấm/CEFR/receptive naming (feature khác).

## 3. Acceptance Criteria
**P1**
- [ ] `GET /admin/insights/placement-coverage` trả `skeleton.reading` (4) + `skeleton.listening` (5).
- [ ] Tab "Khung đầu vào" hiện dải giao đề linear (R 4 slot / L 5 slot, map CEFR), cạnh 2 panel coverage.
- [ ] Không regress lưới coverage cũ; tsc/lint pass; không sửa components/ui.

**P2**
- [ ] Config placement fixed/linear có `expectedItemCount` (=31 với preset R16/L15); adaptive không có.
- [ ] Bài thi fixed/linear hiện "câu N/31"; adaptive hiện tử-số-only (không regress).
- [ ] Fallback an toàn khi config thiếu field (attempt cũ/cache miss) → tử-số-only, không vỡ.

## 4. Affected surfaces
- exe-api: `modules/assessment/bank-stats.service.js` (skeleton), `assessment.service.js` (expectedItemCount),
  có thể `constants/blueprint.js` (hằng itemsPerModule nếu cần).
- exe-admin: `services/admin.service.ts` (type), `components/features/question-bank/placement-framework.tsx`.
- exe-web: `lib/intake-config-cache` (carry field), `components/features/assessment/exam/MstExamFlow.tsx`.

## 5. Open Questions
- Q1 (P1): skeleton vào `getPlacementCoverage` service hay merge ở insights.admin? → đề xuất **service** (1 nơi).
- Q2 (P2): `itemsPerModule` (reading 4/listening 3) — hằng mới trong blueprint.js hay derive? → đề xuất **hằng nhỏ trong blueprint.js** cạnh CORE_PLACEMENT_PRESET.
- Q3 (P2): config-cache (`readIntakeConfig`) hiện có carry field mới không, hay cần thêm ở intake response DTO + chỗ ghi cache? → **chốt ở design** sau khi đọc intake-config-cache.
