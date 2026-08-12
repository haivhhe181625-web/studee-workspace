# Spec: Nâng cấp nhập liệu nội dung Lesson (Course Content Authoring)

- **Ngày:** 2026-08-10
- **Tác giả:** AI Brainstorm (đã được product owner duyệt thiết kế)
- **Repos/surfaces ảnh hưởng:** `exe-api` (BE), `exe-admin` (editor), `exe-web` (learner render)
- **Module liên quan (exe-api):** `services/api/src/modules/course-content/`, `services/api/src/modules/storage/`, tái dùng `services/api/src/modules/news/news-html.js`
- **Trạng thái:** Đã duyệt (spec + design) — sẵn sàng implement qua superpowers
- **Brainstorm nguồn:** `../exe-api`… → `plans/reports/from-brainstorm-to-spec-lesson-content-authoring-260810-0120-report.md` (repo `Đồ án` root)

## 1. Mục tiêu

Cho phép đội học thuật soạn nội dung Lesson có định dạng (rich text), ảnh (inline + block), video/audio (upload/URL + nhúng YouTube/Vimeo) trong admin, và học viên xem đúng định dạng đó ở không gian học.

## 2. Bối cảnh

Hiện trạng (đã kiểm tra code, không đoán):

- `CourseStructure` (`course-content.model.js`) là cây nội dung học thật (Phase→Module→Lesson→exercise). **Khác** `Course` (`course.model.js`) là catalog marketing/recommend — feature này **chỉ** đụng `CourseStructure`.
- Lesson lưu: `theory` (string, model chú là markdown), `media[]` (`{type: video|audio, url, label, order}` — **không có image**), `contentUrl` (asset chính), `exercises[]` (soft-ref).
- Editor admin (`exe-admin/src/components/features/course-import/lesson-editor-row.tsx:126`): `theory` chỉ là `<textarea rows=2>` thô.
- **Gap thực:** học viên (`exe-web/src/components/features/learn/LessonViewer.tsx:47-48`) render `theory` bằng `whitespace-pre-wrap` → **plain text**; markdown tác giả gõ hiện không hiển thị định dạng gì.
- Đã có sẵn rich-text editor TipTap hoàn chỉnh ở module News (`exe-admin/.../news/news-document-editor.tsx`) + sanitizer BE (`exe-api/.../news/news-html.js`) → **tái dùng được**.
- Storage (`storage-categories.js`): có `course-video` (500MB, signed), `course-audio` (20MB, signed), `news-image` (5MB, public). Typedef đã hỗ trợ `kind:'image'|'file'`. **Chưa có** category ảnh khóa học.
- BE validate media (`course-content.service.js:82-87`): check `MEDIA_TYPES` + chấp nhận http(s) URL hoặc ref nội bộ, chặn Base64.

## 3. Phạm vi

### Trong phạm vi

- Trình soạn thảo `theory`: thay `<textarea>` bằng TipTap WYSIWYG (tái dùng từ News), lưu **HTML đã sanitize** thay markdown.
- Ảnh inline: chèn trong TipTap.
- Ảnh block: thêm `image` vào `MEDIA_TYPES`, hiện trong danh sách `media[]`.
- Storage category mới `course-image` (**public**, `kind:'image'`), dùng chung cho ảnh inline + ảnh block.
- Nhúng YouTube/Vimeo: parse link → embed; áp cho `contentUrl` (video_lesson) và `media[]` video.
- Sanitize `theory` ở BE khi save (tái dùng allowlist `news-html.js`).
- **Migrate dữ liệu `theory` markdown cũ → HTML một lần** (content hiện chủ yếu là markdown — đã xác nhận).
- Learner render: `theory` → HTML sanitize; ảnh block; embed YouTube/Vimeo.

### Ngoài phạm vi

