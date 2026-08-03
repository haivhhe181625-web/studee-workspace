# PRD — Bộ 3 Bài Học Tương Tác trên Bản Đồ Lộ Trình (MCQ, Flashcard, Video)

- **Ngày:** 2026-08-03 · **Trạng thái:** Đã build (spec hồi tố từ code) · **Đối tượng đọc:** Founder / Giáo vụ / BA
- **Chi tiết kỹ thuật:** `specs/learning-map-content-players/spec.md` · `design.md` · `data-model.md`
- **Phụ thuộc:** Nền tảng Lộ trình 4 tầng (Course/Phase/Module/Lesson) đã có sẵn

> Tài liệu mô tả **bộ 3 loại bài học** (trắc nghiệm, thẻ ghi nhớ, video) sẽ được học viên làm trực tiếp trên bản đồ lộ trình.
> Mục tiêu: Từ bỏ bài học giả, chuyển sang **nội dung thật, chấm điểm tự động, chống gian lận**.

---

## 1. Mục đích & bối cảnh

### Tình hình hiện tại
Trước đây, khi học viên nhấn vào một bài học trên bản đồ lộ trình, hệ thống cho họ làm một bài kiểm tra **giả** (nhấn "Làm" → chọn các tùy chọn ngẫu nhiên → tất cả đều "đúng"). Dù tiện, nhưng **không phản ánh được trình độ thực tế** của học viên, làm mất **ý nghĩa của lộ trình cá nhân hóa**.

### Tại sao cần thay đổi
Để lộ trình cá nhân hóa **thực sự hoạt động**, mỗi bài học cần **nội dung thật, được chấm một cách công bằng**. Chỉ như vậy:
- **Học viên** biết mình thực sự thành thạo hay vẫn còn yếu (feedback chính xác).
- **Hệ thống** nắm được năng lực thực tế để điều chỉnh lộ trình phù hợp (bỏ bài dễ, tập trung vào chỗ yếu).
- **Giáo vụ** tin tưởng dữ liệu tiến bộ để quyết định can thiệp ngoài khóa (luyện thêm, tương tác 1-1).

### Giải pháp
Xây dựng **3 loại bài học tương tác** được đặt vào các nút của bản đồ, mỗi loại:
- Có **nội dung thật** do giáo vụ soạn/chuẩn bị (câu hỏi trắc nghiệm curated, thẻ ghi nhớ hệ thống, video giáo dục).
- Được **chấm server** (không tin client, không thể gian lận bằng cách sửa kết quả).
- **Tự động cập nhật hồ sơ năng lực** khi học viên hoàn thành.

---

## 2. Phạm vi

### TRONG phạm vi

1. **Bài trắc nghiệm (MCQ_QUIZ)** — Câu hỏi nhiều lựa chọn cố định, chấm server-side, đáp án ẩn.
2. **Thẻ ghi nhớ (FLASHCARD_DECK)** — Bộ thẻ hệ thống (term ↔ definition), học viên tự đánh giá, chấm FSRS đơn giản.
3. **Video bài giảng (VIDEO_LESSON)** — Video MP4 hoặc YouTube, học viên báo cáo tỷ lệ xem, chấm server.
4. **Chống gian lận** — Đáp án không lộ client, nội dung ràng buộc 1 lần (replay không tái dùng), chấm server tất cả.
5. **Chỉnh sửa UI cho admin** — Thêm nút "Kiểm tra" để admin xác minh video tồn tại trước khi xuất bản.
6. **Khóa gating (guardrail)** — Thẻ ghi nhớ và video **KHÔNG được** làm yêu cầu tiên quyết dạng "cứng" (bài học tiếp theo chỉ mở nếu bài này đạt A).

### NGOÀI phạm vi

