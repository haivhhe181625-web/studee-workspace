---
stream: 2
feature: flashcard-guided-learning-flow
repo: exe-api
endpoint: PATCH new-limit setting
status: todo
---

# Stream 2 — new-limit setting

**Mục tiêu:** đọc/ghi số từ mới/ngày per-user (mặc định 10, hợp lệ 5/10/20).

## Đọc trước
- `User` model (`modules/user/user.model.js:141`) — `profile.studyPlan` là **StudyPlanSchema strict, `default: null`**, **chưa có writer nào**.
- `modules/user/user.routes.js:26` — `PATCH /me` (`updateMe`) + `schemas.updateMe` (whitelist). **Chốt** mở rộng endpoint này, KHÔNG route flashcard riêng (design §1).
- `design.md §1, §2`.

## Tasks (TDD)
- [ ] Test: PATCH `/me` đặt `profile.studyPlan.flashcardNewPerDay=20` → đọc lại =20; ngoài {5,10,20} → 400 (Joi); user chưa có studyPlan (null) → ghi khởi tạo object OK; mặc định khi chưa set = 10 (đọc ở stream 1).
- [ ] Thêm field `flashcardNewPerDay` vào **`StudyPlanSchema`** (Mongoose strict drop field lạ nếu không khai báo). Không cần default field — stream 1 fallback `?? 10`.
- [ ] Mở rộng `schemas.updateMe` (whitelist `profile.studyPlan.flashcardNewPerDay`) + controller: ghi bằng path `$set: {'profile.studyPlan.flashcardNewPerDay': v}` (an toàn khi studyPlan đang null).
- [ ] Validate Joi `valid(5,10,20)`.

## Acceptance
- [ ] Ghi/đọc đúng qua `PATCH /users/me`; user studyPlan=null vẫn set được; validate chặt {5,10,20}; userId từ JWT.

**Status khi xong:** DONE + tóm tắt.
