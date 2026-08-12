# Spec: Campaign QR & Lead Management (tuyển sinh qua QR)

- **Ngày:** 2026-08-10
- **Tác giả:** AI Brainstorm (chốt scope với dev)
- **Repos/surfaces ảnh hưởng:** `exe-api` (module mới `campaign` + admin-api resource + sửa nhỏ `assessment`/`auth`), `exe-admin` (dashboard chiến dịch + tải QR + export), `exe-web` (bắt `campaign` code lúc đăng ký + landing `/r/:code`).
- **Module liên quan:** `exe-api/services/api/src/modules/campaign/` (mới), `.../modules/assessment/` (thêm `campaignId`), `.../modules/user` + auth (khóa campaign lúc đăng ký).
- **Trạng thái:** Đã duyệt (BA + design) — sẵn sàng Task Generation/Code
- **ERD:** xem `design.md` §2.

## 1. Mục tiêu

Cho Admin/Marketing tạo chiến dịch tuyển sinh gắn với một campus, sinh QR/URL để in, và thu — theo dõi tân sinh viên làm bài test đầu vào qua từng chiến dịch (đếm click → đăng ký → làm bài → hoàn thành), có dashboard + export và báo cáo tổng quát theo trường.

## 2. Bối cảnh

Đã kiểm code (không đoán):

**Đã có, tái dùng:**
- `Center` (campus/tenant): slug, branding, `notifyEmail`, `classMapping`, policy — `center.model.js`.
- Bắt code từ URL: `captureCenterCode()` đọc `?center=<slug>` lưu localStorage, sống sót qua login redirect — `exe-web/src/lib/center-code.ts`.
- Gán center vào lượt thi: `createFromIntake` giải `intake.centerCode` → `centerId`, lưu trên `Assessment` (và `Result`) — `assessment.service.js:83-106`.
- Test bắt buộc đăng nhập (JWT) — `assessment.routes.js:33-35`; ownership theo `userId` từ JWT.
- Admin CRUD factory + tenant-scope + audit — `src/admin-api/` (`results.admin.js`, `assessment.admin.js`).

**Chưa có (phần feature này):**
- Entity `Campaign` (nhiều chiến dịch/đợt trên 1 campus).
- Sinh QR + URL builder + tải PNG/SVG.
- Đếm funnel per-link (click/đăng ký/làm bài/hoàn thành) + dashboard lead + filter (campus/thời gian/điểm) + export Excel/CSV + roll-up theo trường.
- Khóa account↔campus khi đăng ký.

## 3. Phạm vi

### Trong phạm vi
- Model `Campaign` (thuộc 1 `Center`; `code` unique; `status`; `startAt/endAt`; `metrics.clicks`).
- Bắt buộc **đăng ký** (bỏ chế độ khách): khi đăng ký qua QR, set `User.centerId` + `User.campaignId` **một lần, immutable** (first-touch).
- Thêm `campaignId` vào `Assessment` (+ `Result`) — đóng dấu theo lượt thi; sửa nhỏ `createFromIntake`.
- Endpoint redirect `/r/:code` (public): `$inc clicks` → forward vào landing test kèm `?campaign=<code>`.
- Admin: CRUD campaign · tải QR (PNG/SVG) · dashboard lead (filter campaign/campus/thời gian/khoảng điểm) · export Excel/CSV · báo cáo roll-up theo trường.
- Metrics tối thiểu: click, đăng ký, làm bài, hoàn thành + tỉ lệ chuyển đổi + phân bố band.

### Ngoài phạm vi
- **Chế độ khách/ẩn danh** — **Lý do:** đã quyết bỏ; chỉ đăng ký/đăng nhập.
- **Đổi campaign của account sau khi khóa** (mở rộng đa-campaign/user) — **Lý do:** "tính mở rộng sau nếu cần"; giờ khóa cứng first-touch.
- **A/B testing, UTM đa kênh, pixel quảng cáo** — **Lý do:** YAGNI giai đoạn này.
- **Thay đổi logic chấm điểm / lắp đề / ngân hàng M3** — **Lý do:** feature chỉ đọc output M3 + thêm 1 field attribution.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Admin/Marketing | Tạo chiến dịch gắn campus + đợt, tải QR đi in | Phát tán tuyển sinh theo từng campus/đợt |
| Admin/Marketing | Xem dashboard lead + funnel + lọc + export | Đánh giá hiệu quả từng chiến dịch |
| Quản lý trường | Xem báo cáo tổng quát của campus mình | Nhìn toàn cảnh nhiều chiến dịch |
| Tân sinh viên | Quét QR → đăng ký → làm test luôn | Biết trình độ, để lại thông tin |
| Hệ thống | Gắn đúng campaign vào hồ sơ + đếm funnel | Truy vết nguồn lead chính xác |

