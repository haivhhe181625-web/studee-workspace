# Tasks: Két lỗi sai — Mistake tracking & smart review layer

> **Cho Developer Agent:** implement theo đúng thứ tự Task 1 → 6. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task.

**Design:** `specs/mistake-review-layer/design.md`
**Lưu ý chung khi chạy lệnh:** mọi lệnh `npm`/`jest` chạy trong `exe-api/services/api`. Test integration dùng helper `src/__tests__/helpers/db.js` (`connectDb/clearDb/disconnectDb`, mongodb-memory-server). Chạy 1 file: `npm test -- <path>`.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/review/review-schedule.model.js` | Model `review_schedule` | Tạo |
| `src/modules/review/review-scheduler.js` | Leitner thuần (`initState/applyResult/isGraduated/INTERVALS`) | Tạo |
| `src/modules/review/review.service.js` | `enqueueMistakes/recordReview/getReviewQueue` | Tạo |
| `src/modules/review/__tests__/review-schedule.model.test.js` | Test model | Tạo |
| `src/modules/review/__tests__/review-scheduler.test.js` | Test scheduler (pure) | Tạo |
| `src/modules/review/__tests__/review.service.test.js` | Test service | Tạo |

Đọc (không sửa): `modules/adaptive/learning-event.model.js`, `modules/quiz/quiz-attempt.model.js`.

---

## Task 1: Model `review_schedule`

**Files:**
- Create: `src/modules/review/review-schedule.model.js`
- Test: `src/modules/review/__tests__/review-schedule.model.test.js`

- [ ] **Step 1: Viết test thất bại**

```javascript
const mongoose = require('mongoose');
const { connectDb, clearDb, disconnectDb } = require('../../../__tests__/helpers/db');
const { ReviewSchedule } = require('../review-schedule.model');

beforeAll(connectDb);
afterEach(clearDb);
afterAll(disconnectDb);

