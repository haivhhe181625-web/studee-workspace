---
feature: Kiểm tra vocab MCQ trong khóa học (tách học flip khỏi bài tập chấm điểm)
slug: course-flashcard-vocab-mcq
status: draft
date: 2026-08-08
owner: Vũ Hồng Hải
repos: [exe-api, exe-web]
tier: lite
brainstorm_ref: plans/reports/brainstorm-260808-1710-course-flashcard-mcq-vocab-signal-report.md
---

# Spec: Kiểm tra vocab MCQ trong khóa học

## 1. Mục tiêu

Flashcard trong khóa hiện chỉ **lật thẻ để học**, không sinh điểm/tín hiệu. Tách rõ:
- **Bài học** (`vocab_flashcard` lesson) = lật thẻ để học, như video/audio.
- **Bài tập** (`flashcard` exercise) = **kiểm tra vocab bằng MCQ**, chấm điểm per-thẻ → sinh tín hiệu **từ thuộc/chưa thuộc** per-user.

Tín hiệu này (a) nuôi **ôn tập thông minh M4-9** (round sau đọc `FlashcardSrs`), (b) cho học viên theo dõi **kỹ năng nhỏ "từ vựng"** (subskill `vocabulary` riêng, không nhét chung `use_of_english`).

## 2. Bối cảnh (đã verify trong code)

- `LESSON_TYPES` đã có `vocab_flashcard` (`course-content.model.js:20`); `EXERCISE_TYPES` có `flashcard` (`:15`) hiện = flip-complete.
- Course flip: `quiz.service.getFlashcardCards`/`completeFlashcard` (`:151-194`) — *"NO SRS, NO learning_events — flashcard does not feed mastery"*.
- MCQ có sẵn: `flashcard.service.generateQuiz`/`buildQuizOptions` (`:506-544`) — chiều EN→VI, min 2 thẻ, distractor co giãn (≤3), chấm per-thẻ, không ngưỡng %.
- SRS: `FlashcardSrs` (FSRS) keyed `(userId, cardId)`; `applyReviewEvent` cập nhật SRS + ghi `FlashcardReviewLog`. Standalone review scoped ownership (`getOwnedCard`) → **không đụng thẻ system**.
- Mastery: `LearningEvent`(correct/skill/subskill) → `computeMastery` → `StudentModel.mastery`. `SOURCES` chưa có `'flashcard'`; `SUBSKILLS` chưa có `'vocabulary'`; `knowledge-graph.subskillToSkillMap` map subskill→skill.

## 3. Phạm vi

### Trong phạm vi (v1)
- **IS-1** Đổi ngữ nghĩa exercise `flashcard` (course-side): từ flip-complete → **bài kiểm tra MCQ** trên system deck của lesson.
- **IS-2** Course MCQ path: tái dùng `buildQuizOptions`; đọc **system deck** (`loadSystemDeck`) thay `getOwnedDeck`. Chiều **EN→VI**, min 2 thẻ, distractor co giãn.
- **IS-3** Chấm per-thẻ → cập nhật `FlashcardSrs(userId, system cardId)` + ghi `FlashcardReviewLog`.
- **IS-4** Ghi `LearningEvent{ source:'flashcard', subskill:'vocabulary', correct, itemCefr }` → `computeMastery` cập nhật mastery `vocabulary`.
- **IS-5** Completion exercise = **làm hết 1 lượt bộ thẻ** (mỗi thẻ 1 lần); điểm = số đúng/tổng (hiển thị); **không gate %**.
- **IS-6** Đếm được **"số từ nắm được"/user** cho 1 khóa/deck (state review/mastered qua `FlashcardSrs`).
- **IS-7** FE web: exercise runner course dùng MCQ (tái dùng `QuizRunner`), thay flip-complete cũ.

### Ngoài phạm vi (lý do)
- **type (gõ lại) + match (ghép)** — v1 chỉ MCQ (owner: MCQ ổn định nhất); thêm sau. → khi có `type` mới tách "thuộc (recall)" khỏi "hiểu (recognition)".
- **M4-9 review-candidates** (chọn lọc ôn) — round sau; v1 chỉ dựng nguồn tín hiệu.
- **Hiện từ khóa ra app flashcard ngoài** (cross-surface 1 chiều) — v-next; v1 chỉ ghi dữ liệu.
- **Chiều MCQ linh hoạt / cấu hình đảo chiều** — v1 fixed EN→VI theo logic có sẵn.
- **Ngưỡng pass %** — bỏ, theo logic MCQ có sẵn (chấm per-thẻ).

