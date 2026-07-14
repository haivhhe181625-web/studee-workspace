# Tasks: Theo dõi tiến độ kỹ năng theo phase của lộ trình

> **Cho Developer Agent:** implement theo đúng thứ tự Task 1 → 7. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task. **Không** `git add .`, **không** AI attribution trong commit.

**Design:** `specs/skill-progress-tracking/design.md` · **Contract:** `contracts/get-skill-progress.md`

**Lưu ý chung khi chạy lệnh:**
- Task BE (1–4): mọi lệnh chạy trong `<API_REPO>/services/api`. Test: `npx jest src/__tests__/<file>.test.js` (Jest 30 + mongodb-memory-server, helper `src/__tests__/helpers/{db,app}.js`).
- Task FE (5–6): mọi lệnh chạy trong `<WEB_REPO>`. Test: `npx vitest run <path>` (vitest).
- Branch BE: `feature/api-skill-progress` (base `develop`). Branch FE: `feature/web-skill-progress` (base `develop`).

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/src/modules/adaptive/learning-event.model.js` | + `score`, `roadmapId`, `phaseId` + index | Sửa |
| `services/api/src/modules/adaptive/adaptive.service.js` | + `resolveActivePhaseContext`, sửa `ingestIpaAttempt`, + `getSkillProgress` | Sửa |
| `services/api/src/modules/adaptive/adaptive.controller.js` | + `getSkillProgress` | Sửa |
| `services/api/src/modules/adaptive/adaptive.routes.js` | + `GET /progress` | Sửa |
| `services/api/src/__tests__/adaptive.progress.service.test.js` | Unit aggregate + ingest | Tạo |
| `services/api/src/__tests__/adaptive.progress.api.test.js` | API endpoint | Tạo |
| `src/types/adaptive.types.ts` (web) | type `SkillProgress` | Sửa |
| `src/services/adaptive.service.ts` (web) | `getSkillProgress()` | Sửa |
| `src/constants/query-keys.ts` (web) | key `adaptive.progress()` | Sửa |
| `src/hooks/use-adaptive.ts` (web) | `useSkillProgress()` | Sửa |
| `src/components/features/adaptive/SkillProgressCard.tsx` (web) | Card tiến độ | Tạo |
| `src/components/features/adaptive/{index.ts,AdaptiveContainer.tsx}` (web) | export + render section | Sửa |

---

## Task 1: Mở rộng schema `learning_events`

**Files:**
- Modify: `services/api/src/modules/adaptive/learning-event.model.js`
- Test: `services/api/src/__tests__/adaptive.progress.service.test.js`

- [ ] **Step 1: Viết test thất bại**

Tạo `src/__tests__/adaptive.progress.service.test.js`, phần đầu:
```js
const { connectDb, clearDb, disconnectDb } = require('./helpers/db');
const { LearningEvent } = require('../modules/adaptive/learning-event.model');

beforeAll(connectDb); afterEach(clearDb); afterAll(disconnectDb);

