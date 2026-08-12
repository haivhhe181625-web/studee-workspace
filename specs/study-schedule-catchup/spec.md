# Spec: Lịch hôm nay + đẩy lùi lịch khi trễ (Study Schedule Catch-up · M4-4)

- **Ngày:** 2026-08-06
- **Tác giả:** AI Brainstorm (từ PRD-89 / M4-4, owner Vũ Hồng Hải)
- **Trạng thái:** Các ngã rẽ đã CHỐT (owner 2026-08-06) → sẵn sàng Task Generation. Còn 2 mục kỹ thuật nhỏ cho Technical Review (§8).
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — nhiệm-vụ-hôm-nay + đánh dấu ngày lỡ + đẩy lùi lịch + cảnh báo hạn.
  - `web` (`exe-web`) — **ngoài phạm vi** (màn "Hôm nay" + nút "Dời lịch" = slice sau); spec chỉ định nghĩa contract.
- **Module liên quan (exe-api):**
  - `src/modules/adaptive/` — `LessonSchedule` (lịch đã persist, `days[].items[]` đã kèm `courseSlug/phaseKey/moduleKey/lessonKey`), `schedule/lesson-schedule.generator.js` (thuần, deterministic).
  - `src/modules/course-content/` — **`CourseEnrollment` là nguồn hoàn-thành bài** (`lessons` Map `"<phaseKey>/<moduleKey>/<lessonKey>" → {status, completedAt}`); `recordLesson()` + route `POST /course-content/:slug/lessons/progress` **đã tồn tại**; `getPassedBoundaries()` cho checkpoint.
  - `src/modules/user/` — `User.profile.studyPlan` (`learningDows/reviewDow/focusDurationMin/deadline`). **Không có field timezone** (dùng `?tz=`).
- **Backlog ref:** PRD-89 (M4-4) · SRS M4 FR-007/FR-008 · Q10. Nối tiếp `specs/study-schedule-generation` (generator đã ship) — M4-4 là slice §Ngoài-phạm-vi đã hoãn.

## 1. Mục tiêu

FE lấy được **nhiệm vụ hôm nay** (theo giờ người học); người học **tick hoàn thành** (qua đường sẵn có); nghỉ vài ngày quay lại thì các ngày lỡ **hiện đỏ**, và người học chủ động **"Dời lịch"** để **đẩy toàn bộ phần còn lại về sau** (giữ nguyên cường độ, không nhồi, không bỏ bài nào). Nếu có deadline mà đẩy lùi làm trễ hạn → **cảnh báo mềm + gợi ý** (giãn hạn / tăng phút/ngày), không chặn.

## 2. Quyết định lõi (owner 2026-08-06)

| # | Quyết định |
|---|---|
| Nguồn "done" | ✅ Tái dùng `CourseEnrollment`/`recordLesson`. **Không** store done mới, **không** endpoint tick riêng — FE tick qua `POST /course-content/:slug/lessons/progress`. "done" = join item lịch ↔ enrollment. |
| Ngày ôn (`kind:'review'`) | ✅ **Chỉ hiển thị**, không nút tick, không tính vào `done`/tiến độ (A1). Checkpoint: "done" = boundary đã PASS (`getPassedBoundaries`). |
| Cơ chế sắp lại | ✅ **ĐẨY LÙI timeline** — giữ nguyên `focusDurationMin`/ngày, dời phần chưa-done bắt đầu từ hôm nay. **Không nhồi, KHÔNG trần 150%.** |
| Skip / bỏ bài | ✅ **KHÔNG có** — luôn đi hết, không bỏ nội dung. |
| Trigger + timezone | ✅ **Lazy-on-read** (không cron slice này); TZ = client gửi `?tz=` mỗi request. |
| Deadline khi trễ | ✅ **Cảnh báo mềm + gợi ý** (giãn hạn / tăng phút/ngày), **không** prompt chặn, **không** nút bỏ. |

## 3. Deviation có chủ đích so với PRD-89 (cần BA ký nhận)

| PRD-89 ghi | M4-4 làm | Vì sao |
|---|---|---|
| "job đầu ngày **dồn ≤150%**" | **Đẩy lùi timeline thủ công**, giữ cường độ, bỏ trần 150% | Owner: nhồi 150% dễ quá tải/nản; đẩy lùi nhẹ nhàng hơn, người học nắm quyền |
| "**bỏ phần lỡ**" (1 trong 3 phương án) | **Bỏ hẳn** phương án skip | Owner: app học tích lũy, không cho bỏ nội dung |
| "vượt kéo dài → **hỏi người học** (giãn hạn/tăng phút/bỏ)" | **Cảnh báo mềm + gợi ý** (giãn hạn/tăng phút), không chặn, không nút bỏ | Không skip nên chỉ còn 2 lối; không ép luồng |

