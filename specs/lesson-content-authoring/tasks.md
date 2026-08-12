# Tasks: Nâng cấp nhập liệu nội dung Lesson (Course Content Authoring)

> **Cho Developer Agent:** implement theo đúng thứ tự Task 1 → 6. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task. Test-first theo TDD.

**Design:** `specs/lesson-content-authoring/design.md`
**Lưu ý chung khi chạy lệnh:** lệnh BE chạy trong `exe-api/services/api`; lệnh admin trong `exe-admin`; lệnh web trong `exe-web`. Cross-repo: **hoàn tất Task 1-2 (exe-api) trước** khi làm FE.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `exe-api/services/api/src/modules/course-content/course-content.model.js` | `MEDIA_TYPES += 'image'` | Sửa |
| `exe-api/services/api/src/modules/storage/storage-categories.js` | thêm `course-image`, mở `REF_RE` | Sửa |
| `exe-api/services/api/src/modules/course-content/course-content.service.js` | cho phép image, sanitize theory | Sửa |
| `exe-admin/src/components/common/rich-text-editor/` | component TipTap dùng chung | Tạo |
| `exe-admin/src/components/features/news/news-document-editor.tsx` | dùng component chung | Sửa |
| `exe-admin/src/components/features/course-import/lesson-editor-row.tsx` | theory TipTap + media image + YouTube | Sửa |
| `exe-admin/src/components/features/course-import/course-tree-controls.tsx` | `MEDIA_TYPE += 'image'` | Sửa |
| `exe-admin/src/components/features/course-import/course-media-upload-button.tsx` | category `course-image` | Sửa |
| `exe-web/src/utils/video-embed.ts` | parse YouTube/Vimeo | Tạo |
| `exe-web/src/components/features/learn/LessonViewer.tsx` | theory render HTML | Sửa |
| `exe-web/src/components/features/learn/lesson-primary-content.tsx` | image media + embed | Sửa |

---

> **Đã chốt:** content `theory` hiện chủ yếu là markdown → **migration markdown→HTML bắt buộc** (Task 3). Thứ tự: Task 1-2 (BE nền tảng+sanitize) → **Task 3 (migration)** → Task 4-5-6 (FE) → Task 7 (full suite). Migration phải xong trước khi learner bật render HTML (Task 6).

---

## Task 1: BE — thêm media type `image` + storage category `course-image`

**Files:**
- Modify: `course-content/course-content.model.js`, `storage/storage-categories.js`, `course-content/course-content.service.js`
- Test: `course-content/__tests__/course-content-media-image.test.js`, `storage/__tests__/storage-categories.test.js`

- [ ] **Step 1: Viết test thất bại**
  - `MEDIA_TYPES` chứa `'image'`.
  - `validate` chấp nhận `media` `{type:'image', url:'course-image/cimg_x.webp'}` không lỗi.
  - `validate` `{type:'image', url:'data:image/png;base64,...'}` → `MEDIA_BASE64_BLOCKED`.
  - `parseRef('course-image/cimg_x.webp')` → hợp lệ; `parseRef('course-image/bad.gif')` → null.

- [ ] **Step 2: Chạy test — FAIL**

Run:
```bash
npm test -- course-content-media-image storage-categories
```
Expected: FAIL — `image` chưa có trong enum, `course-image` chưa có trong registry.

- [ ] **Step 3: Implement**
  - `course-content.model.js`: `MEDIA_TYPES = ['video','audio','image']`.
  - `storage-categories.js`: thêm entry `course-image` (public, `kind:'image'`, prefix `cimg_`, MIME jpeg/png/webp, `COURSE_IMAGE_MAX_MB` default 5); mở `REF_RE` để nhận `course-image`.
  - `course-content.service.js`: cập nhật message `UNSUPPORTED_MEDIA_TYPE` (video/audio/image). `checkUrl` không đổi (đã nhận ref nội bộ + chặn Base64).
  - env/config: khai báo `COURSE_IMAGE_MAX_MB` default 5.

- [ ] **Step 4: Chạy test — PASS**

Run:
```bash
npm test -- course-content-media-image storage-categories
```
Expected: ALL PASS. Regression: `npm test -- course-content` vẫn xanh (Base64 video/audio vẫn bị chặn).

