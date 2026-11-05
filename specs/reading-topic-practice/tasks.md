# Tasks: Reading — import ngân hàng + Luyện tập theo chủ đề

> **Cho Developer Agent:** implement theo thứ tự Task 1 → 6. Mỗi Task test được độc lập, commit riêng.
> Backend là `exe-api`; FE là `exe-web` (PR riêng repo đó — xem Task 6).

**Design:** `specs/reading-topic-practice/design.md`
**Lưu ý chung:** mọi lệnh jest chạy trong `exe-api/services/api` (`npm test` = `jest --runInBand`). Scout trước Task 3: chốt module `reading` mới vs mở rộng `test-paper`, và attempt model (design §4).

---

## File Structure (dự kiến, chốt sau scout)

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/scripts/seed-reading-bank.js` | Runner ETL: đọc JSON → upsert bank | Tạo |
| `services/api/src/modules/reading/reading-bank.mapper.js` | Hàm map thuần JSON→model (test được, no DB) | Tạo |
| `services/api/src/modules/reading/reading.service.js` | list topics/passages, detail, submit+grade | Tạo |
| `services/api/src/modules/reading/reading.controller.js` / `.routes.js` | Endpoint learner | Tạo |
| `services/api/src/modules/reading/reading-attempt.model.js` | Attempt tối giản (nếu không tái dùng `PracticeAttempt`) | Tạo (có điều kiện) |
| `services/api/src/modules/reading/__tests__/*.test.js` | Unit/integration | Tạo |

---

## Task 1: Mapper thuần JSON → model (không DB)

**Files:** Create `reading-bank.mapper.js`; Test `__tests__/reading-bank.mapper.test.js`

- [ ] **Step 1 — test FAIL:** viết test cho `mapPassage()` + `mapItem()` với fixture rút gọn từ `reading-bank-v1.json`: kiểm `cefr B2 → difficultyTier 'mid' + irt.b 0`; `type 'ynng' → itemType 'yes_no_ng'`; `type 'match_info' → 'matching_information'`; topic `coffee → 'health'`; choice single → answerKey string id; fill 1 blank → string; matching → array pairs. Kiểm itemType/topic lạ → throw/log (không nuốt).
- [ ] **Step 2 — Run:** `npx jest reading-bank.mapper --runInBand` → FAIL (chưa có hàm).
- [ ] **Step 3 — Implement:** viết map thuần theo design §2 (itemType map, topic coarse map, answerKey theo payload.kind, irt.b theo tier). Không đụng DB.
- [ ] **Step 4 — Run:** cùng lệnh → PASS.

**Verify:** `npx jest reading-bank.mapper --runInBand` xanh.

## Task 2: Seed script idempotent nạp bank

**Files:** Create `scripts/seed-reading-bank.js`

- [ ] **Step 1:** script load JSON, gọi mapper, upsert Stimulus theo `provenance.sourceRef=passage_id`, upsert Question theo `(importBatch, stimulusId, order)`; in summary (tạo mới/bỏ qua/cảnh báo).
- [ ] **Step 2 — Run lần 1:** `node scripts/seed-reading-bank.js` → tạo 17 Stimulus + 118 Question `status:'draft'`.
- [ ] **Step 3 — Run lần 2:** chạy lại → 0 doc mới (idempotent), summary báo "skipped".
- [ ] **Step 4 — verify DB:** query `AssessmentStimulus.countDocuments({'provenance.importBatch':'reading-bank-v1'})`=17; `AssessmentQuestion...`=118; mọi Question có `stimulusId`, `targetGoal:'ielts'`, `skill:'reading'`.

**Verify:** counts đúng + chạy 2 lần không nhân đôi.

## Task 3: API list chủ đề + list passage (ẩn đáp án)

**Files:** `reading.service.js`, `reading.controller.js`, `reading.routes.js`; Test `__tests__/reading-list.test.js`

- [ ] **Step 1 — test FAIL:** seed vài stimulus `active`; test `GET /reading/topics` trả nhóm + count; `GET /reading/passages?topic=health` chỉ trả bài health `active`; response **không** chứa `answerKey`.
- [ ] **Step 2 — Run:** FAIL.
- [ ] **Step 3 — Implement:** service query `AssessmentStimulus{skill:'reading',status:'active'}` group theo `topic` (lấy topic từ Question hoặc lưu ở stimulus khi seed — chốt ở scout); list lọc `topic`+`cefrBandHint`. Controller `asyncHandler`, `toDto`.
- [ ] **Step 4 — Run:** PASS.

**Verify:** `npx jest reading-list --runInBand` xanh; grep response không có `answerKey`.

## Task 4: API detail + submit (chấm bằng grade.js)

**Files:** `reading.service.js` (+attempt model nếu cần); Test `__tests__/reading-submit.test.js`

- [ ] **Step 1 — test FAIL:** `GET /reading/passages/:id` trả `body`+items không kèm `answerKey`/`explanation`; `POST .../submit` với đáp án đúng → score đầy đủ + explanation hiện; đáp án sai → chấm sai đúng. Test cả 3 kind (choice/fill/matching).
- [ ] **Step 2 — Run:** FAIL.
- [ ] **Step 3 — Implement:** detail `.select('-answerKey -explanation')`; submit load key có `select('+answerKey')`, chấm bằng `assessment/grade.js`, lưu attempt, trả score+đáp án+explanation.
- [ ] **Step 4 — Run:** PASS.

**Verify:** `npx jest reading-submit --runInBand` xanh; chấm đúng cả 3 kind; không leak key trước submit.

## Task 5: Regression + full suite

- [ ] **Run:** `npm test` trong `exe-api/services/api`.

**Verify:** full suite xanh, 0 regression assessment/test-paper.

## Task 6: FE Luyện tập Reading (exe-web — PR riêng)

> Repo `exe-web` chưa có tooling spec này; mô tả ở mức đủ để dev FE làm PR riêng.

- [ ] Trang list chủ đề (gọi `GET /reading/topics`).
- [ ] Trang list bài trong 1 chủ đề, chip lọc CEFR (`GET /reading/passages?topic=&cefr=`), chọn bài tự do.
- [ ] Trang làm bài: render 3 kiểu (choice/fill/matching) từ `passages/:id`.
- [ ] Màn kết quả: score + đáp án + giải thích từ `submit`.

**Verify:** click chủ đề → chọn bài bất kỳ → làm → nộp → thấy điểm+giải thích; không thấy đáp án trước khi nộp.
