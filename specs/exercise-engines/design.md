<!-- Tiếng Việt — thiết kế Phase 4 (engine bài tập tương tác). Coding-oriented, không quá chi tiết.
     Chốt: 3 engine modality; QuizSet→assessment_questions; 1 ExerciseProgress; threshold theo bộ; talk≠speaking;
     hoãn taxonomy Part. Framing: docs/plan-phase4-exercise-engines.md. -->
# Thiết kế: Exercise Engines (Phase 4)

- **Ranh giới:** `exe-api` (module MỚI `modules/exercise/` + mở rộng `course-content` progress/refs + `adaptive` ingest),
  `exe-admin` (authoring bộ item), `exe-web` (player trong lesson viewer).
- **Quyết định đã chốt:** Q-P4.1 (3 engine modality) · Q-P4.2/3 (QuizSet→`assessment_questions`) · Q-P4.4 (1
  `ExerciseProgress` chung) · Q-P4.5 (threshold theo bộ) · Q-P4.6 (gate `withQuota`+`aiCallLimiter`) · Q-P4.7 (talk≠
  speaking) · Q-P4.8 (hoãn taxonomy Part, lọc theo `targetGoal`) · Q-P4.9 (admin CRUD tối thiểu).

## 1. Ba engine theo modality (kỹ năng = tag)
| type | phủ kỹ năng | nguồn item | chấm | ngưỡng xong |
|---|---|---|---|---|
| `quiz` | Nghe/Đọc/Ngữ pháp/Từ vựng | `QuizSet` → `assessment_questions` (+stimulus) | `gradeObjective()` | `passScore` (%) |
| `writing` | Viết | `WritingTask` (prompt+rubric) | `scoreWriting()` | `passBand` |
| `speaking` | Nói (đơn thoại/read-aloud) | `SpeakingTask` (prompt+rubric) | `asr()`+`scoreSpeaking()` | `passBand` |

`talk` (hội thoại tự do) giữ nguyên — KHÔNG chấm, KHÔNG gate (Q-P3.5).

## 2. Data model (mới — `modules/exercise/`)

```
quiz_sets            code(unique,index) title skill cefrLevel targetGoal
                     questionIds:[ObjectId→AssessmentQuestion] stimulusIds?:[…]
                     passScore(Number %,default 70) masteryScore?(default 90)
                     status['draft','published'] timeLimitSec?  {timestamps}
writing_tasks        code(unique) title prompt taskType? rubricId(→Rubric) cefrTarget targetGoal
                     passBand(Number,default 5.5) masteryBand?(default 7) status  {timestamps}
speaking_tasks       code(unique) title prompt referenceText?(read-aloud) rubricId cefrTarget targetGoal
                     passBand(default 5.5) masteryBand? minDurationSec? status  {timestamps}

exercise_progress    userId(→User,index) type['quiz','writing','speaking'] refId(String=code)
                     status['in_progress','completed','mastered'] score(Number|null)
                     bestScore(Number|null) attempts(Number,0) lastAttemptAt
                     index unique {userId,type,refId}   {timestamps}
```
- **Tái dùng** `assessment_questions`/`assessment_stimuli`/`Rubric` — KHÔNG chép item. `answerKey`/`transcript` là
  `select:false`, không bao giờ ship cho learner.
- `ExerciseProgress` chung 3 type (Q-P4.4) → dễ query tổng hợp; ipa/talk giữ hệ riêng (IpaProgress/none).
- Program-aware ở **authoring**: `targetGoal` gắn trên bộ; khi curate lọc `assessment_questions` theo
  `skill/cefr/targetGoal`. Runtime chỉ ref theo `code` (không lọc lại).

## 3. Contracts — learner (`/api/exercises`, `verifyToken`)
Envelope `{ <key>: … }` (khớp modules/ipa, không `{data}`).

| Method | Đường | Việc |
|---|---|---|
| GET | `/quiz/:code` | Trả bộ câu hỏi **ẩn answerKey** + stimulus (audio/passage). 404 nếu chưa published. |
| POST | `/quiz/:code/submit` | Body `{answers:[{questionId,response}]}` → `gradeObjective` từng câu → `score=%đúng` → cập nhật `ExerciseProgress` + feed `learning_events` → trả `{score, passed, perQuestion:[{questionId,correct}], status}`. |
| GET | `/writing/:code` | Trả prompt (ẩn rubric nội bộ). |
| POST | `/writing/:code/submit` | Body `{text}` → `scoreWriting(rubric,prompt,cefrTarget)` → band → progress + event → `{band, passed, criteria, status}`. Gate `withQuota`+`aiCallLimiter`. |
| GET | `/speaking/:code` | Trả prompt/referenceText. |
| POST | `/speaking/:code/submit` | Body `{audioUrl|audioB64}` → `asr` → `scoreSpeaking` → band → progress + event → `{band, passed, transcript, status}`. Gate quota. |

