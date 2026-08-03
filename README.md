<!-- Tiếng Việt — tài liệu điều hướng BA/plan cho studee-workspace. Cập nhật khi thêm/đổi trạng thái feature. -->
# Studee Workspace — Bảng điều khiển Feature (BA / Plan)

Repo tài liệu đặc tả (specs) của Studee: mỗi feature đi qua pipeline **Idea → Merge** có 1 thư mục ở
`specs/<feature-slug>/`. File này là **điểm vào duy nhất** để nắm nhanh *đang có những gì* và *đang ở trạng thái nào*.

- 🟢 **Mỗi feature có `prd.md`** — bản PRD kiểu SRS cho người **KHÔNG kỹ thuật** (founder/giáo vụ/BA đọc trước, không endpoint/schema). Muốn chi tiết kỹ thuật thì vào `spec.md` / `design.md`.
- Quy ước thư mục specs: [`specs/README.md`](specs/README.md)
- Vòng đời đầy đủ (vai trò, gate duyệt, exit criteria): [`.ai/workflows/feature-workflow.md`](.ai/workflows/feature-workflow.md)
- Mẫu tài liệu: [`templates/`](templates/) · Hướng dẫn đội: [`TEAM-GUIDE.md`](TEAM-GUIDE.md)
- Doc API tích hợp (đã chạy, chi tiết endpoint): `exe-api/docs/api/integration/`

---

## Vòng đời 1 feature (11 bước)

```
Idea → AI Brainstorm → BA Review → Specification → Tech Lead Design → Technical Review
     → Task Generation → Development → AI Self Review → Tech Lead Review → Merge
```

Gộp thành 4 nhóm dễ đọc:

| Ký hiệu | Nhóm | Ứng với bước |
|---|---|---|
| 🟣 **BA** | Nghiệp vụ đang định hình | Idea · Brainstorm · BA Review · Specification |
| 🔵 **Design** | Kỹ thuật đang thiết kế | Tech Lead Design · Technical Review |
| 🟠 **Code** | Đang triển khai | Task Generation · Development |
| 🟢 **Done** | Đã ghép/đang chạy | Self Review · Tech Review · Merge |
| ⚪ **Backlog** | Chưa có plan/code | (ngoài pipeline) |

> **Hai trục trạng thái** — đọc bảng dưới cần phân biệt:
> - **Feature** = code đã chạy tới đâu.
> - **Tài liệu** = bộ spec đã hoàn chỉnh/được duyệt tới đâu.
> Nhiều feature dưới đây **đã chạy** nhưng tài liệu là **spec hồi tố** (AI viết lại từ code + plans) → *feature = 🟢 Done*
> nhưng *tài liệu vẫn cần BA/Tech review người thật ký duyệt*.

---

## Dòng thời gian phát triển (theo ngày)

> Ngày lấy từ tên plan (`YYMMDD`) + lịch sử code. Mỗi dòng là một câu **kiểu SRS** — *"Hệ thống cho phép / sẽ …"* — để nắm nhanh feature làm gì.

### ≈ 15–20/07/2026 · Nhập liệu & học tập cơ bản
- **course-content-import** (15/07) · 🟠 Code — *Hệ thống cho phép đội học thuật nhập cả khóa học 4 tầng từ 1 file, tự kiểm tra hợp lệ toàn-hoặc-không rồi lưu vào kho nội dung số (cấm bài học rỗng).*
- **course-management-admin** (≈16/07) · 🟢 Done — *Hệ thống cho phép quản trị viên tạo/sửa khóa và điều khiển vòng đời trạng thái xuất bản: nháp → sẵn sàng → xuất bản → lưu trữ.*
- **lesson-runtime** (17/07) · 🟢 Done — *Hệ thống cho phép học viên đã ghi danh mở và học nội dung bài học (lý thuyết/video/bài tập); nội dung trả phí chỉ hiện cho người đã ghi danh.*
- **course-progress** (≈20/07) · 🟢 Done — *Hệ thống theo dõi tiến độ học, tự mở khóa bài kế tiếp theo thứ tự và ghi nhận chuỗi ngày học (streak).*

