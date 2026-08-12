# Design: Campaign QR & Lead Management

- **Spec:** `specs/campaign-qr-lead-management/spec.md`
- **Trạng thái:** Nháp — chờ Technical Review
- **Nguyên tắc:** tái dùng tối đa `Center`/`Assessment`/`Result`; thêm tối thiểu (1 entity + `campaignId`). Không đụng logic chấm/lắp đề M3.

## 1. Kiến trúc tổng thể

```
QR in ra  ──►  GET /r/:code  ──($inc clicks)──►  redirect  /assessment/intake?campaign=<code>
                                                              │
exe-web: capture campaign code (giống center-code.ts) ───────┘
   │  (chưa đăng nhập) → ép Đăng ký (Google) → khóa User.centerId + campaignId (first-touch)
   ▼
POST /assessments/intake  → createFromIntake gắn Assessment.campaignId = user.campaignId
   ▼  (làm bài → chấm — M3 nguyên trạng)
Result (valid) 
   ▲
exe-admin dashboard  ◄── GET /api/admin/campaigns, /leads, /stats, /report, /qr, /export
```

3 service Node giữ nguyên; feature nằm ở `services/api`. `services/cat`/`llm` không đụng.

## 2. Data model

### 2.1 Campaign — MỚI (`modules/campaign/campaign.model.js`, collection `campaigns`)

| Field | Kiểu | Ràng buộc |
|---|---|---|
| `centerId` | ObjectId → Center | required, index |
| `name` | string | required, trim |
| `code` | string | required, **unique**, lowercase, slug hóa (`[a-z0-9-]`) |
| `status` | enum `active\|paused\|archived` | default `active` |
| `startAt` / `endAt` | Date | optional (đợt) |
| `metrics` | `{ clicks: Number default 0 }` | counter cache; chỉ click lưu ở đây |
| `createdBy` | ObjectId → User | audit |
| timestamps | | |

Index: `{ centerId: 1, status: 1 }`, `{ code: 1 } unique`.

### 2.2 Field thêm vào bảng có sẵn (additive)

| Bảng | Field thêm | Ghi chú |
|---|---|---|
| `User` | `campaignId: ObjectId → Campaign, default null` | **immutable** — chỉ set khi đang null (đăng ký). `centerId` đã có. |
| `Assessment` | `campaignId: ObjectId → Campaign, index, default null` | đóng dấu theo lượt thi. `centerId` đã có. |
| `Result` | `campaignId: ObjectId → Campaign, index, default null` | tùy chọn — để lọc/aggregate nhanh; nếu không thêm thì join qua Assessment. |

> **Lưu ý strict schema:** app chạy `strict + strictQuery` → path không khai báo bị drop khỏi update & query filter. Bắt buộc khai báo các field trên trong schema.

### 2.3 Quan hệ
Center 1:N Campaign · Campaign 1:N User (lead) · Campaign 1:N Assessment · User 1:N Assessment · Assessment 1:1 Result · Center bao trùm (tenant, đã có). ERD: artifact đính kèm phiên làm việc.

## 3. Attribution & khóa account (first-touch)

1. `exe-web` bắt `?campaign=<code>` (mở rộng `center-code.ts` hoặc thêm `campaign-code.ts` tương tự), lưu localStorage, sống sót login redirect.
2. Đăng ký (auth): resolve `code` → `Campaign` → set `User.campaignId` + `User.centerId = campaign.centerId` **chỉ khi đang null** (guard immutable). Đăng nhập lại không đổi.
3. `createFromIntake` (`assessment.service.js`): thêm `campaignId: user.campaignId` vào `Assessment.create(...)`. Giữ nguyên precedence center hiện có (đã khớp vì account đã khóa).

## 4. Tính metrics (DRY — chỉ click là lưu mới)

