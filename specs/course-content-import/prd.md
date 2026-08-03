<!-- Tiếng Việt — PRD/SRS cho tính năng "Nhập cấu trúc Khóa học toàn bộ (Phase 1 — Ingestion)". Viết cho người KHÔNG chuyên kỹ thuật.
     Bản tóm tắt C-level: srs/executive-summary-course-structure.md
     Bản kỹ thuật/thiết kế: spec.md, design.md -->

# PRD — Nhập cấu trúc Khóa học 4 tầng toàn bộ (Import Phase 1)

- **Ngày:** 2026-07-16 · **Trạng thái:** Sẵn sàng duyệt · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật/thiết kế:** `spec.md`, `design.md` (các quyết định đã chốt)
- **Bản tóm tắt C-level:** `srs/executive-summary-course-structure.md` (overview 1 trang)

> Tài liệu mô tả **tính năng sẽ có** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `spec.md` + `design.md`.

---

## 1. Mục đích & bối cảnh

Hiện tại, đội học thuật chuẩn bị một khóa học (có lộ trình, chặng, chuyên đề, bài học) bằng file Excel rồi **không có cách nào** đưa nó vào hệ thống một cách tự động. Họ phải yêu cầu dev tạo tay, từng mục một, hoặc dùng các công cụ quản trị tạm bợ không được hỗ trợ chính thức.

**Mục tiêu:** xây một **kênh nhập dữ liệu chính thức** cho đội học thuật: họ chuẩn bị khóa học theo một mẫu file chuẩn (file Excel), tải lên một trang nhập dữ liệu trên hệ thống admin, và **toàn bộ cấu trúc khóa học được lưu tự động vào kho nội dung** — sẵn sàng cho học viên xem và học trong các giai đoạn tiếp theo (Phase 2, 3).

Điều này giúp:
- **Đội học thuật** tự chủ: không phụ thuộc dev, tiết kiệm thời gian, có thể phát hành khóa theo lô.
- **Hệ thống** có kho nội dung khóa học chuẩn hóa, sẵn sàng mở rộng tính năng (cá nhân hóa lộ trình, theo dõi tiến độ, v.v.).

---

## 2. Phạm vi

### **TRONG phạm vi**

1. **Trang nhập dữ liệu trên Admin** — cho phép người quản lý nội dung tải file Excel chứa cấu trúc khóa học.
2. **Xử lý import** — hệ thống đọc file, kiểm tra lỗi (cấu trúc, tham chiếu bài tập, kiểu dữ liệu), rồi lưu toàn bộ cấu trúc vào kho nội dung khóa học **hoàn toàn hoặc không tạo gì cả** (all-or-nothing).
3. **Quản lý khóa đã import** — liệt kê, xem chi tiết, xoá và import lại khóa từ trang admin.
4. **Hỗ trợ nội dung học đa dạng** — mỗi bài học có thể chứa lý thuyết (văn bản), video/audio (link URL), hoặc bài tập tương tác (phát âm IPA, hội thoại AI).
5. **Metadata cấu trúc** — lưu thông tin nâng cấp (từ A2 lên B1), phân loại kỹ năng (ngữ pháp, phát âm, từ vựng, nghe, đọc, viết, nói).
6. **Mẫu file & hướng dẫn** — cung cấp file Excel mẫu để đội học thuật dễ biết cách điền.

### **NGOÀI phạm vi**

- Hiển thị khóa học cho học viên, logic mở khóa bài học tuần tự (Phase 2).
- Theo dõi tiến độ học, cộng điểm, tính năng streak (Phase 3).
- Luồng soạn khóa thủ công kéo-thả trên giao diện CMS (giải pháp thay thế — để sau nếu cần).
- Tạo/sửa nội dung bài tập IPA/Talk/Quiz — Phase 1 chỉ **tham chiếu** ID bài tập đã có sẵn.
- Upload/host video, audio — đội học thuật tự host, hệ thống chỉ lưu link URL.

---

## 3. Người dùng

