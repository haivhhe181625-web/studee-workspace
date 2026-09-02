---
status: draft
spec: ./spec.md
created: 2026-09-02
---

# Tasks — Placement R/L review remediation

Thứ tự: T1 (rẻ, chốt nhanh) → T2 → T3 (cần gate duyệt mẫu) → T4 (điền cả 98 explanation).
Mỗi task độc lập-test-được. **Không đụng engine đo lường.** Repo mặc định `exe-api` trừ khi ghi khác.

Phụ thuộc: T3 chờ **mẫu distractor duyệt** (gate ở T3.0). T1/T2/T4 chạy độc lập được.

---

## T1 — [Doc] Sync Listening 14 → 15 (§2b)
Repo: `studee-workspace`. Chỉ sửa doc.
- [ ] Sửa bảng Listening trong `specs/placement-item-bank/spec.md §2b`: tổng = **15**, thêm +1 câu
  vào competency đã chốt (Q3, mặc định `detail` 4→5). Cập nhật dòng tổng + ghi chú khớp
  `CORE_PLACEMENT_PRESET.fixedModuleCounts.listening=5` (5 audio × 3).
- [ ] Sửa `docs/review_cefr_placement_RL.md §3` chú thích "Listening 14" → nêu rõ code chạy 15.
- **Verify:** `grep -n "Listening" specs/placement-item-bank/spec.md` — bảng cộng ra 15; không
  còn số 14 mâu thuẫn.

## T2 — [BE] Naming "receptive" ở certificate + result text
Repo: `exe-api`.
- [ ] Đọc `certificate.service.js` — chỗ dựng nội dung certificate từ `overallCefr`. Đổi wording:
  không để chuỗi ngụ ý CEFR 4 kỹ năng; thêm "Receptive (Reading + Listening) / Đọc–Nghe".
- [ ] Rà nơi tạo text nhãn kết quả trong DTO/service (nếu có chuỗi "CEFR" trơn cho kết quả
  placement) — sửa tương tự. Chỉ đổi **text**, không đổi field/logic.
- **Verify:** unit/snapshot certificate (nếu có) phản ánh chuỗi mới; `grep -rn "CEFR" certificate.service.js`
  không còn claim 4-skill. Chạy test module assessment không đỏ mới.
- Note: FE label (exe-admin/exe-web) tách ra, chỉ làm nếu Q2 = có.

## T3 — [Content] Viết lại distractor C1 (CẦN — gate duyệt mẫu)
Repo: `exe-api`. Sửa batch JSON `scripts/data/placement-content/*` + reload.

### T3.0 — Gate: mẫu duyệt trước
- [ ] Chọn 1 item C1 attitude/inference tiêu biểu (vd item "attitude toward the gig economy"),
  viết lại 3 option theo tiêu chí §7.2 (bỏ từ tuyệt đối; mỗi distractor là 1 diễn giải sai
  sắc thái). **Trình user duyệt phong cách. DỪNG tới khi OK.**

### T3.1 — Rà + liệt kê item C1 cần sửa
- [ ] Xuất danh sách item C1 (skill reading+listening, cefr C1) có `itemType=mcq`, kèm options,
  từ DB (hoặc từ `placement-forms-simulated.json`). Đánh dấu item có distractor chứa từ tuyệt đối
  (entirely/completely/fully/never/always/none/entire…) hoặc distractor "vô lý hiển nhiên".
- **Verify:** danh sách có ≥ số item C1 MCQ; mỗi item ghi rõ "cần sửa / đạt".

### T3.2 — Sửa distractor trong batch JSON
- [ ] Với mỗi item "cần sửa": chỉnh options trong đúng file batch nguồn (giữ answerKey trỏ đúng
  đáp án sau khi đổi thứ tự nếu có). Theo phong cách đã duyệt ở T3.0.
- [ ] Reload: chạy loader `scripts/load-placement-content.js` cho các batch đã sửa (idempotent).
- **Verify:**
  - `getPlacementCoverage({minPerCell:5})` số ô **không đổi** (chỉ sửa nội dung).
  - smoke `placement-assemble-smoke.test.js` xanh (ráp đủ R16/L15, item well-formed).
  - Chạy lại `simulate-placement-forms.py` (profile C1) → không còn distractor tuyệt đối trong đề C1.

## T4 — [Content] Explanation fill — CẢ 98 câu (Q1 chốt: điền hết)
Repo: `exe-api`. Sửa batch JSON + reload. Scope: **mọi item placement thiếu `explanation`**
(reading+listening, mọi CEFR), không chỉ C1.
- [ ] Liệt kê tất cả item placement (`targetGoal=general`, active) có `explanation` trống/rỗng.
- [ ] Điền 1 câu/mỗi item: chỉ chứng cứ trong passage/transcript (vì sao đáp án đúng, distractor
  sai). Sửa đúng file batch nguồn + reload (idempotent).
- **Verify:** đếm `explanation` trống toàn bank placement = **0** sau reload (dùng block
  `qc.itemsMissingExplanation` trong `simulate-placement-forms.py`, hoặc query đếm trực tiếp).
- Note: khối lượng lớn (~98 câu) — có thể chia batch nhỏ khi /superpowers để review từng phần.

---

## Định nghĩa done (feature)
- [ ] T1, T2, T3 xong + verify xanh; T4 theo Q1.
- [ ] Không regress: engine/coverage/form không đổi số; test module assessment xanh.
- [ ] Diff + acceptance trình user duyệt trước merge (gate người, không tự merge).

## Còn chặn
- **Mẫu distractor (T3.0)** — cần user duyệt phong cách trước khi rà loạt C1.
- (Q1/Q2/Q3 đã chốt: điền cả 98 explanation · BE+certificate · detail 4→5.)
