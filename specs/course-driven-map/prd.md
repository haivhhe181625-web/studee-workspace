<!-- Tiếng Việt — PRD/SRS cho "Khóa học điều phối Bản đồ game".
     Viết cho người KHÔNG chuyên kỹ thuật.
     Chi tiết kỹ thuật: specs/course-driven-map/spec.md + design.md (khi có).
     Quyết định chốt: spec.md §5. -->

# PRD — Khóa học điều phối Bản đồ game

- **Ngày:** 2026-08-03 · **Trạng thái:** Sẵn sàng triển khai · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Chi tiết kỹ thuật:** `specs/course-driven-map/spec.md` (quyết định §5) + `acceptance.md` (tiêu chí)

> Tài liệu mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `specs/course-driven-map/spec.md`.

---

## 1. Mục đích & bối cảnh

Hiện tại, **khóa học** chỉ là một "container" lưu trữ nội dung; hệ thống chỉ dùng khóa để tìm bài tập, không dùng nó để **tự sinh hành trình học**. Kết quả:
- Học viên **không thấy trước** bản đồ lộ trình của khóa → không biết khóa này dạy gì, bao gồm những chuyên đề nào.
- Mỗi khóa là một **"hòn đảo riêng"** (máy chơi game riêng) → trải nghiệm **không thống nhất** (vào khóa A có giao diện X, khóa B có giao diện Y).
- Không có **một bản đồ duy nhất** thể hiện tất cả khóa học viên đang học.

**Mục tiêu:** biến **khóa học thành "nguồn gốc sự thật"** — hệ thống **tự động sinh bản đồ game** từ cấu trúc khóa, **gộp tất cả khóa vào 1 lộ trình duy nhất**, cho phép học viên **xem trước trước khi làm bài kiểm tra xếp lớp**, và **hiển thị thẻ năng lực** (trình độ hiện tại, mục tiêu, những chỗ cần cải thiện).

---

## 2. Phạm vi

**TRONG phạm vi:**

1. **Khóa học = nguồn sự thật** — từ kho khóa đã xuất bản (CourseStructure), hệ thống **tự động tạo bản đồ game** (map-v2). Nội dung sống ở khóa: lý thuyết, video, bài tập → bản đồ chỉ là **cách hiển thị có thứ tự**.

2. **Generator khóa→bản đồ** — khi một khóa xuất bản, hệ thống **tự động sinh bản đồ** gồm các **nút (node)** đại diện cho **lý thuyết** (nút nội dung) hoặc **bài tập** (nút câu hỏi). Mỗi khóa **sinh ra 1 phiên bản bản đồ**; khi khóa cập nhật, **phiên bản mới được sinh**, học viên mới nhận phiên bản mới, học viên cũ tiếp tục với phiên bản cũ (không vỡ tiến độ).

3. **Bind quiz vào bài học** — mỗi **bài học** có thể **gắn kèm bài tập (quiz)**; khi làm bài qua bản đồ, **chấm điểm tự động** (không dùng hệ thống chấm riêng song song).

4. **Ghi danh → auto-append** — khi học viên **ghi danh một khóa học**, khóa đó **tự động thêm vào lộ trình duy nhất** của học viên (không phải nhập bằng tay). Nếu ghi danh cùng khóa lần 2, hệ thống **tự động phát hiện & bỏ qua** (không lặp lại).

5. **Một lộ trình duy nhất** — học viên có **1 lộ trình chứa tất cả khóa đã ghi danh**, render theo **thứ tự ghi danh**. Khi xem bản đồ, học viên thấy **tất cả khóa xếp hàng ngay từ trang chính** (không phải vào từng khóa riêng).

6. **Duyệt trước (Roadmap explorer)** — trước khi làm bài kiểm tra xếp lớp, học viên có thể **xem toàn bộ các khóa có sẵn**, chọn **mục tiêu trình độ ban đầu** (A1, A2, B1, v.v.) → hệ thống hiển thị **những khóa phù hợp từ điểm đó đến trình độ mong muốn** (ví dụ A1→B1). Học viên có thể **xem cấu trúc khóa** (có những chuyên đề gì, bao gồm những nút nào) nhưng **không thấy nội dung cụ thể** (để bảo mật và tạo tò mò).

