# Design: Import cấu trúc Khóa học tĩnh (Admin Content Ingestion — Phase 1)

- **Spec:** `specs/course-content-import/spec.md`
- **Ngày:** 2026-07-15 (cập nhật sau khi PO chốt Q1–Q6 + Q-Quiz)
- **Tác giả:** Tech Lead Agent (AI) — chờ Technical Review (human Tech Lead)
- **Trạng thái:** ✅ Đã duyệt Technical Review (2026-07-15), Q4 = phương án **(a)**. **Cập nhật mở rộng nội dung
  (Hướng B, 2026-07-15):** thêm ngữ nghĩa CEFR ở Chặng, phân loại ở Chuyên đề, và nội dung lý thuyết/video/audio ở
  Bài học (IS-9, IS-10). `tasks.md` đã cập nhật theo. Mở rộng này KHÔNG đổi kiến trúc lõi (vẫn embedded-tree một
  document, import qua admin-api) — chỉ thêm field + validate.
- **ADR liên quan:** `docs/adr/0001-course-content-store-embedded-tree.md`

## 0. Bối cảnh đã kiểm chứng trong code (không đoán từ spec)

Đã đọc code thực tế trước khi thiết kế; các phát hiện điều chỉnh vài giả định của spec:

| Điều spec/idea nói | Thực tế trong code | Ảnh hưởng thiết kế |
|---|---|---|
| "Chưa có module practice" (spec §2) | `src/modules/practice/` **tồn tại nhưng RỖNG** (stub 13/07) | Không tái dùng stub; module mới `course-content` (§1). Khuyến nghị xoá stub `practice` — ngoài phạm vi feature này. |
| Idea docs: mỗi dòng EXERCISE **tạo mới** bài tập (sinh phoneme/TTS/AI prompt) | Spec Phase 1 (IS-4) chỉ **tham chiếu ID bài tập có sẵn** | Design bám **spec**. Luồng tạo bài tập khi import là việc sau (ngoài phạm vi). |
| "Course Catalog" có thể là collection `course` | `modules/course/` = catalog marketing B2B (`source: partner`, không tham chiếu bài tập); `course:*` đã bị chiếm | **Q1 chốt: thực thể miền mới.** Module `course-content`, permission `coursecontent:*`. |
| Tham chiếu "IPA / Talk / Quiz" đồng nhất | **3 nguồn khác nhau:** IPA → `ipa_lessons` (có `code`+`status`); Talk → hằng in-code `SCENARIO_IDS` (không DB); Quiz → **không có nguồn tái dùng rõ ràng** | **Q-Quiz chốt: hoãn `quiz` sang Phase 2.** Phase 1 chỉ `ipa` + `talk`. |
| All-or-nothing "cân nhắc transaction" (§8) | MongoDB dev **standalone, không replica set**; `payment.service.js` fallback ghi **non-atomic** | Lưu **1 embedded document** (ghi 1 doc nguyên tử) — `research.md`. |

## 1. Tóm tắt kiến trúc

Thêm **module mới** `services/api/src/modules/course-content/` (pattern 5-file) sở hữu collection mới
`course_structures`. Mỗi khóa học import được lưu thành **MỘT document nhúng cả cây** Lộ trình → Chặng
(Phase) → Chuyên đề (Module) → Bài học (Lesson); mỗi Bài học chứa danh sách tham chiếu bài tập `{ type, refId }`
(Phase 1 chỉ `ipa`/`talk`). Ghi 1 document là **nguyên tử trên MongoDB standalone lẫn replica set** ⇒ đáp ứng
IS-5/AC-4 mà không phụ thuộc transaction (bằng chứng: `research.md`).

Chức năng Import là **collection-level action của admin-api factory** (`_crud.factory.js` hỗ trợ sẵn `actions` với
`collection: true`, tự audit) tại `POST /api/admin/course-imports/_/import`. Theo HARD RULE admin-api
(`<API_REPO>/CLAUDE.md`), action **không chứa business logic** — chỉ nhận file `.xlsx` (qua multer) rồi
**delegate xuống `course-content.service.importCourse(...)`**. Toàn bộ parse Excel → validate cấu trúc → validate
tham chiếu → build document → ghi nằm trong service.

