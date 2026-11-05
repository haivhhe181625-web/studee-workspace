# Spec: Luyện đề (IELTS/TOEIC Exam Practice) — Phase 1

- **Ngày:** 2026-08-14
- **Tác giả:** AI Brainstorm → BA
- **Repos/surfaces ảnh hưởng:** `api` (chính — model + service + admin CRUD), `web` (UX luyện đề), `admin` (đóng gói đề). Mobile: không.
- **Module liên quan (exe-api):** module mới `services/api/src/modules/test-paper/`; tái dùng `modules/assessment` (item bank, exam runner data), `modules/review` (mistake-review layer), `constants/blueprint.js`.
- **Trạng thái:** Đã duyệt scope (§5 chốt 2026-08-14) — chờ Tech Lead Design
- **Nguồn:** brainstorm `plans/reports/brainstorm-260814-1528-luyen-de-ielts-toeic-report.md`

## 1. Mục tiêu

Cho phép learner **chọn 1 đề IELTS/TOEIC (Reading/Listening), làm có tính giờ, nộp và nhận điểm + đáp án + giải thích + phân tích điểm yếu theo dạng câu**, câu sai tự đẩy vào tầng ôn tập ngắt quãng (mistake-review) — mô phỏng thi thật, tái dùng engine assessment sẵn có.

## 2. Bối cảnh

Hiện trạng (đã kiểm code, không đoán):

- **Item bank có sẵn:** `AssessmentQuestion` (`modules/assessment/assessment-question.model.js`) — có `itemType` (mcq, true_false_ng, matching_headings, sentence_completion, labeling, cloze…), `cefrLevel`, `targetGoal` (`ielts`/`toeic`), `answerKey` (`select:false`), `irt`, `statistics`. `AssessmentStimulus` có `difficultyTier` (easy/mid/hard) + transcript/audio.
- **Engine thi có sẵn:** `modules/assessment` — `testType: mock_ielts|mock_toeic`, `assemblyMode: adaptive|fixed`, section timers, proctor. Chấm khách quan `grade.js`. Exam UI `exe-web` (`ExamContainer`, `QuestionCard`, `ExamTimer`, Zustand `exam.store`).
- **Pattern freeze items có sẵn:** `LessonQuiz` (`modules/quiz/lesson-quiz.model.js`) reference `AssessmentQuestion` ids, đóng băng `resolvedItemIds` lúc publish — pattern tái dùng cho "đóng gói đề".
- **Mistake-review có sẵn (spec):** `specs/mistake-review-layer/` — module `review/`, collection `review_schedule`, Leitner SR, đọc từ `learning_events`/`quiz_attempts`.
- **AI sinh item có sẵn:** `gen-testlet.service.js` + admin `question-bank.admin.js`.

**CHƯA có (khoảng trống P1 phải lấp):**
- Entity "Đề thi" persistent full-length có tên, tái sử dụng (hiện fixed đề ráp runtime từ testlet, không lưu thành paper).
- **Khung đề (blueprint) full-length**: `constants/blueprint.js` hiện chỉ là POOL-coverage matrix per skill×CEFR cho Diagnostic/CAT — **không có** khung 40-câu IELTS Reading / 100-câu TOEIC theo section+itemType.
- **Trường `explanation`/giải thích** trên item (`AssessmentQuestion` không có field này) — cần cho review sau nộp.
- Luồng "practice attempt" (làm đề → nộp → xem đáp án). Engine hiện thiên placement/diagnostic (che đáp án).

Ý tưởng "Luyện đề" = UX layer + entity mỏng phủ lên engine sẵn có, KHÔNG xây engine mới, KHÔNG scrape đề, KHÔNG tách product clone riêng.

## 3. Phạm vi

### Trong phạm vi (Phase 1)

