# Spec: Trò chuyện trực tiếp (realtime) với AI

- **Ngày:** 2026-09-10
- **Tác giả:** AI Brainstorm + dev
- **Repos/surfaces ảnh hưởng:** `api` (cấp vé + webhook), `llm` (worker giọng nói), `web` (mode UI mới)
- **Module liên quan (exe-api):** `services/api/src/modules/talk/`, `services/llm/src/`
- **Trạng thái:** Nháp — chờ review

## 1. Mục tiêu

Cho phép học viên nói chuyện **trực tiếp, liên tục** với gia sư AI bằng giọng nói (nói–nghe realtime, ngắt lời được), song song với mode nói theo lượt hiện có.

## 2. Bối cảnh

- Mode talk hiện tại (`talk.service.js`) là **theo lượt, REST**: thu âm → gửi → STT → chat → TTS → trả về. Có độ trễ cả câu, không ngắt lời được, không liên tục.
- Đã kiểm tra code: **chưa có** hạ tầng realtime nào. `conversation:token` chỉ là **ví dụ trong quy ước đặt tên** (socket.js:6), không phải feature đã build. Socket hiện chỉ chạy `focus-room`.
- `llm` là ACL bọc provider ngoài qua adapter (`openai_*`). Chưa có adapter cho mô hình giọng nói realtime.
- Cần một đường truyền âm 2 chiều độ trễ thấp — dùng hạ tầng ngoài (LiveKit, bản free) thay vì tự xây.

## 3. Phạm vi

### Trong phạm vi (nền móng, hạn 15/9)
- Endpoint cấp "vé" vào phòng realtime (chặn theo đăng nhập).
- Worker AI trong `llm` vào phòng, trả lời bằng giọng nói, ngắt lời được.
- Giao diện mode trực tiếp trên web (kết nối / nói / nghe / kết thúc + trạng thái).
- Giới hạn thời lượng mỗi buổi (bảo vệ hạn mức free).

### Ngoài phạm vi
- **Góp ý/sửa sai chi tiết sau buổi** — **Lý do:** cần dữ liệu transcript ổn định trước; làm ở đợt sau.
- **Tính giờ theo phút gắn gói cước** — **Lý do:** nền móng chỉ cần chặn thời lượng cứng; wiring quota là việc riêng.
- **Lưu lịch sử hội thoại vào DB** — **Lý do:** chưa cần cho luồng chạy được; thêm khi làm feedback.
- **Kiểm duyệt nội dung realtime** — **Lý do:** pipeline moderate của mode cũ không áp được vào luồng streaming; dựa an toàn của mô hình + hiến chương gia sư cho giai đoạn nền (cần chốt nghiệp vụ, xem §5).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Học viên | Bấm kết nối rồi nói chuyện liền mạch với AI | Luyện phản xạ nói như hội thoại thật |
| Học viên | Ngắt lời AI giữa chừng | Trao đổi tự nhiên, không phải chờ hết câu |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| Realtime bỏ kiểm duyệt nội dung ở giai đoạn nền — chấp nhận? | Chấp nhận, dựa an toàn mô hình + hiến chương; thêm moderate sau | Product owner / Tech Lead |
| Giới hạn thời lượng mỗi buổi | 5 phút/buổi (bảo vệ hạn mức free) | Product owner |

## 6. Acceptance criteria (tóm tắt)

1. Người **đã đăng nhập** gọi endpoint cấp vé → nhận vé hợp lệ, hạn ngắn; người **chưa đăng nhập** → bị chặn (401).
2. Từ web, bấm "Kết nối" → vào phòng, xin quyền micro thành công.
3. Nói → nghe AI trả lời bằng giọng nói, đúng vai gia sư tiếng Anh.
4. Ngắt lời khi AI đang nói → AI dừng và nghe lại.
5. Màn hình phản ánh đúng trạng thái: đang kết nối / đang nghe / AI đang nói.
6. Bấm "Kết thúc" → ngắt sạch, nhả micro.
7. Buổi tự dừng khi chạm giới hạn thời lượng.
8. Mode nói theo lượt cũ (`/talk/turn`) vẫn hoạt động bình thường.
9. Độ trễ nói–nghe cảm nhận < ~2 giây (đo ở spike Phase 0).

## 7. Câu hỏi mở

- [ ] Quyết định nghiệp vụ §5 (moderation + thời lượng) cần product owner ký.

## 8. Ghi chú cho Tech Lead Design

- Bí mật (khóa hạ tầng/mô hình) **chỉ ở server**, không xuống client. FE chỉ nhận vé từ `api`.
- Chữ UI **trung tính**, không nêu tên nhà cung cấp.
- Provider mới **phải qua adapter** (luật `llm` §3), không gọi thẳng SDK trong logic.
- Worker là tiến trình **thường trú**, khác kiểu tải request/response của FastAPI — cần tách entrypoint. Đây là dependency ngoài mới → cần ADR.