### ≈ 17–24/07/2026 · Năng lực, lộ trình & engine bài tập
- **adaptive-path** (PRD 17/07) · 🟢 Done — *Hệ thống dựng hồ sơ năng lực + lộ trình cá nhân hóa theo trình độ/mục tiêu.*
- **exercise-engines** (≈24/07) · 🟠 Code — *Hệ thống chấm bài tập viết/nói và đưa kết quả vào đánh giá năng lực (nhánh quiz đã chuyển sang course-graded-exercises).*

### 24–27/07/2026 · Bản đồ game — nền & người chơi
- **learning-map-content-players** (26–27/07) · 🟢 Done — *Hệ thống cho học viên làm trắc nghiệm / thẻ ghi nhớ / video ngay trên nút bản đồ, chấm điểm phía máy chủ và chống gian lận.*
- **map-template-admin-builder** (26–27/07) · 🟢 Done (BE) — *Hệ thống cho phép quản trị viên soạn bản đồ học (chặng/nút + gán bài) và tự chọn đúng bản đồ theo trình độ + chương trình của học viên.*

### 28/07/2026 · Khóa điều phối bản đồ + bài chấm điểm
- **course-driven-map** (28/07) · 🟢 Done — *Hệ thống lấy khóa học làm nguồn gốc, tự sinh bản đồ game; cho học viên xem trước lộ trình trước khi thi xếp lớp và gộp về một màn học duy nhất.*
- **course-graded-exercises** (28/07) · 🟢 Done — *Hệ thống chấm quiz/flashcard trong bài học (đạt ≥ 80%) và đưa kết quả vào đánh giá năng lực; giáo vụ soạn quiz bằng cách chọn nguyên cụm bài từ ngân hàng câu hỏi.*

### 29–30/07/2026 · Checkpoint, band-up & snapshot
- **course-checkpoints-band-up** (29–30/07) · 🟢 Done — *Hệ thống bắt học viên vượt bài kiểm tra cuối chặng mới mở chặng sau, và nâng dần trình độ CEFR theo điểm; thi lại không giới hạn, không phạt.*
- **adaptive-path · StudentPath snapshot** (30/07) · 🟢 Done — *Hệ thống lưu ảnh chụp lộ trình đã giao để nâng trình không làm mất chặng đã học; đổi lộ trình là hành động chủ động (bấm nút).*

### 03/08/2026 · Tài liệu hồi tố (đợt này)
- Viết retro specs cho 5 feature bản đồ/checkpoint/graded + mở rộng adaptive-path; **11 `prd.md`** (PRD non-tech) cho từng feature; và file dashboard này.

---

## Bảng trạng thái feature

Bề mặt: 👤 learner · 🛠 admin · ⚙️ core/nền.

