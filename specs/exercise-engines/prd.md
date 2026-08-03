<!-- Tiếng Việt — PRD/SRS cho "Exercise Engines" (Bộ engine chấm bài tập). 
     Viết cho người KHÔNG chuyên kỹ thuật.
     Bản kỹ thuật/khung: specs/exercise-engines/design.md
     CHÚ Ý: Nhánh Quiz đã chuyển hướng → specs/course-graded-exercises/spec.md -->

# PRD — Exercise Engines (Bộ Engine Chấm Bài Tập Tương Tác)

- **Ngày:** 2026-08-03 · **Trạng thái:** Nháp để duyệt · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật/khung:** `specs/exercise-engines/design.md` (các quyết định đã chốt)
- **Phụ thuộc:** Tích hợp khi **Phase 4** hoàn thành; phù hợp với bài tập tương tác & đánh giá năng lực.

**📌 TRẠNG THÁI:** 
- ✅ **Nhánh Viết & Nói:** Engine chấm tích hợp trong bài học, cập nhật hồ sơ năng lực (ĐANG PHÁT TRIỂN).
- ⚠️ **Nhánh Quiz:** Đã được thay thế bằng mô hình **LessonQuiz curated** tích hợp trong khóa học (xem `specs/course-graded-exercises/`). Nếu muốn chấm quiz, tham khảo [Chi tiết Quiz tại Course Graded Exercises](../course-graded-exercises/spec.md).

> Tài liệu mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `specs/exercise-engines/design.md`.

---

## 1. Mục đích & bối cảnh

Hiện nay, bài học chỉ có nội dung lý thuyết và bài tập tương tác không chấm (nói chuyện tự do, nghe hiểu…). Để **đo được năng lực học viên một cách chính xác**, cần phải **chấm điểm các bài tập nhất định** —
- **Viết** (essay/paragraph) — bạn viết bao nhiêu từ? Cấu trúc + ngữ pháp tốt không? 
- **Nói** (đơn thoại/đọc to) — phát âm chuẩn không? Lưu loát không? 

Kết quả chấm này sẽ được **đưa vào hồ sơ năng lực** của học viên, từ đó hệ thống có dữ liệu để **đề xuất lộ trình học tích nghi** (xem PRD Lộ trình thích nghi) hoặc **gợi ý bài tập thích ứng** tiếp theo.

## 2. Phạm vi

**TRONG phạm vi:**
1. **Engine Viết** — chấm điểm bài viết của học viên (dữ liệu + cấu trúc + logic), cập nhật điểm mạnh/yếu.
2. **Engine Nói** — chấm điểm bài nói học viên (phát âm, tính lưu loát, kỹ thuật), cập nhật điểm mạnh/yếu.
3. **Tích hợp bài tập** — khi có bài tập loại "viết" hoặc "nói" trong bài học, bài tập đó sẽ được chấm server-side (chặn rò rỉ đáp án) và kết quả đưa vào hồ sơ năng lực.
4. **Tiến độ bài học** — hoàn thành một bài tập = hoàn thành phần đó (dùng chung với các bài tập khác).

**NGOÀI phạm vi:**
- **Quiz/Trắc nghiệm:** dùng mô hình riêng tại [Course Graded Exercises](../course-graded-exercises/) (LessonQuiz curated). KHÔNG quản lý quiz trong engine này.
- Sinh nội dung/bài tập bằng AI → để sau.
- Lập lịch học chi tiết (ngày/giờ).
- So sánh nhiều thuật toán chấm điểm.

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên** | Làm bài viết/nói → nhận điểm ngay; điểm được thêm vào hồ sơ năng lực; theo dõi tiến độ. |
| **Giáo viên/Admin** | Tạo bài viết/nói (đặt đề bài, rubric chấm điểm, ngưỡng đạt); publish; xem kết quả học viên từng bài. |

## 4. Tổng quan

