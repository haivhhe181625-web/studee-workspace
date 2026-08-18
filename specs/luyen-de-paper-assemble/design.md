---
feature: luyen-de-paper-assemble
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Design — Random assemble + admin surface

## 1. Random sampling: `.limit()` → `$sample`

### Hiện tại (deterministic)
```js
const docs = await AssessmentQuestion.find({
  skill: bp.skill, cefrLevel: section.cefr,
  itemType: { $in: section.itemTypes }, targetGoal: bp.exam,
  status: 'active', _id: { $nin: frozenItemIds },
}).limit(section.itemCount).select('_id').lean();
```
`.limit()` không có sort → MongoDB trả N doc **đầu tiên theo natural order** → assemble lại cùng blueprint
ra cùng bộ câu.

### Đổi thành (random)
```js
const docs = await AssessmentQuestion.aggregate([
  { $match: {
      skill: bp.skill, cefrLevel: section.cefr,
      itemType: { $in: section.itemTypes }, targetGoal: bp.exam,
      status: 'active', _id: { $nin: frozenItemIds },
  } },
  { $sample: { size: section.itemCount } },
  { $project: { _id: 1 } },
]);
for (const d of docs) frozenItemIds.push(d._id);
```

**Vì sao `$sample` chọn được:**
- Random đều, native MongoDB, không kéo cả collection về app tự shuffle.
- Trả `min(size, số doc khớp match)` — hụt bank thì trả ít hơn (giống `.limit`), không lỗi.
- `$match` giữ nguyên **mọi tiêu chí cũ** kể cả `_id: { $nin: frozenItemIds }` → **dedup cross-section
  vẫn đúng** vì các section chạy tuần tự, `frozenItemIds` tích luỹ dần (blueprint có section trùng
  cefr+itemType như toeic-listening S2/S3 vẫn không lấy trùng câu).

**Lưu ý kỹ thuật:**
- Thứ tự câu *trong 1 section* thành ngẫu nhiên — không sao: `validatePaper` chỉ kiểm theo **slice
  section** (mọi câu trong slice phải thuộc `itemTypes` cho phép), không phụ thuộc thứ tự nội bộ. Thứ tự
  **giữa các section** vẫn đúng vì push tuần tự theo `bp.sections`.
- `$nin` với mảng ObjectId cỡ 40–100 phần tử: chi phí không đáng kể ở quy mô này.
- `$sample` cần đọc từ collection, không dùng được `.select()` của query builder → dùng `$project`.
- `aggregate` trả plain object (không phải mongoose doc) — chỉ lấy `_id`, khớp cách dùng hiện tại (đang
  `.lean()`).

### Không làm (YAGNI)
Stratified/weighted sampling theo độ khó mịn hơn trong 1 section: blueprint đã cố định cefr + itemTypes
cho mỗi section; `$sample` đều trong tập đã lọc là đủ cho P1.

## 2. Report thiếu bank — không đổi contract `assemble`

`assemble` **giữ nguyên return = TestPaper doc** (test cũ + chữ ký hàm không đổi, surgical). Admin route
tự suy số thiếu từ blueprint + `frozenItemIds.length`:

```js
const paper = await testPaperService.assemble(blueprintKey, { title, createdBy: req.user.id });
const bp = getBlueprint(blueprintKey);
res.status(201).json({ data: {
  paperId: paper._id,
  requested: bp.totalItems,
  got: paper.frozenItemIds.length,
} });
```

`got < requested` → UI cảnh báo "thiếu N câu trong ngân hàng, không publish được". Không cần per-section
breakdown ở P1: publish (đã có) khi reject sẽ in lỗi chi tiết theo section. Nếu sau này cần chi tiết hơn,
mở rộng return của `assemble` sau — không làm sớm.

**Không chặn tạo draft thiếu câu:** giữ hành vi hiện tại (draft ngắn → publish reject). Chặn sớm là speculative.

## 3. Admin surface (delegate-only, đúng HARD RULE admin-api)

`test-paper.admin.js` — thêm 2 route hand-mounted **trước** factory router (thứ tự load-bearing: factory
`GET /:id` không được nuốt `/blueprints`, `/assemble`), mirror cách `/import` + `/:id/preview` đã làm:

| Method | Path | Perm | Delegate |
|---|---|---|---|
| GET | `/api/admin/test-papers/blueprints` | `test-paper:read` | `testPaperService.listBlueprints()` |
| POST | `/api/admin/test-papers/assemble` | `test-paper:write` | `testPaperService.assemble(key, {title, createdBy})` + audit |

`listBlueprints()` (service mới): trả `Object.values(PAPER_BLUEPRINTS)` rút gọn
(`key, exam, skill, variant, totalItems, timeLimitSec, targetBandRange, sections:[{order,cefr,difficultyTier,itemCount}]`).
Chỉ đọc hằng, không side effect — đặt ở service để admin-api không nhét logic.

`assemble` route: bọc `try/catch` map lỗi input (blueprint không tồn tại → `getBlueprint` throw) sang 400,
giống pattern `/import`. Audit `admin.command.executed` command `test-paper.assemble` target paperId.

`centerId`: giữ `null` (assemble mặc định `centerId=null`). Center-scoping ngoài phạm vi (câu hỏi mở trong spec).

## 4. FE exe-admin — mở rộng trang import, tái dùng preview/publish

Không tạo trang mới. Trong `test-paper-import.tsx` thêm panel "Tạo đề từ khung":

- `useBlueprints()` (GET) → dropdown; `useAssemblePaper()` (POST) → onSuccess refresh `usePapers()` (bảng
  đề sẵn có), hiện `got/requested` (cảnh báo nếu thiếu).
- Service `test-paper.service.ts`: `listBlueprints()`, `assemblePaper({blueprintKey, title})`.
- Bảng đề + nút Xem (preview) + Publish: **không đổi** — draft do assemble tạo hiện chung bảng, dùng
  đúng luồng vet/publish của import.
- Gate hiển thị `test-paper:write` (giống panel import); API mới là lớp enforce thật.

## 5. Rủi ro & giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| Test randomness flaky | Không assert phân phối; AC2 chỉ assert 2 assemble trên bank dư (≥2×) cho ra 2 bộ khác nhau — xác suất trùng ~0. Test "đủ bank → full set" + "dedup" giữ deterministic. |
| `$sample` đổi thứ tự phá validate | Đã phân tích §1: validate theo slice section, không phụ thuộc thứ tự nội bộ. |
| Route mới bị factory `GET /:id` nuốt | Mount trước factory (đã có tiền lệ `/import`, `/:id/preview`). |
| Admin-api chứa logic | `listBlueprints`/`assemble` nằm ở service; route chỉ gọi + map lỗi + audit. |

## Câu hỏi mở

- centerId cho center staff (xem spec §6) — chờ owner.
