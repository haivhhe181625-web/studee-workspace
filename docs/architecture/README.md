<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8 -->
# docs/architecture/

Thư mục này dành cho tài liệu kiến trúc **chi tiết theo từng domain** (vd: sơ đồ sâu về pipeline MST/IRT, luồng
proctoring, cơ chế cache của `llm`) — bổ sung cho, **không thay thế**, các file kiến trúc cấp cao đã có:

- `<API_REPO>/docs/ARCHITECTURE.md` — kiến trúc tổng hệ thống (cross-service).
- `<API_REPO>/docs/api/ARCHITECTURE.md` — kiến trúc riêng service `api`.
- `<API_REPO>/docs/llm/ARCHITECTURE.md` — kiến trúc riêng service `llm`.

Ba file trên **giữ nguyên vị trí hiện tại** (không di chuyển vào đây) — đây chỉ là nơi chứa tài liệu sâu hơn khi
một domain cụ thể cần giải thích riêng, dài hơn mức phù hợp để nhét vào file kiến trúc tổng.

## Hiện tại: trống

Chưa có tài liệu nào trong này — được tạo sẵn theo cấu trúc AI workspace, sẽ điền khi Tech Lead Design của một
feature cụ thể cần một bản kiến trúc domain riêng (thường đi kèm `specs/<feature>/design.md`, xem
`templates/design-template.md` §1).
