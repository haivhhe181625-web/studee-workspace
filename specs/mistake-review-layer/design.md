# Design: Két lỗi sai — Mistake tracking & smart review layer

- **Spec:** `specs/mistake-review-layer/spec.md`
- **Ngày:** 2026-08-13
- **Tác giả:** Tech Lead Agent
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** không có (không thêm dependency mới, không đổi kiến trúc cross-service)

## 1. Tóm tắt kiến trúc

Module mới `modules/review/` (5-file pattern, Phase 1 chỉ cần model + service + scheduler). **Hybrid**: dữ liệu "câu nào sai" **derive** từ nguồn có sẵn (`learning_events`, `quiz_attempts`) — không chép lại; chỉ **materialize** trạng thái giãn ngắt quãng (thứ không derive được) vào collection mỏng `review_schedule`. SR **tự viết Leitner** (interval cố định `[1,3,7,14,30]`) trong module — tách khỏi flashcard để product chỉnh interval trực tiếp. Không đụng `exe-web`/`exe-admin`, không đụng đường submit quiz. M4-9 (`lesson-schedule.generator.js`) tiêu thụ service này ở task riêng của nó.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Lưu trạng thái lỗi sai | Hybrid: derive câu sai + materialize chỉ lịch SR | DRY (1 nguồn đúng/sai), vẫn có giãn cách thật, bề mặt ghi nhỏ | Query hàng đợi phải ghép 2 nguồn |
| Nguồn "câu đang sai" | `learning_events` (`source:'quiz', correct:false`) | Submit đã dedup về latest (`quiz.service.js:147`) → correct:false = đang sai; đã có skill/subskill | Chỉ phủ item có `skill` (điều kiện enum) |
| SR engine | **Tự viết Leitner** `[1,3,7,14,30]` ngày, trong `review/review-scheduler.js` | Interval minh bạch, product chỉnh trực tiếp; state tối thiểu `{box,nextReviewAt}`; không coupling flashcard | Tự lo test SR; thô hơn FSRS thích ứng (đủ tốt, đúng "SM-2 rút gọn" M4-9) |
| Luật graduate | `box > INTERVALS.length` (qua hết bậc 30 ngày → mastered) | Đơn giản, suy từ box | Cần vài lần đúng liên tiếp mới rời hàng đợi |
| Tier 2 pool (Phase 1) | Câu đã từng làm (`quiz_attempts`) và không đang sai | Không cần kéo course-progress vào; đủ tốt cho "duy trì" | Không đúng chính xác "lesson đã completed" — refine sau |
| Giới hạn hàng đợi | Tham số `limit` (mặc định 20), service cắt tổng ≤ limit | Linh hoạt; M4-9 tự quyết độ dài buổi ôn | Không có hard-cap chống lạm dụng (chấp nhận Phase 1) |
| Nạp câu sai | `getReviewQueue` tự gọi `enqueueMistakes` (idempotent) | 1 điểm vào cho caller; luôn có dữ liệu mới nhất | Hàm "get" có side-effect ghi (rẻ, idempotent) |
| Rewire generator | KHÔNG (để M4-9) | Giữ Phase 1 = tầng dữ liệu thuần | M4-9 phải làm bước tích hợp |

## 3. Data model

Collection mới: **`review_schedule`** — 1 row / `(userId, questionId)` **chỉ khi câu vào hàng đợi ôn**.

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `userId` | ObjectId ref User | ✅ | index |
| `questionId` | ObjectId ref AssessmentQuestion | ✅ | |
| `skill` | String (enum SKILLS) | ✅ | copy từ event để query/lọc |
| `subskill` | String (enum SUBSKILLS) \| null | | copy từ event |
| `origin` | String enum `['mistake','manual']` | ✅ default `mistake` | đường vào hàng đợi (manual = M4-10 remediation sau) |
| `status` | String enum `['active','graduated']` | ✅ default `active` | graduated = rời Tier 1 |
| `lastResult` | String enum `['correct','incorrect']` \| null | | kết quả ôn gần nhất |
| `srs.box` | Number | ✅ default 0 | bậc Leitner = số lần đúng liên tiếp (0..5) |
| `srs.nextReviewAt` | Date \| null | ✅ | đến hạn ôn; null khi graduated |
| `srs.lastReviewedAt` | Date \| null | | lần ôn gần nhất |
| timestamps | | ✅ | |

