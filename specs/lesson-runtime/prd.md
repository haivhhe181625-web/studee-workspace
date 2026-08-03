<!-- Tiếng Việt — PRD/SRS cho "Lesson Runtime": Học viên xem & học nội dung bài học sau khi ghi danh khóa.
     Viết cho người KHÔNG chuyên kỹ thuật.
     Chi tiết kỹ thuật: design.md (Giai đoạn B & C) + contracts/web-course-consume.md
     Trạng thái: Hoàn tất Giai đoạn B (API) + C (Web UI) — sẵn sàng đưa vào sử dụng. -->

# PRD — Học viên xem & học nội dung bài học (Lesson Runtime)

- **Ngày:** 2026-08-03 · **Trạng thái:** Hoàn tất (Giai đoạn B & C) · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Chi tiết kỹ thuật:** `specs/lesson-runtime/design.md` (Giai đoạn B: API · Giai đoạn C: Web) + `specs/lesson-runtime/contracts/web-course-consume.md` (hợp đồng api↔web) + `specs/lesson-runtime/tasks.md` (danh sách công việc)

> Tài liệu mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu cho người không chuyên kỹ thuật. Kỹ thuật đọc kèm `specs/lesson-runtime/design.md`.

---

## 1. Mục đích & bối cảnh

Hiện tại, khi học viên **ghi danh một khóa học**, họ chỉ **nhìn thấy tên khóa** nhưng **chưa có cách để xem chi tiết nội dung bên trong** (như lý thuyết, video, âm thanh, hoặc các bài tập). Khóa học tồn tại ở hệ thống nhưng **không thể mở để học**.

**Mục tiêu:** xây dựng khả năng cho học viên **mở một khóa học đã ghi danh**, **xem toàn bộ cấu trúc của khóa** (các Chặng → Chuyên đề → Bài học), **đọc nội dung lý thuyết**, **xem video/nghe audio**, và **làm bài tập** (bài tập phát âm, bài tập hội thoại). **Nội dung trả phí chỉ được hiển thị cho học viên đã ghi danh** (bảo vệ quyền sở hữu nội dung).

---

## 2. Phạm vi

**TRONG phạm vi:**
1. **Danh sách khóa học** — hiển thị các khóa đã được xuất bản (công khai) cho học viên xem lựa chọn.
2. **Chi tiết khóa học** — khi học viên chọn một khóa, hiển thị **toàn bộ cấu trúc bên trong** (các Chặng → Chuyên đề → Bài học) với đầy đủ thông tin.
3. **Xem bài học** — học viên mở một bài học cụ thể để xem nội dung:
   - Lý thuyết (văn bản + hình ảnh định dạng markdown).
   - Video (nếu bài có).
   - Âm thanh (nếu bài có).
4. **Làm bài tập** — học viên làm các bài tập trong bài học:
   - Bài tập phát âm (dùng chính engine phát âm hiện có).
   - Bài tập hội thoại (dùng chính engine hội thoại hiện có).
5. **Xác thực & bảo vệ nội dung** — chỉ những học viên **đã đăng nhập** mới xem được nội dung (không xem được khi chưa đăng nhập). Chỉ khóa **đã xuất bản** mới hiển thị (ẩn khóa nháp hoặc đã lưu trữ).

**NGOÀI phạm vi (để sau):**
- Ghi nhận tiến độ học & lưu điểm số chi tiết (đó là công việc của Phase 3 "Tiến độ & chấm điểm").
- Tự động mở khóa bài học tuần tự theo kết quả học (đó là Phase 3).
- Tích hợp với "Lộ trình thích nghi" (đó là feature riêng khác).
- Thay đổi nội dung khóa — hệ thống chỉ **đọc** khóa, không chỉnh sửa (chỉnh sửa do admin làm ở phần khác).

---

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên** | Xem được danh sách khóa học đã ghi danh (hoặc sắp tới); chọn khóa → xem chi tiết cấu trúc; mở bài học để đọc lý thuyết, xem video, nghe audio; làm bài tập phát âm & hội thoại để luyện tập. |
| **Đội học thuật (admin)** | Không cần thao tác mới ở phần này — chỉ cần **ghi rõ nội dung khóa** (lý thuyết, media, bài tập) ở phần quản lý khóa (admin-api). Phần này chỉ **đọc & hiển thị** nội dung đã có. |
| **Hệ thống** | **Bảo vệ nội dung trả phí**: chỉ học viên đã đăng nhập mới xem được; **ẩn khóa chưa xuất bản** (khỏi bị học viên thấy nháp hoặc lỗi). |

---