- **Tài liệu đính kèm tải về (PDF/DOCX/slide)** — **Lý do:** product owner chốt YAGNI vòng này; chưa có nhu cầu thật, thêm sau nếu cần (cần khái niệm attachments[] + storage `kind:file` mới).
- **Phụ đề .vtt + ảnh poster cho video/audio** — **Lý do:** ngoài scope đã chốt; giá trị chưa đủ rõ cho MVP.
- **Ảnh signed/gate nội dung** — **Lý do:** ảnh `course-image` để public; gate ảnh trả phí đòi giải bài toán refresh signed-URL trong HTML, phức tạp không tương xứng MVP.
- `Course` catalog (`course.model.js`) — **Lý do:** khác concept, không liên quan nhập liệu nội dung học.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Đội học thuật (admin) | Soạn lý thuyết có heading/list/bold/link/màu + chèn ảnh | Nội dung bài học dễ đọc, không phải gõ markdown thô |
| Đội học thuật (admin) | Thêm ảnh minh hoạ dạng block + dán link YouTube/Vimeo | Đa dạng media không tốn dung lượng self-host |
| Học viên (web) | Xem lý thuyết đúng định dạng, ảnh, video nhúng | Trải nghiệm học đầy đủ, đúng ý tác giả |

## 5. Quyết định nghiệp vụ cần chốt

Đã chốt trong brainstorm — không còn điểm chờ product owner:

| Câu hỏi | Đã chốt | Người quyết |
|---|---|---|
| Editor theory | TipTap WYSIWYG (lưu HTML) | Product owner |
| Ảnh | Cả inline + block | Product owner |
| Tài liệu đính kèm | Không làm (YAGNI) | Product owner |
| Video/audio | Thêm nhúng YouTube/Vimeo | Product owner |
| Access ảnh | Public | Product owner |

## 6. Acceptance criteria (tóm tắt)

1. Admin soạn `theory` bằng TipTap: heading/list/bold/italic/underline/link/màu/canh lề hoạt động; save → load lại giữ nguyên định dạng.
2. Admin chèn ảnh inline trong theory → ảnh upload lên `course-image`, hiển thị trong editor và sau khi save/load.
3. Admin thêm `media[]` type `image` với URL/ref → lưu và validate OK (không còn lỗi `UNSUPPORTED_MEDIA_TYPE`).
4. Admin dán link YouTube/Vimeo vào `contentUrl` (video_lesson) hoặc `media[]` video → lưu OK.
5. BE chặn Base64/`data:` ở ảnh y như video/audio (regression giữ nguyên).
6. BE sanitize `theory`: HTML chứa `<script>`/handler `onerror=` bị strip khi save (không tin client).
7. Learner (`LessonViewer`) render `theory` là HTML đã sanitize (không còn plain `whitespace-pre-wrap`); heading/list/ảnh hiển thị đúng.
8. Learner render `media[]` image (ảnh) + embed YouTube/Vimeo (iframe) ở `contentUrl`/media video.
9. News editor + News article view **không regression** sau khi tách component TipTap dùng chung.
10. Migration markdown→HTML: `theory` cũ (vd `## Tiêu đề`, `**đậm**`, `- item`) sau migrate hiển thị đúng định dạng ở learner (không còn ký tự markdown thô); script idempotent (chạy lại không hỏng HTML đã convert).

## 7. Câu hỏi mở

- [x] ~~Có `theory` dùng markdown thật trong DB không?~~ → **Đã trả lời: content chủ yếu là markdown.** → Migration markdown→HTML **bắt buộc**, đã đưa vào scope §3 + tasks. Không còn câu hỏi chặn approval.

## 8. Ghi chú cho Tech Lead Design

- **Bảo mật:** `theory` là HTML do admin nhập → XSS surface. Bắt buộc sanitize BE, không tin client. Tái dùng allowlist `news-html.js` thay vì viết mới.
- **DRY:** không viết editor TipTap mới — tách từ `news-document-editor.tsx`. Rủi ro regression News → phải test News trước khi dùng chung.
- **Storage:** `course-image` phải `public` (ảnh nhúng HTML, signed-URL sẽ hết hạn gây vỡ ảnh — đúng ghi chú trong `news-image`).
- **Cross-repo:** đụng 3 repo (api/web/admin), deploy theo thứ tự BE → admin/web (BE phải nhận `image` + `course-image` trước khi FE gửi).
