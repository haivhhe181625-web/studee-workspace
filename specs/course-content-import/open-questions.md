# Câu hỏi cần chốt + Khuyến nghị của Tech Lead — course-content-import (Phase 1)

- **Spec:** `specs/course-content-import/spec.md` · **Design:** `design.md`
- **Ngày:** 2026-07-15 · **Người gửi:** Tech Lead Agent
- **Trạng thái:** ✅ **PO đã chốt toàn bộ ngày 2026-07-15** — xem cột "PO chốt" bên dưới. Design/data-model/contract/
  ADR/acceptance/spec đã cập nhật theo.
- **Mục đích:** gom các câu hỏi mở (Q1–Q6 của spec) + 1 câu hỏi kỹ thuật MỚI phát hiện khi đọc code (Q-Quiz),
  kèm **phương án khuyến nghị** để BA/Product Owner chốt.

| # | Câu hỏi | Khuyến nghị | ✅ PO chốt (2026-07-15) |
|---|---|---|---|
| **Q1** | "Course Catalog" = `course` hay kho mới? | Thực thể miền mới | **Thực thể miền mới** (`course_structures`) — trùng khuyến nghị |
| **Q2** | Excel / JSON / cả hai? | Excel (nếu học thuật tự soạn) | **Chỉ Excel `.xlsx`** + `exceljs` (giải phóng đội non-tech Epic 4) |
| **Q3** | Chấp nhận draft/archived? | Chỉ published | **Chỉ published** — trùng khuyến nghị |
| **Q4** | Trùng khóa? | Từ chối | **Từ chối**, HTTP 400 + lỗi trỏ dòng ("Dòng 45: ID 'mod-01' đã tồn tại") |
| **Q5** | Đa tenant? | Platform dùng chung | **Platform dùng chung** (`centerId: null`) — trùng khuyến nghị |
| **Q6** | Cỡ file / đồng bộ vs nền? | Đồng bộ, ≤5MB | **Đồng bộ**, ≤5MB (~5.000 bài học) — trùng khuyến nghị |
| **Q-Quiz** ⭐ | "Quiz" trỏ tới đâu? | Hoãn Phase 2 | **Hoãn `quiz` sang Phase 2**; Phase 1 ship `ipa`+`talk` — trùng khuyến nghị |

---

## Chi tiết & lý do

### Q1 — Khuyến nghị: Thực thể miền mới
`modules/course/` hiện là **catalog marketing đối tác B2B** (`source: partner`, chỉ tag skills/subskills/cefr,
**không** có tham chiếu bài tập, permission `course:*` đã bị chiếm). Khóa học có lộ trình + bài tập là **bản chất
miền khác hẳn**. Trộn 2 thứ sẽ phá ngữ nghĩa "khóa học". → Tạo `course_structures` riêng, module `course-content`.

### Q2 — Khuyến nghị: theo người soạn file
Đây là quyết định **ai soạn file**, không thuần kỹ thuật:
- **Excel (.xlsx):** đúng nhu cầu đã ghi trong `content_ingestion_schema.md` — đội học thuật non-tech soạn trực
  tiếp, không thấy JSON. **Chi phí:** thêm dependency `exceljs` (⇒ cập nhật ADR-0001), parse phức tạp hơn, lỗi
  trỏ theo số dòng.
- **JSON:** **0 dependency mới**, lỗi trỏ vị trí chính xác (JSON path), khớp output mẫu trong idea docs. **Chi
  phí:** đội non-tech khó tự soạn JSON thô.

→ **Nếu đội học thuật tự tạo file:** chọn **Excel** (đúng giá trị người dùng, chấp nhận +1 dependency).
**Nếu có dev/kỹ thuật trung gian chuẩn bị file:** chọn **JSON** (rẻ và an toàn hơn). Parser đã tách adapter nên đổi
sau rẻ; khuyến nghị **chốt 1 định dạng cho Phase 1** để thu hẹp phạm vi.

### Q3 — Khuyến nghị: Chỉ chấp nhận bài tập đã publish
Khóa import ra để learner dùng ở Phase 2. Trỏ tới bài tập `draft`/`archived` sẽ vỡ khi render. → Từ chối kèm lỗi
chỉ rõ ID chưa publish (IPA có `status`; Talk scenario luôn "live" vì là hằng in-code; Quiz phụ thuộc Q-Quiz).

