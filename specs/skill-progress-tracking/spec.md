# Spec: Theo dõi tiến độ kỹ năng theo phase của lộ trình

- **Ngày:** 2026-07-14
- **Tác giả:** BA Agent (AI Brainstorm)
- **Repos/surfaces ảnh hưởng:** `api` (exe-api) · `web` (exe-web) · *(admin/mobile: ngoài phạm vi đợt này — xem §3)*
- **Module liên quan (exe-api):** `<API_REPO>/services/api/src/modules/adaptive` (mở rộng), đọc từ `roadmap`, và các surface luyện tập sẵn có (`ipa`, `course`, ...)
- **Trạng thái:** Chờ BA review *(còn câu hỏi mở ở §7 — chưa duyệt để design)*

## 1. Mục tiêu

Cho phép học viên nhìn thấy tiến độ **theo từng kỹ năng và từng phase của lộ trình cá nhân hóa**, dựa trên **mức độ hoàn thành và điểm số các bài luyện tập họ đã làm**, để biết mình đang bám lộ trình tới đâu và còn yếu kỹ năng nào.

## 2. Bối cảnh

Đây là lát cắt "đóng vòng lặp" (Framing A) — về bản chất là **Phase 3 "Behavior/Continuous"** từng bị defer trong `<API_REPO>/docs/active/personalized-study-schedule/plan.md`. Hiện trạng đã verify trong code, không đoán:

**Đã có:**
- **Lịch học hằng ngày:** `adaptive/study-schedule.service.js` (553 dòng) + model `study_schedules` (weekly template, mỗi slot có `activityType/skill/subskill/route/durationMin`, deep-link tới bài luyện). API `GET/POST /api/adaptive/schedule` đã chạy.
- **Lộ trình:** `user_roadmaps` — mỗi phase có `phaseId`, `name`, `skills[]`, `topics[]`, `status` (locked/unlocked/in_progress/completed), `startedAt`, `completedAt`. **Một phase KHÔNG chứa danh sách bài luyện cố định** (chỉ có kỹ năng + chủ đề).
- **Log kết quả học:** `learning_events` — append-only, mỗi câu đã chấm: `source` (assessment/lesson/quiz/speaking_drill), `skill`, `subskill`, `correct` (0/1) hoặc `band` (0–9), `countsTowardMastery` (loại trừ gaming/đoán bừa).
- **Trạng thái năng lực:** `student_models.mastery` (0..1 mỗi subskill), `weakPhonemes`.

**Chưa có (khoảng trống feature này lấp):**
- Slot trong `study_schedules` **không có field "đã hoàn thành"** → chưa biết học viên có làm buổi đã lên lịch không.
- **Không có tiến độ theo (kỹ năng × phase lộ trình)** — không màn nào cho học viên thấy "trong phase đang học, mình đã luyện bao nhiêu, điểm trung bình bao nhiêu theo từng kỹ năng".
- Module `practice/` tồn tại nhưng **rỗng** — nội dung luyện tập hiện nằm rải ở `ipa`, `course`, `talk`.

## 3. Phạm vi

### Trong phạm vi