7. **Gộp giao diện** — tất cả khóa học viên dùng **cùng một bản đồ game** (/adaptive) với **cùng giao diện, cùng cách chơi**. Không còn "máy chơi cũ" — tất cả đều mới.

8. **Thẻ năng lực** — khi học viên **đã làm bài kiểm tra xếp lớp**, hệ thống hiển thị **thẻ năng lực** (mức độ từng kỹ năng, mục tiêu, những điểm yếu chi tiết) trên **giao diện bản đồ**, giúp học viên **hiểu rõ mình cần cải thiện gì**.

**NGOÀI phạm vi (để sau):**
- Tự cá nhân hóa lộ trình (bỏ qua bài thành thạo, xen kẽ kỹ năng yếu) — đó là công việc của "Lộ trình thích nghi" (adaptive path), một tính năng riêng.
- Tự động nâng trình độ khi làm bài (auto-checkpoint) — cần dữ liệu kỹ năng gắn vào bài tập, sẽ làm ở giai đoạn sau.
- Chuyển tiếp học viên cũ (từ bản đồ cũ sang mới) — bản đồ cũ sẽ lưu trữ (không xóa), học viên cũ tự tiếp tục; học viên mới xử lý riêng.
- Đồng bộ tiến độ giữa 2 chỗ lưu (bản đồ vs chi tiết khóa) — tạm thời song song; đồng bộ là refactor sau.

---

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên (chưa kiểm tra)** | Xem toàn bộ khóa học có sẵn, chọn mục tiêu trình độ, duyệt cấu trúc khóa (có những chuyên đề gì), để quyết định ghi danh. |
| **Học viên (đã kiểm tra xếp lớp)** | Ghi danh khóa → bản đồ tự cập nhật; xem bản đồ duy nhất chứa tất cả khóa; xem thẻ năng lực (mình mạnh/yếu gì); chơi bản đồ, làm bài tập, thấy tiến độ. |
| **Đội học thuật (admin)** | Tạo/xuất bản khóa học (với cấu trúc chặng→chuyên đề→bài, gắn theory/media/quiz). Không cần thao tác "tạo bản đồ" — bản đồ tự sinh. |

---

## 4. Tổng quan

### Khóa học → Bản đồ (tự động)

Mỗi khóa học có **cấu trúc phân tầng**:
- **Khóa (Course):** IELTS 5.0→6.0, TOEIC 700→800, v.v.
- **Chặng (Phase):** bước tiến độ (A2→B1, hay "Intermediate 1")
- **Chuyên đề (Module):** mảng kỹ năng (Nghe, Đọc, Ngữ pháp, v.v.)
- **Bài học (Lesson):** đơn vị học tập (một bài về "Quá khứ hoàn thành", một bài "Nghe tin tức"), mỗi bài có **lý thuyết** (text/video) và có thể có **bài tập** (quiz, flashcard).

Khi xuất bản khóa, hệ thống **tự động quét cấu trúc** và **sinh bản đồ**:
- **Nút nội dung (CONTENT node):** từ lý thuyết mỗi bài → học viên đọc/xem lý thuyết.
- **Nút câu hỏi (MCQ/FLASHCARD/etc node):** từ bài tập gắn trong bài → học viên làm bài tập, chấm tự động.
- **Thứ tự:** tuần tự theo cấu trúc khóa (chặng→chuyên đề→bài).

**Ví dụ:** khóa "IELTS 5.0→6.0" có:
- Chặng "A2→B1"
  - Chuyên đề "Nghe"
    - Bài "Nghe ý chính" (theory + quiz) → bản đồ sinh 2 nút: nút nội dung "Nghe ý chính", nút câu hỏi "Nghe ý chính - Quiz"
    - Bài "Nghe chi tiết" (theory + quiz) → 2 nút
  - Chuyên đề "Ngữ pháp"
    - Bài "Quá khứ hoàn thành" (theory) → 1 nút nội dung
    - Bài "Quá khứ hoàn thành - Luyện tập" (quiz) → 1 nút câu hỏi

### Lộ trình duy nhất (Roadmap)

