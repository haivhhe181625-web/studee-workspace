<!--
  design-template.md — dùng cho .ai/prompts/generate-design.md (Tech Lead Agent, phase Tech Lead Design).
  Copy thành specs/<feature-slug>/design.md. Tham khảo phong cách phần "Thiết kế chi tiết" trong
  <API_REPO>/docs/superpowers/specs/2026-06-15-user-profile-edit-avatar-design.md.
-->
# Design: <Tên feature>

- **Spec:** `specs/<feature-slug>/spec.md`
- **Ngày:** <YYYY-MM-DD>
- **Tác giả:** <Tech Lead / Tech Lead Agent>
- **Trạng thái:** <Nháp / Chờ Technical Review / Đã duyệt>
- **ADR liên quan:** <docs/adr/NNNN-....md, hoặc "không có">

## 1. Tóm tắt kiến trúc

<1-2 đoạn: cách tiếp cận tổng thể, module nào mở rộng / module nào mới và vì sao. Đối chiếu
<API_REPO>/docs/ARCHITECTURE.md — không được mâu thuẫn ADR hiện có mà không nêu rõ lý do supersede.>

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| <vd: Nơi lưu file> | <lựa chọn> | <lý do> | <cái gì đánh đổi> |

## 3. Data model

<Tham chiếu chi tiết đầy đủ ở `data-model.md` nếu phức tạp; nếu đơn giản, viết thẳng ở đây.>

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|

## 4. Luồng dữ liệu

```
<Client/service> --(request)--> <endpoint/handler>
  <middleware 1>
  -> <middleware 2>
  -> <business logic>
  -> <side effect: DB / queue / socket / external call qua adapter>
  -> <response>
```

## 5. Contracts

<Liệt kê contract nào cần viết ở contracts/ — API request/response, cross-service (api↔llm/cat),
cross-repo (api↔web/admin). Mỗi contract = 1 file trong specs/<feature>/contracts/, dùng contract-template.md.>

- `contracts/<tên>.md` — <mô tả ngắn>

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `<đường dẫn>` | <mô tả> | Tạo/Sửa |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|

## 8. Bảo mật & quyền

<Permission string mới (nếu có, theo `<resource>:<action>`), tenant scope, dữ liệu nhạy cảm nào cần
`select: false` / audit log. Đối chiếu <API_REPO>/docs/api/CONVENTIONS.md §5 rules auth tuyệt đối.>

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

<Nếu có câu hỏi cần spike trước khi chốt design — tách ra `research.md` thay vì đoán ở đây.>

## 10. Testing strategy

<Loại test (unit/integration/API), case chính cần cover — chi tiết case cụ thể nằm trong tasks.md khi viết
test-first.>

## 11. Rollout / cross-repo sequencing

<Nếu feature này đụng nhiều repo (api/web/admin), nêu rõ thứ tự deploy — xem
.ai/workflows/release-workflow.md §3 Cross-repo releases.>

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion (từ acceptance.md) | Được đáp ứng bởi (phần nào trong design này) |
|---|---|
