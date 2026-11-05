---
feature: learning-progress-dashboard
status: draft
role: TechLead
created: 2026-08-25
---

# Tasks — index & chạy song song

8 stream **độc lập** (không chia sẻ file — xem `design.md §7`). Mở nhiều terminal, mỗi terminal chạy 1 stream:

```
/superpowers execute studee-workspace/specs/learning-progress-dashboard/tasks/stream-1-flashcard-summary.md
/superpowers execute studee-workspace/specs/learning-progress-dashboard/tasks/stream-2-ipa-summary.md
...
```

## Sơ đồ phụ thuộc

- **Không có phụ thuộc chạy giữa các stream** — cả 8 code theo DTO đóng băng ở `design.md §1`.
- Stream 8 (FE) tích hợp thật cần endpoint stream 1-7, nhưng **bắt đầu ngay được** với mock adapter theo DTO; nối API khi sẵn sàng.
- Điều kiện tiên quyết chung: đã đọc `design.md` (DTO + công thức + ranh giới file).

## Danh sách stream

| # | Stream | Repo | File task | Chi phí |
|---|---|---|---|---|
| 1 | Flashcard summary (gộp deck) | exe-api | `tasks/stream-1-flashcard-summary.md` | nhẹ |
| 2 | IPA summary (coverage) | exe-api | `tasks/stream-2-ipa-summary.md` | vừa |
| 3 | Grammar summary (wrap) | exe-api | `tasks/stream-3-grammar-summary.md` | nhẹ |
| 4 | Listening summary | exe-api | `tasks/stream-4-listening-summary.md` | vừa |
| 5 | Reading summary | exe-api | `tasks/stream-5-reading-summary.md` | vừa |
| 6 | Luyện đề summary | exe-api | `tasks/stream-6-exam-summary.md` | vừa |
| 7 | Talk tracking + summary | exe-api | `tasks/stream-7-talk-tracking-summary.md` | nặng |
| 8 | Dashboard FE | exe-web | `tasks/stream-8-web-dashboard.md` | vừa |

## Nguyên tắc chung mọi stream BE (exe-api)
- Theo 5-file module pattern; auth `verifyToken`; response qua `toDto()`; service throw `ApiError` + controller `asyncHandler`.
- KHÔNG sửa `src/app.js` (route mount qua file routes của module đã mount sẵn).
- Trả đúng DTO `FeatureProgress` (`design.md §1`). Code + comment tiếng Anh.
- Test: unit cho service (công thức) + smoke cho route. Chạy `npm test` trong `exe-api/services/api` (xem `exe-api/TESTING.md`).

## Definition of done toàn feature
- [ ] 7 endpoint trả đúng DTO, test xanh.
- [ ] Dashboard render 7 thanh % + khối hôm nay + streak tổng + % tổng.
- [ ] 1 endpoint lỗi không làm vỡ dashboard (skeleton/error per card).
- [ ] Human review trước merge (gate cuối, không tự merge).
