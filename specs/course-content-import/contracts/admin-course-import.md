# Contract: Admin — Course Content API (import + quản lý khóa)

- **Loại:** Cross-repo (api↔admin)
- **Bên cung cấp (provider):** `api` (`exe-api`, `src/admin-api/`)
- **Bên tiêu thụ (consumer):** `admin` (`exe-admin` — trang Import + màn quản lý khóa đã import)
- **Trạng thái:** Đã thống nhất (PO chốt Q1–Q6 + Q-Quiz + Hướng B). **PO đã duyệt nâng §2–§5 (template/list/
  detail/delete) vào Phase 1 (2026-07-15)** — toàn bộ 5 endpoint đều thuộc Phase 1.

## 0. Tổng quan

### 0.1 Quy ước chung

- **Base path:** `/api/admin/course-imports` (thuộc admin surface, mount trong `src/admin-api/_router.js`).
- **Auth:** JWT user (Bearer) cho mọi endpoint — surface admin dùng JWT client, **KHÔNG** `X-Service-Key`.
- **Envelope:** thành công `{ "data": ... }`; lỗi `{ "message": string, "code": string, "errors"?: [...] }`
  (theo `errorHandler`). **Mọi lỗi phía client dùng HTTP 400** (repo không dùng 422); phân biệt bằng `code`.
- **Tenant (Q5):** Phase 1 nội dung dùng chung `centerId = null`; **không** nhận `centerId` từ client. Resource
  **không** bật `tenantScoped` ở Phase 1.
- **Audit:** mọi mutation (import, delete) phát `admin.command.executed` qua audit hook — không lộ ra client.

### 0.2 Bảng tổng quan endpoint

| # | Method | Path | Permission | Phase | Mục đích |
|---|---|---|---|---|---|
| §1 | `POST` | `/course-imports/_/import` | `coursecontent:write` | **1** | Import 1 file `.xlsx` → tạo khóa |
| §2 | `GET` | `/course-imports/template` | `coursecontent:read` | **1** | Tải template `.xlsx` chuẩn |
| §3 | `GET` | `/course-imports` | `coursecontent:read` | **1** | Liệt kê khóa đã import (phân trang) |
| §4 | `GET` | `/course-imports/:id` | `coursecontent:read` | **1** | Chi tiết 1 khóa (cả cây) |
| §5 | `DELETE` | `/course-imports/:id` | `coursecontent:delete` | **1** | Xoá 1 khóa (để sửa & import lại) |

> **Thứ tự đăng ký route (quan trọng):** `/_/import` và `/template` phải mount **trước** route `/:id` của factory,
> nếu không `:id` sẽ "nuốt" `template` (Express match `/template` thành `:id='template'`).

### 0.3 Vì sao §2–§5 thuộc Phase 1 (PO đã duyệt 2026-07-15)

Bản design gốc hoãn list/read sang Phase 2; rà lại plan cho thấy 4 endpoint này **cần cho Phase 1 dùng được thực
tế**, và chi phí thấp (list/get/delete là truy vấn Mongoose đơn giản; template là 1 GET nhỏ):

> **Cài đặt (đã cân nhắc):** §3/§4/§5 **hand-mount read/delete** trong `course-content.admin.js`, **KHÔNG** dùng
> `adminResource` factory generic — vì factory tự thêm `POST`/`PATCH` cho phép tạo/sửa cây khóa **bỏ qua validate
> import** (phá bất biến all-or-nothing). Chỉ mở đúng đọc + xoá; tạo khóa duy nhất qua §1 import.

- **Template (§2):** đội học thuật non-tech cần file mẫu đúng cột để điền — thiếu nó thì tính năng "để non-tech tự
  soạn" (giá trị cốt lõi Epic 4) không trọn.
- **List/Get (§3/§4):** sau khi import, admin cần **thấy** đã có khóa gì, và cần `id` để xoá — nếu không, admin
  "mù" sau khi rời trang Import.
- **Delete (§5):** vì Q4 = **từ chối trùng**, muốn sửa một khóa đã import buộc phải xoá rồi import lại. Không có
  endpoint xoá ⇒ phải can thiệp DB thủ công (đã nêu là điểm yếu ở `design.md` §9). §5 đóng lỗ hổng này.

→ **PO đã duyệt** đưa cả 5 endpoint vào Phase 1.

---

## §1. `POST /course-imports/_/import` — Import khóa *(Phase 1 — đã chốt)*

```
POST /api/admin/course-imports/_/import
Content-Type: multipart/form-data
```

