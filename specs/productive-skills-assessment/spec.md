---
feature: productive-skills-assessment
status: draft
owner: BA
created: 2026-09-14
affects: [exe-api, exe-web, studee-app]
---

# Đề đánh giá Nói & Viết riêng + khoá kỹ năng chưa đánh giá

## 1. Bối cảnh & vấn đề

Bài **Đánh giá năng lực (placement)** hiện là bài cố định **Reading + Listening**, chấm khách quan,
**không dùng AI, không tốn quota** (`exe-api` bypass `withQuota`, trả `UNLIMITED`). Writing/Speaking chấm
bằng AI (tốn phí) nên đã bị loại khỏi placement.

Hệ quả:
- Người học sau khi làm placement **không có** điểm Nói/Viết → trang kết quả "Đánh giá trình độ" thiếu 2 kỹ năng.
- Trên **mobile**, luồng `exam_mst` vẫn cố vào phase Writing/Speaking sau R+L → gặp danh sách rỗng → **kẹt, không ra kết quả** (bug hiện hữu).

## 2. Mục tiêu

1. Thêm **bài đánh giá "Nói & Viết"** riêng: chấm AI, **có giới hạn (quota)**, tách khỏi placement.
2. Trang kết quả: kỹ năng **chưa đánh giá** hiển thị **mờ + khoá**, kèm nút **"Đi đến đánh giá"** để vào thẳng bài Nói & Viết; kỹ năng đã có giữ nút "Làm lại" tổng.
3. Sửa bug mobile: placement (chỉ R+L) phải hoàn tất được, không kẹt ở phase W/S rỗng.

## 3. Quyết định đã chốt (owner, 14/09/2026)

| # | Quyết định | Chọn |
|---|---|---|
| D1 | Gộp hay tách W/S | **Gộp 1 bài "Nói & Viết"** (`config.skills=['writing','speaking']`). Tách sau chỉ là đổi config. |
| D2 | Quota bucket | **Dùng chung bucket `assessment`** (như diagnostic/mock). |
| D3 | Cờ MST cho `ws-tasks` | **Mở rộng bypass theo testType** (không bật cờ toàn cục). |
| D4 | Enforce quota | **Enforce ngay** qua `withQuota` (giá trị limit tinh chỉnh khi entitlement xong). |

## 4. Phạm vi

**Trong phạm vi**
- `exe-api`: goal/testType `productive_skills`; config skills=['writing','speaking']; quota qua `withQuota`; mở bypass `ws-tasks`/submit theo testType; giữ system-check mic+consent.
- `exe-web`: khoá row kỹ năng chưa đánh giá (`SkillBars`), CTA "Đi đến đánh giá" (`MissingSkillCta`) → khởi tạo bài Nói & Viết.
- `studee-app`: đọc `skills` từ response → fix placement skip W/S; entry + luồng bài Nói & Viết; result locked row + CTA.

**Ngoài phạm vi** (nêu rõ lý do)
- Tách riêng bài Viết / bài Nói (D1 chọn gộp) — để lại vì chỉ là đổi config, chưa cần.
- Wire giá trị limit paid/entitlement cụ thể — đang defer ở lộ trình khác; feature này chỉ dùng cơ chế `withQuota` sẵn có.
- Thay đổi scoring cutoff / cơ chế chọn đề seeded — đã hoàn tất trước đó, không đụng.
- Chuẩn hoá IRT / pilot calibration — lộ trình riêng.

## 5. Acceptance Criteria (checkable)

**Backend (exe-api)**
- [ ] AC1: Intake `goal='productive_skills'` tạo assessment với `config.skills=['writing','speaking']`, `assemblyMode='fixed'`, `testType='productive_skills'`.
- [ ] AC2: Tạo bài này **tiêu tốn 1 đơn vị quota `assessment`** (đi qua `withQuota`), KHÔNG bypass như placement; hết quota → 402.
- [ ] AC3: System-check thiếu mic hoặc thiếu recordingConsent → 422 `SYSTEM_CHECK_FAILED` (vì có speaking).
- [ ] AC4: `ws-tasks` và `submit-module` cho attempt `productive_skills` hoạt động **khi cờ `ASSESSMENT_MST_ENABLED` tắt** (bypass theo testType); placement vẫn chạy; diagnostic khi cờ tắt vẫn 404.
- [ ] AC5: Kết quả trả CEFR cho writing + speaking; `validationStatus='uncalibrated'`. Placement/diagnostic/mock không đổi hành vi.

**Web (exe-web)**
- [ ] AC6: Sau placement, trang kết quả hiển thị Viết & Nói **mờ + khoá** (không hiện band số, có nhãn "Chưa đánh giá").
- [ ] AC7: Nút **"Đi đến đánh giá"** khởi tạo bài `productive_skills` và điều hướng vào màn làm bài.
- [ ] AC8: Sau khi hoàn tất bài Nói & Viết, 2 row có band, hết trạng thái khoá.

**Mobile (studee-app)**
- [ ] AC9: Làm trọn 1 bài placement trên mobile → **ra được kết quả R+L**, không xuất hiện màn Writing "0/0" hay Speaking trống.
- [ ] AC10: Có entry vào bài Nói & Viết; làm trọn → ra CEFR Nói/Viết.
- [ ] AC11: Result mobile: kỹ năng chưa đánh giá mờ + khoá + nút "Đi đến đánh giá" hoạt động.
- [ ] AC12: `flutter analyze` = 0 lỗi; đề mock 4 kỹ năng (nếu có) không regress.

## 6. Open questions

- Số lượng task Writing/Speaking cho bài này (mấy đề Viết, mấy phần Nói)? — cần chốt ở design nếu khác preset hiện có.
- Bank W/S `targetGoal=general` có đủ item để phục vụ bài này không? — kiểm ở task đầu Phase backend.
