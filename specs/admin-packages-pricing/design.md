<!-- Tech Lead design (LITE). Data model gộp trong file này (đủ đơn giản). -->
# Design — Admin Packages (SKU per-platform + promo discount)

- **Ngày:** 2026-08-26 · **Trạng thái:** Draft · Dựa trên spec.md (đã chốt A2 + discount-b + 3 platform)
- **Plan kỹ thuật chi tiết (CK):** `exe-api`/… → `plans/260826-1457-admin-packages-pricing-screen/` (phase files). File này là bản design gọn cho spec pipeline; task chi tiết ở tasks.md.

## 1. Quyết định kiến trúc

**Plan = tier logic (giữ nguyên), THÊM `skus[]` subdoc per-platform.** Không phá `Plan.key` unique, không đụng `features[]`. Lý do: `entitlement`/`quota`/seed đang khóa vào `key`+`features` — reshape sâu = blast radius lớn (loại phương án "B: mỗi SKU 1 document").

FE bespoke ("A2") vì bảng generic render 1 dòng/document, không tự tách SKU thành nhiều dòng như mẫu.

## 2. Data model — SkuSchema (subdoc của Plan)

```
SkuSchema (_id: true — FE cần row key ổn định để sửa/xoá từng SKU):
  platform:       enum ['web','ios','android']   required
  productId:      String  (nullable — web/ios/android chưa tạo → TBD)
  durationMonths: enum [1, 3, 6, 12]  required    # thời hạn (chốt từ Marketing)
  price:          Number (VND)  required          # giá bán ô Marketing
  originalPrice:  Number (VND)  nullable           # = giá 1th × durationMonths (mốc mua lẻ)
  discountPercent:Number 0–100  nullable           # suy ra (originalPrice→price); promo gắn SKU (≠ Coupon)
  isActive:       Boolean  default true
```

Thêm vào PlanSchema: `skus: [SkuSchema] default []`. **Nới `key`** từ enum cứng (`PLAN_KEYS`) → `String unique` (verified không code nào hardcode key ngoài module plan → an toàn; entitlement tra Plan theo `sub.planKey` data-driven). Giữ `appleProducts[]` **song song** (Q6: giữ).

Seed đợt này: 3 gói `standard`/`premium`/`infinity`, mỗi gói 4 SKU platform=`web` theo bảng giá Marketing (spec §3). Ẩn `basic/single_product/b2b_center` (isActive=false).

Ràng buộc: nếu có `originalPrice` thì `price ≤ originalPrice`; `discountPercent` nhất quán với (originalPrice→price) — validate ở schema/service.

## 3. Backend thay đổi

- `plan.model.js`: thêm SkuSchema + `skus[]`. Cân nhắc index `skus.productId` (reverse lookup checkout tương lai) — chỉ thêm nếu cần, YAGNI.
- `admin-api/resources/ops.admin.js` (plan resource):
  - **Fix bug:** bỏ `description` khỏi writeFields (`:37` — field không tồn tại trên model, đang drop âm thầm).
  - Thêm `skus`, `sortOrder`, `appleProducts` (nếu vẫn muốn sửa iOS) vào writeFields.
- **SKU-rows endpoint** cho FE bespoke: expose danh sách phẳng (1 row/SKU) — hoặc action riêng trên plan resource, hoặc `GET /api/admin/plans/sku-rows`. NO business logic ở admin-api → flatten trong `plan.service.js`.
- Fix nav gate lệch permission (`plan:manage` vs nút New `plan:write`) — thống nhất một chuẩn.

## 4. FE (exe-admin) — bespoke, ~5 file

- `app/(admin)/packages/page.tsx` — render container, no logic.
- `components/features/packages/packages-table.tsx` — bảng đúng mẫu (cột + 4 filter + search + New).
- `hooks/use-packages.ts` — TanStack Query gọi service.
- `services/packages.service.ts` — gọi SKU-rows + mutation qua axios.
- `constants/query-keys.ts` + `constants/routes.ts` — bổ sung key/route.
- LƯU Ý `exe-admin/AGENTS.md`: "This is NOT the Next.js you know" — đọc `node_modules/next/dist/docs/` trước khi viết code Next.

## 5. Impact analysis — Config giá KHÔNG ảnh hưởng tính năng (VERIFIED)

**Toàn cục:** limit theo gói CHƯA enforce ở đâu. `entitlement` (đọc Plan.features[]) viết xong nhưng **chưa wire vào tính năng nào** (`entitlement.service.js:33`); `quota.config.js` chỉ có tier `guest`/`free`, **không có tier trả tiền** (comment: "No 'paid' tier — billing is a future plan"). Chỉ 5 quota trial guest/free đang chặn: pronunciation/IPA, conversationTurns, assessment, adaptive, flashcardGeneration — không liên quan gói. Giá (priceMonthly/skus) không tính năng nào đọc. → sửa giá/thêm SKU/thêm gói/đổi key = 0 tác dụng lên hành vi app.

**Flashcard (chi tiết):** trace bằng code thật:
- Flashcard **không đọc `Plan`**. Dùng module `quota` (counters trên User doc + `quota.config.js`) — `flashcard-generation-quota.service.js:18`.
- `flashcard.service.js:609` đọc `user.profile.studyPlan.flashcardNewPerDay` = **study plan SRS trên User**, KHÔNG phải Plan bảng giá (trùng chữ, false friend).
- Cầu nối Plan→quyền duy nhất: `entitlement.resolveFeatureAccess` (`entitlement.service.js:85-87`) — chỉ đọc `Plan.key` + `Plan.features[]`, và **chưa wire vào flashcard** (`entitlement.service.js:33`).

→ Thêm `skus[]`/`originalPrice`/`discountPercent`, giữ `key` unique + `features[]` → **flashcard zero impact**. (Rủi ro chỉ xuất hiện nếu chọn phương án B phá `key` unique — đã loại.)

## 6. Rủi ro & giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| Backfill mất productId iOS | Idempotent upsert appleProducts→sku(ios); giữ appleProducts[] song song; test AC6 |
| Leak productId ra web | `planToDto` KHÔNG thêm skus[] (giữ như appleProducts); test AC8 |
| FE bespoke lệch chuẩn Next mới | Đọc node_modules/next/dist/docs/ trước |
| Duration↔cycle không nhất quán (one_time) | Chốt mapping ở Q4 trước khi code model |

## 7. Câu hỏi mở (kỹ thuật)

- SKU-rows: action trên resource hay endpoint riêng? (nghiêng endpoint riêng cho FE bespoke).
- `skus[]._id`: bật `_id` để FE có row key ổn định khi sửa/xoá từng SKU?
- Có cần index `skus.productId` ngay không (checkout chưa migrate → chưa cần)?
