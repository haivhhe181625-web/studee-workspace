<!-- BA spec (LITE) — mục tiêu, phạm vi, acceptance. Chốt giá/gói do BA+PO. -->
# Spec — Màn quản lý Packages (bảng giá gói) trong Admin

- **Ngày:** 2026-08-26 · **Trạng thái:** Draft · **Tier:** LITE (spec-only, dừng trước code)
- **Repo ảnh hưởng:** `exe-api` (backend Plan/SKU + admin-api) · `exe-admin` (màn admin mới)
- **KHÔNG ảnh hưởng:** `exe-web`, mobile (chỉ đọc `GET /api/plans` — hợp đồng giữ nguyên, xem §Out-of-scope)

## 1. Mục tiêu & vì sao

Admin cần **tự cấu hình giá gói + lợi ích + mã sản phẩm theo nền tảng** qua UI, thay cho cách hiện tại: sửa qua bảng generic `/r/plans` (chỉ có Key/Tên/Loại/Giá tháng, `features` phải gõ JSON thô) hoặc chạy seed hardcode (`plan.seed.js`).

Mẫu đích: bảng **Packages** — cột `Product Name | Product ID | Platform | Type | Duration | Discount | Price | Status | Actions`, filter theo Platform/Duration/Type/Status + search theo tên/ID + nút "New".

## 2. Bối cảnh hiện trạng (đã verify)

- Backend `Plan` model (`exe-api/.../modules/plan/plan.model.js`): key(unique, enum 5), name, type, priceMonthly/priceYearly, features[], **appleProducts[]** (chỉ iOS), isFeatured/isActive/sortOrder.
- CRUD gói ĐÃ có qua factory (`admin-api/resources/ops.admin.js:31`), permission `plan:*` đã đăng ký, menu "Gói dịch vụ" FE đã chạy (`exe-admin/.../resources.ts:161`).
- Khoảng trống vs mẫu: **không có** khái niệm SKU per-platform, **thiếu Android**, không có cột Platform/Duration/Discount, không có promo giảm giá gắn gói.

## 3. Phạm vi (đã chốt với PO)

- **Data model:** giữ `Plan` = 1 tier logic (`features[]` nguyên vẹn); **THÊM `skus[]`** subdoc. → phương án "A2". **Nới `key`** từ enum cứng → chuỗi unique tự do (để thêm gói `infinity`; đã verify không code nào hardcode key).
- **Gói (Marketing):** `standard` (129k), `premium` (249k), `infinity` (369k). Ẩn (isActive=false) các gói cũ `basic/single_product/b2b_center` (không xoá).
- **Thời hạn (Duration):** enum **1 / 3 / 6 / 12 tháng**.
- **Giá theo (gói × thời hạn)** — bảng Marketing:
  | Gói | 1th | 3th | 6th | 12th |
  |---|---|---|---|---|
  | Standard | 129.000 | 349.000 | 639.000 | 999.000 |
  | Premium | 249.000 | 669.000 | 1.199.000 | 1.899.000 |
  | Infinity | 369.000 | 989.000 | 1.799.000 | 2.799.000 |
  - `originalPrice` = giá 1 tháng × số tháng (mốc mua lẻ). `discountPercent` = suy ra (originalPrice→price), khớp % Marketing.
- **Platform:** model hỗ trợ `web`/`ios`/`android`, nhưng đợt này **chỉ seed `web`** (chưa có productId iOS/Android — tích hợp checkout tính sau).
- **Discount:** promo gắn SKU (`originalPrice` + `discountPercent`), **tách khỏi Coupon**.
- **Actions:** Edit / Delete(soft) / **Duplicate** (nhân bản gói/SKU — đã chốt).
- **FE:** màn **bespoke** đúng mẫu (~5 file), mỗi dòng = 1 SKU.
- **Backend:** SkuSchema + SKU-rows endpoint + fix 3 bug (xem design.md).

## 4. Out-of-scope (có lý do)

- **Limit tính năng theo gói** (20 thẻ/ngày, 5 lượt AI/tháng…) — **KHÔNG làm đợt này.** Verified: `entitlement` chưa wire vào tính năng nào (`entitlement.service.js:33`), `quota.config.js` chỉ có tier guest/free, chưa có tier trả tiền. Config giá **không đụng** hành vi tính năng. Cột features trong bảng Marketing hiện chỉ là dữ liệu tham chiếu, chưa enforce. Enforcement = việc riêng sau (wire entitlement + thêm tier 'paid').
- **Sửa `features[]` qua màn Packages** — màn này chỉ config giá/SKU. Ma trận tính năng theo gói làm ở màn/đợt riêng.
- **Tính năng khóa/chưa có** (lộ trình động, lịch học nhắc ngày, …) — không đưa vào đợt này (chỉ định của PO).
- **Luồng checkout/thanh toán thật** theo SKU — chỉ CRUD catalog; `payment`/`subscription` chưa migrate sang skus[] (giữ appleProducts[] song song).
- **Product ID Web/Android** — chưa tạo trên cổng thanh toán / Google Play Console → để `null`/TBD, tích hợp sau.
- **`GET /api/plans` public DTO** — mặc định KHÔNG expose `skus[]` ra web.
- **`path-pricing-packaging`** (bán lộ trình theo khóa) — feature riêng.

## 5. Acceptance Criteria (checkable)

- [ ] AC1 — Admin (có `plan:write`) tạo được 1 gói mới với ≥1 SKU (platform+duration+price), lưu thành công, xuất hiện trong danh sách.
- [ ] AC2 — Bảng hiển thị **1 dòng / SKU**; cột Platform/Duration/Discount/Price/Status/Product ID render đúng dữ liệu.
- [ ] AC3 — 4 filter (Platform/Duration/Type/Status) + search (tên/Product ID) lọc đúng kết quả.
- [ ] AC4 — Sửa `price`, `originalPrice`, `discountPercent`, `productId` của 1 SKU → lưu → phản ánh ngay khi reload.
- [ ] AC5 — `discountPercent` hiển thị + tính giá sau giảm nhất quán (giá gạch ngang originalPrice → price).
- [ ] AC6 — SKU platform `ios` giữ productId cũ từ `appleProducts[]` sau backfill (không mất dữ liệu iOS đang có).
- [ ] AC7 — Xoá gói là **soft-delete** (`isActive=false`), không xoá cứng; gói inactive bị loại khỏi `GET /api/plans` public.
- [ ] AC8 — **Regression:** `GET /api/plans` public trả đúng như trước (không lộ skus[]/appleProducts), `Plan.key` vẫn unique.
- [ ] AC9 — **Regression flashcard:** luồng tạo/ôn flashcard chạy y nguyên (không đọc Plan — verified, xem design.md §Impact).
- [ ] AC10 — RBAC: user không có `plan:write` không thấy/không gọi được tạo-sửa-xoá (nút New gate đúng permission).

## 6. Câu hỏi mở

Q1–Q4 (Actions/appleProducts/productId/Duration) đã chốt — xem §3. Còn:
- [ ] Nới `key` enum → chuỗi tự do (để thêm `infinity`) — chờ xác nhận cuối (đề xuất: đồng ý, đã verify an toàn).
- [ ] Ẩn hẳn (isActive=false) hay xoá dữ liệu gói cũ `basic/single_product/b2b_center`? (đề xuất ẩn).
