# Tasks: Lịch hôm nay + đẩy lùi lịch khi trễ (M4-4)

> Ngã rẽ nghiệp vụ đã CHỐT (owner 2026-08-06, spec §2). Còn OQ-1/OQ-2 (spec §8) cho Technical Review — không chặn khởi động; chốt OQ-1 trước Task 3, OQ-2 khi implement Task 3.

**Spec:** `spec.md` · **Design:** `design.md`
**Repo/branch:** `exe-api`, branch `feature/study-schedule-catchup` **base lên nhánh generator** (`feature/study-schedule-generation` / `…-review-and-planning`).
**Lưu ý chạy lệnh:** mọi `npx jest` trong `services/api` (`cd services/api`). Test ở `src/__tests__/`; helper `__tests__/helpers/{db,app}.js`. Chạy 1 file: `npx jest <pattern> -i`.

---

## Global Constraints

Bám nguyên văn (đối chiếu code thật 2026-08-06):

1. **NFR-1 — lõi thuần xác định.** `buildTodayView`, `deadlineWarn` + việc gọi `generateSchedule`: **KHÔNG** `Date.now()`/`Math.random()`/`new Date()` không-tham-số. `todayLocal`/`asOfDate` là **tham số**; TZ + ngày-local chỉ ở **controller (biên)**.
2. **`CourseEnrollment` READ-ONLY từ adaptive.** KHÔNG store hoàn-thành mới; KHÔNG ghi vào `CourseEnrollment` từ module adaptive. Join theo `"<phaseKey>/<moduleKey>/<lessonKey>"`.
3. **Đẩy lùi = gọi lại `generateSchedule`** (design §6) trên bài chưa-done + cường độ **thường** (`focusDurationMin`, KHÔNG ×1.5) + `asOfDate=todayLocal`. KHÔNG thuật toán đẩy-item riêng, KHÔNG trần 150%, KHÔNG logic skip.
4. **KHÔNG thêm field** trên `LessonSchedule`. Cảnh báo hạn tái dùng `warning` (`WarningSchema`) sẵn có. KHÔNG khóa idempotent (postpone tất định).
5. **`generate`/`suggest`/`prefs` cũ giữ nguyên contract.** KHÔNG đụng `StudySchedule`/service LLM cũ.
6. **Endpoint:** mount `/api/adaptive`, gate `verifyToken → withTenant`. Self-service JWT `req.user.id`, KHÔNG `verifyPermission`.
7. **Ngày ôn (`kind:'review'`) chỉ hiển thị** — không `done`, không tính tiến độ, không tính missed. Checkpoint "done" = `getPassedBoundaries` (chỉ đọc).
8. **Commit:** Conventional Commits; `git add <files>` cụ thể; KHÔNG `Co-Authored-By`/AI attribution.
9. **Code & comment = English only.** Comment tiếng Việt trong plan → dịch English khi viết file (kể cả test).
10. **KHÔNG tham chiếu plan-taxonomy trong code.** Comment + tên test KHÔNG nhắc `AC-n`/`IS-n`/`OQ`/số phase — mô tả hành vi. Ref ổn định (E11000, SQLSTATE) thì được.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/adaptive/schedule/today-view.js` | `buildTodayView` + `isDone` + `deadlineWarn` — thuần | Tạo |
| `src/modules/adaptive/schedule/postpone.js` | `postponeSchedule(userId, todayLocal)` — orchestrator gọi generator | Tạo |
| `src/modules/adaptive/schedule/local-date.js` | `todayLocalDate(tz, now)` — biên timezone | Tạo |
| `src/modules/adaptive/adaptive.controller.js` | `getToday`, `postponeSchedule` handlers | Sửa |
| `src/modules/adaptive/adaptive.routes.js` | `GET /schedule/today`, `POST /schedule/postpone` | Sửa |
| `src/modules/adaptive/study-schedule.schema.js` | Joi `today` (query tz) | Sửa |
| `src/__tests__/today-view.test.js` | Unit join done + missed + progress + deadlineWarn | Tạo |
| `src/__tests__/schedule-postpone.test.js` | Unit đẩy lùi + cường độ giữ + nghỉ-7-ngày + tất định + no-skip | Tạo |
| `src/__tests__/schedule-today.api.test.js` | API today/postpone (auth, done reflect, missed, warning) | Tạo |

---

## Task 1: `today-view.js` — join done + missed + tiến độ + cảnh báo hạn + ngủ đông (AC-1,2,4,8,11)

**Files:** Create `today-view.js`; Test `today-view.test.js`

> Thuần: nhận `sched` (plain), `doneMaps` (Map courseSlug→Map path→{status}), `passedBoundaries` (Set), `lastActiveDate` (Date|null), `todayLocal` (string). Không DB, không `new Date()`.

- [ ] **Step 1: Test thất bại**
  - **Ngủ đông (AC-11):** `lastActiveDate` cách `todayLocal` ≥ `DORMANT_INACTIVE_DAYS` (14) → trả `{status:'dormant', resume:true, message}`, **không** có `items`/`missed`. Cách < 14 hoặc `lastActiveDate` null-nhưng-mới → `status:'active'`.
  - `buildTodayView` (active): ngày hôm nay 2 lesson (1 completed, 1 chưa) + 1 review → item học có `done` đúng; review `done:null`; `progress:{doneCount:1,totalCount:2,remainingMin:<đúng>}` (review **không** vào totalCount). Status `mastered` → done. Ngày rỗng → items:[], progress zero.
  - `missed`: có ngày `< todayLocal` còn lesson chưa done → `missed.count>0, oldestDate, days[]`; ngày quá hạn chỉ có review/checkpoint → **không** vào missed; hết → `missed:null`.
  - checkpoint item: `done` = `passedBoundaries.has(boundaryKey)`.
  - `deadlineWarn(finish, deadline)`: finish>deadline → `{overdue:true, suggestions:[…2 mục…]}`; không deadline / kịp → `{overdue:false, suggestions:[]}`.
- [ ] **Step 2:** `npx jest today-view -i` → FAIL.
- [ ] **Step 3: Implement** (design §4/§5/§7) — `DORMANT_INACTIVE_DAYS=14`, `isDone`, `buildTodayView` (nhánh dormant trước), `deadlineWarn`; export. `daysBetween` thuần từ 2 chuỗi/Date, không `new Date()` không-tham-số.
- [ ] **Step 4:** PASS.
- [ ] **Step 5: Commit** `feat(adaptive): today's tasks, missed days, deadline warning, and dormant state`

