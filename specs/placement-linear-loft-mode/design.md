---
status: draft
spec: ./spec.md
created: 2026-09-02
---

# Design — Placement LINEAR (LOFT) mode

## 1. Nguyên tắc: chỉ đổi NGUỒN TIER, giữ mọi thứ khác

3 mode dùng chung **bank + scoring (EAP)**; chỉ khác **cách chọn tier của testlet kế tiếp**:

| Mode | Nguồn tier mỗi module | θ dùng để chọn? |
|---|---|---|
| `adaptive` | `route_next_tier(θ)`, hard-route + stop theo SE | Có |
| `fixed` (hiện tại) | `route_next_tier(θ)` (tier=None + rank tier-match) | Có (mềm) |
| **`linear` (LOFT)** | **`skeleton[slotIndex].tier`** (cố định) | **KHÔNG** |

→ LOFT = "fixed mode nhưng ghim tier theo skeleton". **Không entity mới, không bỏ estimate.**

## 2. Skeleton (config trong `blueprint.js`)

Hằng số cạnh `CORE_PLACEMENT_PRESET`:
```js
// Fixed difficulty progression per slot (LOFT). Tier ≈ CEFR under provisional b.
const PLACEMENT_LINEAR_SKELETON = Object.freeze({
  reading:   ['easy', 'mid', 'mid', 'hard'],        // A2, B1, B2, C1  (4 slot → 16 câu)
  listening: ['easy', 'mid', 'mid', 'hard', 'mid'], // A2, B1, B2, C1, B1 (5 slot → 15 câu) [Q1]
});
```
- Độ dài mảng = `fixedModuleCounts[skill]` (reading 4, listening 5) — **phải khớp**, thêm assert.
- Tier lấy từ `stimulus.difficultyTier` (đã authored easy/mid/hard). CEFR pin **thô** qua tier
  (spec §2c) — v1 chấp nhận.
- **KHÔNG** hard-code stimulusId → testlet vẫn **rút live + xoay** (chống lộ).

## 3. Serving flow — nhánh linear

### 3a. Gateway (`attempt.service.js`)
- `nextModuleForAttempt` (~:1045): đã có `mode = assessment.config.assemblyMode`. Khi `mode==='linear'`:
  - slotIndex = số module skill này đã phục vụ (`modulesAdministered.filter(skill).length`).
  - `tier = PLACEMENT_LINEAR_SKELETON[skill][slotIndex]`.
  - gọi `mstFlow.advanceMstModule({ ...args, mode:'linear', forcedTier: tier })` — **không** đọc θ để chọn.
  - Stop khi slotIndex ≥ fixedCount (tái dùng `fixedStopDecision`).
- `submitModuleForAttempt` (~:739): **GIỮ NGUYÊN** `cat.estimate` (θ vẫn cập nhật cho chấm cuối).
  Không thay đổi — linear chỉ khác ở *chọn*, không ở *chấm*.

### 3b. `mst-flow.service.js` `advanceMstModule`
- Thêm param `forcedTier`. Truyền xuống `catAdapter.nextModule({ ..., forcedTier })`.
- Fixed-stop logic dùng chung (mode='linear' → `fixedStopDecision`).

### 3c. `cat.adapter.js` + `cat.py` `/cat/next-module`  [Q2 — quyết định ở đây]
Hiện `next_module`: `mode='fixed'` → `find_stimuli(skill, None, …)`; `mode='adaptive'` →
`find_stimuli(skill, route_next_tier(θ), …)`.
**Chọn (đơn giản, ít đụng nhất):** thêm field `forcedTier: str | None` vào `NextModuleRequest`.
Trong `_find`: **nếu `forcedTier` có** → `find_stimuli(skill, forcedTier, exclude, …)` (bỏ qua cả
theta lẫn mode-routing). Node truyền `forcedTier` khi mode='linear'.
- `select_next_stimulus` giữ nguyên (rank blueprint-coverage + tier-match); với forcedTier, mọi
  candidate đã đúng tier → tier_match hoà → blueprint coverage quyết → vẫn tất định.
- **Rotation/chống lộ**: `recentStimulusIds` (soft-exclude 30 ngày) **đã có** → tự xoay testlet.

## 4. Config (`assessment.service.js:109`)
Hiện: `config.assemblyMode = intake.betaAdaptive === true ? 'adaptive' : 'fixed';`
Đổi (placement default linear, giữ đường bật fixed/adaptive):
```js
if (intake.betaAdaptive === true) config.assemblyMode = 'adaptive';
else if (intake.betaFixedMst === true) config.assemblyMode = 'fixed';  // giữ mode cũ, opt-in
else config.assemblyMode = 'linear';                                   // mặc định placement
```
(Tên cờ `betaFixedMst` tuỳ chốt — miễn giữ được đường vào `fixed`.)

## 5. mode-comparison (`mode-comparison.service.js`)
`const MODES = ['fixed', 'adaptive', 'linear'];` — attempt đã snapshot `assemblyMode` nên
aggregation tự gồm linear. Không đổi logic khác.

## 6. Vì sao 3 mode KHÔNG đụng nhau (bất biến cần giữ)
- Chỉ **1 điểm rẽ**: nguồn tier trong selection (`forcedTier` vs `route_next_tier(θ)`).
- **Scoring dùng chung, không rẽ** → θ mọi mode cùng thang → comparison hợp lệ.
- Mỗi attempt mang **1 mode cố định** (snapshot) → không có state chia sẻ giữa 2 mode trong 1 attempt.

## 7. Rủi ro
- **Tier≈CEFR thô** (mid=B1|B2) → slot "mid" có thể ra B1 hoặc B2. v1 chấp nhận; pin CEFR mịn cần
  thêm filter `cefrBandHint` vào `find_stimuli` (enhancement, ghi Open Question).
- **`forcedTier` cạn pool** (tier đó hết stimulus chưa dùng) → `find_stimuli` fail-soft trả rỗng →
  cần fallback: nới `recentStimulusIds` trước, rồi (nếu vẫn rỗng) nới sang tier lân cận + log.
  Định nghĩa fallback rõ trong task.
- **Không phá fixed/adaptive**: mọi thay đổi additive; test regression 3 mode.
