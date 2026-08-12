> `plan.md` trong cùng thư mục là bản kế hoạch LỊCH SỬ (2026-06-08, mô tả `buildWeekSchedule` rule-based + `StudySchedule` LLM cũ — đã bị thay bằng generator thuần v1/v2 mô tả dưới đây). Đọc file này (`module-overview.md`) để biết trạng thái CODE THẬT hiện tại (2026-08-06).

# M4 — Lịch học cá nhân hoá (Personalized Study Schedule): Tổng quan module

Tài liệu tổng hợp toàn bộ M4, gộp các spec rời (`study-schedule-generation`, `schedule-module-finishing`, `schedule-config-projection`, `study-schedule-catchup`, `unified-study-flow`) đối chiếu với code đã ship trong `exe-api`/`exe-web`. Ưu tiên CODE khi spec và code lệch nhau (ghi rõ ở mục liên quan).

## 1. Tổng quan

Module biến **lộ trình đã gán (`StudentPath`) hoặc khóa đang học (`CourseEnrollment`)** + **quỹ thời gian người học khai báo** thành một **lịch học theo ngày, xác định (deterministic)**, rồi phục vụ 1 vòng đời:

```
Khai prefs (learningDows/reviewDow/focusDurationMin)
      → GET /schedule/suggest (xem trước, không cần lưu prefs)
      → POST /schedule/generate (sinh + persist LessonSchedule)
      → GET /schedule/today (xem việc hôm nay, tick qua route tiến độ khóa học có sẵn)
      → (trễ) GET /schedule/today trả missed → POST /schedule/postpone (đẩy lùi timeline)
      → (nghỉ ≥14 ngày) GET /schedule/today trả status:'dormant' (ngủ đông, không đỏ)
```

### Bản đồ M4-x

| Nhánh | Nội dung | Trạng thái |
|---|---|---|
| M4-0 | `estimatedDurationMin` trên `LessonSchema` + heuristic theo `type` (tiền đề tính thời lượng bài) | **ĐÃ SHIP** |
| M4-1 | Sinh lịch xác định (chunking, lọc band, ngày ôn cuối tuần, chọn thứ học/ôn, gợi ý cường độ, projection) | **ĐÃ SHIP** (v1+v2) |
| M4-2/M4-6 | Web: form khai prefs + xem lịch tháng | **ĐÃ SHIP** (`ScheduleContainer`, `SchedulePrefsForm`, `MonthCalendar`) |
| M4-3 | Ngày ôn tập cuối tuần (tổng hợp `exercises[]` trong tuần) | **ĐÃ SHIP** (là 1 phần của M4-1 v2, `IS-6`) |
| M4-4 | Lịch hôm nay + tick (tái dùng `CourseEnrollment`) + đẩy lùi khi trễ + ngủ đông + cảnh báo hạn mềm | **ĐÃ SHIP** (mới nhất, 2026-08-06) |
| M4-5 | Thông báo/nhắc học (cron) | **HOÃN** — chưa code; M4-4 đã chừa chỗ loại dormant khỏi vòng quét |
| M4-10 | Luồng học hợp nhất: item lịch bấm được (deep-link), checkpoint thành nhiệm vụ, tick qua `useCourseProgress`, "Nhiệm vụ hôm nay" là CTA chính | **ĐÃ SHIP** (BE Task 1-2 + FE Task 3-5 đều có trong code hiện tại) |

Ôn thông minh v3 (chọn-lọc nội dung ôn theo điểm/kỹ năng yếu), track thời gian học thực tế, đa lộ trình/người, seat-reclaim admin: chưa làm, xem mục 9.

## 2. Kiến trúc

3 lớp tách bạch — resolver (I/O đọc nguồn bài) → lõi thuần (generator/suggest/today-view/postpone) → controller mỏng ở biên (đọc DB, đọc đồng hồ, ghi response):

