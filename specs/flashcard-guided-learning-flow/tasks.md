---
feature: flashcard-guided-learning-flow
status: draft
role: TechLead
created: 2026-08-25
---

# Tasks — index & chạy song song

7 stream. Mỗi terminal chạy 1 stream qua superpowers:
```
/superpowers execute studee-workspace/specs/flashcard-guided-learning-flow/tasks/stream-1-today-session-endpoint.md
...
```

## Phụ thuộc
- BE: stream 1b (field `bestRecallLevel` + cập nhật review) là **nền** cho stream 1 (endpoint đọc levels). 2, 2b độc lập.
- FE stream 3-6 code theo contract `design.md §2-§4`, mock được; tích hợp khi BE xong.
- Stream 5 (thanh 4 màu) đọc `progress.levels/slipping` từ stream 1; stream 3/6 dùng menu + PATCH.
- Tiên quyết chung: đọc `design.md` (thang màu §1, contract §2, menu §3, thanh §4) + `spec.md` (Acceptance §5).

## Danh sách
| # | Stream | Repo | Trạng thái | Chi phí |
|---|---|---|---|---|
| 1b | Recall-level tracking: field `bestRecallLevel` + cập nhật `applyReviewEvent` (leo nấc/lapse reset) + cờ `crossedMastery` | exe-api | **mới** | vừa |
| 1 | Today-session endpoint: `progress{levels,slipping,mastered}` + dueCards(+level) + newRemaining | exe-api | **sửa nhiều** | vừa |
| 2 | new-limit setting (PATCH /me) | exe-api | giữ (xong) | nhẹ |
| 2b | Bỏ thưởng practice + vnDayKey | exe-api | giữ (xong) | nhẹ |
| 3 | StudyMenu (Học mới/Ôn/Kiểm tra/Luyện tự do), gỡ funnel | exe-web | **thay** | nặng |
| 4 | Lật thẻ 3 nút (again/good/easy) | exe-web | sửa | vừa |
| 5 | Thanh 4 màu + đoạn "đang tụt" kẻ sọc + chip 4 nấc | exe-web | **thay** | vừa |
| 6 | new-limit chọn-rồi-Bắt-đầu (5/10/20 + nút Bắt đầu) | exe-web | sửa | nhẹ |

> File stream-N chi tiết viết lại sau khi PO duyệt design (bản này thay đổi cấu trúc so với 25/08).

## Nguyên tắc chung
- BE: 5-file module pattern, verifyToken, envelope `{data}`, ApiError+asyncHandler, userId từ JWT. KHÔNG đổi engine FSRS/scheduler/4 runner — chỉ chèn cập nhật `bestRecallLevel` vào `applyReviewEvent` + guard gamification. Test jest (`npx jest flashcard --runInBand`).
- **Nấc màu ≠ FSRS**: `bestRecallLevel` cập nhật bất kể `practice` (trục hiển thị riêng, không bơm stability). Chỉ practice=false đẩy `due`/XP.
- **Cờ practice**: Học mới + Ôn đến hạn = `practice=false` (thật, đẩy FSRS + XP + lấp tụt); Kiểm tra + Luyện tự do = `practice=true` (leo nấc, không FSRS/XP).
- FE: TS strict, TanStack Query, token app-* (không hex/any), service→hook→component. Dùng lại 4 runner + `POST /cards/:id/review`. Giữ test runner cũ xanh. TDD.
- Reset ngày VN: `utils/vn-date.js`.

## Definition of done
- [ ] Menu 4 hành động; KHÔNG ép chuỗi. Học mới = chọn N + bấm Bắt đầu.
- [ ] `bestRecallLevel` leo đúng (flip/match/quiz/type→1/2/3/4, easy→3); lapse→1; Kiểm tra sai không phạt.
- [ ] `progress.levels` (chưa tụt) + `slipping` (due≤now) đúng bất biến; `mastered` (⭐, stability≥21) trục riêng; thanh 4 màu + đoạn kẻ sọc + chip ⭐ khớp.
- [ ] `crossedMastery` khi thẻ lần đầu vượt 21 ngày → FE chúc mừng.
- [ ] introducedToday theo ngày VN (createdAt); new-limit 5/10/20 qua PATCH /me.
- [ ] Practice/Kiểm tra KHÔNG cộng XP/quest/streak; Học mới + Ôn vẫn thưởng; guest phòng chung giữ thưởng.
- [ ] Gỡ funnel; test 4 runner cũ + test mới xanh. Human review trước merge.
