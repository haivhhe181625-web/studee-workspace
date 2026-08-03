<!-- Tiếng Việt — hợp đồng api↔web cho learner tiêu thụ khóa (Giai đoạn B). exe-web (Giai đoạn C) code theo file này.
     Khác admin-api: envelope learner là { <key>: ... } (không bọc { data }), khớp modules/ipa. -->
# Hợp đồng API — Learner tiêu thụ khóa (api↔web)

Base: `/api/courses` (mount trong `app.js`, module `course-content`). Auth: **`verifyToken`** (đăng nhập bất kỳ —
Q-B2). **Chỉ khóa `status: 'published'`** (Q-B1 xem tự do, chưa enroll). Khóa chưa publish ⇒ 404 (không lộ tồn tại).

---

## §B1 — `GET /api/courses` — Catalog khóa đã xuất bản

- Query tuỳ chọn **`?program=`** (`general|ielts|toeic`, Phase 1.5): lọc theo chương trình. `general` gồm cả khóa
  cũ chưa có field. Không truyền ⇒ mọi khóa published.
- **200**
```jsonc
{
  "courses": [
    { "id": "…", "slug": "khoa-a2", "title": "Khóa A2", "description": "…", "thumbnail": "https://… | null",
      "program": "ielts", "targetScore": 6.5, "phaseCount": 3 }
  ]
}
```
- Sắp xếp `createdAt` giảm dần. Không trả cả cây (nhẹ). `phaseCount` = số Chặng. `program` mặc định `general`
  (khóa tiền-1.5); `targetScore` = null nếu không đặt.
- **401** thiếu token.

---

## §B2 — `GET /api/courses/:slug` — Chi tiết 1 khóa (cả cây)

- **200**
```jsonc
{
  "course": {
    "id": "…", "slug": "khoa-a2", "title": "Khóa A2", "description": "…", "thumbnail": "…", "status": "published",
    "program": "ielts", "targetScore": 6.5,
    "phases": [{
      "key": "p1", "title": "Chặng 1", "order": 1, "cefrFrom": "A2", "cefrTo": "B1", "goalNote": "…",
      "scoreBand": { "min": 5.5, "max": 6.5, "unit": "band", "estimate": "5.5–6.5", "disclaimer": "…" },
      "modules": [{
        "key": "m1", "title": "Ngữ pháp", "order": 1, "category": "grammar",
        "lessons": [{
          "key": "l1", "title": "Bài 1", "order": 1, "theory": "# …",
          "media": [{ "type": "video", "url": "https://…", "label": null, "order": 1 }],
          "exercises": [{ "type": "ipa", "refId": "L1", "label": null, "order": 1 }]
        }]
      }]
    }]
  }
}
```
- **Ẩn field nội bộ**: `importedBy`, `sourceMeta`, `centerId`, `__v` KHÔNG trả.
- **`program`** (mặc định `general`) + **`targetScore`** (null nếu không đặt). Mỗi Chặng có **`scoreBand`** (Phase 1.5,
  Q4): khoảng điểm gốc suy từ `cefrTo`(fallback `cefrFrom`) theo `cefr-mapping.js` — `null` khi program `general`
  hoặc CEFR không map. `scoreBand.disclaimer` = miễn trừ "điểm ước tính nội bộ" (phải hiển thị kèm).
- **404** slug không tồn tại **hoặc** khóa chưa `published`. **401** thiếu token.

> **Bài tập**: `exercises[].refId` trỏ tới engine đã có — `ipa` (theo IpaLesson **`code`**), `talk` (scenario id ∈
> `coffee|interview|travel|freetalk`). exe-web mở bằng player ipa/talk sẵn có (Giai đoạn C). Media là URL trực tiếp.

---

## §B3 — `GET /api/ipa/lessons/by-code/:code` — Resolve code→id cho launch ipa (bổ sung Giai đoạn C)

**Vì sao:** `exercises[].refId` (ipa) là IpaLesson **`code`** (vd `"L1"`), nhưng runner phát âm exe-web chạy theo
Mongo **`_id`** (`GET /api/ipa/lessons/:id` dùng `findOne({_id})`; route `/pronunciation/:id`). Cần 1 bước dịch.

- **Auth:** `verifyToken`. Module `ipa` (KHÔNG phải course-content) — đây là endpoint đọc của chính engine ipa.
- **200** `{ "id": "665f…" }` — `_id` của IpaLesson `published` có `code` tương ứng.
- **404** code không tồn tại **hoặc** chưa `published` (khớp guard runner: chỉ mở bài đã publish).
- **401** thiếu token.

> Web: click bài tập ipa → `GET by-code/:refId` → `router.push('/pronunciation/:id')`. Giai đoạn B **không đổi**
> (endpoint cộng thêm ở module ipa). Talk không cần resolve — `refId` = scenario id, mount `ConversationView` trực tiếp.