## 4. Tổng quan

**Học viên khi mở app sẽ trải qua 3 màn chính:**

### 4.1. Danh sách khóa học (Catalog)
- Hiển thị các **khóa đã xuất bản** (công khai sẵn sàng học).
- Mỗi khóa hiện: tên khóa, mô tả, ảnh thumbnail, số lượng Chặng, chương trình (nếu có: IELTS, TOEIC, v.v.), mục tiêu điểm (nếu có).
- Học viên chọn một khóa → đi tới trang chi tiết.

### 4.2. Chi tiết khóa (Cây cấu trúc)
- Khi vào khóa, hiển thị **toàn bộ cấu trúc bên trong** như một cây:
  - **Chặng** (ví dụ: "Chặng 1: A2→B1") → mở rộng để xem các Chuyên đề bên trong.
  - **Chuyên đề** (ví dụ: "Ngữ pháp", "Từ vựng") → mở rộng để xem các Bài học.
  - **Bài học** (ví dụ: "Bài 1", "Bài 2") → click để mở bài học.
- Mỗi Chặng còn hiển thị khoảng điểm gốc (ví dụ: 5.5–6.5 band IELTS) và lưu ý về mục tiêu.

### 4.3. Xem bài học (Lesson Viewer)
- Khi học viên click vào một bài, hiển thị:
  - **Lý thuyết**: văn bản nội dung (hỗ trợ định dạng Markdown: tiêu đề, danh sách, bảng, liên kết, v.v.).
  - **Media**: video hoặc âm thanh (nếu bài có).
  - **Bài tập**: danh sách bài tập thuộc bài này — khi click bài tập, **mở engine phát âm hoặc hội thoại** để làm bài.

---

## 5. Quy tắc & luồng chính

### 5.1. Quy tắc xác thực
1. **Chỉ đăng nhập xem được**: học viên phải **đăng nhập** mới truy cập được danh sách/chi tiết khóa (nếu chưa đăng nhập → redirect đến trang đăng nhập).
2. **Chỉ khóa xuất bản**: chỉ khóa có **trạng thái "xuất bản"** mới hiển thị (khóa nháp, sẵn sàng, hoặc lưu trữ → ẩn không hiện, trả lỗi 404 nếu học viên cố tình truy cập trực tiếp).

### 5.2. Luồng chính (Learner Journey)
1. **Học viên đăng nhập** → trang chủ.
2. **Tìm & chọn khóa** → click vào mục "Khóa học" (hoặc "Học") → xem danh sách khóa đã xuất bản.
3. **Xem chi tiết khóa** → click một khóa → hiển thị cấu trúc (Chặng → Chuyên đề → Bài).
4. **Chọn bài học** → click vào bài → xem nội dung (lý thuyết + media).
5. **Làm bài tập** → click bài tập → mở engine phát âm/hội thoại (hoặc ngoài ra, chuyển sang trang riêng).
6. **Quay lại** → quay lại danh sách bài hoặc cấu trúc khóa để chọn bài khác.

---

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Danh sách & Khám phá khóa học**
- **FR-A1** Hiển thị **danh sách khóa đã xuất bản** với thông tin cơ bản (tên, mô tả, thumbnail, số Chặng, chương trình, mục tiêu điểm).
- **FR-A2** **Sắp xếp khóa** theo thời gian tạo (mới nhất trước). Có thể **lọc theo chương trình** (nếu có: IELTS, TOEIC, v.v.) để học viên tìm khóa phù hợp.
- **FR-A3** **Mỗi khóa là một thẻ clickable** — click vào → đi tới trang chi tiết khóa.

**Nhóm B — Chi tiết & Cấu trúc khóa**
- **FR-B1** Hiển thị **toàn bộ cấu trúc khóa** dưới dạng **cây có thể mở rộng**: Chặng → Chuyên đề → Bài học.
- **FR-B2** Mỗi **Chặng** hiển thị:
  - Tên Chặng + thứ tự (vd "Chặng 1").
  - Khoảng CEFR (vd "A2→B1") + khoảng điểm (vd "5.5–6.5 band IELTS", nếu có).
  - Ghi chú mục tiêu (nếu có).
  - Nút/icon mở rộng để xem Chuyên đề bên trong.
- **FR-B3** Mỗi **Chuyên đề** hiển thị:
  - Tên Chuyên đề + danh mục (vd "Ngữ pháp", "Từ vựng", "Nghe").
  - Thứ tự trong Chặng.
  - Danh sách Bài học (dạng danh sách hoặc mở rộng).
