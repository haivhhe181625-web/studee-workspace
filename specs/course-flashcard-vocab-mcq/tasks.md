---
feature: course-flashcard-vocab-mcq
role: tech-lead
status: draft
date: 2026-08-08
depends_on: design.md
---

# Tasks: Kiểm tra vocab MCQ trong khóa học

> **Trạng thái 2026-08-08 — BE + FE ✅ DONE, review clean (chưa push).**
> **BE** (exe-api `feat/PRD-130-course-flashcard-vocab-mcq`): `af9f816` T1 · `660c9b8` T2–T6 · `f95816c` fix · `0120059` T8 · `3fa31da` T9 weak · `67bdd97` T10 · `802eab6` perf/Joi. Test 150/150 targeted (45 fail = sanitize-html pre-existing, độc lập). 3 vòng review clean.
> **FE** (exe-web `feat/PRD-130-course-flashcard-vocab-mcq`): `43c5c4c` T11 · `ac6dd34` T12 · `dbd948e` T13 · `331207d` T14 · `9db835d` fix (1 Critical + 2 Important + 2 minor). tsc clean, vitest 464/466 (2 fail testlet pre-existing). Review + re-review clean.
> **Defer:** E11000 race, cast thừa (BE). **Content migration** (author lại flip→lesson, MCQ→exercise) = content team.
> **Còn lại:** chạy end-to-end (stack) · push/PR 2 repo (human gate).

> TDD-checkbox. Mỗi task 1-PR-worth, test-first. Verify = chạy trong `exe-api/services/api` (hoặc `exe-web`).
> BE (T1–T6) ở `exe-api`; FE (T7) ở `exe-web` — PR riêng (cross-repo, project-context §0).

## BE — exe-api/services/api

- [ ] **T1 — Mở rộng enum tín hiệu vocab (isolation, KHÔNG đụng knowledge-graph)**
  - `learning-event.model.js`: `SOURCES` += `'flashcard'`; `SUBSKILLS` += `'vocabulary'`.
  - **KHÔNG** sửa `adaptive/knowledge-graph.js` (xem design §2.2 — tránh phá gating `use_of_english`).
  - Test: enum chứa giá trị mới; ghi được LearningEvent(source:'flashcard', subskill:'vocabulary'); `computeMastery` lưu `mastery.vocabulary`; **`skillsMeetingMastery(use_of_english)` bất biến** khi có vocab mastery; test attribution cũ vẫn xanh.
  - Verify: `npm test -- learning-event adaptive-mastery-attribution adaptive-path`

- [ ] **T2 — Course MCQ builder (system deck)**
  - Hàm course-side dựng MCQ từ **system deck** của lesson (tái dùng `buildQuizOptions`), đọc `loadSystemDeck` (không ownership). Chiều EN→VI, min 2 thẻ, distractor co giãn.
  - Test: deck 2/3/≥4 thẻ → 2/3/4 đáp án; nhiễu loại trùng nghĩa; đáp án đúng luôn có mặt; deck <2 thẻ → ApiError.
  - Verify: `npm test -- quiz-flashcard-course` (test mới) hoặc `npm test -- flashcard-course.api`

- [ ] **T3 — Chấm 1 thẻ: SRS + review-log + LearningEvent**
  - Endpoint answer: `applyReviewEvent(userId, systemCard, {mode:'quiz', correct})` (tái dùng) + `LearningEvent.create({source:'flashcard', subskill:'vocabulary', correct, itemCefr})`; `itemCefr` = Phase.cefrFrom → deck.level → null (OQ-1).
  - Test: 1 answer → 1 `FlashcardSrs(userId,cardId)` (đúng→good, sai→again) + 1 `FlashcardReviewLog(mode:'quiz')` + 1 `LearningEvent(source:'flashcard',subskill:'vocabulary')`. Học lại → **không** nhân đôi SRS row (AC-7).
  - Verify: `npm test -- quiz-flashcard-course`

- [ ] **T4 — Mastery vocabulary cập nhật, band khác bất biến**
  - Sau ghi LearningEvent → recompute (giống `quiz.service:139`). 
  - Test: sau vài answer → `StudentModel.mastery.vocabulary.value` > 0; các subskill/skill khác **không đổi** (AC-3/AC-8).
  - Verify: `npm test -- adaptive-mastery-attribution quiz-flashcard-course`

