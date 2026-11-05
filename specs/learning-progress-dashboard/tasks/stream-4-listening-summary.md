---
stream: 4
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /listening/summary
status: done
---

# Stream 4 — Listening summary (coverage passage)

**Mục tiêu:** endpoint coverage passage. Hiện chỉ có per-item attempt, chưa có khái niệm "passage done" → phải suy ra.

## Đọc trước
- `exe-api/services/api/src/modules/listening/listening-item-attempt.model.js` — fields `passageId, itemId, score (0-1), isCorrect, status ('submitted')`, timestamps.
- `exe-api/services/api/src/modules/listening/listening-passage.model.js:162` — `ListeningPassage`, `status` enum, mẫu số = count published.
- Cần biết tổng item/passage để xác định "đủ item": lấy từ passage doc (danh sách item) hoặc count item bank theo passageId.
- `design.md §1-§2`.

## Rule "passage done" (ĐÃ CHỐT)
Passage done khi: **mọi item của passage đã có attempt `status='submitted'` VÀ avg `score` các item đó ≥ 0.5**.

## DTO trả về (feature='listening', baselineType='coverage')
- `done` = số distinct passageId thoả rule done.
- `total` = count `ListeningPassage{status='published'}`.
- `completionPct = total ? round(done/total*100) : 0`; `unit='passage'`.
- `secondary = {label:'accuracy TB', value: round(avg score mọi item submitted *100)+'%'}`.
- `todayTask` = passage published chưa done đầu tiên: `{label:'Bài Listening mới', count:1, href:'/listening'}` (hoặc retry passage accuracy thấp nếu muốn).
- `activeDates` = distinct ngày submittedAt, 30d.

## Tasks (TDD)
- [x] Test `getSummary(userId)`: seed 1 passage 3 item — case đủ 3 item & avg≥0.5 → done; case thiếu item → not done; case đủ item nhưng avg<0.5 → not done. → verify: `npm test -- listening`
- [x] Thêm `getSummary` vào `listening-summary.service.js` (file mới; `listening.service.js` không tồn tại): aggregate item-attempts group theo passageId (addToSet itemId + avg score), so với tổng item published của passage; đếm done; count passage published; activeDates.
- [x] Controller + DTO build.
- [x] Route `GET /summary` (verifyToken) → `GET /listening/summary`.
- [x] Smoke test. → verify: `npm test -- listening`

## Acceptance
- [x] Rule done áp đúng: cần CẢ đủ item VÀ accuracy≥0.5.
- [x] Ngưỡng 0.5 để hằng số có tên rõ (comment English), dễ chuyển app-setting sau.

**Status khi xong:** DONE + tóm tắt. Nếu "tổng item/passage" không lấy được từ passage doc → BLOCKED, báo rõ nguồn thiếu.
