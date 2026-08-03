<!-- Tiếng Việt — hợp đồng API admin cho Giai đoạn A (quản lý/sửa/xuất bản khóa). Nối tiếp
     contracts/admin-course-import.md của Phase 1 (§1–§5 đã có). Base path: /api/admin/course-imports -->
# Hợp đồng API — Quản lý khóa (Giai đoạn A)

Base: `/api/admin/course-imports` (giữ nguyên router Phase 1 `course-content.admin.js`). Tất cả cần
`verifyToken`. Thân lỗi chuẩn: `{ message, code, errors? }`.

**Đã có ở Phase 1 (tái dùng, không đổi):**
- `GET /:id` — chi tiết cả cây (`coursecontent:read`). → dùng cho trang xem/sửa.
- `DELETE /:id` — xoá cứng, chặn `published` (`coursecontent:delete`).

**Bổ sung ở Giai đoạn A:**

---

## §6 — `PUT /:id` — Sửa nội dung khóa (ghi lại cả cây + re-validate)

Sửa **cả cây** trong một lần ghi atomic (Q-A2). Chỉ khóa `draft`/`ready` mới sửa được (Q-A1).

- **Quyền:** `coursecontent:write`
- **Body** (JSON — đúng shape cây, **không** có field server-side; `order` suy từ thứ tự mảng):
```jsonc
{
  "title": "Khóa A2",
  "description": "…",
  "thumbnail": "https://cdn/cover.png",   // hoặc null
  "phases": [{
    "key": "p1", "title": "Chặng 1", "cefrFrom": "A2", "cefrTo": "B1", "goalNote": "IELTS 5.0→6.0",
    "modules": [{
      "key": "m1", "title": "Ngữ pháp", "category": "grammar",
      "lessons": [{
        "key": "l1", "title": "Bài 1", "theory": "# …",
        "media": [{ "type": "video", "url": "https://…", "label": null }],
        "exercises": [{ "type": "ipa", "refId": "L1", "label": null }]
      }]
    }]
  }]
}
```
- **Bất biến:** `slug`, `status`, `importedBy`, `createdAt` **không đổi** (slug là định danh; đổi slug = tạo khóa mới).
- **Xử lý:** attach path-locator → `validateCourse` + `resolveReferences` (y hệt import) → nếu sạch thì ghi đè
  `title/description/thumbnail/phases`, cập nhật `updatedAt`.
- **200** `{ data: { summary } }` — `summary` như import (id/slug/title/status/counts).
- **400** `IMPORT_VALIDATION_FAILED` + `errors[]` mỗi phần tử `{ code, message, path }` (**`path` thay `row`** — vd
  `"Chặng \"Chặng 1\" › Chuyên đề \"Ngữ pháp\" › Bài \"Bài 1\""`).
- **409** `COURSE_LOCKED` — khóa đang `published`/`archived` (phải unpublish trước).
- **404** không tồn tại. **403** thiếu quyền.

---

## §7 — `PATCH /:id/status` — Chuyển vòng đời

- **Quyền:** `coursecontent:write`
- **Body:** `{ "status": "published" }` (một trong `draft|ready|published|archived`).
- **Máy trạng thái hợp lệ (Q-A3):**

| Từ \ Đến | draft | ready | published | archived |
|---|---|---|---|---|
| **draft** | — | ✓* | ✗ | ✓ |
| **ready** | ✓ | — | ✓* | ✓ |
| **published** | ✗ | ✓ (gỡ publish) | — | ✓ |
| **archived** | ✗ | ✓ (khôi phục)* | ✗ | — |

`*` = **bắt buộc re-validate** trước khi chuyển (đích ∈ {`ready`,`published`}): chạy `validateCourse` +
`resolveReferences` trên cây đang lưu; có lỗi ⇒ chặn.

- **200** `{ data: { id, status } }`.
- **400** `IMPORT_VALIDATION_FAILED` + `errors[]` (`path`) — khi re-validate thất bại (vd IPA ref bị unpublish sau khi import).
- **409** `INVALID_STATUS_TRANSITION` — chuyển không nằm trong bảng.
- **404 / 403** như trên.

> **Audit:** cả §6 và §7 emit `auditLog('admin.command.executed', { command: 'coursecontent.update' | 'coursecontent.status', target: <slug|id>, ... })`.
