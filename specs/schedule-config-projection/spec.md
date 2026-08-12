# Spec: Cấu hình lịch — tổng giờ khóa + gợi ý giờ/tuần + hạn tự tính

- **Ngày:** 2026-08-05 · **Trạng thái:** Đã duyệt (owner "clear, mặc định"). LITE, không cần `design.md`.
- **Repos:** exe-api (`feature/study-schedule-generation`) · exe-web (`feature/schedule-ui-integration`).
- **Nền tảng:** nối tiếp `specs/schedule-module-finishing` + slice ghép UI.

## 1. Mục tiêu
Ở phần **cấu hình lịch học** (FE), thêm 3 thông tin để người dùng chọn nhịp học chính xác hơn:
1. **Tổng thời lượng khóa = X giờ** (note: đã cộng ~giờ ôn tập dự tính).
2. **Gợi ý ~ giờ/tuần hợp lý** (nhịp Cân bằng).
3. **Hạn hoàn thành = TỰ TÍNH** theo số ngày học × phút/ngày (BỎ ô người dùng chọn hạn).

## 2. Quyết định (owner 2026-08-05)
| # | Nội dung | Chốt |
|---|---|---|
| D1 | Bỏ ô chọn hạn → **overdue warning ngừng hoạt động** (không còn mốc hạn) | ✅ Chấp nhận (các cảnh báo oversized/heavy_review giữ nguyên) |
| D2 | Công thức ngày-xong: `tuần = ⌈tổng phút học ÷ (số ngày học × phút/ngày)⌉`, ngày xong = hôm nay + tuần×7. Buổi ôn nằm TRONG tuần (không kéo dài thêm tuần). | ✅ Chấp nhận |
| D3 | "Tổng giờ khóa" cộng cả giờ ôn (thông tin khối lượng), tách khỏi công thức ngày-xong | ✅ |

## 3. Phạm vi
### Trong phạm vi
- **IS-1 (BE):** `GET /schedule/suggest` trả thêm `totalLessonMin` + `totalReviewMin` (tổng ước lượng exercises toàn khóa, chưa lọc band — nhất quán ước lượng thô của suggest). Giữ nguyên `presets`/`estimatedFinishDate`.
- **IS-2 (FE):** phần cấu hình lịch (form/suggest panel):
  - Hiện **"Tổng thời lượng khóa: {tổng giờ} (gồm ~{giờ ôn} ôn tập)"** — dùng `formatDurationMin(totalLessonMin + totalReviewMin)` + `formatDurationMin(totalReviewMin)`.
  - Hiện **"Gợi ý: ~{giờ/tuần}/tuần (nhịp Cân bằng)"** — từ preset `can_bang` (`learningDays×minutesPerDay`).
  - Hiện **"Hạn hoàn thành dự kiến: {ngày}"** — tính **live** theo `learningDows.length × focusDurationMin` vs `totalLessonMin` (mirror công thức BE `projectFinish`, D2).
  - **BỎ** field/date-picker `deadline` khỏi form + zod; khi lưu **không gửi** `deadline`.

### Ngoài phạm vi
- E2E (owner bổ sung sau) · các slice M4-4/M4-5/ôn thông minh · dọn field `deadline` khỏi schema BE (giữ optional, chỉ FE ngừng gửi).

## 4. Acceptance criteria
1. **AC-1** (IS-1): `GET /schedule/suggest` → response có `totalLessonMin` (Σ durationMin) + `totalReviewMin` (Σ `exerciseDurationOf` mọi exercise của mọi lesson); khớp dữ liệu seed. Giữ `presets`.
2. **AC-2** (IS-2): phần cấu hình hiện đúng tổng giờ (gồm ôn) + gợi ý giờ/tuần (Cân bằng) + hạn dự kiến; hạn đổi live khi chỉnh số ngày/phút.
3. **AC-3** (IS-2): form KHÔNG còn ô chọn hạn; submit không gửi `deadline`; luồng lưu prefs + tạo lịch vẫn chạy.

## 5. Câu hỏi mở
- Không còn (D1–D3 chốt). Overdue warning ghi backlog nếu sau này muốn mốc hạn quay lại.
