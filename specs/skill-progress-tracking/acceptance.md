# Acceptance Criteria: Theo dõi tiến độ kỹ năng theo phase của lộ trình

- **Spec:** `specs/skill-progress-tracking/spec.md`
- **Ngày:** 2026-07-14

## Cách dùng file này

- Mỗi tiêu chí có ID (`AC-1`, ...) để `design.md`/`tasks.md`/`review.md` tham chiếu ngược.
- Given/When/Then khi hành vi phụ thuộc trạng thái; checklist đơn giản khi không cần.
- Reviewer Agent sẽ đối chiếu từng ID với test/behavior thật — ID không map được tới kiểm tra nào là finding blocking.

> ⚠️ Một số tiêu chí phụ thuộc **câu hỏi mở #1/#2/#3** ở `spec.md` (loại nội dung tính điểm, cách đo tiến độ phase, chuẩn hóa thang điểm). Chúng được đánh dấu **[phụ thuộc #n]** và sẽ chốt trước khi test.

## Tiêu chí chức năng

### AC-1: Ghi nhận hoàn thành + điểm của một bài luyện (↔ TP-1)

- **Given** học viên có một lộ trình đang active với một phase đang ở trạng thái `unlocked`/`in_progress`
- **When** học viên hoàn thành một bài luyện tập thuộc loại được tính điểm **[phụ thuộc #1]**
- **Then** hệ thống ghi nhận một bản ghi kết quả gồm: điểm số, kỹ năng (`skill`), và phase/lộ trình active tại thời điểm đó — đủ để tính tiến độ ở AC-2/AC-3.

### AC-2: Tiến độ theo kỹ năng trong phase đang active (↔ TP-2)

- **Given** học viên đã hoàn thành ≥ 1 bài luyện có điểm trong phase đang active
- **When** học viên mở màn tiến độ
- **Then** với mỗi kỹ năng của phase, hiển thị: **số bài đã hoàn thành** và **điểm trung bình** (theo thang đã chuẩn hóa **[phụ thuộc #3]**).

### AC-3: Tiến độ theo từng phase của lộ trình (↔ TP-3)

- **Given** học viên có lộ trình active nhiều phase
- **When** học viên mở màn tiến độ
- **Then** với mỗi phase, hiển thị: **đã có hoạt động luyện tập hay chưa** và **điểm trung bình chung của phase**; cách thể hiện mức độ hoàn thành phase theo **[phụ thuộc #2]** (số bài + điểm, chưa có `%` nếu không chốt được mẫu số).

### AC-4: Chỉ xem tiến độ của chính mình (↔ TP-4)

- [ ] Màn/endpoint tiến độ là user-scoped: chỉ trả về dữ liệu của học viên đang đăng nhập, không của user khác.

## Tiêu chí lỗi / edge case

### AC-E1: Không tính bài không hợp lệ vào điểm trung bình

- **Given** trong dữ liệu có bài không có điểm, hoặc bị đánh dấu gaming/đoán bừa (`countsTowardMastery = false`)
- **When** hệ thống tính điểm trung bình theo kỹ năng/phase
- **Then** các bài đó **không** được đưa vào tính điểm trung bình (không làm sai lệch tiến độ).

### AC-E2: Trạng thái rỗng khi chưa luyện tập

- **Given** học viên có lộ trình active nhưng **chưa hoàn thành** bài luyện có điểm nào
- **When** học viên mở màn tiến độ
- **Then** hiển thị trạng thái rỗng rõ ràng (vd "Chưa có dữ liệu luyện tập") — **không** báo lỗi, **không** hiển thị 0 điểm gây hiểu nhầm là điểm kém.

### AC-E3: Không có lộ trình active

- **Given** học viên chưa có lộ trình active nào
- **When** học viên mở màn tiến độ
- **Then** hiển thị hướng dẫn tạo/kích hoạt lộ trình thay vì màn tiến độ trống, không lỗi hệ thống.

## Tiêu chí phi chức năng

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| Bảo mật | Endpoint tiến độ yêu cầu đăng nhập (verifyToken) và chỉ trả dữ liệu của chính user | Gọi với token user A không lấy được tiến độ user B |
| Hiệu năng | Mở màn tiến độ không quét toàn bộ `learning_events` mỗi lần (đọc dữ liệu đã tổng hợp) | Review truy vấn ở design/PR — không có full-collection scan theo user |

## Ngoài phạm vi kiểm thử

- Không kiểm thử việc **sinh lại lịch / khuyến nghị quay lại phase** — đã liệt kê "Ngoài phạm vi" ở `spec.md` (Framing C, feature riêng).
- Không kiểm thử tiến độ cho loại nội dung **chưa có điểm** (talk/speaking chưa chấm) — loại trừ theo câu hỏi mở #1.
- Không kiểm thử hiển thị tiến độ ở **admin/B2B** — ngoài phạm vi đợt này.
