---
feature: grammar-content-v2
status: draft
owner: BA
created: 2026-08-22
repos: [exe-api, exe-web]
supersedes-data: grammar-content (v1 flat)
---

# Spec — grammar-content v2 (nested tree + 8 block type mới)

## Mục tiêu
Đưa **10 file `*.v2.json` (schemaVersion 2)** đã sinh & QC sạch (validate.py 0 lỗi, `Dữ liệu/…/Dữ liệu mới/`) vào sản phẩm: nạp DB dạng **cây phân cấp (nested)**, phục vụ qua REST, render trên FE với **accordion điều hướng `children`** + **8 block type tương tác mới**.

## Vì sao
Module grammar-content v1 đã ship nhưng là **FLAT** (1 mảng `learn[]` phẳng, 13 block type, 1 doc/bài). Dữ liệu v2 là **cây** (55 smallTopic, mỗi bài có `children` sâu tới 3 tầng, 21 block type) — thiết kế để bài dài (vd đại từ 104 block) gập theo section, học không ngợp. Cần nâng model/service/FE/importer để phục vụ đúng cấu trúc này.

## Quyết định đã chốt (2026-08-22)
1. **NESTED** — DB lưu nguyên cây (`summary`, `objectives`, `children` đệ quy). KHÔNG flatten.
2. **Nguồn importer**: copy 10 file `.v2.json` vào repo `exe-api/services/api/scripts/grammar-content/output-v2/` (versioned).
3. **Validate nông**: `block-registry` chỉ check field top-level + kiểu (như hiện tại). Không lặp lại check sâu của validate.py (data đã pass validate.py lúc sinh — DRY).
4. **Thay thế hoàn toàn v1**: importer v2 wipe collection trước khi nạp (`--fresh`) → không lẫn doc v1 mồ côi.

## Phạm vi (in-scope)
1. **BE (`exe-api`)**:
   - `block-registry.js` — thêm **8 type**: comparison, tip, flashcard, matching, categorize, sort-order, quick-check, recap (validate nông).
   - `grammar-content.model.js` — thêm `summary`, `objectives`, `children` (đệ quy Mixed); `learn` giữ Mixed.
   - `grammar-content.service.js` — `sanitizeLearn` **đệ quy** cho mọi node trong cây; `toDto` trả cây nested.
   - Importer v2 mới — walk 10 file `.v2.json` → 55 doc (1 doc/smallTopic gốc), upsert theo `grammarPoint`, `--fresh` / `--dry-run` / `--publish`.
2. **FE (`exe-web`)** — PR riêng:
   - **8 component block mới** + đăng ký vào BlockRenderer (tái dùng 16 component flat sẵn có).
   - **AccordionTree** điều hướng `children` (gập/mở section); render `learn` mỗi node qua BlockRenderer.

## Ngoài phạm vi (out-of-scope)
- `cast[]` (character bible sinh ví dụ) và `mediumTopic` → authoring metadata, **KHÔNG lưu DB**.
- Type `pronounce` (nút nghe TTS, spec v2 §6) — **HOÃN**, tạm dùng `table`.
- CEFR per-lesson — data v2 chưa có; `cefrLevel` giữ null, thêm sau nếu cần lọc.
- SRS / progress / tracking học — thuộc plan cha, không đụng.
- Sinh/biên tập nội dung — pipeline offline đã xong.
- Admin đổi status — **đã có sẵn từ v1** (status là field top-level, hoạt động nguyên với doc nested); không code mới.

## Repo/surface ảnh hưởng
- `exe-api/…/modules/grammar-content/{block-registry,grammar-content.model,grammar-content.service}.js` (+ `__tests__/*`).
- `exe-api/…/scripts/import-grammar-content-v2.js` (mới) + `output-v2/*.v2.json` (copy).
- `exe-web/src/components/features/grammar/blocks/*` (8 comp mới) + `AccordionTree` + BlockRenderer (PR riêng).
- Không đụng `exe-admin`.

## Acceptance Criteria (checkable)
- [ ] `block-registry` nhận đủ **21 type** (13 cũ + 8 mới); block hợp lệ pass, thiếu field/type lạ → loại + log, **không throw**.
- [ ] Model tạo được doc nested (có `summary`, `objectives`, `children`); trùng `grammarPoint` bị chặn; status ngoài enum reject.
- [ ] `sanitizeLearn`/`toDto` áp **đệ quy**: block nội bộ (`_source`,`_note`) bị strip ở **mọi tầng** children, không chỉ node gốc.
- [ ] `GET /api/grammar-content/:grammarPoint` trả cây nested (`children` + `learn` mỗi node); **404** nếu thiếu/chưa active.
- [ ] Importer `--dry-run` in **55** bài; chạy thật (`--fresh`) → `countDocuments()===55`, chạy lại 2 lần vẫn 55 (không nhân đôi); `--publish` set `status:active`.
- [ ] Toàn bộ `.v2.json` nạp không mất block: tổng block DB == tổng block validate.py báo (**4.995 block / 631 node / 55 bài**).
- [ ] Test BE xanh (`npm test` module grammar-content + importer); không lỗi lint/syntax.
- [ ] FE (PR riêng) render đủ 8 block mới đúng hành vi: matching bấm cặp · categorize chọn bucket · sort-order ghép thứ tự · flashcard lật · quick-check Đ/S+why · comparison cột→tab mobile · tip callout · recap chốt. Accordion gập/mở `children`.

## Câu hỏi mở
- Không còn — 4 quyết định trên đã chốt.