- **Sinh nội dung động** (câu hỏi random từ kho, shuffle thẻ) — loại bài này dùng content **cố định curated**, random thuộc flow tách biệt.
- **Lưu trữ Spaced Repetition (SRS)** — map node chỉ tính điểm 1 lần (throwaway), không persist trạng thái thẻ vào Anki.
- **Upload media** — video dùng self-host hoặc link bên ngoài, không xây upload mới.
- **Bảo vệ video theo giờ/tua** — không kiểm tra giả, không ghi nhật ký từng lần tua.
- **Giao diện nâng cao** — Không thêm thanh tiến độ, bookmark, ghi chú (phase sau).

---

## 3. Người dùng

| Vai trò | Nhu cầu | Lợi ích |
|---|---|---|
| **Học viên** | Làm bài học thật → xem kết quả chính xác → biết mình tiến bộ hay còn yếu | Động lực cao; lộ trình điều chỉnh sát nhu cầu. |
| **Giáo vụ / Admin nội dung** | Gán câu hỏi curated, thẻ, video vào bài học (qua UI hoặc import) → tạo bộ nội dung chuẩn | Nội dung kiểm soát, tránh bài học "rỗng". |
| **Lãnh đạo / Founder** | Xem dữ liệu tiến bộ = chỉ tiêu xây dựng lộ trình cá nhân hoá → chứng minh giá trị platform | ROI cao: học viên học lâu hơn, kết quả tốt hơn. |

---

## 4. Tổng quan

Mỗi bài học trên bản đồ lộ trình sẽ là **một trong 3 loại**:

### **(A) Bài trắc nghiệm curated (MCQ_QUIZ)**
- **Admin soạn:** chọn 5-10 câu hỏi từ kho Assessment, gán vào bài học.
- **Học viên làm:** nhìn thấy câu hỏi + 4 lựa chọn, chọn đáp án, submit.
- **Hệ thống:** chấm server (không lộ đáp án), trả kết quả "Đúng 4/5 câu → Đạt 0.8 điểm → Mở bài tiếp theo".
- **Ví dụ:** "5 câu từ vựng IELTS A2 về du lịch → học viên đúng 4 → có thể học chuyên đề B1 tiếp".

### **(B) Thẻ ghi nhớ hệ thống (FLASHCARD_DECK)**
- **Admin soạn:** bộ 20-30 thẻ term ↔ definition được kiểm duyệt (VD: từ vựng IELTS, phát âm, phrasal verbs).
- **Học viên làm:** xem một thẻ → xem định nghĩa/ví dụ → tự đánh giá "Quên", "Khó", "Bình thường", "Dễ" → chuyển thẻ tiếp.
- **Hệ thống:** chấm đơn giản (% thẻ "Bình thường" trở lên), trả kết quả "18/20 thẻ tốt → Đạt → Mở bài tiếp".
- **Ví dụ:** "Bộ 20 từ vựng IELTS B1 → học viên tự đánh giá → 18 thẻ tốt → có thể nâng lên B1+".

### **(C) Video bài giảng (VIDEO_LESSON)**
- **Admin soạn:** gán video (MP4 trên server hoặc YouTube) vào bài học + yêu cầu tối thiểu xem 80%.
- **Học viên làm:** xem video → báo cáo "Tôi đã xem 85%" (hệ thống kiểm tra client-side).
- **Hệ thống:** chấm (85% ≥ 80% → Đạt → Mở bài tiếp).
- **Ví dụ:** "Video 10 phút về thì Quá khứ Hoàn thành → học viên xem 85% → mở bài tập tiếp".

---

## 5. Quy tắc dễ hiểu

### **Các bước chung cho cả 3 loại**

```
Học viên START → Hệ thống load nội dung (câu hỏi/thẻ/video) & ràng buộc 1 lần
                ↓
             LÀMM BÀI (gửi đáp án/rating/watchedRatio)
                ↓
             SUBMIT → Hệ thống CHẤM SERVER (không tin client)
                ↓
             KẾT QUẢ (score, pass/fail, unlock bài tiếp)
```

### **Cơ chế chống gian lận (3 tầng)**

