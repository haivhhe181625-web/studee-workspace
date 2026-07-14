<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md -->
# Research: [VÍ DỤ] Có cần research.md cho feature này không?

- **Spec:** `specs/example-feature/spec.md`
- **Ngày:** 2026-07-14
- **Trạng thái:** Đã có kết luận

## Câu hỏi

Feature "nhắc học qua email" có đủ đơn giản để bỏ qua bước research, đi thẳng từ spec sang design không?

## Vì sao cần trả lời trước khi design

`templates/research-template.md` chỉ nên dùng khi có câu hỏi kỹ thuật thật sự chưa biết câu trả lời. Ví dụ này
minh hoạ trường hợp **không cần** research.md — để AI agent không tạo file này một cách máy móc cho mọi feature.

## Phát hiện

Hạ tầng BullMQ repeatable job và adapter email đã tồn tại và đã được dùng cho use case tương tự (reset password
email). Không có ẩn số kỹ thuật cần spike.

## Kết luận

Không cần research sâu — `design.md` đi thẳng từ spec. File này được giữ lại trong ví dụ chỉ để minh hoạ: khi
câu trả lời là "không cần research", vẫn có thể tạo file ngắn gọn ghi nhận lý do bỏ qua, thay vì im lặng không
tạo file (giúp Reviewer Agent không phải đoán "quên hay cố ý bỏ qua").

## Rủi ro còn lại (nếu có)

Không có.
