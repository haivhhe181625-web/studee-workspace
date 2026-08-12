# Tasks: Sinh lịch học từ Lộ trình (Study Schedule Generation)

> **v1 (Task 1–8) — ĐÃ SHIP** (branch `feature/study-schedule-generation`, 10 commit, giữ local, review-clean).
> **v2 (Task 9–15) — SLICE MỚI, CHỜ DUYỆT DESIGN** (OQ-5..OQ-8 / T5..T9 ở design §9). Không code tới khi owner chốt.

**Spec:** `spec.md` (v2) · **Design:** `design.md` (v2)
**Repo/branch:** `exe-api`. v2 tách branch `feature/study-schedule-review-and-planning` **base lên nhánh v1 `feature/study-schedule-generation`** (chốt owner 2026-08-05 — v1 chưa merge; v2 sửa trực tiếp code v1). Khi merge → merge cả chuỗi.
**Lưu ý chạy lệnh:** mọi `npm`/`npx jest` trong `services/api` (`cd services/api`). Test ở `src/__tests__/`; helper `__tests__/helpers/{db,app}.js`. Chạy 1 file: `npx jest <pattern> -i`.

---

## Global Constraints

Ràng buộc bám nguyên văn (đối chiếu code thật 2026-08-05 — reviewer dùng khối này làm lens):

1. **NFR-1 — lõi thuần xác định.** `lesson-schedule.generator.js`, `lesson-duration.js`, `cefr.js`, `schedule-suggest.js`: **KHÔNG** `Date.now()`/`Math.random()`/`new Date()` không tham số. `asOfDate`/`pref`/`bands` là tham số. Ngày server chỉ ở **controller (biên)**.
2. **`LessonItem` shape (v2):** `{ lessonKey, title, type, durationMin, skill, phaseBand, exercises:[{type,refId}] }` — nhất quán resolver↔generator. `exercises` phục vụ ngày ôn.
3. **`Preference` DTO generator (v2):** `{ learningDows:[Number], reviewDow:Number, minutesPerDay, deadline }`. **Generator mới KHÔNG đọc `weekendStudy`/`daysPerWeek`.**
4. **`weekendStudy`/`daysPerWeek` KHÔNG xóa khỏi schema** (D8) — `study-schedule.service.js` L112-123 còn dùng; `study-schedule.service.test.js` phải xanh. Chỉ generator mới ngừng đọc.
5. **`StudentPath` query:** `findOne({ userId, isActive: true })` (partial index) — KHÔNG bỏ `isActive`. Cross-module (StudentPath/CourseStructure/StudentModel/CourseEnrollment) **READ-ONLY**.
6. **Endpoint:** mount `/api/adaptive`, gate `router.use(verifyToken, withTenant)`. **Self-service JWT `req.user.id`, KHÔNG `verifyPermission`.** Prefs server-side từ `User.profile.studyPlan` (không nhận từ client).
7. **Coexist:** KHÔNG đụng `StudySchedule` + service/route LLM cũ. `LessonSchedule` (v1) chỉ **thêm** field `days[].isReview` + (tuỳ) `summary`.
8. **Ngày ôn = buổi THÊM** ở `reviewDow`, `isReview:true`, **không** chặn `minutesPerDay`, **không** `oversized`, **không** chứa lesson học. Tuần rỗng exercises → không chèn.
9. **Commit:** Conventional Commits; `git add <files>` cụ thể, KHÔNG `git add .`; KHÔNG `Co-Authored-By`/AI attribution.
10. **Code & comment = English only.** Code block trong plan có thể chứa comment tiếng Việt — **implementer BẮT BUỘC dịch sang English** khi viết file thật (kể cả test).
11. **KHÔNG tham chiếu plan-taxonomy trong code.** Comment + tên test KHÔNG nhắc `AC-n`/`IS-n`/`OQ-n`/`Dn`/`Tn`/số phase — mô tả hành vi thay thế. Ref ngoài ổn định (RFC, SQLSTATE, E11000) thì được.

---

