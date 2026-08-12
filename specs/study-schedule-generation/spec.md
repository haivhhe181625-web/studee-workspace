# Spec: Sinh lịch học từ Lộ trình (Study Schedule Generation)

- **Ngày:** 2026-08-04 · **Cập nhật:** 2026-08-05 (slice v2)
- **Tác giả:** AI Brainstorm (từ ý tưởng M4-1 của owner Vũ Hồng Hải)
- **Trạng thái:**
  - **v1 (sinh lịch cơ bản) — ĐÃ SHIP** (branch `feature/study-schedule-generation`, giữ local): D1–D4 + OQ-4 chốt.
  - **v2 (ôn tập cuối tuần + chọn ngày cụ thể + suggestion + projection) — CHỜ DUYỆT DESIGN** (D5–D10 dưới đây).
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — thuật toán sinh lịch thuần + resolver + model + endpoint generate/suggest
  - `web` (`exe-web`) — **ngoài phạm vi feature này** (màn khai báo + Lịch = M4-2/M4-6, feature sau); v2 chỉ định nghĩa contract để web dùng
- **Module liên quan (exe-api):**
  - `src/modules/course-content/` — `CourseStructure` (cây bài + `LessonSchema` + `exercises[]`) + `CourseEnrollment`
  - `src/modules/adaptive/` — `StudentPath`, `StudentModel` (band), `LessonSchedule` (v1), `StudySchedule` (cũ, coexist)
  - `src/modules/user/` — `User.profile.studyPlan` (quỹ thời gian)
- **Backlog ref:** M4-0, M4-1, M4-3 (Plane · Studee Product) · Đối chiếu SRS M4 (`Studee Product` page 311545ba)

## 1. Mục tiêu

Từ **danh sách bài của một lộ trình** + **quỹ thời gian người học khai báo**, sinh ra **lịch học theo ngày** bằng **thuật toán xác định** (cùng input + `asOfDate` → cùng lịch), persist lại cho các feature sau (xem lịch / nhắc học / UI) dùng.

**v2 bổ sung:** (a) **ngày ôn tập cuối tuần** tổng hợp bài tập của tuần; (b) người dùng **chọn thứ học + thứ ôn cụ thể**; (c) **gợi ý** cường độ học phù hợp; (d) **projection** — dự báo bao lâu xong khóa + hạn chế.

## 2. Bối cảnh

### Hiện trạng đã kiểm tra trong code (không đoán)

- **Nguồn bài KHÔNG ở Roadmap.** Danh sách bài theo user chỉ ở **`StudentPath`** (`courses[]→phases[]→modules[]` với `lessonKeys[]`). Cây bài cố định = `CourseStructure`.
- **`LessonSchema`** (`course-content.model.js` L64-85): `key/title/order/type` (9 loại) + `estimatedDurationMin` (v1 đã thêm) + **`exercises[]`** = `[{ type: ipa|talk|quiz|flashcard, refId, order, label }]` (ref mềm, **không có thời lượng**) + `media[]` + `theory`. CEFR chỉ ở tầng `Phase` (`cefrFrom/cefrTo`).
- **Quỹ thời gian** ở `User.profile.studyPlan` (`user/user.model.js` L70-80): `timeSlots[]`, `daysPerWeek` (1–7), `weekendStudy` (bool), `focusDurationMin` (5–120 phút/ngày), `deadline` (string), `commitment` (1–10). Validate qua `study-schedule.schema.js` (`savePrefs`).
- **`weekendStudy` + `daysPerWeek` DÙNG CHUNG:** ngoài generator v1, còn được `study-schedule.service.js` (L112-123) — service `StudySchedule` LLM cũ (coexist) — đọc để chọn `studyDows`. `study-schedule.service.test.js` phụ thuộc. **→ Không xóa được field.**
- **`LessonSchedule`** (v1, `adaptive/lesson-schedule.model.js`): 1 doc/user, `days[{ dateLocal, loadMin, oversized, items[] }]` + `inputHash`. Chưa có `isReview`.
- **Band từng kỹ năng:** `StudentModel.skill` (Map skill→`{cefr}`).
- **`DailyQuest`/tiến độ theo thời gian: CHƯA có** → ôn tập v2 là "ôn tĩnh" (theo lịch, không theo lịch sử làm bài).

