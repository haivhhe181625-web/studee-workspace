---
status: draft
spec: ./spec.md
design: ./design.md
created: 2026-09-02
---

# Tasks — Placement LINEAR (LOFT) mode

Thứ tự: enum+skeleton → cat linear tier → gateway serve → config default → comparison → e2e proof.
Mọi thay đổi **additive**; không phá `fixed`/`adaptive`. Repo: exe-api (Node + Python).
**Không** đụng scoring/bank. **Không** entity DB mới.

---

## T1 — [Node] enum `linear` + skeleton config
- [ ] `assessment.model.js`: `ASSEMBLY_MODES` += `'linear'`. `assessment-attempt.model.js`:
  `assemblyMode` enum += `'linear'`.
- [ ] `blueprint.js`: thêm `PLACEMENT_LINEAR_SKELETON` (reading 4 / listening 5, tier per slot per
  design §2) + export. Assert độ dài khớp `CORE_PLACEMENT_PRESET.fixedModuleCounts`.
- **Verify:** unit test — enum chấp nhận 'linear'; `PLACEMENT_LINEAR_SKELETON.reading.length===4`,
  `.listening.length===5`; mỗi phần tử ∈ {easy,mid,hard}. `npx jest blueprint`.

## T2 — [Python] cat `forcedTier` cho next-module
- [ ] `cat.py` `NextModuleRequest`: thêm `forcedTier: str | None = None`.
- [ ] `_find`: nếu `body.forcedTier` → `mongo.find_stimuli(skill, body.forcedTier, exclude, target_goal, center_id)`
  (bỏ qua theta + mode routing). Giữ nhánh fixed/adaptive cũ nguyên.
- [ ] Fallback khi rỗng (design §7): thử used+recent → used-only → (vẫn rỗng) tier lân cận + log; không crash.
- **Verify:** pytest — `forcedTier='easy'` chỉ trả stimulus tier easy; rỗng → fallback không lỗi.
  `pytest services/cat/tests -k next_module`.

## T3 — [Node] adapter + mst-flow truyền forcedTier
- [ ] `cat.adapter.js` `nextModule`: thêm `forcedTier` vào payload khi có.
- [ ] `mst-flow.service.js` `advanceMstModule`: thêm param `forcedTier`, truyền xuống `catAdapter.nextModule`.
  Với mode='linear' dùng `fixedStopDecision` (đã có).
- **Verify:** unit test mst-flow — mode='linear'+forcedTier → gọi catAdapter.nextModule với đúng tier,
  KHÔNG gọi route_next_tier. `npx jest mst-flow`.

## T4 — [Node] gateway serve linear (không dùng θ để chọn)
- [ ] `attempt.service.js` `nextModuleForAttempt`: khi `mode==='linear'` → tính slotIndex =
  số module skill đã phục vụ → `tier = PLACEMENT_LINEAR_SKELETON[skill][slotIndex]` →
  `advanceMstModule({..., mode:'linear', forcedTier: tier})`. Stop khi slotIndex ≥ fixedCount.
- [ ] `submitModuleForAttempt`: **KHÔNG đổi** (giữ cat.estimate — θ vẫn cập nhật cho chấm cuối).
- **Verify:** integration — attempt linear phục vụ đúng 4 reading + 5 listening theo tier skeleton;
  tier module N == skeleton[N] bất kể đáp án. `npx jest attempt`.

## T5 — [Node] config default placement = linear (giữ đường fixed/adaptive)
- [ ] `assessment.service.js:109`: 3 nhánh (design §4) — betaAdaptive→adaptive, betaFixedMst→fixed,
  else→linear. Chỉ áp cho placement (goal placement); không phá luyện đề.
- **Verify:** unit — intake placement mặc định → config.assemblyMode='linear'; cờ betaFixedMst→'fixed';
  betaAdaptive→'adaptive'. `npx jest assessment.service`.

## T6 — [Node] mode-comparison gồm linear
- [ ] `mode-comparison.service.js`: `MODES = ['fixed','adaptive','linear']`.
- **Verify:** service trả nhóm 'linear' khi có attempt linear. `npx jest mode-comparison`.

## T7 — [Proof] e2e: linear KHÔNG adaptive theo đáp án (bằng chứng sửa drift)
- [ ] Test: 2 attempt linear **cùng skeleton**, mô phỏng **trả lời đúng-hết vs sai-hết** →
  **cùng chuỗi tier** (skeleton) → khác hẳn fixed (fixed rẽ theo θ). Khẳng định LOFT thật.
- [ ] Regression: chạy lại flow fixed + adaptive → không đổi hành vi.
- **Verify:** `npx jest` (assessment suite) xanh; test T7 chứng minh tier độc lập đáp án.

---

## Định nghĩa done
- [ ] T1–T7 verify xanh; 3 mode chạy độc lập; scoring/bank không đổi.
- [ ] Diff + acceptance trình user duyệt trước merge (gate người).

## Đã chốt (2026-09-02)
- Q1 listening slot 5 = **mid (B1)** → skeleton listening = ['easy','mid','mid','hard','mid'].
- Q2 cat = **forcedTier** (design §3c). · Q3 admin expose mode = **v1 không**.
- Design signed off → code.