| Metric | Cách tính |
|---|---|
| **clicks** | `Campaign.metrics.clicks` — `$inc` tại `GET /r/:code` |
| **đăng ký** | `countDocuments(User, { campaignId })` |
| **làm bài** | `countDocuments(Assessment, { campaignId, status ∈ started/submitted })` |
| **hoàn thành** | `countDocuments(Result, { campaignId, invalid: false })` (hoặc join Assessment) |
| **tỉ lệ chuyển đổi** | đăng ký/click · làm bài/đăng ký · hoàn thành/làm bài |
| **phân bố band** | `aggregate(Result group by overallCefr)` theo campaign |
| **lọc thời gian/điểm** | filter `createdAt` / `overallCefr` trong aggregate |

Roll-up theo trường = cùng aggregate nhưng group theo `centerId`, gộp mọi campaign.

## 5. API endpoints

### 5.1 Admin (`src/admin-api/` — resource + custom actions, delegate service)
| Method | Path | Việc |
|---|---|---|
| POST | `/api/admin/campaigns` | tạo (name, centerId, code, startAt/endAt) |
| GET | `/api/admin/campaigns` | list + filter center/status |
| GET | `/api/admin/campaigns/:id` | chi tiết + metrics |
| PATCH | `/api/admin/campaigns/:id` | sửa status/đợt |
| GET | `/api/admin/campaigns/:id/qr?format=png\|svg` | tải QR |
| GET | `/api/admin/campaigns/:id/stats?from&to` | funnel + band dist |
| GET | `/api/admin/centers/:id/report?from&to` | roll-up theo trường |
| GET | `/api/admin/leads?campaignId&centerId&from&to&bandMin&bandMax&page` | dashboard lead |
| GET | `/api/admin/leads/export?<same filters>&format=xlsx\|csv` | export |

Quyền: `campaign:read` / `campaign:manage`, `lead:read` / `lead:export` trong `constants/permissions.js`. Tenant-scope từ `User.centerId` (platform admin thấy tất cả).

### 5.2 Client
| Method | Path | Auth | Việc |
|---|---|---|---|
| GET | `/r/:code` | public + rate-limit | `$inc clicks` → 302 redirect landing test kèm `?campaign=<code>` |
| POST | `/auth/...` (register) | — | nhận + khóa `campaignCode` |
| POST | `/assessments/intake` | JWT | (đã có) gắn `campaignId` |

## 6. Cấu trúc module (5-file pattern)

```
modules/campaign/
├── campaign.model.js       # schema Campaign
├── campaign.schema.js      # Joi validate create/update/query
├── campaign.service.js     # tạo/sửa, sinh QR, aggregate metrics, export, roll-up
├── campaign.controller.js  # client controller (/r/:code redirect)
└── campaign.routes.js      # public route /r/:code (+ rate-limit)
```
Admin: `src/admin-api/resources/campaign.admin.js` (CRUD factory) + custom actions (qr/stats/report/leads/export) gọi `campaign.service`. Không để business logic trong admin-api.

## 7. Tech stack / thư viện
- **QR:** `qrcode` (npm). PNG `QRCode.toBuffer(url,{type:'png'})`, SVG `QRCode.toString(url,{type:'svg'})`. Sinh on-demand, không lưu ảnh.
- **Export:** `exceljs` cho `.xlsx`; CSV stream (exceljs csv hoặc `json2csv`). Set `Content-Disposition`.
- **FE:** `exe-admin` (Next.js) dashboard + nút tải QR/export; `exe-web` capture code + landing.

## 8. Bảo mật & lưu ý
- `/r/:code` public → rate-limit (Redis, đã có `generalLimiter`) chống spam click; validate `code` tồn tại + `status=active`.
- `User.campaignId`/`centerId` immutable: guard set-once ở service, không nhận từ `req` để đổi.
- Export chứa PII (email/điểm) → chỉ role có `lead:export`; đi qua audit hook admin.
- Không expose `campaignId` nội bộ ra client test ngoài mục đích đếm.

## 9. Không làm (chốt lại)
- Không guest; không đổi campaign sau khóa; không A/B/UTM; không sửa chấm/lắp đề/ngân hàng M3.
