<!-- Task list TDD-checkbox cho Developer. Mỗi task 1 PR-worth, có verify. -->
# Tasks — Admin Packages (SKU per-platform + promo discount)

Thứ tự: backend model → admin-api → backfill → FE → regression. Mỗi task test-first nơi codebase đã TDD (backend Jest). `--code` mới thực thi; hiện dừng spec.

## Backend (exe-api)

- [ ] **T1 — SkuSchema + skus[] trên Plan + nới key**
  - Thêm SkuSchema (platform/productId/durationMonths∈{1,3,6,12}/price/originalPrice/discountPercent/isActive, `_id:true`) + `skus:[]` vào `plan.model.js`. Validate `price ≤ originalPrice`. **Nới `key`** từ enum `PLAN_KEYS` → `String unique`.
  - Verify: `npm test -- plan` (SKU hợp lệ pass, price>originalPrice fail, key tự do lưu được).

- [ ] **T2 — Fix plan resource writeFields**
  - `ops.admin.js`: bỏ `description`, thêm `skus`/`sortOrder` (+`appleProducts` nếu giữ sửa iOS).
  - Verify: admin-api CRUD test — set skus qua write, đọc lại khớp; `description` không còn.

- [ ] **T3 — SKU-rows flatten (service + endpoint)**
  - `plan.service.js`: hàm flatten Plan[]→SKU rows (kèm planKey/name/type). Expose qua `/api/admin/plans/sku-rows` (delegate service, no logic ở admin-api).
  - Verify: test — N plan có M sku → trả Σ rows đúng, mỗi row đủ field cột mẫu.

- [ ] **T4 — Fix nav/permission gate lệch**
  - Thống nhất `plan:write` vs `plan:manage` cho list/new. Verify: RBAC test — user chỉ `plan:read` bị chặn write.

## Backfill (exe-api)

- [ ] **T5 — Seed 3 gói Marketing (web) + backfill appleProducts→sku(ios)**
  - Seed idempotent: `standard`/`premium`/`infinity`, mỗi gói 4 SKU platform=`web` theo bảng giá Marketing (spec §3: price + originalPrice=1th×tháng + discountPercent). Ẩn `basic/single_product/b2b_center` (isActive=false).
  - Backfill: mỗi appleProducts entry → sku{platform:ios, productId} nếu skus chưa có (giữ productId iOS — AC6). Chạy nhiều lần không nhân đôi.
  - Verify: chạy 2 lần, count sku ổn định; 3 gói mới đúng giá; iOS productId giữ nguyên.

## FE (exe-admin) — bespoke A2

- [ ] **T6 — service + hook + query-keys/routes**
  - `packages.service.ts` (gọi sku-rows + mutations), `use-packages.ts`, bổ sung query-keys + routes.
  - Verify: `npm run build` pass; hook trả data từ sku-rows.

- [ ] **T7 — packages-table + page**
  - `packages/page.tsx` + `components/features/packages/packages-table.tsx`: đủ cột (Product Name/Product ID/Platform/Type/Duration/Discount/Price/Status/Actions=Edit/Delete/**Duplicate**), 4 filter, search, nút New. Duplicate cần action clone ở backend (thêm vào T3, delegate `plan.service`).
  - Verify: `npm run build` + smoke — bảng render 1 dòng/SKU, filter/search lọc đúng, Duplicate tạo bản sao (AC2/AC3).

- [ ] **T8 — form tạo/sửa gói + SKU (promo)**
  - Form set price/originalPrice/discountPercent/productId per SKU; hiển thị giá gạch ngang.
  - Verify: tạo gói+SKU (AC1), sửa giá/promo (AC4/AC5) reload khớp.

## Regression

- [ ] **T9 — Regression public + flashcard**
  - Verify: `GET /api/plans` không lộ skus[]/appleProducts, `Plan.key` unique giữ (AC8). Chạy suite flashcard: `npm test -- flashcard` xanh (AC9).

## Ghi chú

- Chốt Q1–Q4 (spec §6) trước T7/T8 (Actions/Duration) và T5 (giữ appleProducts).
- Không maintain 2 plan: file này là task canonical; `plans/260826-1457-*` giữ làm ghi chú engineering sâu, không đồng bộ tay 2 chiều.
