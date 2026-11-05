# Tasks: Đánh giá độ sẵn sàng dữ liệu

> Spec-only (lite). Task = bước đánh giá/chuẩn bị dữ liệu, KHÔNG implement feature. Mỗi task có cách verify.
> Nguồn: `spec.md` + `../../../docs/feature-data-inventory.md` (repo `Đồ án/docs/`).

## Cụm A — Gap chặn tiến độ

- [ ] **T1. Kiểm kê nhãn `assessment_questions` (A1/M3).**
  Verify: query đếm % item thiếu `skill`/`itemType`/`targetCefr`/`targetGoal`; xuất bảng thiếu theo skill.
- [ ] **T2. Xác nhận `band-scoring` còn IELTS-gốc (A1/M3-3).**
  Verify: đọc `band-scoring.js`, chỉ ra chỗ hardcode thang IELTS; liệt kê hàm cần đổi sang CEFR-gốc.
- [ ] **T3. Chốt định dạng nguồn dữ liệu ngữ pháp 55 phần (A2).**
  Verify: có mẫu 1 phần đúng block-model 13 type + pool câu hỏi tối thiểu cho SRS ⑤; nguồn file & người cấp xác định.
- [ ] **T4. Đối chiếu model Phase 4 exercise engines (A3).**
  Verify: xác nhận `ExerciseRef.type` hiện chỉ `['ipa','talk']`; liệt kê entity net-new cần (`QuizSet`/`WritingTask`/`SpeakingTask`/`ExerciseProgress`).
- [ ] **T5. Xác nhận LMS 3 module chưa có schema (A4).**
  Verify: grep model `ClassSession`/`AttendanceRecord`/`Assignment`/`Submission` → 0 kết quả; ghi lại 3 quyết định BA đang chặn.

## Cụm B — Nợ dữ liệu tín hiệu (gom spec con riêng)

- [ ] **T6. Xác nhận `lastActiveDate` chỉ bump qua `recordLesson`.**
  Verify: grep writer của `streak.lastActiveDate`; liệt kê hoạt động standalone (flashcard/quiz/assessment) KHÔNG bump.
- [ ] **T7. Xác nhận `studyPlan.timezone` chưa được client persist.**
  Verify: kiểm tra web/mobile có gọi `PUT prefs{studyPlan:{timezone}}` không; nếu không → reminder fallback Asia/Saigon.
- [ ] **T8. Xác nhận mastery mới đạt ở nhánh IPA.**
  Verify: kiểm tra quiz/flashcard cap ở `completed` (không lên `mastered`); ghi ảnh hưởng tín hiệu nạp hồ sơ.

## Tổng hợp & quyết định

- [ ] **T9. Lập bảng ưu tiên 4 cụm A1–A4 kèm dependency.**
  Verify: bảng có cột (cụm · feature liên quan · entity/field thiếu · quyết định chặn · ưu tiên · lý do).
- [ ] **T10. Chốt 4 quyết định nghiệp vụ (spec §5) với product owner/BA.**
  Verify: mỗi câu hỏi có câu trả lời + người quyết; câu hỏi mở spec §7 đóng hết.
- [ ] **T11. Đề xuất tách spec con.**
  Verify: danh sách slug spec con đủ điều kiện tạo ngay (vd `m3-assessment-item-labeling`, `data-signal-fixes`) vs chờ quyết định.

## Human gate

- [ ] BA review spec + bảng gap (không có `design.md` nên bỏ gate design-signoff).
- [ ] Product owner chốt ưu tiên → mở tách spec con.
