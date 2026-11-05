---
stream: 5
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /reading/summary
status: todo
---

# Stream 5 — Reading summary (coverage passage)

**Mục tiêu:** endpoint coverage passage reading. Attempt đã ở mức passage (có `score.percent`) → đơn giản hơn Listening.

## Đọc trước
- `exe-api/services/api/src/modules/reading/reading-attempt.model.js:21-34` — `stimulusId`, `score{raw,total,percent}`, timestamps.
- `exe-api/services/api/src/modules/reading/reading.service.js:16,32,48` — passage = `AssessmentStimulus`, lọc `READING_BANK_FILTER = {'provenance.importBatch': /^reading-bank-/}` + `status`.
- `design.md §1-§2`.

## DTO trả về (feature='reading', baselineType='coverage')
- `done` = số distinct `stimulusId` user đã có reading-attempt (submit).
- `total` = count `AssessmentStimulus` khớp `READING_BANK_FILTER` + published.
- `completionPct = total ? round(done/total*100) : 0`; `unit='passage'`.
- `secondary = {label:'điểm TB', value: round(avg best percent mỗi passage)+'%'}` (best percent/passage = max score.percent theo stimulusId).
- `todayTask` = passage bank chưa làm đầu tiên: `{label:'Bài Reading mới', count:1, href:'/reading'}`.
- `activeDates` = distinct ngày createdAt attempt, 30d.

## Tasks (TDD)
- [ ] Test `getSummary(userId)`: seed bank 5 stimulus, user attempt 2 (một cái 2 lần điểm khác nhau → best), assert done=2, avg best percent. → verify: `npm test -- reading`
- [ ] Thêm `getSummary` vào `reading.service.js`: distinct stimulusId attempts + best percent/passage; count bank published; activeDates. Tái dùng `READING_BANK_FILTER` sẵn có (DRY).
- [ ] Controller + `toDto`.
- [ ] Route `GET /summary` (verifyToken) → `GET /reading/summary`.
- [ ] Smoke test. → verify: `npm test -- reading`

## Acceptance
- [ ] `done` = distinct passage (không đếm số lần attempt).
- [ ] `secondary` dùng best percent/passage rồi mới avg (không avg mọi attempt).

**Status khi xong:** DONE + tóm tắt.
