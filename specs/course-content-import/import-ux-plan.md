<!-- Tiếng Việt — theo .ai/project-context.md §8. Plan giảm ma sát import cho đội dữ liệu non-tech. -->
# Plan: Import thân thiện cho đội dữ liệu non-tech (đội tiếng Anh)

- **Ngày:** 2026-07-15
- **Tác giả:** Product Owner + Tech Lead Agent
- **Bối cảnh:** người nhập là **chuyên gia tiếng Anh, không rành kỹ thuật**; file Excel họ tạo **thường không đúng cấu trúc chuẩn**.
- **Liên quan:** `spec.md` (IS-7), `design.md` (§3.2 parser), `tasks.md` (Task 3, Task 6/7), `contracts/admin-course-import.md`, PRD §7.1/§12.

---

## 1. Vấn đề

Thiết kế Phase 1 hiện tại đặt gánh nặng "đúng ngay từ đầu" lên người nhập:

| Điểm đau | Hiện trạng trong code/design | Hệ quả với non-tech |
|---|---|---|
| **Khớp cột theo vị trí** | Parser đọc `cell(row, 1..11)` (A Level … K Note) theo **thứ tự cột cố định** | Chèn/xóa/xê dịch 1 cột ⇒ toàn bộ file lệch, lỗi hàng loạt khó hiểu |
| **Template tĩnh** | File mẫu chỉ có header + vài dòng ví dụ, **không ràng buộc giá trị** | Gõ sai CEFR (`a2`, `A2 `, `pre-A2`), sai category, sai Level ⇒ lỗi |
| **Phải biết ID kỹ thuật** | Bài học trỏ tới `ipa`/`talk` bằng **ID hệ thống** | Đội tiếng Anh **không thể biết** ID này ở đâu ra |
| **All-or-nothing + upload thẳng** | Có lỗi ⇒ từ chối cả lô, không có bước xem trước | Sửa mò nhiều vòng, nản, tốn thời gian |
| **Lỗi dạng danh sách text** | `errors[]` trả JSON | Không rõ "sửa ô nào trong file của tôi" |

**Nguyên tắc plan:** *ngăn lỗi tại nguồn* > *bắt lỗi sớm & rõ* > *khoan dung khi phân tích* > (sau) *bỏ Excel*.

---

## 2. Bốn tuyến giải pháp

### Tuyến A — Ngăn lỗi tại nguồn: **Template thông minh** (đòn bẩy mạnh nhất, rẻ)

Nâng cấp `GET /course-imports/template` (đang là file tĩnh) thành **workbook nhiều sheet có ràng buộc**:

1. **Sheet `Huong dan`** — giải thích 4 tầng, quy tắc thứ tự dòng cha–con, ví dụ 1 khóa mini hoàn chỉnh.
2. **Sheet `Du lieu`** — vùng nhập chính, có **Excel Data Validation (dropdown)**:
   - `Level` ∈ {ROADMAP, PHASE, MODULE, LESSON, MEDIA, EXERCISE}
   - `CefrFrom/CefrTo` ∈ {A1, A2, B1, B2, C1, C2}
   - `Category` ∈ {ngữ pháp, phát âm, từ vựng, nghe, đọc, viết, nói}
   - `Type` (MEDIA) ∈ {video, audio}; `Type` (EXERCISE) ∈ {ipa, talk} *(Phase 4 nới thêm)*
   - Ô sai dropdown bị Excel chặn ngay khi gõ ⇒ lỗi không kịp sinh ra.
3. **Sheet `Danh muc bai tap`** *(xem Tuyến C)* — danh sách ID `ipa`/`talk` có sẵn để copy.
4. Dòng ví dụ mẫu + comment trên từng cột header.

> `exceljs` hỗ trợ `dataValidation` (list) và nhiều worksheet ⇒ làm được ngay, không thêm dependency.

### Tuyến B — Bắt lỗi sớm & rõ: **Dry-run + báo lỗi trực quan**

1. **Chế độ "Kiểm tra thử" (validate-only)**: thêm `POST /course-imports/_/validate` (hoặc `?dryRun=true`) — chạy **đúng pipeline validate** nhưng **KHÔNG ghi**. Trả về:
   - danh sách lỗi (như import thật), **và**
   - **cây xem trước** (đếm Chặng/Chuyên đề/Bài học, phác thảo cấu trúc) khi hợp lệ.
   - *Chi phí thấp:* service đã tách `validateCourse` khỏi `importCourse` (design §3) ⇒ chỉ cần bỏ bước persist.
2. **Vòng lặp sửa an toàn:** đội học thuật bấm "Kiểm tra" nhiều lần đến khi sạch lỗi rồi mới "Import chính thức".
3. **Báo lỗi trực quan trên `exe-admin`** (không chỉ list text):
   - Bảng lỗi **nhóm theo loại** + **có số dòng** click nhảy tới; đếm "12 lỗi / 3 loại".
   - (Tùy chọn, nâng cao) **tải xuống file Excel gốc được tô màu ô sai** kèm ghi chú — người nhập mở đúng file của họ và thấy đỏ ở đâu.

### Tuyến C — Gỡ nút "không biết ID bài tập"

Đội tiếng Anh không thể biết ID `ipa`/`talk`. Ba mức, chọn theo phase:

