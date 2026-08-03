<!-- Tiếng Việt — PRD/SRS cho "Quản lý khóa học" (Giai đoạn A — Admin).
     Viết cho người KHÔNG chuyên kỹ thuật (đội vận hành/học thuật).
     Chi tiết kỹ thuật: design.md & contracts/admin-course-management.md -->

# PRD — Trang Admin Quản lý Khóa Học

- **Ngày:** 2026-08-03 · **Trạng thái:** Nháp để duyệt · **Đối tượng đọc:** PO/BA/đội vận hành/học thuật + kỹ thuật
- **Chi tiết kỹ thuật/khung:** `specs/course-management-admin/design.md` (quyết định đã chốt)
- **Phụ thuộc:** hoàn thành Phase 1 (import khóa từ file); tái dùng pipeline validate, nội dung các Chuyên đề.

> Tài liệu mô tả **các chức năng quản lý khóa** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `specs/course-management-admin/design.md`.

---

## 1. Mục đích & bối cảnh

Hiện tại, sau khi **nhập khóa từ file** (Phase 1), khóa được lưu ở trạng thái **Nháp** và chỉ có thể **xoá hoàn toàn**. Đội vận hành/học thuật **không thể sửa nội dung khóa, không thể xuất bản, và không thể quản lý vòng đời** của khóa từ nháp → sẵn sàng → xuất bản → lưu trữ.

**Mục tiêu:** cung cấp **trang Admin cho phép đội vận hành/học thuật:**
1. **Xem chi tiết** toàn bộ cấu trúc khóa (các Chặng, Chuyên đề, Bài học, media, bài tập).
2. **Sửa nội dung** của khóa (tiêu đề, mô tả, ảnh bìa, nội dung bài học…) trực tiếp trên trang, mà không cần nhập file lại.
3. **Quản lý trạng thái** — chuyển khóa qua các giai đoạn: **Nháp** → **Sẵn sàng** → **Xuất bản** → **Lưu trữ**, hoặc quay lại nếu cần.

Với chức năng này, đội có thể **chủ động chuẩn bị và ra mắt khóa học** mà không cần can thiệp kỹ thuật sau mỗi bước.

## 2. Phạm vi

**TRONG phạm vi:**
1. **Xem chi tiết khóa** — hiển thị toàn bộ cấu trúc (Chặng → Chuyên đề → Bài học → media/bài tập).
2. **Sửa nội dung khóa** — cho phép cập nhật tiêu đề, mô tả, ảnh bìa, lý thuyết/nội dung bài học, thêm/xoá/di chuyển media và bài tập, **chỉ khi khóa ở trạng thái Nháp hoặc Sẵn sàng**.
3. **Quản lý trạng thái** — các nút/menu để chuyển khóa qua các trạng thái (Nháp, Sẵn sàng, Xuất bản, Lưu trữ) theo quy tắc cho phép.
4. **Kiểm tra tính hợp lệ** — trước khi chuyển sang Sẵn sàng hoặc Xuất bản, hệ thống **tự động kiểm tra** xem khóa có đủ điều kiện (không lỗi tham chiếu, các Bài học đầy đủ…) và thông báo lỗi nếu có.
5. **Lịch sử thay đổi** — ghi lại mỗi lần ai sửa khóa hoặc chuyển trạng thái (để kiểm toán).

**NGOÀI phạm vi (để sau):**
- Tự sinh nội dung mới bằng AI.
- Lập lịch học chi tiết theo ngày/giờ.
- Thay đổi slug khóa (định danh).
- Sửa khóa **đã xuất bản** trực tiếp (phải gỡ xuất bản trước).

## 3. Người dùng

| Vai | Nhu cầu | Thao tác chính |
|---|---|---|
| **Quản trị viên nội dung (đội vận hành/học thuật)** | Quản lý vòng đời khóa, sửa nội dung, ra mắt khóa. | Xem, sửa, chuyển trạng thái, xuất bản khóa. |
| **Kỹ thuật (hỗ trợ)** | Không cần thao tác mới nếu phần trên hoạt động tốt; chỉ theo dõi logs. | Có thể xem logs kiểm toán nếu cần. |

## 4. Tổng quan

Trang Admin cho phép quản trị viên:

