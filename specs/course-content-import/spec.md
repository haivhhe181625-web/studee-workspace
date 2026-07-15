# Spec: Import cấu trúc Khóa học tĩnh (Admin Content Ingestion — Phase 1)

- **Ngày:** 2026-07-15
- **Tác giả:** BA Agent (AI Brainstorm) — chờ human BA review
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — **xây trong Phase này:** chức năng Import (thuộc admin surface) + kho lưu cấu trúc khóa học.
  - `admin` (`exe-admin`) — **xây trong Phase này:** trang Import cho đội học thuật tải file & xem kết quả.
  - `web` (`exe-web`) — **KHÔNG** trong Phase này (learner tiêu thụ nội dung là Phase 2).
  - `mobile` — **KHÔNG** trong Phase này (chưa có surface học tập; xem "Ngoài phạm vi").
- **Module liên quan (nếu là `exe-api`):**
  - `services/api/src/admin-api/` — nơi đặt action Import (theo HARD RULE admin-api trong `<API_REPO>/CLAUDE.md`: admin-api không chứa business logic, phải delegate xuống module service).
  - `services/api/src/modules/course/` — collection `course` hiện hữu (catalog đối tác B2B) — **liên quan vì va chạm khái niệm "Course Catalog"**, xem Câu hỏi mở.
  - `services/api/src/modules/ipa/`, `.../talk/`, `.../assessment/` — nguồn của các exercise ID (IPA lesson, Talk scenario, Quiz/assessment) mà file import tham chiếu tới; Phase 1 chỉ **đọc để validate tồn tại**, không sửa.
- **Trạng thái:** Câu hỏi mở Q1–Q6 + Q-Quiz **đã được PO chốt** ngày 2026-07-15 (xem `open-questions.md` và §7) →
  đủ điều kiện chờ BA duyệt chính thức. Design đã cập nhật theo (`design.md`, `data-model.md`, `contracts/`,
  `docs/adr/0001-...`).

## 1. Mục tiêu

Cho phép đội ngũ học thuật/Admin tải lên một file cấu trúc khóa học tĩnh (phân tầng Lộ trình → Chặng → Chuyên đề → Bài học, mỗi Bài học tham chiếu tới các bài tập IPA/Talk/Quiz đã có sẵn) và hệ thống tự động validate rồi lưu toàn bộ cấu trúc vào kho nội dung khóa học, sẵn sàng cho Phase 2 tiêu thụ.

## 2. Bối cảnh

**Vì sao cần:** Nút thắt vận hành được nêu trong idea (`exe-api/docs/feature_plan_task_based_learning.md`, Luồng 1 — Epic 4) là đội học thuật không có cách đưa một khóa học hoàn chỉnh vào hệ thống theo lô. Framing đã chốt (A "Lesson Wrapper" + C "CMS-Ingestion") chia làm 3 phase; **spec này chỉ là Phase 1 — Ingestion**. Việc render/khóa mở bài học cho learner (Phase 2) và theo dõi tiến độ/streak (Phase 3) tách sang spec riêng.

**Hiện trạng đã kiểm tra trong code (không đoán):**
- Kiến trúc phân tầng: `modules/roadmap/` đã có **2 tầng** (Roadmap template → Phase) với trạng thái khóa/mở (`locked/unlocked/in_progress/completed`) và pattern template → deep-copy sang `user_roadmaps`. **Chưa có** tầng Chuyên đề (Module) và Bài học (Lesson) như idea mô tả — đó là phần Phase 1/Phase 2 sẽ bổ sung.
- Các engine bài tập được tham chiếu đã tồn tại và chạy độc lập: `modules/ipa/` (curriculum phát âm: `ipa_lessons`…), `modules/talk/` (nhập vai AI — scenario hiện hardcode trong `talk.scenarios.js`), `modules/assessment/` (bài test/quiz). Phase 1 **tái sử dụng** các ID của chúng, không tạo mới.
- `modules/course/` đã tồn tại nhưng là **catalog khóa học đối tác B2B để recommend** (`Course` chỉ đọc tag skills/subskills/cefr + `source: partner`), **không** chứa tham chiếu bài tập IPA/Talk/Quiz. → "Course Catalog" trong framing có khả năng là một kho nội dung khác; xem Câu hỏi mở.
- Admin surface: AdminJS đã bị gỡ bỏ hoàn toàn (ADR 2026-07-03). Admin nay là portal `exe-admin` (Next.js) + REST `/api/admin/*`. Chức năng Import mới phải theo mô hình này.
- **Chưa có** module `cms`/`practice`/import file nào trong codebase — đây thực sự là năng lực mới.

