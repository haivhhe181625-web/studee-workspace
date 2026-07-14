<!-- Tiếng Việt vì đây là tài liệu mô tả quy trình cho specs/ — theo .ai/project-context.md §8 -->
# specs/

Mỗi feature đi qua pipeline Idea → Merge (`.ai/workflows/feature-workflow.md`) có 1 thư mục riêng tại đây:

```
specs/<feature-slug>/
├── spec.md            # BA — mục tiêu, phạm vi, acceptance criteria tóm tắt
├── acceptance.md       # BA — tiêu chí chấp nhận chi tiết, checkable
├── design.md           # Tech Lead — kiến trúc, quyết định kỹ thuật
├── data-model.md        # Tech Lead — chi tiết data model (nếu không đơn giản đủ để gộp vào design.md)
├── research.md          # Tech Lead — chỉ tạo khi có câu hỏi kỹ thuật cần điều tra trước khi chốt design
├── tasks.md             # Tech Lead — task list TDD-checkbox cho Developer
├── contracts/            # Tech Lead — 1 file / boundary cross-service hoặc cross-repo
├── diagrams/             # Tuỳ chọn — sequence/flow diagram khi văn bản không đủ rõ
└── review.md             # Reviewer — kết quả AI Self Review / Technical Review
```

`<feature-slug>` dùng kebab-case, ngắn gọn, không kèm ngày (khác với `<API_REPO>/docs/superpowers/specs/<date>-<slug>.md` —
ở đây trạng thái/ngày nằm trong frontmatter của từng file, không nằm trong tên thư mục).

**`specs/example-feature/`** là ví dụ minh hoạ (không phải feature thật) — dùng để hiểu đúng "hình dạng" đầy đủ
trước khi bắt đầu spec đầu tiên. Xem thêm 2 tiền lệ thật đã có trong repo trước khi workspace này tồn tại:
`<API_REPO>/docs/superpowers/specs/2026-06-15-user-profile-edit-avatar-design.md` +
`<API_REPO>/docs/superpowers/plans/2026-06-15-user-profile-edit-avatar.md` — quy trình mới ở đây tách file rõ hơn (BA/Tech
Lead/Reviewer riêng biệt) nhưng giữ đúng tinh thần và mức độ chi tiết của 2 file đó.

## Vòng đời

Xem `.ai/workflows/feature-workflow.md` cho quy trình đầy đủ (vai trò, gate duyệt, exit criteria từng bước) và
`.ai/project-context.md` §9 cho tổng quan nhanh.