- **(A) Engine Viết:** bài viết được **chấm server-side** dựa trên **rubric có sẵn** (đánh giá dữ liệu, cấu trúc, ngữ pháp…). Kết quả trả về một **điểm band** (vd 5.5/9). Ngưỡng đạt (`passBand`) được gắn trên bộ bài tập; nếu band ≥ ngưỡng → đạt.
  
- **(B) Engine Nói:** bài nói được **chuyển đổi thành text** (ASR — Automatic Speech Recognition) rồi **chấm server-side** theo **rubric** (phát âm, lưu loát, độ chính xác…). Kết quả cũng là **band**, so sánh với ngưỡng.

- **(C) Tiến độ:** khi học viên nộp một bài viết/nói:
  - Nếu band ≥ `masteryBand` (ngưỡng thành thạo, vd 7.0) → trạng thái `mastered`.
  - Nếu band ≥ `passBand` (ngưỡng đạt, vd 5.5) → trạng thái `completed`.
  - Nếu band < `passBand` → trạng thái `in_progress` (có thể thử lại).

- **(D) Hồ sơ năng lực:** mỗi kết quả viết/nói sẽ tạo một **sự kiện học tập** (learning event) mô tả kỹ năng nào được cải thiện/yếu để hệ thống cập nhật hồ sơ.

## 5. Quy tắc chấm điểm (dễ hiểu)

### Bài Viết
Mỗi bài viết được chấm theo **rubric** — một bảng tiêu chí:
- **Dữ liệu (Data):** có trả lời đề bài, có ví dụ cụ thể không?
- **Tổ chức (Organization):** bài viết có logic? Lối chuyển tiếp mịn?
- **Ngữ pháp & Từ vựng:** câu câu có đúng? Từ vựng phong phú?
- **Độ dài:** viết đủ từ không?

Mỗi tiêu chí được **đánh giá từ 1–9 (band)**, rồi **trung bình** → kết quả cuối cùng (band).

### Bài Nói
- **Phát âm:** cách phát âm chuẩn bao nhiêu %?
- **Lưu loát:** nói mượt mà, không ngập ngừng?
- **Độ chính xác:** ngữ pháp/từ vựng nói đúng?
- **Độ dài:** nói đủ lâu?

Tương tự, trung bình các tiêu chí → band cuối cùng.

### Ngưỡng Đạt & Thành Thạo
Mỗi bộ bài tập có **2 ngưỡng:**
- **`passBand`** (vd 5.5): học viên đạt yêu cầu → `completed`.
- **`masteryBand`** (vd 7.0): học viên thành thạo → `mastered` (không cần học lại).

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Bộ Bài Tập Viết**
- **FR-A1** Admin có thể **tạo một bộ bài viết**: đặt đề bài, chọn rubric chấm, gắn kỹ năng (vd Viết), gắn trình độ CEFR (vd B1), đặt `passBand` và `masteryBand`.
- **FR-A2** Admin **publish** bộ bài viết; sau khi publish, bộ bài bị **khóa** (không sửa đề bài/rubric). Chỉ có thể đánh dấu là "không dùng nữa" (retire).
- **FR-A3** Học viên **xem đề bài** của bộ bài viết (ẩn rubric, chỉ show tiêu chí chung để biết sẽ chấm như thế nào).
- **FR-A4** Học viên **nộp bài viết** (text); server **gửi đến AI scoring engine** → nhận band.
- **FR-A5** Kết quả chấm được **trả về learner** (band + giải thích từng tiêu chí sao lại được band này); **cập nhật `ExerciseProgress`** (trạng thái completed/mastered/in_progress).
- **FR-A6** Kết quả **sinh learning event** để cập nhật hồ sơ năng lực của học viên.