---

## Task 2: `local-date.js` — biên timezone (AC-1)

**Files:** Create `local-date.js`; test gộp trong `schedule-today.api.test.js`

- [ ] **Step 1: Implement** — `todayLocalDate(tz, now)`: `Intl.DateTimeFormat('en-CA',{timeZone:tz})` → 'YYYY-MM-DD'; `now` **tham số** (controller truyền `new Date()` ở biên); tz lỗi → fallback 'Asia/Saigon' + log.
- [ ] **Step 2: Verify** — unit nhỏ: `todayLocalDate('Asia/Saigon', new Date('2026-08-06T18:00:00Z'))==='2026-08-07'`.
- [ ] **Step 3: Commit** `feat(adaptive): resolve learner-local calendar date from timezone at the boundary`

---

## Task 3: `postpone.js` — đẩy lùi lịch (AC-5,6,7,9)

**Files:** Create `postpone.js`; Test `schedule-postpone.test.js`

> Tách phần thuần (regen từ remaining) khỏi I/O để test không-DB nếu tiện; hoặc seed qua helper `db`.

- [ ] **Step 1: Test thất bại**
  - **AC-5 cường độ:** seed lịch có ngày quá hạn nhiều lesson chưa done + `focusDurationMin=60` → sau postpone **không ngày nào `loadMin>60`** (trừ oversized); bài bắt đầu từ `todayLocal`.
  - **AC-6 nghỉ 7 ngày:** 7 ngày qua chưa done → mọi bài dời từ hôm nay, `estimatedFinishDate` lùi ~7 ngày, lịch hợp lệ (không cắt đôi, ngày ôn sinh lại).
  - **AC-7 tất định:** gọi postpone 2× → `days` byte-identical.
  - **AC-9 no-skip:** đếm tổng lesson-item trước (chưa done) = tổng sau postpone (chỉ đổi ngày, không mất bài).
  - **AC-10 giữ lịch sử:** sau postpone, ngày `< todayLocal` vẫn còn trong `days` nhưng **chỉ chứa item done**; bài chưa-done cũ xuất hiện ở phần từ hôm nay, **không trùng** ở quá khứ.
  - **cảnh báo hạn:** deadline gần → `sched.warning.overdue:true`.
- [ ] **Step 2:** `npx jest schedule-postpone -i` → FAIL.
- [ ] **Step 3: Implement** (design §6): đọc `LessonSchedule`+`CourseEnrollment`; `remaining = resolveLessonList(userId)` lọc bỏ bài đã done (OQ-1); `pref.minutesPerDay = focusDurationMin` (không ×1.5); `generateSchedule(remaining, pref, bands, todayLocal)`; `pastKept = days<today giữ item done`; `sched.days = pastKept.concat(days)`; `sched.warning = deadlineWarn(summary.estimatedFinishDate, pref.deadline)`; cập nhật `asOfDate/generatedAt/summary`; save.
- [ ] **Step 4:** PASS.
- [ ] **Step 5: Commit** `feat(adaptive): postpone remaining lessons from today at the usual daily intensity`