1. **Ẩn đáp án** — Trắc nghiệm: đáp án **KHÔNG gửi client**. Thẻ: chỉ dùng thẻ hệ thống (private deck bị chặn). Video: không có đáp án để gian.
2. **Ràng buộc 1 lần** — Mỗi lần `START`, hệ thống tạo 1 **danh sách câu/thẻ ràng buộc**. Nếu học viên reply lần 2 (cache browser, mạng lag), hệ thống **kiểm tra nonce** → nếu cùng lần, trả kết quả cũ (không chấm lại, không thay đổi điểm).
3. **Chấm server** — Tất cả logic chấm (so đáp án, tính % thẻ tốt, check % video) đều ở server. Client chỉ gửi dữ liệu thô (choice, rating, watchedRatio) — **không gửi "correct:true"** để fake điểm.

### **Nội dung curated ≠ random**

- Mỗi bài học gắn **số định (fixed) câu/thẻ** — lần 2 học viên start, thấy **cùng bộ câu, cùng thứ tự** (không random, không shuffle).
- **Lợi ích:** Admin kiểm soát độ khó, học viên lần 2 không copy đáp án từ bạn (câu không đổi nhưng **chấm server**).

### **Guardrail Gating**

- Thẻ & video **KHÔNG được dùng làm "yêu cầu tiên quyết cứng"** (ví dụ: "Phải hoàn thành video này thì mới mở bài tiếp").
  - **Lý do:** Tự đánh giá (thẻ) + self-report (video) không đủ tin cậy để **đảm bảo** kiến thức. Chỉ dùng cho các bài **Thực hành / Mở rộng** (pass qua nhưng không khóa bài kế).
  - **Trắc nghiệm OK:** Chấm server, kiểm tra chắc chắn → có thể làm yêu cầu tiên quyết cứng.

---

## 6. Yêu cầu chức năng (SRS)

### **Nhóm A — Trắc nghiệm (MCQ_QUIZ)**

- **FR-A1** Học viên **start bài trắc nghiệm** → thấy 5-10 câu hỏi đúng thứ tự (admin đã chọn), không thấy đáp án đúng.
- **FR-A2** Học viên **chọn đáp án cho mỗi câu** → submit → hệ thống **chấm server** (so với đáp án lưu sẵn) → trả "Đúng 4/5 câu (80%) → Đạt".
- **FR-A3** Nếu **bài học có < 3 câu hỏi** (rỗng hoặc hỏng), học viên nhìn thấy lỗi "Bài học này chưa có câu hỏi" (fallback sang bài giả để không bế tắc).
- **FR-A4** Học viên **submit 2 lần cùng lần** (mạng lag, refresh, ...) → lần 1 chấm được 0.8 → lần 2 hệ thống nhận ra "cùng lần, cùng submit nonce" → **trả kết quả cũ, không chấm lại**.

### **Nhóm B — Thẻ ghi nhớ (FLASHCARD_DECK)**

- **FR-B1** Học viên **start bộ thẻ** → thấy 20-30 thẻ hệ thống (admin gán vào) + tính năng tự đánh giá (4 nút: "Quên", "Khó", "Bình thường", "Dễ").
- **FR-B2** Học viên **rating từng thẻ** → submit → hệ thống **chấm** (% thẻ "Bình thường" trở lên) → trả "18/20 thẻ tốt (90%) → Đạt".
- **FR-B3** Nếu admin **gán thẻ private** (thuộc học viên khác), học viên submit → hệ thống **chặn** "Thẻ này không được phép dùng trên map".
- **FR-B4** Học viên **submit thiếu** (chỉ rating 15/20 thẻ) → hệ thống xem 5 thẻ còn lại là "miss" → tính "15/20 tốt" → có thể fail.
- **FR-B5** Bài học thẻ **KHÔNG được dùng làm yêu cầu tiên quyết cứng** (publish bị chặn với cảnh báo).

### **Nhóm C — Video bài giảng (VIDEO_LESSON)**