## 4. Quyết định nghiệp vụ (owner chốt 2026-08-08)

| # | Câu hỏi | Chốt |
|---|---|---|
| Q1 | Tín hiệu nuôi gì | `FlashcardSrs` per-từ **+** subskill **`vocabulary`** riêng (không nhét `use_of_english`) |
| Q2 | Hiểu vs thuộc | Hoãn — v1 chỉ MCQ (recognition); tách khi thêm `type` |
| Q3 | Mode | **Chỉ MCQ**, EN→VI, tái dùng logic có sẵn |
| Q3 | Completion | Lesson flip học như video; exercise MCQ xong 1 lượt = completed, không gate % |
| Q4 | Scope | Chỉ tầng flashcard course-side; M4-9 sau |
| §5 | Trí nhớ | Course học **fresh** (không test-out); ghi SRS. Hiện ra app ngoài = v-next |
| §6 | Đủ thẻ | Theo logic có sẵn (min 2, distractor co giãn) |

## 5. Acceptance Criteria

1. **AC-1** (IS-1/2): Học viên mở exercise `flashcard` của 1 lesson → nhận bộ MCQ dựng từ **system deck** của lesson đó; mỗi câu: `term` (EN) + đáp án `meaning` (VI) đúng + ≤3 nhiễu (loại nhiễu trùng nghĩa). Deck 2–3 thẻ vẫn ra MCQ (2–3 đáp án).
2. **AC-2** (IS-3): Trả lời 1 thẻ → tạo/cập nhật đúng 1 row `FlashcardSrs(userId, cardId)` (thẻ system) + 1 `FlashcardReviewLog(mode:'quiz', correct)`. Đúng → rating `good`, sai → `again`.
3. **AC-3** (IS-4): Mỗi thẻ đã chấm → 1 `LearningEvent{ source:'flashcard', subskill:'vocabulary', correct, itemCefr }`; sau đó `StudentModel.mastery.vocabulary.value` cập nhật; **các mastery/band khác KHÔNG đổi**.
4. **AC-4** (IS-5): Làm hết 1 lượt bộ thẻ → lesson `completed`; response trả `score = đúng/tổng`. Điểm thấp vẫn `completed` (không gate %).
5. **AC-5** (IS-6): API/hàm trả **số từ nắm được** của user cho 1 khóa/deck = số thẻ ở state `review`/mastered (`scheduler.isMastered`).
6. **AC-6** (IS-2 boundary): Course MCQ **không** yêu cầu ownership thẻ (system deck không có `userId`); standalone owned-deck flow **không** bị ảnh hưởng.
7. **AC-7** (§5): Học lại 1 từ đã có SRS trong khóa → **vẫn học/làm bài bình thường** (không skip); SRS row cập nhật, không nhân đôi.
8. **AC-8** (regression/isolation): Enum mới (`'flashcard'` SOURCES, `'vocabulary'` SUBSKILLS) **không** phá `computeMastery`/attribution test hiện có. `vocabulary` **KHÔNG** wire vào `knowledge-graph` (`subskillToSkillMap['vocabulary']` = `undefined`). LearningEvent flashcard mang `skill:'use_of_english'` + `subskill:'vocabulary'` nhưng **bị loại khỏi gating skill** (`computeSkillMastery`/`skillsMeetingMastery` `$match` exclude `subskill:'vocabulary'`) → `use_of_english` checkpoint-readiness + course-skip **bất biến**. *(Sửa 2026-08-08: bản gốc ghi nhầm `subskillToSkillMap['vocabulary']=use_of_english`; đảo lại theo quyết định isolation đã duyệt — xem design §2.2.)*

## 6. Câu hỏi mở (chuyển design)

- Nguồn `itemCefr` cho LearningEvent flashcard: `deck.level` hay CEFR của phase chứa lesson?
- Course MCQ có cộng XP/gamification (`applyReviewEvent` đang cộng) không — hay path riêng không XP?
- Có seed/khóa nào đang dùng flip-complete cần migrate/giữ tương thích khi đổi nghĩa exercise `flashcard`?
