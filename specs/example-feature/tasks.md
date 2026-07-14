<!-- VÍ DỤ MINH HOẠ — xem ghi chú ở spec.md. Rút gọn (2 task) chỉ để minh hoạ hình dạng, không đầy đủ như
     một tasks.md thật (xem <API_REPO>/docs/superpowers/plans/2026-06-15-user-profile-edit-avatar.md để thấy bản đầy đủ). -->
# Tasks: [VÍ DỤ] Nhắc học qua email hằng ngày

**Design:** `specs/example-feature/design.md`
**Lưu ý chung khi chạy lệnh:** mọi lệnh chạy trong `<API_REPO>/services/api`.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `src/modules/user/user.schema.js` | Thêm `reminderEnabled` vào `updateMe` | Sửa |
| `src/modules/reminder/reminder.model.js` | `ReminderLog` schema | Tạo |
| `src/modules/reminder/reminder.job.js` | Job logic | Tạo |

---

## Task 1: Thêm `reminderEnabled` vào `PATCH /api/users/me`

**Files:**
- Modify: `src/modules/user/user.schema.js`
- Test: `src/__tests__/user.profile.api.test.js`

- [ ] **Step 1: Viết test thất bại** — thêm case `reminderEnabled: true` vào `describe('PATCH /api/users/me')`.
- [ ] **Step 2: Run `npx jest user.profile.api -i`** — Expected: FAIL, field bị Joi strip.
- [ ] **Step 3: Thêm `reminderEnabled: Joi.boolean()` vào schema `updateMe`.**
- [ ] **Step 4: Run lại — Expected: PASS.**
- [ ] **Step 5: Commit** `git add src/modules/user/user.schema.js src/__tests__/user.profile.api.test.js`.

---

## Task 2: Job nhắc học (BullMQ)

**Files:**
- Create: `src/modules/reminder/reminder.model.js`, `src/modules/reminder/reminder.job.js`
- Test: `src/__tests__/reminder.job.test.js`

- [ ] **Step 1: Viết test thất bại** cho case AC-1/AC-2/AC-3/AC-E1 (mock adapter email + `ReminderLog`).
- [ ] **Step 2: Run test** — Expected: FAIL, `Cannot find module 'reminder.job'`.
- [ ] **Step 3: Implement `reminder.model.js` + `reminder.job.js`** theo luồng ở design.md §4.
- [ ] **Step 4: Run test** — Expected: 4 test PASS (AC-1..AC-E1).
- [ ] **Step 5: Commit.**

---

## Self-Review

**Acceptance coverage:** AC-1..AC-3, AC-E1 → Task 2. ✅ (Task 1 chỉ là tiền đề cho toggle.)
**Placeholder scan:** Không có TBD (ví dụ này rút gọn có chủ đích, không phải TBD).
**Type/name consistency:** `reminderEnabled` khớp Task 1 và Task 2.