## 5. Quyết định nghiệp vụ (đã chốt)

| Câu hỏi | Đã chốt |
|---|---|
| Khách hay bắt đăng ký | **Chỉ đăng ký** (mở rộng sau) |
| Attribution | **First-touch, khóa cứng account↔campus** khi đăng ký |
| Campaign entity | **Có entity riêng** (1 center N campaign, theo đợt) |
| Báo cáo | Per-campaign **và** roll-up tổng quát theo từng trường |

## 6. Acceptance criteria (tóm tắt)

1. Admin tạo được `Campaign` gắn 1 `Center`, `code` unique; sửa `status`/đợt; liệt kê + lọc theo center/status.
2. Admin tải được QR của campaign ở **PNG và SVG**; QR trỏ tới URL chứa `?campaign=<code>` (hoặc `/r/<code>`).
3. Truy cập `/r/:code` tăng `clicks` của đúng campaign rồi redirect vào landing test kèm code.
4. Đăng ký khi có campaign code → `User.centerId` + `User.campaignId` được set **một lần**; lần đăng ký/đăng nhập sau **không đổi** được (immutable).
5. Lượt thi tạo ra mang đúng `Assessment.campaignId` (kế thừa từ account đã khóa).
6. Dashboard lead hiển thị danh sách tân sinh viên; lọc theo **campaign, campus, khoảng thời gian, khoảng điểm**.
7. 4 chỉ số funnel đúng theo từng campaign: **click, đăng ký, làm bài, hoàn thành** + tỉ lệ chuyển đổi + phân bố band.
8. Export danh sách lead ra **Excel (.xlsx) và CSV** theo bộ lọc hiện tại.
9. Báo cáo **roll-up theo trường**: tổng hợp mọi campaign của 1 center.
10. Admin-api chỉ chứa config/CRUD; mọi side-effect (sinh QR, đếm, export, khóa) nằm ở service module (CLAUDE.md §Admin Rules). Tenant-scope theo `User.centerId`, không tin `req`.
11. Không thay đổi logic chấm điểm/lắp đề/ngân hàng M3.

## 7. Câu hỏi mở — ĐÃ CHỐT (2026-08-10)

- [x] **URL:** dùng path `/r/<code>` → redirect vào landing test kèm `?campaign=<code>`; đếm click tách bạch tại endpoint này.
- [x] **Click:** **raw count** giai đoạn đầu; unique (cookie/ip-hash) để mở rộng sau.
- [x] **`code`:** admin tự đặt; hệ thống validate **unique + slug hóa** (`[a-z0-9-]`, lowercase).
- [x] **Quyền:** `campaign:read|manage`, `lead:read|export`. Bundle center-staff = `campaign:manage` + `lead:read` + `lead:export` (scope theo `User.centerId` của họ); platform admin = tất cả.

## 8. Ghi chú cho Tech Lead Design

- **Attribution precedence:** hiện `own-center thắng intake.centerCode` (`assessment.service.js:86`). Vì account khóa campus lúc đăng ký, lượt thi kế thừa đúng — chỉ cần thêm `Assessment.campaignId = user.campaignId`. Cân nhắc nguồn `campaignId` khi user chưa có (đăng ký ngay trước intake).
- **Immutable field:** `User.campaignId`/`centerId` chỉ set khi còn null (guard chống ghi đè) — không cho đổi qua API.
- **DRY metrics:** chỉ **click** cần lưu mới (`Campaign.metrics.clicks` hoặc event log); đăng ký/làm bài/hoàn thành/điểm/thời gian **suy ra bằng aggregate** trên User/Assessment/Result theo `campaignId` — không nhân bản dữ liệu.
- **Public route `/r/:code`:** nằm ngoài router assessment (đang bắt buộc JWT); cần rate-limit chống spam click.
- **Cross-repo:** FE dashboard/QR/export ở `exe-admin`, capture code khi đăng ký ở `exe-web` — mỗi repo PR riêng (project-context §0). Spec này là nguồn chung.
- **Tech:** QR `qrcode` (npm, PNG+SVG); export `exceljs` (+CSV). Không để logic trong admin-api.
