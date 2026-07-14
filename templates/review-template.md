<!--
  review-template.md — dùng cho .ai/prompts/review-pr.md (Reviewer Agent, phase AI Self Review), và cho
  Technical Review ở design stage. Điền findings theo mức độ nghiêm trọng giảm dần.
-->
# Review: <Tên feature / PR>

- **Loại review:** <Technical Review (design) / AI Self Review (implementation)>
- **Ngày:** <YYYY-MM-DD>
- **Người/agent review:** <Reviewer Agent / tên người>
- **Đối tượng review:** <link PR, hoặc `specs/<feature-slug>/design.md`>

## Kết quả tổng quan

- **Test suite:** <PASS/FAIL — lệnh đã chạy + output tóm tắt, KHÔNG giả định>
- **Acceptance coverage:** <N/M tiêu chí xác nhận được>
- **Findings blocking:** <số lượng>
- **Findings non-blocking:** <số lượng>

## Findings

<Sắp xếp nghiêm trọng nhất trước. Để trống bảng nếu không có finding nào.>

### [BLOCKING] <Tiêu đề ngắn>

- **File:** `<path>:<line>`
- **Mô tả:** <cái gì sai, cụ thể>
- **Kịch bản lỗi:** <input/trạng thái cụ thể → hành vi sai/crash cụ thể — không viết chung chung>
- **Đề xuất fix:** <ngắn gọn, hoặc "trả lại Developer Agent">

### [NON-BLOCKING] <Tiêu đề ngắn>

- **File:** `<path>:<line>`
- **Mô tả:** <cái gì có thể cải thiện>
- **Lý do không block:** <vd: style preference, không có linter enforce>

## Đối chiếu Acceptance Criteria

| ID | Tiêu chí | Trạng thái | Bằng chứng |
|---|---|---|---|
| AC-1 | <tóm tắt> | ✅/❌ | <tên test, hoặc "manual — lệnh curl đã chạy"> |

## Checklist đã chạy

<Copy từ .ai/checklists/review-checklist.md, đánh dấu từng mục.>

## Kết luận

<"Sẵn sàng cho Tech Lead Review" / "Trả lại Developer Agent — N finding blocking cần fix trước".>