**Nhóm B — Bộ Bài Tập Nói**
- **FR-B1** Admin tạo một **bộ bài nói**: đặt đề bài, loại nói (đơn thoại/đọc to), chọn rubric, gắn kỹ năng (Nói), gắn CEFR, đặt `passBand` và `masteryBand`.
- **FR-B2** Admin **publish** bộ bài nói; bị khóa, không sửa sau đó.
- **FR-B3** Học viên **xem đề bài** (ẩn rubric chi tiết).
- **FR-B4** Học viên **nộp file audio** (hoặc quay trực tiếp); server **chuyển ASR** → text; **gửi AI scoring** → band.
- **FR-B5** Kết quả trả về (band + text được ASR + giải thích từng tiêu chí).
- **FR-B6** **Cập nhật progress** + **sinh learning event** (tương tự bài viết).

**Nhóm C — Quản Lý Tiến độ Bài Tập**
- **FR-C1** Một bài học có thể chứa **0, 1 hoặc nhiều bài tập** (viết/nói). Tiến độ bài học = mọi bài tập ≥ `completed`.
- **FR-C2** **Tối ưu lần làm:** nếu học viên làm lại bài tập, chỉ **điểm tốt nhất** được giữ (không cộng điểm các lần). Learning event được **sinh lại** dựa trên lần làm tốt nhất (không trùng lặp tín hiệu).
- **FR-C3** **Ghi nhận chung tiến độ:** hoàn thành bài viết/nói trong bài học → tự ghi nhận vào tiến độ khóa (không cần click riêng).

**Nhóm D — Bảo Vệ & Chất Lượng**
- **FR-D1** **Giữ bí mật rubric chi tiết**: client chỉ xem tiêu chí chung (vd "Sẽ chấm Ngữ pháp, Từ vựng…"), không xem trọng số/điểm cụ thể.
- **FR-D2** **Chấm server-side**: band được tính trên server bằng AI; client không làm điều này.
- **FR-D3** **Giới hạn API**: bài viết/nói có giới hạn số lần nộp trên một khóa (quota), để tránh lạm dụng (cost control).
- **FR-D4** **Mẫu số cố định**: nếu admin chỉnh rubric sau khi có học viên làm bài, các bài cũ vẫn được chấm theo rubric cũ (no retroactive changes).

**Nhóm E — Hồ Sơ Năng Lực & Lộ Trình**
- **FR-E1** Mỗi kết quả viết/nói tạo **learning event** (source='lesson', skill='writing'/'speaking', band, timestamp).
- **FR-E2** **Hồ sơ năng lực tự cập nhật** khi có event mới.
- **FR-E3** **Lộ trình thích nghi** có thể dựa vào hồ sơ để **đề xuất bài tập tiếp theo** (để sau — Phase 4.x).

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Độ chính xác):** càng nhiều học viên làm bài, càng nhiều dữ liệu → chấm điểm càng sát thực.
- **NFR-2 (Chi phí):** writing/speaking dùng AI → có chi phí. Cần **quota control** để tránh vượt budget.
- **NFR-3 (Tốc độ):** chấm viết ~1–2 phút (AI); chấm nói ~2–3 phút (ASR + AI). Học viên chấp nhận chờ hoặc xem kết quả sau 5 phút.
- **NFR-4 (Không đụng Quiz):** engine này KHÔNG quản lý quiz. Quiz dùng LessonQuiz model tại course-graded-exercises.
- **NFR-5 (Phụ thuộc dữ liệu):** cần **rubric, skill metadata** được setup chính xác trước khi publish bài tập.

## 8. Luồng người dùng (tóm tắt)

### Học Viên
1. Mở **bài học** → thấy **bài viết/nói** cần làm.
2. Xem **đề bài** (ẩn rubric chi tiết, chỉ xem "sẽ chấm cái gì").
3. **Viết/Nói** bài → nộp.
4. Server chấm (~1–3 phút) → **nhận điểm (band)** + giải thích.
5. Nếu band ≥ `passBand` → `completed`; ≥ `masteryBand` → `mastered`.
6. **Hồ sơ năng lực tự cập nhật** (kỹ năng Viết/Nói tăng/giảm).
7. (Nếu chưa đạt) **Làm lại** bài → chỉ điểm tốt nhất được giữ.

