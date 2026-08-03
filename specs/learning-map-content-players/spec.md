# Spec: Learning Map Content Players (MCQ, Flashcard, Video Lesson)

- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Trạng thái:** Đã build (spec hồi tố)
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — Xây dựng trong 3 phase song song: node player endpoints `/start` + `/submit`, anti-cheat grading, content resolver với signed URL cho video.
  - `web` (`exe-web`) — FE learner players cho 3 node kinds: MCQ quiz renderer, flashcard self-rater, video player với watchedRatio reporting.
  - `admin` (`exe-admin`) — Node editor UI (reuse existing), thêm nút Verify cho VIDEO_LESSON ref; browse system flashcard decks.
- **Module liên quan (exe-api):**
  - `modules/learning-map/` (hoặc `adaptive/`) — startNode/submitNode endpoints, node player adapters, item binding logic.
  - `modules/assessment/` — AssessmentQuestion bank (MCQ source), answerKey/itemType schemas.
  - `modules/flashcard/` — Flashcard deck model (`ownerType:'system'`|`'user'`), card CRUD.
  - `modules/course-content/` — Media resolver, signed URL generation (video playback), reference validation.
  - `modules/adaptive/` — Mastery/gating logic (KHÔNG đổi), learning-event tracking.
- **Trạng thái:** Đã build — 3 player types live trên node, gating guardrails enforced. Spec này hồi tố code hiện tại từ plans 260726–260727.

## 1. Mục tiêu

Thay thế SubmitSimulator với **3 content player** phục vụ nội dung **thật, curated, server-graded** cho 3 loại node trong learning map:
- **MCQ_QUIZ**: Câu hỏi trắc nghiệm curated (cố định gắn vào node), chấm server-side, answerKey ẩn.
- **FLASHCARD_DECK**: Bộ thẻ hệ thống được gán vào node, học viên tự đánh giá FSRS, bind CardIds chống giả.
- **VIDEO_LESSON**: Video MP4 self-hosted (signed URL), self-report watchedRatio, không time-gate.

Mỗi player: resolve nội dung `/start` → bind resource (items/cards/media) → chấm server-keyed `/submit` → unlock node kế.

## 2. Bối cảnh

**Vì sao cần:** Trước đây, node player (SubmitSimulator) chấp nhận submission giả (client gửi `[{correct:true}]` = pass). Gamified learning-map cần **nội dung thật** để:
- Admin tạo khóa học cấu trúc Lộ trình → Chặng → Chuyên đề → Bài học (import file hoặc UI).
- Bài học tham chiếu câu hỏi MCQ thật, bộ thẻ flashcard, video học.
- Học viên làm bài thật → mastery-based unlock vào node kế.

**Hiện trạng đã kiểm tra trong code:**
- Learning map template + roadmap 2-tầng (Roadmap → Phase) đã sẵn, mastery/gating logic chạy OK.
- SubmitSimulator cũ (test-out flow) dùng `config.answerKey` hardcoded + chấm client-trust → vẫn sống cho fallback node không có content.
- AssessmentQuestion model tồn tại (`itemType: 'mcq'|'audio_mcq'|'mcq_multi'`, `answerKey` — string hoặc array, `select:false`).
- Flashcard model tồn tại (`FlashcardDeck`, `FlashcardCard`, `ownerType: 'user'|'system'`).
- Media storage + signed URL (`course-media`, `signMediaUrl`) đã có, dùng cho course-content + khóa học nội dung trao phí.
- Adapter pattern (`activity/adapters/mcq.adapter.js`, `flashcard.adapter.js`, `video.adapter.js`) chấm submission, trả mastery signal.
- **Chưa có** node player resolver đa hình — hiện `/start` trả raw activityRef, FE phải tự suy luận.

## 3. Phạm vi

### Trong phạm vi

