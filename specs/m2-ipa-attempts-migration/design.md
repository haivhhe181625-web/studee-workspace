---
feature: M2 IPA — SRS review flow (Phần B của m2-ipa-attempts-migration)
status: approved (2026-08-15) — F1–F5 duyệt mặc định, sẵn sàng code Phần B
created: 2026-08-15
---

# Design — SRS review flow (Phần B)

> Chỉ cho **Phần B** (luồng ôn SRS). Phần A (migrate `/attempts`) không cần design.
> BE đã sẵn sàng: `POST /attempts {kind:"srs_review", payload:{score}}` →
> `recordReview` (`ipa.service.js:616`) → `correct = score ≥ masteredThreshold(80)` →
> `srsService.recordReviewResult` đẩy nhịp `advanceReview` (1d→7d→mastered). FE chỉ thiếu
> **UI + đường gọi**.

## Ràng buộc từ BE (đã verify, không đổi)
- `recordReview` nhận **1 số `score` (0–100)**, tự quyết đạt/không theo ngưỡng 80.
- Nhịp SRS: pass nhịp 1 → hẹn 7d; pass nhịp 2 → `status=mastered`; trượt → giữ bài +
  `needsReview`, không thu hồi.
- `GET /ipa/srs/due` trả `{lessons:[{lessonId, lastScore, nextReviewDate}]}` (đã có ở FE service).

## Các fork thiết kế + đề xuất (cần chốt)

### F1. Entry point luồng ôn
- **Đề xuất:** thêm khối **"Ôn tập hôm nay"** trên trang `/pronunciation` (cạnh Section Map),
  list bài từ `GET /srs/due`; bấm → vào runner **mode review**.
- **Route:** tái dùng `/pronunciation/[lessonId]` + query `?mode=review` (ít churn routing
  nhất, dùng lại page + runner). *Cân nhắc khác:* route riêng `/review` — bác vì thêm page thừa.

### F2. `buildSteps(mode)` — bỏ màn Học (AC-007)
- `mode="lesson"` → `[hoc, luyennoi, result]` (như hiện tại).
- `mode="review"` → `[luyennoi, result]` (bỏ `hoc`).
- `LessonRunnerContainer` nhận prop `mode` (page đọc từ `searchParams`).

### F3. Vòng luyện nói "rút gọn" (60–90s)
- **Đề xuất:** review dùng lại `SpeakingLoopStep` với cờ `review`:
  - **Bỏ khối cảm âm** (đã qua cổng khi lên `learning`) → không lặp 4 câu phân biệt.
  - **Cap số mục** để đủ 60–90s: lấy **tối đa 3 mục** (ưu tiên câu > từ) thay vì toàn bộ.
    Con số 3 là **tunable** — cần chốt.
- *Cân nhắc khác:* giữ nguyên toàn bộ mục — bác vì bài nhiều mục sẽ quá 90s.

### F4. Nguồn `score` gửi `srs_review` ⭐ (fork chính)
- **Đề xuất (Option A — cục bộ, cô lập):** mỗi mục ôn submit `/attempts {kind:word|sentence}`
  trả `IpaSubmitResult.score`; FE **trung bình điểm các mục đã chấm** trong vòng ôn → gửi
  làm `score`. Không phụ thuộc `GET /result`.
  - **Side-effect chấp nhận được:** submit mục ôn vẫn cập nhật `screenScores` của bài (BE
    `submitItem`), nhưng **graduation SRS chỉ do post `srs_review` quyết** (không phải từ
    `computeLessonResult`) → không xung đột. Ghi comment *why* ở code.
- *Option B (bác):* sau vòng ôn `GET /result` lấy `overallScore` — bác vì trộn điểm cũ+mới
  trong `screenScores`, phản ánh sai lần ôn.

### F5. KHÔNG gọi `/start` ở mode review ⚠️
- Runner hiện gọi `POST /lessons/:id/start` khi vào bài (`LessonRunnerContainer.tsx:56`).
- **Rủi ro:** `/start` có thể reset `screenScores`/`status` → **hỏng tiến độ đã học**.
- **Đề xuất:** mode review **bỏ hẳn** `/start`. Task T-verify phải đọc `startLesson`
  (`ipa.service.js`) xác nhận nó reset gì trước khi quyết (nếu `/start` idempotent-an-toàn
  thì có thể giữ; mặc định **bỏ** cho chắc).

### F6. Post `srs_review` + màn kết quả ôn
- **Đề xuất:** vào step `result` của mode review → fire **đúng 1 lần** `srs_review {score}`
  (guard chống double-post như `started.current`), rồi hiện thông điệp ôn: đạt → "Đã lên nhịp
  kế / Đã thành thạo"; trượt → "Cần luyện thêm, không mất tiến độ".
- **CHỐT (user 2026-08-15):** dùng **component riêng `ReviewResultStep`** thay vì nhồi nhánh
  vào `ResultStep` — màn ôn không gọi `GET /result`, gần như không dùng chung UI với màn kết
  quả bài học → tách riêng đơn giản + surgical hơn, giữ `ResultStep` nguyên vẹn. Hành vi giống
  hệt ý F6 (post 1 lần, thông điệp đạt/trượt; thông điệp lấy từ kết quả BE `recordReview`).

## Máy trạng thái (tham chiếu, do BE lo)
```
learning --ôn nhịp1 đạt(≥80)--> hẹn 7d --ôn nhịp2 đạt--> mastered
   ^                                |
   +----------- trượt --------------+  (giữ bài, needsReview, không thu hồi)
```

## Sign-off — ĐÃ DUYỆT (2026-08-15, mặc định)
- [x] F1 route `?mode=review` + entry "Ôn tập hôm nay" ở `/pronunciation`.
- [x] F3 cap **3 mục** cho vòng ôn (tunable).
- [x] F4 **Option A** — `score` = trung bình điểm mục đã chấm.
- [x] F5 **GIỮ `/start`** ở mode review — T5.1 verified: `startLesson` (`ipa.service.js:113-126`)
  chỉ `$setOnInsert` + upsert → với progress đã tồn tại là **no-op** (không reset
  `screenScores`/`status`/`overallScore`). Rủi ro F5 bị bác → giữ `/start` (idempotent-safe)
  đơn giản hơn (không cần nhánh điều kiện trong runner).
