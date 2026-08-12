# Spec: Két lỗi sai — Mistake tracking & smart review layer

- **Ngày:** 2026-08-13
- **Tác giả:** AI Brainstorm → BA
- **Repos/surfaces ảnh hưởng:** `api` (chỉ backend, Phase 1). `web`/`admin`: **không** ở Phase 1.
- **Module liên quan (exe-api):** module mới `services/api/src/modules/review/`; đọc dữ liệu từ `modules/adaptive` (learning_events) + `modules/quiz` (quiz_attempts); SR tự viết trong module (Leitner, không phụ thuộc module khác).
- **Trạng thái:** Nháp — chờ BA review
- **Nguồn:** brainstorm `plans/reports/brainstorm-ket-loi-sai-review-layer-260813-0035-report.md`
- **Plane:** unblock PRD-119 (M4-9 · Ôn tập thông minh v3), PRD-120 (M4-10 · remediation).

## 1. Mục tiêu

Cung cấp một **tầng backend** trả về "danh sách câu học viên cần ôn lại" theo thứ tự ưu tiên (câu đang sai trước, giãn ngắt quãng), để M4-9 dùng thay cho buổi ôn "gom tất cả bài tập cuối tuần" hiện tại.

## 2. Bối cảnh

Hiện trạng (đã kiểm code, không đoán):
- **Đã có dữ liệu đúng/sai từng câu quiz:** `quiz_attempts.perItem[]` (`{questionId, correct}`) và `learning_events` (mỗi câu 1 event có `correct/skill/subskill`). Khi học viên nộp lại 1 quiz, `quiz.service.js:147` **xoá event cũ của quiz đó rồi ghi mới** → `learning_events` với `source:'quiz', correct:false` = **câu đang sai (trạng thái mới nhất)**.
- **Đã có tiền lệ SR trong repo:** `flashcard-scheduler.service.js` (wrapper `ts-fsrs`) cho flashcard — tham khảo pattern, nhưng két lỗi sai **tự viết Leitner** riêng (interval cố định, minh bạch cho product).
- **Đã có điểm nối lịch:** `lesson-schedule.generator.js` → `buildReviewDay(weekLessons)` gom **toàn bộ** exercise của tuần vào ngày ôn (chính là "v1" M4-9 muốn thay).
- **CHƯA có:** lớp lịch "câu nào đến hạn ôn, đã ôn đạt chưa" cho câu sai; và service ghép câu-sai + câu-ôn-duy-trì thành hàng đợi ôn.

Ý tưởng "két lỗi sai" = lớp mỏng phủ lên dữ liệu đã có, KHÔNG xây lại từ đầu.

## 3. Phạm vi

### Trong phạm vi (Phase 1)
- Collection mỏng `review_schedule` giữ trạng thái SR cho từng câu-đang-được-ôn của mỗi user.
- Service `review`:
  - **Nạp câu sai:** đọc câu đang sai (quiz) từ `learning_events` → tạo/cập nhật row `review_schedule` (lazy).
  - **Ghi kết quả ôn:** nhận (userId, questionId, đúng/sai) → cập nhật SR (đúng→`good`, sai→`again`) → graduate khi `isMastered`.
  - **Lấy hàng đợi ôn:** trả câu theo thứ tự **Tier 1** (câu sai đến hạn) → **Tier 2** (câu đã từng làm, không phải câu sai — duy trì).
- Hàm/endpoint nội bộ để M4-9 generator gọi được.

### Ngoài phạm vi
- **UI sổ lỗi sai (web/admin)** — **Lý do:** định vị "data layer trước, UI sau"; Phase 2.
- **Writing/Speaking/IPA** — **Lý do:** band-level, không có "câu sai" rời rạc; cần định nghĩa "lỗi" riêng; Phase 3.
- **Tier 3 "bài tập áp dụng biến tấu"** (câu cùng kiến thức nhưng đổi tình huống) — **Lý do:** cần ngân hàng item lớn / sinh item; note lại cho phase sau.
- **Rewire `buildReviewDay`/generator** — **Lý do:** đó là việc của M4-9 (PRD-119); Phase 1 chỉ cung cấp service để M4-9 tiêu thụ.
- **Chọn interval cố định `[1,3,7,14,30]`** — **Lý do:** dùng FSRS có sẵn thay vì tự viết Leitner (xem design §2).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Buổi ôn cuối tuần ưu tiên câu mình đã sai | Ôn trúng chỗ yếu, không phí thời gian làm lại câu đã thạo |
| Hệ thống lịch (M4-9) | Gọi 1 service lấy "câu cần ôn của user X" | Sinh buổi ôn thông minh thay vì gom tất cả |
| Vòng remediation (M4-10) | Lấy câu yếu điểm khi fail checkpoint | Chèn buổi ôn có trọng tâm trước khi thi lại |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Đã chốt | Người quyết |
|---|---|---|
| SR engine | **Tự viết Leitner** bậc thang `[1,3,7,14,30]` ngày (đúng→lên bậc, sai→về bậc 0). Interval cố định, minh bạch, product chỉnh trực tiếp | owner ✅ |
| Tier 2 "bài đã học" ở Phase 1 | Câu **đã từng làm** (`quiz_attempts`) và không đang sai — proxy đơn giản, không kéo course-progress | owner ✅ |
| Giới hạn hàng đợi | Tham số `limit` mặc định **20**, caller (M4-9) tự truyền; không hard-cap | owner ✅ |
| Nạp câu sai khi nào | `getReviewQueue` **tự gọi** `enqueueMistakes` (idempotent) → 1 điểm vào duy nhất | owner ✅ |

## 6. Acceptance criteria (tóm tắt)

1. Sau khi user sai câu Q ở 1 quiz và nộp, gọi service nạp câu sai → `review_schedule` có row `(user, Q)` trạng thái active, due ngay.
2. Gọi "lấy hàng đợi ôn" cho user → trả câu sai đến hạn (Tier 1) trước, sau đó lấp bằng câu đã-từng-làm không-sai (Tier 2), tôn trọng `due`.
3. Ghi kết quả ôn **đúng** → `due` của câu giãn ra (ngày kế xa hơn); ghi **sai** → câu đến hạn lại sớm.
4. Câu ôn đúng đủ nhiều (đạt `isMastered`) → chuyển `graduated`, không còn trong hàng đợi Tier 1.
5. Nạp câu sai **idempotent**: gọi lại nhiều lần không tạo trùng row cho cùng `(user, Q)`.
6. Không hồi quy: đường submit quiz + chấm điểm hiện tại không đổi hành vi; toàn bộ test cũ vẫn xanh.

## 7. Câu hỏi mở

Không còn — 4 quyết định ở §5 đã chốt.

## 8. Ghi chú cho Tech Lead Design

- Ràng buộc DRY: **không** chép lại dữ liệu đúng/sai — chỉ derive từ `learning_events`/`quiz_attempts`. `review_schedule` chỉ giữ trạng thái SR (thứ không derive được).
- `learning_events` index theo `{userId, skill, createdAt}`, **không** theo `questionId` — cân nhắc khi viết query "câu đang sai".
- Không đặt business logic ở admin-api. Module `review/` theo 5-file pattern.
- Đây là tầng bị M4-9/M4-10 phụ thuộc → giữ interface service ổn định, đặt tên rõ.
