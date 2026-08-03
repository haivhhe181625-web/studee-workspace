<!-- Tiếng Việt — PRD/SRS cho "Bài quiz & flashcard được chấm điểm THẬT". Viết cho người KHÔNG chuyên kỹ thuật.
     Bản kỹ thuật/framing: specs/course-graded-exercises/spec.md + acceptance.md. -->
# PRD — Bài Quiz & Thẻ Flashcard Được Chấm Điểm THẬT

- **Ngày:** 2026-08-03 · **Trạng thái:** Đã xây dựng (hồi tố spec) · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật/khung:** `specs/course-graded-exercises/spec.md` (design) + `acceptance.md` (tiêu chí)
- **Phụ thuộc:** bài quiz cần ngân hàng câu hỏi đã có (Assessment Question Bank, ~558 câu).

> Tài liệu mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `specs/course-graded-exercises/spec.md`.

---

## 1. Mục đích & bối cảnh

Hiện tại, các quiz và thẻ flashcard trong bài học chỉ là **bài tập luyện tập không tính điểm** — học viên làm để ôn, nhưng kết quả không ảnh hưởng đến đánh giá năng lực hay tiến độ học. Điều này không thích hợp với học viên muốn biết **đúng mình mạnh/yếu ở đâu**, hay với các tổ chức muốn **dùng kết quả bài tập để đo năng lực thực sự** của học viên.

**Mục tiêu:** tạo **bài quiz & thẻ flashcard có chấm điểm thực (graded exercises)** — tức kết quả **đem vào đánh giá năng lực học viên**, giúp:
- **Học viên:** biết mình đạt hay chưa đạt mục tiêu bài học (ngưỡng ≥80%); thấy rõ điểm yếu từ kết quả.
- **Giáo vụ:** lựa chọn những câu hỏi phù hợp, soạn quiz một lần rồi **không phải lo việc thay đổi sau** (câu hỏi được đóng băng).
- **Hệ thống:** thu thập dữ liệu năng lực thực từ bài tập, cập nhật **hồ sơ năng lực** và **lộ trình thích nghi** của học viên.

---

## 2. Phạm vi

**TRONG phạm vi:**
1. **Bài quiz đa lựa chọn (MCQ)** — từ ngân hàng câu hỏi hiện có, chấm điểm **tự động server-side** (không dùng con người chấm).
2. **Thẻ flashcard từ vựng** — học viên review thẻ, khi xem hết thẻ → bài học đánh dấu hoàn thành.
3. **Quản lý quiz bởi giáo vụ** — soạn quiz bằng cách **chọn từng câu hỏi từ ngân hàng**, đặt ngưỡng đạt (mặc định 80%), rồi **publish (công bố)** để lock (không thay đổi được nữa).
4. **Chấm điểm & ghi nhận kết quả** — học viên làm bài → hệ thống chấm điểm tự động → ghi lại điểm + cập nhật **hồ sơ năng lực** của học viên.
5. **Tích hợp vào tiến độ bài học** — quiz/flashcard đạt yêu cầu → bài học được đánh dấu "hoàn thành".

**NGOÀI phạm vi (để sau):**
- Tự sinh câu hỏi bằng AI.
- Câu hỏi tự luận / viết / nói (chỉ tự chấm đa lựa chọn, cloze, điền chỗ trống v.v.).
- Flashcard toàn cầu (SRS = spaced repetition system) — flashcard ở đây **độc lập** từng bài học.
- Tái publish quiz (một khi publish rồi, chỉ tạo draft mới, không sửa cái đã publish).

---

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên** | Vào bài học, làm quiz để kiểm tra mình + flashcard để ôn từ vựng; nhận điểm & phản hồi về điểm yếu. |
| **Giáo vụ (admin nội dung)** | Soạn quiz bằng chọn câu từ ngân hàng, publish để khóa cấu hình; sau khi publish, không phải lo lo câu hỏi sẽ bị sửa bất thình lị. |
| **Hệ thống (backend)** | Chấm điểm tự động, ghi nhận kết quả, cập nhật dữ liệu năng lực để phục vụ lộ trình thích nghi & phân tích học viên. |

---

## 4. Tổng quan

### (A) Bài Quiz — Luyện tập có điểm

**Là gì?** Một bộ câu hỏi đa lựa chọn được giáo vụ chọn từ ngân hàng, gắn vào bài học để học viên làm & nhận điểm.

