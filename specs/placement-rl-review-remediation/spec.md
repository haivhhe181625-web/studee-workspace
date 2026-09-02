---
status: draft
owner: BA
created: 2026-09-02
affects: [exe-api]        # + exe-admin/exe-web nếu chốt đổi label FE (xem §2b)
source: docs/review_cefr_placement_RL.md
---

# Spec — Placement R/L: khắc phục theo review (content QC + naming)

## 1. Mục tiêu & why

Review ngoài (`docs/review_cefr_placement_RL.md`) chấm kiến trúc 8/10 (APPROVE), đo lường
6/10 (NEEDS VALIDATION). Đã lọc: đa số finding là **chờ data pilot** (calibrate, standard-set,
testlet-dependence) hoặc **đã có sẵn/không hợp placement** — KHÔNG nằm trong spec này.

Spec này chỉ gom **những sửa đổi thật cần, trong tầm kiểm soát, không cần data pilot**:
1. Chất lượng item C1 (distractor + nhãn) — vì placement giờ **đẩy thẳng học viên vào lộ trình**
   (`overallCefr → student-model.level → recommendation`), item C1 không phân loại được người
   giỏi = xếp sai lộ trình cho học viên trình cao.
2. Naming "receptive" — hệ **cấp certificate in `overallCefr`**; bài chỉ đo Đọc+Nghe mà ghi
   "CEFR B2" = nói quá (rủi ro uy tín).

## 2. Phạm vi

### 2a. Trong scope
- **[CẦN] Content QC — distractor C1** (`exe-api`): viết lại distractor MCQ ở item C1 đang dùng
  từ tuyệt đối (entirely/completely/fully/never…) — loại được không cần đọc bài — thành distractor
  "plausible, sai vì sắc thái" (review §7.2). Sửa ở batch JSON `scripts/data/placement-content/*`
  + reload. KHÔNG đụng engine.
- **[NÊN] Naming receptive** (`exe-api`): wording certificate + nhãn kết quả không claim CEFR
  4 kỹ năng. Tối thiểu sửa BE (certificate.service.js / result DTO text). FE label → §2b.
- **[HOUSEKEEPING] Doc sync 14→15** (`studee-workspace`): `placement-item-bank/spec.md §2b` ghi
  Listening 14, code chạy 15 (`CORE_PLACEMENT_PRESET.fixedModuleCounts.listening=5` → 5×3). Đồng
  bộ số. Chỉ sửa doc.

### 2b. Cần chốt trước khi làm (Open Questions — xem §5)
- (Các câu hỏi scope Q1/Q2/Q3 đã chốt — xem §5.)

### 2c. Out of scope (lý do)
- Calibrate `irt.b` thật / CEFR cut-score standard-setting / testlet local-dependence (review
  §6.1–6.3, §14) — **cần data pilot**, là chương trình validation, không phải sửa code. Ghi roadmap.
- Hiển thị confidence/range cho học viên (review §7.4) — **bỏ**: UX noise cho bài ngắn low-stakes;
  số đã tính sẵn (`band-scoring` SE/CI), không cần show. (Quyết định user 2026-09-02.)
- MST chọn testlet theo information (review §7.3) — vô nghĩa khi `irt.b` còn provisional; defer tới
  sau calibration.
- Dual distribution config §11 — placement dùng fixed form, không áp dụng.
- Engine đo lường (EAP/routing/scoring), CORE_PLACEMENT_PRESET, form R16/L15 — không đổi.

## 3. Acceptance Criteria (checkable)

**Distractor C1**
- [ ] Có **1 mẫu distractor C1 viết lại** được user duyệt phong cách TRƯỚC khi rà loạt (gate).
- [ ] Không còn item C1 MCQ nào có distractor chứa từ tuyệt đối "khử được không cần đọc bài"
  (rà bằng danh sách item C1, đối chiếu tay + heuristic từ khoá).
- [ ] Mỗi item C1 sửa: đáp án đúng vẫn 1, các distractor đều "đọc như hợp lý", sai vì sắc thái.
- [ ] Reload xong: `getPlacementCoverage` không đổi số ô (chỉ sửa nội dung, không thêm/bớt item);
  smoke assemble vẫn ráp đủ R16/L15.

**Naming receptive**
- [ ] Certificate không in chuỗi ngụ ý "CEFR đầy đủ 4 kỹ năng" cho kết quả từ bài R/L; ghi rõ
  "Receptive (Reading + Listening)" / "Đọc–Nghe".
- [ ] Nhãn kết quả (BE DTO text) tương tự. FE tuỳ Q2.

**Doc sync**
- [ ] `placement-item-bank/spec.md §2b` bảng Listening cộng ra **15**, khớp `fixedModuleCounts`;
  ghi rõ +1 câu ở competency đã chốt (Q3).

## 4. Affected surfaces
- `exe-api/services/api/scripts/data/placement-content/*.json` (content C1).
- `exe-api/services/api/src/modules/assessment/certificate.service.js` (+ nơi tạo text kết quả).
- `studee-workspace/specs/placement-item-bank/spec.md` (§2b).
- (tuỳ Q2) `exe-admin` / `exe-web` — text hiển thị kết quả.

## 5. Quyết định đã chốt (2026-09-02)
- **Q1 — Explanation:** **điền CẢ 98 câu** thiếu explanation trong bank placement (không giới hạn C1).
- **Q2 — FE label:** **BE + certificate trước** (exe-api). FE (exe-admin/exe-web) làm sau nếu cần.
- **Q3 — +1 Listening (14→15):** vào **detail** (4→5).