```
resolveLessonList(userId)                         ── I/O đọc StudentPath/CourseStructure/CourseEnrollment
  → LessonItem[] (kèm exercises[], courseSlug/phaseKey/moduleKey, checkpoint markers)
        │
        ▼
generateSchedule(lessons, pref, bands, asOfDate)   ── THUẦN (lesson-schedule.generator.js)
  → { days[], warning, warnings[], summary }
        │
        ├── suggestPlans(totalLessonMin, asOfDate, deadline?)   ── THUẦN, độc lập (schedule-suggest.js)
        ├── buildTodayView(sched, doneMaps, passedBoundaries,   ── THUẦN (today-view.js)
        │     lastActiveDate, todayLocal)
        └── postponeSchedule(userId, todayLocal)                ── I/O mỏng gọi lại generateSchedule (postpone.js)

Controllers (adaptive.controller.js) — biên duy nhất chạm đồng hồ/DB:
  asOfDate = req.body.asOfDate || new Date().toISOString().slice(0,10)
  todayLocal = todayLocalDate(tz, new Date())     ── local-date.js
```

**Tính xác định (NFR-1, xuyên suốt mọi slice):** không `Date.now()`/`Math.random()`/`new Date()` không tham số bên trong bất kỳ hàm thuần nào (`lesson-schedule.generator.js`, `schedule-suggest.js`, `today-view.js`). Mọi mốc thời gian (`asOfDate`, `todayLocal`, `lastActiveDate`) là tham số truyền vào; đồng hồ hệ thống và timezone chỉ được đọc ở controller (`adaptive.controller.js`) — cùng input luôn ra cùng output, gọi lại `postpone` 2 lần cho cùng `todayLocal` cho `days` giống hệt nhau (idempotent).

Đọc `CourseEnrollment` là **READ-ONLY** từ toàn bộ module `adaptive/` (không có endpoint tick riêng, không ghi lại completion) — xem `schedule/done-context.js`, `schedule/postpone.js`.

## 3. Data model

### `LessonSchedule` (`adaptive/lesson-schedule.model.js`)

1 document/user (unique index `userId`), ghi đè khi regenerate:

- `sourceType`: `'student_path' | 'course_structure'`.
- `asOfDate`, `generatedAt`: mốc sinh lịch (không phải `now()` lúc đọc).
- `inputHash`: MD5 sắp-xếp-key của `{lessons, pref, bands, asOfDate}` — dùng để `POST /schedule/generate` idempotent (input không đổi → trả lại doc cũ, không sinh lại).
- `warning`: `{overdue, suggestions[]}` (dùng lại cho cả cảnh báo hạn v1 lẫn cảnh báo hạn "mềm" của M4-4 — không field mới).
- `warnings[]`: mảng gộp `overdue | oversized | heavy_review` (schema `Mixed`).
- `summary`: `{totalLearningSessions, totalReviewSessions, totalLessonMin, estimatedFinishDate}`.
- `days[]` (`ScheduleDaySchema`): `dateLocal` ('YYYY-MM-DD'), `loadMin`, `oversized`, `isReview`, `items[]`.
- `items[]` (`LessonItemSchema`): `kind` (`'lesson' | 'review' | 'checkpoint'`, default `'lesson'` — back-compat cho doc cũ trước khi field này tồn tại), `courseSlug/phaseKey/moduleKey/lessonKey` (deep-link + progress-lookup context — keystone của M4-10), `refId` (item ôn), `boundaryKey` (item checkpoint: `phaseKey` hoặc chuỗi `'course'`), `type`, `durationMin`.

M4-4 **không thêm field nào** trên `LessonSchedule` — tái dùng nguyên `warning`/`warnings`/`days` đã có.

### Nguồn hoàn thành = `CourseEnrollment` (không có "done store" riêng)

`course-content.progress.model.js`: `CourseEnrollment.lessons` là Map `"<phaseKey>/<moduleKey>/<lessonKey>" → {status, completedAt}` (`status ∈ LESSON_PROGRESS_STATUSES`). M4-4 join lịch ↔ hoàn thành qua đúng key này (`schedule/today-view.js#isDone`, `schedule/done-context.js`). Checkpoint "done" = `getPassedBoundaries(userId, courseId)` (`course-content.progress.service.js`) — chỉ đọc, không phần nào của `adaptive/` ghi vào `CourseEnrollment`/`CheckpointAttempt`.

### `User.profile.studyPlan` + `streak.lastActiveDate` (`user/user.model.js`)