---

## Task 4: Endpoints `GET /schedule/today` + `POST /schedule/postpone` (AC-1,3,5)

**Files:** Modify `adaptive.controller.js`, `adaptive.routes.js`, `study-schedule.schema.js`; Test `schedule-today.api.test.js`

- [ ] **Step 1: Test thất bại** (helper `app`):
  - `GET /adaptive/schedule/today` no JWT → 401; có JWT + `?tz=Asia/Saigon` → 200 contract §8.
  - **AC-3:** tick 1 lesson qua `POST /course-content/:slug/lessons/progress` → gọi `today` → item đó `done:true`.
  - `today` khi có ngày lỡ → `missed` không null.
  - `POST /adaptive/schedule/postpone` → 200 `{schedule, warning}`; ngày lỡ biến mất; chưa có lịch → 404.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement**
  - Joi: `today = Joi.object({ tz: Joi.string().optional() })` (validate 'query'); export.
  - controller `getToday`: `todayLocal = todayLocalDate(req.query.tz, new Date())`; load `doneMaps` + `passedBoundaries` (course-content.progress.service, chỉ đọc) + `lastActiveDate` (User.profile); `buildTodayView(sched, doneMaps, passedBoundaries, lastActiveDate, todayLocal)`; trả (gồm nhánh `status:'dormant'`).
  - controller `postponeSchedule`: `todayLocal = todayLocalDate(req.query.tz ?? req.body.tz, new Date())`; `postpone.postponeSchedule(userId, todayLocal)`; 404 nếu null; trả `{schedule, warning}`.
  - routes: 2 route mới sau `schedule/prefs`, gate như sẵn.
- [ ] **Step 4:** `npx jest schedule-today.api -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): today's tasks and postpone endpoints for the study schedule`

---

## Task 5: Full suite + purity guard + coexist

**Files:** verify only

- [ ] **Step 1:** `npx jest today-view schedule-postpone schedule-today.api schedule-generator schedule-generate.api -i` → **ALL PASS** (không hồi quy generator/generate cũ).
- [ ] **Step 2:** `grep -RnE "Date\.now|Math\.random|new Date\(\)" src/modules/adaptive/schedule/today-view.js src/modules/adaptive/schedule/postpone.js` → phần thuần **không** có `new Date()` (biên ở controller/local-date).
- [ ] **Step 3:** `grep -Rn "CourseEnrollment" src/modules/adaptive/` → chỉ **đọc** (`findOne`/`.get(`/`.lean(`), KHÔNG `.set(`/`.save(` (Constraint #2).
- [ ] **Step 4:** `grep -RnE "1\.5|150|150%" src/modules/adaptive/schedule/postpone.js` → **không match** (không nhồi — Constraint #3).
- [ ] **Step 5:** Cập nhật `docs/api/integration/` thêm `GET /schedule/today` + `POST /schedule/postpone` nếu docs tồn tại; không mở rộng ngoài feature.

---

## Self-Review

**Acceptance coverage:**
- AC-1/2 (today + done join, ngày ôn không tính) → Task 1 + Task 4. ⬜
- AC-3 (tick reflect, không store mới) → Task 4. ⬜
- AC-4 (missed/red) → Task 1. ⬜
- AC-5 (đẩy lùi giữ cường độ) → Task 3. ⬜
- AC-6 (nghỉ 7 ngày) → Task 3. ⬜
- AC-7 (tất định/idempotent) → Task 3. ⬜
- AC-8 (cảnh báo hạn mềm) → Task 1 + Task 3. ⬜
- AC-9 (no-skip) → Task 3. ⬜
- AC-10 (giữ lịch sử ngày qua) → Task 3. ⬜
- AC-11 (ngủ đông tính-lúc-đọc) → Task 1 + Task 4. ⬜

**DRY/scope check:** không field mới; không store done mới; không cờ dormant lưu; không ghi CourseEnrollment; không endpoint tick/adjust; không cron; không trần 150%; không skip; không auto-đuổi. ⬜
**Coexist check:** generator/generate/suggest/prefs cũ xanh, contract không đổi. ⬜

---

## Câu hỏi mở (Technical Review — chốt khi implement)

1. **OQ-1** — `remaining` = `resolveLessonList` (thứ tự lộ trình) lọc bỏ done — đúng thứ tự học (Task 3).
2. **OQ-2** — ngưỡng dormant 14 ngày + tín hiệu `lastActiveDate` vs `max(lastActiveDate,lastLoginAt)` (Task 1).
3. Checkpoint "done" = `getPassedBoundaries` (chỉ đọc) — xác nhận trong Task 1.