**Không** mở rộng `roadmap` (CEFR-adaptive, 2 tầng nhúng, không có Module/Lesson/tham chiếu bài tập) và **không**
dùng `course` (catalog marketing). Module mới là lựa chọn có chủ đích (bảng §0). Feature đọc chéo `ipa`/`talk`
**chỉ để validate tồn tại** — không import, không sửa.

Phía `exe-admin`: trang Import trong route group `(admin)` gọi endpoint qua `multipart/form-data`, hiển thị tóm
tắt (thành công) hoặc danh sách lỗi cụ thể trỏ đúng dòng Excel (thất bại). Contract api↔admin:
`contracts/admin-course-import.md`.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Nơi lưu (Q1) | Collection mới `course_structures`, module `course-content` | `roadmap`/`course` sai ngữ nghĩa; `course:*` đã bị chiếm | +1 module, +1 permission prefix; giữ tách bạch với `course` marketing |
| Hình dạng lưu | **1 embedded document / khóa học** | Ghi 1 doc nguyên tử → IS-5/AC-4 không cần transaction; khớp `roadmap_templates.phases[]` | Trần 16MB BSON; query xuyên tầng kém linh hoạt (chấp nhận Phase 1) |
| Định dạng file (Q2) | **Chỉ Excel `.xlsx`** + dependency `exceljs` | Giải phóng đội học thuật non-tech (giá trị cốt lõi Epic 4); không cần dev trung gian | +1 dependency ngoài (ghi ADR-0001); parse phức tạp hơn JSON |
| Nơi đặt Import | Collection action `/_/import`, delegate service | Đúng HARD RULE admin-api; tự có audit + verifyPermission | Factory chưa hỗ trợ upload → chèn multer middleware trước handler (§6) |
| Validate rồi mới ghi | Parse → validate **toàn bộ** (cấu trúc + trùng nội bộ + tồn tại DB + tham chiếu) → chỉ ghi khi 0 lỗi | All-or-nothing ở tầng ứng dụng; không ghi thăm dò | Giữ cả cây trong RAM khi validate (chấp nhận, file ≤5MB) |
| Validate tham chiếu (Q3) | Resolver registry theo type; IPA yêu cầu `status: 'published'`; Talk theo `SCENARIO_IDS` | 3 nguồn khác bản chất; chỉ nhận bài đã publish để không vỡ UI learner (Phase 2) | Thêm 1 lớp abstraction |
| Phạm vi type bài tập (Q-Quiz) | **Phase 1 chỉ `ipa` + `talk`; `quiz` hoãn Phase 2** | Quiz không có nguồn tái dùng rõ ràng để validate; ship Phase 1 độc lập | File có `quiz` bị từ chối kèm lỗi rõ; enum mở rộng sau |
| Trùng khóa (Q4) | **Từ chối**; phát hiện trong pha validate, báo lỗi trỏ dòng; `slug` unique làm safety net | Đè rủi ro hỏng khóa active; versioning là over-engineering Phase 1 | Không cho re-import để sửa — phải xoá thủ công rồi import lại (chấp nhận Phase 1) |
| Đa tenant (Q5) | **Platform dùng chung** `centerId: null`; quyền chỉ platform admin/đội học thuật | Lộ trình lõi B2C của platform; đơn giản hóa scope | Center chưa tự tạo khóa riêng (model chừa cột `centerId` để mở rộng) |
| Kích thước / xử lý (Q6) | **Đồng bộ** trong request; file ≤ **5MB**, ≤ ~**5.000 bài học** | Vài giây/1 request; tránh Message Queue/Background phức tạp Phase 1 | Không hợp file rất lớn → đường mở BullMQ để sau |
| Trạng thái sẵn sàng (IS-6) | Field `status: 'ready'` | Phase 2 phân biệt khóa import xong | Vì ghi 1 doc nguyên tử nên không có trạng thái "dở"; `status` mang tính quy ước forward-compat |
| **Nội dung phong phú (IS-9, IS-10 — Hướng B)** | Thêm field: Chặng `cefrFrom/cefrTo/goalNote`; Chuyên đề `category`; Bài học `theory/videoUrl/audioUrl`. Media = **URL** (không upload/host). Bài học lý thuyết-thuần hợp lệ | Đúng tầm nhìn "mỗi tầng nâng trình độ, bài học đủ lý thuyết/video/âm thanh" ở chi phí thấp (chỉ thêm field + cột Excel, không cần engine mới) | Video/audio phụ thuộc content team tự host; Nghe/Đọc/Viết/Ngữ pháp/Từ vựng mới chỉ có **lý thuyết**, chưa có **bài tập tương tác** (cần engine — phase sau) |

