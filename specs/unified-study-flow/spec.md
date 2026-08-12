# Spec: Luồng học hợp nhất v1 — "Nhiệm vụ hôm nay" dẫn vào không gian học

- **Ngày:** 2026-08-05 · **Trạng thái:** ✅ ĐÃ DUYỆT (owner "clear, mặc định hết" — 2026-08-05). D1–D5 chốt; 2 lưu ý (cổng cứng đã có sẵn · deep-link mức module) đã chấp nhận. → Sẵn sàng code (superpowers, 5 task).
- **Repos/surfaces:** `api` (`exe-api`, `feature/study-schedule-generation`) · `web` (`exe-web`, `feature/schedule-ui-integration`).
- **Nền tảng:** nối `specs/study-schedule-generation` + `schedule-module-finishing` + `schedule-config-projection`.

## 1. Mục tiêu
Biến **Lịch học** thành lớp mỏng **"Nhiệm vụ hôm nay"** dẫn học viên đi **một đường**: học bài (ngày thường) → ôn tập (cuối tuần) → vượt chặng (checkpoint) → phase kế. **KHÔNG rebuild không gian học** — lịch chỉ **dẫn vào** study-space đã có ở lộ trình + **đánh dấu tiến độ**.

## 2. Insight từ scout (2026-08-05) — vì sao build nhỏ
Mọi hạ tầng đã tồn tại; chỉ vướng 1 gap gốc: **item `LessonSchedule` thiếu `courseSlug/phaseKey/moduleKey`**. Enrich context này là **keystone** — mở khoá cả deep-link, tick, checkpoint.
- **Deep-link có sẵn:** `ADAPTIVE_PATH_STUDY(slug)?phase=&module=` (bài) · `+&checkpoint=1` (vượt chặng).
- **Tiến độ có sẵn:** `useCourseProgress(slug)` → `lessons[{phase/module/lesson}→status]` + `passedCheckpoints[]` (đọc từ `CourseEnrollment` + `CheckpointAttempt`).
- **Chấm checkpoint + nâng band có sẵn & LIVE:** `POST /courses/:slug/checkpoint/submit` → `applyCheckpointResult` (BR-28). (Khác `/adaptive/checkpoint` đang rào.)
- **Cổng cứng cơ bản đã có:** study-space khóa lesson phase sau tới khi qua checkpoint phase trước (`buildProgress`). → v1 chỉ **surface** checkpoint; **không build gating mới**.

## 3. Phạm vi
### Trong phạm vi
- **IS-1 (BE, KEYSTONE):** enrich item lịch với `courseSlug`, `phaseKey`, `moduleKey` (resolver đã có context; model + generator mang theo cho cả item học lẫn item ôn).
- **IS-2 (BE):** **checkpoint thành 1 loại nhiệm vụ** trong lịch — resolver surface `phase.checkpoint`/`course.checkpoint`; generator đặt item checkpoint ở **ranh giới phase** (sau lesson cuối của phase), mang `boundaryKey (=phaseKey|'course')` + `courseSlug` + `phaseKey`.
- **IS-3 (FE):** item lịch **bấm được** → deep-link đúng: học/ôn → `ADAPTIVE_PATH_STUDY(slug)?phase=&module=`; checkpoint → `+&checkpoint=1`.
- **IS-4 (FE):** **tick tiến độ** — đọc `useCourseProgress(courseSlug)`: bài học done theo `phase/module/lesson`; checkpoint done theo `boundaryKey ∈ passedCheckpoints`. Hiển thị trạng thái xong/khóa (khóa = study-space đang lock).
- **IS-5 (FE):** bề mặt **"Nhiệm vụ hôm nay"** — nâng khối "hôm nay" ở `/schedule` thành CTA chính (item bấm được + trạng thái xong).

### Ngoài phạm vi (ghi backlog)
- **M4-10:** checkpoint **chặn cứng nâng cao + vòng lùi-lịch remediation** (fail → gợi bài yếu → ôn lại → thi lại). *(Cổng cứng cơ bản đã có sẵn ở study-space; đây là phần fail-recovery.)*
- **Tick chính xác từng bài tập ôn:** không có tín hiệu per-refId (evaluator cần `lessonPathKey`) → **buổi ôn = mềm, best-effort, không tick chính xác** (khớp quyết định owner: ôn không ép, điểm thấp vẫn qua).
- **`talk`:** không có tín hiệu hoàn thành → item ôn talk không tick được.
- **Deep-link tới từng lesson cụ thể:** study-space chỉ nhận `module` (mở lesson đầu module, nav nội bộ) — v1 chấp nhận **mở ở mức module**.
- Job dồn trễ (M4-4 FR-008) · widget dashboard riêng.

## 4. Quyết định (owner 2026-08-05)
| # | Nội dung | Chốt |
|---|---|---|
| D1 | Nguồn tiến độ | ✅ Tái dùng `CourseEnrollment`/`useCourseProgress` — không model mới |
| D2 | Nơi học | ✅ Deep-link vào study-space đã có — không rebuild |
| D3 | Buổi ôn | ✅ **Mềm hoàn toàn** (đến ngày làm, có quyền bỏ, điểm thấp vẫn qua; không tick chính xác) |
| D4 | Checkpoint v1 | ✅ **Surface như 1 nhiệm vụ** (deep-link + tick qua `passedCheckpoints`); **không gating mới** (study-space đã khóa); remediation = M4-10 |
| D5 | Độ mịn deep-link | ✅ **Mức module** (giới hạn có sẵn) |

## 5. Acceptance criteria
1. **AC-1** (IS-1): mỗi item học/ôn trong `LessonSchedule` mang `courseSlug/phaseKey/moduleKey` đúng; hash regenerate không vỡ.
2. **AC-2** (IS-2): mỗi phase có `checkpoint` → có **1 item checkpoint** đặt sau lesson cuối của phase, mang `boundaryKey/courseSlug/phaseKey`; phase không checkpoint → không có.
3. **AC-3** (IS-3): bấm item học/ôn → mở đúng module trong study-space; bấm checkpoint → mở đúng bài vượt chặng.
4. **AC-4** (IS-4): item hiện **đã xong** khi bài/checkpoint tương ứng đã hoàn thành theo `useCourseProgress`; **khóa** khi study-space đang lock.
5. **AC-5** (IS-5): khối "Nhiệm vụ hôm nay" hiển thị việc của hôm nay, bấm được, có trạng thái.

## 6. Câu hỏi mở
- Đã chốt D1–D5. **Lưu ý owner (không chặn):** "cổng cứng cơ bản" đã có sẵn ở study-space (khóa phase sau) — nên trải nghiệm v1 thực tế **đã là "phải qua checkpoint mới đi tiếp"**; M4-10 chỉ thêm phần **gợi ý ôn lại khi fail**. Nếu muốn v1 **nới cả lock cơ bản** thì phải sửa `buildProgress` (không khuyến nghị — đang phục vụ study-space).
