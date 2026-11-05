---
stream: 2
feature: learning-progress-dashboard
repo: exe-api
endpoint: GET /ipa/summary
status: todo
---

# Stream 2 — IPA summary (coverage lessons)

**Mục tiêu:** endpoint trả `FeatureProgress` coverage bài IPA. Lưu ý: `overallScore` KHÔNG phải coverage — nó là điểm chất lượng (weighted-avg 5 màn). Coverage = bài "done" / tổng bài published.

## Đọc trước
- `exe-api/services/api/src/modules/ipa/ipa-progress.model.js:28` — `status` enum (not_started/in_progress/completed/mastered), `overallScore:32`, `needsReview:35`.
- `exe-api/services/api/src/modules/ipa/ipa.service.js:129` — `getProgress` (list per lesson).
- Model lesson IPA published (mẫu số): tìm model lesson/screen definition, đếm `status='published'`.
- `design.md §1-§2`.

## DTO trả về (feature='ipa', baselineType='coverage')
- `done` = count `ipa_progress{userId, status ∈ {completed, mastered}}`.
- `total` = count IPA lesson `status='published'`.
- `completionPct = total ? round(done/total*100) : 0`; `unit='bài'`.
- `secondary = {label:'điểm TB', value: round(avg overallScore các bài đã có điểm)}`.
- `todayTask` = nếu needsReview>0: `{label:'Ôn âm cần review', count: needsReviewCount, href:'/pronunciation'}`; else nếu còn not_started: `{label:'Bài phát âm mới', count:1, href:'/pronunciation'}`.
- `activeDates` = distinct ngày updatedAt của `ipa_progress` user (hoặc `ipa-attempt`), 30d.

## Tasks (TDD)
- [ ] Test `getSummary(userId)`: seed lessons published + progress docs với các status khác nhau, assert done (completed+mastered), total, needsReview count. → verify: `npm test -- ipa`
- [ ] Thêm `getSummary` vào `ipa.service.js`: 2 count query (progress theo status, lesson published) + avg overallScore + needsReview + activeDates.
- [ ] Controller + `toDto` → FeatureProgress.
- [ ] Route `GET /summary` vào `ipa.routes.js` (verifyToken) → path cuối `GET /ipa/summary`.
- [ ] Smoke test route. → verify: `npm test -- ipa`

## Acceptance
- [ ] `done` gồm CẢ `completed` và `mastered` (không chỉ mastered).
- [ ] `secondary` là avg overallScore, làm tròn.
- [ ] Không chia 0 khi chưa có lesson published.

**Status khi xong:** DONE + tóm tắt.
