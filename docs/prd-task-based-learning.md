<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8. PRD cấp sản phẩm cho epic "Học theo Lộ trình". -->
# PRD — Học theo Lộ trình (Task-Based Learning)

| | |
|---|---|
| **Phiên bản** | 1.0 |
| **Ngày** | 2026-07-15 |
| **Chủ tài liệu (Owner)** | Product Owner |
| **Đóng góp** | BA Agent, Tech Lead Agent |
| **Trạng thái** | Duyệt định hướng (Phase 1 đang triển khai; Phase 2–5 ở mức khung) |
| **Tài liệu liên quan** | Roadmap: `docs/roadmap-task-based-learning.md` · Spec Phase 1: `specs/course-content-import/` · ADR: `docs/adr/0001-course-content-store-embedded-tree.md` |
| **Nguồn ý tưởng** | `<API_REPO>/docs/feature_plan_task_based_learning.md`, `content_ingestion_schema.md`, `ai_ingestion_tool_plan.md` |

---

## 1. Tóm tắt điều hành (Executive Summary)

ENOVA/studee hiện có các engine luyện tập rời rạc (phát âm IPA, hội thoại AI Talk, ngân hàng câu hỏi Assessment) nhưng **chưa có một "khóa học" hoàn chỉnh** dẫn dắt người học đi từ trình độ này lên trình độ khác. Người học không có con đường rõ ràng; đội học thuật không có cách đưa một chương trình học vào hệ thống theo lô.

PRD này định nghĩa sản phẩm **"Học theo Lộ trình"**: mỗi khóa học là một cây **4 tầng — Lộ trình → Chặng → Chuyên đề → Bài học** — giúp người học **nâng cấp nền tảng tiếng Anh theo từng nấc** (A2→B1, B1→B2, IELTS 5.0→6.0…). Mỗi Bài học học **đủ lý thuyết + video + âm thanh**, phục vụ phát triển **toàn diện Nghe – Nói – Đọc – Viết**, có gắn bài tập tương tác.

Sản phẩm được giao theo **6 phase** (xem §9). **Phase 1 (Ingestion) — đang triển khai** — cho phép đội học thuật import khóa học bằng Excel và lưu nguyên tử vào hệ thống.

---

## 2. Vấn đề & Cơ hội

### 2.1. Bài toán người học
- **Thiếu lộ trình:** người học luyện IPA/Talk lẻ tẻ, không biết mình đang ở đâu và cần học gì tiếp để "lên trình".
- **Thiếu tính toàn diện:** các công cụ hiện tại nghiêng về phát âm và nói; Nghe/Đọc/Viết/Ngữ pháp/Từ vựng chưa được tổ chức thành nội dung học có hệ thống.
- **Thiếu cảm giác tiến bộ:** không có thước đo "tôi đã đi từ A2 tới đâu trên đường lên B1".

### 2.2. Nút thắt vận hành
- Đội học thuật **không có kênh tự phục vụ** để đưa một khóa học hoàn chỉnh vào hệ thống — mọi thứ phải qua dev.
- Không có mô hình dữ liệu chuẩn để mô tả một khóa học phân tầng có gắn bài tập.

### 2.3. Cơ hội
- **Tận dụng tài sản có sẵn:** IPA, Talk, Assessment, roadmap CEFR-adaptive, gamification/streak đã tồn tại → chỉ cần một lớp "khóa học" bọc lại và một kênh nhập liệu.
- **Định vị sản phẩm:** chuyển từ "bộ công cụ luyện tập" sang "nền tảng học có lộ trình" — tăng giữ chân (retention) và giá trị cảm nhận.

---

## 3. Mục tiêu & Chỉ số thành công

### 3.1. Mục tiêu sản phẩm
- **G1 — Nền tảng nội dung có cấu trúc:** có mô hình 4 tầng chuẩn, dùng chung cho mọi phase.
- **G2 — Tự phục vụ cho đội học thuật:** phát hành một khóa học vào hệ thống **không cần dev**.
- **G3 — Học toàn diện:** mỗi Bài học phục vụ đủ 4 kỹ năng ở mức nội dung, tiến tới đủ bài tập tương tác.
- **G4 — Đo được tiến bộ:** người học thấy rõ mình đang tiến từ nấc A2→B1 tới đâu (Phase 3).

### 3.2. Chỉ số (theo từng nấc trưởng thành)

