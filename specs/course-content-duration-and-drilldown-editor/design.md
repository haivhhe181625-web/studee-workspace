# Design: Thời lượng bài học + Editor nội dung drill-down

- **Ngày:** 2026-08-06 · **Trạng thái:** Draft (chờ Tech Lead duyệt) · **Spec:** `./spec.md`
- **Repos:** exe-api (`services/api`) · exe-admin

## 1. Kiến trúc & quyết định
- **Giữ nguyên backbone:** 1 doc `CourseStructure` lồng nhau; sửa qua **whole-tree PUT `updateCourse`** (không thêm endpoint per-node). Drill-down = **điều hướng FE** trên cây đã load. *Lý do:* admin ít concurrency, course nhỏ → per-node là YAGNI (nhiều endpoint + validate cục bộ + version).
- **Model không đổi:** `estimatedDurationMin` đã tồn tại. Chỉ sửa luồng đọc/ghi bỏ sót nó.
- **Nguồn heuristic duy nhất ở BE** (`durationOf`): FE **không** nhân bản bảng heuristic — nhận `effectiveDurationMin` từ API.

## 2. exe-api

### 2.1 Vá `buildTree` (IS-1) — `course-content.service.js`
Thêm `estimatedDurationMin` vào keep-list lesson:
```
strip(l, ['key','title','order','type','contentUrl','theory','estimatedDurationMin'])
```
→ `importCourse` và `updateCourse` (cả hai gọi `buildTree`) ngừng xoá trắng. `null` vẫn round-trip là `null` (hợp lệ, min:1 chỉ áp khi có giá trị).

### 2.2 Parser cột `DurationMin` (IS-2) — `course-content.parser.js`
- Header hiện A–M (1–13). **Thêm cột N (14) = `DurationMin`**, chỉ đọc ở LESSON row (các row khác bỏ qua). Append cuối → file cũ thiếu cột N ⇒ blank ⇒ null ⇒ heuristic (tương thích ngược).
- Parse: `const dRaw = cell(row, 14)` → nếu rỗng `estimatedDurationMin: null`; else `Number(dRaw)`.
- Validate (trong `validateTree`/parse-time): nếu có mà không phải **integer > 0** → push error `INVALID_DURATION` kèm `row` (đúng dòng, không chặn dòng khác — cùng pattern `INVALID_LESSON_TYPE`).

### 2.3 Template + Export round-trip (IS-3)
- Template (`course-content.admin.js §2`): thêm header `DurationMin` (cột N) + để trống ở row LESSON mẫu (minh hoạ optional).
- Export (`course-content.export.service.js` `buildCourseXlsx`): ghi `estimatedDurationMin` vào cột N cho mỗi LESSON row. Import lại file export = không đổi (AC-4).

### 2.4 `effectiveDurationMin` ở detail (IS-4) — contract exe-api → exe-admin
- `§4 GET /:id` (detail cả cây): mỗi lesson thêm field **read-only** `effectiveDurationMin = durationOf(lesson)` (không lưu DB, chỉ shape response). `estimatedDurationMin` (thô, editable) vẫn trả song song.
- **Chỉ ở detail** (không ở list `§3`). Không đưa `effectiveDurationMin` vào body PUT (client gửi lại sẽ bị bỏ qua — chỉ `estimatedDurationMin` được ghi).

## 3. exe-admin

### 3.1 Types (IS-5) — `services/course-content.service.ts`
- `LessonNode` thêm `estimatedDurationMin?: number | null` (editable) và `effectiveDurationMin?: number` (read-only từ detail).
- `updateCourse` payload gửi `estimatedDurationMin` (không gửi `effectiveDurationMin`).

### 3.2 Editor drill-down (IS-6) — tách `course-import/course-detail.tsx`
- **Container** giữ **cả cây** trong state + `dirty`/save/whole-tree PUT **như hiện tại** (không đổi cơ chế lưu/publish-lock).
- Thêm **state điều hướng** `selectedPhaseKey`/`selectedModuleKey` (hoặc route param nội bộ) → render slice tương ứng:
  - **Cấp khóa:** sửa meta khóa + list chặng (bấm 1 chặng → xuống cấp chặng).
  - **Cấp chặng:** sửa meta chặng + checkpoint + list chuyên đề (bấm → cấp chuyên đề).
  - **Cấp chuyên đề:** sửa meta chuyên đề + **list lesson sửa inline** (title/type/contentUrl/theory/**estimatedDurationMin**/media/exercises). Không có cấp lesson riêng.
- Breadcrumb + nút "lên cấp"; chuyển cấp **không mất sửa** (state cây ở container). 1 nút Lưu (whole-tree) như cũ.
- Tách sub-component render từng cấp (vd `course-tree-nav`, `phase-editor`, `module-editor`) — file < 200 dòng, DRY input helpers.

### 3.3 Duration field + rollup (IS-7)
- Ô `estimatedDurationMin` ở mỗi lesson (number, trống = null = "auto theo loại"; hiển thị gợi ý effective bên cạnh).
- **Rollup read-only** = tổng `effectiveDurationMin` các lesson: ở chuyên đề (tổng lesson), chặng (tổng chuyên đề), khóa (tổng chặng). Tính thuần FE từ `effectiveDurationMin` đã có trong cây detail.
- Lưu ý: sau khi admin sửa `estimatedDurationMin` nhưng **chưa Lưu/refetch**, `effectiveDurationMin` (từ server) chưa cập nhật → rollup dùng **giá trị hiệu dụng phía client**: `estimatedDurationMin ?? effectiveDurationMin` (khi field trống, effective = heuristic từ server vẫn đúng). Ghi rõ để không lệch khi đang sửa.

## 4. Edge cases
- File cũ thiếu cột N → mọi lesson null → heuristic (không vỡ).
- Lesson `estimatedDurationMin=null` → export ghi ô trống (không ghi 0).
- Khóa `published/archived` → editor read-only (giữ), rollup vẫn hiện.
- `DurationMin` số thực (vd 12.5) → quyết: **reject** (chỉ integer) hay round? → **reject + error** (nhất quán "phút nguyên").

## 5. Không tạo (YAGNI)
Endpoint per-node · đổi contract PUT · sửa generator lịch · trang lesson riêng · lưu `effectiveDurationMin` vào DB · migration/backfill.

## 6. Files đụng
**exe-api:** `course-content.parser.js` · `course-content.service.js` (buildTree, validateTree) · `course-content.export.service.js` · `admin-api/resources/course-content.admin.js` (§2 template, §4 detail shape) · tests (`__tests__/*` course-content parser/service/export).
**exe-admin:** `services/course-content.service.ts` · `components/features/course-import/course-detail.tsx` (+ sub-view chặng/chuyên đề mới) · tests.

## 7. Câu hỏi mở (cần Tech Lead chốt khi review design)
- Cột N cho `DurationMin` — OK hay muốn vị trí khác?
- Số thực → reject (đề xuất) hay round?
- Có cần diagram điều hướng drill-down không (mục 3.2 đủ rõ chưa)?
