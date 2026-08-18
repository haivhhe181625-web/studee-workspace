---
feature: luyen-de-blueprint-storage
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Tasks — Blueprint storage + CRUD (Bước 1/3)

Thứ tự: model → seed → wire service → admin CRUD → FE. Test-first ở exe-api. Verify chạy từ
`exe-api/services/api`.

## T1 — Model `PaperBlueprint` + validate

**File:** `src/modules/test-paper/paper-blueprint.model.js` (mới) ·
test `src/modules/test-paper/__tests__/paper-blueprint.model.test.js` (mới)

- [ ] Tạo schema theo `design.md §1` (SectionSchema lồng, field mở rộng optional).
- [ ] Pre-save hook validate: `Σ itemCount === totalItems`; `itemTypes` không rỗng & ⊆ `ITEM_TYPES`
  (import từ assessment-question.model); dựa enum cho cefr/exam/skill/status.
- [ ] Export `PaperBlueprint`.
- [ ] Test: tạo khung hợp lệ OK; itemCount lệch totalItems → reject; itemType lạ → reject; key trùng →
  reject (E11000).

**Verify:** `npx jest paper-blueprint.model` → xanh.

## T2 — Seed 4 khung vào DB

**File:** `src/modules/test-paper/paper-blueprint.js` (thêm `loadBlueprint`, bổ sung `title` cho 4 hằng) ·
`scripts/ops/seed-paper-blueprints.js` (mới)

- [ ] Thêm `title` vào 4 object trong `PAPER_BLUEPRINTS` (hoặc map key→title trong seed).
- [ ] `loadBlueprint(key)` async: `PaperBlueprint.findOne({ key })`; null → `throw new Error('Unknown blueprint: '+key)`.
- [ ] Seed script: upsert 4 khung theo `key` (idempotent), in kết quả, exit. Header ghi rõ "chạy thủ công, không auto-boot".
- [ ] Test (integration): chạy seed 2 lần → `countDocuments === 4` (không nhân bản); `loadBlueprint('ielts-listening')` trả đúng khung.

**Verify:** `npx jest paper-blueprint` → xanh.

## T3 — Wire `assemble` / `publish` / `listBlueprints` đọc DB

**File:** `src/modules/test-paper/test-paper.service.js` · sửa test
`__tests__/test-paper.service.test.js` (seed khung vào DB trước khi assemble)

- [ ] `assemble`: `getBlueprint(key)` → `await loadBlueprint(key)`. Phần `$sample`/freeze **không đổi**.
- [ ] `publish`: `getBlueprint(paper.blueprintKey)` → `await loadBlueprint(...)`.
- [ ] `listBlueprints`: đọc `PaperBlueprint.find({ status: 'active' }).lean()` → map metadata (bỏ centerId/__v).
- [ ] Test cũ (assemble/publish) cập nhật: seed khung DB trong beforeEach/helper thay vì dựa hằng. Giữ
  assertions cũ (full frozenItemIds, dedup, random, publish reject khi thiếu).
- [ ] AC7: `assemble('không-tồn-tại')` → throw; admin route map 400 (đã có try/catch).

**Verify:** `npx jest test-paper` → xanh (assemble/publish/import/practice/admin).

## T4 — Admin CRUD resource

**File:** `src/admin-api/resources/paper-blueprint.admin.js` (mới) · `src/admin-api/_router.js` (mount) ·
test `src/admin-api/__tests__/paper-blueprint-admin.test.js` (mới)

- [ ] `adminResource(...)` theo `design.md §4` (listFields/writeFields/filters/softDelete). Permission tái
  dùng `test-paper:read|write`.
- [ ] Sửa admin assemble route (`test-paper.admin.js`): `getBlueprint(blueprintKey).totalItems` →
  `(await loadBlueprint(blueprintKey)).totalItems`.
- [ ] Mount resource vào `_router.js`.
- [ ] Test: POST tạo khung → 201; GET list → có khung; PATCH sửa itemCount → 200; DELETE → status archived
  (không xoá cứng); tạo khung sai validate → 4xx; thiếu quyền → 403.

**Verify:** `npx jest paper-blueprint-admin` + `npx jest test-paper` → xanh.

## T5 — FE exe-admin: trang quản lý khung

**File (exe-admin):** `src/services/paper-blueprint.service.ts` (mới), `src/hooks/use-paper-blueprint.ts`
(mới), `src/app/(admin)/blueprints/page.tsx` (mới), `src/components/features/blueprints/*` (mới),
`src/constants/query-keys.ts`, `src/constants/nav.ts` (thêm mục)

- [ ] Service: list/get/create/update/archive gọi `/admin/paper-blueprints`.
- [ ] Hooks TanStack Query (key mới), mutation invalidate list.
- [ ] Trang list khung + form editor có repeater section (order/cefr/itemTypes multi-select từ ITEM_TYPES/
  itemCount/difficultyTier/label). Hiện lỗi validate từ API.
- [ ] Gate hiển thị `test-paper:write`. Thêm mục nav.
- [ ] Modularize nếu file > 200 dòng (tách form/section-editor).

**Verify:** exe-admin `npm run build && npm run lint` → sạch; smoke: tạo/sửa/archive khung, assemble khung mới.

## Definition of done

- [ ] exe-api `npx jest test-paper && npx jest paper-blueprint` full xanh — AC1–AC8, AC10.
- [ ] exe-admin build + lint sạch — AC9.
- [ ] Seed đã chạy trên DB dev; assemble dùng khung từ DB.

## Câu hỏi mở

- FE trang-riêng vs generic factory-CRUD (spec/design).