## 3. Phạm vi

### Trong phạm vi

- **IS-1** — Trang Import trên `exe-admin`: cho phép user có quyền quản trị nội dung tải lên **một** file cấu trúc khóa học.
- **IS-2** — Chức năng Import phía `exe-api` (admin surface): nhận file, phân tích (parse), trả về kết quả.
- **IS-3** — Validate **cấu trúc phân tầng**: file phải mô tả đúng thứ bậc Lộ trình → Chặng → Chuyên đề → Bài học và các liên kết cha–con hợp lệ.
- **IS-4** — Validate **tham chiếu bài tập**: mỗi exercise ID (IPA / Talk / Quiz) mà một Bài học trỏ tới phải **tồn tại thật** trong hệ thống.
- **IS-5** — **Import toàn-hoặc-không (all-or-nothing)**: nếu có bất kỳ lỗi validate nào, hệ thống KHÔNG tạo dữ liệu một phần.
- **IS-6** — Khi hợp lệ: lưu toàn bộ cấu trúc phân tầng vào kho nội dung khóa học, ở trạng thái "sẵn sàng" để Phase 2 tiêu thụ.
- **IS-7** — Phản hồi kết quả: khi thành công báo tóm tắt (số Chặng/Chuyên đề/Bài học đã tạo); khi thất bại trả **danh sách lỗi cụ thể** (chỉ rõ dòng/mục/trường nào sai và sai gì) đủ để đội học thuật tự sửa file.
- **IS-8** — Phân quyền: chỉ user có quyền quản trị nội dung phù hợp mới thực hiện được Import.

### Ngoài phạm vi

- Render/hiển thị Bài học cho learner, logic mở khóa (unlock) tuần tự khi học — **Lý do:** thuộc Phase 2 (Lesson Wrapper runtime), tách spec riêng để Phase 1 giao được độc lập.
- Theo dõi tiến độ, mastery, streak, aggregate điểm (`user-streak`, `progress.service`) — **Lý do:** thuộc Phase 3, và phần lớn đã có tiền lệ ở `modules/adaptive/` cần khảo sát riêng, không nhồi vào Phase 1.
- Luồng dựng khóa **thủ công kéo-thả trên CMS UI** (Luồng 2 trong idea) — **Lý do:** framing đã chọn ưu tiên luồng Import file; luồng thủ công là giải pháp thay thế cho cùng nhu cầu, làm sau nếu cần.
- **Tạo/sửa nội dung bài tập IPA/Talk/Quiz** — **Lý do:** các bài tập này đã có công cụ quản lý riêng; Phase 1 chỉ **tham chiếu** ID có sẵn, không tạo mới bài tập.
- Refactor đưa prompt AI ra khỏi source code (`ai-prompt.model` — Epic 2) — **Lý do:** là hạng mục độc lập, không cần thiết để import cấu trúc khóa học.
- Xuất/chỉnh sửa lại (export/edit) khóa học đã import qua giao diện — **Lý do:** Phase 1 chỉ giải quyết chiều đưa dữ liệu vào (ingestion); sửa/versioning là nhu cầu sau.
- Learner surface trên `web`/`mobile` — **Lý do:** không có người dùng cuối tiêu thụ ở Phase 1.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Đội học thuật / Admin nội dung | Điền sẵn file cấu trúc khóa học (kèm ID bài tập IPA/Talk/Quiz) rồi tải lên trang Import | Phát hành cả một khóa học vào hệ thống theo lô mà không cần dev can thiệp |
| Đội học thuật / Admin nội dung | Xem kết quả import (thành công hay danh sách lỗi cụ thể) | Biết khóa đã vào hệ thống, hoặc tự sửa file khi có lỗi và thử lại |
| Tech Lead / Dev (gián tiếp) | Có kho nội dung khóa học ở trạng thái "sẵn sàng" sau import | Phase 2 có nguồn dữ liệu chuẩn để render cho learner |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| "Course Catalog" trong framing là **tái dùng** collection `course` hiện có (catalog đối tác B2B) hay là **kho nội dung khóa học có cấu trúc mới**? Hai thứ này khác bản chất miền. | Coi là **thực thể miền mới** (khóa học có lộ trình + bài tập), tách khỏi `course` marketing — nhưng cần chủ sản phẩm xác nhận vì ảnh hưởng toàn bộ ý nghĩa "khóa học". | Product owner |
| Định dạng file chuẩn để import: **Excel**, **JSON**, hay **cả hai**? | Chốt **một** định dạng cho Phase 1 để thu hẹp phạm vi; đội học thuật quen Excel hơn. | Product owner + đội học thuật |
| Đa tenant: khóa học import thuộc **center nào** (hay là nội dung platform dùng chung)? | Cần làm rõ trước khi thiết kế phân quyền/scope. | Product owner |
| Import lại **cùng một khóa** đã tồn tại: **đè cập nhật**, **tạo phiên bản mới**, hay **từ chối**? Ảnh hưởng chính sách lưu trữ/versioning. | Phase 1 đề xuất **từ chối nếu trùng** (đơn giản, an toàn), versioning để sau. | Product owner |
| Chính sách lưu trữ file gốc đã import (giữ lại để đối soát hay không)? | Cần quyết vì liên quan retention. | Product owner |

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map về một mục "Trong phạm vi".