- `status` sau nộp: `score≥masteryScore`→`mastered`; `≥passScore`→`completed`; else `in_progress`. **Monotonic** (không tụt hạng) như Phase 3.
- Dev không set `SCORING_ENGINE_URL`/`LLM_PROXY_URL` ⇒ adapter trả **mock hợp lệ** → chạy local không cần key.

## 4. Bắc cầu (điểm mấu chốt)

### 4.1 Ref bài tập (`course-content`)
- `EXERCISE_TYPES` (`course-content.model.js`) += `'quiz','writing','speaking'`.
- `resolveReferences` (`course-content.references.js`): thêm nhánh mỗi type — batch-resolve `refId` theo
  `QuizSet/WritingTask/SpeakingTask.find({code:{$in},status:'published'})`; lỗi `REFERENCE_NOT_FOUND`/
  `REFERENCE_NOT_PUBLISHED` cùng shape `refErr`. (quiz/writing/speaking thôi hết `UNSUPPORTED_EXERCISE_TYPE`.)

### 4.2 Tiến độ khóa (Phase 3 — `course-content.progress.service.js`)
- `flatten`: hiện gom `ipaCodes`; thêm gom theo type → `exRefs:[{type,refId}]` (hoặc `quizCodes/writingCodes/speakingCodes`).
- `evaluateStatus`: ngoài ipa (`IpaProgress`), đọc `ExerciseProgress.find({userId,type,refId:{$in}})` cho 3 type mới.
  Bài `completed` ⇔ **mọi** bài tập (ipa + quiz/writing/speaking) ở status ≥ completed; `mastered` ⇔ mọi cái mastered.
  `talk` vẫn bỏ qua. Không có bài tập ⇒ completed khi viewed (giữ nguyên).

### 4.3 Mastery (adaptive) — `adaptive.service`
- Thêm `ingestQuizResult` / `ingestWritingResult` / `ingestSpeakingResult` theo template `ingestIpaAttempt`:
  quiz → 1 `learning_event`/câu (`source:'quiz'`, `skill/subskill/itemCefr` từ question, `correct` từ gradeObjective);
  writing/speaking → 1 event (`source:'lesson'`, `skill:'writing'/'speaking'`, `band`). Best-effort (try/catch).
  `computeMastery` tự nhặt (không phụ thuộc source) → tự cập nhật StudentModel nếu đã tồn tại.

## 5. exe-web (runtime)
- `services/exercise.service.ts` + `use-exercise.ts` (get + submit mutations). Types quiz/writing/speaking.
- Player trong feature `exercises` (mới) hoặc mở rộng `courses`: **QuizPlayer** (MCQ/điền/nghe — tái dùng cấu trúc
  step của assessment exam UI), **WritingEditor** (textarea + nộp), **SpeakingRecorder** (tái dùng mic recorder của ipa).
- Lesson viewer `ExerciseLauncher`: type `quiz/writing/speaking` → mở player nội tuyến/route; sau khi đạt → gọi
  `recordLesson` (Phase 3) để đồng bộ hoàn thành Bài. (ipa/talk giữ như cũ.)

## 6. exe-admin (authoring tối thiểu — K6)
- CRUD `QuizSet` (chọn câu từ ngân hàng theo `skill/cefr/targetGoal`, đặt `passScore`, publish),
  `WritingTask`/`SpeakingTask` (prompt + chọn rubric + `passBand`, publish). Qua admin-api resource mới (factory hoặc
  hand-mount, KHÔNG business logic — delegate service).

## 7. Ngoài phạm vi (→ Phase 4.x/5)
Taxonomy Part/Format kỳ thi (TOEIC Part 1–7, IELTS task); sinh item bằng AI; randomize/exposure control; đề xuất bài
tập thích ứng; mô phỏng đề thi đầy đủ (đã có ở assessment).

## 8. Rủi ro & lưu ý
- **Rò rỉ đáp án:** luôn `.select('-answerKey -acceptedVariants')` (question) + ẩn `transcript` (stimulus) + không
  ship `rubric`. Chấm ở server.
- **Chi phí AI:** writing/speaking qua `withQuota`+`aiCallLimiter` (budget circuit-breaker đã có).
- **Chạm Phase 3:** mở rộng `flatten`/`evaluateStatus` là additive; test lại bridge ipa cũ không hồi quy.
- **MCQ shuffle:** giữ hợp đồng `optionOrder[displayed]=originalIndex`, `answerKey` theo thứ tự gốc (như assessment).
