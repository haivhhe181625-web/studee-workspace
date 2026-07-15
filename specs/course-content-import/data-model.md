# Data model: Import cấu trúc Khóa học tĩnh (Phase 1)

- **Spec:** `specs/course-content-import/spec.md`
- **Design:** `specs/course-content-import/design.md`
- **Ngày:** 2026-07-15 (đã áp quyết định Q1–Q6 + Q-Quiz của PO)

> Model lưu **1 khóa học = 1 document nhúng cả cây** trong collection mới `course_structures` (lý do nguyên tử
> IS-5/AC-4: `research.md`). Định danh code giữ tiếng Anh; mô tả tiếng Việt (`.ai/project-context.md` §8). Tiền lệ
> nhúng: `roadmap.model.js` (`RoadmapTemplateSchema.phases[]`).

## 1. Collection `course_structures` — document gốc `CourseStructure`

| Field | Kiểu | Bắt buộc | Mặc định | Ghi chú |
|---|---|---|---|---|
| `_id` | ObjectId | auto | | |
| `slug` | String (kebab-case) | ✓ | | **unique index** — roadmap ID trong Excel; khóa tự nhiên phát hiện trùng (Q4/AC-E5). Trùng ⇒ từ chối. |
| `title` | String (trim) | ✓ | | Tên Lộ trình (tầng gốc) |
| `description` | String | | `''` | |
| `centerId` | ObjectId → `Center` | | `null` | **Luôn `null` ở Phase 1** (Q5 — nội dung platform dùng chung). Giữ cột để mở rộng center-scoped sau. |
| `status` | String enum `['ready']` | ✓ | `'ready'` | IS-6 "sẵn sàng Phase 2". Chừa mở rộng sau — KHÔNG thêm giá trị chưa dùng ở Phase 1. |
| `phases` | `[PhaseSchema]` | ✓ (≥1) | | Chặng — nhúng; rỗng ⇒ chặn ở validate (AC-E2) |
| `importedBy` | ObjectId → `User` | ✓ | | Người import (đối soát) |
| `sourceMeta` | `SourceMetaSchema` | | `null` | Metadata file `.xlsx` gốc |
| `createdAt` / `updatedAt` | Date | auto | | `timestamps: true` |

### Index

- `{ slug: 1 }` **unique** — chống trùng khóa (Q4/AC-E5); cũng là safety net khi đua ghi.
- `{ status: 1 }` — Phase 2 lọc khóa `ready`.
- (KHÔNG cần index `centerId` ở Phase 1 vì luôn `null`; thêm khi mở center-scoped.)

## 2. Sub-document (nhúng) — cây phân tầng

### `PhaseSchema` (Chặng) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Định danh Chặng **duy nhất trong khóa** (dò trùng nội bộ — AC-E3) |
| `title` | String | ✓ | |
| `order` | Number | ✓ | Thứ tự hiển thị |
| `modules` | `[ModuleSchema]` | ✓ (≥1) | Chặng không có Chuyên đề ⇒ lỗi cấu trúc (AC-2) |

### `ModuleSchema` (Chuyên đề) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Duy nhất trong Chặng cha (AC-E3) |
| `title` | String | ✓ | |
| `order` | Number | ✓ | |
| `lessons` | `[LessonSchema]` | ✓ (≥1) | Chuyên đề không có Bài học ⇒ lỗi cấu trúc (AC-2) |

### `LessonSchema` (Bài học) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `key` | String | ✓ | Duy nhất trong Chuyên đề cha (AC-E3) |
| `title` | String | ✓ | |
| `order` | Number | ✓ | |
| `exercises` | `[ExerciseRefSchema]` | ✓ (≥1) | Xem ghi chú "Bài học rỗng" |

> **Ghi chú "Bài học rỗng":** design đề xuất ≥1 bài tập/Bài học (khóa học để learner luyện). Nếu đội học thuật cần
> Bài học "chỉ lý thuyết", nới ràng buộc — chốt ở Technical Review (không chặn thiết kế chính).