- [ ] **Step 5: Commit**
```bash
git add src/modules/course-content/course-content.model.js src/modules/storage/storage-categories.js src/modules/course-content/course-content.service.js <test files> <config>
git commit -m "feat(course-content): add image media type and course-image storage category"
```

---

## Task 2: BE — sanitize `theory` HTML khi save (tái dùng news-html.js)

**Files:**
- Modify: `course-content/course-content.service.js`, (nếu cần) `news/news-html.js` export hàm sanitize
- Test: `course-content/__tests__/course-content-theory-sanitize.test.js`

- [ ] **Step 1: Viết test thất bại**
  - save lesson với `theory = '<h2>Ok</h2><script>alert(1)</script>'` → sau save `theory` KHÔNG chứa `<script>`.
  - `theory` với `<img src=x onerror=alert(1)>` → thuộc tính `onerror` bị strip.
  - HTML hợp lệ (`<h2>`, `<ul><li>`, `<strong>`, `<a href>`) được giữ.

- [ ] **Step 2: Chạy test — FAIL**

Run:
```bash
npm test -- course-content-theory-sanitize
```
Expected: FAIL — `theory` đang lưu nguyên văn, chưa sanitize.

- [ ] **Step 3: Implement**
  - Export hàm sanitize từ `news/news-html.js` (nếu chưa export). KHÔNG sửa allowlist News (tránh side-effect); nếu allowlist cần khác cho lesson → tách hàm dùng chung, giữ hành vi News nguyên.
  - Trong `course-content.service.js` (đường save/update): map qua lessons, `theory = sanitize(theory)` trước khi lưu.

- [ ] **Step 4: Chạy test — PASS**

Run:
```bash
npm test -- course-content-theory-sanitize && npm test -- news
```
Expected: sanitize PASS; News test vẫn xanh (không đổi hành vi News).

- [ ] **Step 5: Commit**
```bash
git add src/modules/course-content/course-content.service.js src/modules/news/news-html.js <test file>
git commit -m "feat(course-content): sanitize lesson theory html on save"
```

---

## Task 3: BE — migration `theory` markdown → HTML (một lần, idempotent)

**Files:**
- Create: `exe-api/services/api/scripts/migrate-lesson-theory-markdown-to-html.js`
- Test: `exe-api/services/api/scripts/__tests__/migrate-lesson-theory.test.js`

- [ ] **Step 1: Viết test thất bại**
  - convert `'## Tiêu đề\n\n**đậm** và list:\n- a\n- b'` → HTML có `<h2>`, `<strong>`, `<ul><li>`.
  - idempotent: đưa HTML đã convert vào lần 2 → không bị double-escape/hỏng.
  - kết quả đi qua `cleanHtml` → không có tag ngoài allowlist.

- [ ] **Step 2: Chạy test — FAIL**

Run:
```bash
npm test -- migrate-lesson-theory
```
Expected: FAIL — script/hàm convert chưa tồn tại.

- [ ] **Step 3: Implement**
  - Dùng `marked` (thêm vào devDependencies — CHỈ cho script, không vào runtime) convert markdown→HTML, rồi `cleanHtml` (news-html.js).
  - Idempotent: nếu chuỗi đã "trông như HTML" (có tag khối `<p>/<h2>/<ul>`) thì bỏ qua convert, chỉ `cleanHtml`.
  - Script: duyệt mọi `CourseStructure`, mọi `phases[].modules[].lessons[]`, cập nhật `theory`. In số lesson đã đổi. Hỗ trợ cờ `--dry-run` (chỉ đếm, không ghi).

- [ ] **Step 4: Chạy test — PASS + dry-run trên copy**

Run:
```bash
npm test -- migrate-lesson-theory
# rồi trên DB copy:
node scripts/migrate-lesson-theory-markdown-to-html.js --dry-run
```
Expected: test PASS; dry-run in ra số lesson sẽ đổi, verify vài mẫu tay.

- [ ] **Step 5: Commit** (chạy thật ở bước rollout, sau backup)
```bash
git add scripts/migrate-lesson-theory-markdown-to-html.js scripts/__tests__/migrate-lesson-theory.test.js package.json
git commit -m "feat(course-content): add lesson theory markdown-to-html migration script"
```

