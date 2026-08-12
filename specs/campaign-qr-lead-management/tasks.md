# Tasks: Campaign QR & Lead Management

> **Cho Developer Agent:** implement theo thứ tự Task 1 → 10, mỗi task commit riêng, TDD nơi codebase đã có test.
> **Design:** `specs/campaign-qr-lead-management/design.md`
> **Lưu ý chung:** lệnh `npm`/`jest` chạy trong `exe-api/services/api`. FE (exe-admin/exe-web) là PR repo riêng (Task 9).

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/campaign/campaign.model.js` | Schema Campaign | Tạo |
| `src/modules/campaign/campaign.schema.js` | Joi validate | Tạo |
| `src/modules/campaign/campaign.service.js` | Business: CRUD, QR, metrics, export, roll-up | Tạo |
| `src/modules/campaign/campaign.controller.js` | `/r/:code` redirect | Tạo |
| `src/modules/campaign/campaign.routes.js` | Public route + rate-limit | Tạo |
| `src/modules/assessment/assessment.model.js` | Thêm `campaignId` | Sửa |
| `src/modules/assessment/assessment.service.js` | Gắn `campaignId` khi intake | Sửa |
| `src/modules/user/*` + auth register | `User.campaignId` + khóa first-touch | Sửa |
| `src/admin-api/resources/campaign.admin.js` | CRUD + actions (qr/stats/report/leads/export) | Tạo |
| `src/constants/permissions.js` | `campaign:*`, `lead:*` | Sửa |

---

## Task 1: Model + schema Campaign

- [ ] **Step 1: Test FAIL** — tạo `src/__tests__/campaign.model.test.js`: tạo Campaign hợp lệ; `code` unique (trùng → lỗi); `code` slug hóa lowercase; `status` default `active`; `centerId` required.
- [ ] **Step 2: Run** `npm test -- campaign.model` → FAIL (chưa có model).
- [ ] **Step 3: Implement** `campaign.model.js` theo design §2.1 + `campaign.schema.js` (Joi create/update/query).
- [ ] **Step 4: Run** `npm test -- campaign.model` → PASS.
- [ ] **Step 5: Commit** `feat(campaign): add campaign model and validation schema`

## Task 2: Thêm `campaignId` (additive) + guard immutable trên User

- [ ] **Step 1: Test FAIL** — test: `Assessment` lưu/đọc `campaignId`; `User.campaignId` set được khi null nhưng **không đổi** khi đã có (guard).
- [ ] **Step 2: Run** test → FAIL.
- [ ] **Step 3: Implement** khai báo `campaignId` trong `assessment.model.js` (+ `result.model.js` nếu chọn thêm) và `User` schema; viết helper set-once `lockCampaign(user, campaignId, centerId)` trong user service.
- [ ] **Step 4: Run** test → PASS.
- [ ] **Step 5: Commit** `feat(campaign): add immutable campaignId attribution fields`

## Task 3: Service — CRUD + metrics aggregate + roll-up

- [ ] **Step 1: Test FAIL** — integration: seed campaign + vài User/Assessment/Result → `getCampaignStats` trả đúng click/đăng ký/làm bài/hoàn thành + phân bố band; `getCenterReport` gộp nhiều campaign.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** `campaign.service.js`: `createCampaign/updateCampaign/listCampaigns`, `incrementClick(code)`, `getCampaignStats({id,from,to})` (aggregate User/Assessment/Result theo `campaignId`), `getCenterReport({centerId,...})`.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(campaign): campaign service with funnel metrics and center rollup`

## Task 4: Sinh QR (PNG/SVG)

- [ ] **Step 1: Test FAIL** — test `generateQr(url,'png')` trả Buffer PNG hợp lệ; `'svg'` trả string `<svg`.
- [ ] **Step 2: Run** → FAIL. (Cài `qrcode`: `npm i qrcode`.)
- [ ] **Step 3: Implement** `generateQr` trong service dùng `qrcode`; URL build từ `code` (dạng `/r/<code>`, xem OQ).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(campaign): generate downloadable qr (png/svg)`

## Task 5: Public route `/r/:code` (đếm click + redirect)

- [ ] **Step 1: Test FAIL** — integration: GET `/r/<code>` → 302 tới landing kèm `?campaign=<code>`, `metrics.clicks` +1; code không tồn tại/không active → 404/redirect trang mặc định; có rate-limit.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** `campaign.controller.js` + `campaign.routes.js` (public, `generalLimiter`), mount ngoài router assessment (không JWT).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(campaign): public redirect route with click tracking`

## Task 6: Attribution — khóa lúc đăng ký + đóng dấu lượt thi

- [ ] **Step 1: Test FAIL** — integration: đăng ký kèm `campaignCode` → `User.centerId`+`campaignId` set; đăng ký lần 2 code khác → **không đổi**; `createFromIntake` → `Assessment.campaignId = user.campaignId`.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** wiring trong auth/register (gọi `lockCampaign`) + thêm `campaignId: user.campaignId` vào `Assessment.create` trong `createFromIntake`. Giữ precedence center hiện có.
- [ ] **Step 4: Run** `npm test -- assessment` + test auth → PASS.
- [ ] **Step 5: Commit** `feat(campaign): lock account to campaign on register and stamp attempts`

## Task 7: Admin resource + endpoints + permissions

- [ ] **Step 1: Implement** `campaign.admin.js` (CRUD factory: listFields/filters center+status/writeFields whitelist); custom actions delegate service: `qr`, `stats`, `report`, `leads` (list + filter campaign/campus/time/band). Thêm `campaign:read|manage`, `lead:read|export` vào `permissions.js` + bundle center-staff.
- [ ] **Step 2: Verify** — test admin/ hoặc smoke: tạo campaign, list có filter, stats trả số; tenant-scope đúng (center-staff chỉ thấy campaign của mình).

Run: `npm test -- admin` → PASS (hoặc smoke OK).
- [ ] **Step 3: Commit** `feat(campaign): admin campaign crud, stats, leads and permissions`

## Task 8: Export Excel/CSV

- [ ] **Step 1: Test FAIL** — test `exportLeads(filters,'xlsx')` trả Buffer workbook có header đúng; `'csv'` trả CSV; áp đúng bộ lọc.
- [ ] **Step 2: Run** → FAIL. (Cài `exceljs`.)
- [ ] **Step 3: Implement** `exportLeads` trong service + admin action set `Content-Disposition`; chỉ role `lead:export`.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(campaign): export leads to xlsx and csv`

## Task 9: FE (repo riêng — không trong exe-api)

> PR riêng ở `exe-admin` + `exe-web`. Liệt kê để không sót; không TDD trong workspace này.

- [ ] `exe-web`: capture `?campaign=<code>` (mở rộng `center-code.ts` → thêm `campaign-code.ts`); truyền `campaignCode` vào đăng ký; landing `/r/<code>` (nếu render ở web).
- [ ] `exe-admin`: trang danh sách/tạo campaign; nút tải QR (png/svg); dashboard funnel + filter; bảng lead + nút export; báo cáo roll-up theo trường.

## Task 10: Full suite + docs

- [ ] **Step 1: Run** `npm test -- campaign assessment admin` → ALL PASS.
- [ ] **Step 2: Docs** — cập nhật `docs` M3/assessment ghi thêm `campaignId` + funnel; ghi endpoint mới.

---

## Self-Review

**Acceptance coverage:**
- AC-1 (CRUD campaign) → Task 1,7. · AC-2 (QR png/svg) → Task 4,7. · AC-3 (click+redirect) → Task 5.
- AC-4 (khóa first-touch) → Task 2,6. · AC-5 (stamp Assessment) → Task 6. · AC-6 (dashboard filter) → Task 7.
- AC-7 (funnel + band) → Task 3,7. · AC-8 (export) → Task 8. · AC-9 (roll-up) → Task 3,7.
- AC-10 (admin không business logic + tenant) → Task 7. · AC-11 (không đụng M3 chấm/lắp đề) → toàn bộ giữ surgical.

**Placeholder scan:** chốt OQ URL (`/r/<code>` vs `?campaign`) trước Task 4/5.

**Type/name consistency:** `campaignId`, `code`, `lockCampaign`, `getCampaignStats`, `getCenterReport` dùng nhất quán giữa model ↔ service ↔ admin ↔ FE.