- **FR-C1** Học viên **start video** → thấy link video + "Xem tối thiểu 80%".
- **FR-C2** Nếu video **trên server nội bộ**, hệ thống **tự động tạo link an toàn** (expires sau 6 giờ). Nếu **YouTube/bên ngoài**, link truyền thẳng.
- **FR-C3** Học viên **xem video** → báo cáo "Tôi đã xem 85%" → submit → hệ thống **kiểm tra** (85% ≥ 80% → Đạt).
- **FR-C4** Nếu video **hỏng hoặc không tồn tại**, học viên thấy lỗi "Video không khả dụng" (fallback sang bài giả).
- **FR-C5** Bài học video **KHÔNG được dùng làm yêu cầu tiên quyết cứng** (publish bị chặn).

### **Nhóm D — Quản lý nội dung (Admin)**

- **FR-D1** Giáo vụ **gán câu hỏi, thẻ, video vào bài học** (qua form edit node hoặc file import).
- **FR-D2** Giáo vụ **click "Kiểm tra"** trước khi xuất bản bài video → hệ thống **xác minh video tồn tại** (nếu nội bộ) hoặc **format URL đúng** (nếu YouTube).
- **FR-D3** Khi giáo vụ **publish bản đồ**, hệ thống **kiểm tra toàn bộ**:
  - Trắc nghiệm: ≥ 3 câu hỏi, tất cả loại MCQ.
  - Thẻ: phải là deck hệ thống.
  - Video: link hợp lệ, file tồn tại (nếu nội bộ).
  - Guardrail: thẻ/video không là yêu cầu tiên quyết cứng.
  - Nếu lỗi → publish fail, yêu cầu sửa.

### **Nhóm E — Cập nhật hồ sơ năng lực**

- **FR-E1** Sau khi học viên hoàn thành bài (MCQ/thẻ/video đạt), hệ thống **cập nhật hồ sơ** (thêm mastery signal) → lộ trình cá nhân hóa **tự điều chỉnh** (thêm/bớt/sắp xếp bài tiếp).

---

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (An toàn dữ liệu):** Đáp án, thẻ private, chọn câu phải ẩn từ client → chỉ server có quyền chấm.
- **NFR-2 (Không thể fake):** Mỗi submit phải có **token nonce + HMAC** để chứng thực → không thể giả mạo request client.
- **NFR-3 (Tái tạo được):** Nếu mạng gián đoạn hoặc client crash, submit lại cùng nonce → hệ thống **không double-score**, trả kết quả cũ.
- **NFR-4 (Hiệu suất):** Start/submit node phải trả trong < 1 giây (load nội dung + ràng buộc).
- **NFR-5 (Backward compatibility):** Nếu bài học **chưa có nội dung** (admin chưa gán), fallback sang bài giả (SubmitSimulator cũ) để học viên khỏi bế tắc.

---

## 8. Luồng người dùng (tóm tắt)

### **Kịch bản 1: Học viên làm bài trắc nghiệm thật**

1. Học viên mở bản đồ → click vào bài "Từ vựng IELTS A2 Unit 3".
2. Hệ thống load 5 câu hỏi từ kho, ràng buộc trên server (browser không thấy).
3. Học viên thấy 5 câu + 4 lựa chọn (không thấy đáp án đúng).
4. Học viên chọn lần lượt: 1A, 2B, 3C, 4B, 5D → submit.
5. Hệ thống chấm: câu 1,2,4 đúng; câu 3,5 sai → 60% → fail.
6. Kết quả: "Bạn trả lời đúng 3/5 câu. Còn yếu phần từ vựng này. Vui lòng luyện lại."
7. Học viên **có thể làm lại** (server không chặn) → hệ thống chấm lần 2 (tính điểm mới, cập nhật lộ trình).

### **Kịch bản 2: Học viên làm bộ thẻ ghi nhớ**

