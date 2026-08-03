# Data model: Import cấu trúc Khóa học tĩnh (Phase 1)

- **Spec:** `specs/course-content-import/spec.md`
- **Design:** `specs/course-content-import/design.md`
- **Feature-spec (BR gốc):** `specs/course-content-import/feature-spec-course-structure.md`
- **Ngày:** 2026-07-15 (đã áp Q1–Q6 + Q-Quiz; **Hướng B**; **cập nhật 2026-07-16 — bao quát đủ BR feature-spec:**
  `thumbnail`, `status` đủ vòng đời, media dạng **danh sách** `media[]`, chặn Base64, Structure-Lock guard)

> Model lưu **1 khóa học = 1 document nhúng cả cây** trong collection `course_structures` (nguyên tử IS-5/AC-4:
> `research.md`). **Mở rộng Hướng B + bao quát BR:** mỗi tầng mang thêm ngữ nghĩa nâng-cấp-trình-độ (CEFR ở Chặng,
> phân loại ở Chuyên đề) và nội dung học đa phương thức ở Bài học — **lý thuyết** (markdown) + **danh sách media**
> (`media[]`, mỗi phần tử là 1 link video/audio) + tham chiếu bài tập. Media chỉ lưu **URL** (đội học thuật tự host;
> **cấm Base64/`data:` — feature-spec §5.1**), KHÔNG upload trong luồng import. Định danh code tiếng Anh; mô tả tiếng
> Việt (`.ai/project-context.md` §8).
>
> **Ánh xạ tên trường (feature-spec → model, giữ định danh code tiếng Anh):** `content_theory` → `theory`;
> `media_urls` → `media[]`; `exercise_refs` → `exercises[]`; `thumbnail/cover` → `thumbnail`.

## 1. Collection `course_structures` — document gốc `CourseStructure` (Lộ trình)

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `slug` | String (kebab-case) | ✓ | | **unique index**; khóa tự nhiên phát hiện trùng (Q4/AC-E5) |
| `title` | String (trim) | ✓ | | Tên Lộ trình |
| `description` | String | | `''` | Giới thiệu khóa |
| `thumbnail` | String (URL) | | `null` | **[MỚI]** ảnh đại diện/cover (feature-spec §3 Tầng 1). URL `http(s)://`, **cấm Base64** (§5.1) |
| `centerId` | ObjectId → `Center` | | `null` | **Luôn `null` Phase 1** (Q5 — dùng chung) |
| `status` | String enum `['draft','ready','published','archived']` | ✓ | `'ready'` | **[MỞ RỘNG]** đủ vòng đời (feature-spec §3 Tầng 1). Import Phase 1 luôn tạo `'ready'` (IS-6 "sẵn sàng Phase 2"); các trạng thái khác dành cho vòng đời Phase 2+ |
| `phases` | `[PhaseSchema]` | ✓ (≥1) | | Chặng — nhúng |
| `importedBy` | ObjectId → `User` | ✓ | | Người import |
| `sourceMeta` | `SourceMetaSchema` | | `null` | Metadata file `.xlsx` gốc |
| `createdAt`/`updatedAt` | Date | auto | | |

Index: `{ slug: 1 }` unique; `{ status: 1 }`.

> **BR Course "Ready cần ≥1 Chặng không rỗng" (feature-spec §3 Tầng 1):** import Phase 1 chỉ tạo document ở
> `status='ready'` **sau khi** đã pass toàn bộ validate `EMPTY_CHILDREN` (khóa ≥1 Chặng, Chặng ≥1 Chuyên đề, Chuyên
> đề ≥1 Bài học) + `EMPTY_LESSON` — nên điều kiện "≥1 Chặng không rỗng" luôn được đảm bảo trước khi khóa đạt
> `ready`. Các phép chuyển trạng thái tường minh (Draft→Ready→Published→Archived) là Phase 2+.

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
| `theory` | String (markdown) | | **[MỚI]** nội dung lý thuyết (`content_theory`). Text thuần/markdown, không HTML thô |
| `media` | `[MediaRefSchema]` | | **[MỚI]** **danh sách** link video/audio bổ trợ (`media_urls`, feature-spec §3 Tầng 4). Mỗi phần tử 1 URL; cho phép nhiều video + nhiều audio/bài. Rỗng `[]` hợp lệ |
| `exercises` | `[ExerciseRefSchema]` | | Tham chiếu bài tập (`exercise_refs`); **KHÔNG bắt buộc ≥1** (cho phép Bài học chỉ lý thuyết/media) |

> **Ràng buộc "Bài học không rỗng" (feature-spec §3 Tầng 4 BR):** một Bài học phải có **ít nhất một** trong
> {`theory`, ≥1 `media`, ≥1 `exercises`}. Rỗng hoàn toàn ⇒ lỗi `EMPTY_LESSON` (AC-E8). Nhờ vậy Bài học lý
> thuyết-thuần (không media/bài tập) là hợp lệ — đúng tầm nhìn "học đủ lý thuyết/video/âm thanh".

