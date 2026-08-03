<!-- Tiếng Việt — tasks Phase 1.5 "Program dimension" (phân biệt IELTS/TOEIC/general trên khóa học).
     Chốt (b): triển khai slice tối thiểu, không viết framing doc. Q1–Q6 đã duyệt (xem "Quyết định"). -->
# Tasks: Phase 1.5 — Program/Track dimension

Thêm chiều **chương trình** (kỳ thi) cho khóa học: `general | ielts | toeic` (mở rộng bằng thêm key). CEFR vẫn là
xương sống nội bộ; mỗi program có **thang điểm gốc** riêng (band vs điểm) tái dùng `constants/cefr-mapping.js`.

## Quyết định đã chốt (Q1–Q6)
- **Q1** program ban đầu: `general, ielts, toeic`. **Q2** 1 khóa = **đúng 1** program.
- **Q3** `targetScore` ở **Course** (theo đơn vị program). **Q4** hiển thị **song song** CEFR + điểm gốc (dùng cefr-mapping).
- **Q5** ràng buộc skills/exercise theo program = **mềm** (warning khi import, không chặn). **Q6** tách onboarding exam → **hoãn**.

## Nhánh (chain vì chưa merge develop)
- exe-api: `feature/course-program` (off `feature/course-progress`).
- exe-admin: tiếp `feature/course-management-admin`.
- exe-web: `feature/course-program-web` (off `feature/course-progress-web`).

## Nguyên tắc tương thích ngược (quan trọng — xuyên suốt)
- `program` **default `'general'`**; khóa đã import (không có field) ⇒ coi như `general`.
- Cần **backfill 1 lần**: script `scripts/backfill-course-program.js` set `program:'general'` cho doc thiếu field
  (để filter `?program=general` khớp) — HOẶC query filter tự coi thiếu field = general (PR-T4 chốt cách).
- Thêm index phục vụ filter (PR-T2) để không chậm catalog.

---

## PR-T1 — Registry chương trình (exe-api)
- Create `constants/programs.js` (Opt 1, in-code):
  ```
  PROGRAMS = {
    general: { key:'general', label:'Tổng quát', scale:null,                                   skills:['listening','reading','writing','speaking','grammar','vocabulary','pronunciation'] },
    ielts:   { key:'ielts',   label:'IELTS', scale:{type:'band',   min:0,  max:9,  step:0.5, unit:'band'},  cefrRef:'ielts', skills:['listening','reading','writing','speaking'] },
    toeic:   { key:'toeic',   label:'TOEIC', scale:{type:'points', min:10, max:990,step:5,  unit:'điểm'}, cefrRef:'toeic', skills:['listening','reading'] },
  }
  export: PROGRAMS, PROGRAM_KEYS, isProgram(key), getProgram(key)
  + helper scoreBandForCefr(programKey, cefr) → dùng cefr-mapping.js (theo cefrRef) trả {min,max,unit} | null (Q4).
  + helper validateTargetScore(programKey, value) → trong [scale.min,scale.max] & đúng step; general ⇒ phải null.
  ```
- `cefrRef` trỏ vào `EXAM_TO_CEFR`/bảng có sẵn trong `cefr-mapping.js` (ielts/toeic đã có) — KHÔNG chép lại số.
- Test `programs.test.js`: isProgram, scoreBandForCefr('ielts','B2')→~5.5–6.5, validateTargetScore range/step, general null.
- Commit: `feat(course-content): program registry (ielts/toeic/general) reusing cefr-mapping`.

## PR-T2 — Course model (exe-api)
- `course-content.model.js`: `program: { enum: PROGRAM_KEYS, default:'general', index:true }`; `targetScore: { type:Number, default:null }`.
- Index catalog: `{ status:1, program:1 }` (thay/bổ sung index status) để filter published+program nhanh.
- `scripts/backfill-course-program.js` (safe, idempotent): `updateMany({program:{$exists:false}},{ $set:{program:'general'} })`.
- Test `course-content.model.test.js` (bổ sung): default 'general'; enum chặn giá trị lạ.
- Commit: `feat(course-content): course program + targetScore fields (+ backfill)`.

## PR-T3 — Import pipeline (template + parser + validate) (exe-api)
- **Template**: thêm ô cấp-khóa **"Chương trình"** (`program`) + **"Điểm mục tiêu"** (`targetScore`) ở sheet/meta khóa
  (nơi đang có title/description/thumbnail). Cập nhật endpoint `GET .../template`.
- **Parser** `course-content.parser.js`: đọc `program` (default 'general' nếu trống), `targetScore`.
- **Validate** `course-content.service.js`:
  - `program` phải ∈ registry (else error `INVALID_PROGRAM`). `targetScore` qua `validateTargetScore` (else error `INVALID_TARGET_SCORE`).
  - **Q5 warnings (không chặn):** nếu module `category`/exercise nằm ngoài `program.skills` → đẩy vào **`warnings[]`**
    (kênh MỚI, không phải errors[]) → import vẫn thành công, trả kèm cảnh báo. Code `PROGRAM_SKILL_MISMATCH`.
  - `importCourse`/`buildDoc` gán `program`, `targetScore` vào doc.
- Error code mới: `INVALID_PROGRAM`, `INVALID_TARGET_SCORE` (+ warning code `PROGRAM_SKILL_MISMATCH`).
- Tests: parser đọc program/target; validate reject program lạ + target ngoài scale; warning khi TOEIC có module writing;
  general + target≠null → reject.
