# Spec: Quiz & Flashcard Graded Exercises (Khóa học tích hợp)

- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Trạng thái:** Đã build (spec hồi tố)
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — **Đã xây:** modules/quiz (LessonQuiz, grading, learner routes + admin authoring), modules/adaptive (ingestQuizAttempt), modules/course-content (progress.service xử lý quiz/flashcard exercises)
  - `web` (`exe-web`) — **Đã xây:** quiz/flashcard inline runners, exercise host dispatcher, hooks, types
  - `admin` (`exe-admin`) — **Đã xây:** lesson-quiz CRUD pages (curate bank, publish)
- **Module liên quan (exe-api):**
  - `services/api/src/modules/quiz/` — quiz grading, flashcard course-side (NOT global SRS)
  - `services/api/src/modules/assessment/` — question bank, testlet stimuli, grading domain
  - `services/api/src/modules/adaptive/` — mastery ingestion (quiz attempts feed learning_events)
  - `services/api/src/modules/course-content/` — lesson progress evaluation
  - `services/api/src/admin-api/` — lesson-quiz authoring resources

---

## 1. Mục tiêu

Cung cấp một loại bài tập graded exercises (có chấm điểm) tích hợp trực tiếp trong `/learn/:slug/study` space nhằm cho phép:
- **Learner:** làm quiz (MCQ) từ ngân hàng câu hỏi chung + flashcard vocabulary để luyện tập, nhận điểm, tiến độ bài học cập nhật.
- **Admin:** tạo cấu hình quiz bằng cách chọn từng câu hỏi từ ngân hàng (hoặc cả testlet) + thẻ flashcard được gán sẵn; publish quiz để đóng băng item set và bắt đầu grading.
- **Hệ thống:** chấm điểm server-side (answerKey giữ bí mật), nuôi dữ liệu mastery/competency từ kết quả quiz, theo dõi tiến độ học.

---

## 2. Bối cảnh

### Hiện trạng (đã kiểm tra code)

**2.1 Quiz — Mô hình LessonQuiz**
- Quiz là **một tính năng graded exercise** tích hợp vào bài học (lesson) thông qua `lesson.exercises[]` (type='quiz', refId=LessonQuiz.code).
- Mỗi quiz được đại diện bởi một **LessonQuiz doc** (model: `services/api/src/modules/quiz/lesson-quiz.model.js`):
  - `code` (unique, string) — định danh (vd: 'quiz_listening_01')
  - `title` — tên hiển thị
  - `source: 'curated'` (chỉ hỗ trợ curate; không có spec-mode động)
  - `itemIds[]` — danh sách câu hỏi do admin chọn tay từ ngân hàng assessment_questions (bao gồm testlet nguyên bộ + grammar/vocab group)
  - `resolvedItemIds[]` — **tập đã đóng băng tại publish** — tính mẫu số chấm điểm (NEVER thay đổi sau publish)
  - `passThreshold: 0.8` (mặc định) — ngưỡng đạt: `score >= 0.8`
  - `status: 'draft' | 'published'` — draft = chưa có learner, published = đã đóng băng
  - `centerId: null` (Phase 1 platform-shared; không tenant-scoped)

**2.2 Question Bank — Được phục vụ từ Assessment**
- Quiz reuse **toàn bộ** ngân hàng `AssessmentQuestion` hiện có (~558 items, active).
- Các loại câu hỏi được phục vụ **chỉ tự chấm được (objective):**
  - Bao gồm: mcq, cloze, fill_blank, dictation, true_false_ng, matching_headings, sentence_completion, labeling
  - **Loại trừ:** matching (FE emits object; grader expect ordered array → mismatch)
  - Set này gọi `COURSE_GRADABLE_TYPES` (service/quiz.grading.js, line 19)
  - Mỗi câu phải có `answerKey` hợp lệ tại publish (labeling cần per-prompt string array → renderable)
