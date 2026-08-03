# Đặc tả tính năng: Bộ khung dữ liệu Lộ trình học 4 tầng (4-Level Course Structure)

## 1. Tổng quan tính năng (Overview)
- **Mục tiêu:** Xây dựng hệ thống khung dữ liệu phân tầng chuẩn hóa để quản lý toàn bộ nội dung học tập. Giúp người học nâng cấp nền tảng theo từng nấc (Adaptive/Task-based Learning) và giúp người quản trị (Admin) dễ dàng tổ chức nội dung số hóa.
- **Cấu trúc cốt lõi:** Lộ trình (Course) ➔ Chặng (Phase) ➔ Chuyên đề (Module) ➔ Bài học (Lesson).

> **Lưu ý về phạm vi (Scope):** Các quy tắc mở khóa (Unlock) và theo dõi tiến độ (Mastery) đề cập ở đây thuộc phạm vi của **Phase 2 & 3**. Trong **Phase 1**, hệ thống chỉ tập trung thiết lập cấu trúc và import dữ liệu tĩnh.

## 2. Mô hình lưu trữ (Data Storage Model)
Thay vì sử dụng mô hình quan hệ (Foreign Key) truyền thống, hệ thống sử dụng mô hình **Nested/Embedded Sub-document** (Nhúng) trong NoSQL (MongoDB). Khi truy xuất 1 Lộ trình (Course), hệ thống sẽ trả về toàn bộ cây dữ liệu bao gồm các Phase, Module và Lesson bên trong.
- **1 Lộ trình (Course)** nhúng **nhiều Chặng (Phase)** `(1:N)`
- **1 Chặng (Phase)** nhúng **nhiều Chuyên đề (Module)** `(1:N)`
- **1 Chuyên đề (Module)** nhúng **nhiều Bài học (Lesson)** `(1:N)`

## 3. Đặc tả chi tiết các tầng (Data & Logic Definition)

### Tầng 1: Lộ trình (Course)
**Định nghĩa:** Đơn vị lớn nhất, đại diện cho một khóa học hoàn chỉnh với một mục tiêu đầu ra bao quát (VD: Khóa học Tiếng Anh Giao Tiếp Toàn Diện, Khóa học IELTS Nền Tảng).
- **Thuộc tính chính (Attributes):**
  - `courseId` (Định danh tự sinh - ObjectId)
  - `title` (Tên Lộ trình)
  - `description` (Mô tả Lộ trình)
  - `thumbnail/cover` (Ảnh đại diện)
  - `status` (Draft / Ready / Published / Archived)
- **Quy tắc nghiệp vụ (Business Rules):**
  - Một Lộ trình phải có ít nhất 1 Chặng (Phase) không rỗng để được chuyển trạng thái sang `Ready`.

### Tầng 2: Chặng (Phase)
**Định nghĩa:** Một "nấc nâng trình độ" cụ thể trong Lộ trình. Giúp chia nhỏ mục tiêu của khóa học lớn thành các chặng đường dễ đạt được hơn.
- **Thuộc tính chính (Attributes):**
  - `key` (Định danh tự nhập từ file Excel để mapping, duy nhất trong khóa)
  - `title` (Tên Chặng - VD: "Mục tiêu B1" hoặc "Giai đoạn 1: Xây nền")
  - `order` (Thứ tự của Chặng trong Lộ trình: 1, 2, 3...)
  - `cefrFrom` ➔ `cefrTo` (Mục tiêu đầu vào/đầu ra chuẩn CEFR, VD: A2 ➔ B1)
  - `goalNote` (Ghi chú mục tiêu, VD: "IELTS 5.0 -> 6.0")
- **Quy tắc nghiệp vụ (Business Rules):**
  - **Tính tuần tự (Phase 2):** Người học thường phải hoàn thành Chặng `order = 1` mới được mở khóa (Unlock) Chặng `order = 2`.

### Tầng 3: Chuyên đề (Module)
**Định nghĩa:** Một mảng kỹ năng hoặc nhóm chủ đề cụ thể bên trong một Chặng. Giúp phân loại các bài học theo nhóm kiến thức.
- **Thuộc tính chính (Attributes):**
  - `key` (Định danh tự nhập từ file Excel để mapping, duy nhất trong Chặng)
  - `title` (Tên Chuyên đề - VD: "Ngữ pháp", "Từ vựng Unit 1", "Luyện Nghe")
  - `category` (Phân loại kỹ năng: `grammar` / `pronunciation` / `vocabulary` / `listening` / `reading` / `writing` / `speaking`)
  - `order` (Thứ tự hiển thị trong Chặng)
- **Quy tắc nghiệp vụ (Business Rules):**
  - Các Chuyên đề trong cùng một Chặng có thể được thiết lập học song song (không bắt buộc khóa) hoặc học tuần tự tùy theo setting.

### Tầng 4: Bài học (Lesson)
**Định nghĩa:** Đơn vị học tập nhỏ nhất, nơi chứa trực tiếp nội dung tương tác, lý thuyết và bài tập đa phương thức.
- **Thuộc tính chính (Attributes):**
  - `key` (Định danh tự nhập từ file Excel để mapping, duy nhất trong Chuyên đề)
  - `title` (Tên bài học - VD: "Thì Hiện Tại Đơn", "Phát âm âm /æ/")
  - `content_theory` (Nội dung lý thuyết định dạng Markdown / HTML)
  - `media_urls` (Danh sách các link Video / Audio bổ trợ)
  - `exercise_refs` (Tham chiếu đến các bài tập thực hành - trỏ tới các Engine khác như IPA, Talk, Quiz...)
  - `order` (Thứ tự Bài học trong Chuyên đề)