| Vai trò | Nhu cầu |
|---|---|
| **Đội học thuật / Admin nội dung** | Chuẩn bị khóa học (file Excel có sẵn cấu trúc 4 tầng + các bài tập), tải lên trang nhập, xem kết quả (thành công hay lỗi), sửa file nếu lỗi rồi thử lại. |
| **Người quản lý nội dung** | Quản lý danh sách khóa đã import, xem chi tiết từng khóa, xoá khóa cũ để import lại bản mới, kiểm soát quyền ai có thể thao tác. |
| **Đội kỹ thuật (gián tiếp)** | Có kho nội dung khóa học ở trạng thái "sẵn sàng" sau import, dùng để xây dựng các tính năng tiếp theo (hiển thị cho học viên, tính năng cá nhân hóa, v.v.). |

---

## 4. Tổng quan

Tính năng nhập khóa học **chuyển đổi một file Excel thành một cây cấu trúc lưu trong hệ thống** như sau:

- **Đầu vào:** 1 file Excel chứa tất cả thông tin của 1 khóa học
  - Cột 1 (Lộ trình): tên khóa, mục tiêu, ảnh đại diện
  - Cột 2 (Chặng): tên giai đoạn, từ mức nào đến mức nào (vd A2→B1), qui tắc mở khóa
  - Cột 3 (Chuyên đề): tên chuyên đề, kỹ năng (ngữ pháp, phát âm, …)
  - Cột 4 (Bài học): tên bài, lý thuyết, video/audio, bài tập (ID phát âm/hội thoại AI)

- **Quá trình xử lý:**
  1. **Kiểm tra cấu trúc:** file có đúng định dạng, có đủ tầng bắt buộc, liên kết cha-con hợp lệ không?
  2. **Kiểm tra tham chiếu:** các ID bài tập (phát âm, hội thoại) có tồn tại trong hệ thống không?
  3. **Kiểm tra nội dung:** bài học không rỗng, URL media hợp lệ, metadata (mức trình độ, kỹ năng) đúng không?
  4. **Lưu toàn bộ hoặc không:** nếu có lỗi → từ chối + báo lỗi cụ thể; không có lỗi → lưu cả cây vào kho nội dung.

- **Đầu ra:** 
  - Thành công: tóm tắt số Chặng/Chuyên đề/Bài học đã tạo, khóa ở trạng thái "sẵn sàng"
  - Lỗi: danh sách lỗi chỉ rõ dòng/mục nào sai, sai gì (để đội học thuật tự sửa file)

---

## 5. Quy tắc kiểm tra & lưu dữ liệu

Hệ thống áp dụng các quy tắc dưới đây khi xử lý file import:

### **Về cấu trúc phân tầng**
- **Bắt buộc:** mỗi **Khóa** phải có ≥1 **Chặng**; mỗi **Chặng** phải có ≥1 **Chuyên đề**; mỗi **Chuyên đề** phải có ≥1 **Bài học**.
- **Liên kết cha-con:** mỗi Bài học phải thuộc 1 Chuyên đề, mỗi Chuyên đề phải thuộc 1 Chặng, mỗi Chặng phải thuộc 1 Khóa.
- **Không trùng lặp:** trong cùng 1 cấp độ (vd cùng Chặng), định danh không được trùng nhau.

### **Về nội dung bài học**
- **Không rỗng:** mỗi bài học phải có **ít nhất một** trong các loại nội dung sau:
  - Lý thuyết (văn bản, markdown)
  - Media (video, audio) — tối thiểu 1 link
  - Bài tập (phát âm, hội thoại AI)
- **Ví dụ hợp lệ:** bài toàn lý thuyết (không video/tập), bài toàn video (không lý thuyết/tập), bài có cả lý thuyết + video + tập.

