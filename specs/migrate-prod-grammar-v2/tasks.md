# Tasks: Migrate nội dung ngữ pháp prod lên v2

> **Cho người thực thi (owner tự chạy trực tiếp tới prod):** làm đúng thứ tự Task 1 → 5. Đây là data migration (không phải code TDD): "test" = các assert verify bằng lệnh. KHÔNG chạy Task tiếp theo nếu Expected của Task trước không đạt.

**Design:** `specs/migrate-prod-grammar-v2/design.md`
**Lưu ý chung:** mọi lệnh chạy trong `exe-api/services/api`. Scripts đã tồn tại (không cần code thêm).

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/scripts/migrate-grammar-v2-to-prod.js` | Backup + copy dev→prod + reslug attempt | Đã tạo |
| `services/api/scripts/restore-grammar-prod.js` | Rollback từ backup JSON | Đã tạo |
| `services/api/scripts/reanchor-grammar-items-v2.js` | Nguồn `ANCHOR_MAP` (reslug attempt import lại) | Có sẵn |

---

## Task 1: Set URI + dry-run (không ghi)

**Files:** không sửa.

- [x] **Step 1: Set biến URI** (lấy password từ `.env`; đừng dán password ra nơi bị log)
```bash
export DEV_URI='mongodb://root:<DEV_PASS>@42.96.12.209:27020/enova_dev?authSource=admin'
export PROD_URI='mongodb://enova_app:<PROD_PASS>@42.96.12.209:27017/enova?authSource=admin&directConnection=true'
```
`<DEV_PASS>` ← `MONGODB_URI` trong `.env`; `<PROD_PASS>` ← `MONGODB_URI_PROD`. **Bắt buộc `&directConnection=true`.**

- [x] **Step 2: Dry-run** ✅ khớp: lessons 55/55, items 448/184, attempts 6, reslug 3+3, không mutation.

Run:
```bash
node scripts/migrate-grammar-v2-to-prod.js
```
Expected: in `lessons dev=55 prod=55`, `items dev=448 prod=184`, `attempts ... prod=6`, và reslug plan `future_perfect_continuous -> future-perfect-continuous: 3`, `simple_past_tense -> past-simple: 3`. Không có mutation.

---

## Task 2: Thực thi migrate (backup + copy + reslug)

**Files:** không sửa (chạy script).

- [x] **Step 1: Chạy --write** ✅ backup 3 collection (1.0MB/174KB/3.8KB) → copy 55 lesson + 448 item → reslug 6 attempt. Prod ghi được (là primary).

Run:
```bash
node scripts/migrate-grammar-v2-to-prod.js --write
```
Expected: 3 dòng `backed up ... (>0B)`; `grammar_lesson_contents: copied 55 ... prod now has 55`; `grammar_items: copied 448 ... prod now has 448`; `... attempts reslugged`; dòng cuối `DONE ... Backup: <path>`.

- [x] **Step 2: Ghi lại `<backup-dir>`**: `C:\Users\Legion5\.claude\jobs\920b7eaa\tmp\grammar-prod-backup\`.

- [x] **STOP nếu gặp `not primary`**: N/A — không gặp, prod 27017 ghi được qua `directConnection`.

---

## Task 3: Verify dữ liệu prod

**Files:** không sửa.

- [x] **Step 1: So sánh lesson dev↔prod** ✅ `55 shared, 0 differ, 0 dev-only, 0 prod-only`.

Run (script cmp đã có trong job tmp):
```bash
node "<job-tmp>/cmp-grammar.js"
```
Expected: `55 shared, 0 differ, 0 dev-only, 0 prod-only`.

- [x] **Step 2: Kiểm item + attempt trên prod** ✅ items=448 (kebab, 0 orphan), attempts=6 (grammarPointAtTime kebab).

Expected: `grammar_items` = 448, `grammar.grammarPoint` toàn kebab, 0 orphan; `grammar_attempts` = 6, `grammarPointAtTime` toàn kebab.

---

## Task 4: Verify FE thật trên prod

**Files:** không sửa.

- [ ] **Step 1: Mở prod web** `/grammar/present-simple` — ⏳ CHỜ owner soi mắt thường (code v2 đã deploy prod qua release v0.6.0; hard-refresh Ctrl+Shift+R để bỏ cache).

Expected: hiển thị **summary** + **accordion phân mục** (children) + tab **Luyện tập** hiện số câu ≥ 1. Mở thêm 1 bài khác (vd `/grammar/conditional-sentence`) xác nhận tương tự.

---

## Task 5: Chốt / rollback

**Files:** không sửa.

- [x] **Nếu tất cả Task 3 PASS:** migrate data thành công. Giữ `<backup-dir>` vài ngày. (Deploy code prod: PR #184 develop→main + release `v0.6.0` → "Deploy to VPS" success 1m39s.) Task 4 chờ owner soi.

- [ ] **Nếu bất kỳ verify FAIL:** rollback. (Chưa cần — chưa có FAIL.)

Run:
```bash
node scripts/restore-grammar-prod.js <backup-dir>          # dry-run xem trước
node scripts/restore-grammar-prod.js <backup-dir> --write  # khôi phục
```
Expected: 3 collection về đúng số docs backup.

---

## Self-Review

**Acceptance coverage:**
- AC1 (lesson 55 kebab active v2) → Task 3.1. ✅
- AC2 (items 448 kebab, 0 orphan) → Task 3.2. ✅
- AC3 (attempts 6 kebab, không mất) → Task 3.2. ✅
- AC4 (0 lesson khác biệt) → Task 3.1. ✅
- AC5 (FE v2) → Task 4.1. ⏳ chờ owner soi (code đã deploy v0.6.0)
- AC6 (backup > 0 trước ghi + rollback chạy) → Task 2.1 + 5. ✅ (backup xong; restore script sẵn, chưa cần dùng)

**Placeholder scan:** `<DEV_PASS>`/`<PROD_PASS>`/`<backup-dir>`/`<job-tmp>` là chỗ điền lúc chạy — không phải TODO code.

**Câu hỏi mở chặn:** `not primary` (spec §7) — chỉ biết khi chạy Task 2; đã có nhánh xử lý.
