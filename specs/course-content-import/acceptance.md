# Acceptance Criteria: Import cấu trúc Khóa học tĩnh (Phase 1)

- **Spec:** `specs/course-content-import/spec.md`
- **Ngày:** 2026-07-15

## Cách dùng file này

- Mỗi tiêu chí có ID (`AC-1`…) để `design.md`/`tasks.md`/`review.md` tham chiếu ngược.
- Given/When/Then cho hành vi phụ thuộc trạng thái; checklist cho tiêu chí đơn giản.
- Reviewer Agent đối chiếu từng ID với test/behavior thật — ID không map được tới kiểm tra nào là finding blocking.
- **Lưu ý:** vài AC phụ thuộc Câu hỏi mở trong spec (Q1–Q6). Các AC đó được đánh dấu **[CHỜ Qx]** và chưa chốt được cho tới khi câu hỏi tương ứng được trả lời — đây là lý do spec chưa đủ điều kiện duyệt.

## Tiêu chí chức năng

### AC-1: Import file hợp lệ tạo đủ cấu trúc phân tầng *(IS-1, IS-2, IS-6, IS-7)*

- **Given** một user có quyền quản trị nội dung, và một file cấu trúc khóa học hợp lệ mô tả 1 Lộ trình có N Chặng, mỗi Chặng có các Chuyên đề, mỗi Chuyên đề có các Bài học, mỗi Bài học tham chiếu tới các exercise ID **đã tồn tại**
- **When** user tải file lên trang Import và xác nhận
- **Then** hệ thống lưu đầy đủ cấu trúc phân tầng vào kho nội dung khóa học ở trạng thái "sẵn sàng", và trả về **tóm tắt đếm được** đúng số Chặng / Chuyên đề / Bài học đã tạo (khớp với nội dung file)

### AC-2: Từ chối file sai cấu trúc phân tầng *(IS-3, IS-7)*

- **Given** một file thiếu tầng bắt buộc hoặc có liên kết cha–con không hợp lệ (vd: Bài học không thuộc Chuyên đề nào, Chặng không thuộc Lộ trình)
- **When** user tải file lên
- **Then** hệ thống từ chối import và trả danh sách lỗi **chỉ rõ vị trí sai** (dòng/mục và loại lỗi cấu trúc), đủ để đội học thuật tự sửa; **không** tạo bản ghi nào

### AC-3: Từ chối khi exercise ID tham chiếu không tồn tại *(IS-4, IS-7)*

- **Given** một file mà ít nhất một Bài học trỏ tới một exercise ID (IPA/Talk/Quiz) **không tồn tại** trong hệ thống
- **When** user tải file lên
- **Then** hệ thống từ chối import và báo lỗi **liệt kê đúng những ID không tồn tại** (kèm loại IPA/Talk/Quiz và vị trí trong file); **không** tạo bản ghi nào

### AC-4: Import toàn-hoặc-không (all-or-nothing) *(IS-5)*

- **Given** một file mà phần đầu hợp lệ nhưng có ít nhất một mục lỗi ở phía sau (cấu trúc hoặc tham chiếu)
- **When** user tải file lên
- **Then** **không** có tầng nào (Lộ trình/Chặng/Chuyên đề/Bài học) được tạo — kho nội dung sau lần import lỗi giống hệt trước khi import (không có dữ liệu rác một phần)

### AC-5: Hiển thị kết quả import trên Admin *(IS-1, IS-7)*

- [ ] Sau khi import **thành công**, trang Import trên `exe-admin` hiển thị tóm tắt số tầng đã tạo (giá trị khớp AC-1).
- [ ] Sau khi import **thất bại**, trang Import hiển thị danh sách lỗi cụ thể (khớp lỗi ở AC-2/AC-3/AC-E1), không chỉ thông báo chung chung "import failed".

## Tiêu chí lỗi / edge case

### AC-E1: File không phân tích được (sai định dạng/hỏng) *(IS-2, IS-7)*

- **Given** một file không đúng định dạng chuẩn hoặc bị hỏng, không thể parse
- **When** user tải lên
- **Then** hệ thống trả lỗi rõ ràng cho biết file không đọc được (không phải lỗi 500 chung chung); **không** tạo bản ghi nào
- **(Q2 đã chốt: Excel `.xlsx`.)** "Định dạng chuẩn" = `.xlsx` theo template. File không phải `.xlsx` hoặc `.xlsx`
  hỏng/không parse được → **400** `PARSE_FAILED` (hoặc `INVALID_FILE_TYPE` nếu sai loại).

### AC-E2: File rỗng hoặc không có tầng nào *(IS-3)*

- **Given** một file rỗng, hoặc parse được nhưng không chứa Lộ trình/Chặng/Chuyên đề/Bài học nào
- **When** user tải lên
- **Then** hệ thống từ chối với thông báo "không có nội dung khóa học để import"; **không** tạo bản ghi nào

