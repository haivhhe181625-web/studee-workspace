---
status: draft
owner: BA
created: 2026-09-02
affects: [exe-api]   # services/api (Node) + services/cat (Python)
source_framework: specs/placement-item-bank/spec.md §2b, design.md §2
---

# Spec — Placement LINEAR (LOFT) delivery mode

## 1. Mục tiêu & why (sửa drift khỏi khung §2b)

Tài liệu **khung** placement mô tả bài đầu vào là **fixed-mode, ráp nhiều form từ phân bổ
competency×CEFR cứng, xoay testlet để chống lộ** (`placement-item-bank/spec.md §2b:13,33,64`;
`design.md §2:29`). §2b:33 ghi rõ **"Adaptive mode → chỉ fixed"**. Đây là **LOFT**
(Linear-on-the-Fly): khung cứng + rút testlet tương đương xoay vòng.

**Code hiện tại** (`assemblyMode='fixed'`) lại **fixed-length + adaptive tier-SELECTION theo θ**
(testlet sau phụ thuộc đáp án trước) — vì serving tái dùng engine MST (`mst-flow` +
`cat.py next-module` route tier theo `route_next_tier(θ)`). Phần **adaptive-select này KHÔNG có
trong khung placement** → là **drift** vô tình do tái dùng MST.

**Mục tiêu:** thêm mode **`linear` (LOFT)** đúng khung, đặt làm **mặc định placement**; **GIỮ
NGUYÊN** `fixed` và `adaptive` làm mode **config-toggle** (ngủ) để sau chỉ việc bật là dùng.

**Vì sao LOFT phù hợp lúc này:** cùng cấu trúc/độ khó cho mọi người → công bằng, chuẩn hoá, và
**thu response nhanh để calibrate irt.b** (chặn P0 của review). Adaptive để dành sau khi có b thật.

## 2. Phạm vi

### 2a. Trong scope (exe-api)
- **Thêm `'linear'`** vào `assemblyMode` enum (`assessment.model.js`, `assessment-attempt.model.js`)
  — additive, **không đổi nghĩa/không xoá** `'fixed'`, `'adaptive'` (đã nằm trong data + mode-comparison).
- **Skeleton cứng** (config, KHÔNG entity DB mới): danh sách tier/CEFR mỗi slot theo §2b —
  Reading 4 slot + Listening 5 slot.
- **Nhánh serving `linear`**: chọn testlet mỗi slot bằng **tier từ skeleton** (không `route_next_tier(θ)`),
  tái dùng `find_stimuli(skill, tier, exclude)` + `recentStimulusIds` (xoay chống lộ có sẵn).
- **Config default**: placement → `'linear'` (`assessment.service.js:109`), giữ đường bật fixed/adaptive.
- **mode-comparison**: `MODES += 'linear'` để dashboard so 3 mode.

### 2b. Giữ nguyên (KHÔNG đụng — điểm khiến 3 mode không ảnh hưởng nhau)
- **Scoring**: EAP → θ_R, θ_L → CEFR **không đổi**. `cat.estimate` sau mỗi module **giữ nguyên**
  (θ vẫn tính, chỉ **không dùng cho chọn testlet** ở linear) → output chấm **y hệt** fixed.
- **Bank / blueprint / model θ**: không đổi.
- **Engine `fixed` + `adaptive`**: giữ nguyên, chỉ thêm nhánh mới song song.

### 2c. Out of scope (lý do)
- Calibrate `irt.b`, CEFR standard-setting, đổi thang chấm — **chờ pilot** (đây chính là việc
  LOFT phục vụ, không phải việc LOFT làm).
- Full adaptive tuning, confidence UI, enrich bank — task khác.
- **Pin CEFR chính xác từng slot**: selection lọc theo `difficultyTier` (easy/mid/hard), không phải
  CEFR trực tiếp → dưới b-provisional, tier≈CEFR **thô** (mid = B1 hoặc B2). Chấp nhận cho v1
  (đủ tạo dải khó cố định); pin CEFR mịn = enhancement sau.
- Ráp form tay / entity `PlacementForm`: **không cần** — skeleton config + rotation qua `recentStimulusIds`.

## 3. Acceptance Criteria (checkable)
- [ ] `assemblyMode` nhận `'linear'`; attempt placement mới có `config.assemblyMode='linear'` +
  snapshot trên attempt.
- [ ] Ở mode linear: testlet mỗi slot **KHÔNG phụ thuộc θ** — chứng minh bằng test: 2 attempt cùng
  skeleton, **trả lời khác nhau (đúng hết vs sai hết)** → **cùng chuỗi tier** (easy→…→hard theo
  skeleton), khác fixed (fixed sẽ rẽ tier theo θ).
- [ ] Skeleton đúng §2b: Reading 4 slot phủ dải A2→C1, Listening 5 slot phủ dải; đủ 31 câu (R16/L15).
- [ ] Rotation chống lộ: attempt lặp lại (cùng user) **không nhận lại đúng testlet** khi bank đủ
  (dùng `recentStimulusIds`), degrade về lặp khi bank cạn (không kẹt).
- [ ] Scoring linear = fixed về công thức: cùng bộ đáp án → cùng θ_R/θ_L/CEFR (EAP không đổi).
- [ ] `fixed` + `adaptive` vẫn chạy như cũ (regression) — test 3 mode độc lập.
- [ ] `mode-comparison` trả cả 3 mode.

## 4. Affected surfaces
- `exe-api/services/api/src/modules/assessment/`: `assessment.model.js`, `assessment-attempt.model.js`,
  `assessment.service.js` (default), `attempt.service.js` (nhánh serve), `mst-flow.service.js`
  (nhánh linear), `blueprint.js` (skeleton config), `mode-comparison.service.js` (MODES).
- `exe-api/services/cat/src/routes/cat.py`: `next-module` — nhánh/`forcedTier` cho linear (xem design.md).

## 5. Open Questions (chốt khi design/review)
- Q1: Skeleton listening slot thứ 5 (từ 14→15 detail) đặt CEFR/tier nào? → đề xuất **B1 (mid)**.
- Q2: cat.py — thêm `mode='linear'`+`forcedTier`, hay gateway truyền tier qua field sẵn? → xem design.md.
- Q3: Có expose chọn mode ở admin (tạo assessment) không, hay chỉ config nội bộ? → v1 **config nội bộ**.