Khi học viên **ghi danh một khóa**, bản đồ của khóa đó **tự động nối vào lộ trình duy nhất** của học viên:

- Học viên đang có lộ trình: `[IELTS 5.0→6.0]` (1 khóa)
- Ghi danh khóa "TOEIC 700→800" → lộ trình tự động trở thành: `[IELTS 5.0→6.0, TOEIC 700→800]` (2 khóa xếp hàng)
- Khi vào /adaptive (giao diện bản đồ), học viên thấy **tất cả 2 khóa trên cùng một bản đồ** (không phải vào khóa A rồi quay lại, vào khóa B).

### Duyệt trước (Pre-placement roadmap explorer)

Trước khi làm **bài kiểm tra xếp lớp**:

- Học viên truy /adaptive → thấy **danh sách các khóa có sẵn** (để lọc theo "chương trình: IELTS / TOEIC").
- Chọn **mục tiêu ban đầu** (A1 / A2 / B1 / v.v.) + **mục tiêu kế tiếp** (B1 / B2 / v.v.) → hệ thống hiển thị **những khóa phù hợp** từ A1→B1, từ B1→B2, v.v.
- Xem **cấu trúc từng khóa** (chặng / chuyên đề / nút) nhưng **chỉ thấy tên, không thấy lý thuyết cụ thể hay bài tập** (bảo mật + tạo tò mò).
- Chọn khóa → **ghi danh** → bản đồ tự cập nhật.

### Thẻ năng lực (Post-placement)

Sau khi **làm bài kiểm tra xếp lớp**:

- Hệ thống **tạo "thẻ năng lực"** ghi lại: **mục tiêu hiện tại** (muốn đạt B1 hay IELTS 6.5), **trình độ từng kỹ năng** (Nghe A2, Đọc B1, Nói A1, v.v.), **những điểm yếu chi tiết** (ví dụ nghe ý chính ổn nhưng chi tiết yếu).
- Thẻ này **hiển thị trên giao diện bản đồ**, giúp học viên **biết mình cần học gì** để tiến lên.
- Khi làm bài tập, thẻ **tự cập nhật** (ví dụ học Nghe nhiều → trình độ Nghe tăng từ A2→B1 → thẻ cập nhật ngay).

---

## 5. Quy tắc sinh bản đồ (dễ hiểu)

**Hệ thống tự động sinh bản đồ dựa trên cấu trúc khóa học:**

1. **Quét khóa đã xuất bản** → lấy tất cả **chặng + chuyên đề + bài học**.

2. **Với mỗi bài học:**
   - Nếu có **lý thuyết** (text/video) → tạo **nút nội dung** (học viên đọc/xem).
   - Nếu có **bài tập gắn kèm** (quiz, flashcard) → tạo **nút câu hỏi** (học viên trả lời, chấm tự động).
   - Nếu bài học **rỗng** (không có gì) → **bỏ qua** (không tạo nút).

3. **Sắp thứ tự nút:** tuần tự theo cấu trúc khóa (chặng → chuyên đề → bài).

4. **Bảo mật:** khi học viên **chưa mở khóa** nút nào, **không thấy chi tiết** (chỉ thấy tên nút, không thấy lý thuyết hay bài tập).

5. **Versioning:** mỗi lần khóa cập nhật → **bản đồ mới được sinh**, có **phiên bản mới**. Học viên mới nhận phiên bản mới; học viên **đang học xong trên phiên bản cũ** (không cắt giữa chừng).

---

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Generator & Bản đồ từ khóa**

- **FR-A1:** Khi khóa học được **xuất bản**, hệ thống **tự động sinh bản đồ game** từ cấu trúc khóa. Bản đồ gồm:
  - **Nút nội dung** từ lý thuyết từng bài học.
  - **Nút câu hỏi** từ bài tập gắn trong bài học (quiz, flashcard, v.v.).
  - **Thứ tự:** tuần tự khoa học (chặng → chuyên đề → bài).
  - **Phiên bản:** mỗi bản đồ được **ghi lại phiên bản**; khi khóa cập nhật → bản đồ mới, phiên bản mới.

- **FR-A2:** Nếu bài học **rỗng** (không có lý thuyết, không có bài tập), hệ thống **bỏ qua bài đó** (không tạo nút, không lỗi).

