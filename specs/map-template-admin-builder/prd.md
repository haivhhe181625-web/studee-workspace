<!-- Tiếng Việt — PRD/SRS cho "Công cụ Admin soạn Bản đồ học gamified + Tự chọn bản đồ". Viết cho người KHÔNG chuyên kỹ thuật.
     Bản kỹ thuật: specs/map-template-admin-builder/spec.md, design.md. -->

# PRD — Công cụ Admin soạn Bản đồ học Gamified + Tự chọn bản đồ

- **Ngày:** 2026-08-03 · **Trạng thái:** Hoàn thành · **Đối tượng đọc:** Founder / Giáo vụ / BA / Đội học thuật
- **Bản kỹ thuật/chi tiết:** `specs/map-template-admin-builder/spec.md` (quyết định §5), `design.md` (lỗi code), `acceptance.md` (kiểm thử)
- **Phụ thuộc:** Sử dụng ngân hàng câu hỏi AssessmentQuestion **đã có**.

> Tài liệu mô tả **chức năng sẽ giải quyết vấn đề nào & học viên được gì** bằng ngôn ngữ dễ hiểu. Chi tiết kỹ thuật đọc kèm tài liệu spec/design.

---

## 1. Mục đích & bối cảnh

**Vấn đề hiện tại:**
- Đội Content phải **lên terminal** (dev seed) mỗi lần tạo bản đồ gamified mới → mất thời gian, khó scale, dễ sai.
- Bản đồ hiện dùng **câu hỏi giả (fake data)** từ database seed → không phải câu hỏi curated thực từ ngân hàng → học viên chơi với nội dung không tối ưu.
- Admin phải **hardcode một bản đồ cố định** (demo-foundation) cho tất cả học viên mới → không căn cứ theo trình độ hay chương trình của người đó → học viên nhận bản đồ không phù hợp.

**Mục tiêu:**
Cho phép **Đội Content (Admin)** tạo & soạn bản đồ học gamified **trực tiếp qua giao diện** mà không cần dev terminal. Bản đồ được tạo từ một **tập hợp node (chặng/chuyên đề)** có kết nối logic (edge/yêu cầu), mỗi node **gán các câu hỏi thực từ ngân hàng** (thay vì fake seed). Khi một **học viên mới bắt đầu**, hệ thống **tự chọn bản đồ phù hợp** với trình độ & chương trình của người đó — **không còn hardcode**, tối ưu hóa trải nghiệm.

## 2. Phạm vi

**TRONG phạm vi:**
1. **Công cụ soạn bản đồ** — Admin tạo & sửa template (tập hợp node/edge/cấu hình).
2. **Gán câu hỏi thực** — Admin chọn câu hỏi từ ngân hàng đã curated gán vào node dạng MCQ (không dùng fake seed).
3. **Phát hành & kiểm soát** — Admin duyệt (publish) bản đồ mới để học viên tiếp cận; hệ thống kiểm tra lỗi (node trống, vòng logic vô hạn…).
4. **Tự chọn bản đồ** — Khi học viên mới vào, hệ thống **tự so khớp trình độ + chương trình** của người đó để chọn bản đồ đúng từ kho templates.
5. **Sửa nhanh nội dung** — Admin sửa câu hỏi, vị trí, ngưỡng hoàn thành của node được **cập nhật ngay** tới học viên đang chơi (live update, không cần phát hành lại).

**NGOÀI phạm vi (để sau):**
- Tự sinh bản đồ từ khóa học bằng AI; lập lịch học chi tiết theo ngày; so sánh A/B thuật toán chọn bản đồ.
- Tính năng undo / export / import bản đồ.
- Xoá cứng dữ liệu (chỉ soft archive).
- **Không sửa nội dung** khóa cố định — bản đồ chỉ **tái sử dụng & bốc** từ kho khóa.

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Đội Content (Admin soạn bản đồ)** | Tạo template mới, soạn node/edge, gán câu hỏi thực, sửa nhanh nội dung mà không cần dev terminal. |
| **Platform Admin (quản lý phát hành)** | Duyệt template, phát hành (publish) bản đồ mới để tất cả học viên mới nhận; xoá/lưu trữ bản đồ cũ. |
| **Học viên (phía người dùng cuối)** | Khi vào ứng dụng, tự động nhận bản đồ phù hợp trình độ & chương trình của mình; chơi bản đồ với câu hỏi thực. |

