---
feature: grammar-content-v2
status: draft
owner: TechLead
created: 2026-08-22
---

# Design — grammar-content v2 (nested)

## Tổng quan
Nâng module FLAT v1 (đã ship) lên phục vụ **cây phân cấp**. Giữ nguyên 5-file pattern, 16 component FE flat, và cơ chế fail-soft registry. Thay đổi cốt lõi: model thêm 3 field nested, service duyệt đệ quy, importer mới đọc `.v2.json`, FE thêm 1 lớp accordion + 8 block.

## Cấu trúc dữ liệu nguồn (`*.v2.json`)
```
file  = { schemaVersion:2, mediumTopic, grammarCategory, categoryVi, cast[], smallTopics[] }
smallTopic (node gốc) = { id, grammarPoint, title, summary, objectives[], learn[], children[] }
child node            = { id, title, level?, learn[], children[] }   // KHÔNG có grammarPoint/summary/objectives
block                = { id, type(1/21), renderTier:'flow'|'framed', ...payload }
```
- 1 file/category (10 category). 55 smallTopic gốc, `grammarPoint` **unique toàn bộ** (đã verify). 576 child, sâu tối đa 3 tầng. `cast`/`mediumTopic` = metadata, KHÔNG lưu.

## Data model — `GrammarLessonContent` (1 doc / smallTopic gốc)
Giữ field v1, **thêm 3 field** (đổi tối thiểu):
```
{
  grammarPoint:    String, unique, index, required   // slug smallTopic gốc, vd "noun-basics"
  grammarCategory: String, index                     // "noun"|...|"tense"
  categoryVi:      String
  cefrLevel:       String|null                        // giữ null (v2 chưa có)
  title:           String, required
  summary:         String                             // MỚI — tóm tắt bài
  objectives:      [String]                           // MỚI — mục tiêu học
  learn:           [Mixed]                            // block của node gốc (như v1)
  children:        [Mixed]                            // MỚI — cây con đệ quy {id,title,level?,learn,children}
  status:          String enum['draft','active'] default 'draft', index
  order:           Number default 9999, index
  timestamps
}
```
- `children` để `[Mixed]` (đệ quy, không ép sub-schema) — cùng lý do `learn` loose ở v1: thêm/sửa cấu trúc node không cần migrate schema.
- Key vẫn là `grammarPoint` → tương thích ngược getter/route hiện có.

## Block registry — thêm 8 type (validate nông)
Thêm vào `REGISTRY` (required top-level + kind), `KNOWN_TYPES` tự cập nhật:
| type | required (nông) |
|---|---|
| comparison | `columns`: array |
| tip | `html`: string |
| flashcard | `cards`: array |
| matching | `pairs`: array |
| categorize | `buckets`: array, `items`: array |
| sort-order | `tokens`: array, `answer`: string |
| quick-check | `items`: array |
| recap | `points`: array |
> Check sâu (mark∈en, sort-order ghép được, answer∈options…) **KHÔNG** làm lại ở JS — validate.py đã gác cổng lúc sinh data. Registry chỉ chặn block rác/type lạ để không sập renderer.

## Service — đệ quy
```
sanitizeTree(node):
  node.learn = sanitizeLearn(node.learn)          // strip _source/_note + loại block invalid
  node.children = (node.children||[]).map(sanitizeTree)
toDto(doc): { grammarPoint, grammarCategory, categoryVi, cefrLevel, title,
              summary, objectives, status,
              learn: sanitizeLearn(doc.learn),
              children: (doc.children||[]).map(sanitizeTree) }
```
- Getter/list giữ nguyên (`getActiveLesson` by grammarPoint, `listActiveByCategory` compact — không trả learn/children). Chỉ `toDto` mở rộng.

## API — không đổi hợp đồng route
- `GET /api/grammar-content/:grammarPoint` → doc active, `toDto` nested (nay có `summary/objectives/children`). 404 nếu thiếu/không active.
- `GET /api/grammar-content?category=<slug>` → list compact (giữ nguyên).
- Admin status toggle: **đã có**, doc nested không ảnh hưởng (status top-level).

## Importer v2 — `scripts/import-grammar-content-v2.js`
```
input : scripts/grammar-content/output-v2/*.v2.json   (copy 10 file vào repo)
map   : với mỗi file → mỗi smallTopic gốc → 1 doc {
          grammarPoint, grammarCategory(file), categoryVi(file), title,
          summary, objectives, learn, children, order, status }
order : theo thứ tự category cố định * 100 + index smallTopic trong file
write : --fresh → deleteMany({}) trước khi nạp (thay thế hoàn toàn v1);
        upsert theo grammarPoint (idempotent); --dry-run in count; --publish → status:active
```
- Không dùng `getCanonicalOrder()` (parse HTML v1) — grammarPoint v2 khác; order tính từ category order + index.
- Importer v1 (`import-grammar-content.js`, đọc `.blocks.json`) **obsolete** sau khi v2 chạy — để lại, đánh dấu obsolete (không xoá, tránh đụng test v1 ngoài scope).

## FE (`exe-web`) — PR riêng
- **8 component block mới** `blocks/<Type>Block.tsx` (state trong component, không cần backend):
  matching (bấm trái→phải) · categorize (chọn bucket) · sort-order (bấm/kéo, check hoán vị) · flashcard (lật) · quick-check (Đ/S + why) · comparison (2-3 cột → tab mobile) · tip (callout theo `variant`) · recap (list chốt + `nextHint`).
- **AccordionTree**: duyệt `children` đệ quy, mỗi node = 1 section gập; body render `node.learn` qua **BlockRenderer sẵn có** (tái dùng 16 comp flat + 8 comp mới). Bài dài (104 block) gập theo section.
- Port CSS/tương tác tham chiếu từ `Dữ liệu mới/<Category>/*demo.html` (đã có renderer 21 type + accordion).
- Container `GrammarLessonContainer` fetch nested → render `learn` gốc + `<AccordionTree>` cho children.

## Rủi ro / lưu ý
- **complex-sentence có `grammarCategory:"clause"`** (trùng file clause) — hợp lệ (validate.py cho phép), nhưng list theo category sẽ gộp 2 file vào "clause". Nếu muốn tách hiển thị, dùng `mediumTopic` làm nhãn nhóm FE (không cần đổi DB). → xác nhận khi làm FE index.
- `children` `[Mixed]` → Mongoose không validate sâu; dựa validate.py + registry đệ quy. Chấp nhận (DRY).
- Đổi model thêm field không phá doc v1 cũ, nhưng `--fresh` xoá sạch → chạy import v2 là điểm cắt v1→v2.

## Quyết định đã chốt (2026-08-22)
NESTED · copy file vào repo · validate nông · `--fresh` thay thế v1. `cast`/`mediumTopic`/`pronounce`/CEFR ngoài phạm vi.
