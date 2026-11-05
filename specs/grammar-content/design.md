---
feature: grammar-content
status: draft
owner: TechLead
created: 2026-08-19
---

# Design — grammar-content

## Tổng quan
Module BE mới theo **5-file pattern** (`.model/.schema/.service/.controller/.routes`) dưới `exe-api/services/api/src/modules/grammar-content/`. Lưu nguyên block-lesson JSON (như output pipeline), phục vụ read-only cho FE, admin đổi status. FE port renderer từ `render-review.js`.

## Data model — `GrammarLessonContent` (1 doc / grammarPoint)
```
{
  grammarPoint:   String, unique, index, required   // slug parser, vd "simple_present_tense"
  grammarCategory: String, index                    // "noun" | "verb" | ... (10 loại)
  categoryVi:     String                             // "Danh Từ" ...
  cefrLevel:      String | null
  title:          String
  learn:          [Block]                            // mảng block (xem contract dưới)
  status:         String enum ['draft','active'], default 'draft', index
  timestamps
}
```
> **Chốt (2026-08-19):** KHÔNG lưu `reference` (HTML thô) — khối `detail` đã chứa đủ mục gốc; giảm payload. Import bỏ qua field `reference` trong output.
- Mongoose 8. `learn` để **Mixed/loose** (block đa hình) — validate bằng `block-registry`, KHÔNG ép Mongoose sub-schema cứng (giữ open/closed cho block mới).

## Block contract (hợp đồng BE↔FE)
Mỗi block: `{ type, renderTier: 'flow'|'framed', _source?, ...payload }`. `_source`/`_note` bị strip ở `toDto` public.

| type | payload chính |
|---|---|
| concept | term, def |
| formula | groups[{ label, items[{ pattern, exampleEn, exampleVi }] }] |
| table | caption, head[], rows[[]] |
| formula-fill | sentence, answer, options[3] |
| usage | items[{ use, character, en, vi, mark }] |
| examples | items[{ useNote, character, en, vi, mark }] |
| signal-words | words[], clauseSignals[], meanings[{ word, vi, freqNote, exampleEn, exampleMark }] |
| rules | groups[{ title, items[] }] |
| minicheck | items[{ q, answer, options[3], explainWrong[{ option, reason }] }] |
| contrast | wrong, right, note |
| detail | sections[{ heading, html }] |

`block-registry.js`: map `type → { validate(payload), publicFields }`. **Fail-soft**: block sai/type lạ → loại khỏi output + log, KHÔNG throw. (Nguồn sự thật đối chiếu: `scripts/grammar-content/block-ai-output.schema.json` + renderer.)

## API (public + admin)
Public (`grammar-content.routes.js`, mount `/api/grammar-content`):
- `GET /:grammarPoint` → 1 bài `active`, qua `toDto` (strip `_source/_note/reference?`), 404 nếu thiếu/không active.
- `GET /?category=<slug>` → list active (title, grammarPoint, cefrLevel), có thể phân trang.

Admin (`src/admin-api/resources/grammar-content.admin.js`, `_crud.factory.js`):
- list/get mọi bài (kể cả draft), `writeFields` = chỉ `status`; permission `grammar-content:manage` (thêm vào `constants/permissions.js`).
- **KHÔNG** business logic ở admin-api — đổi status gọi `grammar-content.service.js` (HARD RULES admin-api).

## Import
`scripts/import-grammar-content.js`: đọc `scripts/grammar-content/output/*.blocks.json` → `upsert` theo `grammarPoint` (idempotent). `--dry-run` in số bài/loại. Chạy 1 lần sau khi content active.

## FE (`exe-web`) — PR riêng
- `components/grammar/blocks/<Type>Block.tsx` + `BlockRenderer` (registry `type → component`), fail-soft cho type lạ.
- Tương tác port từ `render-review.js`: formula toggle ví dụ · signal reveal · minicheck chấm · detail accordion.
- Study-space fetch `GET /api/grammar-content/:grammarPoint` → render `learn[]`.
- Style theo prototype artifact (flow/framed, tô màu S/V/O, không emoji).

## Quyết định / lưu ý
- Tách concern: BE chỉ lưu+phục vụ; **không** parse/sinh (đó là script offline).
- `learn` loose + registry validate = open/closed cho block mới (thêm type không phải migrate schema).
- Cross-repo: 1 spec, 2 PR (exe-api trước → có API; exe-web sau).

## Quyết định đã chốt (2026-08-19)
1. `reference` **KHÔNG lưu** DB/API (detail accordion đã đủ). Model bỏ field; import bỏ qua.
2. Liên kết course lesson → grammar-content: **để sau**, không chặn feature này.
3. Bảng dài (bất quy tắc 85 dòng): **FE render tự gập** (accordion) khi > ~15 dòng — quyết ở component table (T7).
