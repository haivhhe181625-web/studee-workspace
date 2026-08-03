<!-- Tiếng Việt — tasks Phase 3 (tiến độ khóa). Design/data-model/contract cùng thư mục. -->
# Tasks: Phase 3 — Tiến độ, Mastery & Unlock

**Nhánh:** `exe-api` `feature/course-progress` (off `feature/lesson-runtime-api`) · `exe-web`
`feature/course-progress-web` (off `feature/lesson-runtime-web`). Chain vì Phase 2 chưa merge develop.

## P3-T1 — Enrollment + đọc tiến độ (exe-api)
- `course-content.progress.model.js` (CourseEnrollment), `.progress.service.js` (`enroll`, `getProgress` + unlock/
  rollup), controller (+3 handler), routes (`POST /:slug/enroll`, `GET /:slug/progress`).
- Test: enroll idempotent, 404 chưa publish; getProgress chưa-enroll `{enrolled:false}`, unlock Bài đầu, 401.
- Commit: `feat(course-content): enrollment + read course progress`.

## P3-T2 — Ghi nhận học Bài + streak (exe-api)
- `recordLesson` (bắc cầu ipa, unlock, completed khóa), `user/streak.service.js` `touchStreak`, route
  `POST /:slug/lessons/progress`, controller.
- Test: record Bài không-ipa → completed + unlock kế; Bài ipa chưa đạt → in_progress; mọi ipa completed → completed;
  409 chưa enroll; 404 pathKey sai; streak tăng.
- Commit: `feat(course-content): record lesson progress + streak writer`.

## P3-T3 — Guard gỡ xuất bản (Q-P3.2, exe-api)
- `changeStatus`: chặn `published→ready` khi có enrollment (409 COURSE_LOCKED). Test.
- Commit: `feat(course-content): block unpublish when enrolled`.

## P3-T4 — exe-web tiến độ UI
- service + hooks (query progress + mutations enroll/record), types; CourseCard badge; CourseDetailContainer
  khoá/mở + % + "Học tiếp"; LessonViewer mark-done + gate.
- Commit: `feat(courses): enroll + progress + sequential unlock UI`.

## P3-T5 — Verify + chốt
- exe-api `jest`; exe-web `tsc`+`lint`+`build`. Cập nhật spec. Báo cáo.

---

## Kết quả Phase 3 — ✅ Xong

| Task | Commit | Repo · nhánh |
|---|---|---|
| P3-T1 enrollment + read progress | `67dec74` | exe-api · `feature/course-progress` (off `feature/lesson-runtime-api`) |
| P3-T2 record lesson + streak writer | `df6341a` | exe-api |
| P3-T3 guard gỡ xuất bản (Q-P3.2) | `0a406b3` | exe-api |
| P3-T4 exe-web enroll + tiến độ + unlock | `83ab951` | exe-web · `feature/course-progress-web` (off `feature/lesson-runtime-web`) |

**Verify:**
- exe-api `jest course-content course-progress` → **13 suites / 95 pass**. Full `jest` → **900 pass / 2 fail** (2 =
  `avatar-storage` EPERM Windows, môi trường — không hồi quy, như Phase 1/A/B).
- exe-web `tsc` **0 lỗi** · `eslint` **0 error** · `next build` **OK**.
- **Chốt §8:** mọi đề xuất OK — enroll tự chọn, gate completion, ipa đọc `ipa_progress` (không chấm lại), talk
  không gate (Q-P3.5), streak writer, chặn gỡ xuất bản khi có enroll (Q-P3.2), % theo Bài/Chặng (không nhét adaptive).
- **Bắc cầu, không đo lại:** hoàn thành Bài suy từ engine ipa; định danh Bài theo path `phaseKey/moduleKey/lessonKey`.
- **Chưa push/PR.** Deploy: exe-api → exe-web.

**Ngoài phạm vi (→ Phase 4/5):** chấm talk + engine Nghe/Đọc/Viết/quiz; admin xem tiến độ; badge enroll trên
catalog (cần endpoint "khóa của tôi" — chưa làm); XP/gamification sâu.