### Q4 — ✅ Chốt: Từ chối nếu trùng (HTTP 400 + lỗi trỏ dòng)
Đơn giản, an toàn, không mất dữ liệu cũ. Design: phát hiện trùng ngay trong **pha validate** (query DB) → **400**
`IMPORT_VALIDATION_FAILED` với `errors[]` (`ALREADY_EXISTS`, kèm số dòng); `slug` unique index là safety net khi
đua ghi. (Dùng 400 theo đúng yêu cầu PO + convention repo không có 422.)

> ✅ **Đã chốt tại Technical Review (2026-07-15): phương án (a).** Chỉ chặn khi **cả khóa (`slug`/roadmap ID)**
> trùng; ID module/lesson chỉ cần duy nhất **trong phạm vi cha** (AC-E3), KHÔNG bắt duy nhất toàn hệ thống. Design
> §3 validate + AC-E5 đã theo (a).

### Q5 — Khuyến nghị: Nội dung platform dùng chung
Khóa học tĩnh là chương trình chuẩn của platform (đội học thuật trung tâm biên soạn), không phải nội dung riêng
từng center. → `centerId: null`, cấp quyền `coursecontent:*` cho platform admin/đội học thuật, **chưa** đưa vào
`CENTER_PERMISSIONS`. Model đã chừa cột `centerId` nên nếu sau này cho center tự tạo khóa riêng thì mở rộng được,
không phải migration lớn.

### Q6 — Khuyến nghị: Đồng bộ + ngưỡng khởi điểm
Khóa Phase 1 quy mô nhỏ (chục–trăm KB) → xử lý **đồng bộ trong request**, phản hồi ngay (UX import tốt). Đặt
ngưỡng khởi điểm để báo lỗi rõ thay vì chạm trần 16MB BSON: **file ≤ ~5MB, ≤ ~5.000 bài học/khóa** (con số đề
xuất, PO tinh chỉnh — không phải yêu cầu bắt buộc). Nếu tương lai cần file lớn ⇒ chuyển sang BullMQ job (hạ tầng
`queues/` đã có).

### Q-Quiz ⭐ (câu hỏi kỹ thuật MỚI — spec chưa nêu)
Đọc code phát hiện **"Quiz" không có nguồn rõ ràng** để validate tồn tại (IS-4):
- `assessments` = **phiên đánh giá của từng user** (`userId` bắt buộc) — KHÔNG phải nội dung tái dùng để Bài học
  trỏ tới.
- `assessment_questions` = **item ngân hàng đề** cho CAT/MST (mcq/cloze…) — là **từng câu lẻ**, không phải một
  "bài quiz" hoàn chỉnh.
- `questions` (level-test) = một collection khác nữa.
- IPA và Talk thì rõ: IPA → `ipa_lessons.code`; Talk → hằng `SCENARIO_IDS`.

→ **Khuyến nghị Phase 1: hoãn hỗ trợ type `quiz`**, chỉ ship `ipa` + `talk` (nguồn rõ ràng, validate được ngay).
File có tham chiếu `quiz` sẽ báo lỗi "quiz chưa được hỗ trợ" thay vì ngầm chấp nhận. Khi PO/BA xác định "một quiz
trong Bài học" ánh xạ tới đâu (một tập `assessment_questions`? một khái niệm "mini-test" mới?), sẽ bổ sung resolver
`quiz` — không đổi data model. **Cần trả lời cùng Q3.**

---

## Sau khi có câu trả lời

1. Cập nhật các mục **[CHỜ Qx]** trong `design.md`, `data-model.md`, `contracts/admin-course-import.md`.
2. Bổ sung AC còn treo trong `acceptance.md` (AC-E5 theo Q4, AC-E6 theo Q3).
3. Nếu Q2 = Excel → bổ sung mục dependency vào `docs/adr/0001-...` (hoặc ADR-0002).
4. Nâng spec sang "Đã duyệt" (BA) → Technical Review design → generate `tasks.md`.
