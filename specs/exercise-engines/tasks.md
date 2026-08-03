<!-- Tiếng Việt — tasks Phase 4 (exercise engines). Design cùng thư mục. Chốt Q-P4.* xem design §Quyết định. -->
# Tasks: Exercise Engines (Phase 4)

**Design:** `design.md` · **Framing:** `docs/plan-phase4-exercise-engines.md`.
**Nhánh:** exe-api `feature/exercise-engines` (off `feature/course-program`) · exe-web `feature/exercise-engines-web`
(off `feature/course-program-web`) · exe-admin tiếp `feature/course-management-admin`. Chain (chưa merge develop).

**Nguyên tắc:** tái dùng `grade.js`/`assessment_questions`/scoring-engine adapter/`learning_events`; KHÔNG chép item;
answerKey/transcript/rubric không rời server; mở rộng Phase 3 bridge additive (test lại ipa không hồi quy).

---

## Khối K1 — Engine `quiz` + nền `ExerciseProgress` (exe-api)

### K1.1 — Model nền
- `modules/exercise/exercise-progress.model.js` (`ExerciseProgress`, unique `{userId,type,refId}`).
- `modules/exercise/quiz-set.model.js` (`QuizSet`).
- Test model: default `passScore=70`, status enum, unique index.
- Commit: `feat(exercise): ExerciseProgress + QuizSet models`.

### K1.2 — Quiz service + routes (serve + submit + chấm)
- `exercise.quiz.service.js`: `getQuiz(code)` (ẩn answerKey, kèm stimulus), `submitQuiz(userId,code,answers)` →
  `gradeObjective` từng câu → `score` → upsert `ExerciseProgress` (monotonic) → trả kết quả.
- `exercise.controller.js` + `exercise.routes.js` (`GET/POST /quiz/:code`), mount `app.use('/api/exercises', …)`.
- Test API: serve ẩn answerKey; submit đúng/sai → score + status completed khi ≥passScore; 404 chưa publish; 401.
- Commit: `feat(exercise): quiz serve + submit with objective grading`.

### K1.3 — Bắc cầu quiz vào tiến độ + mastery
- `course-content.model.js`: `EXERCISE_TYPES += 'quiz'` (+ writing/speaking luôn ở K4, hoặc từng phần).
- `course-content.references.js`: nhánh `quiz` → validate `QuizSet` published.
- `course-content.progress.service.js`: `flatten` gom `quizCodes`; `evaluateStatus` AND `ExerciseProgress` quiz.
- `adaptive.service.js`: `ingestQuizResult` (learning_events source:'quiz'); gọi từ `submitQuiz` best-effort.
- Test: Bài có quiz chưa đạt → in_progress; đạt → completed; import ref quiz sai → REFERENCE_NOT_FOUND; ipa bridge cũ pass.
- Commit: `feat(exercise): bridge quiz into course progress + mastery`.

## Khối K2 — Engine `writing` (AI) (exe-api)
- `writing-task.model.js`; `exercise.writing.service.js` (`getWriting`, `submitWriting` → `scoreWriting` → band →
  progress + `ingestWritingResult`); routes `GET/POST /writing/:code` (gate `withQuota`+`aiCallLimiter`).
- `EXERCISE_TYPES += 'writing'` + references nhánh + `flatten`/`evaluateStatus` writing.
- Test: submit → band + status (mock adapter); dưới min-words → lỗi; 404/401; bridge tiến độ.
- Commit: `feat(exercise): writing engine (AI-scored) + progress bridge`.

## Khối K3 — Engine `speaking` (AI) (exe-api)
- `speaking-task.model.js`; `exercise.speaking.service.js` (`getSpeaking`, `submitSpeaking` → `asr`+`scoreSpeaking` →
  band → progress + `ingestSpeakingResult`); routes (gate quota).
- `EXERCISE_TYPES += 'speaking'` + references + bridge. **Khác talk** (rõ trong label/route).
- Test: submit audio (mock) → band + status; dưới min-input → unscored; bridge.
- Commit: `feat(exercise): speaking engine (AI-scored) + progress bridge`.

## Khối K4 — Hợp nhất ref + hardening (exe-api)
- Đảm bảo `EXERCISE_TYPES` = `['ipa','talk','quiz','writing','speaking']`; `resolveReferences` đủ 3 nhánh mới; message
  VN theo dòng import. Cập nhật hợp đồng learner (`specs/lesson-runtime/contracts/…` hoặc contract mới exercises).
- Full `jest` — không hồi quy course-content/course-progress/assessment.
- Commit: `feat(exercise): finalize exercise-ref validation for quiz/writing/speaking`.

## Khối K5 — exe-web runtime
- `services/exercise.service.ts` + `hooks/use-exercise.ts` + types. `features/exercises/`: **QuizPlayer**,
  **WritingEditor**, **SpeakingRecorder** (tái dùng mic recorder ipa). Lesson viewer `ExerciseLauncher` mở đúng player
  theo type; đạt ngưỡng → gọi `recordLesson` đồng bộ Bài. Route/deep-link nếu cần.
- Commit: `feat(exercises): quiz/writing/speaking players + lesson integration`.

## Khối K6 — exe-admin authoring
- admin-api resource(s) cho `QuizSet`/`WritingTask`/`SpeakingTask` (delegate service; chọn câu từ ngân hàng theo
  skill/cefr/targetGoal; publish). exe-admin UI CRUD tối thiểu.
- Commit (api): `feat(admin): exercise-set authoring resources` · (admin): `feat(exercise-authoring): quiz/writing/speaking sets`.

## Khối K7 — Verify + chốt
- exe-api `jest`; exe-web + exe-admin `tsc`/`lint`/`build`. Cập nhật spec. Báo cáo bàn giao.

---

### Thứ tự
K1 (khung + quiz) → K2/K3 (song song, bám khung) → K4 (hardening) → K5 (web) → K6 (admin) → K7.
Deploy: exe-api → exe-admin → exe-web.
