# Design: IPA quota tính theo bài + cap 2 lần chấm mỗi mục

- **Spec:** `specs/ipa-quota-per-lesson/spec.md`
- **Ngày:** 2026-08-19
- **Tác giả:** Tech Lead Agent
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** không có (không đổi kiến trúc nền, chỉ đổi granularity quota 1 feature)

## 1. Tóm tắt kiến trúc

Thay cơ chế quota IPA từ **đếm-theo-lần-chấm** (feature `pronunciation` trong hệ `quota.service`) sang **2 tầng riêng cho IPA**:

- **Tầng ngoài — số bài:** đếm tập `lessonId` mà user đã từng chấm audio; cap = **1 bài cho cả guest lẫn free**. Lưu trên `User`.
- **Tầng trong — 2 lần/mục:** tái dùng `IpaAttempt` count sẵn có cho `(userId, lessonId, itemId)`; ≥2 → chặn.

Cả 2 tầng gom vào **1 helper dùng chung** đặt trong module `ipa` (`ipa-quota.js` mới) để mọi entrypoint chấm (`submitItem`, `submitMiniTest`, `submitAttempt`) gọi cùng một chỗ (DRY). Hệ `quota.service` chung **không đổi** (talk/assessment/adaptive vẫn dùng như cũ); IPA đơn giản **thôi không gọi** `withQuota('pronunciation')` nữa → tự nhiên hết mượn bonus.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Nơi đặt logic gate | Helper mới `ipa-quota.js` trong module ipa | Gate riêng của IPA, khác semantics quota chung; gom 1 chỗ cho mọi path | Thêm 1 file; không tái dùng `withQuota` |
| Đếm số bài | Mảng `lessonId` trên User + `$size` guard atomic; cap 1 cho cả 2 tier | Cần biết bài nào đã tính để không trừ trùng | Mảng tối đa 1 phần tử ở trial (rất nhỏ) |
| Cap 2 lần/mục | Đếm `IpaAttempt` theo `(userId,lessonId,itemId)` | Đã có sẵn, không cần schema mới | Phụ thuộc `recordIpaAttempt` chạy sau mỗi lần chấm (đúng hiện trạng) |
| `pronunciation` cũ | Gỡ khỏi `TIERS`, ngừng dùng; giữ field schema `quotaUsed.pronunciation` | Tránh migration; field thừa vô hại | Field chết trong schema (ghi chú deprecated) |
| Refund tầng bài | Chỉ `$pull` lessonId nếu lần này vừa-thêm và LLM lỗi | Giữ tinh thần consume-before-LLM + refund (RT-3) | Thêm 1 nhánh logic nhỏ |

## 3. Data model

### 3.1 User (`user.model.js`) — thêm 1 field

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `quotaIpaLessons` | `[ObjectId]` (ref `IpaLesson`) | default `[]` | Tập bài đã tính lượt (distinct). `length` = số bài đã dùng. Chỉ trial dùng; unlimited bỏ qua. |
| `quotaUsed.pronunciation` | Number | (giữ) | **Deprecated** — ngừng ghi/đọc; để lại cho tương thích, không migrate. |

Không cần counter riêng cho tầng trong — dùng `IpaAttempt`.

### 3.2 Config (`quota.config.js`) — `TIERS`

- Thêm `ipaLessons`: guest **1**, free **1** (cả hai tier như nhau).
- Gỡ `pronunciation` khỏi `TIERS.guest`/`TIERS.free` (không còn feature này). `limitFor` sẽ throw nếu ai đó còn gọi `'pronunciation'` — đúng ý (fail-fast để lộ chỗ gọi sót).

## 4. Luồng dữ liệu