1. **Danh sách khóa** — xem tất cả khóa đã nhập, trạng thái hiện tại (Nháp/Sẵn sàng/Xuất bản/Lưu trữ), ngày nhập, có nút **"Xem / Sửa"** để mở trang chi tiết.

2. **Trang chi tiết khóa** — hiển thị toàn bộ cấu trúc như cây thứ bậc:
   - **Tiêu đề, mô tả, ảnh bìa** của khóa (có thể sửa).
   - **Danh sách Chặng** (ví dụ: "Chặng 1: A2→B1"), mỗi Chặng chứa:
     - **Danh sách Chuyên đề** (ví dụ: "Ngữ pháp"), mỗi Chuyên đề chứa:
       - **Danh sách Bài học** (ví dụ: "Bài 1: Present Simple"), mỗi Bài chứa:
         - **Nội dung lý thuyết** (có thể sửa).
         - **Media** (video, audio…) — có thể thêm/xoá.
         - **Bài tập** — có thể thêm/xoá.

3. **Chế độ sửa** (chỉ khi khóa ở Nháp/Sẵn sàng):
   - Các trường dữ liệu có thể chỉnh sửa (tiêu đề, nội dung…).
   - Có nút **Thêm**, **Xoá**, **Di chuyển** để cấu trúc lại Chặng/Chuyên đề/Bài học.
   - Nút **Lưu** để cập nhật; hệ thống **tự động kiểm tra lỗi** (như tham chiếu bài tập không hợp lệ) và báo cáo.

4. **Điều khiển trạng thái** (máy trạng thái):
   - Tùy theo trạng thái hiện tại, hiển thị các nút khác nhau:
     - **Nháp** → có nút **"Sẵn sàng"**, **"Lưu trữ"** (chỉnh sửa tự do).
     - **Sẵn sàng** → có nút **"Quay lại Nháp"**, **"Xuất bản"**, **"Lưu trữ"**.
     - **Xuất bản** → có nút **"Gỡ xuất bản"** (quay về Sẵn sàng), **"Lưu trữ"** (học viên không thể truy cập nữa).
     - **Lưu trữ** → có nút **"Khôi phục"** (quay về Sẵn sàng).
   - Khi chuyển sang **Sẵn sàng** hoặc **Xuất bản**, hệ thống kiểm tra lỗi; nếu có → hiển thị danh sách lỗi và **cản chuyển**.
   - Khi chuyển từ **Xuất bản** về **Sẵn sàng** (gỡ xuất bản), hỏi xác nhận ("Bạn chắc chắn muốn gỡ xuất bản khóa này? Học viên sẽ không thể truy cập nữa.").

## 5. Quy tắc quản lý trạng thái

**Máy trạng thái (chỉ cho phép các chuyển dưới đây):**

| Từ trạng thái | Có thể chuyển sang | Chú ý |
|---|---|---|
| **Nháp** | Sẵn sàng, Lưu trữ | Được chỉnh sửa tự do. |
| **Sẵn sàng** | Nháp, Xuất bản, Lưu trữ | Chuyển sang Xuất bản phải vượt kiểm tra lỗi. |
| **Xuất bản** | Sẵn sàng, Lưu trữ | Gỡ xuất bản (về Sẵn sàng) cần xác nhận; khóa có thể được sửa lại ở Sẵn sàng. |
| **Lưu trữ** | Sẵn sàng | Khôi phục được; khóa bị ẩn khỏi học viên nhưng không xoá. |

**Kiểm tra tính hợp lệ (re-validate):**
- Trước khi chuyển sang **Sẵn sàng** hoặc **Xuất bản**, hệ thống kiểm tra:
  - Tất cả Bài học có nội dung hợp lệ (không trống, đúng định dạng).
  - Mọi tham chiếu bài tập (ipa, talk…) tồn tại và được xuất bản.
  - Không có lỗi cấu trúc (các node bị hỏng…).
- Nếu có lỗi → hiển thị **bảng lỗi chi tiết** (ví dụ: "Chặng 'Chặng 1' › Chuyên đề 'Ngữ pháp' › Bài 'Bài 1' › bài tập #2: tham chiếu không tồn tại"). Người dùng phải sửa trước khi chuyển.

