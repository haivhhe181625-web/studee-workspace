<!-- Tiếng Việt — thiết kế Phase 3 (tiến độ/mastery/unlock). Không quá chi tiết, tiện code. -->
# Thiết kế: Tiến độ khóa (Phase 3)

- **Ranh giới:** `exe-api` (module `course-content` + `user`), `exe-web` (feature `courses`). Không đụng exe-admin.
- **Quyết định:** đã chốt toàn bộ §8 `docs/plan-phase3-progress-mastery.md` (mọi đề xuất OK).
- **Contract:** `contracts/course-progress.md` · **Data:** `data-model.md`.

## 1. Bắc cầu, không đo lại
Bài học không tự chấm — hoàn thành **suy ra** từ engine: ipa đọc `ipa_progress`; talk bỏ qua. Course chỉ *tổng
hợp* + gate + streak. Không hệ chấm điểm mới (→ Phase 4).

## 2. File

```
exe-api (mới/sửa trong modules/):
  course-content/course-content.progress.model.js    CourseEnrollment (data-model.md)
  course-content/course-content.progress.service.js  enroll, getProgress, recordLesson
  course-content/course-content.controller.js        + enrollCourse, getCourseProgress, recordLessonProgress
  course-content/course-content.routes.js            + POST /:slug/enroll, GET /:slug/progress, POST /:slug/lessons/progress
  course-content/course-content.service.js           changeStatus: guard published→ready khi có enrollment (Q-P3.2)
  user/streak.service.js                             touchStreak(userId)  (K5)

exe-web (feature courses):
  services/course.service.ts   + enrollCourse, getCourseProgress, recordLessonProgress
  hooks/use-courses.ts         + useCourseProgress, useEnrollCourse, useRecordLesson (mutations invalidate progress)
  types/course.types.ts        + CourseProgress, LessonProgress
  components/features/courses/  CourseCard (badge enrolled), CourseDetailContainer (khoá/mở + %), LessonViewer (mark-done + gate + "Học tiếp")
```

## 3. Thuật toán (service)

- **pathKey(p,m,l)** = `` `${p}/${m}/${l}` ``. **flat(course)** = duyệt phases→modules→lessons theo `order` → mảng
  `{ pathKey, phaseKey, exercises }`.
- **enroll**: tìm course published (404) → `updateOne({userId,courseId},{ $setOnInsert }, {upsert})` → getProgress.
- **getProgress**: course published + enrollment. Chưa enroll ⇒ `{enrolled:false}`. Có ⇒ build map `lessons` từ
  `flat` + `enrollment.lessons`; `unlocked[i] = i==0 || flat[i-1].status ∈ {completed,mastered}`; rollup `percent`,
  `phasePercent`, `nextLesson` = Bài unlocked đầu tiên chưa completed.
- **recordLesson**: 404 nếu pathKey không có trong `flat`; 409 nếu chưa enroll. set `viewedAt` + status≥in_progress.
  Đánh giá: ipaEx = exercises type ipa; nếu có → map code→id (IpaLesson published) → `ipa_progress` theo
  `{userId, lessonId:{$in}}`; `completed` ⇔ mọi ipaEx có status∈{completed,mastered}; `mastered` ⇔ mọi
  `mastered`. Không ipaEx ⇒ completed. Ghi `completedAt` khi mới completed. Nếu mọi Bài completed ⇒
  `enrollment.status='completed'`. `touchStreak(userId)`. Trả getProgress.

## 4. Guard Q-P3.2 (course-content.service.changeStatus)
Trước khi đổi: nếu `doc.status==='published' && to==='ready'` và `CourseEnrollment.exists({courseId:doc._id})`
⇒ `ApiError(409, 'Không thể gỡ xuất bản: đã có học viên ghi danh', COURSE_LOCKED)`.

## 5. exe-web
- **Chưa enroll**: giữ Phase 2 (xem tự do, mọi Bài mở), nút **"Ghi danh"**. **Đã enroll**: overlay tiến độ — badge
  status/khoá trên LessonRow (khoá ⇒ disable link), thanh % ở header khóa + mỗi Chặng, nút **"Học tiếp"** →
  `nextLesson`. **LessonViewer**: nếu Bài `locked` ⇒ chặn (điều hướng về chi tiết); nút **"Đánh dấu đã học"** gọi
  §P3; tự gọi §P3 khi mở Bài (mark viewed) để cập nhật in_progress.
- Gate chỉ khi enrolled. Query `useCourseProgress(slug)` (enabled khi đã đăng nhập). Mutation enroll/record →
  `invalidate QUERY_KEYS.courses.progress(slug)`.

## 6. Ngoài phạm vi (→ Phase 4/5): chấm talk + engine Nghe/Đọc/Viết/quiz; admin xem tiến độ; XP/badge ngoài ipa.