> ⚠️ **Rollout:** backup DB → chạy trên copy → verify → chạy prod. Phải xong TRƯỚC khi deploy learner render HTML (Task 6).

---

## Task 4: Admin — tách component TipTap dùng chung (không regression News)

**Files:**
- Create: `exe-admin/src/components/common/rich-text-editor/rich-text-editor.tsx` (+ toolbar tách kèm)
- Modify: `exe-admin/src/components/features/news/news-document-editor.tsx`
- Test: test hiện có của News + smoke test component chung

- [ ] **Step 1: Viết/चạy test hiện trạng News (baseline)**

Run:
```bash
npm run test -- news
```
Expected: ghi nhận baseline PASS trước khi refactor.

- [ ] **Step 2: Implement tách**
  - Trích phần editor TipTap (extensions, toolbar, paste-clean, insert-image) ra `common/rich-text-editor/`. Tham số hoá: `onImageUpload(file) => url` (News truyền `uploadNewsImage`, Lesson sẽ truyền upload `course-image`), `value`/`onChange` dạng HTML.
  - `news-document-editor.tsx` dùng lại component chung; giữ nguyên phần block-converter + excerpt (đặc thù News) ở lại file News.

- [ ] **Step 3: Chạy test — News PASS (no regression)**

Run:
```bash
npm run test -- news && npm run build
```
Expected: News test PASS, build OK. Verify tay: mở trang News editor, soạn/chèn ảnh/paste Word — hành vi y hệt trước.

- [ ] **Step 4: Commit**
```bash
git add src/components/common/rich-text-editor/ src/components/features/news/news-document-editor.tsx
git commit -m "refactor(editor): extract shared tiptap rich-text-editor from news"
```

---

## Task 5: Admin — lesson editor dùng TipTap + media image + input YouTube

**Files:**
- Modify: `course-import/lesson-editor-row.tsx`, `course-import/course-tree-controls.tsx`, `course-import/course-media-upload-button.tsx`, `services/course-content.service.ts` (types)
- Test: `course-import/lesson-editor-row.test.tsx` (hoặc mở rộng `course-tree-controls.test.ts`)

- [ ] **Step 1: Viết test thất bại**
  - `MEDIA_TYPE` chứa `'image'`.
  - `CourseMediaUploadButton` chấp nhận category `course-image` (ACCEPT MIME ảnh).
  - editor: đổi theory qua rich editor → `onEdit` nhận HTML string.

- [ ] **Step 2: Chạy test — FAIL**

Run:
```bash
npm run test -- lesson-editor course-tree-controls
```
Expected: FAIL — `image` chưa có, upload chưa có category ảnh.

- [ ] **Step 3: Implement**
  - `course-tree-controls.tsx`: `MEDIA_TYPE = ['video','audio','image']`.
  - `course-media-upload-button.tsx`: thêm `course-image` vào `ACCEPT` + label; type `MediaCategory` mở rộng.
  - `course-content.service.ts`: `MediaCategory += 'course-image'`; media type union thêm `image`; hàm `uploadCourseMedia` dùng route generic với category ảnh.
  - `lesson-editor-row.tsx`: thay `<textarea theory>` bằng `<RichTextEditor>` (component chung Task 4), truyền `onImageUpload` = upload `course-image`. Với media type `image`: input URL + nút upload `course-image`. Với `video_lesson` contentUrl: cho dán link YouTube/Vimeo (đã có cảnh báo URL sẵn — nới regex chấp nhận youtu/vimeo).

- [ ] **Step 4: Chạy test — PASS**

Run:
```bash
npm run test -- lesson-editor course-tree-controls && npm run build
```
Expected: ALL PASS, build OK. Verify tay: soạn 1 lesson có theory định dạng + ảnh inline + media image + link YouTube → save → reload khớp.

- [ ] **Step 5: Commit**
```bash
git add src/components/features/course-import/ src/services/course-content.service.ts <test>
git commit -m "feat(course-import): rich-text theory, image media and youtube/vimeo input"
```

---