### **Về bài tập tham chiếu**
- **Chỉ tham chiếu, không tạo:** khi file trỏ tới 1 bài tập phát âm hay hội thoại, bài tập đó **phải tồn tại sẵn** trong hệ thống và **phải ở trạng thái đã xuất bản** (không được ở trạng thái nháp/lưu trữ).
- **Các loại hỗ trợ Phase 1:** phát âm (IPA), hội thoại AI (Talk). Quiz (trắc nghiệm) hoãn sang Phase 2.

### **Về metadata cấu trúc**
- **Mức trình độ Chặng:** lưu "từ A2 lên B1" (hoặc "IELTS 5.0→6.0" tự do). Hệ thống kiểm tra giá trị nằm trong khung CEFR (A1, A2, B1, B2, C1, C2) và chiều tăng hợp lệ.
- **Kỹ năng Chuyên đề:** lưu ngữ pháp, phát âm, từ vựng, nghe, đọc, viết, nói. Hệ thống kiểm tra từ danh sách cho phép.

### **Về media URL**
- **Chỉ URL hợp lệ:** video, audio, ảnh phải là link `http://` hoặc `https://` (CDN, YouTube, R2, v.v.). **Cấm** nhúng file Base64 hay `data:` URIs trực tiếp.
- **Kích thước file import:** giới hạn ≤5MB để xử lý nhanh chóng (khoảng ~5.000 bài học).

### **All-or-nothing (quan trọng)**
- **Nếu có bất kỳ lỗi nào** (cấu trúc, tham chiếu, metadata) → **không tạo bản ghi nào cả**. File hoàn toàn được chấp nhận hoặc hoàn toàn bị từ chối; không có "nhập được một phần".

---

## 6. Yêu cầu chức năng (SRS)

### **Nhóm A — Tảng import & xử lý**

- **FR-A1** Trang nhập trên Admin cho phép người quản lý nội dung **chọn file Excel** và **tải lên**. Hệ thống nhận file, kiểm tra, và báo kết quả (thành công hoặc danh sách lỗi).

- **FR-A2** Hệ thống **đọc & phân tích file Excel** theo mẫu chuẩn (11 cột: Lộ trình, Chặng, Chuyên đề, Bài học, Lý thuyết, Media, Bài tập, Metadata, v.v.).

- **FR-A3** Hệ thống **kiểm tra lỗi cấu trúc**: file có đầy đủ tầng, liên kết cha-con hợp lệ, không trùng lặp định danh, không rỗng không?

- **FR-A4** Hệ thống **kiểm tra tham chiếu bài tập**: mỗi ID phát âm/hội thoại mà file trỏ tới có **tồn tại** trong hệ thống, **ở trạng thái đã xuất bản**, không ở nháp hay lưu trữ?

- **FR-A5** Hệ thống **kiểm tra metadata**: mức trình độ (CEFR) trong khung, kỹ năng từ danh sách cho phép, URL media hợp lệ (không Base64), loại media (video/audio) hợp lệ?

- **FR-A6** Nếu **bất kỳ lỗi nào** → từ chối toàn bộ import, **không tạo bản ghi nào**; trả danh sách lỗi chỉ rõ vị trí (dòng) và nội dung sai.

- **FR-A7** Nếu **hợp lệ** → lưu toàn bộ cấu trúc (Chặng, Chuyên đề, Bài học) vào kho nội dung khóa học. Khóa ở trạng thái **"sẵn sàng"** — sẵn sàng cho Phase 2 hiển thị cho học viên.

- **FR-A8** Trả **kết quả tóm tắt** cho người dùng: số Chặng, Chuyên đề, Bài học đã tạo (nếu thành công) hoặc danh sách lỗi (nếu lỗi).

### **Nhóm B — Quản lý khóa đã import**

- **FR-B1** Trang danh sách khóa trên Admin hiển thị **tất cả khóa đã import**: tên, trạng thái, ngày tạo, số lượng Chặng/Chuyên đề/Bài học.

- **FR-B2** Trang xem chi tiết khóa hiển thị **toàn bộ cây cấu trúc** (Chặng → Chuyên đề → Bài học + các chi tiết như lý thuyết, video, bài tập, metadata).