describe('LearningEvent — progress fields', () => {
  it('persists score/roadmapId/phaseId', async () => {
    const { Types } = require('mongoose');
    const roadmapId = new Types.ObjectId();
    const ev = await LearningEvent.create({
      userId: new Types.ObjectId(), source: 'speaking_drill', skill: 'speaking',
      subskill: 'pronunciation', correct: true, score: 72, roadmapId, phaseId: 1,
    });
    expect(ev.score).toBe(72);
    expect(ev.phaseId).toBe(1);
    expect(ev.roadmapId.toString()).toBe(roadmapId.toString());
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js
```
Expected: FAIL — `score`/`phaseId` bị Mongoose strip (chưa có trong schema) → `ev.score` là `undefined`.

- [ ] **Step 3: Implement** — thêm vào `LearningEventSchema`:
`score: { type: Number, default: null, min: 0, max: 100 }`, `roadmapId: { type: Schema.Types.ObjectId, ref: 'UserRoadmap', default: null }`, `phaseId: { type: Number, default: null }`. Thêm index: `LearningEventSchema.index({ userId: 1, roadmapId: 1, phaseId: 1, skill: 1 });`. Comment tiếng Anh.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js
```
Expected: 1 test PASS.

- [ ] **Step 5: Commit**
```bash
git add src/modules/adaptive/learning-event.model.js src/__tests__/adaptive.progress.service.test.js
git commit -m "feat(adaptive): add score/roadmapId/phaseId to learning events"
```

---

## Task 2: Ghi điểm thật + ngữ cảnh phase khi luyện IPA (AC-1)

**Files:**
- Modify: `services/api/src/modules/adaptive/adaptive.service.js`
- Test: `services/api/src/__tests__/adaptive.progress.service.test.js`

- [ ] **Step 1: Viết test thất bại** — thêm `describe('ingestIpaAttempt — score + phase')`: mock `UserRoadmap.findOne` trả roadmap có phase active `{ phaseId:1, status:'unlocked' }`, gọi `ingestIpaAttempt({ userId, score: 80, ... })`, assert event mới nhất có `score===80` và `phaseId===1`.

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js -t "ingestIpaAttempt"
```
Expected: FAIL — event ghi ra có `score=null`, `phaseId=null` (hàm cũ chưa set).

- [ ] **Step 3: Implement**
  - Thêm `resolveActivePhaseContext(userId)`: đọc `UserRoadmap.findOne({ userId, isActive:true })`, tái dùng `resolveCurrentPhase(roadmap)` → trả `{ roadmapId, phaseId }`; bọc try/catch trả `{ roadmapId:null, phaseId:null }` khi lỗi (best-effort, KHÔNG throw — không được làm hỏng việc ghi event).
  - Trong `ingestIpaAttempt`: gọi resolver, thêm `score: attempt.score`, `roadmapId`, `phaseId` vào object `LearningEvent.create(...)`. Giữ nguyên `correct`, `computeMastery` phía sau.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js -t "ingestIpaAttempt"
```
Expected: PASS.

- [ ] **Step 5: Commit**
```bash
git add src/modules/adaptive/adaptive.service.js src/__tests__/adaptive.progress.service.test.js
git commit -m "feat(adaptive): record real score and phase context on ipa learning events"
```

---

## Task 3: Service `getSkillProgress` — tổng hợp tiến độ (AC-2, AC-3, AC-E1, AC-E2)

**Files:**
- Modify: `services/api/src/modules/adaptive/adaptive.service.js`
- Test: `services/api/src/__tests__/adaptive.progress.service.test.js`

- [ ] **Step 1: Viết test thất bại** — `describe('getSkillProgress')`, seed in-memory:
  - `UserRoadmap` active 2 phase (phaseId 1 `unlocked`, 2 `locked`, mỗi phase `skills:['speaking']`).
  - `StudentModel` với `skill.speaking.cefr='A2'`.
  - `learning_events`: 3 event phase 1 skill speaking `score=[70,80,null]` `countsTowardMastery=true`; 1 event `countsTowardMastery=false score=10` (gaming); phase 2 không event.
  - Assert: `phases[0].practicedCount===3` (loại gaming), `phases[0].avgScore===75` (avg 70,80 — bỏ null & gaming, AC-E1), `phases[0].skills[0].skill==='speaking'`, `phases[1].practicedCount===0` và `phases[1].avgScore===null` (AC-E2), `skills` global có `{skill:'speaking',cefr:'A2'}`.

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js -t "getSkillProgress"
```
Expected: FAIL — `getSkillProgress is not a function`.

- [ ] **Step 3: Implement** `getSkillProgress(userId)` theo `design.md` §4:
  - `UserRoadmap.findOne({ userId, isActive:true })`; nếu không có → `throw new ApiError(409, 'Needs assessment', 'NEEDS_ASSESSMENT')`.
  - `StudentModel.findOne({ userId })` (lean) → `skills[]` từ `.skill`.
  - `LearningEvent.aggregate([{ $match: { userId, roadmapId: roadmap._id, countsTowardMastery: { $ne: false } } }, { $group: { _id: { phaseId:'$phaseId', skill:'$skill' }, practicedCount: { $sum: 1 }, avgScore: { $avg: '$score' } } }])` — lưu ý `$avg` tự bỏ qua `null` (đúng AC-E1). Làm tròn `avgScore` (Math.round) hoặc `null` khi không có.
  - Ghép `roadmap.phases` (giữ nguyên thứ tự + phase 0 event) với kết quả aggregate → DTO đúng `contracts/get-skill-progress.md`. Gắn `mastery` per-skill từ `student_model` nếu có.
  - Trả object thuần (không leak `_id/__v` nội bộ) — shape theo contract.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx jest src/__tests__/adaptive.progress.service.test.js
```
Expected: ALL PASS (Task 1+2+3 cùng file).

- [ ] **Step 5: Commit**
```bash
git add src/modules/adaptive/adaptive.service.js src/__tests__/adaptive.progress.service.test.js
git commit -m "feat(adaptive): aggregate skill progress per roadmap phase"
```

---

## Task 4: Endpoint `GET /api/adaptive/progress` (AC-4, AC-E3)

**Files:**
- Modify: `adaptive.controller.js`, `adaptive.routes.js`
- Test: `services/api/src/__tests__/adaptive.progress.api.test.js`

- [ ] **Step 1: Viết test thất bại** — dùng helper `app.js` (supertest):
  - GET `/api/adaptive/progress` không token → 401.
  - Với token user chưa có roadmap → 409 code `NEEDS_ASSESSMENT` (AC-E3).
  - Seed roadmap+events cho user A, token user A → 200, `phases` đúng; token user B (không data) → **không** thấy dữ liệu user A (AC-4).

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx jest src/__tests__/adaptive.progress.api.test.js
```
Expected: FAIL — route `/progress` chưa tồn tại → 404.

- [ ] **Step 3: Implement**
  - `adaptive.controller.js`: `exports.getSkillProgress = async (req, res) => { const data = await adaptiveService.getSkillProgress(req.user.id); res.json(data); };` (mỏng, lấy userId từ token — KHÔNG từ request).
  - `adaptive.routes.js`: `router.get('/progress', asyncHandler(controller.getSkillProgress));` (đặt cùng nhóm route đọc; `verifyToken`+`withTenant` đã áp ở `router.use`). Không cần Joi (không input).

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx jest src/__tests__/adaptive.progress.api.test.js
```
Expected: ALL PASS (401 / 409 / 200 / isolation).

- [ ] **Step 5: Commit**
```bash
git add src/modules/adaptive/adaptive.controller.js src/modules/adaptive/adaptive.routes.js src/__tests__/adaptive.progress.api.test.js
git commit -m "feat(adaptive): expose GET /adaptive/progress endpoint"
```

---

## Task 5: FE — type + service + hook (web)

**Files:** (chạy trong `<WEB_REPO>`)
- Modify: `src/types/adaptive.types.ts`, `src/services/adaptive.service.ts`, `src/constants/query-keys.ts`, `src/hooks/use-adaptive.ts`
- Test: `src/services/adaptive.service.test.ts`

- [ ] **Step 1: Viết test thất bại** — thêm case vào `adaptive.service.test.ts`: mock `apiClient.get` trả payload mẫu ở `contracts/get-skill-progress.md`, gọi `getSkillProgress()`, assert gọi đúng `GET /adaptive/progress` và trả đúng data.

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx vitest run src/services/adaptive.service.test.ts
```
Expected: FAIL — `getSkillProgress` chưa export.

- [ ] **Step 3: Implement**
  - `adaptive.types.ts`: thêm `interface SkillProgress { roadmap: {...} | null; skills: {skill:string;cefr:string|null}[]; phases: {...}[]; generatedAt: string }` (đúng contract; `avgScore: number | null`).
  - `adaptive.service.ts`: `export const getSkillProgress = () => apiClient.get<SkillProgress>('/adaptive/progress').then(r => r.data);` (bắt 409 → giống `getStudentModel` trả `{ needsAssessment:true }` nếu pattern hiện tại làm vậy).
  - `query-keys.ts`: thêm `progress: () => [...adaptive.all, 'progress'] as const` trong nhóm `adaptive`.
  - `use-adaptive.ts`: `export const useSkillProgress = () => useQuery({ queryKey: QUERY_KEYS.adaptive.progress(), queryFn: getSkillProgress, enabled: isAuthenticated, staleTime: ... })` (theo đúng pattern hook adaptive hiện có).

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx vitest run src/services/adaptive.service.test.ts
```
Expected: PASS.

- [ ] **Step 5: Commit**
```bash
git add src/types/adaptive.types.ts src/services/adaptive.service.ts src/constants/query-keys.ts src/hooks/use-adaptive.ts src/services/adaptive.service.test.ts
git commit -m "feat(adaptive): add skill progress service and query hook"
```

---

## Task 6: FE — card tiến độ + render trong AdaptiveContainer (AC-2, AC-3, AC-E2, AC-E3)

**Files:** (chạy trong `<WEB_REPO>`)
- Create: `src/components/features/adaptive/SkillProgressCard.tsx`
- Modify: `src/components/features/adaptive/index.ts`, `src/components/features/adaptive/AdaptiveContainer.tsx`, `src/components/features/adaptive/labels.ts` (nếu thiếu nhãn skill)
- Test: `src/components/features/adaptive/SkillProgressCard.test.tsx`

- [ ] **Step 1: Viết test thất bại** — render `<SkillProgressCard data={mock}/>`:
  - data đủ → hiện tên phase + số bài + điểm TB + tên skill (dùng `skillVi` từ `labels.ts`).
  - `avgScore: null` → hiện "Chưa có dữ liệu", **không** hiện "0".
  - `needsAssessment` / không roadmap → hiện gate (EmptyState với CTA làm test), không crash (AC-E3).

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npx vitest run src/components/features/adaptive/SkillProgressCard.test.tsx
```
Expected: FAIL — component chưa tồn tại.

- [ ] **Step 3: Implement**
  - `SkillProgressCard.tsx` (`"use client"` không cần — nhận props): render theo phase, mỗi phase là block; mỗi skill 1 dòng: nhãn `skillVi[skill]`, `practicedCount` bài, thanh điểm TB (tái dùng phong cách bar div+gradient của `StudentModelCard`, hoặc `ui/progress`). `avgScore===null` → text "Chưa có dữ liệu". Dùng token màu studee (`text-studee-*`), `cn()`, không hex/inline style.
  - `index.ts`: export `SkillProgressCard`.
  - `AdaptiveContainer.tsx`: gọi `useSkillProgress()`, render `<SkillProgressCard>` như một `<section>` mới (đặt sau `StudentModelCard`); tái dùng `LoadingState/EmptyState/ErrorState` cho các state (theo pattern container hiện có).
  - Bổ sung `skillVi` trong `labels.ts` nếu thiếu skill nào.

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npx vitest run src/components/features/adaptive/SkillProgressCard.test.tsx
```
Expected: ALL PASS. Sau đó chạy lint: `npx eslint src/components/features/adaptive/SkillProgressCard.tsx` — Expected: no error.

- [ ] **Step 5: Commit**
```bash
git add src/components/features/adaptive/SkillProgressCard.tsx src/components/features/adaptive/index.ts src/components/features/adaptive/AdaptiveContainer.tsx src/components/features/adaptive/labels.ts src/components/features/adaptive/SkillProgressCard.test.tsx
git commit -m "feat(adaptive): render skill progress section in adaptive page"
```

---

## Task 7: Chạy full suite + Self-Review

**Files:** (không sửa code) — verify

- [ ] **Step 1: Chạy toàn bộ test liên quan**

BE (trong `services/api`):
```bash
npx jest src/__tests__/adaptive.progress.service.test.js src/__tests__/adaptive.progress.api.test.js
```
FE (trong `<WEB_REPO>`):
```bash
npx vitest run src/services/adaptive.service.test.ts src/components/features/adaptive/SkillProgressCard.test.tsx
```
Expected: ALL PASS cả hai repo.

- [ ] **Step 2: Smoke test endpoint** (BE chạy local) — dán kết quả `curl` vào PR body:
```bash
curl -X GET http://localhost:5050/api/adaptive/progress -H "Authorization: Bearer <token>"
```
Expected: 200 + JSON đúng `contracts/get-skill-progress.md`, hoặc 409 `NEEDS_ASSESSMENT` nếu account chưa làm test.

- [ ] **Step 3: Đối chiếu acceptance + cập nhật tài liệu** (chỉ trong phạm vi feature) — điền Self-Review dưới.

---

## Self-Review

**Acceptance coverage:**
- AC-1 (ghi hoàn thành + điểm) → Task 1 (schema) + Task 2 (ingest). ☐
- AC-2 (tiến độ theo kỹ năng của phase) → Task 3 + Task 6. ☐
- AC-3 (tiến độ theo từng phase) → Task 3 + Task 6. ☐
- AC-4 (chỉ xem của mình) → Task 4 (isolation test). ☐
- AC-E1 (loại gaming/không điểm khỏi avg) → Task 3. ☐
- AC-E2 (trạng thái rỗng, không hiện 0) → Task 3 (BE null) + Task 6 (FE "Chưa có dữ liệu"). ☐
- AC-E3 (chưa có roadmap active → gate) → Task 4 (409) + Task 6 (FE gate). ☐

**Placeholder scan:** <xác nhận không còn TBD/TODO sau khi implement>

**Type/name consistency:** `score`/`roadmapId`/`phaseId` (BE) khớp `SkillProgress` (FE); tên field DTO khớp `contracts/get-skill-progress.md`.

**Cross-repo:** contract `get-skill-progress.md` là nguồn chung 2 bên — nếu BE đổi shape, cập nhật contract + Task 5 type trước khi FE dùng.
