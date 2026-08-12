# Tasks: M3 Fixed Mode — Bước 1: Nhãn phân loại & độ sẵn sàng ngân hàng

> **Cho Developer Agent:** implement theo thứ tự Task 1 → 6. Mỗi task độc lập test được, commit riêng.
> Không có `design.md` (approach đủ hiển nhiên — xem `spec.md` §8).
> **Lưu ý chung:** mọi lệnh `npm`/`jest` chạy trong `exe-api/services/api`.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/assessment/assessment-stimulus.model.js` | Thêm `passageTheme` + `isAnchor` | Sửa |
| `src/__tests__/assessment-stimulus.test.js` | Test 2 field mới | Sửa |
| `src/admin-api/resources/assessment.admin.js` | Mở 2 field cho nhập liệu | Sửa |
| `src/modules/assessment/assessment.model.js` | Set `fixedModuleCount` thật | Sửa |
| `scripts/report-assessment-bank-coverage.js` | Script inventory read-only (nếu chưa có endpoint) | Tạo |

---

## Task 1: Thêm `passageTheme` + `isAnchor` vào model Stimulus

**Files:**
- Modify: `src/modules/assessment/assessment-stimulus.model.js`
- Test: `src/__tests__/assessment-stimulus.test.js`

- [ ] **Step 1: Viết test thất bại** — thêm case: tạo Stimulus với `passageTheme` hợp lệ (vd `'travel'`) + `isAnchor: true` thì lưu & đọc lại đúng; `passageTheme` ngoài enum bị validate lỗi; mặc định `isAnchor === false`, `passageTheme === null`.

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
npm test -- assessment-stimulus
```
Expected: FAIL — field chưa khai báo nên `passageTheme` bị strict drop / `isAnchor` undefined.

