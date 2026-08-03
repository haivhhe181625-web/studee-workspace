<!-- Tiếng Việt — PRD/SRS cho "Checkpoint chặng + Nâng cấp band mượt". Viết cho người KHÔNG chuyên kỹ thuật.
     Bản kỹ thuật/chi tiết: specs/course-checkpoints-band-up/spec.md + design.md. -->

# PRD — Bài kiểm tra chặng & Nâng cấp trình độ mượt

- **Ngày:** 2026-08-03 · **Trạng thái:** Hoàn thành & deploy · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Chi tiết kỹ thuật:** `specs/course-checkpoints-band-up/spec.md` (hiện trạng code) | `design.md` (nếu có)

> Tài liệu mô tả **chức năng đã có & sẽ hoàn thiện** bằng ngôn ngữ dễ hiểu. Chi tiết kỹ thuật đọc kèm spec.md.

---

## 1. Mục đích & bối cảnh

Để bảo đảm học viên **nắm vững các kỹ năng** trước khi tiến sang chặng tiếp theo, hệ thống cần một **bài kiểm tra** ở cuối mỗi chặng (phase) và cuối khóa (course). Khi học viên pass bài kiểm tra, không chỉ chứng minh năng lực mà **trình độ của họ sẽ được nâng cấp dần dần** (không nhảy cả bậc) dựa trên điểm số đạt được.

**Bối cảnh hiện tại:** Học viên học theo một khóa cố định (ví dụ A1→A2 hoặc IELTS 5.0→6.0); khi hoàn thành chặng, trình độ nâng 1 bậc nguyên (A1→A2). Nhưng điều này **không công bằng**: nếu học viên đạt 85% ở chặng này nhưng 95% ở chặng tiếp theo, tại sao cả hai đều nâng +1 bậc? Lý tưởng hơn là **nâng dần theo từng điểm**, để học viên chậm tiến độ được ghi nhận tiến bộ liên tục.

**Mục tiêu:** 
- Có một **bài kiểm tra gating** rõ ràng ở cuối chặng/khóa, đảm bảo chất lượng.
- **Nâng cấp trình độ mượt** (ví dụ A2 → A2+ → B1 thay vì nhảy) dựa trên điểm checkpoint.
- Học viên được **thi lại vô hạn** (không phạt, không cooldown) để cải thiện điểm nếu muốn.
- Có **trang lịch sử** để học viên theo dõi quá trình nâng cấp của mình rõ ràng.

---

## 2. Phạm vi

**TRONG phạm vi:**
1. **Bài kiểm tra chặng (Checkpoint)** — bài thi ngắn cuối chặng, gồm các câu hỏi trắc nghiệm từ các kỹ năng của chặng đó (Nghe, Đọc, Viết, Nói, Ngữ pháp, Từ vựng).
2. **2-bước mở khóa (2-step gate):**
   - Bước 1: Hoàn thành tất cả bài học của chặng → checkpoint sẵn sàng để làm.
   - Bước 2: Đạt điểm yêu cầu ở checkpoint → chặng tiếp theo mở khóa (học viên có thể tiếp tục).
3. **Nâng cấp mượt (Smooth Band-up)** — khi pass checkpoint, trình độ kỹ năng được tính theo điểm chia nhỏ (ví dụ nâng 0.5 bậc nếu đạt 75%, 1.0 bậc nếu đạt 95%), không nhảy cả bậc.
4. **Thi lại không giới hạn** — học viên được phép submit checkpoint lại bất kỳ lúc nào, không bị phạt band, và hệ thống lấy điểm cao nhất.
5. **Trang lịch sử tiến bộ** — một dòng thời gian hiển thị mỗi lần học viên nâng trình độ (từ đánh giá ban đầu, checkpoint, hoặc tái kiểm tra), giúp họ thấy rõ hành trình.
6. **Kiểm tra chặng & khóa** — checkpoint có thể ở cuối chặng hoặc cuối khóa tùy thiết lập.

