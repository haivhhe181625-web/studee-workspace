---
feature: M2 IPA — FE migrate sang endpoint gộp /attempts (hoàn tất FR-006)
slug: m2-ipa-attempts-migration
status: approved (2026-08-15) — Phần A + Phần B duyệt, code Phần A trước
tier: lite
version: v1.0
created: 2026-08-15
parent-spec: specs/m2-ipa-pronunciation-refactor/
repos: [exe-web (chính), exe-api (chỉ đọc — verify)]
branch: feat/ipa-m2-pronunciation-refactor
---

# Spec — FE migrate sang `/attempts` (hoàn tất FR-006)

## 1. Bối cảnh & vì sao

Spec M2 gốc (`specs/m2-ipa-pronunciation-refactor/`) đã build endpoint gộp
`POST /lessons/:id/attempts` ở BE (Phase 2, có test) nhưng **cố ý HOÃN** việc chuyển FE
sang nó — xem quyết định mở `specs/m2-ipa-pronunciation-refactor/tasks.md` §"Quyết định
mở — FE chuyển /attempts?" (dòng 93-95): *"cần user chốt: hoãn / làm luôn"*.

**User đã chốt: LÀM LUÔN.** Spec này resolve đúng quyết định mở đó (không lặp lại nội
dung spec gốc — chỉ tham chiếu).

**Trạng thái hiện tại (verified):**
- FE ghi điểm qua **2 route legacy còn hoạt động**: `POST /lessons/:id/submit`
  (`submitIpaItem`, dùng cho word/sentence/dialogue) + `POST /lessons/:id/practice`
  (`recordIpaPractice`, dùng cho cảm âm m3). Nguồn: `exe-web/src/services/ipa.service.ts:67,93`.
- **Không có** chỗ nào ở FE gọi `/attempts` (đã grep).
- BE `controller.submitAttempt` (`exe-api/services/api/src/modules/ipa/ipa.controller.js:62`)
  chỉ **dispatch nội bộ** về chính service cũ (`submitItem`/`recordPractice`/`recordReview`)
  → **response y hệt** route legacy.

**Vì sao làm bây giờ:** route legacy chỉ sống trong **cửa sổ rollback 1 tuần** (FR-012).
Sau khi BE gỡ → FE gãy. DoD spec v1.3: *"FE mới gọi ≤ 2 endpoint"* — hiện chưa đạt.

Spec này có **2 phần**:
- **Phần A — Migrate write path** (plumbing, UI không đổi): §2 dưới đây đủ, không cần design.
- **Phần B — SRS review flow** (feature UI mới, có fork thiết kế): xem **`design.md`** cùng
  thư mục; cần **design sign-off** trước khi code.

## 2. Design note Phần A (đủ gọn — không cần design.md)

Đây là **thay ống nước phía sau**, UI không đổi:

- **Giữ nguyên tên hàm** service (`submitIpaItem`, `recordIpaPractice`) + type
  (`IpaSubmitPayload`, `IpaPracticePayload`) + hook (`use-ipa.ts`) + component
  (`SpeakingLoopStep`, `ObjectiveChoiceStep`). **Chỉ đổi URL + bọc body** bên trong service.
- Mapping body:

  | Hàm FE | Trước (legacy) | Sau (`/attempts`) |
  |---|---|---|
  | `submitIpaItem` | `POST /submit` `{itemType, itemId, audioBase64, ...}` | `POST /attempts` `{kind: itemType, payload:{itemId, audioBase64, answer?, tapPattern?, attemptId?}}` |
  | `recordIpaPractice` | `POST /practice` `{screen, score}` | `POST /attempts` `{kind:"perception", payload:{screen, score}}` |