- `learningDows: [Number]` (default `null`), `reviewDow: Number` (1-7), `focusDurationMin: Number` (min 5 trong schema Mongoose — Joi ở tầng validate chốt 60-240, xem mục 4/6), `deadline`, cộng các field cũ (`timeSlots`, `daysPerWeek`, `weekendStudy`, `commitment`) **giữ nguyên** cho `study-schedule.service.js` (LLM cũ, coexist, route đang tắt).
- `streak.lastActiveDate: Date` — tín hiệu "ngủ đông" của M4-4 (`daysBetween(lastActiveDate, todayLocal) >= 14` → `status:'dormant'`). Không lưu cờ dormant riêng, tính lúc đọc.

## 4. API contract

Tất cả mount `/api/adaptive`, gate `verifyToken → withTenant`, self-service (`req.user.id`), không `verifyPermission`.

| Method + path | Request | Response |
|---|---|---|
| `PUT /adaptive/schedule/prefs` | body `{ studyPlan: { learningDows[2..7], reviewDow, focusDurationMin[60..240], deadline? } }` — Joi reject nếu `reviewDow ≤ max(learningDows)` | 200 `{ studyPlan }` (merge vào `profile.studyPlan` hiện có, không ghi đè `timeSlots` v.v.) |
| `GET /adaptive/schedule/prefs` | — | 200 `{ studyPlan }` hoặc `{ studyPlan: null }` |
| `POST /adaptive/schedule/generate` | body `{ asOfDate?: 'YYYY-MM-DD' }` | 200 `{ schedule, warning, warnings, summary }`; chưa khai đủ prefs → `{ needsPrefs: true }`; chưa có nguồn bài → 409 `SCHEDULE_NO_SOURCE` |
| `GET /adaptive/schedule/suggest` | query `{ asOfDate?, deadline? }` — không cần prefs đã lưu | 200: không deadline dùng được → `{ presets[], totalLessonMin, totalReviewMin }`; có deadline dùng được → `{ requiredWeeklyMin, tooTight, fallbackPlan, totalLessonMin, totalReviewMin }` |
| `GET /adaptive/schedule/today` | query `{ tz? }` (mặc định `Asia/Saigon`) | 200 active: `{ status:'active', date, items[{...item, done}], progress{doneCount,totalCount,remainingMin}, missed{count,oldestDate,days[]}\|null, warning }`; 200 dormant: `{ status:'dormant', date, resume:true, message }` |
| `POST /adaptive/schedule/postpone` | body `{}` (tz theo query/body, mặc định `Asia/Saigon`) | 200 `{ schedule, warning }`; đủ điều kiện nhưng thiếu prefs → `{ needsPrefs: true }`; chưa có `LessonSchedule` → 404 `SCHEDULE_NOT_FOUND` |

Các route lịch tuần LLM cũ (`GET /schedule`, `POST /schedule/prefs`, `POST /schedule/regenerate`) đang **comment-out** trong `adaptive.routes.js` (đóng tạm 2026-07-19, trả `ROUTE_NOT_FOUND` nếu gọi) — service/schema vẫn giữ nguyên cho coexist, không phải phần đang hoạt động của M4.

Mã lỗi liên quan (`constants/error-codes.js`): `SCHEDULE_NO_SOURCE` (chưa có `StudentPath`/`CourseEnrollment` để lấy bài), `SCHEDULE_NOT_FOUND` (postpone gọi khi chưa từng generate).

## 5. Thuật toán chính

### Sinh lịch — `generateSchedule` (`schedule/lesson-schedule.generator.js`)

Pipeline: `filterByBand` → `chunkLessons` → duyệt lịch ngày-qua-ngày đặt bucket vào `learningDows`, chèn ngày ôn ở `reviewDow` cuối mỗi tuần ISO (Hai→CN) → `overdueWarning` + `buildWarnings` + `buildSummary`.