| Mục tiêu | Chỉ số | Phase đo được |
|---|---|---|
| G2 | Số khóa import thành công / tháng; tỉ lệ import lỗi phải sửa lại; thời gian trung bình từ file → khóa `ready` | Phase 1 |
| G1 | 100% khóa mới dùng mô hình 4 tầng (không phát sinh mô hình rời) | Phase 1 |
| G3 | % Bài học có đủ theory + (video hoặc audio); số kỹ năng có bài tập tương tác | Phase 1 → Phase 4 |
| — | Tỉ lệ người học hoàn thành ≥1 Chuyên đề; thời lượng học trung bình/tuần | Phase 2 |
| G4 | % người học đạt mốc nâng trình (self-report hoặc placement lại); streak trung bình | Phase 3 |

> **Ghi chú PO:** các chỉ số Phase 2–3 chỉ đo được khi có learner surface. Phase 1 chỉ cam kết chỉ số vận hành nhập liệu.

---

## 4. Đối tượng người dùng (Personas)

| Persona | Mô tả | Nhu cầu cốt lõi | Phase phục vụ |
|---|---|---|---|
| **Đội học thuật / Admin nội dung** | Người biên soạn chương trình học | Đưa cả một khóa vào hệ thống nhanh, biết ngay đúng/sai để tự sửa | Phase 1 (import), Phase 5 (CMS/AI) |
| **Người học (Learner)** | Người dùng cuối muốn nâng trình | Có lộ trình rõ, học đủ 4 kỹ năng, thấy mình tiến bộ | Phase 2–4 |
| **Quản trị viên nền tảng** | Vận hành, phân quyền | Kiểm soát ai được tạo/xóa nội dung; đối soát | Phase 1+ |
| **Tech Lead / Dev** | Người tiêu thụ dữ liệu | Có kho khóa `ready` chuẩn để Phase 2 render | Xuyên suốt |

---

## 5. Tầm nhìn sản phẩm & Mô hình phân tầng

Mỗi khóa học là một cây 4 tầng, **chốt và hiện thực hóa ngay từ Phase 1**, các phase sau chỉ **tiêu thụ và làm giàu**:

| Tầng | Tên miền | Vai trò | Dữ liệu mang theo |
|---|---|---|---|
| 1 | **Lộ trình** (Course) | Một khóa học hoàn chỉnh | tên, mô tả, slug |
| 2 | **Chặng** (Phase) | Một **nấc nâng trình độ** | `cefrFrom→cefrTo` (A2→B1) hoặc `goalNote` ("IELTS 5.0→6.0") |
| 3 | **Chuyên đề** (Module) | Một **mảng kỹ năng/chủ đề** | `category`: ngữ pháp / phát âm / từ vựng / nghe / đọc / viết / nói |
| 4 | **Bài học** (Lesson) | Đơn vị học đa phương thức | **lý thuyết (markdown) + video (URL) + audio (URL)** + danh sách tham chiếu bài tập |
| 4↳ | Bài tập (Exercise ref) | Luyện tập tương tác | trỏ tới engine: `ipa`, `talk` *(sau: `quiz`, nghe/đọc/viết…)* |

**Nguyên tắc "học đủ":** ở tầng Bài học, người học tiếp cận lý thuyết đọc, video xem, audio nghe, và bài tập thực hành — bốn kỹ năng Nghe–Nói–Đọc–Viết được nuôi từ chính một Bài học.

---

## 6. Phạm vi sản phẩm

### 6.1. Trong phạm vi (toàn epic)
- Mô hình dữ liệu 4 tầng dùng chung.
- Kênh nhập nội dung (Phase 1: import Excel; Phase 5: CMS + AI generator).
- Bề mặt tiêu thụ cho learner (Phase 2) + đo lường tiến độ (Phase 3).
- Bài tập tương tác đủ 4 kỹ năng (Phase 1: ipa+talk; Phase 4: nghe/đọc/viết/ngữ pháp/từ vựng + quiz).

### 6.2. Ngoài phạm vi (toàn epic — hoặc hoãn có chủ đích)
- **Host media tập trung** (upload/serve video–audio): epic dùng **URL do đội học thuật tự host** cho tới Phase 5.
- **Sửa/versioning khóa qua UI:** tới Phase 5; trước đó dùng "xóa rồi import lại".
- **Đa tenant (center tự tạo khóa riêng):** epic mặc định nội dung **platform dùng chung** (`centerId=null`); model chừa sẵn cột để mở sau.
- **Đưa prompt AI ra khỏi source** (Epic 2 của idea gốc): hạng mục độc lập, không thuộc epic này.

