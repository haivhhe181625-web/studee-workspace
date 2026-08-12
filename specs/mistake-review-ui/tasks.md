# Tasks: Két lỗi sai Phase 2 — REST + UI "Sổ lỗi sai"

> **Cho Developer Agent:** implement theo thứ tự. exe-api (Task 1-2) ship trước; exe-web (Task 3-6) gọi endpoint đã live.
> Mỗi Task độc lập test được, commit riêng. Tách 2 repo → 2 PR (`.ai/project-context.md` §0).

**Design:** `specs/mistake-review-ui/design.md` · **Contract:** `contracts/review-api.md`
**Quyết định đã chốt:** chấm-ở-server · phiên ôn từng-câu phản hồi ngay · sổ có tab đang-ôn/đã-thạo · hoãn testlet (chỉ item `stimulusId=null`) · route `exe-web/src/app/(main)/practice/review`.
**Lệnh (exe-api):** trong `exe-api/services/api`. **Lệnh (exe-web):** trong `exe-web` (xem `TESTING.md`).

---

## ── exe-api (PR #1) ──

## Task 1: Service — list / session / submit-answer (server-graded)

**Files:**
- Modify: `src/modules/review/review.service.js`
- Test: `src/modules/review/__tests__/review-api.service.test.js`

- [ ] **Step 1: Viết test thất bại** (integration, memory-server)

Seed `review_schedule` + `AssessmentQuestion` (mcq có `answerKey`). Test:
- `listReviewItems(userId)` → trả item kèm `stem/skill/status/nextReviewAt`, **không** field `answerKey`.
- `getReviewSessionItems(userId, 10)` → trả câu ẩn đáp án; item có `stimulusId` bị loại.
- `submitReviewAnswer(userId, qId, correctResponse)` → `{correct:true, ...}` và `review_schedule` box tăng.
- `submitReviewAnswer(userId, qId, wrongResponse)` → `{correct:false}`, box về 0.
- gọi answer với `questionId` ngoài két → throw 404.

```bash
npm test -- src/modules/review/__tests__/review-api.service.test.js
```
Expected: FAIL (hàm chưa có).

- [ ] **Step 2: Implement** trong `review.service.js` (thêm, giữ 3 hàm Phase 1):

```javascript
const { AssessmentQuestion } = require('../assessment/assessment-question.model');
const { gradeObjective } = require('../assessment/grade');

async function listReviewItems(userId, { status } = {}) {
  const q = { userId };
  if (status) q.status = status;
  const rows = await ReviewSchedule.find(q).sort({ 'srs.nextReviewAt': 1 }).lean();
  const ids = rows.map((r) => r.questionId);
  const qs = await AssessmentQuestion.find({ _id: { $in: ids } })
    .select('stem skill subskill').lean();
  const byId = new Map(qs.map((x) => [String(x._id), x]));
  return rows.map((r) => {
    const meta = byId.get(String(r.questionId)) || {};
    return {
      questionId: r.questionId, stem: meta.stem || null, skill: r.skill, subskill: r.subskill,
      status: r.status, box: r.srs.box, nextReviewAt: r.srs.nextReviewAt, lastResult: r.lastResult,
    };
  });
}

async function getReviewSessionItems(userId, limit = 20) {
  const queue = await getReviewQueue(userId, limit);
  const ids = queue.map((x) => x.questionId);
  const tierById = new Map(queue.map((x) => [String(x.questionId), x.tier]));
  const rows = await AssessmentQuestion.find({
    _id: { $in: ids }, status: 'active', stimulusId: null, // Phase 2: standalone only
  }).select('stem options itemType skill').lean();
  return rows.map((q) => ({
    questionId: String(q._id), stem: q.stem, options: q.options || [],
    itemType: q.itemType, skill: q.skill, tier: tierById.get(String(q._id)) || 2,
  }));
}

async function submitReviewAnswer(userId, questionId, response) {
  const q = await AssessmentQuestion.findById(questionId)
    .select('itemType stimulusId status +answerKey +acceptedVariants');
  if (!q) throw new ApiError(HTTP.NOT_FOUND, 'Question not found', CODES.REVIEW_ITEM_NOT_FOUND);
  const graded = gradeObjective({
    itemType: q.itemType, answerKey: q.answerKey, acceptedVariants: q.acceptedVariants,
    response, optionOrder: undefined,
  });
  const correct = graded === 1; // recordReview throws 404 if not in this user's schedule
  const out = await recordReview(userId, questionId, correct);
  return { correct, status: out.status, nextReviewAt: out.nextReviewAt };
}
```
Export thêm 3 hàm này.

- [ ] **Step 3: Chạy test — PASS.** `npm test -- src/modules/review/__tests__/review-api.service.test.js`

- [ ] **Step 4: Commit**
```bash
git add src/modules/review/review.service.js src/modules/review/__tests__/review-api.service.test.js
git commit -m "feat(review): list/session/answer service with server-side grading"
```

---

## Task 2: Controller + routes + mount + API test

**Files:**
- Create: `src/modules/review/review.controller.js`, `src/modules/review/review.routes.js`
- Modify: `src/app.js` (mount `/api/review`)
- Test: `src/modules/review/__tests__/review.api.test.js` (supertest)

- [ ] **Step 1: Viết test thất bại** (supertest + `buildApp`, theo `__tests__/quiz-submit.api.test.js`):
  - `GET /api/review/items` (auth) → 200, payload không có `answerKey`.
  - `GET /api/review/session` → 200 items ẩn đáp án.
  - `POST /api/review/answer` với `{questionId, response, correct:true(client nói dối)}` → server **vẫn tự chấm** (correct theo response thật, bỏ qua `correct` client gửi).
  - không token → 401.