## 4. Tổng quan

### A. Bản đồ học (CourseMapTemplate)
Một **bản đồ học** là một tập hợp các **node** (chặng / chuyên đề) được **kết nối logic** bằng edges (yêu cầu). Mỗi bản đồ có:
- **Mã định danh duy nhất** (templateKey) để hệ thống nhận diện, ví dụ: "course-ielts-foundation", "toeic-b1-b2".
- **Phiên bản** (version) — khi sửa cấu trúc, hệ thống tạo version mới để học viên cũ giữ version cũ, học viên mới nhận version mới.
- **Trạng thái** — draft (soạn), published (phát hành), archived (lưu trữ).
- **Phạm vi trình độ** (CEFR range) — bản đồ này phù hợp từ A1 tới B1, hoặc B1 tới B2 (nếu null = phù hợp tất cả trình độ, dùng làm fallback).
- **Chương trình** (program tag) — bản đồ này dành cho IELTS, TOEIC, hay không phân biệt (null = universal).

### B. Các loại node (node kind)
Mỗi **node** (chặng nhỏ) đại diện cho một **mảng học** có một hay nhiều hoạt động:
- **CORE** — bài cơ bản, phải làm để tiến tới bài khác.
- **PRACTICE** — bài luyện tập, là phần phụ (không bắt buộc nhưng được khuyến khích).
- **CHECKPOINT** — bài kiểm tra để xác nhận đã thông hiểu.
- **REMEDIATION** — bài ôn lại (dùng khi phát hiện học viên đang yếu ở chỗ nào).
- **SIDE_QUEST** — bài bonus, thêm kỹ năng hoặc kỹ năng mềm.

Mỗi node **gắn một kỹ năng chính** (skill) — Nghe, Đọc, Viết, Nói, Ngữ pháp, Từ vựng — để hệ thống biết nó dạy cái gì.

### C. Gán câu hỏi vào node MCQ
Với node **dạng MCQ** (trắc nghiệm), Admin **chọn danh sách câu hỏi thực** từ ngân hàng (AssessmentQuestion) gán vào node. 
- Trước đây: dùng câu hỏi giả (seed).
- Từ giờ: gán **câu hỏi curated thực**, học viên gặp nội dung chất lượng cao.
- Hệ thống **kiểm tra** rằng những câu được chọn phải active (còn dùng), không phải riêng của một center (chỉ dùng câu chung).

### D. Sửa nhanh & phát hành
- **Sửa nhanh (live-safe):** Sửa câu hỏi, vị trí, ngưỡng hoàn thành → cập nhật **ngay** tới học viên (không cần phát hành lại).
- **Sửa cấu trúc (structural):** Thêm/xoá node, đổi yêu cầu → tạo **version mới**. Học viên cũ (đã chơi) giữ version cũ, học viên mới nhận version mới (tránh phá tiến độ học viên).
- **Phát hành (publish):** Admin duyệt, kiểm tra lỗi (node trống, vòng lặp…), sau đó nhấn publish để bản đồ chính thức tới học viên mới.

### E. Tự chọn bản đồ theo trình độ & chương trình
Khi **học viên mới** vào ứng dụng, hệ thống **tự động chọn** một bản đồ phù hợp:
- Ví dụ: Học viên mục tiêu IELTS 6.0, trình độ hiện tại B1 → hệ thống tìm bản đồ program="ielts", cefrRange cover B1 (B1→B2 chẳng hạn) → tự động gán bản đồ đó.
- Nếu không tìm thấy → dùng bản đồ fallback (ví dụ "demo-foundation") để học viên vẫn có gì đó học (không bỏ rơi).
- Học viên cũ (đã có bản đồ) **không bị đổi** — tiếp tục bản đồ cũ của mình.

---

## 5. Quy tắc soạn bản đồ (dễ hiểu)

### Cấu trúc cơ bản
Mỗi **bản đồ** bao gồm:
- **Danh sách node** — mỗi node có: loại (kind), kỹ năng chính (skill), danh sách yêu cầu (cái nào phải làm trước).
- **Danh sách edge** — mỗi edge nối 2 node, thể hiện "A phải làm trước B".
- **Kỹ năng & bài luyện phát âm** — tùy chọn chi tiết hơn nếu node là luyện phát âm riêng.

