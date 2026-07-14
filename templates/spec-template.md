<!--
  spec-template.md — dùng cho .ai/prompts/generate-spec.md (BA Agent, phase Specification).
  Copy file này thành specs/<feature-slug>/spec.md rồi điền. Xoá comment hướng dẫn (như đoạn này) sau khi điền.
  Tham khảo phong cách: <API_REPO>/docs/superpowers/specs/2026-06-15-user-profile-edit-avatar-design.md
-->
# Spec: <Tên feature>

- **Ngày:** <YYYY-MM-DD>
- **Tác giả:** <BA / AI Brainstorm>
- **Repos/surfaces ảnh hưởng:** <api / web / admin / mobile — liệt kê tất cả, kể cả repo chưa có tooling này>
- **Module liên quan (nếu là `exe-api`):** <<API_REPO>/services/api/src/modules/...>
- **Trạng thái:** <Nháp / Chờ BA review / Đã duyệt — chờ design>

## 1. Mục tiêu

<Một câu, không mơ hồ. "Cho phép X làm được Y trong tình huống Z."></br>

## 2. Bối cảnh

<Vì sao feature này cần thiết. Hiện trạng liên quan (đã kiểm tra code/docs, không đoán) — liệt kê cụ thể cái gì
đã tồn tại, cái gì chưa.>

## 3. Phạm vi

### Trong phạm vi

- <Mục 1>
- <Mục 2>

### Ngoài phạm vi

- <Mục loại trừ> — **Lý do:** <vì sao loại trừ, không chỉ liệt kê>

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| <vd: Learner> | <hành động> | <giá trị nhận được> |

## 5. Quyết định nghiệp vụ cần chốt

<Những điểm cần founder/product owner quyết — KHÔNG tự quyết trong spec. Để trống nếu không có.>

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|

## 6. Acceptance criteria (tóm tắt)

<Danh sách ngắn, chi tiết đầy đủ ở `acceptance.md`. Mỗi dòng ở đây phải map được về 1 mục "Trong phạm vi".>

1. <Tiêu chí 1 — checkable được>
2. <Tiêu chí 2>

## 7. Câu hỏi mở

<Để trống nếu không còn gì chặn approval. Nếu còn — spec CHƯA được duyệt.>

- [ ] <Câu hỏi 1>

## 8. Ghi chú cho Tech Lead Design

<Ràng buộc đã biết (hiệu năng, bảo mật, dữ liệu nhạy cảm...) mà BA biết trước, giúp design không đi sai hướng.
KHÔNG đề xuất giải pháp kỹ thuật ở đây — đó là việc của design.md.>