**Cách hoạt động:**
1. **Giáo vụ soạn:** mở admin, chọn từng câu hỏi từ ngân hàng (~558 câu có sẵn) → tạo quiz mới (draft).
2. **Giáo vụ publish:** xem lại câu hỏi đã chọn, bấm "Công bố" → quiz **lock** (câu hỏi không thể thay đổi nữa).
3. **Học viên làm quiz:** vào bài học, nhìn câu hỏi, chọn đáp án, submit → nhận **điểm = số câu đúng ÷ tổng câu** (vd: 4/5 = 80%).
4. **Chấm điểm tự động:** hệ thống chấm lại server-side (đáp án được **giữ bí mật**, không gửi xuống client).
5. **Kết quả:** nếu điểm ≥ 80% → "đạt"; nếu < 80% → "chưa đạt" → học viên có thể làm lại.
6. **Cập nhật năng lực:** mỗi câu hỏi được gắn **kỹ năng** (vd: Nghe, Đọc, Ngữ pháp) → kết quả quiz **cập nhật** vào hồ sơ năng lực (dùng cho lộ trình thích nghi).

### (B) Thẻ Flashcard — Ôn từ vựng

**Là gì?** Một bộ thẻ học từ vựng (mặt trước = từ, mặt sau = nghĩa) được gắn vào bài học.

**Cách hoạt động:**
1. **Học viên mở flashcard:** vào bài học, kéo lướt thẻ, ấn "biết rồi" khi thuộc từ.
2. **Hoàn thành:** khi xem hết mọi thẻ → bài học được **đánh dấu hoàn thành**.
3. **Không feed năng lực:** không giống quiz, flashcard **không cập nhật** hồ sơ năng lực (chỉ ghi nhận "đã ôn").

---

## 5. Quy tắc chấm điểm (dễ hiểu)

### Ngưỡng đạt & điểm

- **Mặc định:** 80% → đạt (tức 4/5 câu đúng trở lên).
- **Giáo vụ có thể sửa:** khi soạn quiz, có thể đặt ngưỡng khác (vd: 70%, 90%), nhưng chỉ được sửa **trước khi publish**; **sau publish không sửa được nữa**.

### Cách tính điểm

- **Công thức:** Điểm = (Số câu đúng) ÷ (Tổng số câu) × 100%
- **Nguyên tắc quan trọng:** mẫu số **luôn là tổng số câu đã publish**, không phải số câu mà học viên nộp → tránh học viên bỏ câu khó để tăng điểm.
- **Câu không trả lời:** cũng tính là **sai** (điểm 0 cho câu đó).

### Học lại (Re-attempt)

- **Nếu chưa đạt:** học viên làm lại → **lần nộp mới nhất sẽ thay thế kết quả cũ** (best-score principle).
- **Ví dụ:** lần 1 = 60%, lần 2 = 85% → kết quả ghi nhận là 85%, lần 1 bị bỏ qua.

---

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Quiz (Học viên)**
- **FR-A1** Khi vào bài học, nếu bài có quiz → học viên **xem danh sách câu hỏi** (không có đáp án, chỉ thấy câu hỏi, các tùy chọn/lựa chọn).
- **FR-A2** Học viên **trả lời từng câu**, sau đó bấm **Submit** → hệ thống **chấm điểm tự động** tại server (không công khai đáp án tới client) → trả về **điểm % + kết quả từng câu** (đúng/sai).
- **FR-A3** Nếu **chưa đạt** → học viên có nút **"Làm lại"** để tái làm quiz; nếu **đạt** → bài học **đánh dấu hoàn thành**.
- **FR-A4** **Kết quả được ghi lại:** lần nộp nào sau cùng sẽ được dùng để tính điểm cuối cùng (không tính lần 1, 2, 3... lần cũ).

**Nhóm B — Quiz (Giáo vụ / Admin)**
- **FR-B1** Giáo vụ **tạo quiz mới** (draft): chọn từng câu hỏi từ ngân hàng, đặt tên quiz + ngưỡng đạt (mặc định 80%).
- **FR-B2** Giáo vụ **xem lại draft**: danh sách câu được chọn, có thể **xoá/thêm** câu, **sửa ngưỡng đạt**.
- **FR-B3** Giáo vụ **Publish (công bố) quiz:** lúc này, quiz **lock** — các câu hỏi được **đóng băng** (nếu ai đó sửa/xoá câu hỏi ở ngân hàng sau này, hệ thống sẽ cảnh báo "câu đang dùng ở quiz đã publish, không thể sửa").
- **FR-B4** Nếu công bố thất bại (vd: 1 câu bị xoá khỏi ngân hàng): hệ thống **thông báo lỗi cụ thể**, quiz **không publish**, giáo vụ phải **sửa draft rồi publish lại**.