## 3. Data model

Chi tiết đầy đủ ở `data-model.md`. Tóm tắt: 1 collection `course_structures`, 1 document = 1 khóa học, cây nhúng.

### 3.1 Document gốc `CourseStructure`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `slug` | String (kebab-case) | ✓ | Định danh khóa (roadmap ID trong Excel), **unique** — khóa tự nhiên phát hiện trùng (Q4/AC-E5) |
| `title` | String | ✓ | Tên Lộ trình |
| `description` | String | | |
| `centerId` | ObjectId → Center | | **Luôn `null` ở Phase 1** (Q5 — nội dung dùng chung). Cột giữ sẵn để mở rộng center-scoped sau |
| `status` | String enum `['ready']` | ✓ | IS-6 — "sẵn sàng Phase 2" |
| `phases` | `[PhaseSchema]` | ✓ (≥1) | Chặng — nhúng |
| `sourceMeta` | `{ filename, contentHash, nodeCount }` | | Metadata file `.xlsx` gốc (đối soát; `nodeCount` phục vụ ngưỡng an toàn) |
| `importedBy` | ObjectId → User | ✓ | Người import (audit) |
| `timestamps` | | auto | |

Cây nhúng: `phases[].modules[].lessons[].exercises[]`. Xem `data-model.md` cho schema con + index.

### 3.2 Parser Excel (`course-content.parser.js`)

`parse(buffer) -> NormalizedCourse | ParseError` dùng **`exceljs`** (đọc `.xlsx` từ memory buffer). Cột `Level`
(ROADMAP/PHASE/MODULE/LESSON/EXERCISE) dẫn dắt dựng cây từ các dòng liên tiếp. Trả về **IR**: cây object thuần,
mỗi node đính **số dòng Excel** để lỗi trỏ đúng dòng (IS-7 — vd "Dòng 45"). Parser cô lập định dạng.

**Cột file Excel Phase 1 (mở rộng Hướng B)** — mỗi dòng chỉ điền các ô liên quan tới `Level` của nó:

| Cột | Header | Dùng cho Level | Ý nghĩa |
|---|---|---|---|
| A | `Level` | tất cả | ROADMAP/PHASE/MODULE/LESSON/EXERCISE |
| B | `Key` | ROADMAP=`slug`, PHASE/MODULE/LESSON=`key` | định danh |
| C | `Title` | trừ EXERCISE | tên tầng |
| D | `Type` | EXERCISE | `ipa` / `talk` |
| E | `RefId` | EXERCISE | mã bài tập (IPA `code` / talk scenario id) |
| F | `CefrFrom` | PHASE | trình độ bắt đầu (A1..C2) |
| G | `CefrTo` | PHASE | trình độ đích |
| H | `Category` | MODULE | grammar/pronunciation/vocabulary/listening/reading/writing/speaking/other |
| I | `Content` | ROADMAP=`description`, LESSON=`theory` | mô tả / lý thuyết (markdown) |
| J | `VideoUrl` | LESSON | link video (http/https) |
| K | `AudioUrl` | LESSON | link audio (http/https) |
| L | `Note` | PHASE=`goalNote` | vd "IELTS 5.0 → 6.0" |

> Dòng `EXERCISE` mang **ID/mã bài tập tham chiếu** + `type` (khác luồng cũ tạo mới bài tập). Bài học có thể chỉ
> điền lý thuyết/video/audio mà không có dòng EXERCISE nào (Bài học lý thuyết-thuần — hợp lệ). Chốt template `.xlsx`
> với đội học thuật khi làm (tasks.md).

### 3.3 Reference resolver registry (`course-content.references.js`) — validate IS-4

```
resolvers = {
  ipa:  async (refIds) => IpaLesson.find({ code: { $in: refIds }, status: 'published' })
                            .select('code') → set các code hợp lệ (Q3: chỉ published)
  talk: (refIds) => refIds.filter(id => SCENARIO_IDS.includes(id))   // hằng in-code, KHÔNG query DB
  // quiz: HOÃN Phase 2 (Q-Quiz) — không đăng ký resolver ở Phase 1
}
```

