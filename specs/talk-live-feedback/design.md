# Design: Feedback cuối buổi cho Trò chuyện trực tiếp (Phase A)

- **Spec:** `specs/talk-live-feedback/spec.md`
- **Ngày:** 2026-09-10
- **Tác giả:** Tech Lead Agent + dev
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** không có mới (nằm trong khung ADR-0002 realtime voice)

## 1. Tóm tắt kiến trúc

Lớp feedback tách khỏi luồng realtime: FE gom transcript trong buổi (text-stream `lk.transcription` của LiveKit, có sẵn), kết thúc buổi POST lên `api`. `api` dựng prompt từ rubric seed cứng + CEFR → gọi `llm /chat` (tái dùng) → nhận JSON feedback → lưu (`TalkLiveFeedback`) → trả FE hiển thị ở `AssessmentCard`. Không sửa worker `llm`, không tạo route AI mới. Đúng nguyên tắc: business logic ở `api`, AI ở `llm`.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Nguồn transcript | FE gom từ `lk.transcription` rồi gửi api | Không phải sửa worker; dùng FE→api→llm sẵn có | Transcript client-controlled (chấp nhận: feedback không gating) |
| Sinh feedback ở đâu | `api` orchestrate, gọi `llm /chat` | Rubric/lưu là business (Node); AI proxy (llm) | api phải parse JSON từ llm |
| Rubric | Seed cứng trong code (`talk-feedback-rubric.js`) | YAGNI; admin hoá sau | Đổi rubric phải sửa code |
| Khi nào sinh | Cuối buổi, 1 lần | Rẻ + tổng thể; tức thì đã do gia sư | Không có báo cáo giữa buổi |
| Lưu | Model mới `TalkLiveFeedback` (kèm transcript) | Cần cho lịch sử + Phase B phát âm | Thêm 1 collection |

## 3. Data model

Collection `talk_live_feedback` (model `TalkLiveFeedback`):

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| userId | ObjectId | ✓ | từ JWT |
| scenarioId | String | ✓ | kịch bản buổi nói |
| cefr | String | | trình độ lúc chấm (A1..C2) |
| transcript | `[{ role:'user'\|'ai', text }]` | ✓ | dữ liệu buổi |
| feedback | Object (xem §5) | ✓ | JSON rubric AI trả |
| createdAt | Date | ✓ | |

## 4. Luồng dữ liệu

```
Trong buổi: FE (use-live-session) đăng ký registerTextStreamHandler('lk.transcription')
  → tích luỹ transcript [{role,text}]  (role: local identity = 'user', agent = 'ai')
Kết thúc buổi (Kết thúc/Disconnected) + transcript đủ:
FE  --(POST /api/talk/live/feedback, JWT, {scenarioId, transcript})-->  api
      verifyToken → guard (transcript rỗng/quá ngắn → 422 NOT_ENOUGH_DATA)
      -> talk-feedback.service: build prompt (rubric + cefr từ user.level) 
      -> llmProxy.chat(messages)  [tái dùng]
      -> parse JSON (fallback an toàn nếu hỏng)
      -> lưu TalkLiveFeedback
      -> trả feedback DTO
FE  -> AssessmentCard hiển thị (tổng + 5 mục + topImprovements)
```

## 5. Contracts

- `contracts/talk-live-feedback.md` — `POST /api/talk/live/feedback`.

Request: `{ scenarioId: string, transcript: [{role:'user'|'ai', text:string}] }` (JWT).
Response `{ feedback }` với shape:
```
feedback = {
  overall: string,
  topImprovements: string[],            // 2–3 điều
  criteria: {
    grammar:    { level: string, examples: string[], suggestion: string },
    vocabulary: { level, examples, suggestion },
    task:       { level, examples, suggestion },
    interaction:{ level, examples, suggestion },
    coherence:  { level, examples, suggestion }
  }
}
```

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `api/.../talk/talk-feedback-rubric.js` | 5 tiêu chí seed + build prompt (JSON-forcing, CEFR) | Tạo |
| `api/.../talk/talk-feedback.service.js` | Orchestrate: guard → chat → parse → lưu | Tạo |
| `api/.../talk/talk-live-feedback.model.js` | Model `TalkLiveFeedback` | Tạo |
| `api/.../talk/talk.controller.js` | +`createLiveFeedback` | Sửa |
| `api/.../talk/talk.routes.js` | +`POST /live/feedback` (verifyToken, aiCallLimiter) | Sửa |
| `api/.../talk/talk.schema.js` | Joi cho body feedback | Sửa |
| `web/src/services/talk-live.service.ts` | +`postLiveFeedback(payload)` | Sửa |
| `web/src/hooks/use-live-session.ts` | Gom transcript từ `lk.transcription` | Sửa |
| `web/src/hooks/use-talk-feedback.ts` | TanStack mutation gọi feedback | Tạo |
| `web/src/components/features/talk/LiveFeedbackReport.tsx` | Card báo cáo góp ý (thay vì sửa AssessmentCard — nó thuộc mode nói-theo-lượt cũ, ngoài scope) | Tạo |
| `web/src/components/features/talk/TalkLivePanel.tsx` | Gom transcript → kết thúc → trigger feedback → render report | Sửa |

## 7. Xử lý lỗi

| Tình huống | Mã | code |
|---|---|---|
| Chưa đăng nhập | 401 | NOT_AUTHENTICATED |
| Transcript rỗng/quá ngắn | 422 | NOT_ENOUGH_DATA |
| llm /chat lỗi | 503 | SERVICE_UNAVAILABLE |
| AI trả JSON hỏng | 200 + fallback | (feedback tối giản "chưa phân tích được chi tiết") |

## 8. Bảo mật & quyền

- `verifyToken` — userId từ JWT, không từ body. `aiCallLimiter` chặn spam (mỗi buổi 1 call, tốn token).
- Transcript client-controlled → chỉ ảnh hưởng feedback của chính user, không cross-user. Chấp nhận ở Phase A; hardening (transcript phía worker) ghi là follow-up.
- Không log transcript ra ngoài; lưu trong collection của user.

## 9. Rủi ro / câu hỏi kỹ thuật mở

- Model đôi lúc trả JSON kèm markdown/```json → parse phải strip code-fence + thử/parse phòng thủ.
- Transcript dài (buổi 5') → cắt bớt hoặc tóm nếu vượt context (đặt trần ký tự, giữ lượt gần nhất).

## 10. Testing strategy

- `api` unit: rubric prompt build (có/không CEFR); parse JSON (hợp lệ / có code-fence / hỏng → fallback); guard transcript rỗng → 422.
- `api` integration: POST feedback (mock llmProxy.chat) → lưu + trả DTO đúng shape; không JWT → 401.
- FE: hook gom transcript đúng role; AssessmentCard render từ feedback thật.

## 11. Rollout / cross-repo sequencing

Thứ tự: `api` (rubric+endpoint+model) → `web` (gom transcript + card). `llm` không đổi. Không đụng mode cũ.

## 12. Đối chiếu Acceptance criteria

| AC | Đáp ứng bởi |
|---|---|
| AC1 (tự sinh + hiển thị) | §4 luồng + §6 TalkLivePanel/AssessmentCard |
| AC2 (tổng + 5 mục + top) | §5 shape feedback |
| AC3 (CEFR) | §6 rubric build dùng user.level |
| AC4 (401) | §8 verifyToken |
| AC5 (rỗng → báo nhẹ) | §7 422 NOT_ENOUGH_DATA |
| AC6 (JSON hỏng → fallback) | §7 fallback + §9 |
| AC7 (mode cũ nguyên) | §1 tách lớp, không sửa /talk/turn |
