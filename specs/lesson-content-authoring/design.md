# Design: Nâng cấp nhập liệu nội dung Lesson (Course Content Authoring)

- **Spec:** `specs/lesson-content-authoring/spec.md`
- **Ngày:** 2026-08-10
- **Tác giả:** Tech Lead Agent (spec-workflow LITE)
- **Trạng thái:** Đã duyệt (2026-08-10, product owner — giữ khuyến nghị HTML + migrate markdown→HTML)
- **ADR liên quan:** không có (không tạo quyết định kiến trúc durable mới; mở rộng registry sẵn có)

## 1. Tóm tắt kiến trúc

Mở rộng module `course-content` + `storage` (không tạo module mới). `theory` chuyển từ "string markdown render plain text" sang **HTML string đã sanitize**, dùng editor TipTap tái dùng từ module News và sanitizer BE `news-html.js` (đã tồn tại). Ảnh dùng 1 storage category mới `course-image` (public) cho cả 2 lối: inline trong TipTap và block trong `media[]`. Video/audio thêm nhúng YouTube/Vimeo qua 1 util parse URL dùng chung admin+web. Không mâu thuẫn ADR nào; storage-registry đã thiết kế sẵn để "thêm 1 entry = thêm media type".

## 2. Quyết định kiến trúc (6 quyết định — đã chốt, kèm khuyến nghị)

| # | Quyết định | Lựa chọn (KHUYẾN NGHỊ) | Lý do | Đánh đổi | Phương án loại bỏ |
|---|---|---|---|---|---|
| 1 | Định dạng lưu `theory` | **HTML đã sanitize + migrate markdown cũ → HTML một lần** | Tái dùng TRỌN pipeline News: `cleanHtml` (BE, đã có `sanitize-html`) + render `dangerouslySetInnerHTML` (web, như `NewsArticleView.tsx:145`). **0 dependency mới.** 1 định dạng duy nhất về sau. | Phải migrate dữ liệu markdown cũ 1 lần (chi phí thật — content đang chủ yếu markdown) | Giữ markdown: cần `tiptap-markdown` (admin) + `react-markdown` (web) — 2 dep mới, round-trip TipTap↔markdown lossy (mất color/font-size/align), 2 code-path song song |
| 2 | Editor | **Tách component TipTap dùng chung từ News** | DRY — không viết editor mới | Rủi ro regression News khi refactor → test News trước | Viết editor riêng cho lesson (vi phạm DRY) |
| 3 | Sanitize | **BE sanitize `theory` khi save, allowlist dùng chung với News** | Không tin client; chống XSS dù admin tin cậy | Thêm 1 bước ở service save | Chỉ sanitize client (không an toàn) |

> **Cập nhật khi implement (batch 1):** News `cleanHtml` hoá ra là **inline-only** (strip `<h2>/<ul>/<li>`), không dùng thẳng được cho theory. Đã refactor `news-html.js` thành `buildOptions()` dùng chung + thêm export **`cleanLessonHtml`** (allowlist block `h1-h3/p/ul/ol/li` + inline formatting + `<img src,alt>` an toàn, scheme http/https). News giữ nguyên hành vi (`cleanHtml`/`OPTIONS`/`ALLOWED_TAGS` byte-identical). Ảnh inline (`<img>`) trong theory được cho phép qua `cleanLessonHtml` → không bị strip khi save. Migration (Task 3) cũng pipe qua `cleanLessonHtml`.
| 4 | Ảnh access | **`course-image` = public** | Ảnh nhúng trong HTML; signed-URL hết hạn gây vỡ ảnh (đúng ghi chú `news-image`) | Không gate được ảnh trả phí (chấp nhận cho MVP) | Signed — phải giải refresh-URL trong HTML, phức tạp không tương xứng |
| 5 | Ảnh inline + block | **Cùng dùng category `course-image`** | 1 nơi lưu, 2 lối hiển thị | 2 đường render ở learner | 2 category riêng (thừa) |
| 6 | Nhúng video | **1 util `parseVideoEmbed` dùng chung admin+web** (YouTube/Vimeo) | DRY, nhất quán validate/render | Thêm nhánh render iframe | Nhúng ad-hoc mỗi nơi (lệch nhau) |

