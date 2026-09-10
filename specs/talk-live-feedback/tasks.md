# Tasks: Feedback cuối buổi (Phase A) — TDD checkbox

Thứ tự có phụ thuộc: api trước (T1–T7), web sau (T8–T12). Mỗi task độc lập test được.
Đường dẫn gốc: `exe-api/services/api` (BE), `exe-web` (FE).

## Backend (`api`)

- [ ] **T1 — Model `TalkLiveFeedback`**
  Tạo `src/modules/talk/talk-live-feedback.model.js` — collection `talk_live_feedback`, fields theo design §3.
  *Verify:* `node --check src/modules/talk/talk-live-feedback.model.js`

- [ ] **T2 — Rubric + prompt builder**
  Tạo `src/modules/talk/talk-feedback-rubric.js`: 5 tiêu chí seed + `buildFeedbackPrompt({ transcript, cefr })` trả messages ép JSON đúng shape (design §5), có/không CEFR.
  Test-first: `src/__tests__/talk-feedback-rubric.test.js` — prompt chứa đủ 5 khoá tiêu chí; có CEFR thì nhắc level; transcript được nhúng.
  *Verify:* `npx jest src/__tests__/talk-feedback-rubric.test.js`

- [ ] **T3 — Parse JSON phòng thủ**
  Trong service (hoặc util `talk-feedback-parse.js`): strip ```json fence, parse; hỏng → `null`.
  Test-first: hợp lệ / có code-fence / rác → fallback null.
  *Verify:* `npx jest` (cùng file test service)

- [ ] **T4 — `talk-feedback.service.js`**
  `generateFeedback({ userId, scenarioId, transcript, cefr })`: guard transcript rỗng/quá ngắn → throw 422 `NOT_ENOUGH_DATA`; gọi `llmProxy.chat` (mock trong test); parse (T3) → hỏng thì fallback tối giản; lưu `TalkLiveFeedback`; trả feedback.
  Test-first (mock `llm-proxy.adapter` + model): transcript rỗng → 422; chat trả JSON hợp lệ → lưu + trả đúng shape; chat trả rác → fallback không throw.
  *Verify:* `npx jest src/__tests__/talk-feedback.service.test.js`

- [ ] **T5 — Schema + controller + route**
  `talk.schema.js`: Joi body `{scenarioId, transcript[]}` (role enum, cap length). `talk.controller.js`: +`createLiveFeedback` (userId từ JWT, cefr từ user). `talk.routes.js`: `POST /live/feedback` (verifyToken, aiCallLimiter, validate).
  *Verify:* `node --check` 3 file + route xuất hiện.

- [ ] **T6 — Error code**
  Thêm `NOT_ENOUGH_DATA` vào `constants/error-codes.js` (nếu chưa có).
  *Verify:* grep thấy code.

- [ ] **T7 — Integration test endpoint**
  `POST /talk/live/feedback` không JWT → 401; có JWT + transcript hợp lệ (mock chat) → 200 + shape; transcript rỗng → 422.
  *Verify:* `npx jest src/__tests__/talk-live-feedback.api.test.js`

## Frontend (`web`)

- [ ] **T8 — Service + types**
  `services/talk-live.service.ts`: +`postLiveFeedback({scenarioId, transcript})` → `POST /talk/live/feedback`; type `TalkFeedback` (design §5).
  *Verify:* `npx tsc --noEmit` (lọc file).

- [ ] **T9 — Gom transcript trong hook**
  `hooks/use-live-session.ts`: đăng ký `registerTextStreamHandler('lk.transcription')`, tích luỹ `[{role,text}]` (role theo sender identity: local = 'user', agent = 'ai'); expose `transcript`. Dọn khi disconnect.
  *Verify:* `npx tsc --noEmit` + `npx eslint`.

- [ ] **T10 — Mutation hook**
  `hooks/use-talk-feedback.ts`: TanStack `useMutation` gọi `postLiveFeedback`; trạng thái loading/error/data.
  *Verify:* `npx tsc --noEmit`.

- [ ] **T11 — Wire UI**
  `AssessmentCard.tsx`: nhận prop `feedback` thật (bỏ mock), render tổng + 5 mục + topImprovements; state trống/đang tạo. `TalkLivePanel.tsx`: khi kết thúc + transcript đủ → gọi mutation → hiện card; transcript rỗng → thông báo nhẹ.
  *Verify:* `npx tsc --noEmit` + `npx eslint` + chạy thử tay (kết thúc buổi → thấy card).

- [ ] **T12 — Lint/type toàn phần FE thay đổi**
  *Verify:* `npx tsc --noEmit` sạch cho file Phase A; `npx eslint <các file>` sạch.

## Success criteria (Definition of Done)
- Toàn bộ T1–T12 checked; jest (api) + tsc/eslint (web) xanh.
- Đối chiếu đủ AC1–AC7 (spec §6).
- Không đụng `/talk/turn` cũ; phát âm (Phase B) chưa làm.