- **FR-B4** Mỗi **Bài học** hiển thị:
  - Tên Bài + thứ tự.
  - Nút click để mở bài → trang xem bài.
- **FR-B5** Có **nút quay lại** để học viên quay lại danh sách khóa hoặc cấu trúc khóa.

**Nhóm C — Xem & Học nội dung bài**
- **FR-C1** Khi học viên click vào bài, **hiển thị nội dung lý thuyết** của bài đó (văn bản Markdown: tiêu đề, đoạn văn, danh sách, bảng, liên kết, hình ảnh, v.v.).
- **FR-C2** **Hiển thị media** nếu bài có:
  - **Video**: hiển thị player video với nút play/pause/tua (dùng player chuẩn).
  - **Âm thanh**: hiển thị player audio với nút play/pause/tua.
  - Media có **nhãn** (vd "Video giải thích") hoặc **không nhãn** → hiển thị hoặc bỏ qua tùy có.
- **FR-C3** **Hiển thị danh sách bài tập** của bài đó:
  - Danh sách các bài tập (phát âm, hội thoại, hoặc loại khác).
  - Mỗi bài tập có thể có **nhãn** (vd "Bài 1: Phát âm từ") → hiển thị nhãn nếu có.
- **FR-C4** **Nút mở bài tập**:
  - **Bài tập phát âm** → click → **mở engine phát âm** (dùng chính player phát âm có sẵn, không tạo cái mới).
  - **Bài tập hội thoại** → click → **mở engine hội thoại** (dùng chính player hội thoại có sẵn, hoặc hiển thị trực tiếp nếu engine tích hợp).
  - Học viên làm bài tập, hoàn thành, quay lại bài.
- **FR-C5** **Nút quay lại** để quay lại cấu trúc khóa.
- **FR-C6** **Ghi nhận hành động** học viên: khi học viên **hoàn thành bài tập** (phát âm/hội thoại), tiến độ được **ghi nhận chung với khóa gốc** (không phải coi là làm lại ở nơi khác) — điều này sẽ được xử lý ở Phase 3 (Tiến độ).

**Nhóm D — Xác thực & Bảo vệ**
- **FR-D1** **Chỉ đăng nhập xem được**: khi truy cập danh sách khóa mà chưa đăng nhập → **redirect đến trang đăng nhập** (hoặc hiển thị thông báo "vui lòng đăng nhập").
- **FR-D2** **Chỉ khóa xuất bản**: khi học viên cố gắng xem khóa chưa xuất bản (nháp, sẵn sàng, lưu trữ) → **trả lỗi 404** (khỏi bị lộ tên khóa chưa công khai).
- **FR-D3** **Ẩn field nội bộ**: không hiển thị các trường nội bộ của khóa (vd người import, metadata nguồn, ID trung tâm, v.v.) cho học viên — chỉ hiển thị trường công khai.

**Nhóm E — Hiệu năng & Tải trang**
- **FR-E1** **Tải danh sách nhẹ**: khi liệt kê khóa, không tải toàn bộ cây nội dung (sẽ chậm), chỉ tải thông tin cơ bản + số lượng Chặng.
- **FR-E2** **Tải chi tiết khi cần**: khi học viên vào chi tiết khóa, mới tải cả cây (Chặng → Chuyên đề → Bài) để hiển thị đầy đủ.

---

## 7. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Xác thực bắt buộc):** học viên chưa đăng nhập → không xem được bất cứ thông tin khóa nào (chỉ thấy trang đăng nhập).
- **NFR-2 (Bảo vệ nội dung trả phí):** khóa chưa xuất bản → không lộ tồn tại (404, không lộ danh sách khóa nháp).
- **NFR-3 (Hiệu năng danh sách):** danh sách khóa tải nhanh (chỉ cơ bản, không ship cây).
- **NFR-4 (Không sửa khóa):** hệ thống chỉ **đọc** khóa; không cho phép học viên/app sửa nội dung khóa ở chỗ này (chỉnh sửa ở admin-api).
- **NFR-5 (Rõ ràng & dễ sử dụng):** giao diện hiển thị cấu trúc khóa dễ hiểu (cây có thể mở rộng, icon rõ ràng, nút quay lại dễ tìm).

---

## 8. Luồng người dùng (tóm tắt)

```
Học viên đăng nhập
  ↓
Truy cập "Khóa học" / "Học"
  ↓
Xem danh sách khóa đã xuất bản (catalog)
  ↓
Click khóa → xem chi tiết (cấu trúc Chặng → Chuyên đề → Bài)
  ↓
Click bài → xem nội dung (lý thuyết + media)
  ↓
Click bài tập → mở engine phát âm/hội thoại
  ↓
Làm bài → hoàn thành
  ↓
Quay lại → chọn bài khác hoặc quay về danh sách
```