**NGOÀI phạm vi (để sau):**
- Tạo câu hỏi checkpoint bằng AI.
- Thi lại kỹ năng riêng lẻ mà không học lại cả chặng (sẽ là tính năng "tái kiểm tra kỹ năng" sau).
- Chặn tốt nghiệp hoặc bài tập bù nếu cuối khóa chưa đạt mục tiêu (để lần sau).
- Phân tích chi tiết per-bài hoặc per-chuyên đề (analytics).

---

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên** | Làm bài kiểm tra cuối chặng → xem kết quả (pass/fail + điểm per kỹ năng) → nếu pass, trình độ nâng + chặng tiếp theo mở → xem lịch sử tiến bộ. |
| **Học viên (fail)** | Thi lại checkpoint lần khác, KHÔNG bị phạt, tìm lại resources yếu để luyện. |
| **Người quản lý nội dung (admin)** | Chọn bài kiểm tra (quiz) cho từng chặng, cài ngưỡng điểm (ví dụ 70% pass, 80% pass, tùy chặng). |
| **Tech Lead / Ops** | Xem nhật ký nâng cấp band (từ checkpoint, từ đánh giá, …) để audit trước khi học viên thi IELTS/TOEIC thật. |

---

## 4. Tổng quan

### (A) Bài kiểm tra chặng — Cấu trúc & Gating

Mỗi **chặng** (hoặc khóa) có thể được gắn một **bài kiểm tra** cuối cùng. Bài kiểm tra gồm:
- **Các câu hỏi trắc nghiệm (MCQ)** từ các kỹ năng khác nhau (Nghe, Đọc, Viết, Nói, Ngữ pháp, Từ vựng).
- **Ngưỡng pass** được cài bởi quản lý (mặc định 80% = điểm ≥ 0.8/1.0).
- **Tài liệu hỗ trợ** nếu cần (audio nghe, đoạn đọc, …).

Để học viên có thể làm checkpoint:
1. Phải hoàn thành tất cả **bài học của chặng** trên lộ trình chính (on-path).
2. Sau đó, checkpoint xuất hiện & sẵn sàng.
3. Học viên submit → hệ thống chấm & trả kết quả ngay.

Khi pass:
- Chặng đánh dấu hoàn thành ✓.
- **Chặng tiếp theo tự động mở khóa** → học viên có thể bấm "Tiếp tục" hoặc "Chặng kế".
- Trình độ kỹ năng **nâng cấp mượt** (chi tiết ở mục 4B).

Khi fail:
- Checkpoint vẫn ở trạng thái "sẵn sàng" → học viên có thể thi lại.
- Chặng tiếp theo **vẫn khóa**.
- Trình độ kỹ năng **KHÔNG thay đổi** (không bị giảm, không được nâng).

### (B) Nâng cấp mượt — Từ điểm sang trình độ

Hiện tại, trình độ được ghi dưới dạng **bậc CEFR** (A1, A2, B1, B2, C1, C2). Nhưng thực tế, **trình độ là liên tục**, không phải nhảy cắp. Để mô hình hóa điều này:

- Mỗi kỹ năng có một **"điểm năng lực"** nội bộ (ví dụ 1.0–6.0) → khi hiển thị, hệ thống lấy phần nguyên (floor) để đổi thành bậc CEFR (1.0–1.99 = A1, 2.0–2.99 = A2, 3.0–3.99 = B1, …).
- Khi học viên pass checkpoint, **per-skill sub-score** (ví dụ Nghe 0.90, Đọc 0.75) được dùng để tính **cộng điểm** cho kỹ năng đó.
- Phân bổ: từ khi chọn khóa, hệ thống tính **ngân sách per-kỹ năng** (chia mục tiêu thành số chặng dạy kỹ năng đó) → mỗi pass checkpoint credit một phần ngân sách đó.

