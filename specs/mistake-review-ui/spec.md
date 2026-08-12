# Spec: Két lỗi sai Phase 2 — REST + UI "Sổ lỗi sai"

- **Ngày:** 2026-08-13
- **Tác giả:** AI Brainstorm → BA
- **Repos/surfaces ảnh hưởng:** `api` (REST cho module `review` đã có) + `web` (màn "Sổ lỗi sai" + phiên ôn). `admin`: **không**.
- **Module liên quan:** `exe-api/services/api/src/modules/review/` (thêm controller/routes) · `exe-web/src/services` + `src/hooks` + `src/app`.
- **Trạng thái:** Nháp — chờ BA review
- **Tiền đề:** Phase 1 = PRD-142 (tầng service `review` đã ship: `enqueueMistakes/recordReview/getReviewQueue`). Phase 2 phơi REST + dựng UI để học viên tự ôn.

## 1. Mục tiêu

Cho học viên **xem "mình sai ở đâu"** và **tự ôn lại** các câu sai theo lịch giãn ngắt quãng, qua một màn hình "Sổ lỗi sai" trên exe-web — nối vào engine `review` đã có ở Phase 1.

## 2. Bối cảnh

- Phase 1 đã có service `review` nhưng **không REST, không UI** — hiện chưa ai gọi được từ client.
- Pattern exe-api REST: `verifyToken` + `asyncHandler`, envelope `{ <key>: ... }` (xem `modules/quiz/quiz.routes.js`/`quiz.controller.js`).
- Pattern exe-web: `src/services/<x>.service.ts` (gọi API) + `src/hooks/use-<x>.ts` (React Query) + trang trong `src/app`. Có sẵn `quiz.service.ts` + `use-lesson-quiz.ts` để tham khảo.
- Chấm điểm objective đã có ở `modules/quiz/quiz.grading.js` (`gradeObjective`) — tái dùng, KHÔNG viết lại.

## 3. Phạm vi

### Trong phạm vi
- **exe-api** — REST cho module `review` (mount `/api/review`, `verifyToken`):
  - Xem danh sách két lỗi sai của chính user (stem câu hỏi + kỹ năng + trạng thái + lịch ôn).
  - Lấy bộ câu cho 1 phiên ôn (câu hỏi ẩn đáp án).
  - Nộp 1 câu trong phiên ôn → **server chấm** → cập nhật SR.
- **exe-web** — màn "Sổ lỗi sai": danh sách câu sai + trạng thái (đang ôn / đã thạo) + nút "Ôn ngay" mở phiên ôn (tái dùng player quiz), phản hồi đúng/sai từng câu.

### Ngoài phạm vi
- **Tin `correct` từ client** — **Lý do:** bảo mật; server phải tự chấm (xem §5, quyết định đã chốt).
- **Writing/Speaking/IPA trong sổ lỗi sai** — **Lý do:** band-level, chưa có "câu sai"; Phase 3.
- **exe-admin** — **Lý do:** không có gì để admin quản trị (dữ liệu derive tự động).
- **Rewire buổi ôn cuối tuần (M4-9)** — **Lý do:** là task M4-9; Phase 2 chỉ làm ôn chủ động do học viên bấm.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Mở "Sổ lỗi sai" xem mình sai ở đâu | Biết điểm yếu, chủ động ôn |
| Learner | Bấm "Ôn ngay" làm lại câu sai đến hạn | Sửa lỗi, được hệ thống giãn lịch khi làm đúng |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Đề xuất | Người quyết |
|---|---|---|
| Chấm điểm phiên ôn ở đâu? | **Server** (nhận `response`, `gradeObjective`, suy `correct`, gọi `recordReview`) — client KHÔNG gửi đúng/sai | (bảo mật — mặc định server) |
| Luồng phiên ôn | Từng-câu-một, phản hồi đúng/sai ngay sau mỗi câu | owner |
| Sổ lỗi sai hiển thị | Cả câu đang ôn + câu đã thạo (tab lọc) | owner |
| Câu dạng testlet (đọc/nghe có ngữ liệu) | Phase 2 chỉ ôn câu đứng-một-mình; hoãn testlet | owner |
| Vị trí trong điều hướng exe-web | Cần product chỉ định (mục "Ôn tập"?) | owner |

## 6. Acceptance criteria (tóm tắt)

1. GET danh sách sổ lỗi sai trả về câu của chính user kèm stem/kỹ năng/trạng thái/nextReviewAt — **không lộ đáp án**.
2. GET phiên ôn trả bộ câu đến hạn (Tier1→Tier2) đã ẩn đáp án.
3. POST nộp 1 câu với `response` → server chấm → trả `{correct, status, nextReviewAt}`; làm đúng → nextReviewAt giãn ra, sai → đến hạn lại sớm.
4. Client **không** thể tự khai đúng/sai (bỏ qua mọi `correct` client gửi).
5. Màn "Sổ lỗi sai" render danh sách + mở được phiên ôn + hiển thị phản hồi từng câu.
6. Chỉ đọc/ghi dữ liệu của chính user (verifyToken); không cross-user.

## 7. Câu hỏi mở

- [ ] Vị trí "Sổ lỗi sai" trong điều hướng exe-web (product quyết).
- [ ] 4 quyết định §5 (trừ chấm-server đã chốt) — xác nhận.

## 8. Ghi chú cho Tech Lead Design

- Tái dùng `gradeObjective` + đường serve item ẩn đáp án của quiz (`.select('-answerKey')`, `buildTestletStimuli`), KHÔNG copy.
- `recordReview` hiện nhận `correct:boolean` (nội bộ) — REST bọc thêm 1 lớp chấm ở server, giữ `recordReview` nguyên.
- Không thêm business logic vào admin-api.
