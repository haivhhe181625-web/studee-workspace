---
feature: luyen-de-paper-import
status: draft
created: 2026-08-17
role: reviewer
phase: technical-review (lite)
---

# Technical Review — Import canonical JSON (adversarial pass)

Pressure-test design/tasks trước khi owner duyệt. Findings xếp theo mức chặn.

## F1 — BLOCKING: free-form không publish được

`test-paper.service.publish()` (`modules/test-paper/test-paper.service.js:68`) gọi `getBlueprint(paper.blueprintKey)` → `getBlueprint` **throw "Unknown blueprint"** khi `blueprintKey` null. Nhưng quyết định #2 = **free-form** (`blueprintKey` optional). → Đề import free-form ra `draft` rồi **KHÔNG bao giờ publish được** → toàn bộ vet-gate → publish vô nghĩa.

**Fix (bắt buộc, thêm task):** `publish()` tolerate free-form — nếu `paper.blueprintKey == null` → **bỏ `validatePaper`**, chỉ flip `status='published'`. Nếu có blueprintKey → giữ validate như cũ.
→ Đã thêm **T3b** vào `tasks.md`.

## F2 — CONFIRM: shape `matching_headings`

`grade.js` chú thích `matching_headings` = "ordered array map", nhưng `QuestionCard` render nó bằng single-select (McqBody, response=index string). Chưa rõ item **lưu trong bank** answerKey dạng array (cả nhóm) hay single value (mỗi câu 1 heading). Ép validator "array" có thể sai với dữ liệu thật.

**RESOLVED (2026-08-17):** kiểm bank — 18 item `matching_headings` đều `answerKey = number (index)`, KHÔNG array. Validator xử như `mcq` (number/string index). `data-model.md §2` đã cập nhật.

## F3 — HARDEN: `ref` stimulus khác skill

Stimulus `ref` chưa kiểm `skill` khớp `paper.skill`. Rủi ro: reuse stimulus listening trong đề reading (hoặc ngược lại) → testlet sai. **Fix:** trong service khi resolve `ref`, kiểm `stimulus.skill === paper.skill` (+ centerId null ở P1) → không khớp thì reject. → Bổ sung vào T3.

## F4 — ACCEPTED: atomicity (no multi-doc txn)

Mongo standalone → ghi nhiều doc không nguyên tử. Validate-all-first (design §3) bắt hầu hết lỗi trước khi ghi; lỗi hiếm giữa chừng (vd copy audioFile fail) để lại **draft mồ côi** — KHÔNG tới học viên (chưa publish), admin dọn được. Chấp nhận P1. Không cần fix.

## F5 — NOTE: preview lộ answerKey

CLI P1 in preview ra console (dev/ops) — có answerKey để duyệt, chấp nhận. **Khi lên admin API preview (M7-16 phase sau)** phải gate `test-paper:manage` + không trả learner. Ghi để không quên.

---

## Kết luận

- **F1 phải fix** (đã thêm T3b) trước khi feature dùng được.
- F2 kiểm nhanh trước khi code T2. F3 gộp vào T3.
- F4/F5 chấp nhận/ghi chú.
- Sau khi vá: design nhất quán, sẵn sàng owner sign-off → superpowers.
