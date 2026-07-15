# Data model: Import cấu trúc Khóa học tĩnh (Phase 1)

- **Spec:** `specs/course-content-import/spec.md`
- **Design:** `specs/course-content-import/design.md`
- **Ngày:** 2026-07-15 (đã áp Q1–Q6 + Q-Quiz; **cập nhật mở rộng nội dung — Hướng B**, PO chốt 2026-07-15)

> Model lưu **1 khóa học = 1 document nhúng cả cây** trong collection `course_structures` (nguyên tử IS-5/AC-4:
> `research.md`). **Mở rộng Hướng B:** mỗi tầng mang thêm ngữ nghĩa nâng-cấp-trình-độ (CEFR ở Chặng, phân loại ở
> Chuyên đề) và nội dung học đa phương thức (lý thuyết + video + audio ở Bài học — dạng **text + link URL**, KHÔNG
> upload media trong luồng import). Định danh code tiếng Anh; mô tả tiếng Việt (`.ai/project-context.md` §8).

## 1. Collection `course_structures` — document gốc `CourseStructure` (Lộ trình)

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `slug` | String (kebab-case) | ✓ | | **unique index**; khóa tự nhiên phát hiện trùng (Q4/AC-E5) |
| `title` | String (trim) | ✓ | | Tên Lộ trình |
| `description` | String | | `''` | Giới thiệu khóa |
| `centerId` | ObjectId → `Center` | | `null` | **Luôn `null` Phase 1** (Q5 — dùng chung) |
| `status` | String enum `['ready']` | ✓ | `'ready'` | IS-6 "sẵn sàng Phase 2" |
| `phases` | `[PhaseSchema]` | ✓ (≥1) | | Chặng — nhúng |
| `importedBy` | ObjectId → `User` | ✓ | | Người import |
| `sourceMeta` | `SourceMetaSchema` | | `null` | Metadata file `.xlsx` gốc |
| `createdAt`/`updatedAt` | Date | auto | | |

Index: `{ slug: 1 }` unique; `{ status: 1 }`.

## 2. Sub-document (nhúng) — cây phân tầng

### `PhaseSchema` (Chặng — một nấc nâng cấp trình độ) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Duy nhất trong khóa (AC-E3) |
| `title` | String | ✓ | Tên Chặng |
| `order` | Number | ✓ | Thứ tự |
| `cefrFrom` | String enum CEFR | | **[MỚI]** trình độ bắt đầu, `['A1','A2','B1','B2','C1','C2']`. Optional (Chặng có thể dùng band thay CEFR) |
| `cefrTo` | String enum CEFR | | **[MỚI]** trình độ đích. Nếu cả `cefrFrom` & `cefrTo` có → validate `cefrTo` ≥ `cefrFrom` |
| `goalNote` | String | | **[MỚI]** mục tiêu dạng tự do, vd `"IELTS 5.0 → 6.0"` |
| `modules` | `[ModuleSchema]` | ✓ (≥1) | Chuyên đề con |

> Ngữ nghĩa "từ A2 lên B1": điền `cefrFrom=A2`, `cefrTo=B1`. IELTS: điền `goalNote`. Không bắt buộc để linh hoạt,
> nhưng validate enum + thứ tự nếu có (AC-2 mở rộng).

### `ModuleSchema` (Chuyên đề — theo mảng kỹ năng) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Duy nhất trong Chặng cha (AC-E3) |
| `title` | String | ✓ | |
| `order` | Number | ✓ | |
| `category` | String enum | | **[MỚI]** `['grammar','pronunciation','vocabulary','listening','reading','writing','speaking','other']`; default `'other'`. Ngữ nghĩa "ngữ pháp / phát âm / từ vựng…" |
| `lessons` | `[LessonSchema]` | ✓ (≥1) | Bài học con |

### `LessonSchema` (Bài học — học đa phương thức) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Duy nhất trong Chuyên đề cha (AC-E3) |
| `title` | String | ✓ | |
| `order` | Number | ✓ | |
| `theory` | String (markdown) | | **[MỚI]** nội dung lý thuyết. Text thuần/markdown, không HTML thô |
| `videoUrl` | String (URL) | | **[MỚI]** link video (YouTube/CDN/R2) — content team tự host; validate dạng `http(s)://` |
| `audioUrl` | String (URL) | | **[MỚI]** link audio — tương tự |
| `exercises` | `[ExerciseRefSchema]` | | Tham chiếu bài tập; **KHÔNG còn bắt buộc ≥1** (cho phép Bài học chỉ lý thuyết/video) |