### AC-E3: Trùng ID nội bộ trong file *(IS-3)*

- **Given** một file mà hai mục cùng tầng bị trùng định danh (vd hai Bài học cùng mã trong cùng một Chuyên đề)
- **When** user tải lên
- **Then** hệ thống từ chối kèm lỗi chỉ rõ định danh bị trùng và vị trí; **không** tạo bản ghi nào

### AC-E4: User không đủ quyền *(IS-8)*

- **Given** một user **không** có quyền quản trị nội dung
- **When** user gọi chức năng Import (qua UI hoặc trực tiếp)
- **Then** thao tác bị từ chối vì thiếu quyền (không thực thi parse/ghi), và không có dữ liệu nào bị tạo

### AC-E5: Import trùng khóa học đã tồn tại *(IS-5, IS-6)* — **(Q4 đã chốt: từ chối)**

- **Given** một file có `slug` (định danh khóa/roadmap ID) trùng với một khóa **đã tồn tại** trong hệ thống
- **When** user tải file lên
- **Then** hệ thống **từ chối** import với **400** `IMPORT_VALIDATION_FAILED`, `errors[]` kèm `ALREADY_EXISTS` chỉ
  rõ định danh trùng + số dòng (vd "Dòng 45: khóa 'mod-01' đã tồn tại"); **không đè** dữ liệu cũ, **không** tạo bản
  ghi nào. (Sửa khóa đã import: xoá thủ công rồi import lại — versioning ngoài phạm vi Phase 1.)

### AC-E6: Tham chiếu tới bài tập ở trạng thái nháp/lưu trữ *(IS-4)* — **(Q3 đã chốt: chỉ published)**

- **Given** một file mà ít nhất một Bài học trỏ tới exercise ID **tồn tại nhưng chưa publish** (vd IPA lesson
  `status ∈ {draft, review, archived, generating}`)
- **When** user tải file lên
- **Then** hệ thống **từ chối** với **400** `IMPORT_VALIDATION_FAILED`, `errors[]` kèm `REFERENCE_NOT_PUBLISHED`
  chỉ rõ ID chưa publish + type + số dòng; **không** tạo bản ghi nào.

### AC-E7: Tham chiếu type `quiz` *(IS-4)* — **(Q-Quiz đã chốt: hoãn Phase 2)**

- **Given** một file có ít nhất một bài tập `type = quiz`
- **When** user tải file lên
- **Then** hệ thống **từ chối** với **400** `IMPORT_VALIDATION_FAILED`, `errors[]` kèm `UNSUPPORTED_EXERCISE_TYPE`
  ("quiz sẽ được hỗ trợ ở Phase 2") + số dòng; **không** tạo bản ghi nào. Phase 1 chỉ chấp nhận `ipa` + `talk`.

## Tiêu chí phi chức năng (nếu áp dụng)

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| Bảo mật | Chức năng Import yêu cầu quyền quản trị nội dung `<resource>:<action>`; user thiếu quyền bị từ chối (AC-E4) | Test tự động: gọi với user không quyền → bị từ chối; với user có quyền → cho phép |
| Bảo mật | **(Q5 đã chốt: dùng chung)** `centerId = null` cho mọi khóa Phase 1; quyền `coursecontent:*` chỉ cấp platform admin/đội học thuật, không vào `CENTER_PERMISSIONS` | Test: user center-staff (không có `coursecontent:write`) → 403; import không set `centerId` từ body |
| Audit | Mỗi lần import thành công được ghi nhận qua audit hook của admin-api factory | Kiểm tra bản ghi audit `admin.command.executed` (`coursecontent.import`) tồn tại sau import |
| Hiệu năng | **(Q6 đã chốt)** Xử lý **đồng bộ**; file ≤ **5MB** (≤ ~5.000 bài học) xử lý xong trong 1 request (vài giây) | Test: file quá 5MB → 400 `FILE_TOO_LARGE`; file hợp lệ cỡ lớn (gần ngưỡng) → 200 trong thời gian request bình thường |

## Ngoài phạm vi kiểm thử

Các phần đã liệt kê "Ngoài phạm vi" ở `spec.md` §3 — nhắc lại để Reviewer Agent không flag nhầm là thiếu coverage:

- Render/hiển thị và logic mở khóa Bài học cho learner (Phase 2).
- Theo dõi tiến độ/mastery/streak (Phase 3).
- Luồng dựng khóa thủ công kéo-thả trên CMS UI.
- Tạo/sửa nội dung bài tập IPA/Talk/Quiz (Phase 1 chỉ tham chiếu ID có sẵn).
- Refactor prompt AI ra khỏi source code (Epic 2).
- Learner surface trên `web`/`mobile`.
