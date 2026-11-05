# Spec: Đánh giá độ sẵn sàng dữ liệu (Data Readiness Assessment)

- **Ngày:** 2026-08-27
- **Tác giả:** AI Brainstorm (từ khảo sát toàn hệ thống)
- **Repos/surfaces ảnh hưởng:** api (chính) · web · admin · lms · mobile — spec đánh giá, chưa động code repo nào
- **Module liên quan (nếu là `exe-api`):** assessment · course-content · grammar · adaptive · quiz · lms/class (điểm danh/bài tập/sổ điểm)
- **Trạng thái:** Nháp / Chờ BA review
- **Context gốc:** `../exe-api`… đầy đủ inventory tại `exe-web/../docs/feature-data-inventory.md` (repo gốc `Đồ án/docs/feature-data-inventory.md`)

## 1. Mục tiêu

Xác định rõ **dữ liệu nào còn thiếu / chưa sẵn sàng** để mở khóa các feature đang đóng & roadmap sắp tới, xếp ưu tiên, và chốt các quyết định nghiệp vụ chặn — làm cơ sở tách thành các spec feature con. **Đây là spec đánh giá/định hướng, KHÔNG implement.**

## 2. Bối cảnh

Khảo sát toàn hệ thống (2026-08-27) cho thấy phần lớn feature đã có route/model, nhưng chuỗi giá trị lõi **Đánh giá → Hồ sơ → Lộ trình → Lịch → học → mastery → checkpoint** bị nghẽn vì thiếu dữ liệu ở vài mắt xích. Đồng thời một số feature đang bị "đóng" (comment nav ở web, kill-switch mobile, hoặc chưa có schema ở LMS). Chi tiết đầy đủ: `docs/feature-data-inventory.md` (mục 5 = khoảng trống dữ liệu).

Hiện trạng đã kiểm chứng (không đoán):
- Đã có: ~78 Mongoose models; Adaptive Path ~95%; Lịch M4 v1/v2; Push infra FCM; Course ingestion; IPA đầy đủ.
- Chưa có / chưa sẵn sàng: nhãn chuẩn cho `assessment_questions`; band-scoring còn IELTS-gốc; dữ liệu grammar chưa import; model Phase 4 exercise engines; schema LMS điểm danh/bài tập/sổ điểm; track thời gian học thực; cut-score θ→CEFR còn placeholder.

## 3. Phạm vi

### Trong phạm vi

- Tổng hợp & xếp ưu tiên 4 cụm khoảng-trống-dữ-liệu (A1–A4 mục 6 dưới).
- Nêu rõ mỗi cụm cần: entity/field mới nào · dữ liệu nguồn nào cần ingest · quyết định nghiệp vụ nào chặn.
- Chốt danh sách quyết định nghiệp vụ (mục 5) làm điều kiện tách spec con.
- Sản phẩm đầu ra: spec này + `tasks.md` (các bước đánh giá/chuẩn bị dữ liệu có verify).

### Ngoài phạm vi

- Viết code, tạo migration, import dữ liệu thật — **Lý do:** đây là bước đánh giá; implement thuộc các spec feature con sau khi chốt ưu tiên.
- Thiết kế chi tiết schema từng model mới — **Lý do:** thuộc `design.md` của từng spec con.
- Feature không liên quan dữ liệu (UI polish, refactor) — **Lý do:** ngoài mục tiêu.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Product owner | Nhìn 1 bảng gap dữ liệu có ưu tiên | Quyết feature nào làm trước |
| Tech Lead | Biết entity/field còn thiếu mỗi cụm | Ước lượng & tách spec con |
| BA | Biết quyết định nghiệp vụ nào đang chặn | Chuẩn bị câu hỏi cho founder |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| M3 (đánh giá đầu vào) có phải ưu tiên #1? | Có — vì là "mồi" cho cả chuỗi hồ sơ/lộ trình/lịch | Product owner |
| "Speaking 1-1 với AI" giữ chat-scenario hay hội thoại có state/chấm điểm? | Chốt trước khi đầu tư data model Talk | Product owner |
| LMS 3 module (điểm danh/bài tập/sổ điểm) có vào phạm vi sắp tới? | Chờ 3 quyết định BA (nộp bài / tính điểm làm lại / quy đổi điểm thi) | BA + founder |
| Course: nguồn thật là `CourseStructure` hay legacy `Course`? Có cần FK marketing↔cây học? | Chốt `CourseStructure`; đánh giá nhu cầu FK | Tech Lead |

## 6. Acceptance criteria (tóm tắt)

1. Có bảng 4 cụm gap (A1 M3-nhãn-đề · A2 grammar-import · A3 Phase4-engines · A4 LMS-3-module) kèm: entity/field thiếu, dữ liệu nguồn cần, quyết định chặn.
2. Mỗi cụm gap map được về feature đang đóng/planned trong `feature-data-inventory.md`.
3. 4 quyết định nghiệp vụ (mục 5) được liệt kê rõ actor quyết & lựa chọn đề xuất.
4. Có thứ tự ưu tiên đề xuất giữa 4 cụm, có lý do (dependency, chặn chuỗi).
5. Nêu rõ cụm nào đủ điều kiện tách spec con ngay, cụm nào phải chờ quyết định nghiệp vụ.

## 7. Câu hỏi mở

- [ ] Product owner xác nhận M3 là ưu tiên #1 (điều kiện tách spec con `m3-assessment-item-labeling`).
- [ ] Định dạng dữ liệu nguồn ngữ pháp gốc (55 phần) — file gì, ai cung cấp?
- [ ] "quiz" cuối cùng của Phase 4 định nghĩa thế nào (Q-P4.2/3)?
- [ ] 3 quyết định BA cho LMS đã có lịch chốt chưa?

## 8. Ghi chú cho Tech Lead Design

- Spec này KHÔNG cần `design.md` (không quyết data model mới ở đây — chỉ liệt kê gap). Mỗi spec con tách ra sau sẽ có design riêng.
- Ràng buộc đã biết: `assessment_questions.answerKey` phải `select:false`; đa-tenant qua `organizationId` plugin; cut-score θ→CEFR chờ Angoff (mọi band hiện là ước lượng `Result.ciPlaceholder=true`).
- Nợ dữ liệu tín hiệu (mục 5B inventory: `lastActiveDate` bump thiếu, `studyPlan.timezone` chưa persist, mastery chỉ nhánh IPA) nên gom vào 1 spec con "data-signal-fixes" riêng, tránh trộn vào các cụm A.