**Ví dụ cụ thể:**
- Học viên hiện tại: Listening A2 (điểm năng lực = 2.5), mục tiêu = B1 (3.0).
- Chặng A2→B1 có 3 checkpoint (cuối mỗi tuần).
- Ngân sách Listening cho chặng = (3.0 − 2.5) ÷ 3 = 0.167/checkpoint.
- Pass checkpoint 1 → Listening += 0.167 → 2.667 (hiển thị A2, nhưng sẽ sớm nâng).
- Pass checkpoint 2 → Listening += 0.167 → 2.834 (hiển thị A2).
- Pass checkpoint 3 → Listening += 0.167 → 3.0 (hiển thị B1 ✓).

Nếu học viên thi lại checkpoint 1 lần nữa → KHÔNG được credit lại (hệ thống nhớ đã credit checkpoint 1 cho Listening rồi); chỉ cập nhật nếu điểm mới cao hơn (best-score-wins).

### (C) Trang lịch sử tiến bộ

Học viên có thể xem một **dòng thời gian** ghi lại mỗi lần trình độ thay đổi:
- **Lần 1:** Đánh giá ban đầu → Listening A1, Reading A2, …
- **Lần 2:** Pass checkpoint chặng 1 → Listening nâng A1→A2 ✓, Reading vẫn A2.
- **Lần 3:** Pass checkpoint chặng 2 → Reading nâng A2→B1 ✓, Listening vẫn A2.
- Mỗi entry ghi rõ **nguồn** (đánh giá, checkpoint, tái kiểm tra), **ngày giờ**, **toàn bộ trình độ hiện tại**, và **kỹ năng nào nâng**.

---

## 5. Quy tắc gating & retry

### Mở khóa chặng
- **Điều kiện 1:** Hoàn thành tất cả bài học chặng hiện tại.
- **Điều kiện 2:** Pass checkpoint (overall score ≥ ngưỡng, mặc định 80%).
- **Kết quả:** Chặng kế mở; nút "Tiếp tục" enable; học viên có thể nhảy sang chặng tiếp.

### Retry checkpoint
- **Được phép:** Thi lại bất kỳ lúc nào, unlimited attempts.
- **Không phạt:** Fail KHÔNG deduct band, KHÔNG cooldown, KHÔNG cảnh báo hạn chế.
- **Best-score-wins:** Hệ thống lưu điểm cao nhất; nếu lần 2 pass nhưng điểm thấp hơn lần 1, vẫn tính lần 1.
- **Idempotent credit:** Nếu đã credit ngân sách kỹ năng từ lần 1, lần 2 KHÔNG credit lại (tránh stack 2 lần).

### Nâng cấp per-kỹ năng
- Khi overall score ≥ ngưỡng, hệ thống kiểm tra **sub-score từng kỹ năng**:
  - Nếu sub-score kỹ năng X ≥ ngưỡng (ví dụ 0.8) → **kỹ năng X được nâng** (credit ngân sách).
  - Nếu sub-score kỹ năng Y < ngưỡng (ví dụ 0.65) → **kỹ năng Y KHÔNG nâng** (người dùng vẫn được thấy điểm nhưng yêu cầu học lại).
- Nếu lần tới pass checkpoint (sub-score kỹ năng Y ≥ ngưỡng) → kỹ năng Y mới được nâng.

---

## 6. Yêu cầu chức năng (SRS)

### Nhóm A — Bài kiểm tra & Gating

**FR-A1** Hệ thống **tạo node checkpoint** ở sidebar (giao diện học tập) với 3 trạng thái:
- **Khóa:** Còn bài học chưa hoàn thành (ghi "Hãy hoàn thành bài học trước").
- **Sẵn sàng:** Tất cả bài học xong, có thể làm ngay (hiển thị nút "Làm bài kiểm tra").
- **Đạt:** Đã pass (hiển thị badge ✓, kèm kết quả & nút "Xem lại" hoặc "Thi lại").