- **Lọc band** (`filterByBand`): bỏ bài mà `bands[skill] ≥ phaseBand` (đã vượt trình); `bands=null` hoặc thiếu thông tin → giữ nguyên (không lọc nhầm). Checkpoint marker không bao giờ bị lọc.
- **Chunking** (`chunkLessons`): không cắt đôi bài; bài `durationMin > cap` độc chiếm ngày, cờ `oversized`; checkpoint marker luôn độc chiếm 1 ngày riêng (`CHECKPOINT_DURATION_MIN = 30`, không tính oversized).
- **Ngày ôn cuối tuần** (`buildReviewDay`): tổng hợp `exercises[]` của mọi bài đã xếp trong tuần ISO hiện tại thành các item `kind:'review'`, thời lượng theo `exerciseDurationOf` (không chặn trần `minutesPerDay`, không tính oversized); tuần không có exercise nào → không có ngày ôn.
- **Checkpoint là 1 loại nhiệm vụ** (M4-10 IS-2, đã ship): resolver chèn marker `kind:'checkpoint'` sau bài cuối mỗi phase có `phase.checkpoint`, và cuối khóa nếu có `course.checkpoint`; `boundaryKey` = `phase.key` hoặc `'course'`.
- **Heuristic thời lượng** (`schedule/lesson-duration.js`): bài học ưu tiên `estimatedDurationMin`, thiếu thì fallback theo `type` (`vocab_flashcard/listening_quiz` 10 · `theory/video_lesson/ipa_exercise` 15 · `grammar_quiz/reading_quiz` 20 · `speaking_task` 25 · `writing_task` 60; type lạ → 15). Exercise ôn: `quiz` 8 · `flashcard` 5 · `ipa` 5 · `talk` 10 (type lạ → 5).
- **Tuần ISO**: `isoDow` chuyển `getUTCDay()` (0=CN) sang 1..7 (CN=7), tuần bắt đầu Thứ Hai.
- **Cảnh báo**: `overdueWarning` — ngày HỌC cuối cùng (bỏ qua ngày ôn) vượt `deadline` → `{overdue:true, suggestions: 2 mục}`. `buildWarnings` gộp thêm `oversized` (danh sách ngày) và `heavy_review` (ngày ôn có `loadMin > 2×minutesPerDay`).

### Gợi ý cường độ — `suggestPlans` (`schedule/schedule-suggest.js`)

Không cần prefs đã lưu, chỉ cần tổng phút bài học:
- **Không có deadline dùng được** (chuỗi không phải `YYYY-MM-DD` hợp lệ, hoặc không sau `asOfDate` → coi như không dùng được): trả 3 preset cố định — `thong_tha` (2 ngày×60′) · `can_bang` (4×60′, khuyến nghị) · `cap_toc` (5×90′) — mỗi preset kèm `estimatedFinishDate` (`projectFinish`: số tuần = `ceil(totalLessonMin / weeklyCapacity)`, cộng theo bội số 7 ngày).
- **Có deadline dùng được**: `requiredWeeklyMin = ceil(totalLessonMin / weeksBetween(asOfDate, deadline))`; vượt trần lành mạnh `HEALTHY_MAX_WEEKLY_MIN` (= 5×90 = 450′/tuần, lấy từ chính preset `cap_toc`) → `tooTight:true` kèm `fallbackPlan` (preset `cap_toc` + ngày xong thực tế).

### Việc hôm nay — `buildTodayView` (`schedule/today-view.js`)

- **Ngủ đông trước tiên**: `daysBetween(lastActiveDate, todayLocal) >= DORMANT_INACTIVE_DAYS (14)` → trả ngay `{status:'dormant', resume:true, message}`, **không** kèm `items`/`missed` (đỡ "đỏ" gây nản khi quay lại sau kỳ nghỉ dài).
- **Join done** (`isDone`): `kind:'review'` → luôn `null` (chỉ hiển thị, không tính tiến độ/missed); `kind:'checkpoint'` → `passedBoundaries.has(boundaryKey)`; bài học → `status ∈ {'completed','mastered'}` từ `doneMaps` (map theo `courseSlug` → map `"phaseKey/moduleKey/lessonKey"` → status).
- **Tiến độ**: `counted` = item có `done !== null` (loại ngày ôn); `progress = {doneCount, totalCount, remainingMin}`.
- **Missed (đỏ)**: ngày `< todayLocal` còn item **học** (không phải review/checkpoint) chưa done → gộp vào `missed.days[]`, `missed.oldestDate` = ngày cũ nhất (do `sched.days` luôn lưu theo thứ tự thời gian).
- **Cảnh báo hạn mềm** (`deadlineWarn`): `estimatedFinishDate > deadline` → `{overdue:true, suggestions:['Giãn hạn tới …', 'Tăng số phút học mỗi ngày để kịp …']}`; không deadline hoặc kịp → `{overdue:false, suggestions:[]}`. Chỉ cảnh báo, không chặn — người học tự sửa qua `PUT /schedule/prefs`.

### Đẩy lùi lịch khi trễ — `postponeSchedule` (`schedule/postpone.js`)