---

## 7. Yêu cầu chức năng (Functional Requirements)

Đánh số theo tầng để truy vết. Cột "Phase" cho biết khi nào giao.

### 7.1. Nhập & quản trị nội dung
| ID | Yêu cầu | Phase |
|---|---|---|
| FR-IN-1 | Admin import **một** file Excel mô tả cả cây 4 tầng; hệ thống parse và trả kết quả. | 1 |
| FR-IN-2 | Validate **cấu trúc phân tầng** (đúng thứ bậc, liên kết cha–con hợp lệ, không mồ côi). | 1 |
| FR-IN-3 | Validate **tham chiếu bài tập** (mỗi ID `ipa`/`talk` phải tồn tại và ở trạng thái `published`). | 1 |
| FR-IN-4 | Validate **metadata nâng trình** (CEFR hợp lệ ở Chặng; category hợp lệ ở Chuyên đề; URL hợp lệ ở Bài học). | 1 |
| FR-IN-5 | Import **toàn-hoặc-không**: bất kỳ lỗi nào ⇒ không tạo dữ liệu một phần. | 1 |
| FR-IN-6 | Thành công ⇒ lưu nguyên tử, đặt trạng thái `ready`; báo tóm tắt số tầng đã tạo. | 1 |
| FR-IN-7 | Thất bại ⇒ trả **danh sách lỗi cụ thể** (dòng/mục/trường + mã lỗi) đủ để tự sửa. | 1 |
| FR-IN-8 | **Tải template** Excel chuẩn để đội học thuật điền. | 1 |
| FR-IN-9 | **Liệt kê** các khóa đã import (phân trang, có trạng thái). | 1 |
| FR-IN-10 | **Xem chi tiết** một khóa (toàn cây). | 1 |
| FR-IN-11 | **Xóa** một khóa (giải phóng slug để import lại). | 1 |
| FR-IN-12 | Phân quyền `coursecontent:<read/write/delete/manage>`; xóa cần quyền cao (`manage`). | 1 |
| FR-IN-13 | CMS kéo-thả dựng khóa trực tiếp trên UI. | 5 |
| FR-IN-14 | AI Course Generator: mô tả ngắn → sinh cây khóa để review & publish. | 5 |
| FR-IN-15 | Sửa/versioning khóa đã import (bỏ ràng buộc xóa-rồi-import-lại). | 5 |

### 7.2. Tiêu thụ (learner)
| ID | Yêu cầu | Phase |
|---|---|---|
| FR-RT-1 | Gán/ghi danh khóa cho người học (assign/enroll). | 2 |
| FR-RT-2 | Render cây Lộ trình→Chặng→Chuyên đề→Bài học cho người học. | 2 |
| FR-RT-3 | Mở Bài học: đọc lý thuyết, phát video/audio, mở bài tập `ipa`/`talk`. | 2 |
| FR-RT-4 | **Unlock tuần tự cơ bản** (mở bài/chặng kế khi hoàn thành) — chưa gắn điểm. | 2 |

### 7.3. Tiến độ & Mastery
| ID | Yêu cầu | Phase |
|---|---|---|
| FR-PR-1 | Theo dõi tiến độ per Bài học/Chuyên đề/Chặng. | 3 |
| FR-PR-2 | Tính **mastery** theo điểm bài tập; **streak**. | 3 |
| FR-PR-3 | **Unlock theo ngưỡng điểm** (vd ≥80% mới mở bài kế). | 3 |
| FR-PR-4 | Aggregate "đã tiến bộ A2→B1 tới đâu" — dùng `cefrFrom/cefrTo` gắn từ Phase 1. | 3 |

### 7.4. Bài tập đủ 4 kỹ năng
| ID | Yêu cầu | Phase |
|---|---|---|
| FR-EX-1 | Bài tập tương tác **Phát âm (ipa)** + **Nói (talk)**. | 1 |
| FR-EX-2 | Engine bài tập **Nghe / Đọc / Viết / Ngữ pháp / Từ vựng**. | 4 |
| FR-EX-3 | Mở loại **`quiz`** (chốt "một quiz = gì"; ứng viên: `assessment_questions`). | 4 |
| FR-EX-4 | Mở rộng enum `ExerciseRef.type` từ `['ipa','talk']` sang đủ loại. | 4 |

---