### `MediaRefSchema` (một link media bổ trợ) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | String enum **`['video','audio']`** | ✓ | Loại media; type khác ⇒ `UNSUPPORTED_MEDIA_TYPE` |
| `url` | String (URL) | ✓ | Link `http(s)://` (YouTube/CDN/R2 — content team tự host). **Cấm Base64/`data:`** (§5.1) ⇒ `MEDIA_BASE64_BLOCKED`; không phải http(s) ⇒ `INVALID_URL` |
| `label` | String | | Nhãn hiển thị tuỳ chọn |
| `order` | Number | ✓ | Thứ tự media trong Bài học |

> **Cấm nhúng Base64 (feature-spec §5.1 "Ràng buộc Import Media"):** hệ thống KHÔNG nhận media nhúng thẳng dạng
> `data:` (Base64) vào DB — bắt buộc lưu URL. Áp cho cả `media[].url` lẫn `thumbnail`.

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
   ≥1 trong {lý thuyết, media, bài tập} (`EMPTY_LESSON`).
4. **[MỚI] CEFR hợp lệ:** `cefrFrom`/`cefrTo` (nếu có) thuộc enum CEFR; nếu cả hai có thì `cefrTo` ≥ `cefrFrom`
   (`INVALID_CEFR`).
5. **[MỚI] Category hợp lệ:** `category` (nếu có) thuộc enum (`INVALID_CATEGORY`).
6. **[MỚI] URL hợp lệ:** mọi `media[].url` và `thumbnail` (nếu có) khớp `^https?://` (`INVALID_URL`).
7. **[MỚI] Cấm Base64 media (§5.1):** `media[].url`/`thumbnail` bắt đầu bằng `data:` ⇒ `MEDIA_BASE64_BLOCKED`.
8. **[MỚI] Type media:** chỉ `video`/`audio`; khác ⇒ `UNSUPPORTED_MEDIA_TYPE`.
9. **Type bài tập:** chỉ `ipa`/`talk`; `quiz`/khác ⇒ `UNSUPPORTED_EXERCISE_TYPE`.
10. **Exercise ID tồn tại & published (AC-3/AC-E6):** resolver registry.
11. **`slug` chưa tồn tại (AC-E5/Q4):** query + unique index safety net (`ALREADY_EXISTS`).

> Các mã lỗi con (`INVALID_CEFR`, `INVALID_CATEGORY`, `INVALID_URL`, `MEDIA_BASE64_BLOCKED`, `UNSUPPORTED_MEDIA_TYPE`,
> `EMPTY_LESSON`, …) là **code trong từng phần tử `errors[]`**, không phải hằng trong `error-codes.js` (top-level
> vẫn là `IMPORT_VALIDATION_FAILED`).

## 4. Ước lượng kích thước

`theory` là text (có thể vài KB/bài); `media[]` và `thumbnail` chỉ là **URL** (không nhúng media — §5.1 cấm Base64)
nên không phình document. Khóa ≤~5.000 bài học vẫn xa trần 16MB (feature-spec §5.1). Parser đặt ngưỡng số node báo
lỗi rõ (`research.md`).

## 5. Migration / backfill

- **Collection mới, không migration.** Chỉ đọc chéo `ipa_lessons` để validate.
- **Seed permission** `coursecontent:read/write/manage` (platform-only, KHÔNG vào `CENTER_PERMISSIONS`).
- **Dependency mới** `exceljs`.
- **Media (video/audio) do content team tự host** ngoài hệ thống (CDN/YouTube/R2) — Phase 1 chỉ lưu URL, KHÔNG
  có luồng upload/serve media cho khóa học. (Nếu sau này cần host tập trung → dùng module `media`, phase sau.)

## 6. Trường/BR forward-compat (đã bao quát nhưng cài đặt ở Phase 2+)

Các BR nâng cao trong feature-spec §5.3–§5.4 thuộc Phase 2/3 — ghi lại đây để "được bao quát" (không bỏ sót), model
đã chừa chỗ mở rộng, KHÔNG thêm trường chưa dùng ở Phase 1:

- **Structure-Lock (§5.3):** khi khóa đã `published` **và** có học viên enroll, KHÔNG cho hard-delete Chặng/Chuyên
  đề/Bài học (chỉ soft-delete/vô hiệu hóa). Phase 1: endpoint `DELETE` chặn khóa `status='published'` như **guard
  forward-compat** (Phase 1 chỉ tạo `ready` nên chưa kích hoạt) — xem `contracts/admin-course-import.md` §5.
- **Progress Recalculation (§5.3):** chèn Bài học vào Chặng đã "Completed" → lùi Chặng về "In Progress", **không**
  relock Chặng đang học. Thuộc runtime tiến độ Phase 2/3, không lưu trong `course_structures`.
- **Pass Rate (§5.4):** điểm pass cấu hình ở cấp Lộ trình hoặc override từng Lesson/Exercise. Phase 2/3 — khi làm sẽ
  thêm `defaultPassRate` (Course) + `passRate` (override Lesson/Exercise); Phase 1 chưa thêm để tránh trường rác.
- **Mở khóa tuyến tính, không chéo (§5.4):** unlock tuần tự theo `order`; không mở khóa chéo Module. Thuộc Phase 2.