- **Quy tắc nghiệp vụ (Business Rules):**
  - **Quy tắc Bài học không rỗng (Non-empty Lesson):** Một Bài học không bắt buộc phải có bài tập (Exercise), nhưng phải có ít nhất 1 nội dung truyền tải (Lý thuyết, Video, Audio, hoặc Bài tập). Nếu rỗng hoàn toàn sẽ báo lỗi khi import.
  - **Trạng thái hoàn thành (Phase 2):** Một Bài học được đánh dấu "Hoàn thành" (Completed) khi người dùng lướt hết lý thuyết và (hoặc) vượt qua điểm pass của các bài tập (`exercise_refs`) đính kèm.

---

### 4. Luồng hoạt động cơ bản (User Flows)
- **Luồng Admin (Content Ingestion - Phase 1):** Admin nhập toàn bộ cấu trúc (từ Course đến Lesson) thông qua một file import (ví dụ: Excel) hoặc qua giao diện CMS. Hệ thống sẽ validate tính toàn vẹn (Lesson phải thuộc Module, Module phải thuộc Phase...) và kiểm tra các quy tắc rỗng trước khi lưu toàn bộ cây dữ liệu vào DB dưới dạng một Document duy nhất.
- **Luồng Learner (Course Consumption - Phase 2):** Học viên enroll vào `Course` ➔ Màn hình chính render danh sách `Phase` ➔ Bấm vào `Phase` đang học để xem các `Module` ➔ Bên trong `Module` là danh sách `Lesson`. Hoàn thành `Lesson` hiện tại sẽ unlock `Lesson` tiếp theo.

---

## 5. Các Quy tắc Nghiệp vụ Nâng cao (Advanced Business Rules)

### 5.1. Giới hạn dung lượng và Kiến trúc (Data & Architecture Rules)
- **Giới hạn kích thước (16MB Limit):** Do sử dụng mô hình nhúng (Nested/Embedded Sub-document), tổng dung lượng 1 Lộ trình không được vượt quá giới hạn cứng 16MB của MongoDB. Khuyến nghị mỗi Lộ trình chứa tối đa ~5.000 Bài học để đảm bảo tốc độ truy xuất.
- **Ràng buộc Import Media:** Khi Admin import từ file Excel, hệ thống chặn việc import media (ảnh/video) trực tiếp dưới dạng Base64 vào DB. Bắt buộc phải lưu trữ file trên nền tảng khác (CDN, YouTube) và chỉ nhập liên kết (URL) vào trường `media_urls`.

### 5.2. Xử lý đồng bộ & Import dữ liệu (Ingestion Rules - Phase 1)
- **Tính toàn vẹn (Idempotency):** Hiện tại hệ thống phát hiện trùng lặp dựa trên `slug` (đường dẫn khóa học). Nếu Admin upload lại file Excel đã từng import, hệ thống sẽ báo lỗi trùng lặp (`ALREADY_EXISTS`) thay vì ghi đè (upsert). Để cập nhật ở Phase 1, Admin cần tuân thủ quy trình "Xóa khóa cũ - Import khóa mới". (Tính năng upsert/versioning sẽ có ở Phase 5).

### 5.3. Quản lý phiên bản và Cập nhật nội dung (Versioning Rules - Phase 2, 3)
- **Rule khóa cấu trúc (Structure Lock):** Khi một Khóa học đã chuyển sang trạng thái `Published` và có học viên đăng ký (enroll), Admin không được phép xóa cứng (Hard Delete) các Chặng, Chuyên đề hay Bài học bên trong để tránh làm hỏng dữ liệu tiến độ. Chỉ cho phép ẩn (Soft Delete) hoặc vô hiệu hóa.
- **Rule cập nhật tiến độ (Progress Recalculation):** Nếu học viên đã học xong Chặng 1 và đang ở Chặng 2, nhưng Admin chèn thêm một Bài học mới vào Chặng 1, trạng thái của Chặng 1 sẽ tự động lùi từ "Completed" về "In Progress". Tuy nhiên, hệ thống sẽ **không khóa lại** (relock) Chặng 2 mà người dùng đang học dở để tránh gián đoạn trải nghiệm.

### 5.4. Logic mở khóa và Ràng buộc học tập (Unlocking Rules - Phase 2, 3)
- **Quy tắc Pass Rate:** Một Bài học được tính là hoàn thành khi điểm bài tập đạt mức Pass Rate. Mức điểm này có thể được cấu hình chung ở cấp Lộ trình (Course) hoặc thiết lập riêng rẽ (override) cho từng Lesson/Exercise.
- **Mở khóa chéo (Cross-dependency):** Hiện tại hệ thống áp dụng mở khóa tuyến tính. Không hỗ trợ mở khóa chéo phức tạp (Ví dụ: Không yêu cầu hoàn thành Bài tập của Module A mới được mở Bài học ở Module B).