### Vì sao cần v2

v1 đã sinh được lịch, nhưng: (1) thiếu **củng cố** — học xong không ôn lại → mau quên; (2) chọn ngày còn do hệ thống tự đoán (`weekendStudy` trơ) → không khớp nhịp sống người học; (3) người dùng **không biết trước** khai cường độ này thì bao lâu xong / có kịp không. v2 lấp 3 khoảng này, đồng thời **đưa code khớp lại SRS M4 FR-002** (1 ngày ôn/tuần — trước đó lệch do owner hoãn D3 ở v1).

## 3. Phạm vi

### Trong phạm vi — v1 (ĐÃ SHIP, giữ nguyên)

- **IS-1** — `estimatedDurationMin` trên `LessonSchema` + heuristic fallback theo `type`.
- **IS-2** — Resolver nguồn bài (`StudentPath` → `CourseStructure` fallback), danh sách lesson đã sắp thứ tự.
- **IS-3** — Thuật toán sinh lịch thuần: chunking, không cắt đôi bài, bài quá khổ độc chiếm ngày, lọc bài dưới trình theo band từng kỹ năng, xác định.
- **IS-4** — Cảnh báo không kịp `deadline` + đúng 2 gợi ý.
- **IS-5** — Persist `LessonSchedule` + `POST /schedule/generate` idempotent theo `inputHash`.

### Trong phạm vi — v2 (SLICE MỚI)

- **IS-6 — Ngày ôn tập cuối tuần (thuần):** mỗi tuần chèn **1 ngày ôn** vào `reviewDow` (**buổi THÊM**, không ăn buổi học), **nằm cuối tuần**. Nội dung = **tổng hợp `exercises[]` của các bài đã xếp trong tuần đó**. Thời lượng = tổng ước lượng theo bảng heuristic exercise (§8). **Không chặn trần phút/ngày**, **không tính `oversized`**. Tuần **không có exercise nào** → **bỏ** ngày ôn tuần đó. *(Logic thêm/bớt/chọn-lọc nội dung ôn = v3, để sau; v1-của-ôn gom hết.)*
- **IS-7 — Người dùng chọn thứ học + thứ ôn (thay `weekendStudy`):** prefs thêm `learningDows` (danh sách thứ học) + `reviewDow` (1 thứ ôn), ràng buộc **`reviewDow` sau thứ học cuối trong tuần** (ôn luôn cuối). Generator mới **chỉ dùng `learningDows`/`reviewDow`**, **ngừng dùng `weekendStudy`/tự-đoán-thứ**. `weekendStudy`/`daysPerWeek` **giữ nguyên trong schema** cho service cũ (D8).
- **IS-8 — Ràng buộc input + validation:** thứ học **≥ 2 và ≤ 7 ngày/tuần**; **≥ 60 và ≤ 240 phút/ngày**. Reject ngoài khoảng (Joi).
- **IS-9 — Gợi ý cường độ (`GET /schedule/suggest`, thuần):** từ **tổng phút khóa** + `asOfDate` (+ `deadline` nếu có):
  - **Có deadline:** tính **phút/tuần cần** để kịp; nếu vượt ngưỡng lành mạnh → cờ `tooTight` + phương án nhanh-nhất-lành-mạnh kèm ngày xong thực tế.
  - **Không deadline:** trả **3 preset** — `thong_tha` / `can_bang` (khuyến nghị) / `cap_toc` — mỗi preset kèm cấu hình (ngày/tuần · phút/ngày) + **ngày xong dự kiến**.
- **IS-10 — Projection trong `generate`:** response kèm summary `{ totalLearningSessions, totalReviewSessions, totalLessonMin, estimatedFinishDate }` + danh sách cảnh báo (kịp hạn · có bài quá khổ · chọn tối thiểu → lịch dài · tuần ôn nặng).

### Ngoài phạm vi