### `ExerciseRefSchema` (Tham chiếu bài tập) — `{ _id: false }`

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | String enum **`['ipa','talk']`** | ✓ | **Phase 1 chỉ `ipa`+`talk`** (Q-Quiz: `quiz` hoãn Phase 2). File có `quiz` ⇒ lỗi `UNSUPPORTED_EXERCISE_TYPE` |
| `refId` | String | ✓ | IPA = `ipa_lessons.code` (phải `status:'published'` — Q3); talk = phần tử `SCENARIO_IDS` |
| `order` | Number | ✓ | Thứ tự trong Bài học |
| `label` | String | | Nhãn hiển thị tuỳ chọn |

**Tham chiếu "mềm" (soft reference), CỐ Ý không phải Mongoose `ref`/ObjectId:**

- Talk **không có collection** — scenario là hằng in-code (`talk.scenarios.js: SCENARIO_IDS`) ⇒ không thể ref.
- IPA phân giải theo **`code`** ("L1"…) — mã người-đọc-được đội học thuật điền, không phải `_id`.
- `refId` là `String`, validate bằng resolver registry lúc import (IS-4), không dùng `populate`. Toàn vẹn tham
  chiếu đảm bảo **tại thời điểm import**; Phase 2 khi render nên kiểm tra lại tồn tại (bài tập có thể bị xoá/unpublish
  sau) — ghi chú cho spec Phase 2.

### `SourceMetaSchema` — `{ _id: false }`

| Field | Kiểu | Ghi chú |
|---|---|---|
| `filename` | String | Tên file `.xlsx` gốc |
| `contentHash` | String | Hash nội dung (đối soát / phát hiện re-import y hệt) |
| `nodeCount` | Number | Tổng số node — dùng cho ngưỡng an toàn (≤ ~5.000 bài học, Q6) trước trần 16MB |
| `format` | String | `'xlsx'` (Phase 1 chỉ Excel — Q2) |

## 3. Ràng buộc toàn vẹn (kiểm ở tầng validate trong service, trước khi ghi)

Gom lỗi trả 1 lần (IS-7), mỗi lỗi kèm **số dòng Excel**:

1. **Cha–con hợp lệ (AC-2):** IR giữ quan hệ cây → node "mồ côi" phát hiện khi dựng cây.
2. **Không trùng `key` trong cùng phạm vi cha (AC-E3):** báo lỗi `DUPLICATE_KEY` + dòng.
3. **Không rỗng (AC-E2):** khóa ≥1 Chặng, và (đề xuất) mỗi tầng có ≥1 con tới `exercises`.
4. **Type bài tập hợp lệ (Q-Quiz):** chỉ `ipa`/`talk`; `quiz` ⇒ `UNSUPPORTED_EXERCISE_TYPE`.
5. **Exercise ID tồn tại & đã publish (AC-3/AC-E6/IS-4):** resolver registry (IPA lọc `status:'published'`).
6. **`slug` chưa tồn tại trong DB (AC-E5/Q4):** query kiểm tra trong pha validate → `ALREADY_EXISTS` + dòng; unique
   index là safety net khi đua ghi (E11000 → cùng lỗi).

## 4. Ước lượng kích thước (đối chiếu trần 16MB BSON)

Mỗi node metadata thuần cỡ vài trăm byte → khóa Phase 1 (≤~5.000 bài học, Q6) cỡ ≤ vài MB, xa trần 16MB. Parser
vẫn đặt **ngưỡng số node** để báo lỗi rõ thay vì để Mongo ném lỗi ghi. Xem `research.md` §Rủi ro còn lại.

## 5. Migration / backfill

- **Collection mới, không migration dữ liệu cũ** — không đụng `roadmap_templates`, `courses`, `ipa_lessons`
  (chỉ đọc để validate).
- **Seed permission:** thêm `coursecontent:read/write/manage` vào `src/constants/permissions.js` và cấp cho tài
  khoản đội học thuật/platform admin (design §11). **KHÔNG** đưa vào `CENTER_PERMISSIONS` (Q5). Đây là thay đổi cấu
  hình quyền, không phải migration schema.
- **Dependency mới:** `exceljs` vào `services/api/package.json` (Q2, ADR-0001).
- **Index tạo tự động** khi model đăng ký; DB chưa có dữ liệu khóa nên không rủi ro trùng lúc tạo unique index.