- **FR-A3:** Nếu bài học có **bài tập gắn sai** (ví dụ quiz ID không tồn tại), hệ thống **bỏ qua bài đó hoặc skip bài tập đó** (không vỡ quá trình xuất bản).

**Nhóm B — Ghi danh & Append**

- **FR-B1:** Khi học viên **ghi danh một khóa học**, khóa đó **tự động thêm vào lộ trình duy nhất** của học viên (không phải nhập bằng tay). Nếu **ghi danh cùng khóa lần 2**, hệ thống **tự động phát hiện & bỏ qua** (không duplicate).

- **FR-B2:** Nếu **ghi danh 2 khóa cùng lúc** (race condition), hệ thống **đảm bảo cả 2 được thêm vào** lộ trình, với **thứ tự rõ ràng** (khóa nào vào trước/sau).

- **FR-B3:** Lộ trình được **render theo thứ tự ghi danh** (khóa ghi danh đầu tiên ở phía trước, khóa ghi danh sau ở phía sau). Mỗi khóa trong lộ trình **tự động mở khóa** (học viên có thể bắt đầu ngay), **các nút trong khóa có cổng** (xong nút A mới mở nút B).

**Nhóm C — Duyệt trước (Pre-placement)**

- **FR-C1:** Trước khi làm bài kiểm tra xếp lớp, học viên có thể **duyệt toàn bộ khóa học** có sẵn:
  - **Chọn chương trình** (IELTS / TOEIC / v.v.).
  - **Chọn mục tiêu** (A1 / A2 / B1 / v.v. → B1 / B2 / v.v.) → hệ thống hiển thị **các khóa phù hợp** từ trình độ ban đầu tới mục tiêu.
  - **Xem cấu trúc khóa** (chặng → chuyên đề → nút → tên nút).
  - **KHÔNG thấy nội dung chi tiết** (không thấy lý thuyết, không thấy bài tập) — chỉ thấy tên, tạo bảo mật + tò mò.

- **FR-C2:** Từ duyệt trước, học viên có thể **chọn khóa → ghi danh** (nút "Bắt đầu khóa học" hoặc tương tự).

**Nhóm D — Chấm bài tập qua bản đồ**

- **FR-D1:** Khi học viên **làm bài tập qua bản đồ** (nút câu hỏi), hệ thống **chấm tự động qua nút đó** (không gọi hệ thống chấm riêng song parallel). Kết quả chấm **cập nhật tiến độ nút**.

- **FR-D2:** Nếu học viên **chưa làm bài kiểm tra xếp lớp** (chưa có thẻ năng lực), mà **cố tấn công vào bản đồ để làm bài**, hệ thống **từ chối** (yêu cầu làm kiểm tra trước).

**Nhóm E — Giao diện & Thẻ năng lực**

- **FR-E1:** Tất cả khóa học viên đang học **dùng cùng giao diện bản đồ game** (/adaptive). Không còn máy chơi riêng per khóa — tất cả **thống nhất trên 1 bản đồ**.

- **FR-E2:** Khi học viên **đã làm bài kiểm tra xếp lớp**, hệ thống hiển thị **thẻ năng lực**:
  - **Mục tiêu:** muốn đạt trình độ nào (B1 / IELTS 6.5 / v.v.).
  - **Trình độ từng kỹ năng:** Nghe / Đọc / Viết / Nói / Ngữ pháp / Từ vựng → mỗi cái A1 / A2 / B1 / v.v.
  - **Điểm yếu chi tiết:** ví dụ "Nghe ý chính ổn nhưng chi tiết yếu", "Quá khứ hoàn thành yếu".

- **FR-E3:** Thẻ năng lực **tự cập nhật** khi học viên làm bài tập (làm nhiều bài Nghe tốt → Nghe tăng; quá khứ yếu nhưng luyện nhiều → quá khứ cải thiện).

- **FR-E4:** Khi học viên **mới (chưa ghi danh khóa nào)**, bản đồ hiển thị **trạng thái rỗng** (không có nút nào) + **nút CTA "Ghi danh khóa học"** dẫn đến duyệt khóa (FR-C1).

**Nhóm F — Cổng kiểm tra xếp lớp (Placement gate)**