1. **Phase 1 — Danh mục nhúng trong template** *(khuyến nghị làm ngay):* sheet `Danh muc bai tap` được **sinh động** khi tải template — liệt kê mọi bài tập `published`: cột `Type | Id | Ten hien thi`. Người nhập **copy Id** từ đây sang cột `RefId`. → cần template thành **endpoint sinh file** (đọc DB) thay vì file tĩnh.
   - *Thay thế nhẹ hơn:* endpoint riêng `GET /course-imports/exercise-catalog` (JSON/CSV) để tra cứu.
2. **Phase 1.5 — Cho tham chiếu bằng tên/slug thân thiện:** resolver chấp nhận `RefId` là **slug/tên** rồi map sang ID hệ thống; nếu trùng tên ⇒ báo lỗi "cần ID chính xác". Giảm phụ thuộc vào ID thô.
3. **Phase 5 — Picker trong UI:** khi có CMS, chọn bài tập bằng **hộp tìm kiếm**, không gõ ID.

### Tuyến D — Khoan dung khi phân tích (parser bền hơn)

Nới parser để dung thứ lỗi vô hại, **vẫn giữ all-or-nothing với lỗi ngữ nghĩa thật**:

1. **Khớp cột theo TÊN HEADER, không theo vị trí** — đọc dòng header, lập map `{tên cột → chỉ số}`. Cho phép chèn/xê dịch/thêm cột phụ. *(Thay đổi cốt lõi ở Task 3.)*
2. **Chuẩn hóa giá trị:** `trim()` (đã có), enum **không phân biệt hoa/thường & bỏ dấu cách** (`" A2 "`→`A2`), chấp nhận **từ đồng nghĩa category** (vd "grammar"→"ngữ pháp").
3. **Bỏ qua dòng trống & dòng ghi chú** (đã bỏ dòng trống); cho phép cột `#`/ghi chú tự do mà parser lờ đi.
4. **Thông báo lỗi giàu ngữ cảnh:** kèm `sheet`, `row`, `column`, **giá trị đang sai**, và **gợi ý sửa** ("Category 'grammer' không hợp lệ — có phải 'ngữ pháp'?").

### Tuyến E *(Phase 5)* — Bỏ Excel: CMS + AI chuẩn hóa

- **CMS kéo-thả** dựng khóa trực tiếp (PRD FR-IN-13) — không còn khớp cột.
- **AI "Smart Import"** (FR-IN-14, `ai_ingestion_tool_plan.md`): nhận **file lộn xộn/mô tả tự do**, LLM **đề xuất** cây khóa chuẩn để người nhập **review & sửa** trước khi publish. Đây là lời giải triệt để cho "file không đúng cấu trúc", nhưng cần bước review con người để an toàn.

---

## 3. Khuyến nghị đưa vào Phase 1 (rẻ, tác động lớn)

| Hạng mục | Tuyến | Chi phí | Ghi chú triển khai |
|---|---|---|---|
| Template thông minh (dropdown + hướng dẫn + ví dụ) | A | Thấp | `exceljs dataValidation`, nhiều sheet — sửa Task 6/7 (route template) |
| Khớp cột theo header | D1 | Thấp–TB | Sửa parser Task 3: build header-map thay vì `cell(row, N)` cứng |
| Chuẩn hóa enum (hoa/thường, đồng nghĩa) | D2 | Thấp | Bảng ánh xạ trong parser/validate |
| Sheet danh mục bài tập trong template | C1 | TB | Template thành **endpoint sinh động** (đọc ipa/talk published) |
| Dry-run / Kiểm tra thử | B1 | Thấp | `validateCourse` đã tách sẵn — thêm route không-ghi |
| Báo lỗi nhóm-theo-loại + số dòng click | B3 | Thấp | FE `exe-admin` (Task 8) render bảng lỗi |

**Hoãn có chủ đích sang sau:** tô màu ô sai trong file tải về (B3 nâng cao), tham chiếu bằng tên (C2), picker UI (C3), CMS + AI (E).

---

## 4. Tác động tới tài liệu hiện có

- **spec.md:** thêm IS mới — IS-11 "Template thông minh + Kiểm tra thử (dry-run)"; ghi rõ parser khớp theo header.
- **acceptance.md:** thêm AC — dry-run trả lỗi & không ghi; template có dropdown; cột đảo thứ tự vẫn parse đúng; enum sai hoa/thường vẫn nhận.
- **design.md §3.2:** đổi parser từ positional → header-map; thêm bảng chuẩn hóa enum.
- **contracts:** thêm `POST /course-imports/_/validate` (dry-run); ghi rõ template là endpoint sinh động (đọc catalog bài tập).
- **tasks.md:** cập nhật Task 3 (parser header-map + chuẩn hóa), Task 6/7 (template động + route validate), Task 8 (FE: nút Kiểm tra thử + bảng lỗi trực quan).
- **PRD §7.1:** bổ sung FR-IN — dry-run, template thông minh, danh mục bài tập; **§12 rủi ro** "non-tech điền sai" gắn với plan này.

---

## 5. Quyết định cần PO chốt

1. **Đưa gói Phase-1 ở §3 vào ngay đợt này** hay tách một đợt "Import UX" nhỏ sau khi core import chạy? *(Khuyến nghị: gộp Template thông minh + Dry-run + Header-map vào Phase 1 — chúng rẻ và quyết định trải nghiệm thành/bại.)*
2. **Template:** giữ file tĩnh hay chuyển sang **endpoint sinh động** (để nhúng danh mục bài tập)? *(Khuyến nghị: động.)*
3. **Tham chiếu bài tập:** Phase 1 dùng **ID copy từ danh mục** (C1) là đủ, hay cần cho **tên/slug** (C2) ngay? *(Khuyến nghị: C1 trước.)*
