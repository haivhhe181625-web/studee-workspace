# Kế hoạch Kiến trúc & Triển khai: Task-based Learning (Hành trình Thực nhiệm)

Tài liệu này mô tả chi tiết kiến trúc và kế hoạch triển khai cho 4 Epic liên quan đến định hình lộ trình học tập, engine luyện tập đa hình, theo dõi tiến độ và hệ thống nhập liệu.

---

## 1. Cấu trúc 4 Tầng (4 Layers)
Hệ thống học tập được tổ chức theo cấu trúc phân tầng nghiêm ngặt, cho phép khóa học hỗ trợ cơ chế mở khóa tuần tự (Unlock Logic).

1. **Lộ trình (Roadmap/Course):** Cấp độ cao nhất. Quản lý mục tiêu tổng thể. *Ví dụ: "Giao tiếp toàn diện (Mục tiêu A2)".*
2. **Chặng (Phase/Stage):** Các cột mốc lớn trong khóa học. Cần vượt qua Chặng trước để mở khóa Chặng sau. *Ví dụ: "Chặng 1: Sinh tồn tại Sân bay".*
3. **Chuyên đề (Module):** Nhóm các bài học chung 1 chủ đề hoặc kỹ năng nhỏ. *Ví dụ: "Chuyên đề 1.1: Làm thủ tục (Check-in)".*
4. **Bài học (Lesson/Test):** Đơn vị tương tác nhỏ nhất. Đây là một "ổ cắm" có thể chứa nhiều Bài tập (Exercises) khác nhau ở bên trong. *Ví dụ: "Bài học 1: Hội thoại check-in", "Bài Test Chặng 1".*

---

## 2. Luồng Vận hành & Xây dựng Khóa học (Admin Flow)

Để phát hành một khóa học, đội ngũ Học thuật / Admin có thể đi theo một trong hai luồng vận hành sau:

**Luồng 1: Nhập liệu tự động qua Template (Khuyên dùng - Epic 4)**
*   **Bước 1:** Điền file Excel/JSON theo cấu trúc chuẩn từ Tên lộ trình -> Chặng -> Chuyên đề -> Bài học -> Câu hỏi trắc nghiệm / Prompt AI / Text cần luyện phát âm.
*   **Bước 2:** Lên hệ thống Admin Web, vào mục **Import Khóa học**, tải file lên.
*   **Bước 3:** Hệ thống tự động phân tích (parse), tạo dữ liệu vào các bảng Database tương ứng và tự động móc nối các khóa ngoại (Foreign Keys). Khóa học sẵn sàng hoạt động ngay lập tức.

**Luồng 2: Xây dựng thủ công trên giao diện (CMS UI)**
*   **Bước 1 (Tạo Root):** Khởi tạo Roadmap mới, quy định tổng thời gian và chuẩn đầu ra (CEFR).
*   **Bước 2 (Xây Cây):** Kéo thả thêm Phase, tạo các Module bên trong Phase, tạo các Lesson bên trong Module.
*   **Bước 3 (Thiết lập Unlock):** Đặt điều kiện mở khóa cho từng Lesson (vd: Phải đạt bài trước >80% điểm).
*   **Bước 4 (Cắm Bài tập):** Mở Lesson, chọn "Add Exercise", gọi các Component từ Engine Đa hình (trắc nghiệm, audio, nhập vai AI) và cấu hình ngay tại chỗ.

---

## 3. Cấu trúc Bài tập bao quát 4 Kỹ năng

Nhờ thiết kế **Engine Luyện tập Đa hình (Polymorphic Engine - Epic 2)**, một Lesson có thể chứa hỗn hợp nhiều dạng bài tập nhằm đánh giá toàn diện 4 kỹ năng:

*   **🗣️ Kỹ năng Nói (Speaking):**
    *   **Luyện phát âm câu/từ (Module `ipa`):** Đọc text, hệ thống chấm điểm từng âm đỏ/xanh/vàng, chỉ ra lỗi trọng âm.
    *   **Nhập vai AI (Module `talk`):** Gọi bằng Voice với nhân vật AI theo kịch bản Prompt cấu hình sẵn để luyện độ trôi chảy (Fluency).
*   **🎧 Kỹ năng Nghe (Listening):**
    *   **Nghe chép chính tả (Dictation):** Nghe audio và gõ lại văn bản.
    *   **Multiple Choice:** Nghe và chọn đáp án đúng để kiểm tra khả năng nắm bắt ý chính (Gist) và chi tiết.
*   **📖 Kỹ năng Đọc (Reading):**
    *   Đọc hiểu đoạn văn/hình ảnh thực tế (biển báo, email) và trả lời câu hỏi, hoặc điền từ vào chỗ trống (Fill in the blanks).
    *   Sắp xếp lại câu (Reordering) để tạo thành một đoạn hội thoại logic.
*   **✍️ Kỹ năng Viết (Writing):**
    *   **Điền từ vào câu (Cloze test):** Kéo thả từ vựng hoàn thành ngữ pháp.
    *   **Viết đoạn văn / Email (AI Grading):** Dùng AI Prompt "Grader" để phân tích lỗi ngữ pháp và chấm điểm Band score ngay khi user nộp bài.

---

## 4. Đề xuất Kiến trúc Database & API (Kế hoạch Cấp thấp)

### Epic 1: The Learning Journey (Cấu trúc phân tầng)
*   **[NEW]** `roadmap-module.model.js`: Quản lý tầng Module (thuộc Phase). Chứa logic `unlockCriteria`.
*   **[NEW]** `roadmap-lesson.model.js`: Quản lý tầng Lesson (thuộc Module). Chứa `type` (theory, practice, test).
*   **[MODIFY]** `roadmap.service.js`: Sinh dữ liệu phân tầng vào `user_roadmap_progress` khi gán roadmap cho User. Viết event-listener cho **Unlock Logic**.

### Epic 2: Dynamic Exercise Engine (Động cơ luyện tập đa hình)
*   **[NEW]** `practice-exercise.model.js` (trong module `practice`): Cấu trúc Polymorphic kết nối Lesson với Bài tập cụ thể (`type: 'ipa' | 'talk' | 'quiz'`, `refId`).
*   **[NEW]** `ai-prompt.model.js` (trong module `talk`): Dịch chuyển Prompt AI ra khỏi source code, cho phép quản lý biến động (Variables: `[USER_LEVEL]`, `[TOPIC]`).
*   **[MODIFY]** `practice.controller.js`: API `GET /practice/lesson/:lessonId` trả về metadata tổng của Lesson cùng mảng bài tập đã được "populate" dữ liệu để Frontend render động.

### Epic 3: Progress & Performance Tracking (Theo dõi & Lưu vết)
*   **[NEW]** `user-streak.model.js`: Ghi nhận `currentStreak`, `longestStreak`, `lastActiveDate`.
*   **[MODIFY]** `learning-event.model.js`: Đảm bảo mọi event từ mọi bài tập (ipa, talk, quiz) đều lưu lại `score`, `durationSec`, và band điểm để phục vụ tính toán.
*   **[NEW]** `progress.service.js`: Aggregate điểm số để update % tiến độ của Lesson -> Module -> Phase tương ứng, giảm tải query trực tiếp lên dashboard.

### Epic 4: Luồng Vận hành & Nhập liệu (Content Ingestion)
*   **[NEW]** `cms.controller.js` & `cms-import.service.js`: Xây dựng module Admin chuyên trách việc import file Excel/JSON, parse dữ liệu, và tự động INSERT phân tầng từ Roadmap xuống đến Bài tập.