- **`kind` hợp lệ BE** (`ipa.schema.js:66`): `word|sentence|dialogue|perception|srs_review`.
  Vòng luyện nói chỉ gửi `word/sentence/dialogue` (map `ITEM_TYPE` ở `SpeakingLoopStep`)
  → đều là kind hợp lệ. `minimalPair`/`miniTest` **không** phải kind hợp lệ ⇒ phải xác nhận
  không caller nào đẩy 2 giá trị này qua `submitIpaItem` (T-guard).
- **Giữ timeout theo kind**: audio (word/sentence/dialogue) giữ 60s như hiện tại
  (`ipa.service.ts:76`); perception nhanh, timeout mặc định.
- Response type **không đổi** (`IpaSubmitResult`, `{screen,score}`).

## 3. Phạm vi

### Trong scope — Phần A (code — exe-web)
1. **Migrate `submitIpaItem` + `recordIpaPractice`** sang `POST /attempts` (mapping §2).
2. **Dọn dead code FE**: `recordIpaWarmup` + `/warmup` (màn warmup đã bỏ v1.3);
   xác minh & dọn `submitIpaMiniTest` + `/mini-test` **nếu** không còn component nào gọi
   (v1.3 bỏ màn Mini Test — Phase 6 spec gốc). Chỉ xóa hàm khi grep sạch caller.

### Trong scope — Phần B (feature mới — exe-web, xem design.md)
3. **SRS review flow (AC-007, FR-008/D6)**: FE hiện **chưa có** UI ôn — `getIpaDueReviews`
   định nghĩa nhưng 0 caller; `buildSteps()` không có review mode. Cần:
   - a) **Danh sách bài tới hạn ôn** (dùng `GET /ipa/srs/due` đã có).
   - b) **Luồng ôn rút gọn**: `buildSteps(mode:"review")` bỏ màn Học, chỉ vòng luyện nói
     (60–90s). Quyết định chi tiết (bỏ cảm âm?, cap số mục?) ở `design.md`.
   - c) **Ghi kết quả ôn**: cuối vòng → `POST /attempts {kind:"srs_review", payload:{score}}`;
     `score` = điểm tổng lần ôn (nguồn score: design.md). BE `recordReview` tự quyết
     đạt/không theo ngưỡng 80 → đẩy SRS 1d→7d→mastered.

### Trong scope (chỉ verify — exe-api, không sửa trừ khi lệch)
3. **Verify SRS 2 nhịp 1d→7d** (FR-008/D6). Spec gốc Phase 5 (T5.1) khai đã xong +
   test `advance-review 4/4`. Task = chạy lại test đó + đọc `ipa-srs`/`advanceReview` xác
   nhận; **chỉ fix nếu** lệch với "1 ngày → 7 ngày → mastered".

### Ngoài scope (KHÔNG code ở spec này)
- **Gỡ route legacy ở BE** (`/submit`,`/practice`,`/mini-test`,`/warmup`) — làm ở PR
  retire-legacy sau khi hết cửa sổ rollback.
- **Drop field vestigial** (`earTrainingTypes`, `masteredStreak`, `warmUpConfidence`,
  `easeFactor`) + dead adapter (`buildPatternMatch`/`buildIpaRecognition`) — gom vào PR
  retire-legacy (spec gốc `tasks.md` §Ghi chú).
- **Cập nhật Plane v1.3** (giữ nút Nghe chậm) — việc doc, riêng.
- **Order + ingest audio MiniMax** — content/ops, manifest ở `plans/260814-1545-ipa-audio-order/`.
- (đã gộp vào Phần B: `kind:"srs_review"` + màn ôn rút gọn)

## 4. Acceptance Criteria (checkable)

- [ ] **AC-1** Grep `exe-web/src` sạch: không còn `/ipa/lessons/*/submit` và
  `/ipa/lessons/*/practice` trong đường ghi điểm đang dùng (chỉ còn `/attempts`).
- [ ] **AC-2** `submitIpaItem` gọi `POST /attempts` với `{kind, payload}`; audio kind giữ
  timeout 60s; response `IpaSubmitResult` không đổi.