describe('ReviewSchedule model', () => {
  const base = () => ({
    userId: new mongoose.Types.ObjectId(),
    questionId: new mongoose.Types.ObjectId(),
    skill: 'reading',
    srs: { box: 0, nextReviewAt: new Date(), lastReviewedAt: null },
  });

  it('defaults status=active, origin=mistake', async () => {
    const doc = await ReviewSchedule.create(base());
    expect(doc.status).toBe('active');
    expect(doc.origin).toBe('mistake');
    expect(doc.srs.box).toBe(0);
  });

  it('enforces unique (userId, questionId)', async () => {
    const b = base();
    await ReviewSchedule.create(b);
    await expect(ReviewSchedule.create(b)).rejects.toThrow(/duplicate key/i);
  });

  it('rejects invalid status enum', async () => {
    await expect(ReviewSchedule.create({ ...base(), status: 'bogus' })).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npm test -- src/modules/review/__tests__/review-schedule.model.test.js
```
Expected: FAIL — module `review-schedule.model` chưa tồn tại.

- [ ] **Step 3: Implement**

Tạo `review-schedule.model.js` theo design §3: `skill` enum SKILLS, `subskill` enum SUBSKILLS+null, `origin` enum `['mistake','manual']` default `mistake`, `status` enum `['active','graduated']` default `active`, `lastResult` enum `['correct','incorrect']`+null. `srs` subdoc `{ box:Number default 0, nextReviewAt:Date, lastReviewedAt:{type:Date,default:null} }`. Index unique `{userId,questionId}` và `{userId,status,'srs.nextReviewAt'}`. `timestamps:true`, `collection:'review_schedule'`. Export `{ ReviewSchedule }`, có `toDto()` xoá `__v`. (SKILLS/SUBSKILLS import từ `../adaptive/learning-event.model` để nhất quán enum.)

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run: `npm test -- src/modules/review/__tests__/review-schedule.model.test.js`
Expected: 3 test PASS.

- [ ] **Step 5: Commit**

```bash
git add src/modules/review/review-schedule.model.js src/modules/review/__tests__/review-schedule.model.test.js
git commit -m "feat(review): add review_schedule model for mistake tracking"
```

---

## Task 2: Leitner scheduler (pure)

**Files:**
- Create: `src/modules/review/review-scheduler.js`
- Test: `src/modules/review/__tests__/review-scheduler.test.js`

- [ ] **Step 1: Viết test thất bại**

```javascript
const s = require('../review-scheduler');

describe('review-scheduler (Leitner)', () => {
  const now = new Date('2026-08-13T00:00:00Z');
  const days = (a, b) => Math.round((new Date(a) - new Date(b)) / 86400000);

  it('initState: box 0, due now', () => {
    const st = s.initState(now);
    expect(st.box).toBe(0);
    expect(new Date(st.nextReviewAt).getTime()).toBe(now.getTime());
  });

  it('correct raises box and pushes due out (1,3,7,14,30)', () => {
    let st = s.initState(now);
    st = s.applyResult(st, true, now); // box1 -> +1d
    expect(st.box).toBe(1);
    expect(days(st.nextReviewAt, now)).toBe(1);
    st = s.applyResult(st, true, now); // box2 -> +3d
    expect(days(st.nextReviewAt, now)).toBe(3);
  });

  it('incorrect resets box to 0 and due now', () => {
    let st = s.applyResult(s.initState(now), true, now); // box1
    st = s.applyResult(st, false, now);                  // box0
    expect(st.box).toBe(0);
    expect(days(st.nextReviewAt, now)).toBe(0);
  });

  it('graduates after passing the last interval', () => {
    let st = s.initState(now);
    for (let i = 0; i < s.INTERVALS.length + 1; i += 1) st = s.applyResult(st, true, now);
    expect(s.isGraduated(st)).toBe(true);
    expect(st.nextReviewAt).toBeNull();
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run: `npm test -- src/modules/review/__tests__/review-scheduler.test.js`
Expected: FAIL — `review-scheduler` chưa tồn tại.

- [ ] **Step 3: Implement**

Tạo `review-scheduler.js` theo design §3b:
```javascript
'use strict';
// Leitner box spaced repetition. box = consecutive-correct streak.
// Correct -> box+1, wait INTERVALS[box-1] days. Incorrect -> box 0, due now.
// Graduates once box passes the final interval.
const INTERVALS = [1, 3, 7, 14, 30]; // days
const addDays = (d, n) => new Date(d.getTime() + n * 86400000);

function initState(now = new Date()) {
  return { box: 0, nextReviewAt: now, lastReviewedAt: null };
}
function applyResult(state, correct, now = new Date()) {
  const box = correct ? (state.box || 0) + 1 : 0;
  if (box > INTERVALS.length) return { box, nextReviewAt: null, lastReviewedAt: now };
  const waitDays = box === 0 ? 0 : INTERVALS[box - 1];
  return { box, nextReviewAt: addDays(now, waitDays), lastReviewedAt: now };
}
function isGraduated(state) {
  return (state.box || 0) > INTERVALS.length;
}
module.exports = { initState, applyResult, isGraduated, INTERVALS };
```

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run: `npm test -- src/modules/review/__tests__/review-scheduler.test.js`
Expected: 4 test PASS.

- [ ] **Step 5: Commit**

```bash
git add src/modules/review/review-scheduler.js src/modules/review/__tests__/review-scheduler.test.js
git commit -m "feat(review): add Leitner spaced-repetition scheduler"
```

---

## Task 3: `enqueueMistakes` — nạp câu sai từ learning_events (idempotent)

**Files:**
- Create/Modify: `src/modules/review/review.service.js`
- Test: `src/modules/review/__tests__/review.service.test.js`

- [ ] **Step 1: Viết test thất bại**

```javascript
const mongoose = require('mongoose');
const { connectDb, clearDb, disconnectDb } = require('../../../__tests__/helpers/db');
const { LearningEvent } = require('../../adaptive/learning-event.model');
const { ReviewSchedule } = require('../review-schedule.model');
const review = require('../review.service');

beforeAll(connectDb);
afterEach(clearDb);
afterAll(disconnectDb);

const uid = () => new mongoose.Types.ObjectId();
const wrong = (userId, questionId, skill = 'reading') =>
  LearningEvent.create({ userId, source: 'quiz', refId: uid(), questionId, skill, correct: false });

describe('enqueueMistakes', () => {
  it('creates one active row per wrong quiz item, due now', async () => {
    const userId = uid(); const q1 = uid();
    await wrong(userId, q1);
    const res = await review.enqueueMistakes(userId);
    expect(res.enqueued).toBe(1);
    const rows = await ReviewSchedule.find({ userId });
    expect(rows).toHaveLength(1);
    expect(rows[0].status).toBe('active');
    expect(rows[0].srs.box).toBe(0);
  });

  it('is idempotent — no duplicate row on second call', async () => {
    const userId = uid(); const q1 = uid();
    await wrong(userId, q1);
    await review.enqueueMistakes(userId);
    await review.enqueueMistakes(userId);
    expect(await ReviewSchedule.countDocuments({ userId })).toBe(1);
  });

  it('ignores correct events', async () => {
    const userId = uid();
    await LearningEvent.create({ userId, source: 'quiz', refId: uid(), questionId: uid(), skill: 'reading', correct: true });
    const res = await review.enqueueMistakes(userId);
    expect(res.enqueued).toBe(0);
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t enqueueMistakes`
Expected: FAIL — `review.service` chưa có `enqueueMistakes`.

- [ ] **Step 3: Implement**

Tạo `review.service.js`. `enqueueMistakes(userId)`: `LearningEvent.find({userId, source:'quiz', correct:false})` lấy `questionId/skill/subskill`; gom unique theo `questionId`; mỗi câu `ReviewSchedule.updateOne({userId,questionId}, {$setOnInsert:{skill,subskill,origin:'mistake',status:'active',srs: scheduler.initState()}}, {upsert:true})`; đếm `upsertedCount`. Import `scheduler` từ `./review-scheduler`. Return `{enqueued}`.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t enqueueMistakes`
Expected: 3 test PASS.

- [ ] **Step 5: Commit**

```bash
git add src/modules/review/review.service.js src/modules/review/__tests__/review.service.test.js
git commit -m "feat(review): enqueue wrong quiz items into review schedule"
```

---

## Task 4: `recordReview` — cập nhật SR (Leitner) + graduate

**Files:**
- Modify: `src/modules/review/review.service.js`
- Test: `src/modules/review/__tests__/review.service.test.js` (thêm describe)

- [ ] **Step 1: Viết test thất bại**

```javascript
describe('recordReview', () => {
  const seed = (userId, questionId) => {
    const s = require('../review-scheduler');
    return ReviewSchedule.create({ userId, questionId, skill: 'reading', srs: s.initState(new Date()) });
  };

  it('correct pushes nextReviewAt further out than incorrect', async () => {
    const userId = uid(); const qa = uid(); const qb = uid();
    await seed(userId, qa); await seed(userId, qb);
    const good = await review.recordReview(userId, qa, true);
    const bad = await review.recordReview(userId, qb, false);
    expect(new Date(good.nextReviewAt).getTime()).toBeGreaterThan(new Date(bad.nextReviewAt).getTime());
  });

  it('sets lastResult, keeps active on single correct', async () => {
    const userId = uid(); const q = uid();
    await seed(userId, q);
    const res = await review.recordReview(userId, q, true);
    expect(res.status).toBe('active');
    const row = await ReviewSchedule.findOne({ userId, questionId: q });
    expect(row.lastResult).toBe('correct');
    expect(row.srs.box).toBe(1);
  });

  it('graduates after enough consecutive correct', async () => {
    const s = require('../review-scheduler');
    const userId = uid(); const q = uid();
    await seed(userId, q);
    let res;
    for (let i = 0; i < s.INTERVALS.length + 1; i += 1) res = await review.recordReview(userId, q, true);
    expect(res.status).toBe('graduated');
  });

  it('throws 404 when item not in schedule', async () => {
    await expect(review.recordReview(uid(), uid(), true)).rejects.toThrow();
  });

  it('throws 400 when correct is not boolean', async () => {
    const userId = uid(); const q = uid();
    await seed(userId, q);
    await expect(review.recordReview(userId, q, 'yes')).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t recordReview`
Expected: FAIL — chưa có `recordReview`.

- [ ] **Step 3: Implement**

`recordReview(userId, questionId, correct)`: nếu `typeof correct !== 'boolean'` → throw `ApiError(400,'INVALID_REVIEW_RESULT')`. `findOne({userId,questionId})`; null → `ApiError(404,'REVIEW_ITEM_NOT_FOUND')`. `next = scheduler.applyResult(doc.srs, correct)`; gán `doc.srs=next`, `doc.lastResult = correct?'correct':'incorrect'`, `doc.status = scheduler.isGraduated(next)?'graduated':'active'`; `save()`. Return `{nextReviewAt: next.nextReviewAt, status: doc.status}`. Import `ApiError` theo pattern hiện có (xem `modules/quiz/quiz.service.js`).

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t recordReview`
Expected: 5 test PASS.

- [ ] **Step 5: Commit**

```bash
git add src/modules/review/review.service.js src/modules/review/__tests__/review.service.test.js
git commit -m "feat(review): record review outcome and graduate mastered items"
```

---

## Task 5: `getReviewQueue` — tự nạp + Tier 1 (câu sai đến hạn) → Tier 2 (câu đã làm)

**Files:**
- Modify: `src/modules/review/review.service.js`
- Test: `src/modules/review/__tests__/review.service.test.js` (thêm describe)

- [ ] **Step 1: Viết test thất bại**

```javascript
const { QuizAttempt } = require('../../quiz/quiz-attempt.model');

describe('getReviewQueue', () => {
  const s = require('../review-scheduler');

  it('self-enqueues then returns due items sorted by nextReviewAt (Tier 1)', async () => {
    const userId = uid(); const q1 = uid();
    await wrong(userId, q1);                    // not yet enqueued
    const queue = await review.getReviewQueue(userId, 10);
    const t1 = queue.filter((x) => x.tier === 1).map((x) => String(x.questionId));
    expect(t1).toContain(String(q1));           // enqueue happened inside
  });

  it('orders Tier 1 by most-overdue first', async () => {
    const userId = uid(); const qSoon = uid(); const qOld = uid();
    await ReviewSchedule.create({ userId, questionId: qSoon, skill: 'reading', srs: { ...s.initState(new Date()), nextReviewAt: new Date(Date.now() - 2 * 86400000) } });
    await ReviewSchedule.create({ userId, questionId: qOld, skill: 'reading', srs: { ...s.initState(new Date()), nextReviewAt: new Date(Date.now() - 5 * 86400000) } });
    const queue = await review.getReviewQueue(userId, 10);
    const t1 = queue.filter((x) => x.tier === 1).map((x) => String(x.questionId));
    expect(t1[0]).toBe(String(qOld));
  });

  it('fills remaining slots with previously-attempted non-wrong items (Tier 2)', async () => {
    const userId = uid(); const qWrong = uid(); const qDone = uid();
    await ReviewSchedule.create({ userId, questionId: qWrong, skill: 'reading', srs: { ...s.initState(new Date()), nextReviewAt: new Date(Date.now() - 86400000) } });
    await QuizAttempt.create({
      userId, courseId: uid(), lessonPathKey: 'p1:m1:l1', quizConfigId: uid(),
      score: 1, passed: true, perItem: [{ questionId: qDone, correct: true }],
    });
    const queue = await review.getReviewQueue(userId, 5);
    const t2 = queue.filter((x) => x.tier === 2).map((x) => String(x.questionId));
    expect(t2).toContain(String(qDone));
    expect(t2).not.toContain(String(qWrong));
  });

  it('caps total at limit', async () => {
    const userId = uid();
    for (let i = 0; i < 5; i += 1) {
      await ReviewSchedule.create({ userId, questionId: uid(), skill: 'reading', srs: { ...s.initState(new Date()), nextReviewAt: new Date(Date.now() - 86400000) } });
    }
    const queue = await review.getReviewQueue(userId, 3);
    expect(queue).toHaveLength(3);
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t getReviewQueue`
Expected: FAIL — chưa có `getReviewQueue`.

- [ ] **Step 3: Implement**

`getReviewQueue(userId, limit = 20)` theo design §4C:
1. `await this.enqueueMistakes(userId)` (tự nạp — quyết định ④).
2. Tier 1: `ReviewSchedule.find({userId, status:'active', 'srs.nextReviewAt':{$lte:new Date()}}).sort({'srs.nextReviewAt':1}).limit(limit)` → `{questionId, tier:1}`.
3. Nếu `tier1.length >= limit` → trả `tier1`.
4. Tier 2: `need = limit - tier1.length`; `QuizAttempt.find({userId}).distinct('perItem.questionId')` → loại `questionId` đang ở Tier 1 và câu có row active trong review_schedule → lấy `need` phần tử đầu → `{questionId, tier:2}`.
5. Trả `[...tier1, ...tier2]`.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run: `npm test -- src/modules/review/__tests__/review.service.test.js -t getReviewQueue`
Expected: 4 test PASS.

- [ ] **Step 5: Commit**

```bash
git add src/modules/review/review.service.js src/modules/review/__tests__/review.service.test.js
git commit -m "feat(review): build tiered review queue with lazy mistake enqueue"
```

---

## Task 6: Full suite + regression + ghi chú bàn giao

**Files:** (không sửa code) — verify + tài liệu

- [ ] **Step 1: Chạy toàn bộ test module review**

Run: `npm test -- src/modules/review`
Expected: ALL PASS.

- [ ] **Step 2: Regression — không đụng module khác**

Run: `npm test -- src/modules/quiz src/modules/adaptive src/modules/flashcard`
Expected: ALL PASS.

- [ ] **Step 3: Ghi chú bàn giao cho M4-9 (trong PR description, KHÔNG comment trong code)**

Nêu rõ: M4-9 tiêu thụ `review.getReviewQueue(userId, limit)` tại điểm nối `buildReviewDay` (`lesson-schedule.generator.js`); buổi ôn ghi kết quả qua `review.recordReview(userId, questionId, correct)` (KHÔNG submit quiz thường) để vòng Leitner chạy đúng.

---

## Self-Review

**Acceptance coverage:**
- AC1 (sai→row active due ngay) → Task 3. ⬜
- AC2 (Tier1 trước Tier2, tôn trọng due) → Task 5. ⬜
- AC3 (đúng→giãn, sai→sớm) → Task 2 + Task 4. ⬜
- AC4 (mastered→graduated) → Task 2 + Task 4. ⬜
- AC5 (enqueue idempotent) → Task 3. ⬜
- AC6 (không hồi quy) → Task 6. ⬜

**Placeholder scan:** xác nhận không còn TBD/TODO khi implement xong.

**Type/name consistency:** export `enqueueMistakes/recordReview/getReviewQueue`; scheduler `initState/applyResult/isGraduated/INTERVALS` dùng nhất quán model/service/test.

---

## Ghi chú mở (chuyển tiếp)

- Tier 2 "lấy `need` phần tử đầu" (không random mạnh) — nếu Phase sau cần random thật + đúng "lesson đã completed" + tránh lặp câu tuần trước, refactor `getReviewQueue` (spec §5 / design §2).
- `ApiError` signature: kiểm tra chữ ký thực tế trong `modules/quiz/quiz.service.js`/util lỗi trước khi dùng (mã + code string).
