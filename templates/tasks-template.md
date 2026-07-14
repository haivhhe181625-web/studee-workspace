<!--
  tasks-template.md — dùng cho .ai/prompts/generate-tasks.md (Tech Lead Agent, phase Task Generation).
  Copy thành specs/<feature-slug>/tasks.md. Giữ đúng phong cách checkbox TDD của
  <API_REPO>/docs/superpowers/plans/2026-06-15-user-profile-edit-avatar.md — mỗi step có Run: + Expected: cụ thể,
  không có bước mơ hồ kiểu "implement the feature".
-->
# Tasks: <Tên feature>

> **Cho Developer Agent:** implement theo đúng thứ tự Task 1 → N. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task.

**Design:** `specs/<feature-slug>/design.md`
**Lưu ý chung khi chạy lệnh:** <vd: mọi lệnh npm/jest chạy trong `<API_REPO>/services/api`>

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `<path>` | <mô tả> | Tạo/Sửa |

---

## Task 1: <Tên task>

**Files:**
- Create/Modify: `<path>`
- Test: `<path test>`

- [ ] **Step 1: Viết test thất bại**

Tạo/sửa `<test file>`:

```<lang>
<test code>
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
<lệnh chạy test>
```
Expected: FAIL — <lý do fail cụ thể>.

- [ ] **Step 3: Implement**

<Mô tả thay đổi cụ thể, kèm code nếu cần.>

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
<lệnh chạy test>
```
Expected: <N> test PASS.

- [ ] **Step 5: Commit**

```bash
git add <files cụ thể — KHÔNG git add .>
git commit -m "<type>(<scope>): <mô tả>"
```

---

## Task 2: <...>

<Lặp lại cấu trúc Task 1.>

---

## Task N: Chạy full suite + dọn dẹp

**Files:** (không sửa code) — verify + tài liệu

- [ ] **Step 1: Chạy toàn bộ test liên quan**

Run:
```bash
<lệnh>
```
Expected: ALL PASS.

- [ ] **Step 2: Cập nhật tài liệu nếu cần** (chỉ trong phạm vi feature này)

---

## Self-Review

<Điền sau khi hoàn tất — đối chiếu lại từng mục trong acceptance.md, đánh dấu ✅/❌.>

**Acceptance coverage:**
- <Tiêu chí 1> → Task <N>. ✅/❌

**Placeholder scan:** <Không có TBD/TODO còn sót — xác nhận.>

**Type/name consistency:** <Tên hàm/biến dùng ở Task này có khớp với Task khác không.>