- Câu hỏi được **gom theo testlet** (audio passage/reading passage + questions) qua `stimulusId`:
  - Một stimulus = một testlet (vd: 1 bài nghe + 3 câu hỏi về bài nghe)
  - Client render stimulus + items nó (group display)
  - Admin chọn testlet nguyên bộ (selectTestletAware: không cắt ngang testlet)

**2.3 Flashcard — Bài tập self-contained**
- Flashcard là bài tập **độc lập, không dùng global SRS** (clone-based).
- Bound vào lesson qua `lesson.exercises[].type='flashcard'`, refId=FlashcardDeck._id.
- **Self-contained completion:** learner review all cards → `reviewedCardIds >= deck cardIds` → mark lesson completed.
- **Không feed mastery:** flashcard pass chỉ đánh dấu lesson done, không sinh learning_events → competency không thay đổi.
- Minimum cards: ≥2 (`MIN_FLASHCARD_CARDS`).

**2.4 Admin Authoring — LessonQuiz CRUD**
- Quản lý qua `/api/admin/lesson-quizzes` REST resource:
  - CRUD config (tạo draft, chỉnh sửa itemIds, đặt passThreshold)
  - **Action: publish** — delegate `quiz.service.publishLessonQuiz`:
    - Freeze `itemIds` → `resolvedItemIds` (tập không đổi lại)
    - Validate EVERY item:
      - Status='active' + itemType ∈ COURSE_GRADABLE_TYPES
      - answerKey không null
      - Labeling items: answerKey must be renderable (per-prompt string array)
    - Reject nếu bất kỳ item fail → all-or-nothing (không publish một phần)
  - **Guard (H1):** nếu admin edit/retire/delete 1 câu hỏi trong `AssessmentQuestion` mà câu đó đã frozen trong published quiz → reject với ITEM_REFERENCED_BY_PUBLISHED_QUIZ

**2.5 Grading — Server-side, Mẫu số cố định**
- Denominator invariant: `score = correctCount / resolvedItemIds.length` (tập đã đóng băng, KHÔNG số câu learner nộp)
- Mục tiêu: ngăn chặn cheat (chỉ trả lời câu dễ → không thể tăng tỷ lệ bằng cách bỏ câu khó)
- Per-item grading: `gradeObjective(itemType, answerKey, response)` → 1 or 0
  - MCQ: so khớp option index (địa chỉ ban đầu, không shuffle course-side)
  - Cloze/fill_blank: so khớp text (normalize)
  - Labeling: so khớp per-prompt
  - Etc. (see gradeObjective logic)
- Answer keys: `select: false` trên AssessmentQuestion → NEVER emit lên client
- Transcript (stimulus audio): **NEVER emit** → server-side only

**2.6 Learner Routes — Được mount tại `/api/courses/:slug/quiz|flashcard`**
- Gated bởi `verifyToken` + `assertEnrolled` (must have active CourseEnrollment + course must be published)
- POST `/courses/:slug/quiz/items` — lấy items + stimuli (answers stripped)
- POST `/courses/:slug/quiz/submit` — chấm điểm, persist attempt, feed mastery
- POST `/courses/:slug/flashcard/cards` — lấy flashcard cards
- POST `/courses/:slug/flashcard/complete` — mark complete (upsert attempt)

**2.7 Learner Progress & Mastery**
- Quiz attempt = `QuizAttempt` doc: userId, courseId, lessonPathKey, quizConfigId, score, passed, perItem[], timestamp
- Learning events (mastery-feeding): mỗi resolved item sinh 1 event (source='quiz', skill/subskill/itemCefr từ item, correct from grading)
- Best-score-wins: re-attempt drop prior events on same lesson (latest submission dedup) → không inflate subskill sampleCount
- Progress evaluation: lesson completed nếu mọi bài tập (ipa + quiz + flashcard) ≥ completed status; mastered nếu mọi ≥ mastered
- Flashcard attempt = `FlashcardAttempt` doc: upsert (latest pass replaces reviewed set), NO learning_events

