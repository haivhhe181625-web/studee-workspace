---
feature: luyen-de-item-bank-generation
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Design — Blueprint-driven batch generation (tái dùng hạ tầng sẵn có)

## 1. Hạ tầng đã có (KHÔNG viết lại — chỉ điều phối)

| Thành phần | File | Vai trò |
|---|---|---|
| `generateAndInsertTestlet({skill, tier, itemTypes, targetGoal, topic, ...})` | `assessment/gen-testlet.service.js` | Gọi LLM sinh testlet → validate cấu trúc → quality gate → insert `draft`/`ai-generated-reviewed` + metadata (cefr từ tier, `targetGoal`, `irt.b` tạm) |
| `runQualityGates` (moderation + dedup) | `assessment/gen-quality.service.js` | Loại nội dung vi phạm / trùng |
| `embedAndIndex` / `findSimilarQuestions` (Qdrant) | `assessment/dedup.service.js` | Near-dup, ngưỡng 0.92 |
| `summarizeBank` / gap analysis | `assessment/bank-coverage.js` | Pure — đếm bank + tính thiếu vs target matrix |
| Admin action `generate` | `admin-api/resources/assessment.admin.js` | `POST /stimuli/_/generate` đã expose |

`tier → cefr`: `TIER_CEFR = { easy:'A2', mid:'B1', hard:'C1' }`; `tier → irt.b`: `{-1, 0, +1}` (tạm).

## 2. Lớp mới (mỏng)

```
Khung active (Bước 1, DB)
   │  gom section → cell {skill, cefr, itemTypes, targetGoal}, need = Σ itemCount × surplus
   ▼
planExamPracticeCoverage()  ── reuse bank-coverage gap ──►  [{skill,cefr,itemType,targetGoal,need,have,gap}]
   │
   ▼  cho mỗi cell gap>0
runGenerationBatch()  ──►  generateAndInsertTestlet(...)  (loop tới gap≤0 hoặc chạm trần)
   │                         (đã tự qua moderation + dedup + insert draft)
   ▼
báo cáo: cell nào đủ / thiếu / cost
```

- **`tier` từ `cefr`**: planner map ngược cefr→tier (A2→easy, B1→mid, C1→hard) để gọi gen (gen nhận tier).
  Cell cefr B2 (TOEIC) chưa có tier trực tiếp → dùng `cefrHint`/`tier='hard'` gần nhất; ghi rõ giới hạn.
- **targetGoal bắt buộc**: runner luôn truyền `targetGoal` của khung → item assemble thấy được.
- **Trần an toàn**: max attempts/cell, max tổng cost; chạm trần → dừng + report (AC2). Không loop vô hạn
  (gen có thể fail gate liên tục nếu topic cạn).

## 3. Queue vs script

Đề xuất **BullMQ job** (đã có trong stack): LLM chậm + rate-limit → cần throttle, retry, resume, theo dõi
tiến độ. Script ops đồng bộ chỉ hợp cho batch nhỏ/dev. FE đọc tiến độ job. (Xác nhận khi review — nếu muốn
đơn giản trước, script ops chạy tay cho lần mồi đầu, queue hoá sau.)

## 4. Vet gate

Item ra `draft` + `ai-generated-reviewed`. Đường lên `active`:
- **Vet tay**: examiner xem (màn review sẵn có của assessment) → active.
- **No-human-review**: gen-testlet đã hỗ trợ insert thẳng active khi machine gate pass — chế độ vận hành,
  **không bật mặc định** ở feature này (hỏi owner). Mặc định an toàn = draft + vet.

## 5. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| LLM sinh trùng lặp nhiều (topic cạn) | dedup gate 0.92 loại; xoay `topic`/`languagePoint`; trần attempts báo thiếu |
| Dedup mock vector (không có OPENAI key) chỉ match exact | Cần OPENAI_API_KEY thật để dedup semantic (dedup.service note) — nêu prerequisite |
| Cost LLM lớn khi mồi 100s câu TOEIC | Trần cost + queue throttle; mồi từng phần; ưu tiên cell thiếu nặng |
| cefr B2 không map tier trực tiếp | dùng cefrHint; chấp nhận provisional; calibrate sau |

## Câu hỏi mở

- Surplus factor, queue vs script, no-human-review (xem spec §Câu hỏi mở).
