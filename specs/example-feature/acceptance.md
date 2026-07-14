<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md -->
# Acceptance Criteria: [VÍ DỤ] Nhắc học qua email hằng ngày

- **Spec:** `specs/example-feature/spec.md`
- **Ngày:** 2026-07-14

## Tiêu chí chức năng

### AC-1: Bật nhắc học → nhận email khi chưa đạt mục tiêu

- **Given** user có `reminderEnabled: true` và tổng phút học hôm nay < `dailyGoalMinutes`
- **When** job nhắc học chạy lúc 20h (giờ VN)
- **Then** user nhận 1 email nhắc học, log job ghi nhận đã gửi

### AC-2: Tắt nhắc học → không nhận email

- [ ] `reminderEnabled: false` → job bỏ qua user này, không gọi adapter email

### AC-3: Đã đạt mục tiêu → không nhận email dù đã bật

- **Given** user có `reminderEnabled: true` và tổng phút học hôm nay ≥ `dailyGoalMinutes`
- **When** job chạy lúc 20h
- **Then** không gửi email cho user này

## Tiêu chí lỗi / edge case

### AC-E1: User chưa từng set `dailyGoalMinutes`

- **Given** `dailyGoalMinutes` là `null`
- **When** job chạy
- **Then** bỏ qua user này (không có mục tiêu để đối chiếu), không lỗi job

## Ngoài phạm vi kiểm thử

- Giờ nhắc tuỳ chỉnh theo user — đã loại khỏi phạm vi ở spec.md.