### Quy tắc kiểm tra tự động (hệ thống kiểm soát)
Khi Admin soạn:
1. **Node phải có mã duy nhất** — không có 2 node cùng mã trong 1 bản đồ.
2. **Yêu cầu phải có node tương ứng** — nếu node A yêu cầu node B, B phải tồn tại.
3. **Edge phải gương lại yêu cầu** — nếu node B yêu cầu node A, phải có edge A→B (nếu không, hệ thống báo lỗi).
4. **Vòng lặp yêu cầu không được phép khi phát hành** — ví dụ A→B→C→A là vòng (được cảnh báo ở bản nháp, bị chặn khi phát hành).
5. **MCQ phải có câu hỏi** — nếu node là MCQ, phải gán ít nhất 1 câu hỏi để publish (bản nháp cho phép trống để dỡi lại sau).
6. **Câu hỏi phải hợp lệ** — câu phải active (còn dùng), không phải riêng của center (chỉ dùng câu chung), phải là loại MCQ.

### Ví dụ cụ thể
Bản đồ "IELTS Listening A2→B1":
```
Node 1: CORE, Nghe chủ đề, MCQ [câu 1, 2, 3]
Node 2: CHECKPOINT, Kiểm tra nghe, MCQ [câu 4, 5, 6] — yêu cầu: Node 1
Edge: Node 1 → Node 2
```
→ Học viên phải làm Node 1 xong mới được Node 2; Node 2 gán câu thực từ ngân hàng.

---

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Quản lý Template (Admin soạn bản đồ)**

- **FR-A1** Admin có thể **tạo template mới** bằng cách cung cấp mã (templateKey), phạm vi trình độ (CEFR range, ví dụ A1→B1), và chương trình (ví dụ IELTS). Hệ thống lưu template ở trạng thái "nháp" (draft) để Admin soạn tiếp.
- **FR-A2** Admin có thể **sửa nội dung bản đồ** (thêm/xoá/sửa node, thay đổi yêu cầu). Khi soạn bản nháp, sửa được ngay (in-place). Khi bản đồ đã phát hành (published), **chỉ có thể sửa câu hỏi/vị trí/ngưỡng** được cập nhật ngay; **sửa cấu trúc (node/edge) phải tạo version mới** để học viên cũ không bị ảnh hưởng.
- **FR-A3** Admin **gán câu hỏi thực** từ ngân hàng vào node MCQ bằng cách chọn danh sách câu trong một bộ lọc. Hệ thống kiểm tra câu hỏi hợp lệ (active, loại MCQ, không riêng center).
- **FR-A4** Admin có thể **xem danh sách template** với lọc theo trạng thái (nháp / phát hành / lưu trữ), tìm kiếm theo tên. Xem chi tiết từng template gồm toàn bộ node, edge, câu hỏi.
- **FR-A5** Admin có thể **xoá (lưu trữ) template** — bản đồ không xoá cứng mà chuyển sang trạng thái "archived" (lưu trữ). Không được xoá bản đồ cuối cùng đã phát hành của một mã (để tránh học viên cũ mất bản đồ).

**Nhóm B — Kiểm tra & Phát hành (Platform Admin phát hành bản đồ)**

- **FR-B1** Platform Admin **kiểm tra chất lượng** template nháp trước phát hành. Hệ thống tự động báo lỗi (node trống, vòng lặp yêu cầu, MCQ không có câu…) để Admin sửa.
- **FR-B2** Platform Admin **phát hành (publish) template** từ trạng thái nháp. Sau khi publish, bản đồ tới **tất cả học viên mới**. Nếu có lỗi, publish bị chặn, yêu cầu fix trước.
- **FR-B3** Khi Admin **tạo version mới** của bản đồ (do sửa cấu trúc), nó bắt đầu ở trạng thái nháp. Học viên cũ (đã chơi) **vẫn giữ version cũ**, học viên mới nhận **version mới** khi phát hành.
- **FR-B4** Platform Admin có **quyền đặc biệt** để xoá tiến độ học viên (wipe progress) khi test — loại bỏ tất cả dữ liệu học viên liên quan bản đồ này (chỉ dành mục đích test, bị chặn ở production).

