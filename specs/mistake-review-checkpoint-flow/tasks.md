# Tasks: Sổ lỗi sai — Ôn neo checkpoint (Phase 3+)

> Tách theo phase §8 của `spec.md`. Chỉ 3a/3b độc lập data; 3c/3d cần checkpoint config + phối hợp M4-9.

## Phase 3a — Nhận thức + tiến trình (làm ngay, dựa P1/P2) ✅
- [x] Thêm trường tiến trình vào payload `/review/items`: `dueState` (`due|later|graduated`), `stepsToMaster` (= `INTERVALS.length + 1 − box`).
- [x] UI: box dots / "còn N lần để thạo" / đến hạn / còn X ngày / đã thạo (giữ nhãn "chưa hỗ trợ ôn" cho testlet).
- [x] Test: payload tiến trình đúng (due/later/graduated + stepsToMaster).
> Bỏ config `REVIEW_MODE_BY_SUBSKILL` — §7.1: mọi lỗi objective đều exact-repeat, chỉ cần config khi bật "câu biến tấu".

## Phase 3b — Facade hub + surface từ vựng ✅
- [x] Endpoint `GET /review/overview` gom đếm `{ vocab: {weak}, grammar: {due, total} }` (facade tới `getWeakCourseVocabulary` + review_schedule; không lưu).
- [x] UI hub: 2 thẻ active (từ vựng → link `/flashcards`; ngữ pháp → "Ôn ngay" vào session) + 2 thẻ "để xem xét" (kỹ năng, IPA).
- [x] Test: overview đếm đúng (grammar due standalone, vocab từ SRS); render counts + parked labels + CTA.

## Phase 3c — Buổi ôn neo checkpoint (KHUNG backend ✅ · trigger/UI còn lại)
- [x] Chặng context trên `review_schedule`: `courseId` + `phaseKey` (bắt lazy ở `enqueueMistakes` từ `quiz_attempts` qua `refId`; `lessonPathKey.split('/')[0]`).
- [x] Service `getCheckpointReviewSession(userId, {courseId, phaseKey, limit})`: rút lỗi phạm vi chặng, size-adaptive (`limit` 0 = tất cả). `getReviewSessionItems` uỷ thác vào nó (DRY).
- [x] Endpoint `GET /review/checkpoint?courseId=&phaseKey=`.
- [x] Test: bắt phaseKey đúng; session scope theo chặng; no-scope = cả bank.
- [x] **Trigger** (neo checkpoint, không đụng ôn-tuần): generator (pure) emit marker `checkpoint_review` ngay trước checkpoint có `reviewPolicy`; resolver mang `reviewPolicy` lên marker; today-view coi `checkpoint_review` như display-only; controller `getToday` **late-bind** lỗi chặng qua `getCheckpointReviewSession` (+ honour `minItems`). Ôn-tuần `buildReviewDay` giữ nguyên (backward-compat).
- [x] UI buổi ôn theo chặng: `DayQuestPanel` render item `checkpoint_review` (chip "Ôn trước chặng" + mô tả định tính, KHÔNG hứa số câu — khung persisted không mang lỗi, ngày tương lai chưa biết số; số thật tính live khi vào `/practice/review`) + link; `schedule.utils` href/status/label; type `ScheduleItemKind`.
- [x] `reviewPolicy` đọc LIVE từ `CourseStructure` trong `resolveFromStudentPath` (gate `quizCode`/`passThreshold` vẫn đóng băng từ snapshot; snapshot schema không mang `reviewPolicy`) → admin bật ôn áp dụng ngay cho học viên đang học. `normalizeReviewPolicy` phẳng Mongoose subdoc tránh crash circular-ref ở `computeInputHash`.
- [x] `MonthCalendar` đánh dấu ngày checkpoint (badge "KT") — trước đó ngày chỉ-có-checkpoint hiện như ô trống vì bị loại khỏi `count`.
> Kế hoạch: `exe-api/docs/superpowers/plans/2026-08-13-checkpoint-anchored-review.md`. Progressive reveal = tự động (§7.2). "Dãn nhiều ngày" (daysBefore>1) = deferred; hiện emit 1 buổi ngay trước checkpoint.

## Phase 3d — Admin config
- [x] Mở rộng `CheckpointRefSchema`: `reviewPolicy { daysBefore, minItems }` (default null → schedule không đổi).
- [ ] exe-admin: form cấu hình checkpoint + review policy theo chặng.
- [x] **Thay hẳn ôn-tuần**: generator bỏ `buildReviewDay`/`reviewDow`; prefs bỏ picker "Ngày ôn tập cuối tuần" + "Thứ ôn"; reviewDow → optional/deprecated. Review giờ thuần checkpoint-anchored + sổ luôn mở.
- [ ] "Dãn nhiều ngày" (daysBefore>1) — deferred.

## Parked (xem xét sau)
- [ ] Engine "câu biến tấu" (concept-mode) — bật khi ngân hàng câu đủ lớn.
- [ ] "Làm thử như thi" trước checkpoint (trộn câu cả chặng, tính giờ).
- [ ] Gộp IPA vào hub.
- [ ] Writing/Speaking (chấm AI llm).

## Điều kiện tiên quyết (không phải task code)
- [ ] Cấu hình checkpoint cho khóa published (hiện chỉ khóa `ready` có).
- [ ] Nhập exercises thực tế (hiện gần rỗng) để có lỗi mà ôn.