**Không được sửa khóa Xuất bản:**
- Khóa ở trạng thái **Xuất bản** **không thể sửa trực tiếp**. Phải **gỡ xuất bản** (chuyển về Sẵn sàng) trước, khi đó mới được chỉnh sửa. Điều này để bảo vệ khóa đang có học viên học.

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Xem & Quản lý khóa**
- **FR-A1** Quản trị viên **xem danh sách khóa** với các cột: tên khóa, trạng thái, ngày nhập, có nút **"Xem / Sửa"** để mở trang chi tiết.
- **FR-A2** Trang chi tiết **hiển thị toàn bộ cấu trúc khóa** (Chặng → Chuyên đề → Bài học → media/bài tập) dưới dạng cây xếp tầng, dễ đọc.

**Nhóm B — Chỉnh sửa nội dung**
- **FR-B1** Khi khóa ở **Nháp** hoặc **Sẵn sàng**, quản trị viên **có thể sửa** tiêu đề, mô tả, ảnh bìa của khóa.
- **FR-B2** Quản trị viên **có thể sửa nội dung** từng Bài học (tiêu đề, lý thuyết).
- **FR-B3** Quản trị viên **có thể thêm, xoá, di chuyển** các Chặng, Chuyên đề, Bài học, media, bài tập (chỉ khi khóa ở Nháp/Sẵn sàng).
- **FR-B4** Sau mỗi lần **lưu thay đổi**, hệ thống **tự động kiểm tra lỗi** (re-validate) và báo cáo để quản trị viên khắc phục.

**Nhóm C — Quản lý trạng thái**
- **FR-C1** Quản trị viên **chuyển trạng thái khóa** theo quy tắc máy trạng thái ở §5 (nút hiển thị tùy trạng thái hiện tại).
- **FR-C2** Khi chuyển sang **Sẵn sàng** hoặc **Xuất bản**, hệ thống **kiểm tra tính hợp lệ**; nếu có lỗi → cản chuyển và hiển thị danh sách lỗi cụ thể (**đường dẫn trong khóa** của từng lỗi).
- **FR-C3** Khi chuyển từ **Xuất bản** về **Sẵn sàng** (gỡ xuất bản), yêu cầu **xác nhận** trước để tránh vô tình; thông báo cảnh báo cho biết: "Khóa sẽ không xuất hiện cho học viên. Bạn có thể sửa lại ở trạng thái Sẵn sàng."

**Nhóm D — Kiểm toán & Lịch sử**
- **FR-D1** Hệ thống **ghi lại lịch sử** mỗi lần quản trị viên sửa khóa hoặc chuyển trạng thái (ai, khi nào, làm gì) để kiểm toán.

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Bảo vệ khóa Xuất bản):** Khóa ở trạng thái Xuất bản **không thể sửa** trực tiếp; phải gỡ xuất bản trước. Điều này đảm bảo học viên không bị mất nội dung đang học.
- **NFR-2 (Kiểm tra chất lượng tự động):** Trước mỗi lần xuất bản, hệ thống **bắt buộc kiểm tra** tất cả lỗi có thể (tham chiếu, cấu trúc…) — **chỉ cho phép xuất bản khóa hợp lệ**.
- **NFR-3 (Tái dùng hạ tầng):** Sử dụng lại pipeline kiểm tra lỗi từ Phase 1 (import) — không cần xây dựng logic từ đầu.
- **NFR-4 (Không sửa nội dung khóa chuẩn):** Chỉ **đọc & sửa** nội dung **khóa admin tạo** (từ import); **không ảnh hưởng** đến nội dung khóa chuẩn (lộ trình cố định).

## 8. Luồng người dùng (tóm tắt)

1. Quản trị viên **mở trang Admin** → **xem danh sách khóa**.
2. Chọn khóa **muốn sửa** → mở **trang chi tiết**.
3. **Chế độ Nháp/Sẵn sàng:** sửa nội dung, thêm/xoá Bài học, lưu → hệ thống kiểm tra lỗi tự động.
4. Khi khóa **hoàn chỉnh**, chuyển sang **Sẵn sàng** → hệ thống kiểm tra lần cuối; nếu không có lỗi, lưu trạng thái.
5. Khi **sẵn sàng ra mắt**, chuyển sang **Xuất bản** → khóa hiển thị cho học viên.
6. Nếu muốn **sửa khóa đã xuất bản**, phải **gỡ xuất bản** trước (quay về Sẵn sàng), sửa, rồi xuất bản lại.
7. Khi **kết thúc vòng đời khóa**, chuyển sang **Lưu trữ** → khóa bị ẩn (không xoá).