**Nhóm C — Tự chọn bản đồ (Học viên mới)**

- **FR-C1** Khi **học viên mới vào**, hệ thống **tự động chọn bản đồ phù hợp** dựa trên mục tiêu & trình độ của họ (ví dụ: program=IELTS, trình độ=B1 → chọn bản đồ IELTS cover B1).
- **FR-C2** Nếu **không có bản đồ matching**, hệ thống dùng **fallback (bản đồ mặc định)** để học viên vẫn có gì đó học. Nếu cả fallback không có, báo lỗi.
- **FR-C3** Học viên **cũ (đã pinned version cũ)** không bị ảnh hưởng khi Admin đổi logic chọn bản đồ hoặc phát hành version mới — tiếp tục bản đồ cũ của mình.

**Nhóm D — Cảnh báo & Health Check**

- **FR-D1** Hệ thống **cảnh báo trước publish** nếu phát hiện: node MCQ không có câu, vòng lặp yêu cầu, node trống (loại MCQ nhưng không có cấu hình).
- **FR-D2** Hệ thống **báo sức khỏe bản đồ** — hiển thị mỗi node có bao nhiêu câu hỏi, liệu có đủ số lượng tối thiểu không (phòng trường hợp sau này câu bị retire).

---

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Quyền hạn):** Chỉ Đội Content (quyền `write`) được tạo/sửa template nháp. Chỉ Platform Admin (quyền `manage`) được phát hành & xoá. Học viên không được phép xem template draft hoặc câu hỏi gốc (chỉ xem nội dung bản đồ khi chơi).
- **NFR-2 (Bảo mật dữ liệu):** Câu hỏi gán vào template phải từ **ngân hàng chung** (không riêng center). Nếu Admin chọn câu riêng, hệ thống tự động lọc/từ chối.
- **NFR-3 (Tốc độ sửa):** Sửa nhanh (câu hỏi, vị trí, ngưỡng) trên bản đồ đã phát hành phải **cập nhật ngay** tới học viên chơi (không delay, không cần phát hành lại).
- **NFR-4 (Bảo vệ học viên):** Học viên cũ không bị mất tiến độ khi Admin sửa bản đồ. Nếu Admin sửa cấu trúc → tạo version mới, học viên cũ giữ version cũ.
- **NFR-5 (Kiểm tra tự động):** Toàn bộ các lỗi cấu trúc (duplicate node, yêu cầu không tồn tại, vòng lặp, câu hỏi không hợp lệ…) phải được **phát hiện tự động** khi sửa hoặc publish, không cần Admin phát hiện thủ công.
- **NFR-6 (Hoà nhập với hệ thống):** Bản đồ chỉ **sử dụng lại** câu hỏi & node từ kho khóa có sẵn, **không tự sinh** nội dung. Tiến độ học viên chơi bản đồ được **ghi nhận chung** với khóa cố định (nếu cùng node).

---

## 8. Luồng người dùng (tóm tắt)

### Luồng Admin soạn bản đồ:
1. Admin vào công cụ → **Tạo template mới** (mã + trình độ + chương trình) → bản đồ ở trạng thái nháp.
2. Admin **soạn node**: thêm chặng (CORE/PRACTICE/CHECKPOINT…), gắn kỹ năng, thiết lập yêu cầu.
3. Admin **gán câu hỏi**: cho node dạng MCQ, chọn danh sách câu từ ngân hàng (hệ thống kiểm tra hợp lệ).
4. Admin **sửa & kiểm tra**: hệ thống báo lỗi (node trống, vòng lặp…) nếu có.
5. Admin **lưu nháp**, sau đó yêu cầu Platform Admin phát hành.

### Luồng Platform Admin phát hành:
1. Platform Admin **xem danh sách template nháp** chờ duyệt.
2. Platform Admin **xem chi tiết**, kiểm tra lỗi & cảnh báo.
3. Nếu OK → **Publish** (bản đồ tới tất cả học viên mới). Nếu lỗi → yêu cầu Đội Content fix.

### Luồng học viên mới:
1. Học viên **vào ứng dụng, khai báo mục tiêu & trình độ**.
2. Hệ thống **tự chọn bản đồ phù hợp** (program + trình độ).
3. Học viên **nhìn thấy bản đồ** với các chặng, bắt đầu chơi.

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

