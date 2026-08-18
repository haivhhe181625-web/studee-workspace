---
feature: luyen-de-paper-assemble
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Tasks — Nối assemble ra admin + random thật

Thứ tự: BE service → BE admin route → FE. Mỗi task independently-testable. Test-first ở exe-api (đã có
suite test-paper). Verify command chạy từ `exe-api/services/api`.

---

## T1 — `assemble` random thật (`$sample`)

**File:** `src/modules/test-paper/test-paper.service.js` ·
test `src/modules/test-paper/__tests__/test-paper.service.test.js`

- [x] Đổi vòng lặp section trong `assemble` từ `AssessmentQuestion.find(...).limit(itemCount).select('_id').lean()`
  sang `AssessmentQuestion.aggregate([{ $match }, { $sample: { size: itemCount } }, { $project: { _id: 1 } }])`.
  `$match` giữ **nguyên** filter cũ kể cả `_id: { $nin: frozenItemIds }` (design.md §1).
- [x] Giữ nguyên phần còn lại (push `d._id`, tạo TestPaper) — chữ ký + return **không đổi**.
- [x] Test cũ giữ pass: "builds full frozenItemIds" (đủ bank → 40), "no reuse across sections"
  (toeic-listening → 100, không trùng), publish tests.
- [x] Thêm test AC2 random: seed bank **dư** (vd `perSection = s => s.itemCount * 2`) cho
  `ielts-academic-reading`, assemble 2 lần → 2 mảng `frozenItemIds` (map String) **khác nhau**
  (`expect(setA).not.toEqual(setB)` hoặc so sánh mảng đã sort).
- [x] Thêm test AC4: mọi frozen id thuộc tập seed đúng section (khác skill/cefr/itemType/targetGoal
  không lọt) — seed nhiễu 1 câu sai cefr, assert không xuất hiện.

**Verify:** `npx jest test-paper.service` → xanh.

---

## T2 — `listBlueprints()` service

**File:** `src/modules/test-paper/test-paper.service.js`

- [x] Thêm export `listBlueprints()`: `Object.values(PAPER_BLUEPRINTS)` map ra
  `{ key, exam, skill, variant, totalItems, timeLimitSec, targetBandRange, sections: [{order, cefr, difficultyTier, itemCount}] }`.
  Chỉ đọc hằng, không async cần DB (có thể sync trả mảng).
- [x] Import `PAPER_BLUEPRINTS` từ `./paper-blueprint`.
- [x] Test: `listBlueprints()` trả 4 khung, mỗi khung có `sections.length ≥ 3`, không có field
  `frozenItemIds`/nội dung câu.

**Verify:** `npx jest test-paper.service` → xanh.

---

## T3 — Admin route `GET /blueprints` + `POST /assemble`

**File:** `src/admin-api/resources/test-paper.admin.js` ·
test `src/admin-api/__tests__/test-paper-admin-assemble.test.js` (mới, mirror `test-paper-admin-import.test.js`)

- [x] Import `getBlueprint` từ module test-paper.
- [x] Thêm route **trước** `router.use(testPapers)` (thứ tự load-bearing):
  - `GET /blueprints` — `verifyPermission(PERMISSIONS.TEST_PAPER_READ)` → `res.json({ data: testPaperService.listBlueprints() })`.
  - `POST /assemble` — `verifyPermission(PERMISSIONS.TEST_PAPER_WRITE)`: đọc `{ blueprintKey, title }`;
    `try { paper = await testPaperService.assemble(blueprintKey, { title, createdBy: req.user.id }) }`
    `catch → ApiError(400, err.message, CODES...)`; audit `admin.command.executed` command
    `test-paper.assemble`; trả `{ data: { paperId, requested: getBlueprint(blueprintKey).totalItems, got: paper.frozenItemIds.length } }` status 201.
- [x] Kiểm hằng `PERMISSIONS.TEST_PAPER_READ` tồn tại (dùng cho GET); nếu registry chỉ có READ/WRITE/MANAGE
  thì READ dùng cho GET, WRITE cho POST — xác nhận trong `src/constants/permissions.js`.
- [x] Test AC1: seed bank đủ → POST assemble → 201, `got === requested`, DB có 1 TestPaper draft
  `blueprintKey` đúng, `frozenItemIds.length === totalItems`.
- [x] Test AC5: seed thiếu 1 section → POST → 201 `got < requested`, paper vẫn draft.
- [x] Test AC6: GET blueprints → 200, 4 khung.
- [x] Test AC7: gọi thiếu quyền → 403 (mirror cách import test kiểm permission).
- [x] Test: blueprintKey rác → 400 (getBlueprint throw → map 400).

**Verify:** `npx jest test-paper-admin` → xanh.

---

## T4 — exe-admin service + hooks

**File (exe-admin):** `src/services/test-paper.service.ts`, `src/hooks/use-test-paper.ts`,
`src/constants/query-keys.ts`

- [x] Service `listBlueprints()` → `GET /admin/test-papers/blueprints` trả `Blueprint[]`
  (type: key/exam/skill/variant/totalItems/targetBandRange/sections).
- [x] Service `assemblePaper({ blueprintKey, title })` → `POST /admin/test-papers/assemble` trả
  `{ paperId, requested, got }`.
- [x] Hook `useBlueprints()` (TanStack Query, key mới trong query-keys).
- [x] Hook `useAssemblePaper()` (mutation) — onSuccess `invalidate` key của `usePapers()` để bảng đề refresh.

**Verify:** `npm run build` (exe-admin) → không lỗi type.

---

## T5 — exe-admin UI panel "Tạo đề từ khung"

**File (exe-admin):** `src/components/features/test-paper-import/test-paper-import.tsx`
(+ tách panel con nếu file > 200 dòng: `assemble-panel.tsx`)

- [x] Thêm panel (trong `RequirePermission perm="test-paper:write"`): dropdown blueprint (từ
  `useBlueprints`), ô input tiêu đề, nút "Tạo đề từ khung".
- [x] onClick → `useAssemblePaper().mutate({ blueprintKey, title })`; onSuccess hiện
  "Đã tạo đề (nháp) — got/requested câu", cảnh báo vàng nếu `got < requested`
  ("thiếu N câu trong ngân hàng, chưa publish được"); onError hiện message.
- [x] Bảng đề + Xem + Publish: **không sửa** — draft mới hiện chung, dùng luồng cũ (AC8).
- [x] Nếu file `test-paper-import.tsx` vượt 200 dòng sau khi thêm → tách `assemble-panel.tsx`
  (kebab-case), import vào.

**Verify:** `npm run build` + `npm run lint` (exe-admin) → sạch; smoke: chọn khung → Assemble → draft
xuất hiện → Xem/Publish chạy.

---

## Definition of done

- [x] exe-api: `npx jest test-paper` full xanh (service + admin + practice + import không hồi quy) — AC1–AC7, AC9.
- [x] exe-admin: `npm run build && npm run lint` sạch — AC8.
- [ ] Smoke thủ công trên live server (chưa chạy — cần bật exe-admin + exe-api): admin chọn blueprint →
  tạo draft → assemble lại ra bộ câu khác → publish OK; blueprint thiếu bank → cảnh báo + publish reject.
  (AC2/AC5 đã cover bằng automated test; smoke UI còn lại cho pre-merge review.)

## Câu hỏi mở

- ~~centerId cho center staff~~ — **CHỐT: `centerId=null` (platform-level)**, khớp import P1. Owner xác
  nhận 2026-08-18.