## 4. Phạm vi

### Trong phạm vi

- **IS-1 — Nhiệm vụ hôm nay (`GET /schedule/today?tz=`):** item của ngày-local hôm nay, mỗi item học annotate `done` (join `CourseEnrollment`); ngày ôn hiển thị (không `done`); tiến độ ngày `doneCount/totalCount` + `remainingMin`; **khối `missed`** (các ngày quá hạn còn item học chưa done → FE tô đỏ) + cảnh báo hạn.
- **IS-2 — Hoàn-thành:** tái dùng `POST /course-content/:slug/lessons/progress` sẵn có; M4-4 chỉ đảm bảo item lịch trỏ đúng khóa để `today` phản ánh done. **Không endpoint mới.**
- **IS-3 — Đẩy lùi lịch (`POST /schedule/postpone`):** sinh lại phần **bài chưa-done** bắt đầu từ hôm nay bằng generator thuần (cường độ = `focusDurationMin`, **không ×1.5**); **giữ các ngày đã qua trong doc** (chỉ giữ item đã done — lịch sử hoàn thành), thay phần từ hôm nay trở đi. Sau đó không còn ngày lỡ (đỏ) actionable. Deterministic → bấm lại an toàn.
- **IS-4 — Cảnh báo hạn (mềm):** nếu có `deadline` và ngày-xong-dự-kiến > `deadline` → gắn `warning{ overdue:true, suggestions:['giãn hạn tới …','tăng lên … phút/ngày'] }` (tái dùng `WarningSchema` sẵn có). Không deadline → không cảnh báo. Người học tự sửa qua `PUT /schedule/prefs` sẵn có nếu muốn.
- **IS-5 — Ngủ đông khi nghỉ quá lâu (dormant):** nếu **số ngày bất hoạt** (hôm-nay − `User.profile...lastActiveDate`) ≥ `DORMANT_INACTIVE_DAYS` (đề xuất **14**, tinh chỉnh sau) → `GET /schedule/today` trả `status:'dormant'` + banner "đã tạm dừng, quay lại bất cứ lúc nào" **thay cho** danh sách ngày-đỏ (đỡ spam). **Giữ nguyên mọi data.** Quay lại (postpone / hoàn thành 1 bài → `lastActiveDate` cập nhật) → tự về `status:'active'` + gợi ý "Dời lịch". **Không auto-đuổi khỏi khóa.** Trạng thái **tính lúc đọc** từ `lastActiveDate` — **không lưu field, không ghi `CourseEnrollment`**; M4-5 cron tự tính cùng công thức để **loại dormant khỏi vòng quét nhắc học** (đó mới là chỗ tiết kiệm chi phí thật, vì M4-4 lazy-on-read vốn tốn 0 cho user nghỉ).

> **Cơ sở (edtech):** Duolingo/Coursera/Khan/Babbel đều **ngủ đông + giữ data + nhắc thưa dần**, không hệ nào trục xuất vì nghỉ. Thu hồi ghế B2B (nếu cần) là **thao tác thủ công của admin center**, module riêng — ngoài M4-4.

### Ngoài phạm vi

- **Cron / nhắc học + loại dormant khỏi vòng quét (M4-5)**, **skip/bỏ nội dung**, **nhồi ≤150%**, **ôn thông minh v3**, **track thời gian thực tế**, **UI exe-web**, **lưu timezone**, **đa lộ trình/người**.
- **Auto-đuổi khỏi khóa** — không làm (phá retention). **Thu hồi ghế B2B** = nút thủ công admin center, module riêng.
- **Lưu cờ dormant / dashboard đếm dormant** — M4-4 tính lúc đọc; lưu để phân tích (nếu cần) là việc M4-5/admin.
- **Đổi lõi generator** — chỉ *gọi lại*, không refactor.

## 5. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner | Mở app thấy đúng "việc hôm nay" theo giờ mình | Biết học gì hôm nay |
| Learner | Tick xong 1 bài, tiến độ hôm nay cập nhật | Theo dõi hoàn thành |
| Learner (nghỉ vài ngày) | Thấy ngày lỡ đỏ, bấm "Dời lịch" → lịch đẩy về sau, giữ cường độ | Học tiếp không quá tải, không mất bài |
| Learner (có hạn) | Được cảnh báo nếu sẽ trễ hạn + gợi ý | Tự quyết giãn hạn / tăng cường độ |

## 6. Acceptance criteria

