---
feature: luyen-de-difficulty-assembly
status: draft
created: 2026-08-18
repos: [exe-api]
epic: PRD-151 (M7 Luyện đề)
step: 3/3 (blueprint storage → item generation → difficulty assembly)
---

# Spec — Bật độ khó thành filter thật trong assemble (Bước 3/3)

## 1. Mục tiêu & lý do

Hiện assemble bốc câu chỉ theo `cefr` + `itemTypes`; `difficultyTier` là nhãn mô tả không tác động. Khi
bank đã dày (Bước 2), muốn kiểm soát độ khó **mịn hơn mức cefr** — chọn "B1 dễ" vs "B1 khó" trong cùng một
section.

Điểm tận dụng: item bank **đã có `irt.b`** (tham số độ khó liên tục), và gen-testlet đã set `irt.b` tạm theo
tier (easy=-1, mid=0, hard=+1). Bước 3 thêm **1 ràng buộc `irt.b` theo section** vào aggregation assemble —
không phải làm lại gì, chỉ thêm `$match`.

**Chỉ hiệu quả sau Bước 2** (bank đủ dư để có chỗ chọn theo độ khó). Non-adaptive như IELTS/TOEIC: neo
chính vẫn là cefr; `irt.b` chỉ tinh chỉnh trong tầng.

## 2. Phạm vi

- **Blueprint section** (`PaperBlueprint`, Bước 1): thêm field optional `irtBand { min, max }` **hoặc** dù
  map `difficultyTier → dải b` mặc định (easy≤-0.5, mid (-0.5,0.5), hard≥0.5). Ưu tiên map từ tier để không
  bắt người soạn nhập số IRT (KISS); `irtBand` chỉ khi cần override.
- **`assemble`**: thêm `$match` `irt.b` vào pipeline mỗi section theo dải của section. **Fallback an toàn**:
  nếu bank thiếu câu trong dải → nới/bỏ ràng buộc b (giữ cefr+itemTypes) để không tạo đề rỗng; báo khi phải nới.
- **Không** thêm `difficultyTier` vào `AssessmentQuestion` (dùng `irt.b` sẵn có thay vì đẻ field trùng nghĩa
  — tránh 2 nguồn sự thật độ khó, theo phân tích đã thống nhất).

## 3. Ngoài phạm vi

- **Calibrate IRT thật từ dữ liệu trả lời** — `item-quality.service` xử lý sau khi có lượt làm; Bước 3 dùng
  `b` hiện có (tạm hoặc đã calibrate, đều chạy).
- **Adaptive/CAT** — vẫn non-adaptive; chỉ lọc tĩnh theo dải b.
- **Thêm field difficultyTier vào item** — cố ý không làm (xem §2).

## 4. Acceptance Criteria

- [ ] AC1 — Section có dải b (từ tier hoặc irtBand) → assemble chỉ bốc câu có `irt.b` trong dải + cefr +
  itemTypes; câu ngoài dải không lọt (bank đủ dư).
- [ ] AC2 — Bank thiếu câu trong dải → assemble **nới** ràng buộc b (giữ cefr+itemTypes) để vẫn đủ câu, và
  **báo** section nào phải nới (không âm thầm tạo đề lệch/rỗng).
- [ ] AC3 — Section không cấu hình dải b → hành vi như hiện tại (chỉ cefr+itemTypes) — backward compat.
- [ ] AC4 — Random ($sample) vẫn giữ trong tập đã lọc theo b (assemble lại ra bộ khác).
- [ ] AC5 — Full suite test-paper xanh; test cũ (không cấu hình b) không đổi kết quả.

## 5. Repos ảnh hưởng

- **exe-api** — `paper-blueprint.model.js` (field optional), `test-paper.service.js` (assemble `$match` b + fallback).

## 6. Liên kết

- Phụ thuộc **Bước 1** (schema khung ở DB) + **Bước 2** (bank đủ dư mới có tác dụng). Làm sau cùng.
- Research: `plans/reports/research-260818-ielts-toeic-paper-structure-for-blueprint-model.md` (§difficulty
  modeling: cefr vs IRT b vs tier).
- Phân tích quyết định cefr+irt (không dùng tier trên item): xem lịch sử spec + chat design.

## Câu hỏi mở

- Dải b mặc định theo tier (ngưỡng ±0.5) — chốt số khi review.
- Ngưỡng "phải nới" (thiếu bao nhiêu % thì bỏ ràng buộc b) — chốt khi review.
