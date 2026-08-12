# Contract: Review API (`/api/review`) — api ↔ web

- **Spec:** `specs/mistake-review-ui/spec.md` · **Design:** `../design.md`
- Envelope: `{ <key>: ... }` (khớp modules/quiz). Auth: `verifyToken` (Bearer). Mọi resource scope theo user trong token.

## GET `/api/review/items?status=active|graduated`

Sổ lỗi sai của user. `status` tuỳ chọn (bỏ trống = tất cả).

**200**
```json
{ "items": [
  { "questionId": "…", "stem": "…", "skill": "reading", "subskill": "inference",
    "status": "active", "box": 2, "nextReviewAt": "2026-08-20T00:00:00.000Z", "lastResult": "incorrect" }
] }
```
`answerKey` KHÔNG bao giờ có trong payload.

## GET `/api/review/session?limit=20`

Bộ câu cho 1 phiên ôn (Tier1 câu sai đến hạn → Tier2 câu đã làm). Chỉ item đứng-một-mình (Phase 2 hoãn testlet).

**200**
```json
{ "items": [
  { "questionId": "…", "stem": "…", "options": [{ "index": 0, "text": "…" }],
    "itemType": "mcq", "skill": "reading", "tier": 1 }
] }
```

## POST `/api/review/answer`

Nộp 1 câu. Server chấm (KHÔNG nhận `correct` từ client — bỏ qua nếu gửi).

**Body**
```json
{ "questionId": "…", "response": <tuỳ itemType: number | number[] | string | string[]> }
```

**200**
```json
{ "result": { "correct": true, "status": "active", "nextReviewAt": "2026-08-20T00:00:00.000Z" } }
```

**Lỗi**
| HTTP | code | khi |
|---|---|---|
| 404 | `REVIEW_ITEM_NOT_FOUND` | câu không có trong két của user |
| 400 | `INVALID_REVIEW_RESULT` | `response` thiếu/sai shape |
