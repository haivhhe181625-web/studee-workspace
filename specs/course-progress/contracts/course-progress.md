<!-- Tiếng Việt — hợp đồng api↔web cho tiến độ khóa (Phase 3). Envelope learner { progress } (khớp course-content).
     Quyết định đã chốt: §8 plan-phase3-progress-mastery.md (mọi đề xuất OK). -->
# Hợp đồng API — Tiến độ khóa học (Phase 3)

Base: `/api/courses` (module `course-content`). Auth: **`verifyToken`** (mọi user đăng nhập). Chỉ thao tác trên khóa
`status: 'published'`. Envelope learner: **`{ progress }`** (không bọc `{ data }`).

**Định danh Bài** (data-model): `pathKey = "<phaseKey>/<moduleKey>/<lessonKey>"` (Bài `key` chỉ unique trong Chuyên
đề, node embedded không có `_id`). exe-web tự dựng `pathKey` từ cây `GET /courses/:slug` để overlay tiến độ.

**Gate (Q-P3.6):** completion-gate — chỉ áp dụng **khi đã enroll**. Chưa enroll ⇒ xem tự do (giữ hành vi Phase 2),
không tiến độ. Bài `unlocked` ⇔ là Bài đầu **hoặc** Bài liền trước (theo thứ tự phẳng) đã `completed`/`mastered`.

---

## Shape `progress` (dùng chung cho enroll / get / record)

```jsonc
// Đã enroll:
{
  "progress": {
    "enrolled": true,
    "status": "active",              // 'active' | 'completed'
    "percent": 42,                    // completedLessons/totalLessons, làm tròn
    "completedLessons": 5,
    "totalLessons": 12,
    "phasePercent": { "p1": 100, "p2": 20 },
    "lessons": {
      "p1/m1/l1": { "status": "mastered",   "unlocked": true,  "completedAt": "…" },
      "p1/m1/l2": { "status": "completed",  "unlocked": true,  "completedAt": "…" },
      "p2/m1/l1": { "status": "in_progress","unlocked": true,  "completedAt": null },
      "p2/m1/l2": { "status": "not_started","unlocked": false, "completedAt": null }
    },
    "nextLesson": { "phaseKey": "p2", "moduleKey": "m1", "lessonKey": "l1" } // hoặc null (xong hết)
  }
}
// Chưa enroll:
{ "progress": { "enrolled": false } }
```
- `status` Bài ∈ `not_started | in_progress | completed | mastered`. `not_started` = chưa có bản ghi (không lưu).
- `nextLesson` = Bài `unlocked` đầu tiên chưa `completed` (cho nút "Học tiếp"); null khi đã hoàn thành khóa.

---

## §P1 — `POST /api/courses/:slug/enroll` — Ghi danh (Q-P3.9 tự ghi danh)
- Idempotent (đã enroll ⇒ trả trạng thái hiện tại). **200** `{ progress }` (enrolled=true).
- **404** slug sai / khóa chưa `published`. **401** thiếu token.

## §P2 — `GET /api/courses/:slug/progress` — Đọc tiến độ
- **200** `{ progress }` (enrolled=true) hoặc `{ progress: { enrolled:false } }` nếu chưa ghi danh.
- **404** slug sai / chưa `published`. **401** thiếu token.

## §P3 — `POST /api/courses/:slug/lessons/progress` — Ghi nhận học Bài
- Body: `{ "phaseKey": "p1", "moduleKey": "m1", "lessonKey": "l1" }`.
- Hành vi: đánh dấu **viewed** (→ `in_progress`); **đánh giá hoàn thành** (Q-P3.3):
  - Bài có exercise `ipa`: `completed` ⇔ **mọi** exercise ipa có `ipa_progress.status ∈ {completed,mastered}`;
    `mastered` ⇔ **mọi** ipa `mastered`. Điểm ipa **đọc** qua `refId`(code)→`_id`, KHÔNG chấm lại (Q-P3.4).
    `talk` **bỏ qua** khi xét hoàn thành (Q-P3.5 — luyện tự do, chưa có engine chấm).
  - Bài không có exercise ipa: `completed` ngay khi viewed.
  - Hoàn thành → mở Bài kế (unlock tính lại). Hoàn thành hết ⇒ `enrollment.status='completed'`.
  - Chạm **streak** (Q-P3.8) `touchStreak(userId)`.
- **200** `{ progress }` (đã cập nhật). **409** `NOT_ENROLLED` nếu chưa ghi danh. **404** slug/khóa sai hoặc
  Bài (`pathKey`) không tồn tại trong cây. **401** thiếu token.

---

## Guard liên quan (Q-P3.2) — admin-api, không phải learner
`PATCH /api/admin/course-imports/:id/status`: chặn `published → ready` (gỡ xuất bản để sửa) khi khóa **đã có
enrollment** → **409** `COURSE_LOCKED` ("đã có học viên ghi danh"). Bảo vệ tiến độ đang keyed theo cấu trúc cây.
`published → archived` vẫn cho (chỉ ẩn, không sửa cây).