> **Quyết định #1 cần Tech Lead/Product xác nhận riêng** vì migration đụng dữ liệu learner thật (rủi ro reversibility). Khuyến nghị HTML+migrate vì tối đa hoá reuse + 0 dep mới; nếu muốn tránh migrate hoàn toàn thì chọn phương án giữ markdown (đánh đổi 2 dep + lossy).

## 3. Data model

Thay đổi trên `CourseStructure` (`course-content.model.js`) — **nhỏ**:

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `MEDIA_TYPES` | enum | — | Thêm `'image'`: `['video','audio','image']` |
| `Lesson.theory` | String | không | Ngữ nghĩa đổi: markdown → **HTML đã sanitize**. Kiểu không đổi (vẫn String) → không cần migration *schema*, NHƯNG cần migration *dữ liệu* (content cũ đang là markdown → convert sang HTML một lần) |
| `Lesson.media[].type` | enum | có | Nay cho phép `image` (URL/ref ảnh) |

`storage-categories.js` — thêm 1 entry:

| Field category `course-image` | Giá trị |
|---|---|
| `kind` | `'image'` |
| `access` | `'public'` |
| `dir/subPath` | `image/course` |
| `prefix` | `cimg_` |
| `filenameRe` | `/^cimg_[A-Za-z0-9_-]+\.(jpg\|jpeg\|png\|webp)$/` |
| `uploadMime` | `['image/jpeg','image/png','image/webp']` |
| `maxMb` | `COURSE_IMAGE_MAX_MB` (default 5) |
| `xaccel` | `false` |
| `cacheSec` | dài (ảnh không đổi tên) |

`REF_RE` trong `storage-categories.js` phải mở rộng để nhận `course-image/` (hiện chỉ khớp `course-video\|course-audio`).

## 4. Luồng dữ liệu

```
Admin editor (TipTap) --(theory HTML + media[])--> PUT /api/admin/course-structures/:id
  verifyPermission (course-content:manage)
  -> course-content.service: validate media (image cho phép, chặn Base64)
  -> sanitizeHtml(theory) qua news-html.js  ← chống XSS
  -> lưu CourseStructure
  -> response toDto

Admin upload ảnh --(file)--> POST /api/admin/media/course-image (generic route)
  -> lưu image/course/cimg_x.webp -> trả ref "course-image/cimg_x.webp" (public URL)

Learner --(GET lesson)--> LessonViewer
  theory: render HTML đã sanitize (dangerouslySetInnerHTML sau sanitize, hoặc tin BE đã sanitize)
  media image: <img>
  contentUrl/media video: nếu YouTube/Vimeo -> <iframe> embed; ngược lại <video>
```

## 5. Contracts

Không cần file contract riêng — thay đổi nằm trong request/response CRUD sẵn có của `course-structures` (mở rộng enum + field, không đổi shape endpoint). Nhúng YouTube/Vimeo là logic client-side thuần trên URL đã lưu.

- (không có `contracts/*.md`)

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `exe-api/.../course-content/course-content.model.js` | `MEDIA_TYPES += 'image'` | Sửa |
| `exe-api/.../storage/storage-categories.js` | thêm `course-image`, mở `REF_RE` | Sửa |
| `exe-api/.../course-content/course-content.service.js` | cho phép `image`, sửa message, sanitize `theory` | Sửa |
| `exe-api/.../news/news-html.js` | (tái dùng) export sanitizer nếu chưa | Sửa nhẹ/không |
| `exe-api` env/config | `COURSE_IMAGE_MAX_MB` default 5 | Sửa |
| `exe-api/.../scripts/migrate-lesson-theory-markdown-to-html.js` | migration 1 lần markdown→HTML→cleanHtml | Tạo |
| `exe-admin/.../course-import/lesson-editor-row.tsx` | theory→TipTap; media image; input YouTube | Sửa |
| `exe-admin/.../course-import/course-tree-controls.tsx` | `MEDIA_TYPE += 'image'` | Sửa |
| `exe-admin/.../course-import/course-media-upload-button.tsx` | thêm category `course-image` | Sửa |
| `exe-admin/src/components/common/rich-text-editor/*` | component TipTap tách dùng chung | Tạo |
| `exe-admin/.../news/news-document-editor.tsx` | dùng lại component chung | Sửa |
| `exe-admin/.../course-content.service.ts` | type `MediaCategory += 'course-image'`; type media image | Sửa |
| `exe-web/.../learn/LessonViewer.tsx` | theory render HTML sanitize | Sửa |
| `exe-web/.../learn/lesson-primary-content.tsx` | render image media + embed YouTube/Vimeo | Sửa |
| `exe-web/src/utils/video-embed.ts` (+ admin bản sao/shared) | parse YouTube/Vimeo ID | Tạo |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| media type ngoài enum | 422 | `UNSUPPORTED_MEDIA_TYPE` (message cập nhật: video/audio/image) |
| ảnh Base64/`data:` | 422 | `MEDIA_BASE64_BLOCKED` (dùng lại) |
| URL/ref ảnh không hợp lệ | 422 | (dùng lại validate URL sẵn có) |
| upload sai MIME/quá size | 422 | (dùng lại lỗi upload route generic) |