- **FR-B3** Cho phép **xoá khóa** (chỉ những khóa chưa xuất bản) để giải phóng tên khóa, rồi **import lại** khóa mới có tên tương tự.

- **FR-B4** Chỉ **người quản lý nội dung có quyền** mới thao tác (import, xem, xoá); người khác bị từ chối.

### **Nhóm C — Hỗ trợ người dùng**

- **FR-C1** Cung cấp **file mẫu Excel** có sẵn dòng header (11 cột) + vài dòng ví dụ để đội học thuật download và điền.

- **FR-C2** Khi import **lỗi**, hiển thị **danh sách lỗi chi tiết** trên trang, không chỉ thông báo chung chung "thất bại".

---

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Bảo mật):** chỉ người quản lý nội dung (quyền `coursecontent:*`) mới được import; người khác bị từ chối. Không tin dữ liệu từ body request; lấy quyền từ phiên đăng nhập server.

- **NFR-2 (Bảo mật — multi-tenant):** toàn bộ khóa import Phase 1 thuộc về **nội dung platform dùng chung** (không riêng center nào), để chuẩn bị cho tính năng cá nhân hóa sau.

- **NFR-3 (Hiệu năng):** xử lý file đồng bộ (yêu cầu/trả lời một lần, không bất đồng bộ); file ≤5MB xử lý xong trong vài giây, trong thời gian chờ request bình thường.

- **NFR-4 (Audit):** mỗi lần import thành công được ghi lại trong nhật ký audit để kiểm soát ai import cái gì, khi nào.

- **NFR-5 (Tính nguyên tử):** all-or-nothing là ràng buộc dữ liệu quan trọng — hệ thống đảm bảo **không có trạng thái trung gian** (import được một phần).

---

## 8. Luồng người dùng (tóm tắt)

1. **Chuẩn bị file:** đội học thuật lấy file mẫu Excel, điền thông tin khóa (tên, mục tiêu), chặng (từ mức nào đến mức nào), chuyên đề (ngữ pháp, phát âm, …), bài học (lý thuyết, video, bài tập phát âm/hội thoại AI).

2. **Tải lên:** đăng nhập trang Admin nhập dữ liệu, chọn file Excel vừa chuẩn bị, bấm "Tải lên".

3. **Hệ thống kiểm tra:** xử lý file, kiểm tra cấu trúc/tham chiếu/metadata.

4. **Nhận kết quả:**
   - **Thành công:** "Khóa X đã nhập thành công: 3 Chặng, 12 Chuyên đề, 45 Bài học." Khóa sẵn sàng cho học viên xem ở giai đoạn sau.
   - **Lỗi:** "Lỗi ở dòng 25: bài tập phát âm ID 'ipa-201' không tồn tại." Đội học thuật sửa file rồi thử lại.

5. **Quản lý:** từ trang danh sách khóa, có thể xem chi tiết khóa, xoá khóa cũ (nếu chưa xuất bản), import lại bản mới.

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

Tính năng được coi là **hoàn thành** khi:

1. ✓ Tải file Excel hợp lệ → hệ thống lưu đầy đủ cấu trúc phân tầng, báo tóm tắt số tầng đã tạo **đúng** số lượng.

2. ✓ File sai cấu trúc (thiếu tầng, liên kết cha-con sai, bài học rỗng) → bị từ chối kèm **lỗi chỉ rõ vị trí** (dòng nào, sai cái gì).

3. ✓ File tham chiếu ID bài tập không tồn tại hoặc chưa xuất bản → bị từ chối kèm lỗi liệt kê **đúng ID nào bị lỗi**.

4. ✓ Bất kỳ lỗi nào → **không có bản ghi nào được tạo** (all-or-nothing đảm bảo).

5. ✓ User không đủ quyền → bị từ chối, không import được.

6. ✓ Kết quả (thành công/lỗi) **hiển thị rõ** trên trang Admin.

7. ✓ Cung cấp file mẫu Excel có sẵn dòng header + ví dụ để download.