| Feature | Ngày | Bề mặt | Feature | Tài liệu spec | Verify code (đợt này) | Ghi chú |
|---|:---:|:---:|:---:|---|:---:|---|
| [course-content-import](specs/course-content-import/) — Import cấu trúc khóa (Phase 1) | 15/07 | 🛠 | 🟠 Code | 🟢 Đã duyệt Tech Review (2026-07-15); đủ bộ + research/open-questions | — | Đang hoàn thiện "rich lessons" (còn WIP chưa commit) |
| [lesson-runtime](specs/lesson-runtime/) — API learner đọc bài (Phase B) | 17/07 | 👤 | 🟢 Done | 🔵 design + tasks + contract; *thiếu spec/acceptance* | — | Spec cũ, chưa verify lại đợt này |
| [course-progress](specs/course-progress/) — Tiến độ khóa (Phase 3) | ≈20/07 | 👤 | 🟢 Done | 🔵 design + data-model + tasks + contract; *thiếu spec/acceptance* | — | Spec cũ, chưa verify lại đợt này |
| [course-management-admin](specs/course-management-admin/) — Quản lý khóa (Admin, GĐ A) | ≈16/07 | 🛠 | 🟢 Done | 🔵 design + tasks + contract; *thiếu spec/acceptance* | — | Trạng thái theo tài liệu, chưa verify lại |
| [exercise-engines](specs/exercise-engines/) — Engine bài tập (Phase 4) | ≈24/07 | 👤 | 🟠 Code | 🔵 design + tasks; *thiếu spec/acceptance* | — | Nhánh quiz đã **thay** bằng course-graded-exercises (LessonQuiz) |
| [adaptive-path](specs/adaptive-path/) — Hồ sơ năng lực & lộ trình thích nghi (+ StudentPath snapshot) | 17/07 · 30/07 | ⚙️ | 🟢 Done | 🔵 design + data-model + contract (+ StudentPath hồi tố); *thiếu spec/acceptance* | ✅ (phần StudentPath) | Band-up primitive + snapshot bất biến chống mất chặng khi band-up |
| [learning-map-content-players](specs/learning-map-content-players/) — Player node MCQ/Flashcard/Video | 26–27/07 | 👤 | 🟢 Done | 🟢 Đủ bộ — *spec hồi tố, chờ review người thật* | ✅ | Chấm server-key, tamper-guard, HARD-gate |
| [map-template-admin-builder](specs/map-template-admin-builder/) — Soạn CourseMapTemplate + chọn template | 26–27/07 | 🛠 | 🟢 Done (BE) | 🟢 Đủ bộ — *spec hồi tố* | ✅ | FE admin ngoài scope tài liệu này |
| [course-driven-map](specs/course-driven-map/) — CourseStructure → bản đồ (view) + explorer + surface | 28/07 | 👤 | 🟢 Done | 🟢 Đủ bộ — *spec hồi tố* | ✅ | Đã sửa endpoint về `/api/adaptive/map*` |
| [course-checkpoints-band-up](specs/course-checkpoints-band-up/) — Checkpoint chặng/khóa + band-up | 29–30/07 | 👤 | 🟢 Done | 🟢 Đủ bộ — *spec hồi tố* | ✅ | Hợp nhất 4 plan → bandPoint fractional (bỏ +1-CEFR cũ) |
| [course-graded-exercises](specs/course-graded-exercises/) — Quiz/Flashcard chấm điểm → mastery | 28/07 | 👤 | 🟢 Done | 🟢 Đủ bộ — *spec hồi tố* | ✅ | LessonQuiz curated + ngân hàng câu hỏi, ngưỡng 0.8 |
| [course-program](specs/course-program/) — (chưa rõ phạm vi) | — | ⚙️ | 🟣 BA | 🟣 chỉ có tasks — **nháp** | — | Cần BA làm rõ mục tiêu/phạm vi |

**Chú giải "Tài liệu spec":** *Đủ bộ* = có `spec.md` + `acceptance.md` + `design.md` + `data-model.md` + `contracts/`.
Các spec cũ *thiếu spec/acceptance* nghĩa là mới có phần Tech (design/tasks), chưa viết phần BA.

---

## Backlog — có ý tưởng, **chưa** có plan/code

Nguồn: [`../plans/Prep_System_Flow_Spec.md`](../plans/Prep_System_Flow_Spec.md) (tài liệu luồng hệ thống tổng).

| Ý tưởng | Nhóm | Ghi chú |
|---|:---:|---|
| Đối soát cam kết đầu ra (output-guarantee): video 100% + bài tập 100% + giới hạn trễ hạn → voucher/hoàn phí | ⚪ Backlog | §4.3 Prep spec — chưa có plan, chưa specced |
| Mock-test chấm người thật cho Speaking/Writing | ⚪ Backlog | §4.1 Prep spec — chưa có engine/plan |

---

## Trạng thái Git của repo (2026-08-03)

> Repo này **chỉ mới commit** `course-content-import` từ trước. Đợt này đã commit thêm 6 thư mục
> (`9e25e6e` trên nhánh `feature/course-content-rich-lessons`): 5 spec mới hồi tố + `adaptive-path`.

**Chưa commit** (WIP các session trước, cần chủ nhân xử lý): `course-content-import` (đang sửa),
nhiều file `docs/*` (PRD/roadmap), và các thư mục specs cũ `course-progress` · `exercise-engines` ·
`lesson-runtime` · `course-management-admin` · `course-program` → link tham chiếu chéo từ spec mới sẽ
"treo" cho tới khi các thư mục này được commit.

---

## Cách cập nhật file này

- Thêm feature mới hoàn thành 1 bước pipeline → cập nhật cột **Feature** / **Tài liệu spec** tương ứng.
- Feature "spec hồi tố" sau khi được BA/Tech review người thật ký → đổi ghi chú, bỏ chữ *"chờ review người thật"*.
- Giữ đúng thứ tự: nền/cũ trước, các feature bản đồ game + checkpoint + graded ở giữa, nháp/backlog cuối.