```
POST /ipa/lessons/:id/submit  (word|sentence|dialogue, audioBase64)
  verifyToken -> aiCallLimiter -> validate -> controller.submitItem
  -> ipaQuota.consumeIpaScoring(user, lessonId, itemId, { priorAttemptCount })
       0. unlimited (TRIAL_LIMIT_MODE|quotaUnlimited) -> allow, no writes
       1. tầng trong: priorAttemptCount >= 2 -> throw 403 ITEM_LIMIT (meta.limitKind='item')
       2. tầng ngoài:
            lessonId ∈ user.quotaIpaLessons -> allow (đã tính, justAdded=false)
            else atomic: findOneAndUpdate(
                   { _id, quotaIpaLessons: { $ne: lessonId },
                     $expr: { $lt: [ { $size: '$quotaIpaLessons' }, limit ] } },
                   { $addToSet: { quotaIpaLessons: lessonId } })
              matched -> justAdded=true ; null -> throw 402 LESSON_LIMIT (meta.limitKind='lesson', upgradeAction)
  -> llmProxy.analyzeAudio(...)          (paid call — sau khi gate qua)
       throw -> ipaQuota.refundLesson(user, lessonId, { justAdded })  // $pull nếu justAdded
  -> recordIpaAttempt(...)               (tạo IpaAttempt => tầng trong tự tăng cho lần sau)
  -> DTO { ..., quota: { lessonsRemaining, itemAttemptsRemaining } }
```

`submitMiniTest`: mỗi câu mic trong vòng lặp gọi cùng `consumeIpaScoring` với `itemId` của câu đó (per-question cap 2). Lượt bài tính 1 lần cho cả bài (câu đầu tiên trigger). `submitAttempt` (route `/attempts`, kind word/sentence/dialogue) gọi cùng helper.

## 5. Contracts

Không thêm file `contracts/` — chỉ mở rộng DTO trả về của các endpoint chấm (bổ sung khối `quota`), mô tả gọn tại §4 + §7. FE đọc thêm 2 field, backward-compatible.

- Response `submit`/`mini-test`/`attempts` thêm:
  `quota: { lessonsRemaining: number, itemAttemptsRemaining: number }` (unlimited → trả số lớn/`null` như convention hiện tại của quota DTO).