**Nhóm C — Flashcard (Học viên)**
- **FR-C1** Khi vào bài học, nếu bài có flashcard → học viên **xem thẻ** (mặt trước = từ, mặt sau = nghĩa).
- **FR-C2** Học viên **kéo lướt thẻ**, mỗi thẻ ấn nút "Biết rồi" → hệ thống **ghi nhận** thẻ đã xem.
- **FR-C3** Khi **xem hết mọi thẻ** → bài học tự động **đánh dấu hoàn thành** (không cần bấm nút khác).
- **FR-C4** Flashcard **không cập nhật hồ sơ năng lực** (không giống quiz), chỉ ghi nhận "đã ôn".

**Nhóm D — Cập nhật năng lực**
- **FR-D1** Mỗi **câu hỏi ở ngân hàng gắn sẵn kỹ năng** (vd: Nghe, Đọc, Ngữ pháp) → khi quiz submit, **mỗi câu đúng/sai sẽ cập nhật** kỹ năng tương ứng vào **hồ sơ năng lực học viên**.
- **FR-D2** Hồ sơ năng lực **luôn dùng kết quả lần nộp cuối cùng** (lần cũ bị xoá khi có lần nộp mới).

**Nhóm E — Bảo mật & Toàn vẹn**
- **FR-E1** Đáp án **KHÔNG bao giờ** được gửi xuống cho học viên; chỉ hệ thống mới biết đáp án → chấm server-side.
- **FR-E2** Khi giáo vụ cố gắng **sửa/xoá câu hỏi** đã được publish ở quiz → hệ thống **từ chối**, báo lỗi "câu đang dùng ở quiz publish, vui lòng liên hệ admin để xử lý".

---

## 7. Yêu cầu phi chức năng (SRS)

| Loại | Tiêu chí |
|---|---|
| **Bảo mật** | Đáp án LUÔN giữ server-side, không gửi xuống client (select=false) |
| **Bảo mật** | Chỉ học viên đã enrolled + course published mới truy cập quiz/flashcard |
| **Bảo mật** | Giáo vụ cần quyền 'lesson-quiz:write' để tạo/sửa/publish quiz |
| **Hiệu năng** | Chấm điểm phải tính toán nhanh (< 1 giây cho 1 quiz ~20 câu) |
| **Hiệu năng** | Cập nhật năng lực phải batch (không insert 1 sự kiện 1 lần) |
| **Độ tin cậy** | Ngân hàng câu hỏi phải stable (~558 câu có sẵn, không thay đổi liên tục) |
| **Độ tin cậy** | Khi publish quiz, TOÀN BỘ câu phải hợp lệ (all-or-nothing validation); nếu 1 câu fail → không publish được |
| **Trải nghiệm** | Câu hỏi phải **xâm nhập (testlet)** nguyên bộ — vd: nếu 1 bài nghe + 3 câu → phải hiển thị cùng nhau, không tách riêng |

---

## 8. Luồng người dùng (tóm tắt)

### Giáo vụ — Soạn & Publish

```
Giáo vụ bấm "Tạo Quiz"
  ↓
Chọn từng câu từ ngân hàng (vd: 5 câu)
  ↓
Đặt ngưỡng đạt (mặc định 80%)
  ↓
Lưu draft
  ↓
Xem lại (tên, câu, ngưỡng)
  ↓
Bấm "Công bố"
  ↓
Hệ thống kiểm tra: mỗi câu có đáp án không? loại câu có tự chấm được không?
  ↓
[Thành công] Quiz lock → học viên có thể làm
[Thất bại] Báo lỗi → giáo vụ sửa rồi publish lại
```

### Học viên — Làm Quiz

```
Vào bài học → thấy quiz
  ↓
Đọc câu hỏi, chọn đáp án (mỗi câu)
  ↓
Bấm Submit
  ↓
Hệ thống chấm → trả về điểm % + kết quả câu
  ↓
[≥80%] "Đạt" → bài học hoàn thành
[<80%] "Chưa đạt" → nút "Làm lại" → có thể tái quiz
  ↓
[Làm lại] Kết quả mới thay thế cái cũ
```