1. Học viên click vào bài "Phrasal Verbs IELTS B1".
2. Hệ thống load 25 thẻ hệ thống.
3. Học viên lật thẻ 1: "Go on" → định nghĩa "Continue" → tự đánh giá "Dễ" → thẻ tiếp.
4. Cứ như vậy 25 thẻ (5 phút).
5. Submit: 20 thẻ "Dễ"/"Bình thường", 5 thẻ "Quên" → 80% → pass.
6. Kết quả: "Bạn nắm vững 20/25 phrasal verbs. Tiếp tục với chuyên đề tiếp."
7. Lộ trình **tự điều chỉnh**: có thể skip bài cơ bản, vào thẳng chuyên đề nâng cao.

### **Kịch bản 3: Học viên xem video bài giảng**

1. Học viên click vào bài "Video: Thì Quá khứ Đơn vs Quá khứ Tiếp Diễn" (8 phút).
2. Hệ thống phát video + yêu cầu "Xem tối thiểu 80%".
3. Học viên xem video → 6 phút 30 giây (81%) → submit.
4. Hệ thống chấm: 81% ≥ 80% → pass.
5. Kết quả: "Bạn đã hoàn thành video. Bài kiểm tra ngay sau video."
6. Lộ trình **mở khóa bài tiếp**: bài kiểm tra tiến độ (post-video test).

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

Để xác nhận feature **hoạt động đúng**:

### **Đối với Trắc nghiệm**
- ✅ Học viên mở bài → thấy 5+ câu hỏi (không thấy đáp án đúng).
- ✅ Chọn đáp án → submit → hệ thống chấm **server** (so đáp án lưu sẵn).
- ✅ Kết quả chính xác: 4/5 đúng → score 80% → pass (nếu threshold 80%).
- ✅ Nếu bài < 3 câu → lỗi 409, fallback sang bài giả (không crash).
- ✅ Submit 2 lần cùng nonce → lần 2 không double-score (kết quả không đổi).

### **Đối với Thẻ ghi nhớ**
- ✅ Học viên mở bài → thấy 20+ thẻ, có nút tự đánh giá.
- ✅ Rating thẻ → submit → hệ thống chấm (% thẻ "Bình thường" trở lên).
- ✅ Nếu admin gán thẻ private → lỗi 409 (chặn rò rỉ).
- ✅ Thiếu rating (chỉ 15/20) → tính "miss" → có thể fail.
- ✅ Bài thẻ publish với yêu cầu tiên quyết cứng → lỗi 422 (publish fail).

### **Đối với Video**
- ✅ Học viên mở bài → thấy video link + yêu cầu % xem.
- ✅ Xem video → báo cáo % → submit → hệ thống chấm (% ≥ threshold → pass).
- ✅ Video nội bộ → link có chữ ký (expire 6h), YouTube → URL direct.
- ✅ Video không tồn tại → lỗi 404 (không crash).
- ✅ Bài video publish với yêu cầu tiên quyết cứng → lỗi 422.

### **Đối với Admin**
- ✅ Giáo vụ gán câu/thẻ/video via UI → lưu thành công.
- ✅ Giáo vụ click "Kiểm tra" video → xác nhận tồn tại ✅ hoặc lỗi ❌.
- ✅ Giáo vụ publish bản đồ → hệ thống validate tất cả node, chặn nếu lỗi.

### **Đối với Tiến bộ Học viên**
- ✅ Sau khi hoàn thành bài (MCQ/thẻ/video pass), hệ thống **cập nhật hồ sơ** (thêm mastery signal).
- ✅ Lộ trình cá nhân hóa **tự điều chỉnh** (thêm chuyên đề mới phù hợp, bỏ bài đã thành thạo).

---

## 10. Ngoài phạm vi

