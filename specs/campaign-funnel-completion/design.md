# Design: Campaign Funnel Completion

- **Ngày:** 2026-08-11
- **Phạm vi design:** chỉ B4 (lead union) cần thiết kế; các task khác additive nên bỏ qua.
- **Spec:** `spec.md` · **Tasks:** `tasks.md`

## Bối cảnh
`spec.md` §5 + `tasks.md` mô tả 8 task additive rõ ràng. Duy nhất **B4** reshape lõi query
`listLeads`/`exportLeads` (hiện `Result.find()` đơn nguồn — `campaign.service.js:306,338`) →
cần thiết kế union + dedup + phân trang/count. Doc này chốt cách làm B4 + ghi lại 3 quyết định
cơ chế đã chốt cho B1/B3/AC7.

## B4 — Lead gồm người đăng-ký-chưa-thi

### Hai nguồn, một trạng thái
| Nguồn | Điều kiện | `status` |
|---|---|---|
| `Result` (đã thi) | `campaignId` khớp, `invalid=false` | `completed` |
| `Result` (đã thi, hỏng) | `campaignId` khớp, `invalid=true` | `invalid` |
| `User` (đăng ký, chưa thi) | `campaignId` khớp **và KHÔNG có Result** trong campaign đó | `registered` |

**Dedup:** ưu tiên Result. Một User vừa có Result vừa là registration → chỉ hiện dòng Result.
Thực hiện bằng cách loại User đã có Result (nhánh `registered` `$lookup` results, giữ `$size:0`).

### Approach: `$unionWith` aggregation + `$facet` (chọn)
Lý do chọn thay vì 2-query-merge-in-memory: phân trang + đếm tổng **trên tập đã dedup** cần đúng
tuyệt đối; `$facet` trả `items` + `total` trong 1 round-trip, tránh count 2 collection rồi lệch.

```
Result.aggregate([
  { $match: <leadResultFilter> },                    // completed + invalid
  { $set: { status: { $cond: ['$invalid','invalid','completed'] } } },
  { $lookup: users → name/email/phoneNumber },       // thay .populate hiện tại
  { $unionWith: { coll: 'users', pipeline: [
      { $match: { campaignId, ...centerScope } },     // registered candidates
      { $lookup: results theo (userId, campaignId) as r },
      { $match: { 'r.0': { $exists: false } } },      // loại người đã có Result (dedup)
      { $set: { status: 'registered', overallCefr: null, invalid: false } },
  ] } },
  { $sort: { createdAt: -1 } },
  { $facet: { items: [ {$skip}, {$limit} ], total: [ {$count:'n'} ] } },
])
```

`exportLeads`: cùng pipeline, bỏ `$facet`/`$skip`, `$limit EXPORT_CAP` (5000, giữ cap hiện có),
thêm cột **Trạng thái** vào `LEAD_COLUMNS`.

### Quy tắc filter (giữ semantics cũ)
- `campaignId` / `centerId` / `from,to` (createdAt): áp cho **cả hai** nhánh (User có `centerId` do
  `lockCampaign` set, có `createdAt`).
- **`bandMin/bandMax`**: chỉ có nghĩa với người đã thi → khi set band, **loại nhánh `registered`**
  (họ không có `overallCefr`). Tức band-filter ⇒ chỉ `completed` khớp band. (tasks: "band chỉ áp completed").
- `status` filter mới (B4/C2): `registered|completed|invalid` — áp sau khi dựng union (thêm
  `$match status` trước `$sort`), hoặc bỏ nhánh tương ứng để rẻ hơn.

### leadRow / DTO
Thêm field `status` vào output row. Row `registered`: `overallCefr=null`, `invalid=false`,
`resultId=null`, giữ `name/email/phoneNumber` từ User (Google cho email; phone có thể trống — chấp nhận).

### Rủi ro
- `$lookup` lồng ở nhánh registered tốn hơn `Result.find()` cũ; chấp nhận (lead list phân trang,
  không hot-path). Đảm bảo index `User.campaignId` (đã có: `user.model.js`) + `Result.{userId,campaignId}`.
- Nếu sau này scale lớn, cân nhắc materialized. **YAGNI hiện tại** — không làm.

## Quyết định cơ chế (chốt cho B1/B3/AC7)
- **B1 (409):** map `E11000` tại `_crud.factory.js` chỗ `model.create` (`:209`) → `ApiError(409, 'Mã đã tồn tại')`.
  Dùng chung mọi resource unique, không nhét business-logic vào `admin-api/` (đúng CLAUDE.md exe-api).
- **B3 (unique click):** cookie `rc_<code>` TTL theo đợt; `/r/:code` chưa có cookie → set; có cookie ⇒
  bỏ `$inc uniqueClicks`. `cookie-parser` đã mount (`app.js:78`). Bỏ bot theo User-Agent trước khi đếm.
- **AC7 (hết đợt) ở exe-web:** redirect `/` kèm toast/banner "chiến dịch đã kết thúc". Không tạo route mới.
- **Timezone:** so `now` với `startAt/endAt` bằng UTC (Date native, Mongo lưu UTC).

## Không đổi
Logic chấm/lắp đề M3; `getCampaignStats` (registrations vẫn đếm `User.countDocuments{campaignId}`);
attribution chuỗi (`attributeCampaign` → `resolveActiveByCode` → `lockCampaign`).