- **Khung đề (blueprint) full-length** cho: IELTS Reading (40 câu/3 passage), IELTS Listening (40 câu/4 section), TOEIC Listening (100 câu/P1-4), TOEIC Reading (100 câu/P5-7). Khai báo section + itemType distribution + difficulty/CEFR mix + timeLimit. Gắn nhãn dải mục tiêu cả paper (band/CEFR).
- **Entity `TestPaper`** (mode `static` cho MVP): `{exam, skill, blueprintId, frozenItemIds[], targetBandRange, timeLimit, status}`. Đóng băng danh sách item khi publish (pattern `LessonQuiz`).
- **Thêm field `explanation`** trên `AssessmentQuestion` (giải thích đáp án; hiển thị sau nộp).
- **`PracticeAttempt`**: làm đề timed → nộp → chấm khách quan (`grade.js`) → trả điểm + đáp án + giải thích + breakdown đúng/sai theo `itemType`.
- **Author item TOEIC** (~200 item P1-7) qua AI-gen (`gen-testlet`) + examiner vet — **prerequisite** vì bank hiện RỖNG TOEIC (verified: seed bank 0 câu toeic). Cần bổ sung TOEIC-specific itemTypes vào `ITEM_TYPES` nếu mcq/cloze không phủ đủ (vd Part 1 photograph).
- **Đẩy câu sai vào mistake-review (chỉ TRACK ở P1)**: practice attempt emit `learning_events` để `modules/review` nạp câu sai vào `review_schedule`. **Re-drill item passage (testlet) trong SRS = KHÔNG ở P1** (verified: review session chỉ chơi item `stimulusId==null`, testlet deferred). Câu passage vẫn xem lại được ở trang review-sau-nộp (đáp án+giải thích).
- **Admin "đóng gói đề"**: chọn/AI-gen item theo blueprint → freeze → publish (`test-paper.admin.js`, delegate về service).
- **Web UX luyện đề**: danh sách đề → làm (reuse exam runner) → trang kết quả review (điểm, đáp án, giải thích, điểm yếu).
- **Nội dung seed**: AI-gen (`gen-testlet`) + examiner vet, 100–200 item khởi đầu.

### Ngoài phạm vi

- **Re-drill item passage (testlet) trong SRS** — **Lý do:** review module hiện chỉ chơi item standalone (`stimulusId==null`); mở testlet-review đụng module đang ổn định, rủi ro hồi quy. P2. P1 chỉ track + xem lại ở trang review-sau-nộp.
- **Micro-drill theo itemType** (vd 20 câu Matching Headings) — **Lý do:** P2; cần item bank đủ lớn theo từng dạng. Documented, không làm P1.
- **General Training** — **Lý do:** P1 IELTS Academic only (tệp chính VN); General sau.
- **Entitlement/quota gating** — **Lý do:** P1 free toàn bộ để lấy traction; tier hóa sau.
- **IELTS Writing + Speaking chấm AI** — **Lý do:** P3; phụ thuộc `llm /score/*`, tuning độ chính xác band ±0.5, double-scoring. Documented.
- **Dynamic/adaptive assembly** (mode `dynamic` của TestPaper) — **Lý do:** P3; cần bank calibrate IRT (~500+ item). MVP chỉ `static`.
- **Scrape/clone đề từ web + tool lấy đề** — **Lý do:** rủi ro bản quyền Cambridge/ETS; đã có nguồn hợp pháp (AI-gen+vet).
- **Dự đoán band/score chính thức từ mock** — **Lý do:** cần calibration; MVP chỉ hiển thị raw + % theo dạng, không quy band.
- **TOEIC Speaking/Writing** — **Lý do:** ít phổ biến ở VN; sau P3.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Chọn 1 đề IELTS/TOEIC R/L, làm có giờ | Mô phỏng thi thật, tự đánh giá |
| Learner | Sau nộp xem đáp án + giải thích + dạng câu mình yếu | Học từ lỗi, biết chỗ cần cải thiện |
| Learner | Câu sai tự vào sổ ôn ngắt quãng | Ôn trúng chỗ yếu, không phải tự quản lý |
| Content admin | Đóng gói 1 đề từ item bank (AI-gen/soạn) theo khung | Tạo đề hợp pháp, đúng format, tái dùng |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Đã chốt (2026-08-14) | Người quyết |
|---|---|---|
| IELTS Academic hay có cả General Training? | **Academic only** ở P1 | owner ✅ |
| TOEIC trong P1 (bank rỗng TOEIC) | **Có** — chấp nhận author ~200 item TOEIC trước khi đóng gói đề | owner ✅ |
| Câu sai passage vào SRS | **Chỉ track ở P1**, re-drill passage để P2 | owner ✅ |
| Monetization ở P1 | **Free toàn bộ**, không gating | owner ✅ |
| Số đề seed tối thiểu để launch | 2 IELTS (R+L) + 2 TOEIC (R+L) = **4 đề** (mặc định, owner chỉnh được) | owner |
| UI ngôn ngữ | **Vietnamese-first** (giải thích tiếng Việt) | owner |

## 6. Acceptance criteria (tóm tắt)

