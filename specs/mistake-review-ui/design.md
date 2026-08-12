# Design: Két lỗi sai Phase 2 — REST + UI "Sổ lỗi sai"

- **Spec:** `specs/mistake-review-ui/spec.md`
- **Ngày:** 2026-08-13
- **Tác giả:** Tech Lead Agent
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** không có

## 1. Tóm tắt kiến trúc

Phơi module `review` (Phase 1) ra REST dưới `/api/review` (controller + routes mỏng, delegate service). Thêm 3 hàm service mức "phơi client": list sổ lỗi sai, serve bộ câu ôn (ẩn đáp án), nộp-và-chấm 1 câu. **Chấm ở server** bằng cách tái dùng `quiz.grading.gradeObjective` + đường serve item ẩn đáp án của quiz — `recordReview` giữ nguyên (chỉ được gọi sau khi server tự suy `correct`). exe-web thêm `review.service.ts` + `use-review.ts` + trang "Sổ lỗi sai" (tái dùng player quiz). exe-admin không đụng.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Chấm phiên ôn | Server: `response` → `gradeObjective` → `correct` → `recordReview` | Bảo mật — client tự khai đúng/sai sẽ phá SR | REST cần fetch answerKey server-side |
| Tái dùng chấm/serve item | Dùng `quiz.grading` + select ẩn `answerKey` của quiz | DRY, không lệch logic chấm | Coupling review→quiz grading (chấp nhận) |
| Giữ `recordReview` | Không đổi chữ ký (`correct:boolean`) | Phase 1 đã test; REST bọc ngoài | 2 lớp (grade wrapper + record) |
| Mount REST | Router mới `/api/review` (không nhét vào `/api/courses`) | Review không thuộc 1 course cụ thể | Thêm 1 mount trong app.js |
| Phạm vi item | Chỉ item đứng-một-mình (stimulusId=null) ở Phase 2 | Tránh serve/render testlet phức tạp | Câu testlet-sai chưa ôn được (Phase sau) |

## 3. Data model

**Không collection mới.** Dùng `review_schedule` (Phase 1) + đọc `AssessmentQuestion` (stem/options/itemType/skill, ẩn `answerKey`). Bổ sung 1 index đọc danh sách nếu cần: `{userId:1, status:1, skill:1}` (cân nhắc — có thể đủ với index Phase 1).

## 4. Luồng dữ liệu

**A. GET /api/review/items** — sổ lỗi sai:
```
verifyToken -> controller -> review.service.listReviewItems(userId, {status?})
  -> ReviewSchedule.find({userId, [status]})   // active/graduated
  -> join AssessmentQuestion (stem, skill, subskill; KHÔNG answerKey)
  -> res { items: [{questionId, stem, skill, subskill, status, box, nextReviewAt, lastResult}] }
```

**B. GET /api/review/session?limit=** — bộ câu ôn:
```
verifyToken -> review.service.getReviewSessionItems(userId, limit)
  -> queue = getReviewQueue(userId, limit)              // Phase 1 (tự nạp câu sai)
  -> lọc item đứng-một-mình; resolve AssessmentQuestion .select('-answerKey -acceptedVariants')
  -> res { items: [{questionId, stem, options, itemType, skill}] }   // ẩn đáp án
```

**C. POST /api/review/answer** `{questionId, response}` — nộp + chấm:
```
verifyToken -> review.service.submitReviewAnswer(userId, questionId, response)
  -> q = AssessmentQuestion.findById(questionId).select('+answerKey +acceptedVariants itemType')
  -> correct = gradeObjective(q, response)             // tái dùng quiz.grading
  -> out = recordReview(userId, questionId, correct)   // Phase 1
  -> res { result: { correct, status: out.status, nextReviewAt: out.nextReviewAt } }
```

## 5. Contracts

Cross-repo (api ↔ web) — viết `contracts/review-api.md` (request/response 3 endpoint trên, envelope `{key:...}`, lỗi). REST client-facing → cần contract rõ cho FE.

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `exe-api/.../modules/review/review.controller.js` | 3 handler thin | Tạo |
| `exe-api/.../modules/review/review.routes.js` | Mount `/api/review`, verifyToken | Tạo |
| `exe-api/.../modules/review/review.service.js` | +`listReviewItems/getReviewSessionItems/submitReviewAnswer` | Sửa |
| `exe-api/.../src/app.js` | Mount reviewRoutes | Sửa |
| `exe-api/.../modules/review/__tests__/review-api.test.js` | API test | Tạo |
| `exe-web/src/services/review.service.ts` | Gọi 3 endpoint | Tạo |
| `exe-web/src/hooks/use-review.ts` | React Query hooks | Tạo |
| `exe-web/src/app/.../review/` (trang Sổ lỗi sai) | UI list + phiên ôn | Tạo |
| `exe-web/src/components/review/*` | ReviewList, ReviewSessionPlayer (tái dùng player quiz) | Tạo |

## 7. Xử lý lỗi

| Tình huống | HTTP | error code |
|---|---|---|
| answer câu không có trong két user | 404 | `REVIEW_ITEM_NOT_FOUND` (Phase 1) |
| `response` thiếu/sai shape | 400 | `INVALID_REVIEW_RESULT` hoặc code chấm hiện có |
| question đã retired/không còn active | 409/404 | tái dùng code quiz phù hợp |

## 8. Bảo mật & quyền

- `verifyToken`; mọi handler dùng `req.user.id`, **không** nhận userId từ body.
- `answerKey`/`acceptedVariants`/`transcript` **không bao giờ** ship cho client (select ẩn ở đường serve; chỉ +answerKey server-side lúc chấm).
- Client gửi `correct` → **bỏ qua tuyệt đối** (server tự chấm).
- Không permission mới (learner-owned resource).

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Testlet trong ôn:** Phase 2 loại item có `stimulusId` khỏi phiên ôn → câu đọc/nghe-sai chưa ôn được. Ghi rõ giới hạn cho học viên hoặc để phase sau. (Câu hỏi mở product.)
- **Item đã retired sau khi vào két:** khi ôn, question có thể đã retired → cần bỏ qua/loại khỏi két. Xử lý: serve bỏ qua, `listReviewItems` đánh dấu hoặc ẩn.
- **Vị trí điều hướng:** cần product chỉ định route/nav — chưa chốt.

## 10. Testing strategy

- API test (supertest + memory-server): items ẩn đáp án; session trả câu đến hạn; answer đúng→giãn/sai→sớm; client gửi `correct` bị bỏ qua (server vẫn chấm từ response); 404 câu ngoài két.
- exe-web: unit test hook/service (mock API); render list; phiên ôn gọi đúng endpoint.

## 11. Rollout / cross-repo sequencing

1. exe-api PR trước (REST) — deploy, có endpoint.
2. exe-web PR sau (UI) — gọi endpoint đã live.
(2 PR ở 2 repo, theo `.ai/project-context.md` §0.)

## 12. Đối chiếu Acceptance criteria

| Acceptance (spec §6) | Đáp ứng bởi |
|---|---|
| 1. List ẩn đáp án | §4A |
| 2. Session câu đến hạn ẩn đáp án | §4B |
| 3. Nộp→chấm→SR cập nhật | §4C |
| 4. Không tin client correct | §2 + §8 |
| 5. UI list + phiên ôn | §6 exe-web |
| 6. Chỉ dữ liệu của user | §8 verifyToken |
