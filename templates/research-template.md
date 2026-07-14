<!--
  research-template.md — dùng khi Tech Lead Design gặp câu hỏi kỹ thuật cần spike/investigate trước khi
  chốt design.md. Không phải feature nào cũng cần file này — chỉ tạo khi có câu hỏi thật sự chưa biết câu
  trả lời (không phải để hợp thức hoá một quyết định đã chốt sẵn).
-->
# Research: <Câu hỏi cần trả lời>

- **Spec:** `specs/<feature-slug>/spec.md`
- **Ngày:** <YYYY-MM-DD>
- **Trạng thái:** <Đang điều tra / Đã có kết luận>

## Câu hỏi

<Câu hỏi kỹ thuật cụ thể, checkable — "Mongoose có support X hiệu quả với dataset hiện tại không", không phải
"làm sao để làm feature này tốt".>

## Vì sao cần trả lời trước khi design

<Quyết định nào trong design.md phụ thuộc vào câu trả lời này.>

## Phương pháp điều tra

- [ ] <vd: đọc source/docs thư viện X>
- [ ] <vd: viết script benchmark nhỏ, chạy trên dữ liệu mẫu>
- [ ] <vd: kiểm tra log/metric hiện có>

## Phát hiện

<Ghi lại bằng chứng cụ thể — số liệu, đoạn code, output lệnh đã chạy. Không kết luận suông.>

## Kết luận

<Câu trả lời cuối cùng + ảnh hưởng cụ thể tới design.md (mục nào, quyết định nào thay đổi).>

## Rủi ro còn lại (nếu có)

<Nếu kết luận không chắc chắn 100%, nêu rõ rủi ro và cách giảm thiểu trong design/rollout.>