- **IS-1** — 3 node player types: `MCQ_QUIZ`, `FLASHCARD_DECK`, `VIDEO_LESSON` là loại activity phục vụ nội dung thật.
- **IS-2** — MCQ node: `config.itemIds` định sẵn curated câu hỏi (ObjectId AssessmentQuestion), `/start` resolve + bind + hide answerKey; `/submit` chấm server-side từ binding.
- **IS-3** — Flashcard node: `activityRef.refId` trỏ `FlashcardDeck` (chỉ `ownerType:'system'` được phép); `/start` resolve deck → bind cardIds; `/submit` chấm FSRS từ bound set.
- **IS-4** — Video node: `activityRef.slug` là internal ref (`course-video/<file>`) hoặc external URL; `/start` resolve signed URL (nếu internal); `/submit` chấm watchedRatio tự report.
- **IS-5** — Anti-cheat/anti-leak: answerKey KHÔNG lộ client (MCQ), only deck `ownerType:'system'` (flashcard), media signed (video).
- **IS-6** — Binding guardrail: bind resource (items/cards) phục vụ single attempt — `/submit` GETDEL atomic; replay không tái dùng binding.
- **IS-7** — Gating guardrail: **FLASHCARD_HARD_GATE + VIDEO_HARD_GATE** — flashcard/video KHÔNG được làm prerequisite của node có `gatePolicy==='HARD'` (publish chặn). Chỉ dùng cho CORE/PRACTICE/SOFT.
- **IS-8** — Adapter độc lập: mỗi node kind có adapter riêng (mcq/flashcard/video), chấm submission → `NormalizedResult` (score/passed/mastery).
- **IS-9** — Error codes: `NODE_NO_ITEMS` (MCQ thiếu item), `FLASHCARD_DECK_NOT_FOUND` (deck không tồn tại hoặc private), `VIDEO_NO_SOURCE` (video ref invalid), `VIDEO_SOURCE_MISSING` (internal ref file không tồn tại), `FLASHCARD_HARD_GATE`/`VIDEO_HARD_GATE` (publish validation).
- **IS-10** — Idempotency: re-start hoặc re-submit cùng `attemptNonce` không double-score; replay trả kết quả cũ.

### Ngoài phạm vi

- Sinh nội dung động (MCQ random bank, flashcard random cards) — **Lý do:** 3 player dùng content **curated cố định** gắn admin vào node; random thuộc flow độc lập (`test-out`).
- Persist SRS user từ map node — **Lý do:** adapter chỉ derive pass/score/mastery (throwaway grade), KHÔNG update Anki scheduling.
- Upload/host media server-side — **Lý do:** video dùng self-host (CDN/YouTube) hoặc signed internal ref đã có, không xây upload mới.
- Multi-deck/multi-video node — **Lý do:** 1 node = 1 loại activity (1 deck, 1 video, 1 bộ MCQ). Combo nội dung ở bài học cấp cao hơn (Phase 3).
- Flashcard advanced (spaced repetition, interval-tracking) — **Lý do:** map node chỉ grade qua FSRS enum (again/hard/good/easy), không persist state.
- Time-gate/seek-protection video — **Lý do:** pure self-report watchedRatio, KHÔNG giả lập xem. Ungainability = architectural choice (A2).
- Learner UI nâng cao (progress bar, bookmark, note) — **Lý do:** player cơ bản chỉ render + submit, enhancement là Phase 4+.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Admin nội dung | Gán câu hỏi MCQ curated vào node MCQ_QUIZ (qua `config.itemIds`) | Học viên làm bài test thật thay vì SubmitSimulator giả |
| Admin nội dung | Gán bộ thẻ hệ thống vào node FLASHCARD_DECK (qua `activityRef.refId`) | Học viên luyện từ vựng thực tế bằng flashcard đã kiểm duyệt |
| Admin nội dung | Gán video lesson vào node VIDEO_LESSON (qua `activityRef.slug`) | Học viên xem video giáo dục + self-report tiến độ |
| Learner | Start node → nhìn nội dung thật (MCQ/flashcard/video) | Biết là phải làm bài thật, không sim; tính seriousness cao |
| Learner | Làm bài → submit → nhận feedback mastery | Biết sẽ nâng cấp nếu pass, unlock node kế |
| Tech lead / QA | Publish node → validate nội dung tồn tại (item hợp lệ, deck hệ thống, video source) | Tránh node broken (missing content) đẩy learner; publish chặn item không tồn tại |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết | Trạng thái |
|---|---|---|---|
| MCQ_QUIZ: answerKey là **chỉ số option (index)** hay **value tên câu trả lời**? | **INDEX** (0,1,2,3…) — client gửi `choice=String(optionIndex)`. Index-agnostic, dễ migrate nếu sắp xếp lại câu trả lời. | Owner (2026-07-26) | ✅ Chốt — INDEX |
| MCQ_QUIZ: item **curated cố định** hay **random từ bank**? | **CURATED** — `config.itemIds` định sẵn danh sách ObjectId. Vừa chống gian lận (lần 2 same câu), vừa fair (admin kiểm soát độ khó). | Owner (2026-07-26) | ✅ Chốt — CURATED |
| Flashcard: chỉ deck **hệ thống (ownerType:'system')** hay cũng cho **user deck**? | **Chỉ hệ thống** — `ownerType:'system'` được publish trên learning map. User deck = private thuộc learner, không embed vào map global. | Owner (2026-07-27) | ✅ Chốt — HỆ THỐNG |
| Flashcard: gating policy **CÓ thể là HARD** hay **CHỈ SOFT/PRACTICE**? | **CẤM HARD** (`FLASHCARD_HARD_GATE`). Tự đánh giá → không cho đảm bảo kiến thức trước (giống video). Chỉ CORE/PRACTICE/SOFT. | Owner (2026-07-27) | ✅ Chốt — CẤM HARD |
| Video: signed URL 6h **có bind userId** hay **share được cho người ngoài**? | **SHARE được** (không bind userID) — consistent scheme nền tảng (course-content chung). Siết TTL/user-bind = hardening riêng TOÀN NỀN, để sau. | Owner (2026-07-27) | ✅ Chốt — SCHEME CHUNG |
| Video: time-gate/seek-protection **có thực hiện** hay **pure self-report**? | **PURE SELF-REPORT** — watchedRatio client-trust (giống flashcard tự đánh giá). KHÔNG forge-proof, KHÔNG interval-tracking. | Owner (2026-07-27) | ✅ Chốt — PURE SELF-REPORT |

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map về một mục "Trong phạm vi".