**Không phải thuật toán mới** — gọi lại nguyên `generateSchedule` trên tập bài **chưa done**, `asOfDate = todayLocal`, cường độ = `focusDurationMin` **nguyên bản** (không nhân 1.5):

1. Guard trước khi đọc/ghi gì: `isUsablePlan(plan)` — thiếu `focusDurationMin`/`learningDows`/`reviewDow` → trả `{needsPrefs:true}`, không đụng DB.
2. `loadDoneContext` (đọc `CourseEnrollment` + `getPassedBoundaries`, chỉ đọc) → `doneMaps`, `passedBoundaries`.
3. `remaining = resolveLessonList(userId).items.filter(item => isDone(item,...) !== true)` — thứ tự lộ trình gốc, chỉ loại bài **đã done** (không skip bất kỳ bài nào khác, kể cả bài chưa từng tới lượt).
4. `generateSchedule(remaining, pref, bands, todayLocal)` → `days`, `summary` mới.
5. **Giữ lịch sử**: ngày `< todayLocal` cũ được giữ lại trong doc nhưng lọc chỉ còn item **đã done** (bài chưa-done của ngày cũ đã nằm trong `remaining` và được xếp lại ở phần mới → không trùng lặp).
6. `sched.days = pastKept.concat(days)`; cập nhật `summary`, `warning = deadlineWarn(...)`, `asOfDate`/`generatedAt = todayLocal`; save.

Tất định: cùng `(remaining, prefs, todayLocal)` → cùng `days`, gọi 2 lần liên tiếp cho kết quả giống hệt (không có khóa idempotent riêng vì bản thân thuật toán đã tất định).

## 6. M4-4 — 3 deviation có chủ đích so với PRD-89 (owner đã chốt 2026-08-06)

| PRD-89 ghi | M4-4 làm (code thật) | Lý do (owner) |
|---|---|---|
| "job đầu ngày dồn ≤150%" | **Đẩy lùi timeline thủ công** (`postponeSchedule`), giữ nguyên cường độ `focusDurationMin`, bỏ hẳn trần 150% | Nhồi 150% dễ quá tải/nản; đẩy lùi nhẹ nhàng hơn, người học chủ động bấm |
| "bỏ phần lỡ" (1 trong 3 phương án đề xuất) | **Bỏ hẳn phương án skip** — `postponeSchedule` chỉ loại bài đã done khỏi `remaining`, không loại bài nào khác | App học tích lũy, không cho bỏ nội dung |
| "vượt kéo dài → hỏi người học (giãn hạn/tăng phút/bỏ)" | **Cảnh báo mềm + gợi ý** (`deadlineWarn` → `warning.overdue` + `suggestions[]`), không chặn, không prompt, không nút bỏ | Đã bỏ skip nên chỉ còn 2 lối hợp lý; không ép luồng người học |

**Bổ sung ngoài PRD-89 — ngủ đông (dormant):** nếu bất hoạt ≥ `DORMANT_INACTIVE_DAYS` (14 ngày, hằng số tại `today-view.js`, chưa tinh chỉnh) → `today` trả `status:'dormant'` thay vì danh sách ngày đỏ; giữ nguyên mọi data, không ghi `CourseEnrollment`, không auto-đuổi khỏi khóa. Cơ sở edtech: Duolingo/Coursera/Khan/Babbel đều ngủ đông + giữ data + nhắc thưa dần, không trục xuất vì nghỉ; thu hồi ghế B2B (nếu cần) là thao tác thủ công riêng của admin center, ngoài phạm vi M4-4.

## 7. Web (exe-web)

Trang `/schedule` (`app/(main)/schedule/page.tsx`) render `ScheduleContainer` (`components/features/schedule/`):