- **Logic chọn-lọc nội dung buổi ôn thông minh** (ưu tiên bài sai / kỹ năng yếu / giãn ngắt quãng SM-2) — **Lý do:** cần tầng tiến độ + điểm số (chưa có); v3.
- **Job dồn bài trễ / trần 150% / reschedule (M4-4)**; **`GET /schedule/today` + múi giờ**; **tick hoàn thành / quest status**; **thông báo nhắc học (M4-5)**; **UI (exe-web)** — **Lý do:** các slice/PR sau, đã liệt kê ở SRS M4.
- **Xóa `weekendStudy`/refactor `study-schedule.service.js` cũ** — **Lý do:** service coexist còn dùng; dọn ở PR deprecate riêng (D8).
- **Track thời gian học thực tế** — không có field sẵn, ngoài release.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Tự chọn thứ học + thứ ôn trong tuần | Lịch khớp nhịp sống thật |
| Learner | Cuối tuần có buổi ôn tổng hợp bài tập tuần | Củng cố, đỡ quên |
| Learner (chưa biết khai bao nhiêu) | Xem gợi ý cường độ + ngày xong dự kiến | Chọn nhịp thực tế, biết trước cam kết |
| Learner (có deadline) | Biết khai cường độ này có kịp hạn không | Điều chỉnh kỳ vọng |
| Hệ thống (feature sau) | Đọc lịch đã persist (gồm ngày ôn) | Dồn trễ / nhắc học / render UI |

## 5. Quyết định nghiệp vụ

> Owner (Vũ Hồng Hải) quyết chính. D1–D4 = v1 (đã chốt). D5–D10 = v2.

| # | Câu hỏi | Chốt | Người quyết |
|---|---|---|---|
| D1 | Nguồn thời lượng bài | ✅ Field `estimatedDurationMin` + heuristic | Owner |
| D2 | Neo nguồn bài | ✅ StudentPath (fallback CourseStructure) | Owner |
| D3 | ~~Chèn ngày ôn (v1)~~ | ⚠️ **ĐẢO ở v2:** v1 bỏ ngày ôn; **v2 CÓ ngày ôn cuối tuần** (IS-6) | Owner |
| D4 | Persist | ✅ `LessonSchedule` mới (coexist `StudySchedule`) | Owner |
| D5 | Hình thức ôn | ✅ **Không đóng tạm** bài tập ở bài học; **bỏ ôn trong ngày**; **ôn cuối tuần** tổng hợp exercises tuần | Owner (2026-08-05) |
| D6 | Ngày ôn ăn buổi hay thêm buổi | ✅ **Buổi THÊM**, nằm cuối tuần (`reviewDow`) | Owner (2026-08-05) |
| D7 | Chọn ngày | ✅ Người dùng **tick `learningDows` + `reviewDow`**; ôn luôn cuối | Owner (2026-08-05) |
| D8 | `weekendStudy` | ✅ **Generator mới ngừng dùng**; **GIỮ field** trong schema (service cũ còn dùng) — không xóa để tránh phá coexist | Owner (2026-08-05) ✅ |
| D9 | Sàn/trần input | ✅ **2–7 ngày/tuần**, **60–240 phút/ngày** | Owner (2026-08-05) |
| D10 | Suggestion | ✅ Có deadline → tính kịp-hạn; không → **3 preset kèm ngày xong** (`thong_tha`/`can_bang`/`cap_toc`) | Owner (2026-08-05) |

## 6. Acceptance criteria

> AC-1…AC-9 = v1 (đã pass, giữ). AC-10…AC-17 = v2 (mới).

**v1 (đã ship):**
1. **AC-1** (IS-1): mọi lesson `durationMin > 0`; field có → dùng field; thiếu → heuristic theo `type`.
2. **AC-2** (IS-2): có StudentPath → list đúng thứ tự + `{lessonKey,title,type,durationMin}`; không → fallback CourseStructure.
3. **AC-3** (IS-3): 60 bài/3 ngày/45' chia đúng, không cắt đôi (khớp ví dụ tính tay M4-1).
4. **AC-4** (IS-3): bài 60'>45' độc chiếm ngày, cờ `oversized`.
5. **AC-5** (IS-3): cùng (input+asOfDate) → cùng output.
6. **AC-6** (IS-3): có band → lọc bài dưới trình theo kỹ năng; không band → vẫn sinh.
7. **AC-7** (IS-3): 1/7 ngày/tuần, 200 bài → chia đúng.
8. **AC-8** (IS-4): quỹ ít so deadline → warning + đúng 2 gợi ý.
9. **AC-9** (IS-5): `POST /schedule/generate` persist + idempotent theo `inputHash`.