1. Admin gán `config.itemIds` curated → learner Start node MCQ → thấy 5 câu đúng thứ tự, không lộ đáp án → submit 5 câu → chấm server key → pass/fail → unlock node kế. *(IS-2, IS-10)*
2. File MCQ chứa `answerKey`, item `itemType:'essay'` lẫn `'mcq'` → system filter chỉ MCQ, answerKey NOT lộ `/start`. *(IS-2, IS-5)*
3. Admin gán flashcard deck `ownerType:'system'` → learner Start → thấy deck card, self-rate → submit → chấm bound set → pass → unlock. *(IS-3, IS-6, IS-8)*
4. Admin gán deck `ownerType:'user'` → node publish fail (422) / learner Start fail (409). *(IS-3, IS-5)*
5. Admin gán node flashcard với gating `HARD` → publish fail (FLASHCARD_HARD_GATE). *(IS-7)*
6. Admin gán video internal ref `/start` → signed URL + minWatchRatio; external URL → pass-through. *(IS-4)*
7. Admin gán video ref không tồn tại → publish fail (VIDEO_NO_SOURCE / VIDEO_SOURCE_MISSING). *(IS-9)*
8. Admin gán node video với gating `HARD` → publish fail (VIDEO_HARD_GATE). *(IS-7)*
9. Learner submit 2 lần cùng `attemptNonce` → lần 1 score 0.8 passed, lần 2 `idempotent:true` (không double-score, không retract). *(IS-10)*
10. Learner submit MCQ: submit count < bind count → score = đúng/total, không quy đổi. Submit item không thuộc bind → bỏ qua. *(IS-2, IS-6)*
11. Learner submit flashcard: submit rating cho card không bound → bỏ qua; thiếu card → miss. Total = |bound|. *(IS-3, IS-6)*
12. Learner submit video: watchedRatio < minWatchRatio → failed; ≥ → passed. *(IS-4)*
13. Error code `NODE_NO_ITEMS` trả 409 khi node MCQ `config.itemIds` rỗng hoặc item hợp lệ < MIN (3). *(IS-9)*

## 7. Câu hỏi mở

Không có. Toàn bộ quyết định lớn đã chốt (curated index, deck hệ thống, HARD gate chặn, pure self-report).

## 8. Ghi chú cho Tech Lead Design

**Ràng buộc đã biết (KHÔNG phải giải pháp):**

- **Curated = cố định:** `config.itemIds` học viên lần 2 Start thấy cùng bộ câu, cùng thứ tự → bỏ được randomize + reroll. Hệ quả: bind Redis **CHỈ cần single-use** (không store state), GETDEL atomic.
- **Answerkey index (D1):** normalize **TOÀN** seed data → index. Client gửi `choice='0'|'1'|...`, server coerce `String(answerKey)===String(choice)` (không strict ===, tránh type err).
- **Anti-leak (D4):** `activityDescriptor` phải whitelist `config` (bỏ `itemIds`, `answerKey`); `publicItem` strip key; `/submit` response ẩn `config` raw.
- **Binding tamper-guard:** `total=|bound|` (số key), không `answers.length`; câu thiếu=sai; bỏ qua `a.correct` client-trust. Flashcard: bound cardIds chốt, card lạ bỏ, miss = câu thiếu rating.
- **Guardrail publish (IS-7):** node publish PHẢI check `gatePolicy==='HARD'` + `nodeKind ∈ {FLASHCARD_DECK, VIDEO_LESSON}` → 422 `FLASHCARD_HARD_GATE` / `VIDEO_HARD_GATE`.
- **Idempotency (IS-10):** `ActivityResult` table track `{userId, attemptNonce}` TRƯỚC grade → replay trả `idempotent:true`. GETDEL binding PHẢI AFTER idempotency check (không retract MASTERED sau replay).
- **Video signed URL:** `signMediaUrl(ref)` TTL 6h (scheme nền tảng), không bind userID → share được. Xác nhận `course-content.public.service` có hàm, dùng nguyên.

---

**Xem thêm:**
- Design chi tiết: `design.md`
- Data model: `data-model.md`
- API contract: `contracts/map-node-players.md`
- Gating guardrail logic: cross-ref `specs/course-driven-map/` (IS-7 thực hiện ở validate stage)