**FR-A2** Khi học viên chọn node checkpoint sẵn sàng, hệ thống **hiển thị bài kiểm tra đầy đủ:**
- Các câu hỏi trắc nghiệm (MCQ) từ các kỹ năng của chặng.
- Tài liệu hỗ trợ (audio nghe nếu có, đoạn đọc nếu có).
- Nút "Nộp bài" để submit câu trả lời.

**FR-A3** Học viên **submit checkpoint** → hệ thống chấm & **trả kết quả ngay (pass/fail):**
- **Nếu pass:** 
  - Tiêu đề "Chúc mừng! Bạn đã pass checkpoint."
  - Hiển thị **điểm từng kỹ năng** (ví dụ Nghe 0.85/1.0, Đọc 0.90/1.0, Viết 0.70/1.0).
  - Highlight **kỹ năng nâng lên** (ví dụ "Listening A2 → A2+ ✓" hoặc "Reading A2 → B1 ✓").
  - Ghi chú **kỹ năng chưa đạt** (ví dụ "Writing 0.70/1.0 — chưa đạt ngưỡng").
  - Nút "Tiếp tục" → mở chặng tiếp theo.
- **Nếu fail:**
  - Tiêu đề "Bạn chưa pass. Hãy thử lại."
  - Hiển thị điểm từng kỹ năng (tương tự).
  - Gợi ý "Kỹ năng yếu nhất: Viết (0.60/1.0) — review [bài học link]".
  - Nút "[Thi lại]" hoặc "[Quay về lộ trình]".

**FR-A4** **Chặng tiếp theo tự động mở khóa** khi pass (học viên thấy nút "Tiếp tục"/"Chặng kế" enable, có thể click).

**FR-A5** Nút **"Chặng kế"** bị disable + tooltip nếu checkpoint chưa pass (giúp học viên hiểu tại sao không thể tiếp tục).

### Nhóm B — Nâng cấp & Retry

**FR-B1** Khi pass checkpoint, hệ thống **per-skill** kiểm tra sub-score:
- **Nếu kỹ năng sub-score ≥ 0.8** (hoặc ngưỡng tùy config) → **cộng điểm năng lực** cho kỹ năng đó (dựa trên ngân sách phân bổ).
- **Nếu sub-score < ngưỡng** → **không cộng** (yêu cầu học viên luyện thêm).

**FR-B2** Trình độ được **ghi dưới dạng điểm năng lực (1.0–6.0) nội bộ**; hiển thị **bậc CEFR (A1–C2)** bằng cách lấy phần nguyên (1.0–1.99=A1, 2.0–2.99=A2, …, 6.0=C2).

**FR-B3** **Retry unlimited:** học viên có thể submit checkpoint lại bất kỳ lúc nào, không cooldown, KHÔNG "banned" message.

**FR-B4** **Best-score-wins:** 
- Lần 1 pass (điểm: Nghe 0.85, Đọc 0.90) → ngân sách Nghe/Đọc được credit.
- Lần 2 pass (điểm: Nghe 0.75, Đọc 0.95) → chỉ Đọc được credit (Nghe đã credit lần 1, không stack).
- Nếu lần 2 Nghe đạt 0.95 (cao hơn 0.85) → Nghe vẫn KHÔNG credit lại (idempotent).

**FR-B5** **Fail KHÔNG deduct band:** đạo lần nào fail, trình độ kỹ năng vẫn giữ nguyên, không bị giảm.

### Nhóm C — Lịch sử tiến bộ

**FR-C1** Hệ thống **ghi nhật ký** mỗi lần trình độ thay đổi (từ đánh giá ban đầu, checkpoint, tái kiểm tra, …) vào một **dòng thời gian**.

**FR-C2** Học viên **xem trang lịch sử** (`/adaptive/history` hoặc tương tự):
- **Timeline:** mới nhất trước.
- **Per entry:**
  - Badge **nguồn:** "📊 Đánh giá", "✓ Checkpoint", "🔄 Tái kiểm tra".
  - **Trình độ snapshot:** "Listening A2, Reading B1, Writing A2, …"
  - **Kỹ năng nâng:** "[Listening A1→A2, Reading A2→B1]"
  - **Ngày giờ:** "2026-08-01 14:30"

