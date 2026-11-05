---
feature: grammar-content-v2
status: draft
owner: TechLead
created: 2026-08-22
---

# Tasks — grammar-content v2

> TDD-checkbox. Mỗi task 1 PR-worth + lệnh verify. BE (`exe-api`) trước → API xanh → FE (`exe-web`) PR riêng.
> Nguồn data: `Dữ liệu/Dữ Liệu của Ngữ pháp/Dữ liệu mới/*/**.v2.json` (10 file, validate.py 0 lỗi).
> Chạy test Windows: `cd exe-api/services/api && npm test -- <pattern>`.

## BE — exe-api (làm trước)

- [ ] **T0. Copy dữ liệu v2 vào repo**
  Tạo `exe-api/services/api/scripts/grammar-content/output-v2/`, copy 10 file `.v2.json` (đổi tên đồng nhất: `<category-slug>.v2.json` — vd `clause.v2.json` thay `menhde.v2.json`, `future-tense.v2.json` thay `futuretense.v2.json`).
  Verify: `ls output-v2/ | wc -l` = 10; mỗi file `JSON.parse` OK.

- [ ] **T1. block-registry — thêm 8 type (validate nông)**
  `block-registry.js`: thêm vào `REGISTRY` 8 entry (comparison/tip/flashcard/matching/categorize/sort-order/quick-check/recap) theo bảng design (required top-level + kind). Giữ `isValidBlock`/`sanitizeLearn`/`toPublicBlock` nguyên. Cập nhật `__tests__/block-registry.test.js`.
  Verify: `npm test -- block-registry` — 8 type hợp lệ pass; thiếu field bị loại; type lạ → không throw; `KNOWN_TYPES.length===21`.

- [ ] **T2. Model — thêm summary/objectives/children**
  `grammar-content.model.js`: thêm `summary:String`, `objectives:[String]`, `children:[Schema.Types.Mixed] default []`. Giữ `grammarPoint` unique index, `status` enum+index, `learn` Mixed. Cập nhật `__tests__/grammar-content.model.test.js`.
  Verify: `npm test -- grammar-content.model` — tạo doc nested (có children) OK; trùng grammarPoint bị chặn; status ngoài enum reject.

- [ ] **T3. Service — sanitize đệ quy + toDto nested**
  `grammar-content.service.js`: thêm `sanitizeTree(node)` (sanitizeLearn + đệ quy children); `toDto` trả thêm `summary/objectives/children`. Getter/list giữ nguyên. Cập nhật `__tests__/grammar-content.service.test.js`.
  Verify: `npm test -- grammar-content.service` — `_source/_note` bị strip ở **node con** (không chỉ gốc); block invalid trong child bị loại; draft không ra public getter.

- [ ] **T4. Route/controller (regression)**
  Không đổi hợp đồng; chỉ đảm bảo `GET /:grammarPoint` trả cây nested. Chạy lại test route hiện có.
  Verify: `npm test -- grammar-content.routes` xanh; response có `children` + không lộ `_source`.

- [ ] **T5. Importer v2**
  `scripts/import-grammar-content-v2.js`: đọc `output-v2/*.v2.json`, mỗi smallTopic gốc → 1 doc (grammarPoint, grammarCategory/categoryVi từ file, summary, objectives, learn, children, order=catOrder*100+idx). Flags: `--dry-run`, `--fresh` (deleteMany trước nạp), `--publish` (status active). Upsert theo grammarPoint. Viết `__tests__/import-grammar-content-v2.test.js` (memory mongo).
  Verify: `--dry-run` in **55**; `--fresh` thật → `countDocuments()===55`; chạy 2 lần vẫn 55; tổng block nạp (đếm đệ quy learn) == **4.995**; `--publish` → mọi doc `status:active`.

- [ ] **T6. Chạy import thật + smoke API**
  Chạy `node scripts/import-grammar-content-v2.js --fresh --publish` trên DB dev.
  Verify: `GET /api/grammar-content/noun-basics` → 200, có `summary/objectives/children`; `GET /api/grammar-content?category=noun` list bài noun; bài chưa publish (nếu có) → 404.

## FE — exe-web (PR riêng, sau khi API xanh)

- [ ] **T7. 8 component block mới + đăng ký BlockRenderer**
  `blocks/{Comparison,Tip,Flashcard,Matching,Categorize,SortOrder,QuickCheck,Recap}Block.tsx` + đăng ký `type→component`. Port style/tương tác từ `*demo.html`. Fail-soft type lạ.
  Verify: test render mỗi type; matching bấm cặp đúng/sai · categorize chọn bucket · sort-order ghép thứ tự (check hoán vị) · flashcard lật · quick-check Đ/S+why · comparison cột→tab mobile · tip callout `variant` · recap chốt + nextHint.

- [ ] **T8. AccordionTree + tích hợp lesson**
  `AccordionTree` duyệt `children` đệ quy (section gập/mở), body render `node.learn` qua BlockRenderer. `GrammarLessonContainer` fetch nested → render learn gốc + AccordionTree. Bài dài gập theo section.
  Verify: mở `noun-basics` (và bài đại từ ~104 block) — hiển thị đúng demo, gập/mở mượt, theme sáng/tối, responsive; 404 xử lý gọn.

## Human gates
- **Design sign-off**: design.md đã chốt 4 quyết định (2026-08-22) → chờ duyệt trước khi bắt đầu T1.
- **Pre-merge review**: duyệt diff + acceptance trước khi merge (BE PR, rồi FE PR). Không tự merge.

## Sau feature (ngoài spec này)
- `pronounce` block (TTS), CEFR per-lesson filter, SRS/progress, tách hiển thị complex-sentence khỏi clause.
- Dọn: importer v1 `import-grammar-content.js` + `.blocks.json` obsolete — xoá khi chắc không còn dùng.
