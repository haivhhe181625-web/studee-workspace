<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8. Roadmap cấp epic cho tính năng "Học theo Lộ trình". -->
# Roadmap: Học theo Lộ trình (Task-Based Learning / Course Content)

- **Ngày:** 2026-07-15
- **Tác giả:** Tech Lead Agent (AI) — chờ Product Owner/BA xác nhận thứ tự phase
- **Trạng thái:** Nháp định hướng (Phase 1 đang triển khai; Phase 2–5 mới ở mức khung)
- **Nguồn ý tưởng:** `<API_REPO>/docs/feature_plan_task_based_learning.md` (Luồng 1 — Epic 4),
  `<API_REPO>/docs/content_ingestion_schema.md`, `<API_REPO>/docs/ai_ingestion_tool_plan.md`
- **Spec Phase 1:** `specs/course-content-import/` · **ADR:** `docs/adr/0001-course-content-store-embedded-tree.md`

## 1. Tầm nhìn

Xây một hệ thống **học theo lộ trình** giúp người học **nâng cấp nền tảng tiếng Anh theo từng nấc** (ví dụ A2→B1,
B1→B2, hoặc IELTS 5.0→6.0). Mỗi khóa học là một cây 4 tầng, mỗi Bài học học **đủ lý thuyết + video + âm thanh**,
phục vụ phát triển **toàn diện Nghe – Nói – Đọc – Viết**.

## 2. Mô hình phân tầng (xuyên suốt mọi phase)

| Tầng | Vai trò | Mang dữ liệu gì |
|---|---|---|
| **Lộ trình** (Course) | Một khóa học hoàn chỉnh | tên, mô tả, slug |
| **Chặng** (Phase) | Một **nấc nâng trình độ** | `cefrFrom→cefrTo` (A2→B1) hoặc `goalNote` ("IELTS 5.0→6.0") |
| **Chuyên đề** (Module) | Một **mảng kỹ năng/chủ đề** | `category`: ngữ pháp / phát âm / từ vựng / nghe / đọc / viết / nói |
| **Bài học** (Lesson) | Đơn vị học đa phương thức | **lý thuyết (markdown) + video + audio** + tham chiếu **bài tập** |
| ↳ Bài tập (Exercise ref) | Luyện tập tương tác | trỏ tới engine có sẵn: `ipa` (phát âm), `talk` (hội thoại AI); *(sau: quiz, nghe/đọc/viết…)* |

> Mô hình này được **chốt và hiện thực hóa ngay từ Phase 1** (data model `course_structures`, xem
> `specs/course-content-import/data-model.md`). Các phase sau **tiêu thụ và làm giàu** cây này, không đổi khung.

## 3. Khối xây dựng đã có trong code (nền tảng — không làm lại)

| Thành phần | Vị trí | Dùng cho |
|---|---|---|
| Engine phát âm IPA (có audio mẫu 3 tốc độ, khẩu hình…) | `modules/ipa/` (`ipa_lessons`) | Bài tập `ipa` |
| Hội thoại nhập vai AI | `modules/talk/` (`SCENARIO_IDS` in-code) | Bài tập `talk` |
| Ngân hàng câu hỏi + CAT/MST (Nghe/Đọc/Ngữ pháp… ở dạng *đề thi*) | `modules/assessment/` (`assessment_questions`) | Nguồn tiềm năng cho `quiz` (Phase 4) |
| Lộ trình CEFR-adaptive (template→user, 2 tầng) | `modules/roadmap/` | Tiền lệ deep-copy & unlock cho Phase 2/3 |
| Tiến độ/mastery/gamification | `modules/adaptive/`, streak | Tiền lệ cho Phase 3 |
| Admin REST factory + portal | `src/admin-api/`, `exe-admin` | Bề mặt quản trị mọi phase |

## 4. Các Phase