- **Sinh nội dung động:** Random câu hỏi từ kho, shuffle thẻ → thuộc loại bài "Test Out" (flow tách biệt), không phải 3 player này.
- **Lưu trữ Spaced Repetition (SRS) Anki:** Map node chỉ tính điểm 1 lần, không persist trạng thái thẻ.
- **Upload media server:** Video dùng self-host sẵn hoặc YouTube, không xây upload mới.
- **Bảo vệ video (seek-protection):** Pure self-report % xem, không verify giả (ungainability là tính năng riêng).
- **Giao diện nâng cao:** Thanh tiến độ, bookmark, ghi chú → Phase 4 hoặc sau.
- **Kiểm tra biến chứng:** Merge 2 player type (VD: MCQ + video) vào 1 bài → Phase sau.

---

## 11. Thuật ngữ (cho người không chuyên)

- **Bài học (Lesson):** Đơn vị học tập cơ bản; ví dụ: "5 câu từ vựng IELTS A2".
- **Nút (Node):** Một điểm trên bản đồ lộ trình; có thể là bài học, hoặc là một milestone (kiểm tra, video).
- **Bản đồ lộ trình (Learning Map):** Biểu đồ trực quan hiển thị các bài học liên kết; học viên đi từ trái sang phải, từ đơn giản sang phức tạp.
- **Curated (Soạn / Chuẩn bị sẵn):** Admin đã chọn cụ thể những câu hỏi, thẻ, video nào để gán vào bài học (không random).
- **Ràng buộc (Binding):** Hệ thống "khóa" danh sách câu/thẻ sau khi học viên start, để replay hay cache không tái dùng.
- **Nonce (unique token):** Mã duy nhất cho mỗi lần start/submit; dùng để kiểm tra replay (lần 2 submit cùng nonce = không chấm lại).
- **HMAC (chữ ký server):** Mã chứng thực request từ client (đảm bảo không bị giả mạo trên đường network).
- **Threshold (ngưỡng):** Điểm tối thiểu để pass; ví dụ: 80% đúng = pass, 79% = fail.
- **Mastery (Thành thạo):** Mức nắm vững một kỹ năng/bài học; signal này giúp hệ thống biết học viên có thể bỏ qua bài dễ, vào chuyên đề khó tiếp.
- **Gating Policy (Chính sách khóa):** Quy tắc để mở bài tiếp. "Cứng" = phải hoàn thành bài này (score ≥ threshold) mới mở. "Mềm" = khuyến khích nhưng không bắt buộc.
- **Hardening (Siết chặt):** Tăng cường bảo mật; ví dụ: Bind userID vào link video (hiện tại không làm).
- **Throwaway grade:** Điểm số chỉ dùng tính mastery 1 lần, không lưu trữ dài hạn.
- **FSRS (Flashcard Spaced Repetition System):** Thuật toán lên lịch thẻ ghi nhớ; ở đây, chỉ dùng version đơn giản (chấm % thẻ "tốt").
- **Fallback:** Biện pháp dự phòng; nếu nội dung thật không khả dụng, hệ thống dùng bài giả (SubmitSimulator).
- **Token (Mã xác thực):** Chuỗi dữ liệu chứng minh danh tính / quyền hạn của người dùng hoặc session.
- **Idempotent / Idempotency (Lặp lại mà không đổi kết quả):** Nếu submit 2 lần cùng nonce, kết quả giống nhau (không chấm lại, không thay đổi điểm).
- **Ownertype (Loại chủ sở hữu):** Thẻ "system" = do hệ thống/giáo vụ soạn; "user" = do học viên tạo riêng.
- **Signed URL (URL có chữ ký):** Link video được mã hóa + expire sau 6h, chỉ server có thể tạo (chống ăn cắp).
- **Adaptive Path / Lộ trình thích nghi:** Lộ trình được cá nhân hóa theo năng lực học viên; dùng hồ sơ năng lực + nội dung curated để xây.
- **Proficiency (Trình độ):** Mức độ nắm vững ngôn ngữ; ví dụ: A1 (sơ cấp), B1 (trung cấp), C1 (cao).
- **CEFR (Khung trình độ châu Âu):** A1–C2 là các mức chuẩn; A2 = Còn sơ cấp, B1 = Trung cấp, B2 = Trên trung cấp.