- [ ] **Step 3: Implement** — trong `StimulusSchema` thêm:
  - `passageTheme: { type: String, enum: TOPICS, default: null }` (import `TOPICS` từ `assessment-question.model`; nếu cần giá trị ngoài 8 `TOPICS`, chốt ở OQ trước — không tự thêm).
  - `isAnchor: { type: Boolean, default: false }`.
  - Export thêm nếu cần cho test. Giữ nguyên các field/enum khác (surgical).

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
npm test -- assessment-stimulus
```
Expected: tất cả case (cũ + mới) PASS.

- [ ] **Step 5: Commit**
```bash
git add src/modules/assessment/assessment-stimulus.model.js src/__tests__/assessment-stimulus.test.js
git commit -m "feat(assessment): add passageTheme and isAnchor labels to stimulus"
```

---

## Task 2: Mở `passageTheme` + `isAnchor` cho nhập liệu admin

**Files:**
- Modify: `src/admin-api/resources/assessment.admin.js`
- Test: theo cách test admin-api hiện có (nếu có) hoặc smoke qua factory config

- [ ] **Step 1: Kiểm resource** — xác định resource của Stimulus trong `assessment.admin.js`; đọc `writeFields`/`listFields`/`filters` hiện tại.

- [ ] **Step 2: Implement** — thêm `passageTheme`, `isAnchor` vào `writeFields` (whitelist) và `listFields`; cân nhắc thêm `passageTheme` + `isAnchor` vào `filters` để lọc khi review kho. KHÔNG thêm business logic vào admin-api (chỉ config factory).

- [ ] **Step 3: Verify** — chạy test admin hiện có (nếu có) hoặc smoke tạo/sửa 1 stimulus qua admin route, xác nhận 2 field ghi được và trả về trong list.

Run:
```bash
npm test -- admin
```
Expected: PASS (hoặc smoke thủ công OK nếu không có test admin cho resource này).

- [ ] **Step 4: Commit**
```bash
git add src/admin-api/resources/assessment.admin.js
git commit -m "feat(assessment): expose stimulus passageTheme and isAnchor in admin"
```

> **Lưu ý:** FE nhập liệu nằm ở repo `exe-admin` (PR riêng) — task này chỉ mở field ở admin-api backend.

---

## Task 3: Báo cáo inventory kho (read-only) — ⚠ cần MongoDB dev

**Files:**
- Create (nếu chưa có endpoint tương đương): `scripts/report-assessment-bank-coverage.js`

- [ ] **Step 1: Kiểm tra đã có sẵn chưa** — grep xem `bank-stats.service.js` / admin endpoint đã expose coverage chưa. Nếu có, dùng luôn, bỏ qua tạo script.

- [ ] **Step 2: Implement** — script đọc `assessment_stimuli` + `assessment_questions`, gom rows `{skill, cefrLevel, itemType, status, contentSource, count}`, gọi `summarizeBank` + `coverageGaps` + `tierGaps` (R/L) và so `UOE_TARGET` (UoE). In bảng `(skill × tier)` active + danh sách deficit. Read-only, không ghi DB.

- [ ] **Step 3: Chạy & lưu kết quả**

Run:
```bash
node scripts/report-assessment-bank-coverage.js
```
Expected: in ra số testlet/item active theo `(skill × tier)` cho reading/listening và theo level cho UoE + danh sách cell thiếu. Dán kết quả vào phần dưới (điền thực tế):

```
# Inventory (điền sau khi chạy)
reading: easy=?, mid=?, hard=?
listening: easy=?, mid=?, hard=?
use_of_english: A1=?, A2=?, B1=?, B2=?, C1=?
```

- [ ] **Step 4: Commit** (nếu tạo script)
```bash
git add scripts/report-assessment-bank-coverage.js
git commit -m "chore(assessment): add read-only bank coverage inventory script"
```

---

## Task 4: Chốt ngưỡng + set `fixedModuleCount` — 🚦 GATE người duyệt

**Files:**
- Modify: `src/modules/assessment/assessment.model.js`

- [ ] **Step 1: Trình số** — dựa kết quả Task 3, đề xuất ngưỡng tối thiểu mỗi `(skill × tier)` (đề xuất ≥ 2× testlet phục vụ/đề/tier) + `fixedModuleCount` (Đọc 7·Nghe 8·UoE 10). **Dừng, chờ Tech Lead/Product duyệt** (spec §5).

- [ ] **Step 2: Implement sau khi duyệt** — set default `fixedModuleCount` theo số duyệt (hiện `=1`); nếu ngưỡng coverage cần dùng trong cảnh báo bank-health, ghi hằng số vào chỗ dùng chung (vd cạnh `UOE_TARGET`) — không rải rác.

- [ ] **Step 3: Verify** — test config: tạo assessment fixed mode, xác nhận `config.fixedModuleCount` đúng số mới cho từng kỹ năng.

Run:
```bash
npm test -- assessment
```
Expected: PASS; không test nào còn giả định `fixedModuleCount === 1`.

- [ ] **Step 4: Commit**
```bash
git add src/modules/assessment/assessment.model.js
git commit -m "feat(assessment): set real fixedModuleCount for core placement"
```

---

## Task 5: Lấp coverage thiếu (AI-gen + expert review) — ⚠ cần DB + pipeline

**Files:** (không sửa code lõi) — dữ liệu + duyệt

- [ ] **Step 1: Sinh testlet theo deficit** — dùng `recommendGeneration`/`recommendTierGen` (từ Task 3) làm kế hoạch, chạy `gen-testlet.service.js` cho từng `(skill × tier)` thiếu (R/L) và UoE tới `UOE_TARGET`.

- [ ] **Step 2: Duyệt sang active** — qua `expert-review.service.js`; gắn `passageTheme` (+ `isAnchor` cho câu neo) khi duyệt. Chỉ item `contentSource` hợp lệ mới lên `active`.

- [ ] **Step 3: Verify — chạy lại inventory**

Run:
```bash
node scripts/report-assessment-bank-coverage.js
```
Expected: `tierGaps`(R/L) rỗng và mỗi level UoE ≥ `UOE_TARGET`; không cell rỗng.

- [ ] **Step 4: Commit** (nếu có thay đổi seed/script; nội dung sinh ra nằm ở DB, không commit)

---

## Task 6: Full suite + dọn dẹp

**Files:** (không sửa code) — verify

- [ ] **Step 1: Chạy toàn bộ test liên quan**

Run:
```bash
npm test -- assessment
```
Expected: ALL PASS.

- [ ] **Step 2: Cập nhật tài liệu** — điền số inventory + ngưỡng đã chốt vào `spec.md` §5/§7 (đóng OQ-3). Cập nhật `docs` M3 nếu có mục coverage.

---

## Self-Review

**Acceptance coverage:**
- AC-1 (schema 2 field) → Task 1.
- AC-2 (admin nhập được) → Task 2.
- AC-3 (báo cáo inventory) → Task 3.
- AC-4 (chốt ngưỡng + fixedModuleCount) → Task 4.
- AC-5 (đạt ngưỡng sau lấp) → Task 5.
- AC-6 (chỉ active + source hợp lệ) → Task 3/5 (giữ guard `isPoolEligible`).
- AC-7 (không đụng chấm/lắp đề/cat) → toàn bộ task giữ surgical.

**Placeholder scan:** phần Inventory ở Task 3 là chỗ điền số thật — phải điền trước khi đóng OQ-3.

**Type/name consistency:** `TOPICS` import từ `assessment-question.model`; `passageTheme`/`isAnchor` đặt tên nhất quán giữa model ↔ admin resource ↔ script.