- [ ] **T5 — Complete theo lượt + score (không gate %)**
  - Thay `completeFlashcard`: xong 1 lượt bộ thẻ → `recordLesson(completed)`; trả `score = đúng/tổng`. Điểm thấp vẫn completed.
  - Test: làm hết lượt → `completed:true` + score đúng; điểm 1/5 vẫn completed (AC-4).
  - Verify: `npm test -- quiz-flashcard-course`

- [ ] **T6 — Đếm "số từ nắm được"/user**
  - Hàm/endpoint trả số thẻ system ở state review/mastered (`scheduler.isMastered`) cho 1 khóa/deck của user.
  - Test: seed vài SRS row mastered/không → count đúng (AC-5).
  - Verify: `npm test -- quiz-flashcard-course`

## FE — exe-web (PR riêng)

- [ ] **T7 — Course exercise runner dùng MCQ**
  - Exercise `flashcard` trong `learn` (course) render **MCQ** (tái dùng `features/flashcard/QuizRunner`), thay flip-complete cũ. Lesson `vocab_flashcard` vẫn flip (`FlipCardRunner`) để học.
  - Test: `flashcard-inline-runner.test.tsx` cập nhật — exercise gọi MCQ endpoint, submit per-thẻ, hiện score cuối; lesson flip không đổi.
  - Verify: `npm test -- flashcard-inline-runner exercise-host` (trong exe-web)

## Round 2 — mở rộng (chốt 2026-08-08: cá nhân hóa "từ yếu để bổ sung" + FE)

### BE (exe-api, cùng nhánh PRD-130)
- [ ] **T8 — `/answer` trả `correctMeaning`.** Thêm `correctMeaning: card.meaning` vào result của `answerFlashcardCard` → FE hiện đáp án đúng khi chọn sai. Test: sai → result có `correctMeaning`.
- [ ] **T9 — Endpoint list từ yếu (learner-wide).** `GET /courses/flashcard/weak` (không slug) → `{ result: { weak: [{cardId, term, meaning, ipa?, state, lapses}], total } }`. Nguồn: `FlashcardSrs(userId)` trên thẻ **system** chưa mastered (`!scheduler.isMastered`) ∪ review-log `correct=false` gần đây; cap ~50, sort theo độ yếu. Đặt route hợp lý (courses-level hoặc adaptive). Test: seed SRS yếu/mastered → trả đúng danh sách yếu.
- [ ] **T10 — Gỡ `vocabulary` khỏi feed recommend cấp-skill.** `adaptive.service.js` `getRecommendations`/`profileWeakSubskills`/`weakSubskillTargets`: loại subskill `vocabulary` (không sinh activity `route:null`). Mastery `vocabulary` vẫn lưu (chỉ không recommend cấp-skill). Test: user yếu vocab → không có activity vocabulary trong feed skill; `mastery.vocabulary` vẫn có.

### FE (exe-web, nhánh riêng `feat/PRD-130-course-flashcard-vocab-mcq`)
- [ ] **T11 — API client + types.** `services/quiz.service.ts` (getFlashcardQuiz/answerFlashcardCard/getFlashcardMastered/getWeakVocab) + `hooks/use-lesson-quiz.ts` + `constants/query-keys.ts` + `types/course.types.ts`. Bỏ `reviewedCardIds` khỏi completeFlashcard.
- [ ] **T12 — MCQ exercise runner.** `flashcard-quiz-inline-runner.tsx` (reuse `QuizRunner` visual): gọi `/quiz` → per-thẻ `/answer` (server-graded, hiện `correctMeaning` khi sai) → `/complete` hiện score. Dispatch: `exercise-host.tsx` `case "flashcard"` → runner MCQ (thay flip-exercise cũ). Test.
- [ ] **T13 — Flip = nội dung LESSON (QĐ-1a).** Render flip (`getFlashcardCards`) cho lesson type `vocab_flashcard` trong `LessonViewer`/PrimaryContent (mới). Test.
- [ ] **T14 — WeakVocabCard.** `components/features/adaptive/WeakVocabCard.tsx` "Từ vựng cần ôn" đọc `/flashcard/weak`, render ở Adaptive page. Test.

> **Content migration (không phải code):** bài cũ để flip ở exercise `flashcard` cần author lại thành lesson `vocab_flashcard` (flip) + exercise `flashcard` (MCQ). Flag cho content team — ngoài phạm vi code.

## Gate

- Sau T7: **pre-merge review** (human) — diff + đối chiếu Acceptance Criteria (spec §5). Không self-merge.

## Phụ thuộc chốt trước khi code
- OQ-1/OQ-2/OQ-3 (design §5) cần owner/tech-lead sign-off (itemCefr nguồn, XP, migrate flip-complete).
