# Spec: Reading — import ngân hàng + Luyện tập Reading theo chủ đề

- **Ngày:** 2026-08-20
- **Tác giả:** AI Brainstorm (spec-workflow lite)
- **Repos/surfaces ảnh hưởng:** api (`exe-api`), web (`exe-web`). Admin: tái dùng vet sẵn có, không đổi.
- **Module liên quan (exe-api):** `modules/assessment` (bank), `modules/test-paper` (tham chiếu), module reading/luyện tập mới hoặc mở rộng (chốt ở design).
- **Trạng thái:** Quyết định nghiệp vụ đã chốt (§5) — chờ Tech Lead design sign-off
- **Nguồn:** `Dữ liệu/Reading/reading-bank-v1.json` (17 passage, 118 item); plan gốc `plans/260820-1057-reading-bank-import-and-practice/`.

## 1. Mục tiêu

Nạp ngân hàng reading có sẵn (JSON) vào bank assessment, và cho học viên **luyện đọc bằng cách duyệt theo chủ đề → chọn bài theo CEFR tự do**, làm bài được chấm và xem giải thích.

## 2. Bối cảnh

Đã kiểm tra code (không đoán):
- **Bank reading = module `assessment`**: `AssessmentStimulus` (passage: `body`, `difficultyTier` bắt buộc, `cefrBandHint`, `kind`, `contentSource`, `status`) + `AssessmentQuestion` (item: `stimulusId`, `skill`, `cefrLevel`, `itemType` (enum 23 loại), `topic` (enum coarse 8 giá trị), `targetGoal`, `answerKey`/`acceptedVariants` `select:false`, `explanation`, `irt.b`, `status`). Xác minh: `assessment-question.model.js:103-158`, `assessment-stimulus.model.js:43-75`.
- **Không có module reading riêng**; không code nào tham chiếu `reading-bank`/`rpsg_`/`ritm_`. Reading hiện chỉ tồn tại như stimulus/question `skill:'reading'`.
- **Luyện đề (`test-paper`)** tiêu thụ bank này; luyện tập reading (feature này) là surface mới, cùng bank.
- Engine chấm sẵn có: `assessment/grade.js`, `shuffle.js`.
- JSON có `difficulty`/`discrimination` = null (IRT chưa calibrate); độ khó neo ở `cefr` cấp passage.

## 3. Phạm vi

### Trong phạm vi

- **Import (api):** seed script ETL `reading-bank-v1.json` → `AssessmentStimulus` + `AssessmentQuestion`, `status:'draft'`, idempotent.
- **Vet:** tái dùng luồng duyệt sẵn có để `draft`→`active` (không viết mới).
- **Luyện tập (api):** endpoint learner — list chủ đề, list passage theo chủ đề (+ lọc CEFR), lấy passage detail (ẩn đáp án), nộp bài (chấm bằng `grade.js`, trả điểm + đáp án + giải thích).
- **Luyện tập (web):** trang list chủ đề → list bài (chip CEFR) → làm bài (render 3 kiểu: choice/fill/matching) → kết quả.

### Ngoài phạm vi

- **Lộ trình dễ→khó / mở tầng / ngưỡng đạt** — **Lý do:** owner đã chốt bỏ; điều hướng theo chủ đề, chọn tự do.
- **Adaptive/CAT theo IRT** — **Lý do:** `irt.b` chưa calibrate → chính xác giả; YAGNI.
- **Dedup Qdrant + moderation khi import** — **Lý do:** 2 gate đó cho nội dung AI-generated; đây là import nội bộ curated, vẫn giữ vet người.
- **Học theo chủ đề gắn course/unit** — **Lý do:** cần bank dày hơn mỗi chủ đề; thuộc phía course-content, tách sau.
- **Đưa câu sai vào Sổ lỗi (`learning_events`)** — **Lý do:** defer; không chặn feature.
- **Đổi schema model assessment** — **Lý do:** tái dùng field sẵn có (KISS).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Học viên | Duyệt reading theo chủ đề, chọn bài theo CEFR, làm và xem giải thích | Luyện đọc hiểu theo hứng thú, không bị ép lộ trình |
| Admin/examiner | Duyệt passage/item import trước khi hiển thị | Kiểm soát chất lượng nội dung |
| Ops/dev | Chạy script nạp bank | Đưa dữ liệu có sẵn vào hệ thống nhanh, an toàn |

## 5. Quyết định nghiệp vụ — ĐÃ CHỐT (owner, 2026-08-20)

| Câu hỏi | Chốt |
|---|---|
| `targetGoal` | `'ielts'` — **chỉ gắn cho item reading mới seed**. Question bank cũ **không động** (không backfill, không sửa doc sẵn có). |
| Chủ đề hiển thị | Nhóm **coarse** (`topic` enum 8 giá trị). |
| `contentSource` | `'self-authored'`. |

**Ràng buộc "không động bank cũ":** seed **thuần cộng thêm** + idempotent; mọi doc mới mang `provenance.importBatch:'reading-bank-v1'`. Không update/xoá/di chuyển doc `assessment` sẵn có.

## 6. Acceptance criteria (tóm tắt)

1. Script nạp đủ **17 Stimulus + 118 Question** ở `status:'draft'`, `skill:'reading'`, `targetGoal:'ielts'`; chạy lại **không nhân đôi** (idempotent).
2. Mọi Question có `stimulusId` đúng, `answerKey` đúng shape theo itemType, `cefrLevel`+`irt.b` khớp tier suy từ cefr; không itemType/answerKey nào bị bỏ âm thầm (log rõ nếu có).
3. Sau vet → `active`: API list chủ đề trả đúng các nhóm có bài; bấm 1 chủ đề trả passage kèm CEFR; lọc CEFR trong chủ đề hoạt động.
4. Học viên chọn **bài bất kỳ tự do** — không có gate/unlock/ngưỡng.
5. Làm được cả **3 kiểu** (choice/fill/matching); chấm đúng bằng `grade.js`; `answerKey`/`explanation` **chỉ hiện sau khi nộp**, không leak ở list/detail.
6. Full suite `exe-api` xanh, 0 regression assessment/test-paper.

## 7. Câu hỏi mở

- [x] Owner chốt §5 (2026-08-20).
- [ ] Attempt: tái dùng `PracticeAttempt` hay tạo attempt reading tối giản? → scout ở phase Development.
- [ ] (Xác nhận nhẹ) Item reading mới seed mang `targetGoal:'ielts'` → sau khi `active` **đủ điều kiện lọt đề luyện đề IELTS reading** (assemble lọc theo targetGoal+skill+active). Hiểu là OK (chung pool, đúng nhãn ielts). Nếu owner muốn reading-practice **tách hẳn** khỏi luyện đề thì báo — khi đó không set targetGoal ielts.

## 8. Ghi chú cho Tech Lead Design

- Mapping JSON→model + suy `difficultyTier`/`irt.b` từ cefr: xem `design.md`.
- Endpoint learner **không** đặt ở `/api/admin` (namespace học viên). Đúng pattern ẩn `answerKey`/`explanation` (`select:false`, chỉ trả ở submit) như exam-practice.
- Tham khảo tiền lệ learner-practice tách khỏi bank: `modules/listening/listening-practice.service.js`.