- **TP-1.** Ghi nhận mỗi lần học viên **hoàn thành một bài luyện tập từ nội dung sẵn có** (loại đã sinh ra điểm — xem câu hỏi mở #1), kèm: điểm số, kỹ năng, và phase/lộ trình đang active tại thời điểm đó.
- **TP-2.** Tổng hợp tiến độ theo **(kỹ năng × phase của lộ trình đang active)**: số bài đã hoàn thành + điểm trung bình của kỹ năng đó trong phase.
- **TP-3.** Tổng hợp tiến độ theo **từng phase của lộ trình**: phase nào đã có hoạt động luyện tập, điểm trung bình chung của phase.
- **TP-4.** Màn hình cho **học viên** xem tiến độ ở TP-2 và TP-3, gắn với lộ trình cá nhân hóa của họ (exe-web).

### Ngoài phạm vi

- **Tự động sinh lại / điều chỉnh lịch học theo tiến độ** — **Lý do:** đó là vòng lặp thích ứng (Framing C); tách ra để đợt này ship nhỏ và mở khóa dữ liệu tiến độ trước.
- **Đề xuất "quay lại phase trước" khi điểm dưới kỳ vọng** — **Lý do:** là feature khuyến nghị/adaptation riêng, không thuộc phạm vi *hiển thị* tiến độ.
- **Tạo dạng bài luyện tập mới / hiện thực hóa `practice/` thành surface mới** — **Lý do:** user đã chốt dùng nội dung sẵn có; nội dung mới phát triển ở đợt sau.
- **Tính tiến độ cho loại nội dung chưa có điểm** (vd talk/speaking chưa chấm) — **Lý do:** chưa có cơ chế chấm ổn định; đưa vào sẽ làm điểm trung bình sai lệch (xem câu hỏi mở #1).
- **Hiển thị tiến độ cho staff/trung tâm (admin/B2B)** — **Lý do:** kéo theo quyền riêng tư/consent; tách thành feature riêng (Framing D).
- **Nhắc lịch / notification** — **Lý do:** cần chính sách opt-in riêng.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner (học viên) | Xem tiến độ theo từng kỹ năng trong phase đang học | Biết kỹ năng nào đang tốt/yếu, có nên luyện thêm không |
| Learner | Xem tiến độ theo từng phase của lộ trình | Biết mình đang bám lộ trình cá nhân hóa tới đâu |
| Hệ thống | Ghi nhận điểm + kỹ năng + phase mỗi khi học viên hoàn thành một bài luyện | Có dữ liệu tính tiến độ (và làm nền cho thích ứng sau này) |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| Chuỗi tiến độ theo thời gian (dữ liệu học tập của học viên) giữ trong bao lâu? Có coi là PII cần chính sách retention/xóa không? | Tái dùng `learning_events` sẵn có (đã lưu), không thêm chính sách mới ở MVP; rà lại khi mở cho B2B | Product owner |
| Có hiển thị nhãn "đạt / cần cải thiện" theo ngưỡng điểm (vd < 50% = cần cải thiện) không? | MVP chỉ hiển thị **số** (điểm trung bình + số bài), chưa gắn nhãn phán xét — tránh ảnh hưởng động lực học viên | Product owner |
| Tiến độ có phản ánh cả các kỳ trước / sau khi retake level test (đổi lộ trình) không? | MVP chỉ tính lộ trình **đang active**; lịch sử để đợt sau | Product owner |

## 6. Acceptance criteria (tóm tắt)

*(Chi tiết + edge case ở `acceptance.md`.)*

1. **AC-1** (↔ TP-1): Hoàn thành một bài luyện có điểm → hệ thống ghi nhận điểm + kỹ năng + phase/lộ trình active.
2. **AC-2** (↔ TP-2): Học viên xem được, theo từng kỹ năng của phase đang active, số bài đã làm và điểm trung bình.
3. **AC-3** (↔ TP-3): Học viên xem được, theo từng phase của lộ trình, phase đã có luyện tập hay chưa và điểm trung bình.
4. **AC-4** (↔ TP-4): Màn tiến độ chỉ hiển thị dữ liệu của chính học viên đang đăng nhập.
5. **AC-E1**: Bài không có điểm / bị loại vì gaming (`countsTowardMastery=false`) không được tính vào điểm trung bình.
6. **AC-E2**: Học viên chưa làm bài nào → màn tiến độ hiển thị trạng thái rỗng rõ ràng, không lỗi.

## 7. Câu hỏi mở

*(Còn tồn tại → spec CHƯA được duyệt để design.)*

- [ ] **#1 — Loại nội dung tính vào tiến độ ở MVP:** chỉ loại **đã sinh điểm sẵn** (vd IPA, course quiz), hay gồm cả loại **chưa chấm điểm** (talk/speaking drill)? *Đề xuất: chỉ loại có điểm; loại chưa chấm để đợt sau.* → chốt cái này mới biết `learning_events.source` nào được tính.
- [ ] **#2 — Đơn vị "hoàn thành" của một phase:** đã verify phase trong `user_roadmaps` **không có danh sách bài cố định** → **không có mẫu số** để tính `% bài đã làm`. Đề xuất MVP đo tiến độ phase bằng **(số bài đã làm + điểm trung bình theo kỹ năng)** trong khoảng thời gian phase active, **không** hiển thị `%`. Cần xác nhận cách này đủ, hay product muốn đặt một **mục tiêu số bài/tuần** làm mẫu số cho thanh `%`.
- [ ] **#3 — Chuẩn hóa thang điểm để tính "điểm trung bình":** các loại bài có thang khác nhau (IPA 0–100, course quiz % đúng, speaking band 0–9). Cần thống nhất **quy về một thang hiển thị** (vd 0–100) trước khi "điểm trung bình" giữa các kỹ năng có nghĩa. *(Chốt hướng ở đây; chi tiết quy đổi để Tech Lead design.)*

## 8. Ghi chú cho Tech Lead Design

*(Ràng buộc BA biết trước — KHÔNG phải giải pháp kỹ thuật.)*

- **Ưu tiên tái dùng dữ liệu sẵn có:** `learning_events` đã là "source-of-record" cho mỗi câu đã chấm (có `skill/subskill/correct/band/countsTowardMastery`); cân nhắc dùng nó làm nguồn tiến độ thay vì tạo log mới. `student_models.mastery` đã tổng hợp per-subskill — cân nhắc quan hệ giữa "mastery" và "tiến độ theo điểm" để không tạo hai con số mâu thuẫn cho cùng một kỹ năng.
- **Gắn event với phase:** phase có `startedAt`/`completedAt` và `status`; đây là gợi ý cách xác định "bài này thuộc phase nào" mà không cần phase chứa danh sách bài. (Quyết định cuối là của design.)
- **Ràng buộc quyền:** endpoint tiến độ là **user-scoped** như các route `adaptive` hiện tại (verifyToken, không verifyPermission).
- **Hiệu năng đọc:** giữ pattern của `student_models` — đọc dữ liệu đã tổng hợp, tránh scan toàn bộ `learning_events` mỗi lần mở màn tiến độ.
- **KHÔNG tự quyết** thang điểm chuẩn hóa (câu hỏi mở #3) và mẫu số `%` (câu hỏi mở #2) — chờ chốt trước khi thiết kế công thức.
- **Nhất quán tiền lệ:** giữ đồng bộ với `<API_REPO>/docs/active/personalized-study-schedule/plan.md` (feature này là Phase 3 của plan đó).