- **`SchedulePrefsForm.tsx`** + `SchedulePrefsDowToggle.tsx` + `SchedulePrefsSuggestPanel.tsx`: khai `learningDows`/`reviewDow`/`focusDurationMin`, có preset quick-fill (map `learningDays → dows` dàn đều tuần) và panel gọi `GET /schedule/suggest` xem trước ngày xong trước khi lưu. Prefill từ `useStudyPrefs()` khi bấm "Chỉnh sửa lịch học" (`values` thay vì `defaultValues` trong RHF để đồng bộ lại khi prefs về sau khi form đã mount).
- **`SchedulePrefsSummary.tsx`**: thẻ tóm tắt prefs đang lưu (thứ học/ôn, phút/ngày, hạn).
- **`MonthCalendar.tsx`**: lưới tháng 6 tuần (CN đầu tuần), chấm số bài/ngày ôn/cờ oversized/mốc chặng; nhận `missedDates` (từ `today.missed`) để tô ngày lỡ.
- **`DayQuestPanel.tsx`**: nhiệm vụ của ngày đang chọn — item bấm được qua `scheduleItemHref()` (deep-link `ROUTES.ADAPTIVE_PATH_STUDY(slug)?phase=&module=`, thêm `&checkpoint=1` cho checkpoint; `talk` và item thiếu context → không bấm được), trạng thái done/locked/pending qua `scheduleItemStatus()` (đọc `useCourseProgress` qua `useScheduleProgress`), badge "Vượt chặng" cho checkpoint.
- **`ScheduleCatchupBanner.tsx`** (M4-4): dormant → banner "Tạm dừng lịch học" (điềm tĩnh, không cảnh báo); có `missed` → banner cam + nút "Dời lịch ra sau" gọi `usePostponeSchedule()`; không có gì bất thường → không render.
- **`ScheduleWarnings.tsx`**: hiển thị `schedule.warnings[]` (overdue/oversized/heavy_review).

**Service/hook wiring** (`services/schedule.service.ts` → `hooks/use-schedule.ts`, query keys `QUERY_KEYS.schedule.*`): `useSchedule` (query, `generateSchedule` — idempotent nên gọi lại an toàn ở mọi lần vào trang), `useStudyPrefs`/`useSaveStudyPrefs`, `useScheduleSuggest`, `useScheduleToday` (key theo `Intl.DateTimeFormat().resolvedOptions().timeZone`, fallback `Asia/Saigon`), `usePostponeSchedule` (invalidate `schedule.all` sau khi thành công), `useScheduleProgress(days)` (fan-out `useQueries` theo `courseSlug` phân biệt trong lịch, phục vụ tick M4-10).

## 8. Kiểm thử

**exe-api** (`src/__tests__/`, Jest):

| Lớp | File | Số case |
|---|---|---|
| Generator thuần | `schedule-generator.test.js` | 30 |
| Suggest thuần | `schedule-suggest.test.js` | 9 |
| Today-view thuần | `today-view.test.js` | 27 |
| Postpone (unit) | `schedule-postpone.test.js` | 12 |
| API generate | `schedule-generate.api.test.js` | 7 |
| API suggest | `schedule-suggest.api.test.js` | 6 |
| API prefs (get/save) | `schedule-prefs-get.api.test.js` + `schedule-prefs-save.api.test.js` | 3 + 7 |
| API today/postpone | `schedule-today.api.test.js` | 9 |
| E2E catch-up (real HTTP stack) | `schedule-catchup-flow.e2e.test.js` | 1 (kịch bản dài: generate → today → tick qua route tiến độ thật → missed tích lũy → postpone → missed sạch + lịch lại → dormant) |
| Model/hash | `lesson-schedule.model.test.js` | 6 |
| Coexist (không hồi quy) | `adaptive-week-schedule.test.js`, `study-schedule.service.test.js` | 3 + 18 |

**exe-web** (Vitest component/unit + Playwright e2e):

| Lớp | File | Số case |
|---|---|---|
| Service | `schedule.service.test.ts` | 7 |
| Hook | `use-schedule.test.ts` | 8 |
| Utils (deep-link, status, date, format) | `schedule.utils.test.ts` | 26 |
| Component | `ScheduleContainer.test.tsx` | 9 |
| Component | `DayQuestPanel.test.tsx` | 9 |
| Component | `SchedulePrefsForm.test.tsx` | 14 |
| Component | `ScheduleCatchupBanner.test.tsx` | 8 |
| Component | `MonthCalendar.test.tsx` | 3 |
| Browser e2e (Playwright) | `e2e/schedule-catchup.spec.ts` | 7 |

Guard đã verify trong task list M4-4 (grep, không phải test riêng): `today-view.js`/`postpone.js` không chứa `Date.now|Math.random|new Date()` không tham số; `CourseEnrollment` trong `modules/adaptive/` chỉ bị đọc (`findOne`/`.get(`/`.lean(`), không `.set(`/`.save(`; `postpone.js` không chứa `1.5|150|150%` (không nhồi cường độ).

