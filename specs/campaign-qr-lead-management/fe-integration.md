# FE Integration — Campaign QR & Lead Management

Tài liệu bàn giao cho nhân sự FE. Backend (`exe-api`) và admin portal (`exe-admin`) đã xong;
phần **bắt buộc còn lại nằm ở `exe-web`** (app học viên). Đọc kèm `spec.md` + `design.md`.

## 0. Trạng thái

| Phần | Repo | Trạng thái |
|---|---|---|
| API + logic (campaign, funnel, QR, lead, export, attribution) | `exe-api` | ✅ Xong |
| Admin: CRUD chiến dịch + dashboard funnel + tải QR + lead + export | `exe-admin` | ✅ Xong |
| **Học viên: bắt `?campaign=` khi đăng ký** | `exe-web` | ⛔ **CẦN LÀM** |
| (Tùy chọn) Hiện ảnh QR trên màn hình admin | `exe-admin` | ◻ Nice-to-have |

## 1. Luồng end-to-end (để hiểu vì sao cần exe-web)

```
QR (in ra)  →  GET <api>/r/<code>  →  302  →  <web>/register?campaign=<code>
                    (API tự đếm click)                     │
                                                           ▼
        exe-web bắt & lưu ?campaign=<code>  →  học viên ĐĂNG KÝ  →  gửi campaignCode khi register
                                                           ▼
                              API khóa User.centerId + campaignId (first-touch, 1 lần)
                                                           ▼
     Sau đăng ký → vào làm bài → chấm → Result gắn campaignId → hiện ở dashboard lead
```

> Link QR trỏ thẳng **trang đăng ký** (`/register?campaign=<code>`). Học viên đăng ký xong
> (mã được gửi kèm → tài khoản gắn campaign), rồi web đưa vào bài test. Dữ liệu funnel chỉ
> ghi từ bước đăng ký trở đi.

> Nếu exe-web **không** bắt & gửi `campaignCode`, học viên vẫn đăng ký/làm bài bình thường
> nhưng **không được gắn vào chiến dịch** → không lên dashboard lead, funnel chỉ đếm được click.

## 2. exe-web — phần CẦN LÀM

### 2.1 Bắt mã từ URL (mirror `src/lib/center-code.ts` đã có)

Tạo `src/lib/campaign-code.ts` — y hệt cơ chế `center-code.ts` hiện tại, chỉ đổi tên param sang `campaign`:

```ts
export const CAMPAIGN_CODE_KEY = "studee_campaign_code";

/** Nếu URL có ?campaign=<code> thì lưu lại (chuẩn hoá). No-op trên server. */
export function captureCampaignCode(): void {
  if (typeof window === "undefined") return;
  const raw = new URLSearchParams(window.location.search).get("campaign");
  if (!raw) return;
  const clean = raw.trim().toLowerCase().slice(0, 64);
  if (clean) {
    try { localStorage.setItem(CAMPAIGN_CODE_KEY, clean); } catch { /* storage blocked */ }
  }
}

/** Mã campaign đã lưu, hoặc undefined. */
export function getCampaignCode(): string | undefined {
  if (typeof window === "undefined") return undefined;
  try { return localStorage.getItem(CAMPAIGN_CODE_KEY) || undefined; } catch { return undefined; }
}
```

### 2.2 Gọi capture app-wide

Trong Providers (chỗ đang mount `CenterCodeCapture` cho `center-code`), gọi thêm `captureCampaignCode()`
— để bắt được mã dù học viên vào bất kỳ trang nào rồi mới đăng ký, và **sống sót qua redirect đăng nhập**.

### 2.3 Gửi `campaignCode` khi đăng ký

Ở luồng đăng ký hiện có, đọc `getCampaignCode()` và đính vào body:

- **Đăng ký email/password:** `POST /api/auth/register`
- **Đăng nhập/đăng ký Google (Firebase):** `POST /api/auth/firebase-login`

```ts
import { getCampaignCode } from "@/lib/campaign-code";

const campaignCode = getCampaignCode();

// register
await api.post("/auth/register", {
  email, password, name,
  ...(campaignCode ? { campaignCode } : {}),
});

// hoặc firebase
await api.post("/auth/firebase-login", {
  idToken,
  ...(campaignCode ? { campaignCode } : {}),
});
```

### 2.4 Quy tắc quan trọng (đừng làm sai)

- **`campaignCode` là TÙY CHỌN.** Người dùng tự do (B2C) đăng ký **không có mã** → vẫn hợp lệ, không lỗi.
  Không được bắt buộc field này.
- **First-touch, khóa 1 lần.** Backend chỉ gắn campaign khi tài khoản **chưa** có. Lần đăng nhập/quét QR
  sau **không đổi** được. FE không cần xử lý gì thêm — cứ gửi mã, backend tự lo idempotent.