**2.8 Relationship với existing features**
- **vs exercise-engines** (specs/exercise-engines/design.md): Khác model entirely. Exercise-engines = QuizSet+WritingTask+SpeakingTask (new, phase 4); course-graded-exercises = LessonQuiz curated từ bank (phase 1, đã xây). Coexist: exercise-engines cover writing/speaking; course-graded-exercises cover reading/listening via curated testlets. Cả hai dùng chung `gradeObjective` từ modules/assessment.
- **vs checkpoint** (specs/course-checkpoints-band-up/): Cùng dùng `gradeObjective` + testlet grouping; checkpoint là boundary-level quiz (phase/course); course-graded-exercises là lesson-level quiz.
- **vs lesson-runtime** (specs/lesson-runtime/): Quiz/flashcard là 2 trong các exercise types vào lesson; runtime dispatcher (host component) route learner tới runner.
- **vs course-content-import**: Question bank source (vd file import thêm câu hỏi) → cấp feed cho quiz curate.

### Thay đổi từ plan
- **Removed spec-mode:** Plan gốc đề cập `spec: {skill, cefr, count}` (dynamic sampling) → **Đã xoá.** Phase 1 chỉ curate mode (admin chọn tay).
- **Removed "matching" từ COURSE_GRADABLE_TYPES:** Plan đầu không loại trừ; nhận ra FE matching response shape (object) ≠ grader expect (array) → matching LUÔN grade 0 course-side → loại trừ để không confuse admin. Checkpoint (proctored) vẫn dùng matching (serialize khác).

---

## 3. Phạm vi

### Trong phạm vi

- **IS-1** — **Learner: fetch quiz items** — POST `/courses/:slug/quiz/items` trả mảng items (answers stripped) + stimuli (testlet grouping).
- **IS-2** — **Learner: submit & grade** — POST `/courses/:slug/quiz/submit` nhận answers, chấm điểm (denominator=resolvedItemIds.length, pass threshold=0.8), persist QuizAttempt, feed mastery (learning_events per item), update lesson progress.
- **IS-3** — **Learner: flashcard** — POST `/courses/:slug/flashcard/cards` lấy cards; POST `/courses/:slug/flashcard/complete` mark pass (upsert FlashcardAttempt, NO mastery feed).
- **IS-4** — **Admin: LessonQuiz CRUD** — `/api/admin/lesson-quizzes` resource (list, create draft, edit itemIds/passThreshold, soft-delete as draft).
- **IS-5** — **Admin: publish** — action publish trên LessonQuiz: freeze itemIds → resolvedItemIds, validate objective+keyless, reject publish nếu fail.
- **IS-6** — **Admin: guard published items** — `assertItemNotInPublishedQuiz`: nếu admin edit/retire/delete question đã frozen trong published quiz → reject (ITEM_REFERENCED_BY_PUBLISHED_QUIZ).
- **IS-7** — **Question bank sourcing** — admin dùng "group-items" action trên question-bank để **reconstruct testlet grouping** (stimulus mapping) khi pick items.
- **IS-8** — **Testlet-whole selection** — selectTestletAware: admin chọn testlet nguyên bộ (không cắt ngang stimulus + items nó).
- **IS-9** — **Best-score tracking** — learner re-attempt, best-score wins; progress marked as QuizAttempt.passed=true (hoàn toàn sau khi score >= threshold).
- **IS-10** — **Mastery ingestion** — quiz attempt → learning_events (skill/subskill/itemCefr, correct from grading) → adaptive.ingestQuizAttempt → recompute StudentModel mastery.

### Ngoài phạm vi

- **Checkpoint grading** — thuộc specs/course-checkpoints-band-up/; checkpoint dùng chung gradeObjective & testlet nhưng là boundary-level, không lesson-level.
- **Writing/Speaking grading & AI scoring** — thuộc specs/exercise-engines/; course-graded-exercises chỉ cover reading/listening (objective types).
- **Learner unlock gates / course gating** — thuộc specs/lesson-runtime/; quiz completion không gate bài học tiếp theo ở phase 1.
- **Dynamic spec-mode (skill/cefr sampling)** — **Removed from final build.** Phase 1 curate-only; spec-mode postponed.
- **Global SRS for flashcard** — flashcard course-side là self-contained; không dùng clone-based SRS platform → ngoài phạm vi.
- **Re-publish, versioning** — một khi published, chỉ có thể create draft mới (soft-delete current); re-publish (edit resolvedItemIds) ngoài phạm vi.

