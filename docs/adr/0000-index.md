<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8 -->
# ADR Index

Danh sách Architecture Decision Record, đánh số tăng dần. Dùng `adr-template.md` trong thư mục này để tạo ADR
mới: `docs/adr/NNNN-<slug-ngắn>.md`.

| # | Tên | Trạng thái | Ngày |
|---|---|---|---|
| — | [Migrate admin từ AdminJS sang admin REST + portal exe-admin](<API_REPO>/docs/ADR-admin-api-migration.md) | Chấp nhận | 2026-07-03 |
| 0001 | [Lưu cấu trúc khóa học tĩnh thành embedded-tree document (module course-content)](0001-course-content-store-embedded-tree.md) | Đề xuất | 2026-07-15 |

> ⚠️ Link ở bảng trên dùng placeholder `<API_REPO>` (xem `docs/cross-repo-linking.md`) — **không phải link bấm
> được**, vì từ ngày 2026-07-14 file này nằm ở repo `exe-api`, khác repo với `docs/adr/` (nay ở `studee-workspace`).
> Trước đó (`docs/adr/` từng nằm trong `exe-api`), link tương đối `../ADR-admin-api-migration.md` bấm được bình
> thường.

> ADR đầu tiên (`<API_REPO>/docs/ADR-admin-api-migration.md`) được viết **trước khi** có `docs/adr/` và **giữ nguyên vị trí
> cũ** (không di chuyển vào đây) để không phá liên kết đã tồn tại từ `<API_REPO>/CLAUDE.md` và các doc khác trỏ tới nó. Từ
> ADR tiếp theo trở đi, tạo trực tiếp trong `docs/adr/` với số thứ tự kế tiếp (0001, 0002, ...) và cập nhật bảng
> trên.

## Khi nào cần viết ADR

Theo `.ai/checklists/techlead-checklist.md`: khi một design đưa vào dependency ngoài mới, một pattern kiến trúc
mới, hoặc đảo ngược một quyết định trước đó. Không viết ADR cho quyết định implementation-level thông thường —
những cái đó thuộc `design.md` của feature, không phải ADR.