**Index:**
- `{userId: 1, questionId: 1}` **unique** — upsert key, chống trùng (Acceptance #5).
- `{userId: 1, status: 1, 'srs.nextReviewAt': 1}` — query Tier 1 "câu active đến hạn, sort theo nextReviewAt".

Data model đủ đơn giản để nằm trong design này — **không** tách `data-model.md`.

## 3b. Thuật toán Leitner (`review-scheduler.js`, pure)

```
INTERVALS = [1, 3, 7, 14, 30]   // ngày

initState(now)            -> { box: 0, nextReviewAt: now, lastReviewedAt: null }   // due ngay
applyResult(state, correct, now):
    box = correct ? state.box + 1 : 0            // đúng: lên bậc; sai: về 0
    if box > INTERVALS.length:                   // qua hết bậc → mastered
        return { box, nextReviewAt: null, lastReviewedAt: now }
    waitDays = box === 0 ? 0 : INTERVALS[box-1]  // sai → due ngay; đúng bậc k → INTERVALS[k-1]
    return { box, nextReviewAt: now + waitDays, lastReviewedAt: now }
isGraduated(state)        -> state.box > INTERVALS.length
```

Lịch giãn khi đúng liên tiếp: 1 → 3 → 7 → 14 → 30 ngày, rồi graduate. Sai bất kỳ lúc nào → về bậc 0, due ngay.

## 4. Luồng dữ liệu

**A. Nạp câu sai (enqueueMistakes)** — idempotent:
```
caller --> review.service.enqueueMistakes(userId)
  -> LearningEvent.find({userId, source:'quiz', correct:false})   // câu đang sai (đã dedup latest)
  -> với mỗi (questionId, skill, subskill) unique:
       ReviewSchedule.updateOne({userId, questionId},
         { $setOnInsert: { skill, subskill, origin:'mistake', status:'active', srs: scheduler.initState() } },
         { upsert: true })                                        // không ghi đè row đã có
  -> return { enqueued: <số row thực sự insert> }
```

**B. Ghi kết quả ôn (recordReview)**:
```
caller --> review.service.recordReview(userId, questionId, correct)
  -> if typeof correct !== 'boolean' -> throw ApiError(400,'INVALID_REVIEW_RESULT')
  -> doc = review_schedule.findOne({userId, questionId})   // null -> ApiError(404,'REVIEW_ITEM_NOT_FOUND')
  -> next = scheduler.applyResult(doc.srs, correct)
  -> doc.srs = next
     doc.lastResult = correct ? 'correct' : 'incorrect'
     doc.status = scheduler.isGraduated(next) ? 'graduated' : 'active'
  -> save; return { nextReviewAt: next.nextReviewAt, status: doc.status }
```

**C. Lấy hàng đợi ôn (getReviewQueue)** — tự nạp câu sai trước:
```
caller (M4-9) --> review.service.getReviewQueue(userId, limit = 20)
  -> await enqueueMistakes(userId)                              // quyết định ④: 1 điểm vào
  -> tier1 = review_schedule.find({userId, status:'active', 'srs.nextReviewAt': {$lte: now}})
             .sort({'srs.nextReviewAt': 1}).limit(limit)        // câu sai đến hạn, quá hạn lâu nhất trước
  -> if tier1.length >= limit: return tier1.map(q => ({questionId, tier:1}))
  -> need = limit - tier1.length
     tier2Pool = QuizAttempt.find({userId}).distinct('perItem.questionId')
                 trừ (questionId ở tier1) trừ (câu đang có row active/đang sai)
     tier2 = lấy tối đa `need` phần tử từ tier2Pool
  -> return [...tier1(tier:1), ...tier2(tier:2)]
```

## 5. Contracts

Phase 1 **không** có contract cross-service/cross-repo (thuần nội bộ api). Interface = **service function** (`enqueueMistakes/recordReview/getReviewQueue`) — hợp đồng nội bộ cho M4-9/M4-10 gọi. Nếu Phase 2 (UI) cần REST → viết `contracts/review-api.md` khi đó.

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/src/modules/review/review-schedule.model.js` | Mongoose model `review_schedule` | Tạo |
| `services/api/src/modules/review/review-scheduler.js` | Leitner thuần: `initState/applyResult/isGraduated/INTERVALS` | Tạo |
| `services/api/src/modules/review/review.service.js` | `enqueueMistakes/recordReview/getReviewQueue` | Tạo |
| `services/api/src/modules/review/__tests__/*.test.js` | Test model + scheduler + service | Tạo |
| `services/api/src/modules/adaptive/learning-event.model.js` | Nguồn câu sai | Đọc (không sửa) |
| `services/api/src/modules/quiz/quiz-attempt.model.js` | Nguồn Tier 2 pool | Đọc (không sửa) |

Controller/routes: **chưa tạo ở Phase 1** (không có REST surface). Thêm khi Phase 2.

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| `recordReview` với `(user,question)` không có trong lịch | 404 | `REVIEW_ITEM_NOT_FOUND` |
| `recordReview` thiếu/không hợp lệ `correct` (không phải boolean) | 400 | `INVALID_REVIEW_RESULT` |

(Service throw `ApiError`; Phase 1 gọi nội bộ → caller xử lý. Phase 2 gắn controller thì dùng `asyncHandler`.)

## 8. Bảo mật & quyền

- Không thêm permission string mới (không có REST surface ở Phase 1).
- `review_schedule` luôn scope theo `userId` — mọi query bắt buộc có `userId`, không nhận từ req.body.
- Không dữ liệu nhạy cảm (chỉ lưu `questionId`, không nội dung/đáp án). Không cần `select:false`.
- Khi Phase 2 mở REST: đọc lịch của chính user (verifyToken), không cross-user.

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Item không có `skill`**: `learning_events` enum bắt buộc `skill` → item không skill không sinh event → không vào két. Chấp nhận (mastery cũng bỏ qua). Ghi nhận, không xử lý thêm.
- **Câu sai đã sửa đúng nhưng row vẫn active**: khi user làm lại quiz đúng (submit thường) → event thành `correct:true`, nhưng `review_schedule` KHÔNG tự cập nhật (chỉ `recordReview` cập nhật). **Thiết kế cố ý**: câu vào két đi theo vòng SR riêng. **M4-9 phải gọi `recordReview` cho buổi ôn**, KHÔNG dùng submit quiz thường.
- **`enqueueMistakes` bên trong `getReviewQueue`**: mỗi lần lấy hàng đợi có 1 lượt ghi upsert — idempotent, rẻ; chấp nhận side-effect trong "get".
- **Không tách research.md**: không có câu hỏi kỹ thuật cần spike — mọi nguồn đã verify bằng đọc code.

## 10. Testing strategy

- **Unit test (pure)** cho `review-scheduler.js`: initState due ngay; đúng→box+1 & nextReviewAt xa hơn; sai→box0 & due ngay; chuỗi đúng → graduate.
- **Integration test** (mongodb-memory-server, helper `src/__tests__/helpers/db.js`): seed `learning_events`/`quiz_attempts` → gọi service → assert `review_schedule` + hàng đợi.
- Case chính: enqueue idempotent; Tier1 sort theo nextReviewAt; Tier2 lấp khi thiếu; recordReview đúng→giãn, sai→sớm; graduate; recordReview 404/400.
- **Regression:** chạy lại test quiz/schedule/flashcard/adaptive cũ — xác nhận không đụng.

## 11. Rollout / cross-repo sequencing

Chỉ `exe-api`, 1 PR. Không cross-repo. Deploy độc lập; M4-9 (PR sau) mới tích hợp vào generator.

## 12. Đối chiếu Acceptance criteria

| Acceptance (spec §6) | Đáp ứng bởi |
|---|---|
| 1. Sai → có row active due ngay | §4A `enqueueMistakes` + `initState` (nextReviewAt = now) |
| 2. Hàng đợi Tier1 trước Tier2, tôn trọng due | §4C `getReviewQueue` |
| 3. Đúng→giãn, sai→sớm lại | §3b `applyResult` + §4B |
| 4. Mastered→graduated, rời Tier1 | §3b `isGraduated` + filter `status:'active'` |
| 5. Enqueue idempotent | §3 unique index + `$setOnInsert` |
| 6. Không hồi quy submit | §6 chỉ đọc nguồn cũ, không sửa; §10 regression |
