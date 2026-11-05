---
feature: grammar-content
status: draft
owner: BA
created: 2026-08-19
repos: [exe-api, exe-web]
---

# Spec — grammar-content (phục vụ & hiển thị 55 bài học ngữ pháp)

## Mục tiêu
Đưa **55 block-lesson ngữ pháp đã sinh** (offline, `exe-api/services/api/scripts/grammar-content/output/*.blocks.json`, `status: active`) vào sản phẩm: lưu DB, phục vụ qua REST, và render trên FE đúng như bản prototype (artifact review).

## Vì sao
Nội dung đã sinh + duyệt (active) nhưng **chưa có đường vào app**: chưa có model, endpoint, hay component FE. Đây là cầu nối từ "đã có dữ liệu" → "người học xem được".

## Phạm vi (in-scope)
1. **BE (`exe-api`)** — module mới `grammar-content` (5-file pattern):
   - Model lưu block-lesson theo `grammarPoint`.
   - `block-registry` validate block theo type, **fail-soft** (type lạ không làm sập).
   - Script import `output/*.blocks.json` → MongoDB (idempotent theo `grammarPoint`).
   - REST public: lấy 1 bài (chỉ `active`) + list theo loại.
   - Admin (qua `admin-api`): xem mọi bài (kể cả draft), đổi `status`.
2. **FE (`exe-web`)** — block renderer registry (port từ `render-review.js`): 1 component / 1 block type; study-space chọn renderer. *(triển khai ở PR riêng của exe-web; spec này đặt hợp đồng.)*

## Ngoài phạm vi (out-of-scope)
- **Nối SRS concept-based / lập lịch ôn** — thuộc plan cha `plans/260817-1604-grammar-srs-shared-module`; feature này chỉ phục vụ + render nội dung, KHÔNG đụng scheduling.
- **Sinh/biên tập nội dung** — pipeline offline đã xong; không build UI biên tập.
- **Gộp vào `course-content`** — grammar-content đứng riêng (course-content là course-centric).
- **Progress/tracking học** bài ngữ pháp — chưa, để phase sau.

## Repo/surface ảnh hưởng
- `exe-api/services/api/src/modules/grammar-content/*` (mới), `src/admin-api/resources/*` (thêm resource), `scripts/` (import).
- `exe-web` — study-space + component renderer (PR riêng).
- Không đụng `exe-admin` (admin qua admin-api + exe-admin portal có thể thêm sau).

## Acceptance Criteria (checkable)
- [ ] `GET /api/grammar-content/:grammarPoint` trả bài `active` với `learn[]` đúng contract; **404** nếu không tồn tại hoặc chưa active.
- [ ] Response public **không lộ** field nội bộ (`_source`, `_note`) và **không trả** bài `draft`.
- [ ] `GET /api/grammar-content?category=<slug>` trả list bài active của loại đó.
- [ ] Import script nạp đủ **55 bài**, chạy lại **không nhân đôi** (upsert theo grammarPoint).
- [ ] `block-registry` gặp block type lạ → **bỏ qua/flag, không throw** (fail-soft), các block hợp lệ vẫn phục vụ.
- [ ] Admin đổi `status` draft↔active có kiểm tra permission (`grammar-content:manage`); đổi xong endpoint public phản ánh đúng.
- [ ] FE render đủ **11 loại block** đúng hành vi prototype: formula ẩn/hiện ví dụ · signal tap nghĩa · minicheck chấm đúng/sai + giải thích · detail accordion mở/gập.
- [ ] Test BE xanh (`npm test` module grammar-content); không lỗi lint/syntax.

## Câu hỏi mở — ĐÃ CHỐT (2026-08-19)
1. `reference` → **KHÔNG lưu** DB/API (detail accordion đã đủ).
2. Liên kết course lesson → **để sau**, không chặn feature này.
3. Bảng dài → **FE tự gập** (accordion) khi quá dài.