---

## 9. Tiêu chí hoàn thành (nghiệm thu)

- ✅ Học viên đăng nhập xem được **danh sách khóa được xuất bản** (không thấy khóa nháp).
- ✅ Chọn khóa → hiển thị **toàn bộ cấu trúc** (Chặng → Chuyên đề → Bài) **đầy đủ, chính xác**.
- ✅ Click bài → xem **lý thuyết Markdown** (định dạng đúng), **video/audio** (nếu có), **danh sách bài tập**.
- ✅ Click bài tập:
  - **Phát âm** → mở player phát âm hiện có.
  - **Hội thoại** → mở engine hội thoại hiện có.
- ✅ **Xác thực**: chưa đăng nhập → redirect; khóa chưa xuất bản → 404.
- ✅ **Hiệu năng**: danh sách tải nhanh, chi tiết khi cần.
- ✅ **Quay lại & điều hướng**: nút quay lại hoạt động, không bị mất tiến trình.

---

## 10. Ngoài phạm vi

- **Tiến độ & Chấm điểm** (Phase 3): ghi nhận % hoàn thành bài tập, điểm số, thời gian học → làm ở phase sau.
- **Mở khóa tuần tự** (Phase 3): tự động mở bài kế tiếp khi hoàn thành bài hiện tại (dựa trên kết quả) → làm ở phase sau.
- **Lộ trình thích nghi** (Feature khác): tạo lộ trình cá nhân hóa dựa trên năng lực → feature riêng.
- **Tích hợp với band/score lịch sử** (Phase 3): hiển thị tuân theo hồ sơ năng lực của học viên → để sau.
- **Thay đổi nội dung khóa**: admin sửa khóa → đó là admin-api, không phải learner-api.

---

## 11. Thuật ngữ (cho người không chuyên)

| Thuật ngữ | Giải thích |
|---|---|
| **Khóa học** | Một chương trình học hoàn chỉnh (vd "Khóa IELTS 5.0→6.5", "Khóa tiếng Anh A1→A2"). |
| **Khóa xuất bản** | Khóa đã hoàn chỉnh, kiểm duyệt xong, công khai để học viên xem & học (không phải nháp). |
| **Khóa nháp** | Khóa đang soạn, chưa hoàn chỉnh, không công khai (học viên không thấy). |
| **Chặng (Phase)** | Một giai đoạn trong khóa (vd "Chặng 1: A2→B1"; "Chặng 2: B1→B2"). Mỗi chặng tương ứng với một khoảng trình độ (CEFR). |
| **Chuyên đề (Module)** | Một mảng kỹ năng trong chặng (vd "Ngữ pháp", "Từ vựng", "Nghe", "Đọc"). |
| **Bài học (Lesson)** | Đơn vị học nhỏ nhất (vd "Bài 1", "Bài 2"). Bài học chứa lý thuyết, media, bài tập. |
| **Lý thuyết** | Nội dung giáo dục dạng văn bản, hình ảnh (định dạng Markdown: tiêu đề, danh sách, bảng, v.v.). |
| **Media** | Video hoặc âm thanh (ví dụ: video giải thích, bài nghe luyện). |
| **Bài tập** | Một hoạt động học tập (bài tập phát âm, bài hội thoại, v.v.). Bài tập được chạy bởi engine chuyên dụng (phát âm, hội thoại). |
| **Engine phát âm** | Công cụ để học viên luyện phát âm (ghi âm, so sánh, đánh giá). |
| **Engine hội thoại** | Công cụ để học viên luyện hội thoại (tương tác, trả lời câu hỏi, có phản hồi). |
| **Đăng nhập** | Quá trình xác thực danh tính (học viên nhập tài khoản/mật khẩu). Chỉ người đã đăng nhập mới xem được nội dung trả phí. |
| **CEFR** | Khung trình độ tiếng Anh quốc tế (A1, A2, B1, B2, C1, C2 — từ thấp tới cao). |
| **Catalog** | Danh sách khóa học để học viên lựa chọn. |
| **Ghi danh khóa (Enroll)** | Hành động học viên đăng ký theo học một khóa (sẽ được xử lý ở phase khác). |
| **Tiến độ (Progress)** | Thông tin về bao nhiêu % bài học/bài tập học viên đã hoàn thành (sẽ được xử lý ở Phase 3). |
| **Mastery / Thành thạo** | Mức nắm vững một nội dung/kỹ năng (dùng để quyết định có cần học lại hay không). |
