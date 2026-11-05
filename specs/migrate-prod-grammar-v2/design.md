# Design: Migrate nội dung ngữ pháp prod lên v2

- **Ngày:** 2026-08-24
- **Spec:** `specs/migrate-prod-grammar-v2/spec.md`
- **Trạng thái:** Đã duyệt (owner sign-off 2026-08-24) — chờ thực thi. Hệ quả phụ (`createdBy`/`itemId` mồ côi) đã được chấp nhận.

## 1. Vì sao cần design (không phải CRUD tầm thường)

Có 3 điểm không hiển nhiên: (a) đổi slug snake→kebab kéo theo coupling ở 2 collection khác; (b) không có script tái tạo 448 item từ nguồn → phải copy cross-DB; (c) ghi lên prod thật (backup/rollback/topology replica set).

## 2. Phương án: copy dev → prod (không dùng import-script cho item)

Dev đã là trạng thái đích chuẩn cho cả `grammar_lesson_contents` (55 v2) lẫn `grammar_items` (448). Đường đúng = copy 2 collection dev→prod, không tái sinh từ `output-v2/*.v2.json` (git-ignored, và không có script sinh 448 item).

Đã cân nhắc & loại:
- **Chạy lại `import-grammar-content-v2.js --fresh --publish` + `reanchor-grammar-items-v2.js` trên prod** (đường dev đã đi): tạo lesson v2 OK nhưng chỉ reslug 184 item cũ → **không đạt D1** (owner muốn 448). Loại.
- **mongodump/mongorestore**: được, nhưng phụ thuộc mongodb-database-tools + xử lý `--drop` mất index. Loại vì node script không phụ thuộc tool ngoài và giữ index.

## 3. Thiết kế script

Một script `services/api/scripts/migrate-grammar-v2-to-prod.js`:
1. **Backup** 3 collection prod ra JSON (verify size > 0 nếu không rỗng) — chạy trước mọi mutation.
2. **Copy** `grammar_lesson_contents` + `grammar_items` dev→prod bằng `deleteMany({})` + `insertMany` (KHÔNG `drop` → giữ index + unique constraint).
3. **Reslug** `grammar_attempts.grammarPointAtTime` theo `ANCHOR_MAP` (import từ `reanchor-grammar-items-v2.js` — DRY). Attempt dev không copy.

Thuộc tính an toàn:
- URI qua env `DEV_URI`/`PROD_URI` — không hardcode secret.
- Mặc định **dry-run**; chỉ ghi khi `--write`.
- Prod URI **bắt buộc `&directConnection=true`** (replica set khai báo host nội bộ `mongodb`; thiếu → `ENOTFOUND mongodb`).

Rollback: `services/api/scripts/restore-grammar-prod.js <backup-dir> --write` — `deleteMany`+`insertMany` từ JSON backup.

## 4. Data model & coupling

Không đổi schema. Chỉ dịch chuyển & remap dữ liệu.

- `grammar_lesson_contents`: fields v2 (`summary`, `objectives`, `children`) đã có trong `grammar-content.model.js` — copy là đủ.
- `grammar_items.grammar.grammarPoint`: sau copy = slug kebab của dev (448 item đã anchor sẵn). Verify: 0 orphan.
- `grammar_attempts.grammarPointAtTime`: mastery match theo field này (`getMastery`/`getMasterySummary`). Prod 6 attempt dùng 2 điểm (`future_perfect_continuous`, `simple_past_tense`) — cả 2 có trong `ANCHOR_MAP` → reslug sang `future-perfect-continuous`/`past-simple`.

Hệ quả phụ chấp nhận được:
- Item copy mang `createdBy` = user id của dev (không có trên prod): vô hại, bị strip khỏi learner DTO.
- Attempt prod sau copy có `itemId` không trỏ item mới nào: điểm vẫn lưu trên attempt, mastery vẫn tính đúng (match theo `grammarPointAtTime`, không theo item tồn tại).

## 5. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Prod là replica-set secondary → insert `not primary` | TB | `directConnection=true`; nếu vẫn fail → dừng, dùng URI `replicaSet=` + seed list (câu hỏi mở spec §7) |
| Backup lỗi mà vẫn ghi | Cao | Backup chạy trước + verify size; giữ `<backup-dir>` cho rollback |
| Window ~1–2s giữa deleteMany→insertMany user thấy trống | Thấp | Giờ thấp tải; thao tác < 1 phút |
| Index không tự tạo lại | Thấp | `deleteMany` (giữ index), không `drop` |

## 6. Đã verify (bằng chứng)

- ANCHOR_MAP phủ 22/22 điểm item prod, 22/22 target tồn tại lesson dev (0 orphan).
- Dry-run script chạy đúng: `lessons dev=55/prod=55, items dev=448/prod=184, attempts prod=6`, reslug plan 3+3.
- Coupling & mastery match: `grammar-practice.service.js:61,96`.
