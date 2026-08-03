# Tài liệu Tóm tắt: Hệ thống Lộ trình học 4 tầng (Dành cho C-Level / CEO)

## 1. Mục tiêu kinh doanh
Tính năng "Lộ trình học 4 tầng" là nền tảng cốt lõi giúp số hóa toàn bộ chương trình đào tạo. Mục tiêu của tính năng này là:
- **Tăng trải nghiệm cá nhân hóa:** Cho phép học viên học theo lộ trình (Adaptive/Task-based Learning), thấy rõ sự tiến bộ và nâng cấp nền tảng theo từng nấc cụ thể.
- **Tối ưu hóa quản lý vận hành:** Đội ngũ Học thuật có thể dễ dàng chuyển đổi giáo trình truyền thống (file Excel) thành một khóa học số hóa với cấu trúc chuẩn hóa, tự động và nhanh chóng.
- **Tạo đà mở rộng tương lai:** Xây dựng bộ khung dữ liệu cực kỳ linh hoạt để sau này hệ thống dễ dàng gắn thêm AI (tạo khóa học tự động), tính năng Gamification (tích điểm, chuỗi ngày học) và phân tích chi tiết dữ liệu học tập.

## 2. Cấu trúc Lộ trình học (4 Tầng)
Đội ngũ phát triển (Dev) đang xây dựng một bộ khung lưu trữ thông minh, chia nhỏ khóa học thành 4 cấp độ để tối ưu hóa việc quản lý và trải nghiệm:

1. **Lộ trình (Course):** 
   - Khóa học tổng thể với mục tiêu đầu ra bao quát (Ví dụ: "Khóa học IELTS 5.0 - 6.0").
2. **Chặng (Phase):** 
   - Các cột mốc nhỏ gọn trong Lộ trình để người học dễ dàng đạt được (Ví dụ: "Mục tiêu từ A2 lên B1").
   - Tính năng mở khóa (Unlock): Học viên thường phải hoàn thành Chặng trước đó để có thể đi tiếp sang Chặng tiếp theo.
3. **Chuyên đề (Module):** 
   - Tập hợp các nhóm kỹ năng bên trong một Chặng (Ví dụ: "Ngữ pháp", "Từ vựng Unit 1", "Luyện Nghe").
4. **Bài học (Lesson):** 
   - Đơn vị học tập cốt lõi, nơi học viên thực sự tương tác. 
   - **Quy chuẩn bắt buộc:** Đội Kỹ thuật đã thiết lập cơ chế tự động kiểm tra, bắt buộc mỗi bài học khi tải lên đều phải mang lại giá trị thực (phải chứa lý thuyết, hoặc video, hoặc bài tập thực hành). Tuyệt đối không có bài học "rỗng" trên hệ thống.

## 3. Hệ thống sẽ hoạt động như thế nào? (User Flow)
- **Đối với Đội Vận hành/Học thuật (Phase 1 đang thực hiện):**
  - Chỉ cần chuẩn bị duy nhất 1 file Excel chứa nội dung từ Lộ trình đến chi tiết Bài học.
  - Hệ thống sẽ tự động quét, kiểm tra các lỗi logic và đưa toàn bộ nội dung này vào cơ sở dữ liệu số trong vài giây.
- **Đối với Học viên (Phase 2 & 3 tiếp theo):**
  - Học viên chọn khóa học, thấy rõ bản đồ tiến trình của mình.
  - Học viên xem lý thuyết, video và làm bài tập. Khi đạt đủ điểm, hệ thống sẽ tự động mở khóa các bài tiếp theo để tạo động lực học tập liên tục.

## 4. Đội Kỹ thuật (Dev) đang làm gì hiện tại?
- Hiện tại, Đội Dev đang tập trung toàn lực vào **Phase 1 (Nhập liệu cấu trúc)**. Họ đang thiết kế kiến trúc cơ sở dữ liệu (Database) chuyên dụng, đảm bảo khả năng chứa và truy xuất siêu tốc hàng chục nghìn bài học mà không gây chậm lag.
- **Giá trị mang lại ngay lập tức:** Khi hoàn thiện Phase 1, Trung tâm đã sẵn sàng một "Nhà kho số hóa" hoàn chỉnh để bơm toàn bộ tài liệu, giáo án vào hệ thống một cách trơn tru, sẵn sàng cho học viên vào học ở giai đoạn sau.
