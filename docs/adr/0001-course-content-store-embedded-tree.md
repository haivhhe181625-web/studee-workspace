<!-- Tiếng Việt — theo format đã có tiền lệ ở <API_REPO>/docs/ADR-admin-api-migration.md -->
# ADR-0001 — Lưu cấu trúc khóa học tĩnh thành một embedded-tree document (module `course-content`) + nhập bằng Excel

**Ngày:** 2026-07-15 · **Trạng thái:** Đề xuất (PO đã chốt các câu hỏi mở; chờ Technical Review) · **Thay thế:** không có

> Ghi cùng thiết kế feature `specs/course-content-import/` (Phase 1). Là ADR đầu tiên tạo trực tiếp trong
> `docs/adr/` (số 0001, sau ADR admin-api migration giữ ở vị trí cũ). Cập nhật `docs/adr/0000-index.md` kèm ADR
> này.

## Bối cảnh

Feature "Import cấu trúc khóa học tĩnh" (Phase 1) cần lưu một cây phân tầng Lộ trình → Chặng → Chuyên đề → Bài học
(mỗi Bài học tham chiếu bài tập IPA/Talk/Quiz có sẵn) với **hai ràng buộc cứng**:

1. **All-or-nothing (IS-5 / AC-4):** lỗi validate bất kỳ ⇒ KHÔNG tạo dữ liệu một phần.
2. Là **năng lực nội dung mới**, không có sẵn nơi lưu phù hợp.

Đã kiểm chứng trong code (không đoán):

- **MongoDB dev là standalone (`mongo:7`, không `--replSet`)** → multi-document transaction không dùng được.
  `payment.service.js` đã có fallback ghi **non-atomic** khi phát hiện thiếu transaction (`_isNoTransactionSupport`)
  — chấp nhận cho 2 bản ghi payment, nhưng để lại **dữ liệu rác một phần** nếu áp cho cây nhiều tầng ⇒ vi phạm
  IS-5. (Bằng chứng đầy đủ: `specs/course-content-import/research.md`.)
- `modules/roadmap/` (`roadmap_templates`) là thực thể **CEFR-adaptive 2 tầng nhúng**, không có Chuyên đề/Bài
  học/tham chiếu bài tập — sai ngữ nghĩa miền nếu nhồi cấu trúc khóa học tĩnh vào.
- `modules/course/` là **catalog marketing đối tác B2B** (`source: partner`); permission `course:*` đã bị chiếm.
- `modules/practice/` **tồn tại nhưng rỗng** (stub) — không có schema để tái dùng.

## Quyết định

1. **Tạo module mới `services/api/src/modules/course-content/`** (pattern 5-file) sở hữu collection mới
   `course_structures`. KHÔNG mở rộng `roadmap` (sai ngữ nghĩa) và KHÔNG dùng `course` (catalog marketing). Đây là
   lựa chọn có chủ đích, không mặc định.
2. **Lưu mỗi khóa học thành MỘT embedded-tree document** (cả cây Chặng→Chuyên đề→Bài học nhúng trong 1 document),
   ghi bằng một lần `create()`/`insertOne`. Ghi 1 document là **nguyên tử trên standalone lẫn replica set** ⇒ đáp
   ứng all-or-nothing **mà không phụ thuộc transaction** và không cần đổi hạ tầng. Noi theo tiền lệ nhúng
   `roadmap_templates.phases[]`.
3. **Validate toàn bộ (parse → cấu trúc → tham chiếu) TRƯỚC khi ghi**; chỉ `create()` khi 0 lỗi. Không ghi thăm dò
   từng tầng.
4. **Tham chiếu bài tập là "soft reference" `{ type, refId: String }`**, KHÔNG Mongoose `ref`/ObjectId — vì Talk
   scenario là hằng in-code (không có collection) và IPA phân giải theo `code`. Toàn vẹn tham chiếu kiểm ở **thời
   điểm import** qua một **reference resolver registry theo type**.
5. **Permission mới `coursecontent:read/write/manage`** trong `src/constants/permissions.js` (không tái dùng
   `course:*`). Phase 1 **chỉ cấp cho platform admin/đội học thuật**, KHÔNG đưa vào `CENTER_PERMISSIONS` (Q5 —
   nội dung dùng chung, `centerId: null`).
6. **Định dạng nhập là Excel `.xlsx`, thêm dependency `exceljs`** (Q2). Đây là dependency ngoài mới — chấp nhận vì
   giá trị cốt lõi Epic 4 là để đội học thuật non-tech soạn trực tiếp trên Excel (không cần dev trung gian chuyển
   JSON). Parser cô lập sau một adapter (`course-content.parser.js`) trả IR trung tính-định-dạng, nên đổi/bổ sung
   định dạng sau này rẻ. Giảm thiểu rủi ro parse file: giới hạn 5MB + xử lý trong memory + không eval nội dung ô.
7. **Phạm vi type bài tập Phase 1 = `ipa` + `talk`; `quiz` hoãn Phase 2** (Q-Quiz) — vì "quiz" chưa có nguồn nội
   dung tái dùng rõ ràng để validate tồn tại (`assessments` là phiên của user, `assessment_questions` là item lẻ).

## Lộ trình

| Phase | Nội dung | Trạng thái |
|---|---|---|
| 1 | Ingestion: model + import (spec `course-content-import`) | ☐ (đang design) |
| 2 | Learner tiêu thụ nội dung (render + unlock) — spec riêng, đọc `status: 'ready'` | ☐ |
| 3 | Tiến độ/mastery/streak — spec riêng | ☐ |

## Hệ quả

- **Cần cập nhật theo:** `docs/adr/0000-index.md` (thêm dòng ADR-0001); `src/constants/permissions.js` (bỏ comment
  placeholder LESSON/SECTION nếu dùng, hoặc thêm `coursecontent:*`); `<API_REPO>/docs/api/CONVENTIONS.md` §"Tạo
  module mới" áp cho `course-content`.
- **Đánh đổi đã chấp nhận:**
  - Trần **16MB/BSON document** — rất rộng với khóa học Phase 1 (Q6: file ≤5MB / ≤~5.000 bài học); giảm thiểu
    bằng ngưỡng số node ở parser (`research.md`).
  - Embedded tree **kém linh hoạt cho query xuyên tầng** ở quy mô lớn (Phase 2/3). Ưu tiên Phase 1 là ghi nguyên
    tử + đọc nguyên khối; tách collection để sau nếu phát sinh nhu cầu query nặng.
  - **Dependency ngoài mới `exceljs`** (điểm 6) — bề mặt parse file; giảm thiểu như trên.
  - **Re-import để sửa bị chặn** (Q4 từ chối trùng) — sửa khóa đã import phải xoá thủ công rồi import lại;
    versioning/edit là nhu cầu sau.
- **KHÔNG supersede ADR nào.** Tương thích ADR admin-api (2026-07-03): Import là admin-api action delegate xuống
  service, đúng HARD RULE.
- **Trạng thái quyết định:** các câu hỏi mở của spec (Q1–Q6, Q-Quiz) **đã được PO chốt** (xem
  `specs/course-content-import/open-questions.md`) và đã hợp nhất vào các điểm quyết định 1–7 ở trên. Nâng ADR lên
  "Chấp nhận" sau Technical Review + khi spec được BA duyệt chính thức.
