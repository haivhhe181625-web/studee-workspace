---
feature: luyen-de-paper-assemble
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
epic: PRD-151 (M7 Luyện đề)
---

# Spec — Nối `assemble` ra admin + random thật (M7 Luyện đề)

## 1. Mục tiêu & lý do

`assemble(blueprintKey)` (test-paper.service.js) đã code + test: query ngân hàng câu hỏi theo từng
section của blueprint (skill/cefr/itemTypes/targetGoal/active) rồi freeze `frozenItemIds` theo thứ tự
section → tạo TestPaper `draft`. **Nhưng nó chưa nối ra route/admin/UI nào** — chỉ tồn tại trong test,
content team không dùng được.

Ngoài ra `assemble` **chưa random thật**: dùng `.limit(itemCount)` không có sort ngẫu nhiên → lấy N câu
*đầu tiên* theo thứ tự index. Assemble lại cùng blueprint luôn ra cùng bộ câu → không dùng để tạo nhiều
biến thể đề từ 1 ngân hàng.

Feature này: (1) đổi `assemble` sang **random thật** bằng `$sample`, (2) **expose ra admin** (REST +
UI trong exe-admin) để tạo đề `draft` từ khung → tái dùng luồng preview/publish sẵn có.

## 2. Phạm vi

- **BE (exe-api):**
  - `assemble` chuyển từ `find().limit()` sang aggregation `$sample` (random mỗi section, vẫn dedup
    cross-section qua `$nin`). Xem `design.md`.
  - Service mới `listBlueprints()` — trả metadata các blueprint (đọc hằng `PAPER_BLUEPRINTS`, không side
    effect) cho dropdown UI.
  - Admin REST (`test-paper.admin.js`, hand-mounted, delegate service — không nhét business logic):
    - `GET  /api/admin/test-papers/blueprints` → list khung (perm `test-paper:read`).
    - `POST /api/admin/test-papers/assemble` `{ blueprintKey, title }` → tạo draft, trả
      `{ paperId, requested, got }` để UI cảnh báo thiếu bank (perm `test-paper:write`, audit log).
- **FE (exe-admin):** mở rộng trang `test-paper-import` sẵn có thêm panel "Tạo đề từ khung" (dropdown
  blueprint + ô tiêu đề + nút Assemble). Tạo xong → refresh bảng đề (draft xuất hiện) → **dùng lại**
  nút Xem (preview) + Publish đã có. Không làm trang mới.

## 3. Ngoài phạm vi

- **Center scoping / visibility** — giữ platform-level (`centerId=null`), khớp luồng import P1. Gác lại.
- **Sửa/hoán câu trong đề đã assemble** (M7-15b) — feature khác.
- **Cấu hình blueprint động qua UI/DB** — blueprint vẫn hard-coded trong `paper-blueprint.js`. Chỉ
  *chọn* khung có sẵn, không *tạo/sửa* khung.
- **Weighted/stratified sampling** (theo độ khó tinh hơn trong 1 section) — `$sample` đều là đủ cho P1.
- **Tự publish** — assemble luôn ra `draft`, publish vẫn thủ công qua gate sẵn có.
- **Chặn tạo đề thiếu bank** — vẫn cho tạo draft thiếu câu; publish sẵn có tự reject. Chỉ *report* số
  thiếu, không chặn.

## 4. Acceptance Criteria

- [ ] AC1 — `POST /assemble` với blueprint hợp lệ + bank đủ câu → tạo 1 TestPaper `status='draft'`,
  `frozenItemIds` đúng `totalItems`, `blueprintKey` set đúng; response `{ paperId, requested, got }` với
  `got === requested`.
- [ ] AC2 — Random thật: assemble **2 lần** cùng blueprint trên bank dư (≥ 2× số cần) → hai bộ
  `frozenItemIds` **khác nhau** (không phải luôn N câu đầu cố định).
- [ ] AC3 — Dedup cross-section giữ nguyên: với blueprint có 2 section trùng cefr+itemType
  (toeic-listening S2/S3), `frozenItemIds` **không trùng** id nào, đủ `totalItems`.
- [ ] AC4 — Mọi id trong `frozenItemIds` là câu `status='active'` khớp (skill, cefr, itemType hợp lệ,
  targetGoal) của section tương ứng — assemble không kéo câu ngoài tiêu chí.
- [ ] AC5 — Bank thiếu (1 section không đủ câu) → assemble vẫn tạo draft ngắn, response `got < requested`;
  UI hiện cảnh báo thiếu; publish đề này vẫn reject như hành vi sẵn có (draft giữ nguyên).
- [ ] AC6 — `GET /blueprints` trả đủ 4 khung với metadata (key, exam, skill, variant, totalItems,
  targetBandRange, tóm tắt sections); không lộ `frozenItemIds`/nội dung câu.
- [ ] AC7 — Phân quyền: `POST /assemble` yêu cầu `test-paper:write`, `GET /blueprints` yêu cầu
  `test-paper:read`; thiếu quyền → 403.
- [ ] AC8 — FE: chọn khung + nhập tiêu đề + Assemble → đề draft xuất hiện trong bảng, Xem/Publish chạy
  đúng (tái dùng luồng cũ, không hồi quy import/publish).
- [ ] AC9 — Full suite exe-api xanh; 0 regression engine assessment / test-paper (import, practice,
  publish). Test cũ của `assemble` (đủ bank → full set, dedup) vẫn pass sau khi đổi sang `$sample`.

## 5. Repos ảnh hưởng

- **exe-api** — `test-paper.service.js` (`assemble` → `$sample`, thêm `listBlueprints`),
  `admin-api/resources/test-paper.admin.js` (2 route mới), test.
- **exe-admin** — `services/test-paper.service.ts`, `hooks/use-test-paper.ts`,
  `components/features/test-paper-import/*` (thêm panel assemble).

## 6. Liên kết

- Design + quyết định sampling: `design.md` · Tasks: `tasks.md`.
- Feature nền: `specs/luyen-de-exam-practice/` (assemble/publish gốc),
  `specs/luyen-de-paper-import/` (luồng import + admin UI + preview/publish tái dùng ở đây).
- Epic Plane: PRD-151 (M7 Luyện đề).

## Câu hỏi mở

- ~~centerId cho center staff~~ — **CHỐT: `centerId=null` (platform-level)** khớp import P1 (owner
  xác nhận 2026-08-18). Đề theo center để phase sau nếu cần.