1. Có registry **khung đề** khai báo được 4 loại paper (IELTS R, IELTS L, TOEIC L, TOEIC R) với section + itemType + count + timeLimit; validate được 1 tập item có khớp khung không.
2. Admin đóng gói 1 `TestPaper` static từ item bank theo khung → publish → `frozenItemIds` cố định, ai làm cũng cùng đề.
3. `AssessmentQuestion` có field `explanation`; admin nhập được; chỉ hiện sau khi nộp (không leak lúc đang làm, giống `answerKey`).
4. Learner làm đề timed → nộp → nhận điểm khách quan (`grade.js`) + đáp án + giải thích + breakdown % đúng theo `itemType`.
5. Sau nộp, câu sai (R/L) emit `learning_events` → `modules/review` **nạp (track) được** vào `review_schedule`. Re-drill item passage trong SRS KHÔNG bắt buộc ở P1 (deferred); item passage vẫn xem lại được ở trang review-sau-nộp.
6. Author được ≥4 đề (2 IELTS R+L, 2 TOEIC R+L) từ item bank, kể cả item TOEIC mới AI-gen+vet.
7. Không hồi quy: engine assessment (placement/diagnostic, CAT, exam runner) + module review hiện tại không đổi hành vi; toàn bộ test cũ xanh.

## 7. Câu hỏi mở

Không còn câu chặn approval. Verify đã xong (2026-08-14):
- ✅ `modules/review` đã implement thật (model+service+scheduler+controller+routes+tests) — nhưng chỉ re-drill item standalone; testlet deferred (đã phản ánh vào §3).
- ✅ TOEIC bank rỗng item (seed 0 câu) — đã đưa "author item TOEIC" vào §3 in-scope.
- ⚠️ Số item TOEIC thật ở prod DB: chạy `scripts/ops/bank-status.js` (runtime, cần DB env) để xác nhận trước khi seed — không chặn design.

## 8. Ghi chú cho Tech Lead Design

- **Khung đề mới ≠ `blueprint.js` hiện tại.** `constants/blueprint.js` là POOL-coverage cho Diagnostic/CAT (items per skill×CEFR), KHÔNG phải full-length paper. Cần cấu trúc khung mới (section-based) — cân nhắc đặt cạnh nhưng tách rõ, đừng ghi đè.
- **Freeze paper phải atomic single-doc** (MongoDB standalone, no multi-doc txn) — theo pattern `LessonQuiz` publish.
- **`explanation` hiển thị-sau-nộp**: KHÔNG để `select:false` (cần trả cho client ở trang review) nhưng controller phải chặn không trả trong lúc đang làm đề. Khác cơ chế `answerKey` (`select:false` tuyệt đối).
- **Emit `learning_events` từ practice attempt**: enum `learning_events.source` (`adaptive/learning-event.model.js:13`) hiện KHÔNG có `'practice'`, và `review.service.js:33 enqueueMistakes` hard-code `source:'quiz'`. → Tech Lead chọn: (a) thêm `'practice'` vào enum + mở `enqueueMistakes` đọc cả `practice` (sạch nghĩa, đụng 2 module), hoặc (b) practice emit `source:'quiz'` (tái dùng ngay, lỏng nghĩa; `enqueueMistakes` resolve refId qua `QuizAttempt` → practice attemptId không khớp → courseId/phaseKey null, chấp nhận được). Khuyến nghị (a).
- **Ràng buộc testlet-review**: `review.service.js` (159/195/259) chỉ serve item `stimulusId==null`. Câu passage được enqueue nhưng không vào session re-drill → P1 chấp nhận (track-only). KHÔNG sửa review module để mở testlet ở P1.
- **TOEIC itemTypes**: `ITEM_TYPES` (`assessment-question.model.js:24`) đang IELTS-flavored. Kiểm khi author: Part 1 photograph / Part 2 q-response có map gọn vào `mcq` không, hay cần thêm enum. Thêm enum = surgical, nhưng cân nhắc trước khi gen hàng loạt.
- **Admin theo 5-file + factory** (`_crud.factory.js`), business logic ở service `modules/test-paper/`, KHÔNG ở `admin-api/` (HARD RULE `exe-api/CLAUDE.md`).
- **Web/admin ở repo riêng** (`.ai/project-context.md` §0) — 1 spec này name cả `api`/`web`/`admin` nhưng implementation FE đi PR riêng từng repo.
- Ràng buộc DRY: KHÔNG chép item; `TestPaper` chỉ giữ `frozenItemIds` reference như `LessonQuiz`.