### Học viên — Ôn Flashcard

```
Vào bài học → thấy flashcard
  ↓
Xem thẻ (từ + nghĩa), ấn "Biết rồi"
  ↓
Kéo lướt hết mọi thẻ
  ↓
Hệ thống tự động đánh dấu bài học hoàn thành
```

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

- **Học viên có thể làm quiz** trong bài học, nhận **điểm % tính theo công thức** (số đúng ÷ tổng).
- **Ngưỡng 80%** được **lock khi publish**, giáo vụ không lo thay đổi sau.
- **Câu hỏi ngân hàng được bảo vệ**: nếu công bố quiz rồi mà giáo vụ cố sửa câu → bị từ chối.
- **Kết quả quiz cập nhật năng lực** (tức hồ sơ năng lực & lộ trình thích nghi phản ánh kết quả này).
- **Flashcard hoạt động độc lập** (xem hết → bài học xong, không feed năng lực).
- **Học lại best-score**: lần nộp mới nhất là kết quả cuối cùng; không tính lần cũ.

---

## 10. Ngoài phạm vi

- **Tự sinh câu hỏi bằng AI** — hiện chỉ **curate (chọn tay)** từ ngân hàng.
- **Câu tự luận / viết / nói** — chỉ tự chấm được (MCQ, cloze, fill_blank, v.v.); writing/speaking chấm riêng (future phase).
- **Flashcard toàn cầu (SRS)** — flashcard ở đây **độc lập** từng bài; toàn cầu SRS để sau.
- **Tái-publish (re-publish)** — một khi đã phát hành, bộ câu hỏi bị "đóng băng"; muốn đổi thì tạo quiz nháp mới, không sửa trực tiếp bộ câu của quiz đã phát hành.
- **Checkpoint / bài kiểm tra cuối kỳ** — thuộc feature riêng; quiz ở đây là **lesson-level** (trong bài học).

---

## 11. Thuật ngữ (cho người không chuyên)

- **Quiz (Bài kiểm tra):** bộ câu hỏi đa lựa chọn được giáo vụ chọn từ ngân hàng, gắn vào bài học để học viên làm & nhận điểm.
- **Flashcard (Thẻ học):** bộ thẻ từ vựng (mặt trước = từ, mặt sau = nghĩa) gắn vào bài học; ôn nhưng không ảnh hưởng điểm.
- **Graded exercise (Bài tập có chấm điểm):** bài tập tính kết quả vào đánh giá năng lực (khác với bài tập không chấm điểm chỉ dùng luyện tập).
- **Ngân hàng câu hỏi (Question bank):** kho ~558 câu hỏi có sẵn, giáo vụ chọn từ đó để tạo quiz.
- **Publish / Công bố:** hành động "lock" quiz (câu hỏi không thể thay đổi nữa) để học viên bắt đầu làm.
- **Testlet (Cụm bài):** một nhóm câu hỏi liên quan (vd: 1 bài nghe + 3 câu về bài nghe) → **phải hiển thị nguyên bộ** cho học viên.
- **Hồ sơ năng lực (Competency profile):** bản tóm tắt mạnh/yếu ở mỗi kỹ năng (Nghe, Đọc, Ngữ pháp…), cập nhật dần từ kết quả bài tập.
- **Lộ trình thích nghi (Adaptive path):** lộ trình học cá nhân hóa, tự điều chỉnh theo hồ sơ năng lực (feature liên quan).
- **Ngưỡng đạt (Pass threshold):** điểm tối thiểu để "đạt" bài (mặc định 80%; giáo vụ có thể sửa trước publish).
- **Mẫu số cố định (Fixed denominator):** điểm luôn tính theo **tổng số câu được publish**, không phải số câu học viên trả lời → tránh gian lận.
- **Best-score (Điểm tốt nhất):** nếu làm lại quiz, **lần mới thay thế lần cũ** (không cộng dồn).

---

## Liên kết tài liệu kỹ thuật

- **Chi tiết thiết kế:** [`specs/course-graded-exercises/spec.md`](./spec.md)
- **Tiêu chí kiểm thử:** [`specs/course-graded-exercises/acceptance.md`](./acceptance.md)