## 8. Bảo mật & quyền

- Permission dùng lại của course-content (`course-content:manage` hoặc tương đương hiện hành) — **không thêm permission mới**.
- `theory` HTML: **bắt buộc sanitize BE** qua `cleanLessonHtml` (export mới trong `news-html.js`, dùng chung `sanitize-html`; strip `<script>`, event handler `on*`, scheme `javascript:`/`data:`). Không tin sanitize client-side. Áp tại 1 choke point duy nhất (`buildTree`) phủ cả import + update.
- `course-image` public: chấp nhận ảnh xem được không cần auth (đã chốt product). Không chứa dữ liệu nhạy cảm.
- Base64 vẫn chặn cho mọi media type (giữ regression).

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **`theory` markdown cũ — ĐÃ XÁC NHẬN content chủ yếu là markdown** → **bắt buộc migrate markdown→HTML một lần** (không còn là "giả định text thuần"). Rủi ro: convert sai/mất định dạng trên content thật.
  - Mitigation: migration script chạy trên **bản copy DB** trước; convert bằng `marked`/`markdown-it` → `cleanHtml` → verify sample tay; idempotent (bỏ qua chuỗi đã là HTML); backup trước khi chạy prod.
- **Regression News** khi tách TipTap dùng chung → mitigation: task tách có bước verify News trước khi lesson dùng.
- **Dependency migration**: exe-api chưa có lib markdown→html. Dùng `marked` (nhẹ) CHỈ trong migration script; không thêm vào runtime path (runtime chỉ cần `sanitize-html` đã có).
- Không cần `research.md` — không có câu hỏi kỹ thuật cần spike.

## 10. Testing strategy

- **BE unit** (jest, `exe-api/services/api`): validate media chấp nhận `image`, chặn Base64 ảnh; sanitize `theory` strip `<script>`/`onerror`.
- **BE**: `storage-categories` parseRef nhận `course-image/cimg_x.webp`, từ chối tên sai shape.
- **Admin** (vitest/jest hiện có, xem `course-tree-controls.test.ts`): editor lưu HTML từ TipTap; media image thêm/sửa.
- **Web**: LessonViewer render HTML sanitize; embed util parse đúng ID YouTube/Vimeo; MediaBlock render image.
- **Regression**: News editor/article view test vẫn xanh.

## 11. Rollout / cross-repo sequencing

1. **exe-api** trước: `image` + `course-image` + sanitize `theory` + upload route. Deploy `develop`.
2. **Migration markdown→HTML** (exe-api script): backup → chạy trên copy → verify → chạy prod. **Phải xong TRƯỚC khi bật render HTML ở learner** (nếu không, content cũ hiện ký tự thô).
3. **exe-admin** + **exe-web** sau (song song được): FE chỉ gửi `image`/`course-image` khi BE đã nhận; web đổi sang render HTML sau khi migration xong.
Từng repo 1 PR riêng base `develop` (theo project-context §0/§6).

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion (spec §6) | Đáp ứng bởi |
|---|---|
| 1. Theory TipTap định dạng | §3 (theory HTML), §6 admin editor + component chung |
| 2. Ảnh inline | §3 course-image, §6 lesson-editor + TipTap image |
| 3. media image | §3 MEDIA_TYPES, §7 validate |
| 4. Nhúng YouTube/Vimeo | §4/§6 video-embed util |
| 5. Chặn Base64 ảnh | §7 MEDIA_BASE64_BLOCKED |
| 6. Sanitize theory BE | §8 sanitize news-html.js |
| 7. Learner theory HTML | §6 LessonViewer |
| 8. Learner image + embed | §6 lesson-primary-content |
| 9. News không regression | §9 rủi ro + §10 regression test |
