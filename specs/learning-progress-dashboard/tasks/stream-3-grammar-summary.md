---
stream: 3
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /grammar/summary
status: done
---

# Stream 3 — Grammar summary (wrap logic sẵn có)

**Mục tiêu:** endpoint mỏng map `getMasterySummary` hiện có → `FeatureProgress`. KHÔNG đổi logic tính mastery.

## Đọc trước
- `exe-api/services/api/src/modules/grammar-practice/grammar-practice.service.js:58-72` — `getMastery` (itemsMastered=best===1.0 / itemsPublished).
- `...:93-111` — `getMasterySummary` (gộp mọi grammarPoint user đã đụng).
- `grammar-practice.routes.js:26-28` — route hiện có `/mastery`, `/mastery/summary`.
- `design.md §1-§2`.

## DTO trả về (feature='grammar', baselineType='mastery')
- Gọi `getMasterySummary(userId)`, roll-up: `done = Σ itemsMastered`, `total = Σ itemsPublished` mọi grammarPoint.
- `completionPct = total ? round(done/total*100) : 0`; `unit='điểm ngữ pháp'`.
- `secondary = {label:'đến hạn', value: dueReviewCount}` (nếu summary có due; nếu chưa, đếm nhanh hoặc bỏ).
- `todayTask` = nếu dueReview>0: `{label:'Ôn ngữ pháp đến hạn', count: dueReview, href:'/grammar'}`.
- `activeDates` = distinct ngày `grammar_attempts` user, 30d.

## Tasks (TDD)
- [x] Test `getSummary(userId)`: mock/seed để `getMasterySummary` trả nhiều point, assert roll-up done/total/pct. → verify: `npm test -- grammar`
- [x] Thêm `getSummary` vào `grammar-practice.service.js` (gọi lại getMasterySummary, cộng dồn + activeDates). DRY — không viết lại công thức mastery.
- [x] Controller + `toDto`.
- [x] Route `GET /summary` (verifyToken) → `GET /grammar/summary`.
- [x] Smoke test. → verify: `npm test -- grammar`

## Acceptance
- [x] Roll-up đúng tổng (không trung bình các pct — phải Σmastered/Σpublished).
- [x] Tái dùng logic sẵn có, không copy công thức.

**Status khi xong:** DONE + tóm tắt.