**FR-C3** Nếu lần đầu (chưa có lịch sử), hệ thống hiển thị **hàng cơ sở** từ đánh giá ban đầu.

### Nhóm D — Admin & Cấu hình

**FR-D1** Người quản lý nội dung có thể **set checkpoint cho từng chặng:**
- Chọn một bài kiểm tra (quiz) từ kho có sẵn.
- Cài **ngưỡng pass** (ví dụ 0.70, 0.80, 0.90).
- Lưu → checkpoint được gắn vào chặng.

**FR-D2** Bài kiểm tra phải là **bài được xuất bản** (published status), không phải bản nháp.

---

## 7. Yêu cầu phi chức năng (SRS)

**NFR-1 (Dữ liệu bài kiểm tra):** Checkpoint phải bao gồm **câu hỏi từ nhiều kỹ năng** của chặng đó (Nghe + Đọc + Viết + Nói + Ngữ pháp + Từ vựng, tùy chặng), để đánh giá toàn diện.

**NFR-2 (Chương trình tính ngân sách):** Ngân sách per-kỹ năng được **tính & đóng băng** lúc học viên chọn khóa, KHÔNG thay đổi nếu admin sửa chặng sau (tránh race condition).

**NFR-3 (Hiệu năng):**
- **Chấm nhanh:** submit checkpoint → trả kết quả < 2 giây.
- **Lịch sử load:** trang lịch sử render < 1 giây.

**NFR-4 (Bảo mật):**
- Đáp án đúng KHÔNG gửi cho client (học viên không thấy).
- Audio nghe & đoạn đọc được **ký khoá thời gian** (3 giờ) — học viên không thể tải raw file.
- Chỉ học viên enrolled khóa mới thấy checkpoint (check quyền trước).

**NFR-5 (Idempotent):** Nếu học viên bấm submit 2 lần cùng lúc (double-click) → hệ thống ghi nhận 1 lần, không stack.

---

## 8. Luồng người dùng (tóm tắt)

1. **Học viên chọn khóa** → hệ thống tính ngân sách nâng cấp per-kỹ năng.
2. **Học qua các chặng:**
   - Hoàn thành bài học chặng → checkpoint xuất hiện (sẵn sàng).
   - Submit checkpoint → hệ thống chấm.
3. **Pass checkpoint:**
   - Kỹ năng với sub-score ≥ 0.8 được nâng (credit ngân sách).
   - Trình độ update (ví dụ A2 → A2.5 → A2+ hiển thị → B1 sau next checkpoint).
   - Chặng tiếp theo mở.
4. **Fail checkpoint:**
   - Trình độ KHÔNG đổi.
   - Gợi ý yếu đề → học lại → thi lại.
5. **Xem lịch sử** → dòng thời gian nâng cấp (mỗi checkpoint, mỗi kỹ năng).

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

**Chức năng:**
- [ ] Sidebar hiển thị node Checkpoint (3 trạng thái: khóa/sẵn sàng/đạt).
- [ ] Bài kiểm tra gồm MCQ từ nhiều kỹ năng, tài liệu đầy đủ (audio, đoạn đọc).
- [ ] Kết quả (pass/fail) trả ngay, ghi rõ điểm per-kỹ năng + kỹ năng nâng.
- [ ] Pass checkpoint → chặng tiếp theo unlock; fail → chặng khóa.
- [ ] Thi lại unlimited (no cooldown, no penalty), best-score-wins.
- [ ] Trình độ nâng mượt (ví dụ A2 → A2.x → B1), không +1 nguyên.
- [ ] Per-skill idempotent credit (thi lại chặng 1 lần 2 KHÔNG credit ngân sách lần 2).
- [ ] Trang lịch sử hiển thị timeline (newest first), per entry: badge nguồn + trình độ snapshot + kỹ năng nâng + ngày giờ.
- [ ] Admin set checkpoint per chặng (pick quiz, cài ngưỡng).