Gom `refId` theo type → mỗi type 1 truy vấn theo lô (`$in`). Trả danh sách ID **không phân giải được** (không tồn
tại, hoặc IPA tồn tại nhưng chưa publish) kèm type + số dòng (AC-3, AC-E6). Nếu file chứa `type: 'quiz'` →
lỗi `UNSUPPORTED_EXERCISE_TYPE` ("quiz sẽ hỗ trợ ở Phase 2"). Chỉ đọc, không sửa `ipa`/`talk`.

## 4. Luồng dữ liệu

```
exe-admin (trang Import) --(POST multipart/form-data, field `file` = .xlsx)--> POST /api/admin/course-imports/_/import
  verifyToken                              (JWT — 401 nếu thiếu)
  -> verifyPermission('coursecontent:write')   (RBAC — 403; nạp req.user.permissions)
  -> courseImportUpload (multer.single('file'), memoryStorage, ≤5MB, mime .xlsx) (400 nếu quá cỡ/sai loại)
  -> asyncHandler(action.handler):         // admin-api action — KHÔNG business logic
       delegate → courseContentService.importCourse({ buffer, user })
         1. parser.parse(buffer)                  → NormalizedCourse | throw ApiError(400, PARSE_FAILED)   [AC-E1]
         2. validateStructure(course)             → gom lỗi: thiếu tầng/cha-con sai (AC-2), rỗng (AC-E2),
                                                     trùng key nội bộ (AC-E3), type quiz (Q-Quiz)
         3. checkNotDuplicate(course.slug)        → nếu slug đã tồn tại trong DB → thêm lỗi ALREADY_EXISTS (Q4/AC-E5)
         4. references.resolveAll(course)         → gom exercise ID không tồn tại / chưa publish (AC-3, AC-E6)
         5. nếu CÓ bất kỳ lỗi (2–4): throw ApiError(400, IMPORT_VALIDATION_FAILED, { errors }) — KHÔNG ghi  [AC-4/IS-5]
         6. build CourseStructure document (cả cây, centerId=null, status='ready')
         7. CourseStructure.create(doc)           → ghi 1 document (nguyên tử)                    [AC-1, IS-6]
              (E11000 do đua slug → map ApiError(400, IMPORT_VALIDATION_FAILED, ALREADY_EXISTS) — safety net)
         8. return summary { slug, counts: { phases, modules, lessons, exercises } }
  -> factory auto audit hook (admin.command.executed: coursecontent.import)                       [NFR Audit]
  -> 200 { data: { summary } }  |  400/401/403 { message, code, errors[] }
```

Xử lý **đồng bộ** trong request (Q6). Vài giây cho file ≤5MB.

## 5. Contracts

- `contracts/admin-course-import.md` — **Cross-repo (api↔admin)**: hợp đồng API đầy đủ của bề mặt admin course
  content, **cả 5 endpoint đều Phase 1** (PO duyệt 2026-07-15): **§1 `POST /_/import`** + **§2 `GET /template`** +
  **§3 `GET /` (list)** + **§4 `GET /:id`** + **§5 `DELETE /:id`**. Ranh giới cross-repo duy nhất của Phase 1
  (learner `web` là api↔web, Phase 2 — xem `docs/roadmap-task-based-learning.md`).
- **Toàn bộ §1–§5 hand-mount** trong `course-content.admin.js` (KHÔNG dùng `adminResource` factory generic — factory
  tự thêm `POST`/`PATCH` cho tạo/sửa cây khóa bỏ qua validate import ⇒ phá bất biến; chỉ mở đọc + xoá). §1 cần
  multer; §2 trả binary; §3 `find()`+phân trang; §4 `findById`; §5 `findByIdAndDelete` (xoá cứng) + audit thủ
  công. Cần thêm hằng `COURSECONTENT_DELETE` vào `permissions.js`.