8. ✓ Danh sách khóa đã import hiển thị đầy đủ thông tin (tên, trạng thái, ngày tạo, số tầng).

9. ✓ Xem chi tiết khóa hiển thị toàn bộ cây cấu trúc (từ Chặng tới Bài học).

10. ✓ Xoá khóa chưa xuất bản → giải phóng tên khóa, cho phép import lại.

---

## 10. Ngoài phạm vi

- **Hiển thị khóa cho học viên** — logic mở khóa bài học tuần tự, giao diện học tập → **Phase 2 (Lesson Wrapper)**.

- **Theo dõi tiến độ & streak** — tính mastery, ghi nhận hoàn thành bài, cộng điểm → **Phase 3**.

- **Soạn khóa thủ công trên CMS** — kéo-thả giao diện để dựng khóa → **Giải pháp thay thế, nếu cần**.

- **Tạo/sửa bài tập IPA/Talk/Quiz** — mỗi loại bài tập có công cụ quản lý riêng → **Phase 1 chỉ tham chiếu, không tạo**.

- **Upload/host media** — đội học thuật tự host video/audio, hệ thống chỉ lưu URL → **Luồng upload tập trung, nếu cần**.

---

## 11. Thuật ngữ (cho người không chuyên)

- **Khóa học (Course):** toàn bộ một chương trình học (vd "Khóa IELTS 5.0→6.0"), chứa tất cả Chặng, Chuyên đề, Bài học.

- **Chặng (Phase):** một giai đoạn hoặc "nấc" tiến bộ trong khóa (vd "từ A2 lên B1", "IELTS 5.0→6.0"). Học viên thường phải hoàn thành Chặng này rồi mới tiếp Chặng kế tiếp.

- **Chuyên đề (Module):** một nhóm bài học về một kỹ năng hoặc mảng cụ thể (vd "Ngữ pháp — Present Perfect", "Nghe — Nắm ý chính"). Nằm trong một Chặng.

- **Bài học (Lesson):** đơn vị học tập nhỏ nhất (vd "Bài 1: Giới thiệu Present Perfect"). Bài học chứa nội dung học (lý thuyết, video, audio) và/hoặc bài tập tương tác.

- **Lý thuyết (Theory):** nội dung văn bản, hướng dẫn, giải thích (viết hoặc markdown).

- **Media:** video, audio (link URL) bổ trợ nội dung học.

- **Bài tập tương tác:**
  - **Phát âm (IPA):** bài tập luyện phát âm từng âm tiếng Anh, AI chấm điểm.
  - **Hội thoại AI (Talk):** bài tập nói chuyện với AI nhân vật, AI phản hồi và chấm điểm.
  - **Quiz** (hoãn): trắc nghiệm, tự luận (sẽ hỗ trợ Phase 2).

- **Metadata:** thông tin mô tả (tên, mức trình độ, kỹ năng, ảnh đại diện) của Chặng, Chuyên đề, Bài học.

- **CEFR:** khung trình độ tiếng Anh quốc tế (A1, A2, B1, B2, C1, C2). Các mức tăng dần theo trình độ.

- **Trạng thái khóa:**
  - **Draft (Nháp):** khóa đang soạn, chưa hoàn thành.
  - **Ready (Sẵn sàng):** khóa hoàn thành import, sẵn sàng để học viên xem (Phase 2).
  - **Published (Xuất bản):** khóa đã phát hành cho học viên (Phase 2+).
  - **Archived (Lưu trữ):** khóa không còn sử dụng.

- **Import:** quá trình tải file Excel vào hệ thống, hệ thống tự động kiểm tra & lưu cấu trúc khóa học.

- **All-or-nothing:** nguyên tắc "toàn bộ hoặc không có gì" — nếu import lỗi, không tạo bản ghi nào; nếu hợp lệ, lưu toàn bộ cấu trúc.

- **Tham chiếu:** sự kết nối của bài học tới bài tập đã có sẵn (vd bài học "Phát âm /ɪ/" tham chiếu tới ID phát âm `ipa-45`).
