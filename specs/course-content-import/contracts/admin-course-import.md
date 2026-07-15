# Contract: Admin — Import cấu trúc khóa học

- **Loại:** Cross-repo (api↔admin)
- **Bên cung cấp (provider):** `api` (`exe-api`, `src/admin-api/`)
- **Bên tiêu thụ (consumer):** `admin` (`exe-admin`, trang Import)
- **Trạng thái:** Đã thống nhất định dạng (PO chốt Q1–Q6 + Q-Quiz) — chờ Technical Review

> Endpoint là **collection-level action** của admin-api factory (`_crud.factory.js`, `collection: true` → mount
> `/_/import`). Chain: `verifyToken → verifyPermission → courseImportUpload(multer) → asyncHandler(handler)`.
> Handler KHÔNG business logic — delegate `course-content.service.importCourse()` (HARD RULE admin-api).

## Endpoint

```
POST /api/admin/course-imports/_/import
Content-Type: multipart/form-data
```

- **Auth:** JWT user (Bearer) — surface admin dùng JWT client, KHÔNG X-Service-Key.
- **Permission required:** `coursecontent:write` (`coursecontent:manage` cũng qua — `manage` implies write).

## Request

`multipart/form-data`, đúng **một** file field:

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `file` | file `.xlsx` (binary) | ✓ | File Excel cấu trúc khóa học theo template chuẩn. Mime: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. Giới hạn **5MB** (Q6). |

- **KHÔNG** nhận `centerId` từ form — Phase 1 nội dung dùng chung, `centerId = null` phía server (Q5).
- Cấu trúc file Excel theo `<API_REPO>/docs/content_ingestion_schema.md` (cột `Level` dẫn dắt:
  ROADMAP/PHASE/MODULE/LESSON/EXERCISE). **Khác biệt Phase 1:** dòng `EXERCISE` mang **ID/mã bài tập tham chiếu**
  (`ipa` = mã IPA lesson đã publish; `talk` = scenario id) + `type ∈ {ipa, talk}`. `quiz` **chưa hỗ trợ** (Phase
  2). Template chuẩn `.xlsx` chốt với đội học thuật ở giai đoạn tasks.

## Response — thành công

`200 OK`

```json
{
  "data": {
    "summary": {
      "id": "665f...",
      "slug": "tieng-anh-san-bay-a2",
      "title": "Tiếng Anh Sân Bay",
      "status": "ready",
      "counts": { "phases": 1, "modules": 1, "lessons": 1, "exercises": 2 }
    }
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `data.summary.id` | String | `_id` khóa vừa tạo |
| `data.summary.slug` | String | Định danh khóa |
| `data.summary.status` | String | `"ready"` (IS-6) |
| `data.summary.counts` | Object | Số Chặng/Chuyên đề/Bài học (+bài tập) — **khớp nội dung file** (AC-1) |

FE hiển thị `counts` như tóm tắt thành công (AC-5).

## Response — lỗi

Body chung theo `errorHandler`: `{ "message": string, "code": string, "errors"?: [...] }`. **Mọi lỗi phía client
dùng HTTP 400** (quyết định PO + convention repo không dùng 422); phân biệt bằng `code`.

| Mã HTTP | error code | Khi nào | AC |
|---|---|---|---|
| 400 | `PARSE_FAILED` | File `.xlsx` hỏng/không đọc được | AC-E1 |
| 400 | `FILE_TOO_LARGE` | Vượt 5MB (multer) | Q6 |
| 400 | `INVALID_FILE_TYPE` | Không phải `.xlsx` | Q6 |
| 400 | `EMPTY_COURSE_FILE` | Parse được nhưng không có tầng nào | AC-E2 |
| 400 | `IMPORT_VALIDATION_FAILED` | Lỗi cấu trúc / trùng key nội bộ / trùng khóa DB / ID không tồn tại / ID chưa publish / type `quiz` — kèm `errors[]` | AC-2, AC-3, AC-E3, AC-E5, AC-E6, Q-Quiz |
| 401 | `NOT_AUTHENTICATED` | Thiếu/sai token | — |
| 403 | `PERMISSION_DENIED` | Thiếu `coursecontent:write` | AC-E4 |

`errors[]` (FE render danh sách lỗi cụ thể — AC-5/IS-7), mỗi phần tử:

```json
{
  "code": "REFERENCE_NOT_PUBLISHED",
  "message": "Dòng 45: bài tập IPA 'L99' chưa được publish",
  "row": 45,
  "refType": "ipa",
  "refId": "L99"
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `code` | String | `MISSING_LEVEL` / `ORPHAN_NODE` / `DUPLICATE_KEY` / `ALREADY_EXISTS` / `REFERENCE_NOT_FOUND` / `REFERENCE_NOT_PUBLISHED` / `UNSUPPORTED_EXERCISE_TYPE` / `EMPTY_CHILDREN` |
| `message` | String | Mô tả tiếng Việt, kèm số dòng |
| `row` | Number | Số dòng Excel để đội học thuật tự sửa |
| `refType` / `refId` | String | Chỉ có với lỗi tham chiếu (AC-3/AC-E6) |

**Bất biến all-or-nothing (AC-4/IS-5):** với mọi phản hồi lỗi, hệ thống **KHÔNG tạo bản ghi nào** —
`course_structures` sau lần import lỗi giống hệt trước đó.

## Versioning / breaking change

N/A — contract mới hoàn toàn (chưa có consumer live).

## Sequencing triển khai (cross-repo)

`.ai/workflows/release-workflow.md` §3 — **provider trước consumer**:

1. Deploy `exe-api`: dependency `exceljs` + model + service + admin resource + permission `coursecontent:*`, và
   **seed permission** cho tài khoản đội học thuật/platform admin (nếu không → 403).
2. Deploy `exe-admin`: trang Import + `course-content.service.ts`.

## Ví dụ gọi thực tế

```bash
curl -X POST "$API/api/admin/course-imports/_/import" \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -F "file=@course-san-bay.xlsx;type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"

# Thành công:
# { "data": { "summary": { "slug": "...", "status": "ready", "counts": { "phases": 1, ... } } } }

# Lỗi validate (400):
# { "message": "Import validation failed", "code": "IMPORT_VALIDATION_FAILED",
#   "errors": [ { "code": "ALREADY_EXISTS", "row": 45, "message": "Dòng 45: khóa 'mod-01' đã tồn tại" } ] }

# Thiếu quyền (403):
# { "message": "Permission denied", "code": "PERMISSION_DENIED" }
```