## File Structure (v2)

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/user/user.model.js` | `StudyPlanSchema` + `learningDows`/`reviewDow` | Sửa |
| `src/modules/adaptive/study-schedule.schema.js` | Joi `savePrefs` (learningDows/reviewDow, sàn/trần) + `suggest` | Sửa |
| `src/modules/adaptive/schedule/lesson-duration.js` | Thêm `exerciseDurationOf(type)` + bảng exercise | Sửa |
| `src/modules/adaptive/schedule/lesson-list.resolver.js` | Emit thêm `exercises:[{type,refId}]` | Sửa |
| `src/modules/adaptive/schedule/lesson-schedule.generator.js` | Chọn ngày theo `learningDows`/`reviewDow` + chèn ngày ôn + `summary` | Sửa |
| `src/modules/adaptive/schedule/schedule-suggest.js` | `suggestPlans(totalMin, asOfDate, deadline?)` — thuần | Tạo |
| `src/modules/adaptive/lesson-schedule.model.js` | `days[].isReview` (+ `summary` optional) | Sửa |
| `src/modules/adaptive/adaptive.controller.js` | `generate` đọc learningDows/reviewDow + summary; `suggestSchedule` handler | Sửa |
| `src/modules/adaptive/adaptive.routes.js` | `GET /schedule/suggest` | Sửa |
| `src/__tests__/study-plan-prefs.schema.test.js` | Joi validation prefs v2 | Tạo |
| `src/__tests__/exercise-duration.test.js` | Unit `exerciseDurationOf` | Tạo |
| `src/__tests__/schedule-generator.test.js` | Cập nhật PREF + test ngày ôn/chọn ngày/summary | Sửa |
| `src/__tests__/lesson-list.resolver.test.js` | Assert thêm `exercises` | Sửa |
| `src/__tests__/schedule-suggest.test.js` | Unit suggestion | Tạo |
| `src/__tests__/schedule-generate.api.test.js` | Cập nhật prefs v2 + summary; + suggest API | Sửa |

---

## Task 9: Prefs schema v2 — `learningDows`/`reviewDow` + sàn/trần + `suggest` (AC-13, AC-14)

**Files:** Modify `user.model.js`, `study-schedule.schema.js`; Test `study-plan-prefs.schema.test.js`

- [ ] **Step 1: Test thất bại** — `study-plan-prefs.schema.test.js`: import `savePrefs` từ `study-schedule.schema.js`; assert:
  - hợp lệ: `learningDows:[2,4,6], reviewDow:7, focusDurationMin:120, timeSlots:['morning']` → pass.
  - `learningDows:[2]` (1 phần tử) → error; `learningDows:[1,2,3,4,5,6,7,1]`/trùng → error.
  - `focusDurationMin:45`/`300` → error; `59`/`241` → error; biên `60`/`240` → pass.
  - `reviewDow:5` khi `learningDows:[2,4,6]` (5 < max 6) → error (ôn phải sau học cuối).
  - `reviewDow:7` khi `learningDows:[2,4,6]` → pass.
- [ ] **Step 2:** `npx jest study-plan-prefs.schema -i` → FAIL.
- [ ] **Step 3: Implement**
  - `user.model.js` `StudyPlanSchema`: thêm `learningDows:{type:[Number],default:null}`, `reviewDow:{type:Number,min:1,max:7,default:null}`. **Giữ** field cũ.
  - `study-schedule.schema.js` `studyPlanRule`: thêm `learningDows` (array int 1–7, `.unique().min(2).max(7).required()`), `reviewDow` (int 1–7 required), đổi `focusDurationMin` → `.min(60).max(240).required()`. `daysPerWeek` → optional (suy từ `learningDows.length`). Ràng buộc `reviewDow > max(learningDows)` bằng `.custom()` trên object.
  - Thêm `const suggest = Joi.object({ asOfDate: Joi.string().isoDate().optional(), deadline: Joi.string().allow(null,'').optional() })`; export.
- [ ] **Step 4:** `npx jest study-plan-prefs.schema -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): explicit study/review day-of-week prefs with min/max session bounds`

> **Ghi chú coexist:** đổi sàn/trần `focusDurationMin` áp cả `savePrefs` cũ. Chạy `npx jest study-schedule.service -i` sau step 3; nếu test cũ fixture `focusDurationMin:45`/thiếu `learningDows` fail → cập nhật **fixture test cũ** cho hợp lệ (KHÔNG nới lỏng schema). Ghi vào self-review.

---

## Task 10: Heuristic thời lượng exercise `exerciseDurationOf` (AC-11 nền)

**Files:** Modify `lesson-duration.js`; Test `exercise-duration.test.js`

- [ ] **Step 1: Test thất bại** — assert `exerciseDurationOf('quiz')===8`, `'flashcard'===5`, `'ipa'===5`, `'talk'===10`, `'unknown'>0`.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** — thêm `EXERCISE_DURATION = Object.freeze({ quiz:8, flashcard:5, ipa:5, talk:10 })` + `exerciseDurationOf(type){ return EXERCISE_DURATION[type] ?? 5 }`; export. (Giá trị đề xuất — OQ-7, tinh chỉnh sau.)
- [ ] **Step 4:** PASS.
- [ ] **Step 5: Commit** `feat(adaptive): add per-type exercise duration heuristic for review sessions`

---

## Task 11: Resolver surface `exercises[]` (AC-10 nền)

**Files:** Modify `lesson-list.resolver.js`; Test `lesson-list.resolver.test.js`

- [ ] **Step 1: Cập nhật test** — seed lesson có `exercises:[{type:'quiz',refId:'q1',order:1}]`; assert item trả về có `exercises:[{type:'quiz',refId:'q1'}]` (chỉ type+refId); lesson không exercises → `exercises:[]`. Giữ mọi assert v1.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** — khi emit `LessonItem`, thêm `exercises: (lesson.exercises||[]).map(e=>({type:e.type, refId:e.refId}))`. Cả nhánh StudentPath lẫn fallback.
- [ ] **Step 4:** `npx jest lesson-list.resolver -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): surface lesson exercises in resolved lesson list`

---

## Task 12: Generator v2 — chọn ngày theo `learningDows`/`reviewDow` + chèn ngày ôn + summary (AC-10,11,12,16)

**Files:** Modify `lesson-schedule.generator.js`, `lesson-schedule.model.js`; Update `schedule-generator.test.js`

> **HARD (NFR-1):** không `Date.now()`/`Math.random()`/`new Date()` không-tham-số. Guard vòng lặp bằng trần số ngày.

- [ ] **Step 1: Cập nhật + thêm test** trong `schedule-generator.test.js`:
  - Đổi `PREF` → `{ learningDows:[1,2,3], reviewDow:4, minutesPerDay:45, deadline:null }` (giữ ý nghĩa test chunking cũ: 3 ngày học liên tiếp Hai/Ba/Tư, ôn Năm). Giữ AC-3/4/5/7/8 pass với PREF mới.
  - **Ngày ôn:** lessons có exercises → có 1 day `isReview:true` ở `reviewDow`, `items` = exercises tuần, `loadMin=ΣexerciseDurationOf`, không oversized, không lesson học. Tuần rỗng exercises → **không** có day `isReview`.
  - **Chọn ngày:** học chỉ rơi vào `learningDows`; `reviewDow` mang ngày ôn; đổi `learningDows` không đụng `weekendStudy`.
  - **summary:** trả `{ totalLearningSessions, totalReviewSessions, totalLessonMin, estimatedFinishDate }` đúng.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** (design §4/§6):
  - Bỏ `pickStudyDows`/`WEEKDAY_ORDER`/`WEEKEND`. `assignStudyDates`/vòng chính duyệt ngày, dùng `isoDow(cursor)∈learningDows` để đặt bài, `==reviewDow` để chèn ngày ôn.
  - `isoDow(d)` = `getUTCDay()===0 ? 7 : getUTCDay()`. Tuần lịch ISO (OQ-6): reset `weekLessons` sau mỗi ngày ôn.
  - `buildReviewDay(weekLessons)` gom `exercises` → items (`exerciseDurationOf`), không cap.
  - `buildSummary(days, asOfDate)`.
  - `generateSchedule` trả `{ days, warning, summary }`.
  - `lesson-schedule.model.js`: `days` items schema thêm `isReview:{type:Boolean,default:false}` (+ item cho exercise: `exerciseRef`/`refId` optional bên cạnh `lessonKey`); thêm `summary` (mixed/subdoc, optional).
- [ ] **Step 4:** `npx jest schedule-generator lesson-schedule.model -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): user-picked study/review days, weekly review sessions, and schedule summary`

---

## Task 13: Suggestion engine + `GET /schedule/suggest` (AC-15, AC-17)

**Files:** Create `schedule-suggest.js`; Modify `adaptive.controller.js`, `adaptive.routes.js`; Test `schedule-suggest.test.js`

- [ ] **Step 1: Test thất bại** — unit `suggestPlans(totalLessonMin, asOfDate, deadline?)`:
  - không deadline → `presets` gồm 3 key `thong_tha/can_bang/cap_toc`, mỗi cái có `learningDays,minutesPerDay,estimatedFinishDate`; cùng input → cùng output (determinism).
  - có deadline gấp → `tooTight:true` + `fallbackPlan`; deadline rộng → `tooTight:false`.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** — `schedule-suggest.js` thuần (design §5), export `suggestPlans` + `PRESETS`. Controller `suggestSchedule`: `resolveLessonList(userId)` → `totalLessonMin=ΣdurationOf`; `asOfDate` biên; `deadline` từ query hoặc `studyPlan.deadline`; trả `suggestPlans(...)`. Route `GET /schedule/suggest` (validate `suggest`, gate JWT như các route adaptive). Không cần prefs.
- [ ] **Step 4:** `npx jest schedule-suggest -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): add study intensity suggestions endpoint with completion projection`

---

## Task 14: Controller `generate` v2 — đọc prefs mới + response summary (AC-16)

**Files:** Modify `adaptive.controller.js`; Update `schedule-generate.api.test.js`

- [ ] **Step 1: Cập nhật test** `schedule-generate.api.test.js`: `studyPlan` fixture → thêm `learningDows:[1,2,3,4,5], reviewDow:6, focusDurationMin:60` (bỏ phụ thuộc `weekendStudy`). Assert response có `summary`; `needsPrefs` khi thiếu `learningDows`/`reviewDow`/`focusDurationMin`. Giữ idempotent + 409 + 401.
- [ ] **Step 2:** FAIL.
- [ ] **Step 3: Implement** — controller `generateLessonSchedule`:
  - `needsPrefs` gate: thiếu `plan.learningDows?.length` || `plan.reviewDow==null` || `plan.focusDurationMin==null` → `{needsPrefs:true}`.
  - `pref = { learningDows: plan.learningDows, reviewDow: plan.reviewDow, minutesPerDay: plan.focusDurationMin, deadline: plan.deadline }` (bỏ daysPerWeek/weekendStudy khỏi DTO).
  - `inputHash` gồm pref mới. `{ days, warning, summary } = generateSchedule(...)`; upsert kèm `summary`. Response `{ schedule, warning, summary }`.
- [ ] **Step 4:** `npx jest schedule-generate.api -i` → PASS.
- [ ] **Step 5: Commit** `feat(adaptive): generate schedule from explicit day prefs and return projection summary`

---

## Task 15: Full suite + purity guard + coexist + docs

**Files:** verify only

- [ ] **Step 1:** `npx jest lesson-duration exercise-duration schedule-generator lesson-schedule.model lesson-list.resolver schedule-suggest schedule-generate.api study-plan-prefs.schema study-schedule.service -i` → **ALL PASS** (gồm `study-schedule.service` cũ — Constraint #4/#7).
- [ ] **Step 2:** `grep -RnE "Date\.now|Math\.random|new Date\(\)" src/modules/adaptive/schedule/` → **không match**.
- [ ] **Step 3:** `grep -RnE "weekendStudy" src/modules/adaptive/schedule/` → **không match** (generator mới ngừng dùng; field vẫn còn ở user.model.js + service cũ — đúng D8).
- [ ] **Step 4:** Cập nhật `docs/api/integration/` thêm `GET /schedule/suggest` + summary field nếu docs này tồn tại; không mở rộng ngoài feature.

---

## Self-Review

**Acceptance coverage v2:**
- AC-10 (ngày ôn tổng hợp exercises) → Task 11 + Task 12. ⬜
- AC-11 (ngày ôn không cap/không oversized) → Task 12. ⬜
- AC-12 (chọn thứ học/ôn, không weekendStudy) → Task 9 + Task 12. ⬜
- AC-13 (reviewDow > max learningDows reject) → Task 9. ⬜
- AC-14 (sàn/trần input) → Task 9. ⬜
- AC-15 (suggestion 3 preset / kịp-hạn) → Task 13. ⬜
- AC-16 (summary + warnings trong generate) → Task 12 + Task 14. ⬜
- AC-17 (suggestion/projection determinism) → Task 12 + Task 13. ⬜

**Coexist check:** `study-schedule.service.test.js` xanh; `weekendStudy`/`daysPerWeek` còn trong schema. ⬜
**Type/name consistency:** `exerciseDurationOf`/`suggestPlans`/`buildReviewDay`/`buildSummary`/`isoDow`; `LessonItem` +`exercises`; `Preference` = `{learningDows,reviewDow,minutesPerDay,deadline}`. ⬜

---

## Câu hỏi mở

> Các câu chặn đã chốt (owner 2026-08-05): OQ-5 giữ `weekendStudy` ✅ · OQ-6 tuần ISO ✅ · branch base lên nhánh v1 ✅.

Không chặn (giá trị đề xuất, tinh chỉnh sau):
1. **OQ-7:** heuristic exercise (quiz 8/flashcard 5/ipa 5/talk 10) — đội nội dung tinh chỉnh.
2. **OQ-8:** ngưỡng `tooTight` = >5×90 phút/tuần — tinh chỉnh sau.