- 402/403 body: `{ code: 'QUOTA_EXCEEDED', meta: { limitKind: 'lesson'|'item', upgradeAction?, feature: 'ipa' } }`.

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/src/modules/ipa/ipa-quota.js` | Helper 2 tầng: `consumeIpaScoring`, `refundLesson`, `peekIpaQuota` | Tạo |
| `services/api/src/modules/quota/quota.config.js` | `TIERS`: thêm `ipaLessons` guest1/free1, gỡ `pronunciation` | Sửa |
| `services/api/src/modules/user/user.model.js` | Thêm `quotaIpaLessons: [ObjectId]` | Sửa |
| `services/api/src/modules/ipa/ipa.service.js` | `submitItem`/`submitMiniTest`/(`submitAttempt`) gọi helper thay `withQuota('pronunciation')` | Sửa |
| `services/api/src/modules/ipa/ipa.controller.js` | Bảo đảm DTO chuyển khối `quota` ra response | Sửa (nếu cần) |
| `services/api/src/__tests__/quota.service.test.js` · `quota.trial-mode.test.js` | Gỡ assert `pronunciation`; thêm test `ipaLessons` | Sửa |
| `services/api/src/modules/ipa/__tests__/ipa-quota.test.js` | Test helper 2 tầng (cap bài, cap mục, unlimited, refund, SRS) | Tạo |
| `exe-web/src/types/ipa.types.ts` | Thêm field `quota` + phân loại `limitKind` | Sửa |
| `exe-web/src/components/features/ipa/steps/SpeakingLoopStep.tsx` | Khoá mic khi mục đủ 2 lần; xử lý 403 item-limit | Sửa |
| `exe-web/src/components/features/quota/QuotaExceededModal.tsx` | Phân biệt "hết bài" vs "hết lượt mục" | Sửa |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code / meta |
|---|---|---|
| Hết số bài trial (bài mới, đã đủ cap) | 402 | `QUOTA_EXCEEDED`, `meta.limitKind='lesson'`, `upgradeAction` |
| Mục đã chấm đủ 2 lần | 403 | `QUOTA_EXCEEDED`, `meta.limitKind='item'` |
| LLM lỗi sau khi đã tính lượt bài | (giữ lỗi gốc) | refund: `$pull` lessonId nếu `justAdded` |

## 8. Bảo mật & quyền

Không thêm permission mới. `userId` luôn lấy từ `req.user.id` (không tin body) — giữ nguyên. Không có field nhạy cảm/`select:false`. `quotaIpaLessons` là dữ liệu của chính user, không cross-tenant.

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Coverage entrypoint:** phải chắc `submitAttempt` (merged FR-006) cũng đi qua helper — Task đầu sẽ audit `ipa.controller.js`/`ipa.service.js` xác nhận đủ 3 path, tránh lỗ hổng bỏ sót 1 route vẫn chấm free.
- **Đua khi thêm lessonId:** `$expr $size < limit` + `$addToSet` trong 1 `findOneAndUpdate` là atomic — 2 request đồng thời cho 2 bài mới không vượt cap. Test concurrency.
- **SRS review dùng chung 2 lượt:** đúng chủ đích (spec AC-8). Lưu ý `SpeakingLoopStep` review mode (`REVIEW_ITEM_CAP=3`) vẫn gọi `/submit` → cap 2 vẫn áp; không cần nhánh riêng.
- **`pronunciation` deprecated:** không migrate `quotaUsed.pronunciation` cũ; chấp nhận field chết. Nếu sau này bật lại per-item sẽ phải cân nhắc — ghi chú deprecated trong schema.
- **PR #69 đã merge:** mô hình này thay thế nó trong cùng code; không revert riêng, chỉ ghi đè logic + config.

## 10. Testing strategy

- **Unit (helper `ipa-quota.js`):** cap bài (guest1/free1, bài trùng không trừ lại), cap mục (lần 3 chặn), unlimited bypass, refund khi LLM lỗi (justAdded → pull; không justAdded → giữ), concurrency 2 bài mới.
- **Integration (ipa.service):** `submitItem`/`submitMiniTest`/`submitAttempt` đều gate; cảm âm/TAP/objective không trừ.
- **Regression:** `quota.service.test.js`/`quota.trial-mode.test.js` gỡ `pronunciation`; talk/assessment/adaptive không đổi.
- Chạy trong `exe-api/services/api`: `npx jest src/modules/ipa src/__tests__/quota*.test.js`.

## 11. Rollout / cross-repo sequencing

1. **BE (`exe-api`) trước:** merge config + helper + gate + DTO. BE tương thích ngược (FE cũ vẫn chạy, chỉ không hiển thị số mới).
2. **FE (`exe-web`) sau:** đọc field `quota` mới + khoá mic + modal phân biệt. PR riêng ở repo exe-web.
Không có bước deploy đồng thời bắt buộc (DTO thêm field, không breaking).

## 12. Đối chiếu Acceptance criteria

| Acceptance (spec §6) | Đáp ứng bởi |
|---|---|
| AC1–2 hết bài 402 | §4 tầng ngoài + §7 |
| AC3 cùng bài không trừ lại | §4 membership check |
| AC4–5 cap 2 lần/mục (gồm mini-test) | §4 tầng trong + submitMiniTest per-question |
| AC6 cảm âm/TAP/objective miễn phí | §4 chỉ gate path audio production |
| AC7 unlimited bypass | §4 bước 0 |
| AC8 SRS dùng chung 2 lượt | §9 (review mode vẫn qua cap) |
| AC9 bỏ bonus IPA | §1 (ngừng gọi quota chung) |
| AC10 FE phân biệt modal | §5 + §6 (SpeakingLoopStep/QuotaExceededModal) |
