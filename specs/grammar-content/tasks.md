---
feature: grammar-content
status: draft
owner: TechLead
created: 2026-08-19
---

# Tasks — grammar-content

> TDD-checkbox. Mỗi task 1 PR-worth, có lệnh verify. BE (`exe-api`) trước; FE (`exe-web`) PR riêng sau khi API xanh.
> Ngoài phạm vi: nối SRS (plan cha). Chốt 2 câu hỏi mở ở design trước khi bắt đầu.

## BE — exe-api (làm trước)

- [ ] **T1. block-registry + validate (fail-soft)**
  `src/modules/grammar-content/block-registry.js` — map 11 type → validate + publicFields; type lạ/sai → loại + log, không throw.
  Verify: `npm test -- grammar-content/block-registry` (test: block hợp lệ pass; type lạ bị loại, hàm không throw).

- [ ] **T2. Model `GrammarLessonContent`**
  `grammar-content.model.js` (+ `.schema.js` Joi nếu cần) — fields theo design (**KHÔNG có `reference`**); `grammarPoint` unique index; `status` enum+index; `learn` loose.
  Verify: `npm test -- grammar-content/model` (tạo doc hợp lệ; trùng grammarPoint bị chặn; status ngoài enum bị reject).

- [ ] **T3. Service (get/list/setStatus) + toDto**
  `grammar-content.service.js` — `getActiveByPoint`, `listActiveByCategory`, `setStatus`; `toDto` strip `_source/_note` (+ `reference` theo chốt #1). Dùng registry lọc block trước khi trả.
  Verify: `npm test -- grammar-content/service` (draft không ra ở public getter; dto không chứa `_source`).

- [ ] **T4. Controller + Routes public**
  `.controller.js` (asyncHandler + ApiError), `.routes.js` mount `/api/grammar-content`: `GET /:grammarPoint`, `GET /?category=`.
  Verify: e2e/smoke — `GET /:point` active → 200 + learn[]; point không active → 404; response không lộ `_source`.

- [ ] **T5. Import script**
  `scripts/import-grammar-content.js` — đọc `scripts/grammar-content/output/*.blocks.json`, **bỏ field `reference`**, upsert theo grammarPoint, `--dry-run`.
  Verify: chạy `--dry-run` in **55** bài; chạy thật 2 lần → `GrammarLessonContent.countDocuments()===55` (không nhân đôi).

- [ ] **T6. Admin resource (status)**
  `permissions.js` thêm `grammar-content:manage`; `admin-api/resources/grammar-content.admin.js` qua `_crud.factory.js`, `writeFields:['status']`, delegate `service.setStatus` (không business logic ở admin-api).
  Verify: PATCH status có permission → đổi được + audit; thiếu permission → 403; public phản ánh sau khi active.

## FE — exe-web (PR riêng, sau khi API xanh)

- [ ] **T7. BlockRenderer registry + 11 component**
  `components/grammar/blocks/*` + registry `type→component`, fail-soft type lạ. Port style/tương tác từ `render-review.js`. **TableBlock tự gập** (accordion) khi > ~15 dòng.
  Verify: story/test render mỗi type; formula toggle ví dụ · signal reveal nghĩa · minicheck chấm + giải thích · detail accordion · bảng dài gập được.

- [ ] **T8. Study-space tích hợp**
  Trang học fetch `GET /api/grammar-content/:grammarPoint` → render `learn[]`; theme sáng/tối; responsive.
  Verify: mở 1 bài (vd simple_present_tense) hiển thị đúng như artifact; 404 xử lý gọn.

## Sau feature (không thuộc spec này)
- Nối SRS concept-based + item pool (plan cha `plans/260817-1604-grammar-srs-shared-module`).
- Progress/tracking học bài ngữ pháp.
- Liên kết course lesson (type theory) → grammar-content (câu hỏi mở #2).