- **FR-F1:** Nếu học viên **chưa làm bài kiểm tra** (chưa có thẻ năng lực), **không được vào bản đồ** → hệ thống yêu cầu "Làm bài kiểm tra xếp lớp trước" + link dẫn sang trang kiểm tra.

---

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Dữ liệu kiểm tra):** Bản đồ game cần **thẻ năng lực từ bài kiểm tra** mới có ý nghĩa đầy đủ. Trước kiểm tra → duyệt trước được, nhưng **không vào bản đồ chính**.

- **NFR-2 (Bảo mật):** Nội dung nhạy cảm (lý thuyết, bài tập, đáp án) **chỉ hiển thị sau khi mở khóa**. Duyệt trước chỉ thấy tên, **KHÔNG thấy chi tiết**.

- **NFR-3 (Versioning):** Mỗi khi khóa cập nhật, **bản đồ mới được sinh** (phiên bản mới). Học viên mới nhận phiên bản mới; **học viên đang học giữ phiên bản cũ** để không vỡ tiến độ.

- **NFR-4 (Idempotency):** Ghi danh cùng khóa nhiều lần = **ghi danh 1 lần** (không duplicate, không lỗi).

- **NFR-5 (Tái sử dụng):** Nội dung bài học **sống ở khóa**, bản đồ chỉ là **view (cách nhìn)** → không lặp lại dữ liệu, dễ bảo trì.

- **NFR-6 (Hiệu năng):** Sinh bản đồ khi xuất bản khóa phải **nhanh** (không kéo dài quá trình xuất bản). Duyệt trước & lấy bản đồ phải **trả dữ liệu nhanh** (cached tốt).

---

## 8. Luồng người dùng (tóm tắt)

### Học viên chưa kiểm tra xếp lớp
1. Vào /adaptive → thấy **Roadmap explorer**.
2. Chọn **chương trình & mục tiêu** → xem **danh sách khóa phù hợp**.
3. Xem **cấu trúc khóa** (chặng/chuyên đề/nút) → chọn một khóa.
4. Click **"Ghi danh" / "Bắt đầu khóa học"** → ghi danh thành công.
5. Hệ thống nhắc **"Làm bài kiểm tra xếp lớp trước khi học"** → chuyển sang trang kiểm tra.

### Học viên đã kiểm tra xếp lớp & ghi danh khóa
1. Vào /adaptive → thấy **bản đồ game** + **thẻ năng lực**.
2. Thẻ năng lực cho thấy: **mục tiêu**, **trình độ từng kỹ năng**, **điểm yếu**.
3. Chơi bản đồ: click nút → đọc lý thuyết / làm bài tập → chấm tự động.
4. Xong nút → nút kế tiếp mở khóa.
5. Hoàn thành chuyên đề / chặng → thẻ năng lực **cập nhật** tiến độ.
6. Ghi danh khóa thứ 2 → lộ trình **tự động thêm khóa thứ 2** → bản đồ hiển thị **2 khóa xếp hàng**.

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

Tính năng coi là **XONG** khi:

1. ✅ Khóa xuất bản → **bản đồ tự sinh** gồm nút nội dung (lý thuyết) + nút câu hỏi (bài tập), **thứ tự đúng**.
   
2. ✅ Học viên **duyệt trước được** danh sách khóa theo chương trình + mục tiêu; **không thấy nội dung chi tiết** (bảo mật).

3. ✅ Ghi danh khóa → **lộ trình tự append khóa đó**; nếu ghi danh cùng khóa lần 2 → **không duplicate** (idempotent).

4. ✅ Ghi danh 2 khóa cùng lúc (race) → **cả 2 được thêm vào** lộ trình, **thứ tự phân biệt** (AC-E4).

5. ✅ Học viên **đã kiểm tra → thấy bản đồ + thẻ năng lực** (mục tiêu, trình độ, điểm yếu).

6. ✅ Học viên **chưa kiểm tra → cổng từ chối** (không vào bản đơ chính, yêu cầu kiểm tra trước).

7. ✅ Làm bài tập qua bản đồ → **chấm tự động, cập nhật tiến độ** (không grader song parallel).