```bash
npm test -- src/modules/review/__tests__/review.api.test.js
```
Expected: FAIL.

- [ ] **Step 2: Implement** controller (thin, envelope `{key:...}`, `req.user.id`) + routes (`verifyToken` + `asyncHandler`, theo `quiz.routes.js`):
```
GET  /api/review/items    -> listReviewItems(req.user.id, { status: req.query.status })   -> { items }
GET  /api/review/session  -> getReviewSessionItems(req.user.id, +req.query.limit || 20)    -> { items }
POST /api/review/answer   -> submitReviewAnswer(req.user.id, body.questionId, body.response)-> { result }
```
`app.js`: `const reviewRoutes = require('./modules/review/review.routes'); app.use('/api/review', reviewRoutes);` (theo pin convention của quizRoutes).

- [ ] **Step 3: Chạy test — PASS.**

- [ ] **Step 4: Commit**
```bash
git add src/modules/review/review.controller.js src/modules/review/review.routes.js src/app.js src/modules/review/__tests__/review.api.test.js
git commit -m "feat(review): REST endpoints for mistake bank and review session"
```

- [ ] **Step 5: Regression + PR**
```bash
npm test -- src/modules/review
```
PR #1 (exe-api) base `develop`, nhánh `feat/PRD-XXX-review-rest` (tạo issue Plane con của PRD-142 lấy mã).

---

## ── exe-web (PR #2, sau khi PR#1 merge/deploy) ──

> Theo `exe-web/CLAUDE.md` + `TESTING.md`. Scout `src/services/quiz.service.ts` + `src/hooks/use-lesson-quiz.ts` để khớp api-client/React-Query convention trước khi code.

## Task 3: Service + hooks

**Files:** `src/services/review.service.ts`, `src/hooks/use-review.ts` (+ test cạnh file theo convention repo)

- [ ] **Step 1:** Test (mock api client): `getItems(status?)`, `getSession(limit?)`, `submitAnswer(questionId,response)` gọi đúng path/method (`contracts/review-api.md`).
- [ ] **Step 2:** Implement service (3 hàm map endpoint) + hooks React Query (`useReviewItems`, `useReviewSession`, `useSubmitReviewAnswer` — invalidate items sau submit).
- [ ] **Step 3:** Test PASS.
- [ ] **Step 4:** Commit `feat(review): web service + hooks for mistake bank`.

## Task 4: Trang "Sổ lỗi sai" (list + tab lọc)

**Files:** `src/app/(main)/practice/review/page.tsx`, `src/components/review/review-list.tsx`

- [ ] **Step 1:** Test render list (mock hook): hiển thị stem/skill/badge trạng thái; 2 tab **Đang ôn** (`status=active`) / **Đã thạo** (`status=graduated`).
- [ ] **Step 2:** Implement trang + `ReviewList` (nhóm theo skill, badge trạng thái, `nextReviewAt`), nút **Ôn ngay** → mở phiên ôn.
- [ ] **Step 3:** Test PASS. Commit `feat(review): mistake bank page with status tabs`.

## Task 5: Phiên ôn từng-câu (phản hồi ngay)

**Files:** `src/components/review/review-session-player.tsx`

- [ ] **Step 1:** Test: render 1 câu (tái dùng cấu trúc render item của quiz player); chọn đáp án → gọi `submitAnswer` → hiển thị đúng/sai + `nextReviewAt` → nút "Câu tiếp".
- [ ] **Step 2:** Implement player: lấy `getSession`, lặp từng câu, mỗi câu nộp riêng qua `useSubmitReviewAnswer`, hiện phản hồi tức thì, hết câu → tổng kết + về sổ.
- [ ] **Step 3:** Test PASS. Commit `feat(review): per-question review session with instant feedback`.

## Task 6: Nav entry + full check

- [ ] **Step 1:** Thêm lối vào "Sổ lỗi sai" ở khu Practice (và/hoặc 1 card ở dashboard nếu product muốn) → route `/practice/review`.
- [ ] **Step 2:** `npm run lint` + `npm test` (exe-web) — ALL PASS.
- [ ] **Step 3:** Commit `feat(review): surface mistake bank entry in practice nav`. PR #2 base `develop`.

---

## Self-Review

**Acceptance coverage:**
- AC1 list ẩn đáp án → Task 1/2. ⬜
- AC2 session ẩn đáp án, câu đến hạn → Task 1/2. ⬜
- AC3 nộp→chấm server→SR cập nhật → Task 1/2. ⬜
- AC4 không tin client `correct` → Task 2 (test client-nói-dối). ⬜
- AC5 UI list + phiên ôn → Task 4/5. ⬜
- AC6 chỉ dữ liệu user → Task 2 (verifyToken). ⬜

**Placeholder scan / type-name consistency:** kiểm khi implement.

## Ghi chú mở
- Item `stimulusId≠null` (testlet) bị loại khỏi session → câu đọc/nghe-sai chưa ôn được; báo giới hạn cho học viên hoặc để Phase sau.
- Câu đã `retired` sau khi vào két: `getReviewSessionItems` lọc `status:'active'` nên tự bỏ; cân nhắc dọn/ẩn ở `listReviewItems`.
- Vị trí nav cuối cùng chờ product xác nhận (đề xuất `/practice/review`).
