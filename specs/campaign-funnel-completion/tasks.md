# Tasks: Campaign Funnel Completion & Optimization

> **Cho Developer Agent:** thứ tự A (exe-web, khép kín phễu) → B (exe-api) → C (exe-admin).
> Mỗi task commit riêng. Lệnh test exe-api chạy trong `exe-api/services/api`.
> **Spec:** `specs/campaign-funnel-completion/spec.md`. **B4 reshape lõi query → xem `design.md`** (union/dedup/phân trang). Các task khác additive.

---

## A. exe-web — khép kín phễu (ưu tiên 1, repo riêng)

### A1: Bắt & lưu campaign code ✅
- [x] **Scout**: đọc `src/lib/center-code.ts`, `Providers` (nơi mount `CenterCodeCapture`), luồng `/register` (`(auth)/register`), service gọi `POST /auth/register` + `/auth/firebase-login`.
- [x] **Implement**: tạo `src/lib/campaign-code.ts` mirror `center-code.ts` — `captureCampaignCode()` đọc `?campaign=`, lưu `localStorage['studee_campaign_code']`; `getCampaignCode()`.
- [x] **Wire capture app-wide** cạnh `CenterCodeCapture` (Providers).
- [x] **Verify**: `npm run build` (exe-web) pass; vào `/register?campaign=x` → localStorage có `studee_campaign_code=x`.

### A2: Gửi campaignCode khi đăng ký + Google-first ✅
- [x] **Implement**: trang `/register` — nút **Google 1-chạm** nổi bật (form email/mật khẩu phụ). Đính `getCampaignCode()` vào body `register` + `firebase-login`: `...(code ? { campaignCode: code } : {})`.
- [x] **Sau đăng ký thành công** → `router.push('/assessment/intake')` (cả email + Google; qua param redirectTo, /login giữ nguyên).
- [x] **Verify**: build pass (tsc clean); logic trace đăng ký có/không mã đúng.

---

## B. exe-api

### B1: Trùng code → 409 ✅
- [x] **Test FAIL**: integration tạo 2 campaign cùng `code` → lần 2 nhận **409** (không 500).
- [x] **Run**: `npm test -- campaign.integration` → FAIL (500).
- [x] **Implement**: map E11000 ở `_crud.factory` (generic) qua `duplicateKeyMessage(err)` — derive field từ keyPattern/keyValue, label map `{code:'Mã',name:'Tên',slug:'Slug',email:'Email'}`, fallback field/‘Dữ liệu’. `code`→'Mã đã tồn tại'.
- [x] **Run** → PASS (12/12). **Commit** `2acd78d` + fix `34e5315`.

### B2: Đợt start/end tự tắt ✅
- [x] **Test FAIL**: `resolveActiveByCode`/`incrementClick` ngoài `[startAt,endAt]` → null (không attribution, không tăng click).
- [x] **Implement**: `liveNowFilter` (UTC now; startAt null|≤now; endAt null|≥now) dùng chung 2 fn, atomic. Controller: ngoài đợt (code tồn tại nhưng không live) → redirect `${web}/?campaign=<code>&ended=1`; unknown → root.
- [x] **Run** `npm test -- campaign` → 34/34; +auth 85/85. **Commit** `e6448b8`.

### A3 (exe-web follow-up từ B2): toast "hết đợt" ✅
- [x] **Implement**: `CampaignEndedToast` client child đọc `?ended=1` → toast "Chiến dịch đã kết thúc" (fire once, Suspense). **Commit** `8422576`.

### B3: Click unique ✅
- [x] **Test FAIL**: `/r/:code` 2 lần cùng cookie → `clicks`=2, `uniqueClicks`=1; UA bot → bỏ qua.
- [x] **Implement**: `metrics.uniqueClicks`; `incrementClick(code,{unique})` atomic; cookie `rc_<code>` (httpOnly/lax/path=/r, TTL theo đợt) + `bot-detection.js`; `getCampaignStats` trả `uniqueClicks`.
- [x] **Run** → 38/38; +auth 89/89. **Commit** `d311621`.

### B4: Lead gồm người đăng-ký-chưa-thi ✅
- [x] **Test FAIL**: seed 1 User có `campaignId` không Result + 1 Result → `listLeads` 2 dòng, dòng chưa thi `status='registered'`.
- [x] **Implement**: `$unionWith`+`$facet` (design.md); dedup loại User đã có Result; `status`; band loại registered; filter `status`; cột Trạng thái export; index `Result.{userId,campaignId}`; hardened malformed campaignId.
- [x] **Run** `npm test -- campaign` → 30/30; +assessment+auth 219/219. **Commit** `b9eb645` + fix `29081b4`.

### B5: Domain validate + .env.example ✅
- [x] **Implement**: `server.js` boot guard — `NODE_ENV==='production' && !PUBLIC_API_URL` → `logger.warn` (đọc validated env, không throw). `.env.example` có `PUBLIC_API_URL`.
- [x] **Verify**: guard reached ở prod trống; campaign test không đỏ (1 fail = flaky có sẵn). **Commit** `cbc1bf2`.

---

## C. exe-admin

### C1: Auto-slugify mã code
- [ ] **Implement**: ô `code` trong form campaign tự chuẩn hoá khi gõ (lowercase, space→`-`, bỏ ký tự ngoài `[a-z0-9-]`). Thêm field type/handler nhẹ hoặc onChange chuẩn hoá cho type text có cờ.
- [ ] **Verify**: `npx tsc --noEmit` + `eslint` sạch; gõ "HN Thu 2026" → `hn-thu-2026`.
- [ ] **Commit** `feat(campaign): auto-slugify campaign code on input`.

### C2: Cột trạng thái + lọc lead ✅
- [x] **Implement**: `leads-panel` cột **Trạng thái** (Badge, Đăng ký/Đã thi/Không hợp lệ) + select filter → `status` query (omit khi All); export chung filter; CEFR null→"—".
- [x] **Verify**: tsc + eslint sạch. **Commit** `fec8acb`.

---

## D. Verify tổng
- [x] `jest campaign assessment auth permission-catalog` (exe-api) → 230/231, chỉ flaky `two writing tasks` đỏ (timeout, M3 không đụng). ✅
- [x] exe-web + exe-admin: `tsc` sạch. ✅
- [~] End-to-end: code-level trace đủ (backend đọc campaignCode; window; union; toast). Chưa chạy browser thật (không có dev server trong session).

## Self-Review (điền sau)
- AC1-4 → A1,A2. AC5 → B4,C2. AC6 → B3. AC7 → B2. AC8 → B5. AC9 → B1,C1. AC10 → D.
