# Spec: Campaign Funnel Completion & Optimization

- **Ngày:** 2026-08-11
- **Tác giả:** BA (từ brainstorm)
- **Nguồn quyết định:** `plans/reports/brainstorm-decisions-260811-0945-campaign-lead-funnel-optimization-report.md`
- **Feature gốc:** `specs/campaign-qr-lead-management/` (đã build ở exe-api + exe-admin)
- **Repos/surfaces:** `exe-web` (khép kín phễu — CHÍNH), `exe-api` (tối ưu backend), `exe-admin` (polish).
- **Trạng thái:** Nháp — chờ BA review

## 1. Mục tiêu
Khép kín phễu tuyển sinh qua QR (phần `exe-web` còn thiếu) + áp 7 quyết định tối ưu đã chốt để feature chạy thật end-to-end và bớt lỗi lặt vặt.

## 2. Bối cảnh
Backend + admin đã xong (attribution `Assessment→Attempt→Result`, register-only, QR/export, dashboard). Review: 210/211 test (1 flaky có sẵn). **Blocker: `exe-web` chưa bắt `?campaign=` / chưa gửi khi đăng ký** → phễu chưa chạy. Kèm vài chỗ chưa tối ưu (lead thiếu nhóm chưa-thi, click raw, đợt không tự tắt, domain dễ lệch, trùng code 500).

## 3. Phạm vi

### Trong phạm vi
**A. exe-web (khép kín phễu):**
- `campaign-code.ts`: bắt & lưu `?campaign=<mã>` (mirror `center-code.ts`), capture app-wide.
- Trang `/register`: ưu tiên **nút Google 1-chạm**; gửi `campaignCode` khi `POST /auth/register` + `/auth/firebase-login`.
- Sau đăng ký thành công → điều hướng **`/assessment/intake`**.

**B. exe-api:**
- **Lead gồm người đăng-ký-chưa-thi**: `listLeads`/`exportLeads` union `User`(theo `campaignId`) + `Result`, cột trạng thái `registered|completed|invalid`.
- **Click unique**: thêm `metrics.uniqueClicks`; `/r/:code` set cookie (hoặc ip-hash Redis TTL) → chỉ `$inc uniqueClicks` khi mới; bỏ qua bot theo User-Agent.
- **Đợt tự tắt**: `resolveActiveByCode`/`incrementClick` kiểm `startAt<=now<=endAt`; ngoài đợt → redirect trang thông báo, không attribution.
- **Domain validate**: boot log cảnh báo nếu `NODE_ENV=production` mà `PUBLIC_API_URL` rỗng; thêm `.env.example`.
- **Trùng code → 409**: map `E11000` khi tạo campaign thành `ApiError(409)`.

**C. exe-admin:**
- Lead table: cột **trạng thái** (đăng ký/đã thi) + lọc theo trạng thái.
- Field `code`: **auto-slugify** khi gõ (space/hoa → gạch-ngang-thường).

### Ngoài phạm vi
- Guest/anonymous (làm bài trước, đăng ký sau) — **Lý do:** giữ register-only (QĐ2), tránh đụng lõi M3.
- Kiểm soát phơi nhiễm câu hỏi, đổi logic chấm/lắp đề M3.
- Đa-campaign/user (đổi first-touch) — **Lý do:** đã chốt first-touch khóa cứng.

## 4. Quyết định đã chốt (brainstorm)
1. Lead gồm cả người đăng ký chưa thi. 2. Register-only, Google 1-chạm chính. 3. Click raw + unique (cookie/ip-hash) + bỏ bot UA. 4. Đợt start/end tự tắt ngoài khoảng. 5. Domain qua env + validate cảnh báo. 6. Trùng code → 409 + auto-slugify. 7. Sau đăng ký → vào thẳng `/assessment/intake`.

## 5. Acceptance criteria
1. Vào `<web>/register?campaign=hn-thu2026` → mã lưu localStorage (`studee_campaign_code`), trang đăng ký hiện, nút Google nổi bật.
2. Đăng ký (Google hoặc email) → `campaignCode` được gửi → tài khoản gắn `campaignId`+`centerId` (first-touch); đăng ký lần sau **không đổi**.
3. Đăng ký xong → tự chuyển tới `/assessment/intake` (làm bài).
4. Đăng ký **không có mã** → vẫn thành công, không gắn campaign, không lỗi.
5. Dashboard lead hiển thị **cả người đăng ký chưa thi** với cột trạng thái; export chứa cột trạng thái.
6. `/r/:code`: quét lần 2 cùng thiết bị → `clicks` tăng, `uniqueClicks` **không** tăng; UA bot bị bỏ qua.
7. `/r/:code` **ngoài đợt** (`now` ngoài `[startAt,endAt]`) → không attribution, chuyển trang thông báo hết đợt.
8. Prod thiếu `PUBLIC_API_URL` → log cảnh báo lúc boot; `.env.example` có biến.
9. Tạo campaign trùng `code` → API trả **409** ("Mã đã tồn tại"); ô `code` tự slugify khi gõ.
10. Không đụng logic chấm/lắp đề M3; test hiện có vẫn xanh (trừ flaky đã biết).

## 6. Câu hỏi mở (đã chốt 2026-08-11 — xem `design.md`)
- [x] Lead "chưa thi" trường liên hệ tối thiểu → **email + tên là đủ** (Google cho email; sđt có thể trống).
- [x] Trang "hết đợt" (QĐ4) ở exe-web → **redirect `/` kèm toast/banner**, không tạo route mới.
- [x] Múi giờ so `now` với `startAt/endAt` → **UTC** (Date native, Mongo lưu UTC).
- Cơ chế: B1 map E11000 tại `_crud.factory` chung; B3 dùng cookie `rc_<code>`. Chi tiết B4 union ở `design.md`.

## 7. Ghi chú kỹ thuật
- Register attribution đã có (`auth.controller.attributeCampaign` + `user.service.lockCampaign`) — exe-web chỉ cần **gửi** `campaignCode`.
- Lead union: cẩn thận trùng (một User vừa có Result vừa là registration) → ưu tiên bản Result (đã thi) khi gộp theo userId.
- Click unique cookie ở endpoint public không auth: cookie `rc_<code>` TTL theo đợt, hoặc ip-hash Redis (đã có Redis).
- `.env` exe-api dùng qua docker-compose `env_file`; thêm `PUBLIC_API_URL`.