- Commit: `feat(course-content): parse + validate course program/targetScore with soft skill warnings`.

## PR-T4 — Catalog & detail API (exe-api)
- `course-content.public.service.js`:
  - `listPublishedCourses({ program })` — filter optional theo program (thiếu field = general); projection **+`program`, +`targetScore`**.
  - `getPublishedCourseBySlug` — trả `program`, `targetScore`, và **per-phase `scoreBand`** (Q4, từ `scoreBandForCefr(program, phase.cefrTo/cefrFrom)`).
- `controller.listCourses` đọc `req.query.program`; route giữ nguyên (`GET /api/courses?program=`).
- Cập nhật hợp đồng `specs/lesson-runtime/contracts/web-course-consume.md` (§B1/§B2 thêm program/targetScore/scoreBand).
- Tests `course-content.public.api.test.js` (bổ sung): filter `?program=ielts` chỉ trả IELTS (+ khóa thiếu field lọt vào general);
  detail có program + scoreBand.
- Commit: `feat(course-content): catalog program filter + program/score in learner API`.

## PR-T5 — exe-admin (import form + detail edit)
- Import page: thêm **select "Chương trình"** + input "Điểm mục tiêu" (ẩn/khoá khi program=general). Hiển thị `warnings[]` sau import.
- Course-detail edit: sửa/hiển thị `program` + `targetScore`; badge program ở list.
- (admin-api `course-content.admin.js` writeFields whitelist thêm `program`,`targetScore` nếu cần cho PUT/edit.)
- Commit: `feat(course-import): program selector + target score + import warnings`.

## PR-T6 — exe-web (filter tabs + hiển thị)
- `course.service.getCourses(program?)` + `use-courses` truyền filter; types `CourseSummary/CourseDetail` +`program`,`targetScore`, phase +`scoreBand`.
- **CourseCatalogContainer**: tab lọc **Tất cả / IELTS / TOEIC** (từ registry, không hardcode); `QUERY_KEYS.courses.list(program)`.
- **CourseCard**: badge program. **CourseDetailContainer**: hiển thị program + targetScore; mỗi Chặng show **CEFR + điểm gốc** ("B2 · IELTS ~5.5–6.5") từ `scoreBand`.
- Commit: `feat(courses): program filter tabs + dual CEFR/native score display`.

## PR-T7 — Verify + chốt
- exe-api `jest course-content programs`; exe-web `tsc`+`lint`+`build`. Backfill script chạy thử. Cập nhật spec. Báo cáo.

---

## Kết quả Phase 1.5 — ✅ Xong

| Task | Commit | Repo · nhánh |
|---|---|---|
| PR-T1 registry `programs.js` | `0780e7b` | exe-api · `feature/course-program` (off `feature/course-progress`) |
| PR-T2 model program/targetScore + backfill | `ece69ce` | exe-api |
| PR-T3 parser/validate + warnings mềm | `023a609` | exe-api |
| PR-T4 catalog filter + score API | `8a988c6` | exe-api |
| PR-T5 backend enabling + exe-admin UI | `bf92f9e` (api) · `6b4b3bd` (admin) | exe-api · exe-admin `feature/course-management-admin` |
| PR-T6 exe-web filter + dual score | `354804e` | exe-web · `feature/course-program-web` (off `feature/course-progress-web`) |

**Verify:**
- exe-api full `jest` → **932 pass / 2 fail** (2 = `avatar-storage` EPERM Windows, môi trường — không hồi quy).
  +32 test Phase 1.5 (programs 12 · program.model 3 · program.import 12 · program.api 5).
- exe-admin `tsc`/`lint`/`build` **OK**. exe-web `tsc`/`lint`/`build` **OK**.
- `scripts/backfill-course-program.js` — syntax OK (`node --check`); **cần chạy 1 lần khi deploy** trên DB thật
  (`docker compose exec <api> node scripts/backfill-course-program.js`) để set `program:'general'` cho khóa tiền-1.5.

**Chốt quyết định (Q1–Q6):** registry `general/ielts/toeic`; 1 khóa = 1 program; targetScore ở Course; dual CEFR+điểm
gốc (tái dùng cefr-mapping); warnings mềm skills; onboarding hoãn.

**Tương thích ngược:** additive; khóa cũ = general (backfill + `$ifNull`/`$exists` phòng vệ ở query); index
`{status,program}` cho filter. **Không đụng** Phase 3.

**Chưa push/PR** cả 3 nhánh. Deploy: exe-api → exe-admin → exe-web (+ chạy backfill sau exe-api).

---

### Ghi chú triển khai (đọc trước khi code)
- **Registry là nguồn sự thật duy nhất** cho danh sách program (BE enum + FE tabs đều lấy từ đây/qua API) — thêm kỳ
  thi = thêm 1 entry (+ concordance trong cefr-mapping nếu chưa có). Tránh hardcode 'ielts'/'toeic' rải rác.
- **`warnings[]` là kênh mới, tách khỏi `errors[]`** — errors chặn import (all-or-nothing giữ nguyên), warnings chỉ
  thông báo (Q5 mềm). Đừng nhét vào errors.
- **Không đụng Phase 3** (tiến độ per-lesson, track-agnostic). **Tái dùng** cefr-mapping, **không** chép số điểm.
- Backward-compat: mọi thay đổi additive; khóa cũ = general; có backfill; index để filter không chậm.