### Admin/Giáo Viên
1. **Tạo bộ bài viết/nói** (đề bài, rubric, ngưỡng).
2. **Publish** → bị khóa.
3. Thêm vào bài học.
4. Theo dõi kết quả học viên (dashboard — để sau).

## 9. Tiêu chí hoàn thành (nghiệm thu)

- ✅ Admin có thể **tạo + publish bộ bài viết/nói** (không sửa sau publish).
- ✅ Học viên **nộp bài viết/nói** → server **chấm + trả band** trong vòng 3–5 phút.
- ✅ **Tiến độ cập nhật** (completed/mastered) sau chấm điểm.
- ✅ **Learning event sinh ra** → hồ sơ năng lực tự thay đổi.
- ✅ **Quiz KHÔNG được quản lý** trong engine này (theo tiêu chí ngoài phạm vi).
- ✅ Làm lại bài → chỉ **điểm tốt nhất** được lưu (không trùng lặp event).

## 10. Ngoài phạm vi

- **Quiz**: dùng LessonQuiz curated (xem `specs/course-graded-exercises/`).
- Sinh nội dung/rubric bằng AI.
- Lập lịch học chi tiết (ngày/giờ).
- Đề xuất bài tập thích ứng (Phase 4.x).
- Mô phỏng đề thi đầy đủ (dùng exam player có sẵn).
- Báo cáo chi tiết cho admin (dashboard — Phase 5).
- Randomize/kiểm soát lộ trình bài tập (để sau).

## 11. Thuật ngữ (cho người không chuyên)

| Thuật ngữ | Ý nghĩa |
|---|---|
| **Engine Viết / Nói** | Bộ công cụ chấm điểm bài viết / bài nói của học viên. |
| **Rubric** | Bảng tiêu chí chấm (vd Ngữ pháp, Từ vựng, Lưu loát…), mỗi tiêu chí có điểm từ 1–9. |
| **Band / Điểm Band** | Điểm từ 1.0 đến 9.0 (vd 5.5, 7.0) dùng để đánh giá chất lượng bài viết/nói. |
| **`passBand`** | Ngưỡng đạt (vd 5.5). Nếu band ≥ passBand → học viên đạt yêu cầu bài tập. |
| **`masteryBand`** | Ngưỡng thành thạo (vd 7.0). Nếu band ≥ masteryBand → học viên thành thạo (không cần học lại). |
| **Trạng thái Bài Tập** | `in_progress` (chưa đạt), `completed` (đạt), `mastered` (thành thạo). |
| **ExerciseProgress** | Bản ghi tiến độ của một bài tập (bao gồm số lần làm, điểm tốt nhất, trạng thái cuối). |
| **Learning Event** | Sự kiện ghi nhận kết quả làm bài, dùng để cập nhật hồ sơ năng lực. |
| **CEFR** | Khung trình độ (A1–C2); bài tập gắn trình độ (vd B1) để lọc/gợi ý. |
| **ASR** | Automatic Speech Recognition (nhận dạng giọng nói) — chuyển audio bài nói thành text. |
| **Kỹ năng** | Nghe / Đọc / Viết / Nói / Ngữ pháp / Từ vựng. |
| **Hồ sơ Năng Lực** | Bản tóm tắt điểm mạnh/yếu từng kỹ năng của học viên, cập nhật dần khi làm bài tập. |
| **Lộ Trình Thích Nghi** | Đường học cá nhân hóa, được ghép từ các bài tập phù hợp kỹ năng + trình độ. |
| **Quota** | Giới hạn số lần nộp bài trên một khóa (để tránh lạm dụng chi phí AI). |
| **LessonQuiz** | Mô hình quiz curated tích hợp trong khóa học (xem course-graded-exercises). |
