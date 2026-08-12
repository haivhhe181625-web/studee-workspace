# Spec: Thời lượng bài học + Editor nội dung drill-down

- **Ngày:** 2026-08-06 · **Trạng thái:** Draft (chờ duyệt) · **Ticket:** PRD-114 (M4-0) mở rộng
- **Repos/surfaces:** `exe-api` (`services/api`: course-content module + admin-api) · `exe-admin` (portal soạn nội dung)
- **Nguồn:** brainstorm đã duyệt — `plans/reports/from-brainstorm-to-spec-260806-0353-course-duration-drilldown-editor-report.md`

## 1. Mục tiêu
Cho phép **set thời lượng thật** cho từng bài học (thay vì luôn rơi về heuristic theo type) qua **cả 2 đường**: cột `DurationMin` trong file xlsx nội dung **và** field trong editor admin — không phải nhảy vào xlsx để sửa. Đồng thời **cấu trúc lại editor** nội dung từ 1 trang cả cây sang **drill-down** (khóa → chặng → chuyên đề+lesson).

## 2. Bối cảnh (scout 2026-08-06)
- `LessonSchema.estimatedDurationMin` (Number, min 1, default null) **đã có** (`course-content.model.js:72`).
- `durationOf(lesson)` (`adaptive/schedule/lesson-duration.js`): ưu tiên field, thiếu → heuristic theo type. Generator lịch tiêu thụ qua đây.
- **Bug hiện tại:** `buildTree()` strip lesson **bỏ** `estimatedDurationMin` → sửa/lưu cây in-app **xoá trắng** duration; parser xlsx không map cột duration → **thực tế duration luôn = heuristic**.
- Editor admin **đã là cây 1 trang** (`exe-admin/.../course-import/course-detail.tsx`) — load cả cây, sửa inline, lưu whole-tree PUT `updateCourse` (`course-content.admin.js §6`), sửa được khi status `draft/ready`.

## 3. Phạm vi
### Trong phạm vi
- **IS-1 (exe-api):** vá `buildTree` giữ `estimatedDurationMin` (chấm dứt xoá-trắng khi lưu cây/import).
- **IS-2 (exe-api):** parser xlsx thêm cột **`DurationMin`** (optional, LESSON row) → `estimatedDurationMin`; blank→null→heuristic. Validate int>0 nếu có.
- **IS-3 (exe-api):** template (`§2`) + export (`§9`) có cột `DurationMin` (round-trip không mất).
- **IS-4 (exe-api):** `§4` chi tiết khóa trả kèm **`effectiveDurationMin`** per lesson (từ `durationOf`, read-only, không lưu) — để FE rollup chính xác mà không nhân bản heuristic.
- **IS-5 (exe-admin):** `LessonNode` mang `estimatedDurationMin` (+ đọc `effectiveDurationMin`); editor giữ/lưu qua whole-tree PUT như cũ.
- **IS-6 (exe-admin):** tách `course-detail` thành **điều hướng drill-down** khóa → chặng → chuyên đề(+lesson inline). State giữ **cả cây** ở container, drill-down đổi slice render, 1 nút Lưu whole-tree (giữ `dirty`/save, khóa published không sửa).
- **IS-7 (exe-admin):** field `estimatedDurationMin` per-lesson (trống = "auto theo loại") + **rollup tổng phút read-only** (dùng `effectiveDurationMin`) ở chuyên đề/chặng/khóa.

### Ngoài phạm vi (ghi backlog)
- Endpoint per-node (per chặng/chuyên đề) — *giữ whole-tree PUT, drill-down thuần FE (KISS; admin ít concurrency, data nhỏ)*.
- Đổi contract whole-tree PUT `updateCourse`.
- Đụng generator lịch (`lesson-schedule.generator.js`) — nó đã đọc `durationMin` sẵn.
- Trang sửa lesson riêng (module+lesson đã đủ nhỏ).
- Sửa nội dung khóa `published/archived` (giữ khóa hiện tại).
- Backfill dữ liệu cũ (default null + heuristic đảm bảo > 0).

## 4. Acceptance Criteria
1. **AC-1** (IS-1): import lại **hoặc** sửa+lưu cây in-app cho 1 lesson đã có `estimatedDurationMin` → giá trị **vẫn giữ**, không về null.
2. **AC-2** (IS-2): import xlsx có cột `DurationMin`=25 cho 1 lesson → `estimatedDurationMin=25`; blank → null → `durationOf` trả heuristic theo type.
3. **AC-3** (IS-2): `DurationMin` không phải int>0 (vd 0, "abc") → lỗi validate chỉ đúng dòng đó, không chặn dòng khác.
4. **AC-4** (IS-3): export khóa → mở lại có cột `DurationMin` đúng giá trị; import file vừa export → không đổi dữ liệu (round-trip).
5. **AC-5** (IS-4): `GET /:id` trả mỗi lesson kèm `effectiveDurationMin` = `estimatedDurationMin` nếu có, else heuristic; luôn > 0.
6. **AC-6** (IS-7): admin mở editor, đổi thời lượng 1 lesson, Lưu → persist (không cần xlsx); rollup chuyên đề/chặng/khóa = tổng `effectiveDurationMin` cập nhật đúng.
7. **AC-7** (IS-6): điều hướng khóa → bấm chặng → bấm chuyên đề thấy list lesson sửa inline; chuyển cấp không mất sửa dở; Lưu ghi cả cây; khóa `published` hiện read-only.

## 5. Câu hỏi mở
- Vị trí cột `DurationMin` trong xlsx (cột nối tiếp N?) — chốt ở design.
- Báo đội content về cột mới (đã quyết tự làm parser/template/export).