## Task 6: Web — learner render theory HTML + image media + embed YouTube/Vimeo

**Files:**
- Create: `exe-web/src/utils/video-embed.ts`
- Modify: `learn/LessonViewer.tsx`, `learn/lesson-primary-content.tsx`
- Test: `utils/video-embed.test.ts`, `learn/LessonViewer.test.tsx`

- [ ] **Step 1: Viết test thất bại**
  - `parseVideoEmbed('https://youtu.be/abc123')` → `{provider:'youtube', id:'abc123'}`; Vimeo tương tự; URL .mp4 → null.
  - `LessonViewer` render `theory` HTML (có `<h2>`) ra element heading, KHÔNG còn text thuần `whitespace-pre-wrap`.
  - `media` type `image` render `<img>`.

- [ ] **Step 2: Chạy test — FAIL**

Run:
```bash
npm run test -- video-embed LessonViewer
```
Expected: FAIL — util chưa có; theory đang render plain text; image media chưa xử lý.

- [ ] **Step 3: Implement**
  - `video-embed.ts`: parse YouTube (`youtu.be/`, `watch?v=`, `/embed/`) + Vimeo → `{provider,id,embedUrl}`; else null.
  - `LessonViewer.tsx`: render `theory` bằng HTML đã sanitize (BE sanitize ở Task 2 + migration Task 3 đã convert content cũ → an toàn `dangerouslySetInnerHTML`, giống `NewsArticleView.tsx:145`). CSS đảm bảo heading/list/img hiển thị. **Chỉ deploy sau khi migration Task 3 chạy prod xong.**
  - `lesson-primary-content.tsx` (`PrimaryContent`/`MediaBlock`): nếu URL parse ra embed → `<iframe>`; media type `image` → `<img>`; giữ nhánh `<video>`/`<audio>` cũ.

- [ ] **Step 4: Chạy test — PASS**

Run:
```bash
npm run test -- video-embed LessonViewer lesson-primary && npm run build
```
Expected: ALL PASS, build OK. Verify tay: mở lesson soạn ở Task 4 → định dạng/ảnh/YouTube hiển thị đúng.

- [ ] **Step 5: Commit**
```bash
git add src/utils/video-embed.ts src/components/features/learn/
git commit -m "feat(learn): render html theory, image media and youtube/vimeo embeds"
```

---

## Task 7: Chạy full suite + dọn dẹp

**Files:** (không sửa code) — verify + tài liệu.

- [ ] **Step 1: Full suite 3 repo**

Run:
```bash
# exe-api/services/api
npm test
# exe-admin
npm run test && npm run build
# exe-web
npm run test && npm run build
```
Expected: ALL PASS, cả 3 build OK.

- [ ] **Step 2: Cập nhật tài liệu (chỉ trong phạm vi feature)**
  - Cập nhật chú thích model `course-content.model.js` (theory: markdown→HTML sanitize; media thêm image).
  - Nếu có `docs/` mô tả storage categories → thêm `course-image`.

---

## Self-Review

**Acceptance coverage (spec §6):**
- 1. Theory TipTap định dạng → Task 4+5+6. ☐
- 2. Ảnh inline → Task 5 (upload course-image) + 6 (render). ☐
- 3. media image → Task 1 (BE) + 5 (admin) + 6 (render). ☐
- 4. Nhúng YouTube/Vimeo → Task 5 (input) + 6 (embed util). ☐
- 5. Chặn Base64 ảnh → Task 1. ☐
- 6. Sanitize theory BE → Task 2. ☐
- 7. Learner theory HTML → Task 6. ☐
- 8. Learner image + embed → Task 6. ☐
- 9. News không regression → Task 4. ☐
- 10. Migration markdown→HTML → Task 3. ☐

**Placeholder scan:** xác nhận không còn TBD/TODO khi hoàn tất.

**Type/name consistency:** `course-image` (category), `cimg_` (prefix), `parseVideoEmbed` (util), `RichTextEditor` (component chung) — dùng đồng nhất qua các task.

**Thứ tự bắt buộc:** Task 3 (migration) chạy prod TRƯỚC khi Task 6 (learner render HTML) deploy — nếu không content markdown cũ sẽ hiện ký tự thô.
