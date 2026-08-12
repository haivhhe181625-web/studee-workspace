# Spec: Hoàn thiện module Lịch học (đọc prefs + test-hardening FE)

- **Ngày:** 2026-08-05
- **Trạng thái:** Đã duyệt (owner 2026-08-05 — chốt mặc định Q1a–Q4a). LITE, không cần `design.md` (CRUD + test thẳng). → tiến hành code.
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`, branch `feature/study-schedule-generation`) — thêm `GET /adaptive/schedule/prefs`.
  - `web` (`exe-web`, branch `feature/schedule-ui-integration`) — đọc/sửa prefs + lấp test FE.
- **Nền tảng:** nối tiếp `specs/study-schedule-generation` (v1+v2) + slice ghép UI (`plans/260805-1627-schedule-ui-integration`).

## 1. Mục tiêu
Đóng module lịch học ở mức đáng tin: (1) người dùng **sửa được prefs** (cần cửa đọc prefs); (2) **lấp lỗ hổng test FE** (utils, container, form). KHÔNG mở scope slice sau.

## 2. Bối cảnh (khảo sát 2026-08-05)
- BE test đủ tốt (generator 24, suggest 9, 3 API suite generate/suggest/prefs-save, resolver 3, model/schema/duration).
- FE test mỏng: chỉ `schedule.service.test.ts` (3) + `SchedulePrefsForm.test.tsx` (4, thiếu nhánh preset/preview). Chưa test `schedule.utils.ts`, `ScheduleContainer`, hooks.
- FE **không sửa prefs được**: không có API đọc `studyPlan` → Task 4 slice trước đã bỏ thẻ tóm tắt prefs.

## 3. Phạm vi
### Trong phạm vi
- **IS-1 (BE):** `GET /adaptive/schedule/prefs` → `{ studyPlan | null }`. Self-service JWT, read-only.
- **IS-2 (FE):** hook `useStudyPrefs` + prefill form "Chỉnh sửa" bằng prefs đang lưu + khôi phục thẻ tóm tắt prefs.
- **IS-3 (FE test):** unit `schedule.utils.ts` (`groupDaysByWeek`, `mondayOf`, `todayLocalDateString`/`parseLocalDate` chống off-by-one, `itemTypeLabel`).
- **IS-4 (FE test):** component `ScheduleContainer` — 5 nhánh (loading · error · 409 no-source · needsPrefs→form · schedule render + summary + warnings + badge).
- **IS-5 (FE test):** `SchedulePrefsForm` — preset quick-fill + panel suggest.

### Ngoài phạm vi
- **E2E** — owner bổ sung thông tin sau.
- **M4-4** (job dồn trễ/today/tick), **M4-5** (nhắc học), **cảnh báo "lịch dài"**, **ôn thông minh v3** — slice sau.
- **Deep-link buổi học** (Q2a = hoãn) · **`timeSlots` trong form** (Q3a = không thu) · **per-roadmap scoping** (Q4a = 1 active/người, giữ nguyên) · dọn field `weekendStudy`/`timeSlots` (PR deprecate riêng).

## 4. Quyết định (owner 2026-08-05)
| # | Câu hỏi | Chốt |
|---|---|---|
| Q1 | Đọc prefs thế nào | ✅ **(a) `GET /adaptive/schedule/prefs`** riêng (sạch, cache được) |
| Q2 | Deep-link buổi học | ✅ **(a) Hoãn** (lịch dạng xem ở slice này) |
| Q3 | `timeSlots` trong form | ✅ **(a) Không thu** (chỉ 4 field cốt lõi) |
| Q4 | Số lộ trình active/người | ✅ **(a) 1 active/người** — mô hình 1 lịch/user hiện tại đúng |

## 5. Acceptance criteria
1. **AC-1** (IS-1): `GET /schedule/prefs` + JWT → 200 `{studyPlan}` khớp DB; chưa có → `{studyPlan:null}`; no JWT → 401.
2. **AC-2** (IS-2): "Chỉnh sửa lịch" → form prefill đúng giá trị đang lưu; thẻ tóm tắt hiện thứ học/ôn + phút/ngày + hạn.
3. **AC-3** (IS-3): test `schedule.utils` phủ gom tuần, ngày local không lệch, nhãn item fallback.
4. **AC-4** (IS-4): 5 nhánh container render đúng (mock `useSchedule`), warnings/summary hiển thị.
5. **AC-5** (IS-5): preset-fill + suggest-panel có test khẳng định hành vi.

## 6. Câu hỏi mở
- Không còn (Q1–Q4 đã chốt). Deep-link + e2e ghi backlog cho slice sau.
