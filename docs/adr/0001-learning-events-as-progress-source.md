<!-- Tiếng Việt — theo format có tiền lệ ở <API_REPO>/docs/ADR-admin-api-migration.md -->
# ADR-0001 — Dùng `learning_events` làm nguồn sự thật duy nhất cho tiến độ học tập

**Ngày:** 2026-07-14 · **Trạng thái:** Đề xuất · **Thay thế:** không có

## Bối cảnh

Feature "theo dõi tiến độ kỹ năng theo phase" (`specs/skill-progress-tracking/`) cần một nguồn dữ liệu để tính "học viên đã luyện gì, điểm bao nhiêu, ở phase nào". Khảo sát code (`exe-api`) cho thấy hiện trạng phân mảnh:

- `learning_events` (append-only) đã tồn tại và **đã** được `computeMastery` tổng hợp vào `student_models.mastery`. Nhưng **chỉ 2 nguồn ghi event**: assessment và IPA (`speaking_drill`); `source:'lesson'/'quiz'` khai báo trong enum nhưng chưa nơi nào ghi.
- IPA khi ghi event chỉ lưu `correct = score>=60`, **vứt mất điểm số 0–100 thực tế**.
- `learning_events` **không có** liên kết tới phase của `user_roadmaps`.
- Module `course` không chấm điểm, không có route → bài luyện thường ngày (ngoài IPA) không sinh tín hiệu nào.

Nếu mỗi feature tiến độ/khuyến nghị tự tạo log riêng, hệ thống sẽ có nhiều con số mâu thuẫn cho cùng một kỹ năng, và mỗi nguồn nội dung mới lại phải sửa nhiều nơi.

## Quyết định

1. **`learning_events` là source-of-truth duy nhất** cho "một bài đã chấm". Mọi tính năng tiến độ/mastery/khuyến nghị **đọc từ đây** (hoặc từ `student_models` đã tổng hợp từ đây), **không** tạo log song song.
2. **Chuẩn hóa điểm về thang 0–100 tại write time** qua field `score` trên event. Nguồn nào có điểm số thật (IPA 0–100) ghi vào `score`; nguồn chỉ có đúng/sai ghi `correct` và để `score=null`. "Điểm trung bình" chỉ tính trên `score != null`.
3. **Gắn ngữ cảnh phase vào event tại write time** qua `roadmapId` + `phaseId` (phase active lúc học). Không suy ngược phase theo thời gian (vì vòng đời phase chưa được track).
4. **Nguồn luyện tập mới hòa vào bằng cách emit `learning_event` đúng dạng** (kèm `score` + phase context) — tầng đọc tiến độ là generic, **không** phải sửa khi thêm nguồn.

## Lộ trình

| Phase | Nội dung | Trạng thái |
|---|---|---|
| 0 | Thêm `score`/`roadmapId`/`phaseId` + index vào `learning_events`; IPA ghi điểm thật + phase | ☐ |
| 1 | Tầng đọc `getSkillProgress` + endpoint `GET /api/adaptive/progress` | ☐ |
| 2 | (Tương lai, ngoài feature này) Nguồn khác (course/quiz) chấm điểm → emit event; logic thăng phase để `phaseId` đa dạng | ☐ |

## Hệ quả

- `learning-event.model.js` thêm 3 field nullable (không migration). `adaptive.service.ingestIpaAttempt` sửa để ghi `score` + phase context.
- Ràng buộc mới cho mọi nguồn nội dung tương lai: muốn xuất hiện trong tiến độ/mastery thì **phải** emit `learning_event` chuẩn — ghi vào checklist khi thêm module luyện tập mới.
- Giới hạn chấp nhận ở thời điểm ra mắt: chỉ IPA + assessment có dữ liệu; per-phase progress tập trung ở phase active đầu tiên cho tới khi có logic thăng phase (xem `specs/skill-progress-tracking/design.md` §9).
- Không supersede ADR nào. Không thêm dependency/service ngoài.