## 8. Hành trình & câu chuyện người dùng chính

**US-1 — Phát hành khóa học (Phase 1).** *Là* admin nội dung, *tôi muốn* điền file Excel cả cây khóa (kèm ID bài tập) rồi tải lên, *để* phát hành cả khóa vào hệ thống mà không cần dev. → khi có lỗi, thấy danh sách lỗi trỏ đúng dòng để tự sửa và thử lại.

**US-2 — Học theo lộ trình (Phase 2).** *Là* người học, *tôi muốn* mở khóa đã gán, xem lý thuyết + video + nghe audio + làm bài tập của từng Bài học, *để* học đủ 4 kỹ năng theo một con đường rõ ràng.

**US-3 — Cảm nhận tiến bộ (Phase 3).** *Là* người học, *tôi muốn* thấy mình đã đi bao xa trên nấc A2→B1 và giữ streak, *để* có động lực học tiếp.

**US-4 — Luyện đủ kỹ năng (Phase 4).** *Là* người học, *tôi muốn* làm bài tập tương tác cả Nghe/Đọc/Viết/Ngữ pháp/Từ vựng, *để* luyện tập chứ không chỉ đọc lý thuyết.

**US-5 — Dựng khóa không cần Excel (Phase 5).** *Là* admin nội dung, *tôi muốn* dựng khóa trực tiếp trên UI hoặc để AI sinh nháp, *để* soạn nhanh và sửa được về sau.

---

## 9. Kế hoạch phát hành theo Phase

Chi tiết ranh giới & phụ thuộc: `docs/roadmap-task-based-learning.md`.

| Phase | Tên | Giao gì (tóm tắt) | Bề mặt | Trạng thái |
|---|---|---|---|---|
| 0 | Nền tảng | Engine IPA/Talk/Assessment, admin factory | có sẵn | ✅ |
| **1** | **Ingestion** | Import Excel + template + list + detail + delete; nội dung tĩnh 4 kỹ năng + bài tập ipa/talk | api + admin | 🔄 Đang làm |
| 2 | Runtime | Learner render & học khóa; unlock cơ bản | api + web | ☐ |
| 3 | Tiến độ & Mastery | Progress/mastery/streak/unlock theo điểm | api + web | ☐ |
| 4 | Engine bài tập đủ 4 kỹ năng | Nghe/Đọc/Viết/Ngữ pháp/Từ vựng + quiz | api (+engine mới) | ☐ |
| 5 | Authoring nâng cao | CMS + AI generator + host media + versioning | api + admin | ☐ |

### Ma trận phủ 4 kỹ năng (điểm dễ hiểu nhầm — làm rõ với các bên)
| Kỹ năng | Nội dung tĩnh | Bài tập tương tác |
|---|---|---|
| Phát âm | Phase 1 | **Phase 1** (`ipa`) |
| Nói | Phase 1 | **Phase 1** (`talk`) |
| Nghe / Đọc / Viết | Phase 1 | Phase 4 |
| Ngữ pháp / Từ vựng | Phase 1 | Phase 4 (`quiz`) |

> **Cam kết PO:** ngay Phase 1, mọi kỹ năng có lý thuyết + video + audio. Luyện tập tương tác đầy đủ tới **Phase 4**. Đây là đánh đổi có chủ đích để giao Phase 1 sớm mà không chờ xây 5 engine mới.

---

## 10. Yêu cầu phi chức năng (NFR)

| Nhóm | Yêu cầu |
|---|---|
| **Hiệu năng** | Import đồng bộ, file ≤ 5MB (~5.000 bài học); phản hồi trong ngưỡng chấp nhận được cho thao tác đồng bộ. |
| **Toàn vẹn dữ liệu** | All-or-nothing tuyệt đối — ghi nguyên tử trên MongoDB standalone (không transaction đa-collection); dùng mô hình cây nhúng (ADR-0001). |
| **Bảo mật/phân quyền** | Chuẩn `<resource>:<action>`; tenant scope lấy từ server (`User.centerId`), không tin `req.body`; mọi mutation qua audit hook. |
| **An toàn nghiệp vụ** | Kênh nhập KHÔNG được lộ đường tạo/sửa cây bỏ qua validate (không dùng factory CRUD generic cho ghi). |
| **Khả dụng lỗi** | Thông báo lỗi có mã + vị trí (dòng/mục/trường), đội học thuật tự sửa được không cần dev. |
| **Ngôn ngữ** | Tài liệu/spec tiếng Việt; code tiếng Anh (project-context §8). |
| **Media** | Chỉ nhận URL; không host/serve media cho tới Phase 5. |

