<!-- Tiếng Việt — data-model Phase 3 (tiến độ khóa). -->
# Data model: Tiến độ khóa (Phase 3)

## `course_enrollments` (mới) — `modules/course-content/course-content.progress.model.js`

1 doc / **user × course** (Q-P3.1 nhúng tiến độ Bài, ít join, đọc nhanh cho UI).

```js
LessonProgressSchema { _id:false }
  status:      enum ['in_progress','completed','mastered']  // absent = not_started (không lưu)
  viewedAt:    Date
  completedAt: Date, default null

CourseEnrollmentSchema { timestamps:true, collection:'course_enrollments' }
  userId:   ObjectId ref User,            required, index
  courseId: ObjectId ref CourseStructure, required, index
  status:   enum ['active','completed'],  default 'active'
  lessons:  Map<String, LessonProgressSchema>   // key = "phaseKey/moduleKey/lessonKey"
  // timestamps: createdAt = enrolledAt
index unique { userId, courseId }
```

**Định danh Bài (quan trọng):** Bài `key` chỉ unique **trong Chuyên đề**; mọi node cây để `_id:false` (không
ObjectId). ⇒ khóa tiến độ theo **path** `"<phaseKey>/<moduleKey>/<lessonKey>"`. Dùng `/` làm phân tách; **không**
dùng `.` (Mongo Map cấm key chứa dấu chấm). Giả định: key không chứa `/` (key nhập từ Excel, dạng slug).

**Không lưu `unlocked`/rollup** — tính động khi đọc từ `lessons` + thứ tự cây (nguồn sự thật = completion).

## Nguồn tín hiệu hoàn thành (không nhân bản)
- **ipa**: đọc `ipa_progress` (`{userId, lessonId}`) — `lessonId` lấy bằng resolve `refId`(code)→`_id` qua
  `IpaLesson.find({code:{$in}, status:'published'})`. Điểm phát âm dùng chung mọi khóa (Q-P3.4).
- **talk**: không có persistence → **bỏ qua** khi xét hoàn thành (Q-P3.5).

## `User.streak` (đã có, thêm writer) — `modules/user/streak.service.js`
Subdoc `{ current, longest, lastActiveDate }` đã có sẵn, **chưa có nơi ghi**. `touchStreak(userId)` (Q-P3.8):
- mốc ngày theo UTC. `lastActiveDate == hôm nay` ⇒ no-op. `== hôm qua` ⇒ `current++`. khác ⇒ `current=1`.
- `longest = max(longest, current)`; `lastActiveDate = hôm nay`. Gọi khi ghi nhận học Bài (§P3).