- Đọc chéo `ipa`/`talk` là **cùng service `api`, cùng repo** → không phải ranh giới cross-service → §3.3 là đủ,
  không cần file contract riêng. (Không đụng `llm`/`cat`.)

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/package.json` | Thêm dependency `exceljs` | Sửa |
| `services/api/src/modules/course-content/course-content.model.js` | Schema `CourseStructure` + cây nhúng + index | Tạo |
| `services/api/src/modules/course-content/course-content.parser.js` | Parse `.xlsx` (exceljs) → `NormalizedCourse` IR + số dòng | Tạo |
| `services/api/src/modules/course-content/course-content.references.js` | Resolver registry (ipa published / talk) validate IS-4 | Tạo |
| `services/api/src/modules/course-content/course-content.service.js` | `importCourse()` — parse→validate→build→ghi; throw `ApiError` | Tạo |
| `services/api/src/middlewares/course-import.upload.js` | multer single `file`, ≤5MB, mime `.xlsx` + map lỗi → `ApiError` (mẫu `upload.js`) | Tạo |
| `services/api/src/admin-api/resources/course-content.admin.js` | Router hand-mount §1 import + §2 template + §3 list + §4 detail + §5 delete (xoá cứng); route cụ thể trước `/:id` | Tạo |
| `services/api/src/admin-api/_router.js` | Đăng ký `router.use('/course-imports', ...)` | Sửa |
| `services/api/src/constants/permissions.js` | Thêm `COURSECONTENT_READ/WRITE/DELETE/MANAGE` (chỉ platform, KHÔNG vào `CENTER_PERMISSIONS`) | Sửa |
| `services/api/src/constants/error-codes.js` | Thêm: `PARSE_FAILED`, `INVALID_FILE_TYPE`, `IMPORT_VALIDATION_FAILED`, `EMPTY_COURSE_FILE`, `UNSUPPORTED_EXERCISE_TYPE` | Sửa |
| `exe-admin/src/app/(admin)/course-import/page.tsx` | Trang Import: upload + summary/lỗi; danh sách khóa (list) + nút xoá + tải template (AC-5, AC-8..AC-11) | Tạo |
| `exe-admin/src/services/course-content.service.ts` | Gọi import + template + list + detail + delete | Tạo |
| `services/api/src/__tests__/course-content.import.test.js` | Test AC-1..AC-4, AC-E1..E9 (§10) | Tạo |
| `services/api/src/__tests__/course-content.admin.test.js` | Test §2–§5 (template/list/detail/delete + quyền) | Tạo |

## 7. Xử lý lỗi

Tất cả lỗi phía client dùng **HTTP 400** (theo quyết định Q4 của PO + convention repo — repo **không dùng 422**);
phân biệt bằng `code` machine-readable + mảng `errors[]`.

| Tình huống | Mã HTTP | error code (top-level) | Map AC |
|---|---|---|---|
| File `.xlsx` hỏng/không đọc được | 400 | `PARSE_FAILED` | AC-E1 |
| Vượt 5MB / sai loại file | 400 | `FILE_TOO_LARGE` / `INVALID_FILE_TYPE` | Q6 |
| File rỗng / không có tầng nào | 400 | `EMPTY_COURSE_FILE` | AC-E2 |
| Lỗi cấu trúc, trùng key nội bộ, trùng khóa DB, ID không tồn tại/chưa publish, type quiz | 400 | `IMPORT_VALIDATION_FAILED` (kèm `errors[]`) | AC-2, AC-3, AC-E3, AC-E5, AC-E6, Q-Quiz |
| Bất kỳ lỗi validate nào ⇒ không ghi | 400 | — | AC-4 / IS-5 |
| Thiếu quyền | 403 | `PERMISSION_DENIED` (có sẵn) | AC-E4 |
| Chưa đăng nhập | 401 | `NOT_AUTHENTICATED` (có sẵn) | — |

`errors[]` mỗi phần tử: `{ code, message, row, refId?, refType? }` — `row` là **số dòng Excel** để đội học thuật
tự sửa (IS-7, vd "Dòng 45: Module 'mod-01' đã tồn tại"). `code` con: `MISSING_LEVEL` / `ORPHAN_NODE` /
`DUPLICATE_KEY` / `ALREADY_EXISTS` / `REFERENCE_NOT_FOUND` / `REFERENCE_NOT_PUBLISHED` / `UNSUPPORTED_EXERCISE_TYPE`
/ `EMPTY_CHILDREN` / `EMPTY_LESSON` / `INVALID_CEFR` / `INVALID_CATEGORY` / `INVALID_URL` (các code con này KHÔNG
cần khai báo trong `error-codes.js` — chỉ top-level `IMPORT_VALIDATION_FAILED` là hằng).

## 8. Bảo mật & quyền

- **Permission mới:** `coursecontent:read/write/delete/manage` (2-segment `<resource>:<action>`, thêm vào
  `src/constants/permissions.js`). Import gate `coursecontent:write`; template/list/detail gate
  `coursecontent:read`; **delete gate `coursecontent:delete`** (chỉ `manage` thoả — bar cao vì phá huỷ; `write` để
  import KHÔNG đủ xoá). **Không** tái dùng `course:*`.
- **Chỉ platform admin/đội học thuật** giữ `coursecontent:*` (Q5). **KHÔNG** thêm vào `CENTER_PERMISSIONS` bundle
  ở Phase 1. `centerId` luôn `null` (nội dung dùng chung). Resource **không** bật `tenantScoped` ở Phase 1.
- **5 rules auth tuyệt đối** (`<API_REPO>/docs/api/CONVENTIONS.md` §5): permission ở backend; không trust body;
  middleware explicit; default DENY; status 401/403 nhất quán.
- **Audit:** mutation qua factory tự phát `admin.command.executed` (`coursecontent.import`) — thỏa NFR Audit.
- **Upload an toàn:** memoryStorage, `files: 1`, `fileSize: 5MB`, fileFilter mime `.xlsx`
  (`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`). Không ghi file thô ra disk (chưa có yêu
  cầu lưu đối soát).
- **Response DTO:** list/detail khóa qua projection factory — không trả raw doc.

## 9. Rủi ro / đánh đổi

- **[Đã giải quyết] All-or-nothing trên standalone MongoDB** → single embedded document. `research.md`.
- **Giới hạn 16MB BSON:** file ≤5MB / ≤~5.000 bài học (Q6) an toàn xa trần; parser vẫn chặn ngưỡng số node và báo
  lỗi rõ thay vì để Mongo ném lỗi ghi.
- **Dependency mới `exceljs`:** thư viện đọc `.xlsx` phổ biến, được bảo trì tốt. Đánh đổi đã chấp nhận vì giá trị
  cốt lõi Epic 4 là để đội non-tech dùng Excel (ghi ADR-0001). Rủi ro: bề mặt tấn công parse file — giảm thiểu
  bằng giới hạn dung lượng + xử lý trong memory + không eval nội dung ô.
- **Re-import để sửa bị chặn** (Q4 = từ chối trùng): sửa một khóa đã import phải xoá thủ công rồi import lại. Chấp
  nhận Phase 1; versioning/edit là nhu cầu sau (spec §3 ngoài phạm vi).
- **[Đã chốt tại Technical Review] Cấp độ khóa phát hiện trùng (Q4) = phương án (a):** chỉ chặn khi **cả khóa
  (`slug`/roadmap ID)** trùng, KHÔNG yêu cầu ID module/lesson duy nhất toàn hệ thống (chỉ duy nhất trong phạm vi
  cha — AC-E3). Đơn giản, khớp câu hỏi "trùng khóa". Chi tiết: `open-questions.md` §Q4.
- **`quiz` hoãn Phase 2** (Q-Quiz): template `.xlsx` Phase 1 chỉ tài liệu hoá `ipa`/`talk`; nếu học thuật lỡ điền
  `quiz` → lỗi rõ ràng, không ngầm bỏ qua.
- **Divergence với idea docs:** những doc đó thiết kế "tạo mới bài tập khi import"; spec Phase 1 cố ý thu hẹp về
  "chỉ tham chiếu ID". Design bám spec.

## 10. Testing strategy

Repo dùng Jest + `mongodb-memory-server` (standalone — khớp topo thật, kiểm chứng nhánh không-transaction) +
`supertest` (`package.json` devDeps, `<API_REPO>/TESTING.md`).

- **Unit (service)** với IR dựng sẵn (bỏ qua parser để test độc lập):
  - file hợp lệ → tạo đúng document, summary counts khớp; assert **đúng 1** document ghi (AC-1).
  - lỗi cấu trúc / trùng key nội bộ → throw, `errors[]` trỏ đúng `row`, **0** document (AC-2, AC-E3, AC-4).
  - exercise ID không tồn tại / IPA chưa publish (mock resolver) → `errors[]` đúng ID+type+row (AC-3, AC-E6), **0**
    document (AC-4).
  - file có `type: quiz` → `UNSUPPORTED_EXERCISE_TYPE` (Q-Quiz).
  - slug đã tồn tại → `ALREADY_EXISTS`, **0** document (AC-E5).
  - file rỗng → `EMPTY_COURSE_FILE` (AC-E2).
- **Unit (parser):** `.xlsx` hỏng → `PARSE_FAILED` (AC-E1); một `.xlsx` mẫu hợp lệ → IR đúng cây + số dòng.
- **Integration (supertest)** `POST /_/import`:
  - user thiếu `coursecontent:write` → 403, không ghi (AC-E4).
  - user đủ quyền + `.xlsx` hợp lệ → 200 + summary; assert audit log phát ra.
- **all-or-nothing (AC-4):** `countDocuments` trước & sau lần import lỗi ⇒ bằng nhau.
- Case cụ thể (Given/When/Then → test name) chi tiết trong `tasks.md` (test-first, tiền lệ
  `<API_REPO>/docs/superpowers/plans/`).

## 11. Rollout / cross-repo sequencing

2 repo: `exe-api` (provider) + `exe-admin` (consumer). Theo `.ai/workflows/release-workflow.md` §3: **deploy
`exe-api` trước** (endpoint + permission + `exceljs`), rồi `exe-admin` (trang Import). **Seed permission
`coursecontent:*`** cho tài khoản đội học thuật/platform admin trước khi trang Import dùng (nếu không → 403).
Contract mới hoàn toàn, không breaking change.

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion | Được đáp ứng bởi |
|---|---|
| **AC-1** Import file hợp lệ tạo đủ cấu trúc + summary counts | §3.1 model; §4 bước 6–8; §3.2 parser; §10 unit AC-1 |
| **AC-2** Từ chối file sai cấu trúc, lỗi chỉ rõ dòng, không ghi | §3.2 IR có số dòng; §4 bước 2+5; §7 `IMPORT_VALIDATION_FAILED`; §10 |
| **AC-3** Từ chối exercise ID không tồn tại, liệt kê đúng ID | §3.3 resolver; §4 bước 4+5; §7 `errors[]` refId/refType/row; §10 |
| **AC-4** All-or-nothing | §2 validate-rồi-mới-ghi + single embedded doc; §4 bước 5; `research.md`; §10 đếm doc |
| **AC-5** Hiển thị kết quả trên exe-admin | §6 `course-import/page.tsx` + service; `contracts/admin-course-import.md` |
| **AC-6** Lưu CEFR (Chặng) + category (Chuyên đề) — IS-9 | §3.1 field mới; §3.2 cột F/G/H/L; validate `INVALID_CEFR`/`INVALID_CATEGORY` |
| **AC-7** Lưu theory/video/audio + Bài học lý thuyết-thuần — IS-10 | §3.1 field mới; §3.2 cột I/J/K; ràng buộc `EMPTY_LESSON` (không bắt ≥1 exercise) |
| **AC-E1** File không parse được → lỗi rõ, không 500 | §3.2 `ParseError`; §7 `PARSE_FAILED`; §10 unit parser |
| **AC-E2** File rỗng / không tầng nào | §7 `EMPTY_COURSE_FILE`; §4 bước 2; §10 |
| **AC-E3** Trùng key nội bộ trong file | §3.2/§4 bước 2 `DUPLICATE_KEY`; §7; §10 |
| **AC-E4** User thiếu quyền → từ chối, không ghi | §8 `verifyPermission('coursecontent:write')`; §4 chain; §10 integration 403 |
| **AC-E5** Import trùng khóa đã tồn tại (Q4) | §3.1 `slug` unique; §4 bước 3 + safety net bước 7; §7 `ALREADY_EXISTS`; §10 |
| **AC-E6** Tham chiếu bài tập draft/archived (Q3) | §3.3 resolver IPA lọc `status:'published'`; §7 `REFERENCE_NOT_PUBLISHED`; §10 |
| **AC-E7** Tham chiếu type `quiz` (hoãn Phase 2) | §3.3 `UNSUPPORTED_EXERCISE_TYPE`; enum model `[ipa,talk]` |
| **AC-E8** Bài học rỗng hoàn toàn (IS-10) | §3.1 ràng buộc `EMPTY_LESSON` (≥1 trong theory/video/audio/exercise) |
| **AC-E9** Metadata/URL sai (IS-9/IS-10) | validate `INVALID_CEFR`/`INVALID_CATEGORY`/`INVALID_URL` (§7) |
| **NFR Bảo mật** | §8 permission mới, platform-only, `centerId:null`, tenant từ server |
| **NFR Audit** | §8 + §4 audit — factory tự phát `admin.command.executed` |
| **NFR Hiệu năng** (Q6) | §4 đồng bộ; §8 giới hạn 5MB; §9 ngưỡng số node |