---

## 11. Phụ thuộc, Giả định, Ràng buộc

**Phụ thuộc:**
- Engine `ipa`, `talk`, `assessment` đã chạy độc lập — Phase 1 chỉ đọc để validate ID.
- Admin portal `exe-admin` + REST `/api/admin/*` — bề mặt quản trị.

**Giả định:**
- Đội học thuật quen Excel hơn JSON → chọn Excel cho Phase 1.
- Đội học thuật tự host được media (CDN/YouTube/R2).
- Nội dung Phase 1 là platform dùng chung (`centerId=null`).

**Ràng buộc:**
- MongoDB standalone (không replica set) → không transaction đa-collection.
- HTTP status theo repo: 400/401/403/404/409, **không** dùng 422.
- Không commit `.env`/secret; không `git add .`; PR base luôn `develop`.

---

## 12. Rủi ro & Giảm thiểu

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Cây khóa vượt giới hạn 16MB BSON của document nhúng | Không lưu được khóa rất lớn | Giới hạn file 5MB (~5.000 bài học); theo dõi; tách collection nếu chạm trần (ghi trong ADR-0001) |
| Đội học thuật điền sai file nhiều lần | Nản, tốn thời gian | Template chuẩn + thông báo lỗi trỏ đúng dòng + mã lỗi rõ |
| Hiểu nhầm "Phase 1 học đủ 4 kỹ năng có bài tập" | Kỳ vọng lệch | Ma trận §9 nêu rõ tĩnh vs tương tác; truyền đạt với các bên |
| Factory CRUD generic lộ POST/PATCH bỏ qua validate | Phá toàn vẹn all-or-nothing | Hand-mount chỉ read/delete; tạo khóa duy nhất qua đường import |
| Trùng slug khi import lại | Xung đột dữ liệu | Phase 1: từ chối nếu trùng; xóa để giải phóng slug; versioning ở Phase 5 |
| Media URL chết theo thời gian | Bài học hỏng | Chấp nhận rủi ro Phase 1 (validate định dạng URL); host tập trung ở Phase 5 |

---

## 13. Quyết định đã chốt & Câu hỏi mở

### 13.1. Đã chốt (PO, 2026-07-15) — chi tiết `specs/course-content-import/open-questions.md`
- **Q1:** Thực thể miền mới `course_structures`, tách khỏi `course` marketing.
- **Q2:** Chỉ Excel `.xlsx` (thêm `exceljs`).
- **Q3:** Chỉ chấp nhận bài tập `published`.
- **Q4:** Từ chối nếu trùng slug (400 + lỗi trỏ dòng); xóa để import lại.
- **Q5:** Nội dung platform dùng chung (`centerId=null`).
- **Q6:** Xử lý đồng bộ, file ≤ 5MB.
- **Q-Quiz:** Hoãn `quiz` sang Phase 4; Phase 1 chỉ `ipa` + `talk`.
- **Hướng B (Rich lessons):** Chặng mang CEFR, Chuyên đề mang category, Bài học mang theory + video + audio (URL). Nâng §2–§5 endpoint vào Phase 1.

### 13.2. Câu hỏi mở (chốt khi tới phase tương ứng)
- **Assign/enroll (Phase 2):** learner được gán khóa thế nào — tự chọn / theo placement / admin gán?
- **Định nghĩa `quiz` (Phase 4):** một quiz trong Bài học ánh xạ tới gì (tập `assessment_questions` hay khái niệm mini-test mới)?
- **Đa tenant (Phase 2+):** khi nào cho center tạo khóa riêng?
- **Versioning (Phase 5):** mô hình version/edit thay cho "xóa-rồi-import-lại".

---

## 14. Phụ lục — Liên kết

- **Roadmap epic:** `docs/roadmap-task-based-learning.md`
- **Spec Phase 1:** `specs/course-content-import/{spec,acceptance,research,design,data-model,open-questions,tasks}.md`
- **Contracts Phase 1:** `specs/course-content-import/contracts/admin-course-import.md`
- **ADR:** `docs/adr/0001-course-content-store-embedded-tree.md`
- **Ý tưởng gốc:** `<API_REPO>/docs/feature_plan_task_based_learning.md`, `content_ingestion_schema.md`, `ai_ingestion_tool_plan.md`
