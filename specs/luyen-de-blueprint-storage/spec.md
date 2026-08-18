---
feature: luyen-de-blueprint-storage
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
epic: PRD-151 (M7 Luyện đề)
step: 1/3 (blueprint storage → item generation → difficulty assembly)
---

# Spec — Chuyển khung đề (blueprint) từ hằng code sang DB + CRUD (Bước 1/3)

## 1. Mục tiêu & lý do

Hiện 4 khung đề nằm cứng trong `paper-blueprint.js` (hằng `Object.freeze`), không có tầng lưu trữ:
thêm/sửa khung = dev sửa code + deploy. Content team không tự định nghĩa/điều chỉnh khung được.

Bước 1 chuyển khung thành **dữ liệu trong MongoDB** (`PaperBlueprint` collection) + **admin CRUD**, và
đổi `assemble`/`publish`/`listBlueprints` đọc khung từ DB thay vì hằng. Nền cho Bước 2 (sinh dữ liệu item)
và Bước 3 (bật độ khó thành filter). Xem chuỗi ở `step` frontmatter.

## 2. Phạm vi

- **BE (exe-api):**
  - Model mới `paper-blueprint.model.js` (`PaperBlueprint`, collection `paper_blueprints`) — schema ở
    `design.md`. Validate: `Σ sections.itemCount === totalItems`, `itemTypes ⊆ ITEM_TYPES`, `cefr ∈ CEFR_LEVELS`,
    `key` unique.
  - Seed script `scripts/ops/seed-paper-blueprints.js` — upsert 4 khung hiện có (từ hằng cũ) vào DB. Idempotent.
  - `paper-blueprint.js` giữ lại **chỉ** làm nguồn seed + `validatePaper` (hàm pure, không đổi); thêm
    `loadBlueprint(key)` async đọc DB. `getBlueprint` (sync, đọc hằng) → thay bằng `loadBlueprint` ở call site.
  - Đổi `test-paper.service.js`: `assemble`, `publish`, `listBlueprints` đọc khung từ DB (async). Logic
    assemble/validate **không đổi** — chỉ đổi nguồn khung.
  - Admin CRUD qua `_crud.factory.js`: `paper-blueprint.admin.js` (list/detail/create/patch/archive),
    writeFields whitelist, `status` soft-archive, `test-paper:*` permission. Mount vào `_router.js`.
- **FE (exe-admin):** trang quản lý khung (list + form tạo/sửa section). Tái dùng pattern factory-CRUD FE
  đang có (như các resource `/r/[resource]`) nếu phù hợp; hoặc trang riêng `blueprints`. Chi tiết `tasks.md`.

## 3. Ngoài phạm vi

- **Sinh/nhập câu hỏi vào ngân hàng** — Bước 2 (`luyen-de-item-bank-generation`).
- **Tier/IRT thành filter thật trong assemble** — Bước 3 (`luyen-de-difficulty-assembly`). Bước 1 giữ
  `difficultyTier` ở mức **section, mô tả** (assemble vẫn lọc bằng cefr + itemTypes như hiện tại).
- **Center scoping** — khung `centerId=null` (platform), khớp toàn feature. Gác lại.
- **Đổi `TestPaper.blueprintKey`** — giữ nguyên là string `key` (natural ref) → TestPaper không phải đổi.
- **Đổi `AssessmentQuestion`** — không đụng item bank ở Bước 1.

## 4. Acceptance Criteria

- [ ] AC1 — Seed script chạy → 4 khung (`ielts-academic-reading`, `ielts-listening`, `toeic-listening`,
  `toeic-reading`) có mặt trong `paper_blueprints`, đúng field như hằng cũ; chạy lại lần 2 không tạo trùng
  (idempotent upsert theo `key`).
- [ ] AC2 — `assemble(key)` đọc khung từ DB → tạo đề đúng như trước (cùng totalItems/sections), test cũ
  của assemble/publish vẫn xanh.
- [ ] AC3 — Model reject khi tạo khung sai: `Σ itemCount ≠ totalItems`, `itemTypes` chứa type ngoài
  `ITEM_TYPES`, `cefr` ngoài enum, `key` trùng → lỗi validate rõ ràng, không ghi.
- [ ] AC4 — Admin CRUD: tạo khung mới qua API → xuất hiện trong `listBlueprints` → `assemble` khung mới
  chạy được (nếu bank đủ câu).
- [ ] AC5 — Sửa khung (PATCH section itemCount/itemTypes/cefr) → assemble lần sau dùng cấu hình mới.
- [ ] AC6 — Archive khung (`status='archived'`) → không hiện trong dropdown assemble (list mặc định chỉ
  `active`); khung đã archive không xoá cứng (đề cũ tham chiếu `blueprintKey` vẫn đọc được).
- [ ] AC7 — `loadBlueprint('key-không-tồn-tại')` → lỗi rõ ràng (assemble/publish map 400 như hiện tại).
- [ ] AC8 — Phân quyền: đọc khung `test-paper:read`, ghi/sửa/archive `test-paper:write`; thiếu quyền 403.
- [ ] AC9 — FE: admin xem danh sách khung, tạo/sửa được section, thấy lỗi validate từ API.
- [ ] AC10 — Full suite exe-api xanh, 0 regression test-paper (assemble/publish/import/practice);
  exe-admin build + lint sạch.

## 5. Repos ảnh hưởng

- **exe-api** — model + seed + service wiring + admin resource.
- **exe-admin** — trang quản lý khung.

## 6. Liên kết

- Research nền: `plans/reports/research-260818-ielts-toeic-paper-structure-for-blueprint-model.md`
  (cấu trúc IELTS/TOEIC, cách mô hình độ khó, field cần/tránh).
- Design (schema + wiring): `design.md` · Tasks: `tasks.md`.
- Bước trước (đã làm): `specs/luyen-de-paper-assemble/` (assemble + admin, đọc khung từ hằng — Bước 1 đổi
  nguồn sang DB).
- Bước sau: `specs/luyen-de-item-bank-generation/` (Bước 2), `specs/luyen-de-difficulty-assembly/` (Bước 3).

## Câu hỏi mở

- FE: dùng generic factory-CRUD (`/r/[resource]`) sẵn có cho khung, hay trang riêng với editor section
  chuyên biệt? Design đề xuất trang riêng (section là mảng lồng, form generic khó sửa) — xác nhận khi review.