**v2 (mới):**
10. **AC-10** (IS-6): tuần có ≥1 bài kèm `exercises[]` → chèn **1 ngày ôn** vào `reviewDow` cuối tuần, `isReview=true`, `items` = exercises tổng hợp của tuần, `loadMin` = tổng heuristic exercise. Tuần rỗng exercises → **không** có ngày ôn.
11. **AC-11** (IS-6): ngày ôn **không** bị chặn `focusDurationMin` (có thể vượt), **không** gắn `oversized`; ngày ôn **không** chứa bài học mới.
12. **AC-12** (IS-7): `learningDows=[2,4,6]` (Hai/Tư/Sáu), `reviewDow=7` (CN) → bài học chỉ rơi vào Hai/Tư/Sáu; ngày ôn rơi vào CN; **không** phụ thuộc `weekendStudy`.
13. **AC-13** (IS-7): `reviewDow` **không** sau thứ học cuối trong tuần → validation **reject** (400).
14. **AC-14** (IS-8): `learningDows.length < 2` hoặc `> 7`, hoặc `focusDurationMin < 60` / `> 240` → validation **reject** (400).
15. **AC-15** (IS-9): không deadline → `GET /schedule/suggest` trả 3 preset, mỗi preset có `{ key, daysPerWeek, focusDurationMin, estimatedFinishDate }`; có deadline → trả `{ requiredWeeklyMin, tooTight, fallbackPlan? }`.
16. **AC-16** (IS-10): `generate` response kèm `summary { totalLearningSessions, totalReviewSessions, totalLessonMin, estimatedFinishDate }` + `warnings[]`.
17. **AC-17** (IS-9/10 determinism): suggestion + projection thuần từ tổng phút khóa + `asOfDate` (+deadline); cùng input → cùng output.

## 7. Câu hỏi mở

- [x] ~~OQ-1..OQ-4 (v1)~~ → chốt.
- [x] ~~OQ-5 (D8)~~ → chốt (owner 2026-08-05): **giữ field `weekendStudy`** trong schema; generator mới ngừng dùng. Dọn hẳn ở PR deprecate riêng sau.
- [x] ~~OQ-6~~ → chốt (owner 2026-08-05): tuần = **tuần lịch ISO (Hai→CN)** để nhóm bài + đặt ngày ôn.
- [ ] **OQ-7 (bảng heuristic thời lượng exercise):** **dùng giá trị đề xuất §8** (quiz 8 · flashcard 5 · ipa 5 · talk 10 phút) cho slice này — owner/đội nội dung tinh chỉnh sau (không chặn).
- [ ] **OQ-8 (ngưỡng "lành mạnh"):** **dùng mốc đề xuất** ≤ 5 ngày × 90 phút (~7.5h/tuần) làm cờ `tooTight` cho slice này — tinh chỉnh sau (không chặn).

## 8. Ghi chú cho Tech Lead Design

- **HARD: thuật toán thuần** (NFR-1). Generator + suggestion + projection: không `now()`/`random()`; `asOfDate` là tham số. Ngày server chỉ ở controller (biên).
- **Heuristic thời lượng LESSON** (v1, giữ): flashcard/listening_quiz 10 · theory/video/ipa 15 · grammar_quiz/reading_quiz 20 · speaking_task 25 · writing_task 60.
- **Heuristic thời lượng EXERCISE** (v2, mới, cho ngày ôn): `quiz` 8 · `flashcard` 5 · `ipa` 5 · `talk` 10 (phút). Giá trị đề xuất — tinh chỉnh sau (OQ-7).
- **Ngày ôn = buổi THÊM**, đặt ở `reviewDow`, không ăn `learningDows`.
- **KHÔNG xóa `weekendStudy`** (D8) — chỉ generator mới ngừng đọc.
- **Resolver phải surface `exercises[]` của mỗi lesson** để generator tính tải ngày ôn (mở rộng `LessonItem`).
- **Đối chiếu SRS M4** (page 311545ba): v2 khớp FR-002 (ngày ôn/tuần); FR-006 (cảnh báo+2 gợi ý) đã có; contract server-side là deviation có chủ đích so §10.2 (cần BA ký nhận — ngoài phạm vi code slice này).
