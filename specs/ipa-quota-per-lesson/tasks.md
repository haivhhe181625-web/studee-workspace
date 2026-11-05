# Tasks: IPA quota tính theo bài + cap 2 lần chấm mỗi mục

> **Cho Developer Agent:** implement theo thứ tự Task 1 → N. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task.

**Design:** `specs/ipa-quota-per-lesson/design.md`
**Lưu ý chạy lệnh:** Task 1–6 (BE) chạy trong `exe-api/services/api`. Task 7–8 (FE) chạy trong `exe-web`, **PR riêng ở repo exe-web** (xem design §11). Branch BE: `feat/ipa-quota-per-lesson` (base `develop`).

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/quota/quota.config.js` | `TIERS`: thêm `ipaLessons` (1 cho cả 2 tier), gỡ `pronunciation` | Sửa |
| `src/modules/user/user.model.js` | Thêm `quotaIpaLessons: [ObjectId]` | Sửa |
| `src/modules/ipa/ipa-quota.js` | Helper 2 tầng | Tạo |
| `src/modules/ipa/__tests__/ipa-quota.test.js` | Unit test helper | Tạo |
| `src/modules/ipa/ipa.service.js` | Wire helper vào các path chấm | Sửa |
| `src/modules/ipa/ipa.controller.js` | Đưa khối `quota` ra DTO (nếu cần) | Sửa |
| `src/__tests__/quota.service.test.js` · `quota.trial-mode.test.js` | Gỡ assert `pronunciation` | Sửa |
| `exe-web/src/types/ipa.types.ts` | Field `quota` + `limitKind` | Sửa |
| `exe-web/src/components/features/ipa/steps/SpeakingLoopStep.tsx` | Khoá mic khi đủ 2 lần + xử lý 403 | Sửa |
| `exe-web/src/components/features/quota/QuotaExceededModal.tsx` | Phân biệt lesson/item | Sửa |

---

## Task 1: Config — thêm `ipaLessons` (giữ `pronunciation`)

**Files:** Modify `src/modules/quota/quota.config.js` · Test `src/__tests__/quota.service.test.js`

> **Thứ tự an toàn:** GIỮ `pronunciation` trong `TIERS` ở task này (ipa.service còn dùng tới Task 4). Chỉ THÊM `ipaLessons`. Gỡ `pronunciation` + dọn test của nó nằm ở **Task 5** (sau khi wire helper xong) — để suite luôn xanh giữa các task.

- [ ] **Step 1: Sửa test** — trong `describe('limitFor')`, **thêm** (giữ nguyên các assert `pronunciation` hiện có):
```js
expect(limitFor({ isAnonymous: true }, 'ipaLessons')).toBe(1);
expect(limitFor({ isAnonymous: false }, 'ipaLessons')).toBe(1);
```

- [ ] **Step 2: Chạy — FAIL**
Run: `npx jest src/__tests__/quota.service.test.js -t limitFor`
Expected: FAIL — `ipaLessons` chưa có trong TIERS (`limitFor` throw `Unknown quota feature`).

- [ ] **Step 3: Implement** — trong `quota.config.js` `TIERS`: **thêm** `ipaLessons: 1` vào cả `guest` và `free`. Giữ nguyên `pronunciation`, `flashcardGeneration`, `bonus`, các feature khác.

- [ ] **Step 4: Chạy — PASS**
Run: `npx jest src/__tests__/quota.service.test.js src/__tests__/quota.trial-mode.test.js`
Expected: ALL PASS (không đụng các test `pronunciation` sẵn có).

- [ ] **Step 5: Commit**
```bash
git add src/modules/quota/quota.config.js src/__tests__/quota.service.test.js
git commit -m "feat(quota): add ipaLessons tier for per-lesson ipa quota"
```

---

## Task 2: Field `quotaIpaLessons` trên User

**Files:** Modify `src/modules/user/user.model.js` · Test `src/modules/user/__tests__/*` (hoặc thêm assert vào test model sẵn có)

- [ ] **Step 1: Viết test** — user mới có `quotaIpaLessons` mặc định `[]`:
```js
it('defaults quotaIpaLessons to empty array', () => {
  const u = new User({ email: 'a@b.co' });
  expect(Array.isArray(u.quotaIpaLessons)).toBe(true);
  expect(u.quotaIpaLessons).toHaveLength(0);
});
```

- [ ] **Step 2: Chạy — FAIL** (field chưa tồn tại → undefined).
- [ ] **Step 3: Implement** — thêm field cạnh khối Quota:
```js
// Distinct IPA lessons that have consumed a trial lesson-unit (per-lesson quota).
// length = số bài đã dùng; capped by TIERS[tier].ipaLessons. Unlimited users bypass.
quotaIpaLessons: { type: [{ type: Schema.Types.ObjectId, ref: 'IpaLesson' }], default: [] },
```
Thêm comment `@deprecated` lên `quotaUsed.pronunciation` (ngừng dùng, không migrate).

- [ ] **Step 4: Chạy — PASS.**
- [ ] **Step 5: Commit**
```bash
git add src/modules/user/user.model.js src/modules/user/__tests__/
git commit -m "feat(user): add quotaIpaLessons array for per-lesson ipa quota"
```

---

## Task 3: Helper `ipa-quota.js` (2 tầng)

**Files:** Create `src/modules/ipa/ipa-quota.js` · Test `src/modules/ipa/__tests__/ipa-quota.test.js`

API:
- `consumeIpaScoring(user, lessonId, itemId, { priorAttemptCount })` → `{ justAddedLesson, lessonsRemaining, itemAttemptsRemaining }` hoặc throw.
- `refundLesson(user, lessonId, { justAddedLesson })` → best-effort, never throws.
- `peekIpaQuota(user, lessonId, priorAttemptCount)` → `{ lessonsRemaining, itemAttemptsRemaining }` (read-only, cho DTO).

Luật (design §4): 0) unlimited bypass; 1) `priorAttemptCount >= 2` → 403 `meta.limitKind='item'`; 2) lessonId đã trong `quotaIpaLessons` → allow (justAdded=false); else atomic `$size < limit` + `$addToSet`, null → 402 `meta.limitKind='lesson'` + `upgradeAction`.

- [ ] **Step 1: Viết test** `ipa-quota.test.js` (mock `User`):
```
- free user, bài mới, cap chưa đầy → allow, justAddedLesson=true, lessonsRemaining giảm
- bài đã trong quotaIpaLessons → allow, justAddedLesson=false, KHÔNG gọi findOneAndUpdate
- free đã đủ 1 bài, bài thứ 2 → throw 402 meta.limitKind='lesson'
- guest đủ 1 bài, bài thứ 2 → throw 402
- priorAttemptCount>=2 → throw 403 meta.limitKind='item' (chặn TRƯỚC khi đụng tầng bài)
- quotaUnlimited=true → allow mọi thứ, không DB write
- TRIAL_LIMIT_MODE=unlimited → như trên
- refundLesson(justAddedLesson=true) → $pull lessonId; (false) → không $pull
- concurrency: findOneAndUpdate atomic guard mô phỏng → chỉ 1 trong 2 bài mới vượt cap được nhận
```

- [ ] **Step 2: Chạy — FAIL** (file chưa tồn tại).
- [ ] **Step 3: Implement** `ipa-quota.js` theo design §4. Dùng `limitFor(user,'ipaLessons')`, `env.TRIAL_LIMIT_MODE`, `ApiError` + `CODES.QUOTA_EXCEEDED`, `HTTP.PAYMENT_REQUIRED`/`FORBIDDEN`. Atomic:
```js
const updated = await User.findOneAndUpdate(
  { _id: user._id, quotaIpaLessons: { $ne: lessonId },
    $expr: { $lt: [{ $size: '$quotaIpaLessons' }, limit] } },
  { $addToSet: { quotaIpaLessons: lessonId } },
  { new: true, select: 'quotaIpaLessons' });
```

- [ ] **Step 4: Chạy — PASS**
Run: `npx jest src/modules/ipa/__tests__/ipa-quota.test.js`
Expected: ALL PASS.

- [ ] **Step 5: Commit**
```bash
git add src/modules/ipa/ipa-quota.js src/modules/ipa/__tests__/ipa-quota.test.js
git commit -m "feat(ipa): add two-tier per-lesson + per-item scoring quota helper"
```

---

## Task 4: Wire helper vào mọi path chấm

**Files:** Modify `src/modules/ipa/ipa.service.js` (+ `ipa.controller.js` nếu DTO cần) · Test `src/modules/ipa/__tests__/*` liên quan

- [ ] **Step 1: Audit** — xác nhận đủ 3 entrypoint chấm audio: `submitItem`, `submitMiniTest`, và path `submitAttempt` (route `/attempts`, kind word/sentence/dialogue). Ghi lại chỗ nào đang gọi `withQuota('pronunciation')`/`consumeQuota('pronunciation')`.

- [ ] **Step 2: Viết/điều chỉnh test integration** — mỗi path:
  - chấm được khi trong hạn (trả `quota.lessonsRemaining`/`itemAttemptsRemaining`)
  - path chấm mục lần 3 → 403
  - path đụng bài quá hạn → 402
  - cảm âm/TAP/objective KHÔNG trừ bài, KHÔNG bị cap.
Run trước: `npx jest src/modules/ipa` → FAIL ở các case mới.

- [ ] **Step 3: Implement** — thay `withQuota(user,'pronunciation',idemKey,fn)` bằng:
```js
const consumed = await ipaQuota.consumeIpaScoring(user, lessonId, itemId, { priorAttemptCount: prior.length });
let result;
try { result = await llmProxy.analyzeAudio(...); }
catch (e) { await ipaQuota.refundLesson(user, lessonId, consumed); throw e; }
// ... recordIpaAttempt (giữ nguyên) → tự tăng attempt count cho lần sau
```
`submitMiniTest`: gọi `consumeIpaScoring` cho **từng câu mic** với `itemId` câu đó (đếm `IpaAttempt` per-question); lượt bài chỉ tính 1 lần (câu đầu justAdded). `submitAttempt`: gọi cùng helper. DTO thêm `quota: { lessonsRemaining, itemAttemptsRemaining }` (dùng `peekIpaQuota` hoặc giá trị trả từ consume).

- [ ] **Step 4: Chạy — PASS**
Run: `npx jest src/modules/ipa`
Expected: ALL PASS.

- [ ] **Step 5: Commit**
```bash
git add src/modules/ipa/ipa.service.js src/modules/ipa/ipa.controller.js src/modules/ipa/__tests__/
git commit -m "feat(ipa): gate all audio-scoring paths through per-lesson quota helper"
```

---

## Task 5: Đánh dấu `pronunciation` deprecated + full BE suite

**Files:** Modify `src/modules/quota/quota.config.js` · (verify) `src/modules/ipa`, `src/__tests__`

> **Điều chỉnh so với plan gốc:** GIỮ `pronunciation` trong `TIERS` làm config chết (deprecated) thay vì gỡ. Lý do: sau Task 4 không prod code nào gọi `limitFor('pronunciation')` nữa (vô hại), còn việc gỡ sẽ kéo theo viết lại ~16 test quota không liên quan + bất nhất `KNOWN_QUOTA_FEATURES` ở entitlement (chưa wire). KISS/surgical → giữ, note deprecated.

- [ ] **Step 0: Deprecated note** — thêm comment `@deprecated` lên `TIERS.*.pronunciation` (config) và `quotaUsed.pronunciation` (user.model). Không đụng các test `pronunciation` sẵn có.

- [ ] **Step 1: Grep sót**
Run: `rg "'pronunciation'|\"pronunciation\"" src/modules/ipa src/__tests__ | rg -v "scores\\.pronunciation|scores: \\{"`
Expected: không còn tham chiếu quota `pronunciation` (chỉ còn `scores.pronunciation` là điểm số — hợp lệ).

- [ ] **Step 2: Full suite liên quan**
Run: `npx jest src/modules/ipa src/__tests__/quota.service.test.js src/__tests__/quota.trial-mode.test.js src/__tests__/auth.phase3-5.test.js`
Expected: ALL PASS.

- [ ] **Step 3: Commit** (nếu có dọn dẹp)
```bash
git commit -am "chore(ipa): remove leftover pronunciation quota references"
```

---

## Task 6: Mở PR BE

- [ ] Push `feat/ipa-quota-per-lesson` → mở PR base `develop`, mô tả: mô hình 2 tầng, thay PR #69 (đã merge) bằng đếm-theo-bài. Chờ CODEOWNER (`@dotuananhtb`).

---

## Task 7: FE — khoá mic khi mục đủ 2 lần (repo `exe-web`, PR riêng)

**Files:** `src/types/ipa.types.ts`, `src/components/features/ipa/steps/SpeakingLoopStep.tsx`, `src/hooks/use-ipa.ts`

- [ ] **Step 1:** types — thêm `quota?: { lessonsRemaining: number; itemAttemptsRemaining: number }` vào `IpaSubmitResult`; `QuotaExceededError` mang `limitKind: 'lesson'|'item'`.
- [ ] **Step 2:** `SpeakingLoopStep` — đọc `itemAttemptsRemaining` (hoặc đếm `results`/attempt) để **disable nút mic + ẩn "Ghi âm lại"** khi mục đã đủ 2 lần; hiện nhãn "Đã hết lượt nói cho mục này". `handleRetry` chỉ cho khi còn lượt.
- [ ] **Step 3:** Xử lý lỗi 403 `limitKind:'item'` → hiện thông báo tại mục (không mở modal upgrade); 402 `limitKind:'lesson'` → mở `QuotaExceededModal`.
- [ ] **Step 4:** Chạy test FE liên quan: `npx vitest run src/components/features/ipa` → PASS.
- [ ] **Step 5:** Commit `feat(ipa): cap mic scoring at 2 per item and surface per-lesson quota`.

---

## Task 8: FE — `QuotaExceededModal` phân biệt lesson/item

**Files:** `src/components/features/quota/QuotaExceededModal.tsx`

- [ ] **Step 1:** Thêm prop `limitKind`; nội dung: `lesson` → "Bạn đã dùng hết số bài luyện phát âm miễn phí — đăng ký/nâng cấp để học tiếp"; `item` → "Bạn đã luyện đủ mục này, hãy sang mục/bài khác".
- [ ] **Step 2:** Test render 2 nhánh; `npx vitest run src/components/features/quota` → PASS.
- [ ] **Step 3:** Commit `feat(quota): distinguish lesson-limit vs item-limit in exceeded modal`. Mở PR base `develop` ở repo exe-web.

---

## Task N: Full suite + đối chiếu acceptance

- [ ] **Step 1: BE**
Run: `npx jest src/modules/ipa src/__tests__/quota.service.test.js src/__tests__/quota.trial-mode.test.js`
Expected: ALL PASS.
- [ ] **Step 2:** FE build/test xanh ở repo exe-web.
- [ ] **Step 3:** Cập nhật docs nếu cần (chỉ phạm vi feature).

---

## Trạng thái

- **BE (Task 1–5): HOÀN THÀNH** — nhánh `feat/ipa-quota-per-lesson` (exe-api), 7 commit. Mocked suites 106/106 pass. Code-review đã chạy → 1 lỗi CRITICAL (`$size` field thiếu cho user cũ) đã fix bằng `$ifNull`; mini-test item-cap sửa thành skip-not-abort; doc/param dọn.
- **FE (Task 7–8): HOÀN THÀNH** — exe-web PR #171 (nhánh `feat/ipa-quota-per-lesson`). Vitest 106/106 pass, ESLint sạch. Cần merge BE (#72) trước để có `limitKind`/403.
- **Chưa verify:** integration test cần MongoDB (sandbox không có) — chạy trên máy có DB trước khi merge.

## Self-Review

**Acceptance coverage:**
- AC1–3 (cap bài) → Task 3 + 4. ✅ (review xác nhận)
- AC4–5 (cap 2 lần/mục, gồm mini-test) → Task 3 + 4. ✅
- AC6 (cảm âm/TAP miễn phí) → Task 4. ✅
- AC7 (unlimited bypass) → Task 3. ✅
- AC8 (SRS dùng chung 2 lượt) → Task 3. ✅
- AC9 (bỏ bonus IPA) → Task 1 + 4. ✅
- AC10 (FE phân biệt modal) → Task 7 + 8. ✅ (exe-web #171)

**Review findings còn mở (non-blocking):** #3 `remaining` đổi nghĩa (cần đồng bộ khi làm FE), #4 mất idempotency retry per-item (chấp nhận — cap 2, retry hiếm), #5 `peekIpaQuota` chưa wire (public API để dành FE/endpoint peek).
**Type/name consistency:** `consumeIpaScoring`/`refundLesson`/`peekIpaQuota`/`quotaIpaLessons`/`ipaLessons`/`limitKind` nhất quán BE↔FE.