- [ ] **AC-3** `recordIpaPractice` gọi `POST /attempts` `{kind:"perception", payload:{screen,score}}`.
- [ ] **AC-4** Hành vi UI + điểm số **không đổi**: chạy 1 bài S1–S4 → ghi âm chấm điểm +
  cảm âm 4 câu ≥3/4 vẫn mở được Kết quả; Network tab thấy `/attempts` thay 2 route cũ.
- [ ] **AC-5** DoD v1.3: đường ghi điểm FE = **1 endpoint** (`/attempts`); tổng endpoint
  IPA đang dùng ≤ 2 write (`/start` + `/attempts`) ngoài các GET đọc.
- [ ] **AC-6** Dead code: `recordIpaWarmup`/`/warmup` gỡ (và `submitIpaMiniTest`/`/mini-test`
  nếu không còn caller); build không cảnh báo unused.
- [ ] **AC-7** Toàn bộ test FE hiện có (137) **vẫn xanh** + test mới cho `ipa.service`
  migrate xanh; `tsc --noEmit` exit 0; `eslint` file đổi exit 0.
- [ ] **AC-8** (verify) SRS 2 nhịp 1d→7d: test BE `advance-review` xanh; hành vi khớp D6
  (hoặc ghi rõ lệch + đề xuất fix).

### Phần B — SRS review flow
- [ ] **AC-9** Người học thấy **danh sách bài tới hạn ôn** (từ `GET /ipa/srs/due`); rỗng
  thì trạng thái empty rõ ràng.
- [ ] **AC-10** Vào ôn 1 bài → luồng **bỏ màn Học**, chỉ vòng luyện nói rút gọn; `buildSteps("review")`
  không chứa key `hoc`.
- [ ] **AC-11** Xong vòng ôn → FE gọi **đúng 1 lần** `POST /attempts {kind:"srs_review", payload:{score}}`;
  `score` đúng nguồn đã chốt ở design.md.
- [ ] **AC-12** SRS tiến nhịp thật: ôn đạt (score≥80) ở nhịp 1 → hẹn 7 ngày; nhịp 2 đạt →
  `status=mastered` (verify qua `GET /progress` hoặc socket `ipa:mastered`); ôn trượt → giữ
  bài, `needsReview`, không thu hồi (AC-008 spec gốc).
- [ ] **AC-13** Test FE cho `buildSteps("review")` + đường gọi `srs_review` xanh.

## 5. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Đổi body sai shape → 400 từ Joi `submitAttempt` | Test service map đúng `{kind,payload}`; chạy thật 1 bài |
| Caller ngầm đẩy `minimalPair`/`miniTest` qua `submitIpaItem` → kind không hợp lệ | T-guard: grep caller trước khi migrate |
| Xóa nhầm hàm còn dùng | Chỉ xóa khi grep caller sạch (AC-6) |
| Regression điểm/cảm âm | AC-4 chạy thật + 137 test cũ (AC-7) |

## 6. Open Questions

- [x] **OQ-1 → CHỐT (gộp):** SRS review flow đưa vào **Phần B** spec này. Chi tiết fork +
  quyết định thiết kế ở `design.md` — cần **design sign-off** trước khi code.
- [x] **OQ-2 → CHỐT:** Thời điểm BE gỡ legacy **linh hoạt, không ràng buộc**. Điều kiện
  duy nhất: **công việc spec này phải HOÀN THIỆN + thay thế được legacy** trước khi gỡ.
  → Đây là **thanh hoàn chỉnh (completeness bar)**, không phải deadline: Phần A phải khiến
  toàn bộ đường ghi điểm FE chạy qua `/attempts` (AC-1, AC-5) và dead route (`/mini-test`,
  `/warmup`) được xác nhận không còn caller (AC-6) → khi đó BE gỡ legacy lúc nào cũng an toàn.
