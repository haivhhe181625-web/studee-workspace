---
stream: 6
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /test-paper/summary
status: todo
---

# Stream 6 — Luyện đề summary (coverage đề)

**Mục tiêu:** endpoint coverage đề thi thử + best score + điểm yếu. Hiện chỉ có `/practice-attempts/in-progress`.

## Đọc trước
- `exe-api/services/api/src/modules/test-paper/practice-attempt.model.js:13-52` — `testPaperId`, `status ('in_progress'|'submitted'|'expired')`, `score{percent}`, `weakByItemType[]{itemType,correct,total,percent}`.
- `exe-api/services/api/src/modules/test-paper/test-paper.model.js:19-35` — `TestPaper`, `status='published'`, mẫu số = count published.
- `design.md §1-§2`, §8 (chỉ tính `PracticeAttempt`, KHÔNG gộp QuizAttempt).

## DTO trả về (feature='exam', baselineType='coverage')
- `done` = distinct `testPaperId` có attempt `status='submitted'`.
- `total` = count `TestPaper{status='published'}`.
- `completionPct = total ? round(done/total*100) : 0`; `unit='đề'`.
- `secondary = {label:'điểm cao nhất', value: round(max score.percent)+'%'}`.
- `todayTask`: nếu có attempt `in_progress` → `{label:'Làm tiếp đề đang dở', count:1, href:'/exam'}`; else nếu còn đề chưa làm → `{label:'Đề thi thử mới', count:1, href:'/exam'}`.
- `activeDates` = distinct ngày submittedAt, 30d.
- (tùy) gắn thêm `weakItemTypes` gộp vào `secondary` hoặc field phụ — merge `weakByItemType` các attempt, sort theo percent thấp nhất, lấy top 2 tên itemType.

## Tasks (TDD)
- [ ] Test `getSummary(userId)`: seed 4 paper published, user submit 2 (điểm khác nhau) + 1 in_progress, assert done=2, best percent, todayTask=resume. → verify: `npm test -- test-paper`
- [ ] Thêm `getSummary` vào service (`practice-attempt.service.js` hoặc `test-paper.service.js`): distinct submitted testPaperId, max percent, detect in_progress, merge weakByItemType, count paper published, activeDates.
- [ ] Controller + `toDto`.
- [ ] Route `GET /summary` (verifyToken) → `GET /test-paper/summary` (đặt trong `test-paper.routes.js` hoặc `practice-attempt.routes.js`).
- [ ] Smoke test. → verify: `npm test -- test-paper`

## Acceptance
- [ ] CHỈ tính `PracticeAttempt` (không QuizAttempt).
- [ ] `todayTask` ưu tiên resume in_progress trước đề mới.

**Status khi xong:** DONE + tóm tắt.