- **Không có chế độ khách.** Bắt buộc đăng ký/đăng nhập mới làm bài (giữ nguyên hành vi hiện tại).
- **KHÔNG cần** exe-web tự gọi `/r/:code` hay tự đếm click — đó là việc của API (QR trỏ thẳng vào `/r/:code`).

### 2.5 Contract API (để FE khớp)

`POST /api/auth/register`
```jsonc
// request
{ "email": "a@b.com", "password": "min 8 ký tự", "name": "Tên", "campaignCode": "hn-thu2026" /* optional */ }
// response 201
{ "user": { "id": "...", "email": "...", "name": "...", "role": "user", ... }, "accessToken": "...", "refreshToken": "..." }
```

`POST /api/auth/firebase-login`
```jsonc
// request
{ "idToken": "<firebase id token>", "name": "optional", "campaignCode": "hn-thu2026" /* optional */ }
// response 200 — cùng shape user + tokens
```

> `campaignCode`: chuỗi, đã lowercase/trim, tối đa 64 ký tự, khớp `^[a-z0-9-]+$`. Mã sai/không tồn tại →
> backend bỏ qua (không lỗi), tài khoản không gắn campaign.

### 2.6 Acceptance (exe-web)

- [ ] Vào `<web>/register?campaign=hn-thu2026` → mã được lưu (localStorage `studee_campaign_code`); trang đăng ký hiện ra.
- [ ] Đăng ký mới (có mã) → tài khoản gắn đúng campaign (kiểm ở dashboard lead admin sau khi làm bài).
- [ ] Đăng ký mới **không có mã** → thành công bình thường, không gắn campaign, không lỗi.
- [ ] Đăng nhập lại bằng tài khoản đã gắn campaign qua QR khác → **không đổi** campaign.

## 3. exe-admin — đã xong + (tùy chọn) hiện ảnh QR

**Đã có** (`/leads` + `/r/campaigns`):
- CRUD chiến dịch (tạo/sửa/liệt kê/lưu trữ) qua resource `campaigns`.
- Dashboard funnel per chiến dịch (click → đăng ký → làm bài → hoàn thành + tỉ lệ + phân bố band).
- **Tải QR PNG/SVG** mỗi chiến dịch (để in).
- Danh sách lead + lọc (chiến dịch/band/thời gian) + **export Excel/CSV**.

**Tùy chọn nếu muốn XEM ảnh QR trên màn hình** (không chỉ tải): gọi cùng endpoint QR, nhận blob, hiển thị `<img>`:
```ts
const res = await apiClient.get(`/admin/campaigns/${id}/qr`, { params: { format: "png" }, responseType: "blob" });
const src = URL.createObjectURL(res.data); // dùng cho <img src={src} />; nhớ revokeObjectURL khi unmount
```

## 4. Tham chiếu API admin (exe-admin đang dùng — để đối chiếu)

| Method | Path | Việc | Quyền |
|---|---|---|---|
| GET/POST/PATCH/DELETE | `/api/admin/campaigns` (+ `/:id`) | CRUD chiến dịch (DELETE = lưu trữ) | `campaign:read/write` |
| GET | `/api/admin/campaigns/:id/stats?from&to` | Funnel 1 chiến dịch | `campaign:read` |
| GET | `/api/admin/campaigns/_/report?centerId&from&to` | Roll-up theo trường | `campaign:read` |
| GET | `/api/admin/campaigns/:id/qr?format=png\|svg` | Tải QR (nhị phân) | `campaign:read` |
| GET | `/api/admin/leads?campaignId&centerId&from&to&bandMin&bandMax&page&limit` | Danh sách lead | `campaign:read` |
| GET | `/api/admin/leads/export?<lọc>&format=xlsx\|csv` | Export (nhị phân) | `campaign:export` |

Ghi chú nhị phân: `/qr` và `/export` phải fetch bằng axios `responseType: "blob"` (đã có JWT interceptor)
rồi tự tải xuống client-side — `<a href>` thường **không** mang được header Authorization.

## 5. Env / cấu hình

- `NEXT_PUBLIC_API_URL` (đã có) — base API.
- URL trong QR do backend tự dựng từ host request (`<api-host>/r/<code>`). Khi deploy sau reverse proxy,
  đảm bảo header host đúng để QR trỏ về đúng domain API. (Nếu cần cố định, đề nghị backend thêm env `PUBLIC_API_URL`.)

## Câu hỏi mở

- Có cần trang "QR gallery" riêng trên exe-admin (xem nhanh nhiều QR cùng lúc để in hàng loạt) không? Hiện chỉ tải từng cái.
- exe-web: đặt bước "ép đăng ký" ở đâu trong luồng landing tuyển sinh (trang riêng hay modal trên trang test)?