8. ✅ Bản đồ **giao diện thống nhất** (tất cả khóa trên /adaptive, không riêng máy chơi per khóa).

9. ✅ Khóa cập nhật → **bản đồ phiên bản mới sinh**; học viên mới nhận phiên bản mới, **học viên cũ giữ phiên bản cũ**.

10. ✅ Khi khóa = rỗng hoặc bài tập sai → **không vỡ xuất bản** (skip, không lỗi).

11. ✅ **Regression test:** tất cả test hiện tại vẫn xanh (không vỡ feature cũ).

---

## 10. Ngoài phạm vi

- **Cá nhân hóa lộ trình** (skip bài thành thạo, xen kẽ kỹ năng yếu) — đó là "Lộ trình thích nghi", tính năng riêng, sẽ làm sau.
- **Tự động nâng trình độ từ bài tập** (auto-checkpoint) — cần gắn kỹ năng vào bài tập trước, sẽ làm ở giai đoạn sau.
- **Chuyển tiếp học viên cũ** từ bản đồ cũ → mới — bản đồ cũ lưu trữ (archive), học viên cũ tự tiếp tục; không ảnh hưởng.
- **Đồng bộ tiến độ giữa 2 nơi** (bản đồ vs chi tiết khóa) — tạm thời song song; sẽ đồng bộ ở refactor sau.
- **Tự sinh nội dung bằng AI** — feature riêng, không liên quan.

---

## 11. Thuật ngữ (cho người không chuyên)

- **Khóa học (Course):** một chương trình học tập hoàn chỉnh (IELTS 5.0→6.0, TOEIC 700→800), có tên và mục tiêu rõ ràng.

- **Chặng (Phase/Stage):** một bước tiến độ trong khóa học (A2→B1, hay "Intermediate 1"), thường đánh dấu một cột mốc trình độ.

- **Chuyên đề (Module):** một mảng kỹ năng hoặc chủ đề trong một chặng (Nghe, Đọc, Ngữ pháp, Từ vựng, Nói, Viết).

- **Bài học (Lesson):** đơn vị học tập nhỏ nhất (một bài về "Quá khứ hoàn thành", một bài "Nghe tin tức"), chứa lý thuyết và/hoặc bài tập.

- **Nút (Node):** một điểm trên bản đồ game đại diện cho một hoạt động học (đọc lý thuyết, làm quiz, học flashcard).
  - **Nút nội dung (CONTENT node):** học viên đọc/xem lý thuyết.
  - **Nút câu hỏi (MCQ/QUIZ/FLASHCARD node):** học viên trả lời, chấm tự động.

- **Bản đồ game (Game Map):** cách hiển thị trực quan các nút và thứ tự học, cho phép học viên "chơi" (di chuyển, hoàn thành nút, mở khóa nút kế tiếp).

- **Lộ trình (Roadmap):** danh sách các khóa học viên đang học hoặc ghi danh, xếp hàng theo thứ tự ghi danh.

- **Roadmap explorer:** giao diện cho phép học viên duyệt các khóa học có sẵn trước khi ghi danh (chỉ thấy tên & cấu trúc, không nội dung).

- **Thẻ năng lực (Competency card):** bản tóm tắt trình độ của học viên — mục tiêu, trình độ từng kỹ năng, điểm yếu chi tiết.

- **Ghi danh (Enroll):** học viên chọn một khóa để bắt đầu học; khóa đó tự động thêm vào lộ trình của học viên.

- **Phiên bản bản đồ (Map version):** mỗi lần khóa cập nhật → bản đồ mới được sinh với phiên bản mới; học viên mới nhận phiên bản mới, học viên cũ giữ phiên bản cũ.

- **Cổng kiểm tra (Placement gate):** yêu cầu học viên phải làm bài kiểm tra xếp lớp trước khi vào bản đồ chính (chỉ duyệt trước được, không vào chơi).

- **Idempotent:** hành động thực hiện nhiều lần = kết quả như một lần (ghi danh cùng khóa 2 lần không tạo 2 bản ghi).

- **Source of truth:** "nguồn sự thật" — dữ liệu duy nhất được tin cậy; trong trường hợp này, khóa học là nguồn sự thật, bản đồ là view dẫn xuất.