| Phase | Tên | Mục tiêu | Ranh giới | Trạng thái |
|---|---|---|---|---|
| 0 | Nền tảng | Engine IPA/Talk/Assessment, admin factory | có sẵn | ✅ Đã có |
| **1** | **Ingestion** | Admin import cấu trúc khóa + nội dung tĩnh | api + admin | 🔄 **Đang làm** |
| 2 | Lesson Wrapper (Runtime) | Learner tiêu thụ khóa: render, unlock, học | api + web | ☐ |
| 3 | Tiến độ & Mastery | Theo dõi tiến độ, mastery, streak, unlock theo điểm | api + web | ☐ |
| 4 | Engine bài tập đủ 4 kỹ năng | Bài tập tương tác Nghe/Đọc/Viết/Ngữ pháp/Từ vựng + quiz | api (+ engine mới) | ☐ |
| 5 | Authoring nâng cao | CMS kéo-thả, AI generator, host media, edit/version | api + admin | ☐ |

### Phase 1 — Ingestion (đang triển khai)

**Giao gì:** đội học thuật **import 1 file Excel** mô tả cả cây Lộ trình→Chặng→Chuyên đề→Bài học; hệ thống
validate (cấu trúc + tham chiếu + metadata) rồi lưu **nguyên tử** thành 1 khóa "sẵn sàng". Bài học mang **lý thuyết
+ video + audio (URL) + tham chiếu bài tập ipa/talk**.

- **API (api↔admin):** import + tải template + list + detail + delete (xem `specs/course-content-import/contracts/admin-course-import.md`).
- **Phủ kỹ năng:** *nội dung tĩnh* cho cả 4 kỹ năng; *bài tập tương tác* mới có **phát âm (ipa)** + **nói (talk)**.
- **Chưa có:** learner chưa tiêu thụ được (Phase 2); bài tập Nghe/Đọc/Viết/Ngữ pháp/Từ vựng & `quiz` (Phase 4).
- Chi tiết: `specs/course-content-import/{spec,design,data-model,tasks}.md`.

### Phase 2 — Lesson Wrapper / Runtime (learner tiêu thụ)

**Giao gì:** learner **thấy và học** khóa đã import.

- Gán/ghi danh khóa cho user (assign/enroll); đọc khóa `status: 'ready'`.
- Render Lộ trình→Chặng→Chuyên đề→Bài học; mở Bài học xem **lý thuyết + phát video/audio**; mở **bài tập ipa/talk**.
- **Unlock tuần tự** cơ bản (mở Bài/Chặng kế khi hoàn thành) — logic đơn giản, chưa gắn điểm số (điểm số ở Phase 3).
- **Ranh giới mới:** **api↔web** (`exe-web`) → viết `specs/lesson-runtime/` + `contracts/web-course-consume.md`.
- Tiền lệ: deep-copy template→user của `modules/roadmap/`.

### Phase 3 — Tiến độ, Mastery, Unlock theo điểm

**Giao gì:** biến "xem được" thành "học có đo lường".

- Theo dõi **tiến độ** per Bài học/Chuyên đề/Chặng; **mastery** theo điểm bài tập; **streak**; **unlock theo ngưỡng
  điểm** (vd đạt ≥80% mới mở bài kế).
- Aggregate lên "đã tiến bộ A2→B1 tới đâu" — khớp ngữ nghĩa `cefrFrom/cefrTo` gắn ở Chặng từ Phase 1.
- Tiền lệ: `modules/adaptive/`, user-streak.

### Phase 4 — Engine bài tập đủ 4 kỹ năng

**Đây là phase làm "toàn diện Nghe–Nói–Đọc–Viết" trở thành *luyện tập tương tác*, không chỉ lý thuyết.**

- Xây/nối engine bài tập cho **Nghe, Đọc, Viết, Ngữ pháp, Từ vựng**; mở **`quiz`** (chốt "một quiz = gì" — nguồn
  ứng viên `assessment_questions`; đây chính là câu hỏi **Q-Quiz** đã hoãn ở Phase 1).