- Đội Content có thể **tạo template mới**, soạn node/edge (no terminal script).
- Admin **gán câu hỏi thực** từ ngân hàng vào node (không dùng fake seed nữa).
- **Sửa nhanh** (câu hỏi, vị trí) trên bản đồ published → cập nhật ngay học viên (không 409 lock).
- **Sửa cấu trúc** (node/edge) trên published → tạo version mới, học viên cũ giữ version cũ (not retroactive).
- Hệ thống **phát hiện tự động** lỗi (node trống MCQ, vòng lặp, câu hỏi invalid).
- Khi **publish**, hệ thống kiểm tra hợp lệ, chặn nếu lỗi; nếu OK → bản đồ tới tất cả học viên mới.
- **Học viên mới tự động nhận** bản đồ đúng per mục tiêu/trình độ (không hardcode demo-foundation).
- **Học viên cũ** không bị ảnh hưởng (pinned version, không retroactive).

---

## 10. Ngoài phạm vi

- Tự sinh bản đồ từ khóa học bằng AI.
- Lập lịch học chi tiết theo ngày/giờ.
- So sánh nhiều thuật toán chọn bản đồ (A/B test).
- Undo / export / import bản đồ.
- Xoá cứng dữ liệu (chỉ soft archive).
- Sửa nội dung khóa cố định (chỉ đọc & bốc).

---

## 11. Thuật ngữ (cho người không chuyên)

- **Template / Bản đồ học:** Một cấu trúc node/edge định nghĩa một đường học gamified cho một mục tiêu nào đó (ví dụ IELTS A2→B1).
- **TemplateKey / Mã bản đồ:** Mã duy nhất để nhận diện bản đồ (ví dụ "course-ielts-foundation").
- **Version / Phiên bản:** Số phiên bản bản đồ (v1, v2…). Khi sửa cấu trúc, tăng version; học viên cũ giữ version cũ, mới nhận version mới.
- **Status / Trạng thái:** draft (soạn), published (phát hành), archived (lưu trữ).
- **Node / Chặng:** Một mảng học nhỏ trong bản đồ (ví dụ một buổi luyện nghe).
- **Edge / Kết nối / Yêu cầu:** Liên kết giữa 2 node, chỉ "cái nào phải làm trước".
- **CEFR / Trình độ:** Khung trình độ A1, A2, B1, B2… (tiêu chuẩn quốc tế).
- **CEFR Range / Phạm vi trình độ:** Bản đồ này phù hợp từ mức nào đến mức nào (ví dụ A2→B1 hoặc A1→B2).
- **Program / Chương trình:** Loại chương trình mà bản đồ phục vụ (IELTS, TOEIC, hay không phân biệt/universal).
- **MCQ / Trắc nghiệm:** Câu hỏi dạng chọn đáp án.
- **ItemIds / Danh sách câu hỏi:** Các mã câu hỏi thực từ ngân hàng gán vào node MCQ.
- **Bank / Ngân hàng câu hỏi:** Kho tập trung chứa tất cả câu hỏi curated của hệ thống (AssessmentQuestion).
- **Centerforbidden / Riêng center:** Câu hỏi riêng của một center nào đó (không dùng được, chỉ dùng câu chung).
- **Draft / Nháp:** Trạng thái bản đồ đang soạn, chưa phát hành.
- **Published / Phát hành:** Trạng thái bản đồ đã được duyệt, tất cả học viên mới nhận.
- **Archived / Lưu trữ:** Trạng thái bản đồ đã bỏ dùng nhưng giữ dữ liệu.
- **Pinned Version / Phiên bản cố định:** Học viên cũ được "ghim" vào một version cụ thể (v1 chẳng hạn), không đổi khi admin phát hành version mới.
- **Fallback / Bản đồ mặc định:** Bản đồ được dùng khi hệ thống không tìm thấy match (ví dụ "demo-foundation").
- **Health Check / Kiểm tra sức khỏe:** Xác minh bản đồ có lỗi (node trống, câu bị retire…).
- **Structural / Cấu trúc:** Các sửa đổi về node/edge/yêu cầu (sửa cấu trúc phải tạo version mới).
- **Live-safe / An toàn sửa nhanh:** Các sửa đổi về câu hỏi/vị trí/ngưỡng (sửa được ngay, không ảnh hưởng học viên).