1. **AC-1** (IS-1): `GET /schedule/today?tz=…` + JWT → 200 item của ngày-local hôm nay, item học có `done` khớp `CourseEnrollment`; ngày rỗng → `items:[]`; no JWT → 401.
2. **AC-2** (IS-1): item học `completed`/`mastered` → `done:true`; chưa → `false`. Ngày ôn hiển thị nhưng **không** có `done`/không tính vào `totalCount`. `doneCount/totalCount/remainingMin` đúng.
3. **AC-3** (IS-2): tick 1 bài qua `POST /course-content/:slug/lessons/progress` → gọi lại `today` → bài đó `done:true` (không store mới).
4. **AC-4** (IS-1): lịch có ngày `< hôm-nay` còn item học chưa done → `today.missed{ count>0, oldestDate, days[] }`; hết → `missed:null`. Ngày ôn/checkpoint quá hạn **không** tính là missed.
5. **AC-5** (IS-3): `POST /schedule/postpone` → bài chưa-done xếp lại từ hôm nay; **không ngày nào vượt `focusDurationMin`** (trừ bài quá khổ vốn độc chiếm ngày — không ×1.5); sau đó `today.missed:null`.
6. **AC-6** (IS-3 · AC chính PRD): mô phỏng **nghỉ 7 ngày** rồi postpone → mọi bài chưa-done dời bắt đầu hôm nay, cường độ giữ nguyên, lịch hợp lệ (không cắt đôi bài, ngày ôn/checkpoint sinh lại đúng), ngày-xong lùi ~7 ngày.
7. **AC-7** (IS-3 determinism/idempotent): cùng (bài-chưa-done + prefs + `asOfDate`) → cùng lịch; bấm postpone **2 lần** → kết quả y hệt (không nhân đôi).
8. **AC-8** (IS-4): có `deadline` và finish-dự-kiến > deadline → `warning.overdue:true` + `suggestions` gồm hạn-mới + phút/ngày-cần; không deadline hoặc kịp → `warning` không `overdue`.
9. **AC-9** (không skip): postpone **không loại bỏ** bài nào — tổng số bài trước/sau postpone bằng nhau (chỉ đổi ngày).
10. **AC-10** (IS-3 giữ lịch sử): sau postpone, các ngày `< hôm-nay` vẫn còn trong doc và **chỉ chứa item đã done** (lịch sử hoàn thành); item chưa-done của ngày cũ được dời sang phần từ hôm nay.
11. **AC-11** (IS-5 dormant): `lastActiveDate` cách hôm-nay ≥ ngưỡng → `today.status:'dormant'` + banner, **không** trả danh sách đỏ; < ngưỡng → `status:'active'` bình thường. Sau postpone/hoàn-thành → về `active`. Không có thao tác nào đụng `CourseEnrollment`/ghi field mới.

## 7. Ràng buộc

- **NFR-1 — lõi thuần xác định:** `buildTodayView`, việc gọi `generateSchedule`: không `Date.now()`/`Math.random()`/`new Date()` không-tham-số. `todayLocal`/`asOfDate` là tham số; TZ + ngày-local chỉ ở controller (biên).
- **`CourseEnrollment` READ-ONLY từ adaptive** — chỉ đọc, không ghi.
- **Coexist:** không đụng `StudySchedule`/service LLM cũ; không đổi contract `generate`/`suggest`/`prefs` đã ship.
- **Không thêm field mới** trên `LessonSchedule` (tái dùng `warning`/`warnings`).

## 8. Câu hỏi mở (Technical Review — không chặn khởi động)

> ✅ Chốt (owner 2026-08-06): giữ lịch sử ngày đã qua (item done) ✅ · ngủ đông giữ data, đo bằng ngày-bất-hoạt, định nghĩa ở M4-4 ✅.

- [ ] **OQ-1:** `remaining` cho postpone lấy theo **thứ tự lộ trình gốc** (resolver) rồi lọc bỏ bài đã done — đúng thứ tự học chứ? (đề xuất: có.)
- [ ] **OQ-2:** Ngưỡng `DORMANT_INACTIVE_DAYS` = **14** — owner tinh chỉnh sau (không chặn). Tín hiệu bất hoạt dùng `lastActiveDate` (hoạt-động-học); cân nhắc `max(lastActiveDate, lastLoginAt)` — Tech Lead chốt.
- [ ] **OQ-3:** Ngưỡng "behind/đỏ" (lỡ ≥3 buổi) chỉ ảnh hưởng M4-5 (mức độ nhắc); M4-4 chỉ cần phân biệt active/dormant — xác nhận không cần trạng thái "behind" riêng ở M4-4.