**Ví dụ cụ thể:**
- Học viên Listening hiện A2 (điểm 2.5), mục tiêu B1 (3.0). Chặng A2→B1.
- Pass checkpoint 1 → Listening += 0.167 → 2.667 (hiển thị A2).
- Pass checkpoint 2 → Listening += 0.167 → 2.834 (hiển thị A2).
- Pass checkpoint 3 → Listening += 0.167 → 3.0 (hiển thị B1 ✓).
- Lịch sử ghi 3 entry (3 lần nâng), mỗi entry "Listening A2→A2+" (3 lần) → cuối cùng "Listening → B1".

**Kiểm tra:**
- Lộ trình cố định vẫn hoạt động (không bị ảnh hưởng).
- Checkpoint chỉ phục vụ gating & nâng cấp, không làm đổi nội dung khóa.

---

## 10. Ngoài phạm vi

- **Tái kiểm tra kỹ năng riêng** (short assessment chỉ cho 1 kỹ năng để nâng nhanh) — từ từ sau.
- **Tạo checkpoint bằng AI** — lần sau.
- **Chặn/remediation** (bắt buộc làm bài bù nếu cuối khóa chưa target) — sau.
- **Phân tích per-bài** (analytics: câu nào nhiều người sai nhất?) — sau.
- **Non-MCQ testlet** (true/false, matching, …) — hiện chỉ MCQ.
- **Native exam scores** (IELTS/TOEIC → chỉ map CEFR hiển thị, không lưu) — sau.

---

## 11. Thuật ngữ (cho người không chuyên)

- **Checkpoint (Bài kiểm tra):** Bài thi ngắn ở cuối chặng hoặc khóa, để xác nhận nắm vững kỹ năng.
- **Chặng (Phase):** Một "nấc" trong khóa học, ví dụ A1→A2 hay B1→B2, thường gồm 3–4 tuần học.
- **Khóa (Course):** Toàn bộ chương trình, ví dụ "IELTS 5.0→6.0" (gồm nhiều chặng).
- **Kỹ năng:** Nghe / Đọc / Viết / Nói / Ngữ pháp / Từ vựng.
- **CEFR (Common European Framework of Reference):** Khung trình độ tiếng Anh quốc tế: A1 (sơ cấp nhất), A2, B1, B2, C1, C2 (thành thạo).
- **Nâng cấp mượt (Smooth Band-up):** Nâng trình độ dần dần theo từng điểm nhỏ (ví dụ A2 → A2+ → A2++ → B1), KHÔNG nhảy cả bậc.
- **Điểm năng lực (Band Point):** Con số nội bộ 1.0–6.0 biểu thị trình độ kỹ năng liên tục; khi hiển thị, lấy phần nguyên để đổi thành CEFR (1.0–1.99=A1, 2.0–2.99=A2, …).
- **Sub-score:** Điểm của từng kỹ năng trong một bài kiểm tra (ví dụ Nghe 0.85/1.0, Đọc 0.75/1.0).
- **Ngân sách nâng cấp (Budget):** Tổng điểm được phép cộng cho kỹ năng từ khi chọn khóa đến cuối khóa, chia đều theo chặng.
- **Idempotent (Không stack):** Làm lại action cùng một lần KHÔNG cộng đôi kết quả; ví dụ thi lại checkpoint 1 lần 2 KHÔNG credit ngân sách lần 2.
- **Best-score-wins:** Lấy điểm cao nhất trong tất cả attempt; fail lần này KHÔNG ảnh hưởng nếu pass lần sau.
- **Lộ trình cố định (Fixed Path):** Một đường học tuyến tính có sẵn (A1→A2→B1→…) giống như trước đây.
- **On-path (Trên lộ trình):** Các bài học được gắn vào lộ trình chính của học viên (bắt buộc để mở checkpoint).
