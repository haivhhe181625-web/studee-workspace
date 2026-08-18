---
feature: luyen-de-blueprint-storage
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Design — PaperBlueprint storage + CRUD + wiring

## 1. Schema `PaperBlueprint` (collection `paper_blueprints`)

Giữ **đúng tên field** như hằng hiện tại để `assemble`/`validatePaper` chỉ đổi nguồn đọc, không đổi logic.
Field mở rộng (đón đầu Bước 2/3) để **optional, không enforce** ở Bước 1.

```js
const SectionSchema = new Schema({
  order: { type: Number, required: true },
  label: { type: String, default: null },            // 'Part 7 Reading Comprehension' — hiển thị
  cefr: { type: String, enum: CEFR_LEVELS, required: true },        // HARD filter (assemble)
  itemTypes: { type: [String], required: true },     // HARD filter; validate ⊆ ITEM_TYPES
  itemCount: { type: Number, required: true },
  difficultyTier: { type: String, enum: ['easy','mid','hard'], required: true }, // MÔ TẢ (Bước 1)
  // Mở rộng — optional, chưa dùng trong assemble ở Bước 1 (đón TOEIC Part 7 / metadata):
  stimulusGrouping: { type: String, enum: ['passage','part','image','passage_pair','passage_triple'], default: null },
}, { _id: false });

const PaperBlueprintSchema = new Schema({
  key: { type: String, required: true, unique: true, index: true }, // natural ref (TestPaper.blueprintKey)
  title: { type: String, required: true },
  exam: { type: String, enum: ['ielts','toeic'], required: true },
  skill: { type: String, enum: ['reading','listening'], required: true },
  variant: { type: String, enum: ['academic','general', null], default: null }, // 'general' đón IELTS GT
  totalItems: { type: Number, required: true },
  timeLimitSec: { type: Number, required: true },
  targetBandRange: { type: String, default: null },
  status: { type: String, enum: ['active','archived'], default: 'active' },
  sections: { type: [SectionSchema], required: true },
  centerId: { type: Schema.Types.ObjectId, ref: 'Center', default: null }, // platform-level
  createdBy: { type: Schema.Types.ObjectId, ref: 'User', default: null },
}, { timestamps: true, collection: 'paper_blueprints' });
```

**Validate (pre-save hook + reuse ở service):**
- `sections.reduce(sum itemCount) === totalItems` — else throw.
- mọi `sections[].itemTypes` không rỗng và ⊆ `ITEM_TYPES` (import từ assessment-question.model).
- `cefr ∈ CEFR_LEVELS` (enum lo).
- `key` unique (index lo; bắt lỗi 11000 → 400 ở admin).

**Vì sao string `key` chứ không ObjectId:** `TestPaper.blueprintKey` đang là string; giữ `key` làm ref
→ TestPaper + đề đã tạo **không phải migrate**. Archive khung không phá đề cũ (đề chỉ cần đọc lại khung
để re-validate lúc publish; đề đã published không cần khung nữa).

## 2. Wiring — đổi nguồn khung từ hằng sang DB

| Hàm | Trước | Sau |
|---|---|---|
| `getBlueprint(key)` (sync, đọc hằng) | dùng ở assemble/publish | thay bằng `loadBlueprint(key)` **async** đọc `PaperBlueprint.findOne({ key })`; không thấy → throw như cũ |
| `assemble` | `const bp = getBlueprint(key)` | `const bp = await loadBlueprint(key)` — phần còn lại (aggregate `$sample`, freeze) **giữ nguyên** |
| `publish` | `getBlueprint(paper.blueprintKey)` | `await loadBlueprint(...)` — validate giữ nguyên |
| `listBlueprints` | map hằng | `PaperBlueprint.find({ status: 'active' }).lean()` rồi map metadata (bỏ field nội bộ) |
| `validatePaper(bp, items)` | pure, nhận bp object | **không đổi** — vẫn nhận bp + items |

`paper-blueprint.js`: giữ `PAPER_BLUEPRINTS` (làm dữ liệu seed) + `validatePaper`. Thêm `loadBlueprint`.
Bỏ `getBlueprint` sau khi hết call site (hoặc giữ như wrapper đọc hằng cho test thuần — quyết định lúc code).

**Admin assemble route** (`test-paper.admin.js`, Bước trước) gọi `getBlueprint(blueprintKey).totalItems`
để trả `requested` → đổi sang `await loadBlueprint(...)`. Surgical, 1 chỗ.

## 3. Seed migration

`scripts/ops/seed-paper-blueprints.js`:
- Đọc `PAPER_BLUEPRINTS` từ hằng, `upsert` theo `key` (`updateOne({key}, {$set:...}, {upsert:true})`).
- Idempotent — chạy nhiều lần không nhân bản.
- Bổ sung `title` (chưa có trong hằng — suy từ key hoặc hardcode 4 title). `label` section: null (điền sau
  qua admin nếu muốn). `stimulusGrouping`: null.
- In số khung đã seed. Không đụng đề/câu.

## 4. Admin CRUD (`paper-blueprint.admin.js`, factory-based)

Mirror `test-paper.admin.js` factory config:
```js
adminResource({
  model: PaperBlueprint, resource: 'paper-blueprint',
  listFields: ['key','title','exam','skill','variant','totalItems','status','createdAt'],
  searchFields: ['key','title'],
  filters: ['exam','skill','status'],
  writeFields: ['key','title','exam','skill','variant','totalItems','timeLimitSec','targetBandRange','sections','status'],
  createDefaults: (req) => ({ createdBy: req.user.id }),
  softDelete: { field: 'status', value: 'archived' },
});
```
- `sections` là mảng lồng → factory whitelist cả cụm; **validate nằm ở model pre-save hook** (factory gọi
  `.save()` nên hook chạy). Lỗi validate → factory/error handler map ra 4xx.
- Permission: `paper-blueprint:*`? Hay tái dùng `test-paper:*`? → **Tái dùng `test-paper:*`** (khung thuộc
  cùng domain đề, tránh đẻ permission mới cho 1 resource nhỏ). GET=read, write/archive=write.
- Mount vào `_router.js` cạnh test-papers.

## 5. FE exe-admin

`sections` là mảng lồng → generic `/r/[resource]` form khó sửa từng section. **Đề xuất trang riêng**
`/blueprints`: bảng list + form editor có repeater section (order/cefr/itemTypes multi-select/itemCount/
difficultyTier/label). Service + hooks mirror `test-paper.service.ts`/`use-test-paper.ts`. (Chi tiết bước
FE trong `tasks.md`; xác nhận trang-riêng vs generic khi review — câu hỏi mở ở spec.)

## 6. Rủi ro & giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| `getBlueprint` sync → async đổi nhiều call site | Chỉ 3 chỗ (assemble, publish, admin assemble route) + listBlueprints; đều đã trong hàm async. |
| Seed ghi đè khung admin đã sửa tay | Seed chỉ chạy thủ công (ops script), không auto ở boot; upsert theo key — nếu lo, đổi thành insert-if-absent. Ghi rõ trong script header. |
| Factory không validate mảng lồng | Validate ở model pre-save hook (chạy khi factory `.save()`), không phụ thuộc factory. |
| Đề cũ tham chiếu khung đã archive | `loadBlueprint` đọc bất kể status (chỉ `listBlueprints` lọc active); publish vẫn re-validate được. |

## Câu hỏi mở

- FE trang-riêng vs generic factory (spec §Câu hỏi mở).
- Giữ hay bỏ `getBlueprint` sync sau migration (ảnh hưởng test thuần đọc hằng) — quyết định lúc code.