## 9. Deferred / Next

- **M4-5 (thông báo/nhắc học)**: chưa code. Khi làm, cron phải **loại dormant khỏi vòng quét** bằng đúng công thức `daysBetween(lastActiveDate, todayLocal) >= 14` mà `today-view.js` đang dùng lúc đọc — đây mới là chỗ tiết kiệm chi phí thật (M4-4 lazy-on-read vốn đã tốn 0 cho user nghỉ).
- **Ôn thông minh v3**: chọn-lọc nội dung buổi ôn theo bài sai/kỹ năng yếu/giãn ngắt quãng kiểu SM-2 — cần tầng tiến độ + điểm số theo thời gian (chưa có, ôn hiện tại là "ôn tĩnh": gom hết exercises trong tuần).
- **Seat-reclaim admin (B2B)**: thu hồi ghế học viên ngủ đông quá lâu — nút thủ công ở admin center, module riêng, ngoài M4-4.
- **Track thời gian học thực tế**: chưa có field nào lưu thời gian học thực — ngoài phạm vi hiện tại.
- **`unified-study-flow` (M4-10) phần fail-recovery**: checkpoint hiện chỉ *surface* như 1 nhiệm vụ (đã ship); vòng lùi-lịch remediation khi fail checkpoint (gợi ôn lại → thi lại) **chưa làm** — cổng cứng cơ bản (khóa phase sau tới khi qua checkpoint) đã có sẵn ở tầng study-space (`buildProgress`), M4-10 v1 không đụng vào.
- **Parked minors** (biết, chưa vá, không chặn release):
  - `inputHash` không được tính lại sau `postponeSchedule` — gọi lại `POST /schedule/generate` ngay sau khi postpone có thể coi input "đã đổi" chậm hơn thực tế (lịch đã bị postpone ghi đè `days` nhưng `inputHash` vẫn là hash cũ từ lần `generate` trước).
  - `loadBands()` bị lặp lại y hệt giữa `adaptive.controller.js#generateLessonSchedule` và `schedule/postpone.js` (đọc `StudentModel.skill` map 5 kỹ năng) — chưa rút ra hàm dùng chung (DRY).
  - `isUsablePlan()`/đọc `profile.studyPlan` cũng lặp logic tương tự giữa controller và `postpone.js`.
  - Legacy `study-schedule.service.js#buildFallbackDays` (dịch vụ `StudySchedule` LLM cũ, coexist) vẫn dùng `focusDurationMin || 25` không có sàn — nhưng KHÔNG gọi `generateSchedule`/guard mới nên nằm ngoài phạm vi siết sàn ở trên; dọn khi retire service cũ.

## 10. Câu hỏi mở / Quyết định

- ✅ **CHỐT (owner 2026-08-06) — tín hiệu bất hoạt = `streak.lastActiveDate`** (bump khi hoàn thành bài qua `recordLesson`), KHÔNG dùng `lastLoginAt`. Đúng tinh thần "dormant = không HỌC". *Edge đã biết:* hoạt-động-lẻ không đi qua `recordLesson` (flashcard SRS toàn cục, quiz/assessment standalone) hiện không bump `lastActiveDate` → M4-5 cân nhắc bump thêm ở các activity đó nếu cần.
- ✅ **ĐÃ SIẾT (2026-08-06) — sàn `focusDurationMin` = 60**: nâng ở Mongoose schema (`user.model.js`, min 5→60, null vẫn hợp lệ) + guard `postpone.isUsablePlan`/`controller.generateLessonSchedule` (`<60` hoặc null → `needsPrefs`, không sinh lịch bệnh hoạn). Kèm migration `src/scripts/backfill-clamp-focus-duration-min.js` (dry-run mặc định, `--apply` kẹp legacy `<60→60`). Giải quyết mismatch schema(5) vs Joi(60). **Thứ tự deploy:** chạy migration `--apply` TRƯỚC khi deploy commit schema+guard.
- ⏳ **Hoãn tinh chỉnh (owner OK 2026-08-06)** — không chặn release, chỉnh sau khi có số liệu thật:
  - `DORMANT_INACTIVE_DAYS = 14` (hằng số `today-view.js`) — số đề xuất.
  - Ngưỡng `tooTight` trong `suggestPlans` (~450′/tuần) — số đề xuất (OQ-8 spec generation).
