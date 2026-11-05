---
stream: 7
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /talk/summary  (+ ghi activity ở POST /talk/turn)
status: todo
---

# Stream 7 — Talk tracking + summary (phần mới nhất)

**Mục tiêu:** Talk hiện stateless (không lưu gì). Thêm ghi nhận nhẹ để có coverage kịch bản + hoạt động. Baseline = `scenariosTried / 4`.

## Đọc trước
- `exe-api/services/api/src/modules/talk/talk.service.js:170-204` — `turn()`: quota consume/refund + `_executeTurn`. Ghi activity SAU khi turn thành công.
- `talk.service.js:191` — `consumeQuota` trả `consumed` (chỉ inc khi thực sự trừ → dùng làm mốc idempotent).
- `talk.scenarios.js:18` — `SCENARIO_IDS = ['coffee','interview','travel','freetalk']` → total=4.
- `design.md §4` (chi tiết model + write path).

## Model mới `talk-activity.model.js` (collection `talk_activity`)
```
{ userId, date: 'YYYY-MM-DD', scenarios: [String], turns: Number }  // unique {userId,date}
```

## Write path (sửa `talk.service.js`)
- Sau `_executeTurn` thành công (trong `turn()`, path không throw): upsert `{userId, date: today}` với `$addToSet: {scenarios: scenarioId}`, `$inc: {turns: 1}`.
- Chỉ ghi khi quota thực sự consumed (không double-count khi retry cùng turnId). Không ghi ở nhánh refund/throw.
- Giữ nguyên hành vi hiện tại; chỉ THÊM 1 write, không đổi response turn.

## `GET /talk/summary` (feature='talk', baselineType='coverage')
- `scenariosTried` = distinct scenarios mọi doc user (`$addToSet` qua aggregate).
- `done = scenariosTried.length`; `total = 4`; `unit='kịch bản'`.
- `completionPct = round(done/4*100)`.
- `secondary = {label:'lượt hôm nay', value: turns doc ngày hôm nay || 0}`.
- `todayTask` = kịch bản trong SCENARIO_IDS chưa có trong scenariosTried (đầu tiên): `{label:'Thử kịch bản '+tênVi, count:1, href:'/talk'}`; nếu đủ 4 → gợi ý freetalk luyện thêm.
- `activeDates` = distinct date doc user, 30d.

## Tasks (TDD)
- [ ] Test model + write: gọi turn thành công 2 lần (coffee, travel) → `talk_activity` hôm nay có scenarios [coffee,travel], turns=2. Retry cùng turnId không tăng turns. → verify: `npm test -- talk`
- [ ] Tạo `talk-activity.model.js` (5-file pattern; model có thể đứng riêng, schema Joi cho summary nếu cần).
- [ ] Sửa `talk.service.js turn()` thêm upsert activity (sau success, gate theo consumed).
- [ ] Thêm `getSummary(userId)` (aggregate scenariosTried + todayTurns + activeDates).
- [ ] Controller + route `GET /summary` (verifyToken) → `GET /talk/summary`.
- [ ] Smoke test cả write lẫn summary. → verify: `npm test -- talk`

## Acceptance
- [ ] `POST /talk/turn` không đổi response, chỉ thêm side-effect ghi activity, gate idempotent.
- [ ] `total` luôn = `SCENARIO_IDS.length` (đọc từ scenarios, không hardcode số 4 rời).
- [ ] Turn lỗi/refund KHÔNG ghi activity.

**Status khi xong:** DONE + tóm tắt.