- Mở rộng enum `ExerciseRefSchema.type` (Phase 1 mới có `['ipa','talk']`) → thêm `quiz`, `listening`, `writing`…
- Sau phase này, `ExerciseRef` phủ đủ 4 kỹ năng ⇒ Bài học "học đủ" đúng tầm nhìn.

### Phase 5 — Authoring nâng cao

- **CMS kéo-thả** dựng khóa trực tiếp trên UI (Luồng 2 của idea).
- **AI Course Generator** (`ai_ingestion_tool_plan.md`): mô tả ngắn → AI sinh cây khóa để review & publish.
- **Host media tập trung** (upload video/audio thay vì chỉ URL — dùng `modules/media/`), sinh TTS tự động.
- **Sửa/versioning** khóa đã import qua UI (bỏ ràng buộc "xoá-rồi-import-lại" của Phase 1).

## 5. Ma trận phủ 4 kỹ năng theo phase (quan trọng — tránh hiểu nhầm)

| Kỹ năng | Nội dung tĩnh (lý thuyết/video/audio) | Bài tập tương tác |
|---|---|---|
| Phát âm | Phase 1 | **Phase 1** (engine `ipa`) |
| Nói | Phase 1 | **Phase 1** (engine `talk`) |
| Nghe | Phase 1 | Phase 4 |
| Đọc | Phase 1 | Phase 4 |
| Viết | Phase 1 | Phase 4 |
| Ngữ pháp | Phase 1 | Phase 4 (`quiz`) |
| Từ vựng | Phase 1 | Phase 4 (`quiz`) |

> **Đọc bảng này:** ngay Phase 1, mọi kỹ năng đều có **lý thuyết + video + audio**. Nhưng **luyện tập tương tác**
> chỉ có Phát âm + Nói cho tới **Phase 4** (khi có engine cho các kỹ năng còn lại). Đây là đánh đổi có ý thức để
> Phase 1 giao được sớm mà không chờ xây 5 engine mới.

## 6. Phụ thuộc & thứ tự

```
Phase 0 (đã có) ──► Phase 1 (Ingestion) ──► Phase 2 (Runtime) ──► Phase 3 (Tiến độ)
                                    │                                    ▲
                                    └────────► Phase 4 (Engine bài tập) ─┘  (Phase 4 làm giàu bài tập cho P2/P3)
Phase 5 (Authoring) song song, sau khi P1 ổn định.
```

- Phase 2 **chặn** bởi Phase 1 (cần khóa `ready` để render).
- Phase 3 cần Phase 2 (cần runtime để phát sinh tiến độ).
- Phase 4 **độc lập tương đối** — có thể chạy song song sau Phase 1; khi xong thì Bài học ở P2/P3 tự có thêm loại
  bài tập.
- Phase 5 là nâng UX/tự động hóa, không chặn P2/P3/P4.

## 7. Quyết định mở lớn (cần chốt khi tới phase tương ứng)

- **Q-Quiz (Phase 4):** "một quiz trong Bài học" ánh xạ tới gì? (tập `assessment_questions`? khái niệm mini-test
  mới?) — đã nêu từ Phase 1, hoãn.
- **Assign/enroll (Phase 2):** learner được gán khóa thế nào (tự chọn / theo placement / admin gán)?
- **Đa tenant (Phase 2+):** Phase 1 dùng chung `centerId=null`; khi nào cho center tạo khóa riêng? (model đã chừa
  cột `centerId`.)
- **Versioning (Phase 5):** đổi từ "xoá-rồi-import-lại" sang version/edit khi có consumer thật.

## 8. Liên kết

- Phase 1: `specs/course-content-import/` (spec, acceptance, research, design, data-model, contracts, tasks,
  open-questions).
- ADR: `docs/adr/0001-course-content-store-embedded-tree.md`.
- Ý tưởng gốc: `<API_REPO>/docs/feature_plan_task_based_learning.md`,
  `<API_REPO>/docs/content_ingestion_schema.md`, `<API_REPO>/docs/ai_ingestion_tool_plan.md`.