## 9. Tiêu chí hoàn thành (nghiệm thu)

- Quản trị viên **có thể xem chi tiết** toàn bộ cấu trúc một khóa đã nhập.
- **Chế độ sửa** hoạt động: thay đổi tiêu đề, nội dung Bài học, thêm/xoá media/bài tập được lưu.
- **Kiểm tra lỗi** tự động chạy khi lưu; báo cáo lỗi cụ thể (**đường dẫn trong khóa**) nếu có.
- **Máy trạng thái** hoạt động đúng: chỉ cho phép chuyển hợp lệ, cản lại những chuyển không cho phép.
- **Kiểm tra trước xuất bản:** trước khi sang Sẵn sàng/Xuất bản, hệ thống kiểm tra; nếu lỗi → cản và báo cáo.
- **Gỡ xuất bản** yêu cầu xác nhận; khóa có thể sửa lại khi ở Sẵn sàng.
- Khóa **Xuất bản không thể sửa** trực tiếp (chỉ đọc).
- **Lịch sử kiểm toán** ghi lại mỗi thao tác.

## 10. Ngoài phạm vi

- Tự sinh nội dung/câu hỏi mới bằng AI.
- Lập lịch học chi tiết (ngày, giờ, tuần).
- Đổi slug (định danh) khóa.
- Quản lý phiên bản / clone khóa.
- Tối ưu hóa nâng cao (A/B test thuật toán chọn nội dung).

## 11. Thuật ngữ (cho người không chuyên)

- **Khóa học:** một bộ tài liệu toàn bộ cấu trúc (Chặng, Chuyên đề, Bài học).
- **Chặng:** một nấc trình độ trong khóa (ví dụ: "A2→B1" là từ trình độ A2 lên B1).
- **Chuyên đề:** một mảng kỹ năng trong một Chặng (ví dụ: "Ngữ pháp", "Nghe").
- **Bài học:** đơn vị nhỏ nhất — một bài cụ thể (ví dụ: "Bài 1: Present Simple").
- **Media:** tài liệu đa phương tiện như video, audio.
- **Bài tập:** các câu hỏi/tương tác để học viên luyện.
- **Tham chiếu (reference):** liên kết đến bài tập có sẵn (ví dụ: bài tập Nghe IPA).
- **Nháp (Draft):** khóa chưa hoàn chỉnh, chỉ được sửa, không hiển thị cho học viên.
- **Sẵn sàng (Ready):** khóa đã kiểm tra lỗi, sẵn sàng xuất bản; vẫn có thể sửa.
- **Xuất bản (Published):** khóa đang hoạt động, học viên có thể truy cập; không sửa trực tiếp.
- **Lưu trữ (Archived):** khóa đã kết thúc, bị ẩn khỏi học viên; có thể khôi phục.
- **Re-validate (kiểm tra lỗi):** quá trình tự động kiểm tra tính hợp lệ của khóa trước khi xuất bản.
- **Máy trạng thái:** quy tắc định sẵn cho phép những chuyển trạng thái nào.
- **Lịch sử kiểm toán:** ghi chép ai đã làm gì và khi nào, để theo dõi.

---

## Câu hỏi mở

Hiện tại chưa rõ:
1. **Giới hạn độ phức tạp khóa:** Khóa có thể có bao nhiêu Chặng, Chuyên đề, Bài học tối đa? Có cần giới hạn để tránh UI chậm?
2. **Phân quyền chi tiết:** Có những loại quản trị viên khác nhau (chỉ xem / sửa / xuất bản) không? Hay tất cả quyền `coursecontent:write` đều có cùng quyền hạn?
3. **Thông báo cho học viên:** Khi khóa bị gỡ xuất bản (chuyển từ Xuất bản → Sẵn sàng), có gửi thông báo cho học viên đang học không?
4. **Backup/Khôi phục:** Có lưu lịch sử phiên bản khóa (ví dụ: khôi phục thay đổi cũ) không, hay chỉ ghi lại lịch sử kiểm toán (ai làm gì)?
