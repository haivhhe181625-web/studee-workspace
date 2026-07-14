<!--
  acceptance-template.md — dùng cho .ai/prompts/generate-spec.md (BA Agent). Mỗi tiêu chí phải checkable
  bởi người hoặc bởi test — không viết tiêu chí kiểu "hoạt động tốt", "nhanh".
-->
# Acceptance Criteria: <Tên feature>

- **Spec:** `specs/<feature-slug>/spec.md`
- **Ngày:** <YYYY-MM-DD>

## Cách dùng file này

- Mỗi tiêu chí có ID (`AC-1`, `AC-2`, ...) để `design.md` và `tasks.md` tham chiếu ngược lại được.
- Ưu tiên format Given/When/Then khi hành vi phụ thuộc trạng thái; dùng checklist đơn giản khi không cần.
- Reviewer Agent (AI Self Review) sẽ đối chiếu từng ID này với test/behavior thật — ID không map được tới bất kỳ
  test/kiểm tra thủ công nào là một finding blocking.

## Tiêu chí chức năng

### AC-1: <Tên ngắn>

- **Given** <trạng thái ban đầu>
- **When** <hành động>
- **Then** <kết quả mong đợi, đo được>

### AC-2: <Tên ngắn>

- [ ] <Điều kiện checklist đơn giản>

## Tiêu chí lỗi / edge case

### AC-E1: <Tên>

- **Given** <trạng thái>
- **When** <hành động sai/edge case>
- **Then** <mã HTTP + error code cụ thể, hoặc hành vi UI cụ thể>

## Tiêu chí phi chức năng (nếu áp dụng)

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| Bảo mật | <vd: endpoint yêu cầu permission X> | <cách kiểm tra> |
| Hiệu năng | <vd: p95 < Nms> | <cách đo — chỉ ghi nếu thật sự có yêu cầu, đừng bịa số> |

## Ngoài phạm vi kiểm thử

<Nếu có phần cố ý không viết acceptance criteria (vì đã liệt kê "Ngoài phạm vi" ở spec.md), nhắc lại ở đây để
tránh Reviewer Agent flag nhầm là thiếu coverage.>