---

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| **Learner** | Vào bài học, làm quiz MCQ từ ngân hàng + flashcard vocabulary | Luyện tập, nhận điểm, tiến độ cập nhật, competency feed mastery |
| **Admin (nội dung)** | Tạo quiz config, chọn items/testlets từ ngân hàng, đặt ngưỡng pass, publish | Curate bộ câu hỏi đã chọn, đóng băng để learner grading |
| **Tech Lead / Dev (gián tiếp)** | Có graded exercises tích hợp vào lesson runtime + mastery pipeline | Quiz result cập nhật competency, course progress reflects exercise status |

---

## 5. Quyết định nghiệp vụ cần chốt

*Đã chốt toàn bộ khi build (không còn open questions).*

---

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map về một mục "Trong phạm vi".

1. Learner gọi POST `/courses/:slug/quiz/items` → nhận items (answers stripped) + stimuli (testlet grouping). *(IS-1)*
2. Learner gọi POST `/courses/:slug/quiz/submit` với answers → graded per item, score = correctCount/resolvedItemIds.length, passed = score>=threshold, lesson progress update. *(IS-2)*
3. Learner gọi POST `/courses/:slug/flashcard/cards` → cards list; POST `/courses/:slug/flashcard/complete` với reviewedCardIds → FlashcardAttempt upsert, lesson completed. *(IS-3)*
4. Admin list/create/edit LessonQuiz draft via `/api/admin/lesson-quizzes`. *(IS-4)*
5. Admin publish action → freeze itemIds, validate objective+answerKey, reject nếu fail (all-or-nothing). *(IS-5)*
6. Admin edit/retire/delete question đã frozen trong published quiz → reject với code ITEM_REFERENCED_BY_PUBLISHED_QUIZ. *(IS-6)*
7. Admin dùng group-items endpoint để reconstruct testlet grouping khi curate. *(IS-7)*
8. selectTestletAware: chọn testlet nguyên bộ, không cắt ngang stimulus. *(IS-8)*
9. Learner re-attempt → best-score wins (prior events dropped, latest submission wins mastery). *(IS-9)*
10. Quiz attempt → learning_events per item → adaptive.ingestQuizAttempt → StudentModel mastery recomputed. *(IS-10)*

---

## 7. Câu hỏi mở

*Không còn.* Toàn bộ quyết định đã chốt khi phát triển feature.

---

## 8. Ghi chú cho Tech Lead Design / Integration Notes

- **Grading domain separation:** `quiz.grading.js` cô lập pure grading (no HTTP, mocks) → unit testable.
- **Denominator invariant (critical):** mẫu số = resolvedItemIds.length, NOT answer count → prevent cheat by skipping hard questions.
- **Answer key security:** LUÔN `select: false` khi fetch items cho learner; chỉ server biết key.
- **Testlet coherence:** selectTestletAware never split stimulus → learner thấy testlet đầy đủ (audio passage + all questions).
- **Mastery best-score-wins:** prior quiz attempts' events dropped khi new submit → no sampleCount inflation; completion tracked via QuizAttempt.passed=true (not events).
- **Flashcard self-contained:** không dùng global SRS clone, không feed mastery → score/progress chỉ track completion (view + review all cards).
- **Admin guard (H1, policy a):** freeze tại publish is the key → after publish, question edits blocked nếu referenced; prevents silent grading corruption.
- **Checkpoint integration:** checkpoint dùng chung gradeObjective + testlet; nhưng is boundary-level (phase/course), not lesson-level → separate routes.
- **Exercise-engines coexistence:** writing/speaking/new quiz engines (phase 4) separate model (QuizSet) vs course-graded-exercises (LessonQuiz); chúng coexist, khác miền + contract.