1. Tải file hợp lệ → hệ thống lưu đủ cấu trúc phân tầng và báo tóm tắt số tầng đã tạo. *(IS-1, IS-2, IS-6, IS-7)*
2. File sai cấu trúc phân tầng (thiếu tầng, liên kết cha–con sai) → bị từ chối kèm lỗi chỉ rõ vị trí. *(IS-3, IS-7)*
3. File tham chiếu exercise ID không tồn tại → bị từ chối kèm lỗi chỉ rõ ID nào. *(IS-4, IS-7)*
4. Bất kỳ lỗi validate nào → không có bản ghi nào được tạo (all-or-nothing). *(IS-5)*
5. User không đủ quyền quản trị nội dung → không import được. *(IS-8)*
6. Kết quả import (thành công/lỗi) hiển thị rõ trên trang Import của `exe-admin`. *(IS-1, IS-7)*

## 7. Câu hỏi mở

> **Đã chốt toàn bộ (PO, 2026-07-15).** Chi tiết lý do: `open-questions.md`.

- [x] **Q1:** → **Thực thể miền mới** (`course_structures`), tách khỏi `course` marketing.
- [x] **Q2:** → **Chỉ Excel `.xlsx`** (thêm dependency `exceljs`, ghi ADR-0001).
- [x] **Q3:** → **Chỉ chấp nhận bài tập đã `published`**.
- [x] **Q4:** → **Từ chối nếu trùng** (400 kèm lỗi trỏ dòng); không đè, không versioning ở Phase 1.
- [x] **Q5:** → **Nội dung platform dùng chung** (`centerId: null`); quyền chỉ platform admin/đội học thuật.
- [x] **Q6:** → **Xử lý đồng bộ**, file ≤ 5MB (~5.000 bài học).
- [x] **Q-Quiz (mới, do Tech Lead phát hiện):** → **Hoãn `quiz` sang Phase 2**; Phase 1 chỉ `ipa` + `talk`.

## 8. Ghi chú cho Tech Lead Design

> Ràng buộc đã biết, KHÔNG phải giải pháp (giải pháp thuộc `design.md`).

- **HARD RULE admin-api** (`<API_REPO>/CLAUDE.md` §Admin Rules): action Import đặt trong `src/admin-api/` **không được chứa business logic** — mọi thao tác có side effect (parse + INSERT phân tầng) phải delegate xuống một module service. Mọi mutation phải đi qua audit hook của factory.
- **Phân quyền** theo chuẩn `<resource>:<action>` (`src/constants/permissions.js` là nguồn sự thật duy nhất); tenant scope lấy từ `User.centerId` phía server, **không** tin `req.body`.
- **All-or-nothing** (IS-5) là ràng buộc dữ liệu quan trọng — Tech Lead cần cân nhắc tính nguyên tử của thao tác ghi nhiều tầng (MongoDB không có transaction mặc định giữa nhiều collection nếu không cấu hình).
- Trạng thái "sẵn sàng cho Phase 2" (IS-6) cần một quy ước rõ ràng để Phase 2 phân biệt khóa đã import xong với khóa lỗi dở — Tech Lead thống nhất quy ước này với spec Phase 2.
- Không giả định `course` (marketing catalog) là nơi lưu — chờ Q1 được chốt; nếu là kho mới, cân nhắc vị trí module cho hợp domain.
- Validate tham chiếu (IS-4) đọc chéo sang `ipa`/`talk`/`assessment` — Tech Lead lưu ý coupling này khi thiết kế (chỉ đọc, không sửa các module đó).