> **Ràng buộc "Bài học không rỗng" (thay quy ước cũ):** một Bài học phải có **ít nhất một** trong
> {`theory`, `videoUrl`, `audioUrl`, ≥1 `exercises`}. Rỗng hoàn toàn ⇒ lỗi `EMPTY_LESSON` (AC-E2 mở rộng). Nhờ vậy
> Bài học lý thuyết-thuần (không bài tập) là hợp lệ — đúng tầm nhìn "học đủ lý thuyết/video/âm thanh".

### `ExerciseRefSchema` (Tham chiếu bài tập) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | String enum **`['ipa','talk']`** | ✓ | Phase 1 chỉ `ipa`+`talk` (Q-Quiz: `quiz` hoãn; Nghe/Đọc/Viết/Ngữ pháp/Từ vựng cần engine mới → phase sau) |
| `refId` | String | ✓ | IPA = `ipa_lessons.code` (phải `published` — Q3); talk = phần tử `SCENARIO_IDS` |
| `order` | Number | ✓ | |
| `label` | String | | Nhãn hiển thị tuỳ chọn |

**Tham chiếu "mềm", CỐ Ý không phải Mongoose `ref`** (Talk không có collection; IPA phân giải theo `code`) —
validate ở thời điểm import.

### `SourceMetaSchema` — `{ _id: false }`

| Field | Kiểu | Ghi chú |
|---|---|---|
| `filename` | String | Tên file `.xlsx` |
| `contentHash` | String | Hash nội dung |
| `nodeCount` | Number | Tổng node — ngưỡng an toàn (≤ ~5.000 bài học, Q6) |
| `format` | String | `'xlsx'` |

## 3. Ràng buộc toàn vẹn (validate trong service, trước khi ghi — gom lỗi, kèm số dòng Excel)

1. **Cha–con hợp lệ (AC-2):** IR giữ cây → node "mồ côi" (`ORPHAN_NODE`).
2. **Không trùng `key` trong phạm vi cha (AC-E3):** `DUPLICATE_KEY`.
3. **Không rỗng:** khóa ≥1 Chặng, Chặng ≥1 Chuyên đề, Chuyên đề ≥1 Bài học (`EMPTY_CHILDREN`); Bài học phải có
   ≥1 nội dung/bài tập (`EMPTY_LESSON`).
4. **[MỚI] CEFR hợp lệ:** `cefrFrom`/`cefrTo` (nếu có) thuộc enum CEFR; nếu cả hai có thì `cefrTo` ≥ `cefrFrom`
   (`INVALID_CEFR`).
5. **[MỚI] Category hợp lệ:** `category` (nếu có) thuộc enum (`INVALID_CATEGORY`).
6. **[MỚI] URL hợp lệ:** `videoUrl`/`audioUrl` (nếu có) khớp `^https?://` (`INVALID_URL`).
7. **Type bài tập:** chỉ `ipa`/`talk`; `quiz`/khác ⇒ `UNSUPPORTED_EXERCISE_TYPE`.
8. **Exercise ID tồn tại & published (AC-3/AC-E6):** resolver registry.
9. **`slug` chưa tồn tại (AC-E5/Q4):** query + unique index safety net (`ALREADY_EXISTS`).

> Các mã lỗi con (`INVALID_CEFR`, `INVALID_CATEGORY`, `INVALID_URL`, `EMPTY_LESSON`, …) là **code trong từng phần
> tử `errors[]`**, không phải hằng trong `error-codes.js` (top-level vẫn là `IMPORT_VALIDATION_FAILED`).

## 4. Ước lượng kích thước

`theory` là text (có thể vài KB/bài); video/audio chỉ là **URL** (không nhúng media) nên không phình document.
Khóa ≤~5.000 bài học vẫn xa trần 16MB. Parser đặt ngưỡng số node báo lỗi rõ (`research.md`).

## 5. Migration / backfill

- **Collection mới, không migration.** Chỉ đọc chéo `ipa_lessons` để validate.
- **Seed permission** `coursecontent:read/write/manage` (platform-only, KHÔNG vào `CENTER_PERMISSIONS`).
- **Dependency mới** `exceljs`.
- **Media (video/audio) do content team tự host** ngoài hệ thống (CDN/YouTube/R2) — Phase 1 chỉ lưu URL, KHÔNG
  có luồng upload/serve media cho khóa học. (Nếu sau này cần host tập trung → dùng module `media`, phase sau.)