- **Permission:** `coursecontent:write` (`coursecontent:manage` cũng qua — `manage` implies write).
- **Cài đặt:** hand-mount (KHÔNG qua factory action) vì cần multer.
  Chain: `verifyToken → verifyPermission → courseImportUpload(multer) → asyncHandler(handler)`. Handler **không**
  business logic — delegate `course-content.service.importCourse()` (HARD RULE admin-api) + `auditLog` thủ công.

### Request

`multipart/form-data`, đúng **một** file field:

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `file` | file `.xlsx` (binary) | ✓ | Excel theo template §2. Mime `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. Giới hạn **5MB** (Q6). |

Cấu trúc file: cột `Level` dẫn dắt (ROADMAP/PHASE/MODULE/LESSON/EXERCISE). **Cột (Hướng B):**
`A Level | B Key | C Title | D Type | E RefId | F CefrFrom | G CefrTo | H Category | I Content(desc/theory) |
J VideoUrl | K AudioUrl | L Note` (chi tiết ý nghĩa: `design.md` §3.2). Dòng `EXERCISE` mang **ID tham chiếu** +
`type ∈ {ipa, talk}` (`quiz` chưa hỗ trợ); Bài học có thể chỉ có lý thuyết/video/audio.

### Response — 200 thành công

```json
{
  "data": {
    "summary": {
      "id": "665f0a...",
      "slug": "tieng-anh-san-bay-a2",
      "title": "Tiếng Anh Sân Bay",
      "status": "ready",
      "counts": { "phases": 3, "modules": 8, "lessons": 24, "exercises": 40 }
    }
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `data.summary.id` | String | `_id` khóa vừa tạo (dùng cho §4/§5) |
| `data.summary.slug` | String | Định danh khóa |
| `data.summary.status` | String | `"ready"` (IS-6) |
| `data.summary.counts` | Object | Số Chặng/Chuyên đề/Bài học/bài tập — **khớp file** (AC-1) |

FE hiển thị `counts` như tóm tắt thành công (AC-5).

### Response — lỗi

| Mã | error code | Khi nào | AC |
|---|---|---|---|
| 400 | `PARSE_FAILED` | `.xlsx` hỏng/không đọc được | AC-E1 |
| 400 | `INVALID_FILE_TYPE` | Không phải `.xlsx` | Q6 |
| 400 | `FILE_TOO_LARGE` | Vượt 5MB | Q6 |
| 400 | `EMPTY_COURSE_FILE` | Parse được nhưng không có tầng nào | AC-E2 |
| 400 | `IMPORT_VALIDATION_FAILED` | Lỗi cấu trúc/tham chiếu/trùng — kèm `errors[]` (§6) | AC-2/3, AC-E3/E5/E6/E8/E9, Q-Quiz |
| 401 | `NOT_AUTHENTICATED` | Thiếu/sai token | — |
| 403 | `PERMISSION_DENIED` | Thiếu `coursecontent:write` | AC-E4 |

**Bất biến all-or-nothing (AC-4/IS-5):** với mọi phản hồi lỗi, **KHÔNG** tạo bản ghi nào.

---

## §2. `GET /course-imports/template` — Tải template Excel *(Phase 1)*

```
GET /api/admin/course-imports/template
```

- **Permission:** `coursecontent:read`.
- **Cài đặt:** hand-mount, trả file `.xlsx` sinh sẵn (server dựng bằng `exceljs` từ định nghĩa cột, hoặc đọc 1
  file mẫu tĩnh trong repo). Gồm: **dòng header 12 cột** + **vài dòng ví dụ** (1 ROADMAP→PHASE→MODULE→LESSON→
  EXERCISE mẫu) + (tuỳ chọn) 1 sheet "Hướng dẫn" liệt kê enum hợp lệ (CEFR, category, type).

### Response — 200

- **Headers:** `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`,
  `Content-Disposition: attachment; filename="course-content-template.xlsx"`.
- **Body:** binary `.xlsx`.

### Response — lỗi

| Mã | error code | Khi nào |
|---|---|---|
| 401 | `NOT_AUTHENTICATED` | Thiếu/sai token |
| 403 | `PERMISSION_DENIED` | Thiếu `coursecontent:read` |

---

## §3. `GET /course-imports` — Liệt kê khóa đã import *(Phase 1)*

```
GET /api/admin/course-imports?page=1&limit=20&q=<từ khóa>&sort=-createdAt
```

- **Permission:** `coursecontent:read`.
- **Cài đặt:** hand-mount — `CourseStructure.find()` với projection + skip/limit + `countDocuments`.

### Query params

| Param | Kiểu | Mặc định | Ghi chú |
|---|---|---|---|
| `page` | Number | 1 | Trang |
| `limit` | Number | 20 | ≤ 100 (giới hạn factory) |
| `q` | String | — | Tìm theo `slug`/`title` (regex, factory `searchFields`) |
| `sort` | String | `-createdAt` | vd `title`, `-createdAt` |

### Response — 200

```json
{
  "data": {
    "items": [
      { "id": "665f0a...", "slug": "tieng-anh-san-bay-a2", "title": "Tiếng Anh Sân Bay",
        "status": "ready", "createdAt": "2026-07-15T10:00:00.000Z" }
    ],
    "total": 1, "page": 1, "limit": 20
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `items[]` | Array | Projection `listFields` = `slug, title, status, createdAt` (KHÔNG trả cả cây `phases` ở list) |
| `total` / `page` / `limit` | Number | Phân trang |

> `counts` KHÔNG có trong list (tránh tính toán mỗi hàng); lấy ở §4 hoặc từ `summary` lúc import.

### Response — lỗi: 401 `NOT_AUTHENTICATED` · 403 `PERMISSION_DENIED`.

---

## §4. `GET /course-imports/:id` — Chi tiết 1 khóa *(Phase 1)*

```
GET /api/admin/course-imports/:id
```

- **Permission:** `coursecontent:read`.
- **Cài đặt:** hand-mount — `CourseStructure.findById().lean()`.

### Response — 200

Trả **cả cây** (document là 1 khối):

```json
{
  "data": {
    "id": "665f0a...",
    "slug": "tieng-anh-san-bay-a2",
    "title": "Tiếng Anh Sân Bay",
    "description": "...",
    "status": "ready",
    "centerId": null,
    "phases": [
      { "key": "p1", "title": "Chặng 1", "order": 1, "cefrFrom": "A2", "cefrTo": "B1", "goalNote": "IELTS 5.0→6.0",
        "modules": [
          { "key": "m1", "title": "Ngữ pháp", "order": 1, "category": "grammar",
            "lessons": [
              { "key": "l1", "title": "Bài 1", "order": 1, "theory": "# ...",
                "videoUrl": "https://...", "audioUrl": "https://...",
                "exercises": [ { "type": "ipa", "refId": "L1", "order": 1 } ] }
            ] }
        ] }
    ],
    "sourceMeta": { "filename": "course-san-bay.xlsx", "nodeCount": 76, "format": "xlsx" },
    "importedBy": "665e...",
    "createdAt": "2026-07-15T10:00:00.000Z"
  }
}
```

### Response — lỗi: 401 · 403 · **404 `NOT_FOUND`** (id không tồn tại/sai định dạng).

---

## §5. `DELETE /course-imports/:id` — Xoá 1 khóa *(Phase 1)*

```
DELETE /api/admin/course-imports/:id
```

- **Permission:** `coursecontent:delete` (chỉ `coursecontent:manage` thoả — bar cao vì thao tác phá huỷ; quyền
  `write` để import KHÔNG đủ để xoá).
- **Cài đặt:** hand-mount — `CourseStructure.findByIdAndDelete()` (**xoá cứng**) + `auditLog` thủ công.

> **Vì sao xoá cứng (không soft-delete):** để `slug` được giải phóng, cho phép **import lại** khóa đã sửa (Q4 từ
> chối trùng dựa trên `slug` unique). Soft-delete sẽ giữ `slug` trong DB ⇒ import lại vẫn `ALREADY_EXISTS`. Phase
> 1 chưa có consumer (learner Phase 2) nên xoá cứng an toàn. **Đánh đổi:** Phase 2 (khi có consumer) nên cân nhắc
> chuyển sang soft-delete + versioning + loại `slug` archived khỏi kiểm tra trùng.

### Response — 200

```json
{ "data": { "id": "665f0a...", "deleted": true } }
```

### Response — lỗi: 401 · 403 (thiếu `coursecontent:delete`/`manage`) · **404 `NOT_FOUND`**.

Audit: phát `admin.command.executed` (`coursecontent.delete`, target = id) qua factory hook.

---

## §6. Schema `errors[]` (dùng chung cho §1)

Mỗi phần tử (FE render danh sách lỗi cụ thể — AC-5/IS-7):

```json
{ "code": "REFERENCE_NOT_PUBLISHED", "message": "Dòng 45: bài tập IPA 'L99' chưa được publish",
  "row": 45, "refType": "ipa", "refId": "L99" }
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `code` | String | `MISSING_LEVEL` / `ORPHAN_NODE` / `DUPLICATE_KEY` / `ALREADY_EXISTS` / `REFERENCE_NOT_FOUND` / `REFERENCE_NOT_PUBLISHED` / `UNSUPPORTED_EXERCISE_TYPE` / `EMPTY_CHILDREN` / `EMPTY_LESSON` / `INVALID_CEFR` / `INVALID_CATEGORY` / `INVALID_URL` |
| `message` | String | Mô tả tiếng Việt, kèm số dòng |
| `row` | Number | Số dòng Excel để đội học thuật tự sửa |
| `refType` / `refId` | String? | Chỉ có với lỗi tham chiếu (AC-3/AC-E6) |

> Các `code` con này KHÔNG khai báo trong `error-codes.js` — chỉ top-level `IMPORT_VALIDATION_FAILED` là hằng.

## §7. Permission & scope

| Permission | Endpoint | Ghi chú |
|---|---|---|
| `coursecontent:read` | §2, §3, §4 | Xem template/list/detail |
| `coursecontent:write` | §1 | Import |
| `coursecontent:delete` | §5 | Xoá (chỉ ai có `:delete` hoặc `:manage`) |
| `coursecontent:manage` | tất cả | Implies read/write/delete (`hasPermission`) |

- Thêm cả 4 hằng `COURSECONTENT_READ/WRITE/DELETE/MANAGE` vào `src/constants/permissions.js` (nguồn sự thật duy
  nhất) — **[cập nhật so với design cũ: bổ sung `COURSECONTENT_DELETE`]**. Phase 1 chỉ cấp cho platform admin/đội
  học thuật, **KHÔNG** vào `CENTER_PERMISSIONS` (Q5).
- Tenant từ server (`centerId=null` Phase 1), không tin `req.body` (5 rules auth — `docs/api/CONVENTIONS.md` §5).

## §8. Versioning / breaking change

N/A — contract mới hoàn toàn (chưa có consumer live). Khi Technical Review chốt scope §2–§5, cập nhật cột "Phase"
ở §0.2 và nâng Trạng thái lên "Đã thống nhất 2 bên".

## §9. Sequencing triển khai (cross-repo)

`.ai/workflows/release-workflow.md` §3 — **provider trước consumer**:

1. Deploy `exe-api`: `exceljs` + model + service + admin resource (import hand-mount + factory list/get/delete +
   template) + permission `coursecontent:*` (gồm `:delete`), và **seed permission** cho tài khoản đội học thuật/
   platform admin (thiếu → 403).
2. Deploy `exe-admin`: trang Import + màn quản lý khóa (list/detail/delete) + `course-content.service.ts`.

## §10. Ngoài phạm vi contract này

- **API learner tiêu thụ khóa (đọc lộ trình, render bài học, unlock)** — thuộc **Phase 2**, ranh giới **api↔web**
  (repo `exe-web`), sẽ có contract riêng `contracts/web-course-consume.md` khi tới Phase 2. KHÔNG thuộc api↔admin.
- **Sửa (edit) khóa đã import qua UI, versioning** — Phase 1 chỉ có import + xoá-rồi-import-lại (Q4). Edit/version
  là nhu cầu sau.
- **Upload/host media (video/audio) tập trung** — Hướng B chỉ lưu URL; host media để phase sau.

## §11. Ví dụ gọi thực tế

```bash
# Import
curl -X POST "$API/api/admin/course-imports/_/import" \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -F "file=@course-san-bay.xlsx;type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
# → { "data": { "summary": { "id": "665f...", "status": "ready", "counts": { "phases": 3, ... } } } }

# Tải template
curl -X GET "$API/api/admin/course-imports/template" -H "Authorization: Bearer $ADMIN_JWT" -o template.xlsx

# List
curl -X GET "$API/api/admin/course-imports?page=1&limit=20" -H "Authorization: Bearer $ADMIN_JWT"

# Detail
curl -X GET "$API/api/admin/course-imports/665f0a..." -H "Authorization: Bearer $ADMIN_JWT"

# Delete (để import lại bản sửa)
curl -X DELETE "$API/api/admin/course-imports/665f0a..." -H "Authorization: Bearer $ADMIN_JWT"
# → { "data": { "id": "665f0a...", "deleted": true } }

# Lỗi validate import (400):
# { "message": "Import validation failed", "code": "IMPORT_VALIDATION_FAILED",
#   "errors": [ { "code": "ALREADY_EXISTS", "row": 2, "message": "Dòng 2: khóa 'tieng-anh-san-bay-a2' đã tồn tại" } ] }
```
