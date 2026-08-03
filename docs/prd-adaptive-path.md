<!-- Tiếng Việt — PRD/SRS cho "Hồ sơ năng lực + Lộ trình thích nghi". Viết cho người KHÔNG chuyên kỹ thuật.
     Bản kỹ thuật/framing: specs/adaptive-path/plan.md. Quyết định đã chốt theo đề xuất (plan §7). -->
# PRD — Hồ sơ năng lực & Lộ trình học thích nghi

- **Ngày:** 2026-07-17 · **Trạng thái:** Nháp để duyệt · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật/khung:** `specs/adaptive-path/plan.md` (các quyết định §7 đã chốt)
- **Phụ thuộc:** dùng đủ dữ liệu nhất **sau/song song Phase 4** (bài tập tương tác sinh tín hiệu điểm mạnh/yếu).

> Tài liệu mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu. Kỹ thuật đọc kèm `specs/adaptive-path/plan.md`.

---

## 1. Mục đích & bối cảnh
Lộ trình hiện tại là **cố định** (đi thẳng A1→A2, hoặc IELTS 5.0→6.0). Nhưng mỗi học viên **mạnh yếu không đều**: có
người Đọc tốt nhưng Nghe yếu, Ngữ pháp ổn nhưng Từ vựng kém. Ép mọi người theo **một đường** thì người giỏi phần này
phải học lại cái đã biết, người yếu phần kia lại không được luyện đủ.

**Mục tiêu:** sau khi làm **bài đánh giá năng lực**, hệ thống dựng một **Hồ sơ năng lực** (điểm mạnh/yếu **từng kỹ
năng**), rồi từ đó **tự xây một lộ trình cá nhân hóa** — **bốc đúng những Chuyên đề** phù hợp kỹ năng & trình độ của
người đó từ **kho khóa học có sẵn**. Học viên vẫn có thể chọn học **lộ trình cố định**; lộ trình thích nghi là lựa
chọn thông minh hơn.

## 2. Phạm vi
**TRONG phạm vi:**
1. **Hồ sơ năng lực** — hồ sơ điểm mạnh/yếu từng kỹ năng sinh từ bài đánh giá, **cập nhật dần** khi học viên luyện tập.
2. **Lộ trình thích nghi** — hệ thống **bốc các Chuyên đề** khớp (kỹ năng × trình độ) từ các khóa đã xuất bản để tạo
   một lộ trình riêng, **ưu tiên chỗ yếu**, **tự điều chỉnh** khi năng lực thay đổi.

**NGOÀI phạm vi (để sau):** tự sinh nội dung bằng AI; lập lịch học chi tiết theo ngày/giờ; nhiều mục tiêu song song;
so sánh nhiều thuật toán chọn nội dung (A/B). **Không** thay đổi/sửa nội dung khóa cố định — chỉ **đọc & bốc**.

## 3. Người dùng
| Vai | Nhu cầu |
|---|---|
| **Học viên** | Xem mình mạnh/yếu gì; có một lộ trình đúng trình độ, tập trung vào chỗ yếu; theo dõi tiến bộ. |
| **Đội học thuật (admin)** | Không cần thao tác mới — chỉ cần **gắn đúng kỹ năng & khoảng CEFR** cho Chuyên đề/Chặng khi soạn khóa (đã có sẵn ở luồng import/sửa). |

## 4. Tổng quan
- **(A) Hồ sơ năng lực:** một bản tóm tắt của mỗi học viên gồm: **trình độ tổng** (CEFR), **trình độ từng kỹ năng**
  (Nghe/Đọc/Viết/Nói/Ngữ pháp-Từ vựng), **các điểm yếu chi tiết** (vd: nghe ý chính ổn nhưng nghe chi tiết kém, thì
  quá khứ hoàn thành yếu…), và **các âm phát âm yếu**. Hồ sơ **tự cập nhật** khi học viên làm bài tập.
- **(B) Lộ trình thích nghi:** một danh sách **Chuyên đề có thứ tự**, được **bốc từ nhiều khóa** cho khớp hồ sơ. Ví
  dụ: học viên **Nghe** đang ở **A2** (đích B1) → hệ thống bốc **Chuyên đề Nghe ở Chặng A2→B1** từ các khóa và đưa vào
  lộ trình. Học xong, năng lực cập nhật → lộ trình **tự điều chỉnh** (thêm/bớt/đổi thứ tự).

## 5. Quy tắc xây lộ trình (dễ hiểu)
Mỗi **Chuyên đề** trong khóa đã gắn sẵn **1 kỹ năng** và nằm trong **1 Chặng** có **1 khoảng trình độ** (vd A2→B1).
Hệ thống dùng đúng 2 thông tin đó để bốc:

> **Với mỗi kỹ năng của học viên:** lấy **trình độ hiện tại → bước kế** (vd Nghe A2 → B1) rồi **bốc các Chuyên đề của
> đúng kỹ năng đó, nằm trong Chặng có khoảng trình độ tương ứng**, từ **mọi khóa đã xuất bản** (ưu tiên khóa đúng kỳ
> thi mục tiêu nếu có). **Ưu tiên kỹ năng yếu nhất trước.**

- **Ví dụ (như yêu cầu):** kỹ năng bất kỳ ở mức **A2→B1** → bốc ra **Chuyên đề tương ứng với kỹ năng & mức đó** để đưa
  vào lộ trình.
- **Sắp thứ tự:** kỹ năng yếu nhất (cách đích xa nhất) trước → trình độ thấp trước (học nền trước) → **xen kẽ** các kỹ
  năng để không nhàm một mảng.
- **Bỏ qua** những Chuyên đề học viên đã **thành thạo** (theo hồ sơ) để không học lại.

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Hồ sơ năng lực**
- **FR-A1** Sau khi hoàn tất bài đánh giá năng lực, hệ thống **tạo/cập nhật Hồ sơ năng lực** cho học viên: trình độ
  tổng + **trình độ từng kỹ năng** + **điểm yếu chi tiết** + **âm phát âm yếu**.
- **FR-A2** Học viên **xem được hồ sơ** của mình (mạnh/yếu ở đâu) bằng ngôn ngữ dễ hiểu.
- **FR-A3** Hồ sơ **tự cập nhật** khi học viên làm bài tập (kết quả bài tập làm rõ hơn điểm mạnh/yếu). Khi **thi lại**,
  hồ sơ được tính lại theo kết quả mới.
- **FR-A4** Học viên đặt/điều chỉnh **mục tiêu** (ví dụ đạt B1, hoặc IELTS 6.5); mục tiêu lấy mặc định từ **lựa chọn
  ban đầu (onboarding)** và **kỳ thi của khóa** đang theo, cho phép **sửa tay**.

**Nhóm B — Lộ trình thích nghi**
- **FR-B1** Hệ thống **sinh lộ trình cá nhân** bằng cách **bốc các Chuyên đề** khớp **(kỹ năng × khoảng trình độ)** từ
  **các khóa đã xuất bản**, theo quy tắc ở §5.
- **FR-B2** Lộ trình được **sắp thứ tự ưu tiên** (kỹ năng yếu nhất trước, trình độ thấp trước, xen kẽ kỹ năng). **Bỏ
  qua bài học đã thành thạo:** kỹ năng gap≥1 nhưng thạo (mastery≥0.8 mọi subskill) → **KHÔNG học lại**, thay vào đó
  mời **kiểm tra nâng band** để kéo score lên ngay (thay vì lặp lại nội dung cũ).
- **FR-B3** Học viên **theo lộ trình**: mở Chuyên đề → học các Bài & làm bài tập trong đó (dùng chung nội dung với khóa
  cố định).
- **FR-B4** Khi năng lực thay đổi (làm bài tập/thi lại), hệ thống **cập nhật lại lộ trình** (thêm/bớt/đổi thứ tự Chuyên
  đề) cho sát điểm yếu hiện tại.
- **FR-B5** Lộ trình **mở khóa tuần tự** hợp lý (học nền trước, nâng cao sau); phần phát âm yếu được **chèn bài luyện
  phát âm** phù hợp.
- **FR-B6** **Tiến độ dùng chung** với nội dung gốc: hoàn thành một Chuyên đề/Bài trong lộ trình thích nghi được ghi
  nhận như hoàn thành chính nội dung đó (không phải làm lại ở nơi khác).

**Nhóm C — Quan hệ với lộ trình cố định**
- **FR-C1** **Lộ trình cố định giữ nguyên**: học viên vẫn chọn học thẳng theo mức/kỳ thi nếu muốn.
- **FR-C2** Lộ trình thích nghi **chỉ đọc & bốc** từ kho khóa; **không sửa** nội dung khóa. Một Chuyên đề có thể xuất
  hiện ở **cả hai**.

**Nhóm D — Lịch sử & Kiểm tra**
- **FR-D1 (Band History):** hệ thống **ghi nhật ký** mỗi lần band thay đổi (từ đánh giá, kiểm tra, hoặc can thiệp admin),
  kèm **nguồn thay đổi** (assessment / checkpoint / ops) để học viên/admin theo dõi quá trình.
- **FR-D2 (Checkpoint):** kỹ năng đã thạo có **lựa chọn kiểm tra nâng band** (short assessment) thay vì học lại, tối ưu
  thời gian → nếu pass, band tăng 1 bậc (không học cả chuyên đề).

## 7. Yêu cầu phi chức năng (SRS)
- **NFR-1 (Cần dữ liệu đánh giá):** lộ trình thích nghi chỉ có ý nghĩa sau khi học viên **đã làm bài đánh giá**; chưa
  có → mời làm đánh giá hoặc dùng lộ trình cố định.
- **NFR-2 (Chính xác dần):** càng luyện nhiều, hồ sơ & lộ trình càng sát — **giàu nhất sau Phase 4** (có đủ bài tập
  Nghe/Đọc/Viết/Nói/Ngữ pháp/Từ vựng để đo).
- **NFR-3 (Không trùng lặp công sức):** tái dùng hồ sơ năng lực & cơ chế gợi ý **đã có sẵn**; chỉ **chuyển nguồn** nội
  dung sang **kho khóa học chính thức** và **dựng lộ trình có thứ tự**.
- **NFR-4 (Không đụng khóa cố định):** đọc-bốc, không ghi/sửa nội dung khóa.

## 8. Luồng người dùng (tóm tắt)
Học viên **làm bài đánh giá** → xem **Hồ sơ năng lực** (mạnh/yếu) → hệ thống **đề xuất Lộ trình thích nghi** (các
Chuyên đề theo kỹ năng & trình độ) → học theo lộ trình, làm bài tập → **hồ sơ & lộ trình tự cập nhật** → (khi muốn)
**thi lại** để đo tiến bộ và làm mới lộ trình.

## 9. Tiêu chí hoàn thành (nghiệm thu)
- Sau đánh giá, học viên thấy **Hồ sơ năng lực** đúng (mạnh/yếu theo kỹ năng).
- Hệ thống sinh **Lộ trình thích nghi** gồm các **Chuyên đề đúng (kỹ năng × trình độ)** bốc từ khóa có sẵn, đúng ví dụ
  §5 (A2→B1 bốc Chuyên đề tương ứng).
- Học trong lộ trình được **ghi nhận tiến độ chung** với nội dung gốc; sau khi luyện, **lộ trình tự điều chỉnh**.
- **Lộ trình cố định** vẫn hoạt động song song, không bị ảnh hưởng.

## 10. Ngoài phạm vi
Tự sinh nội dung/câu hỏi bằng AI; lập lịch học theo ngày; nhiều mục tiêu song song; tối ưu thuật toán chọn nâng cao. →
tài liệu/giai đoạn sau.

## 11. Thuật ngữ (cho người không chuyên)
- **Hồ sơ năng lực:** bản tóm tắt điểm mạnh/yếu từng kỹ năng của học viên, sinh từ bài đánh giá và cập nhật dần.
- **Kỹ năng:** Nghe / Đọc / Viết / Nói / Ngữ pháp / Từ vựng.
- **CEFR:** khung trình độ A1–C2 (A2, B1… là các mức).
- **Chặng / Chuyên đề / Bài học:** các tầng của một khóa (Chặng = một nấc trình độ; Chuyên đề = một mảng kỹ năng trong
  Chặng; Bài học = đơn vị học).
- **Lộ trình cố định:** đường học tuyến tính có sẵn (A1→A2, IELTS 5→6).
- **Lộ trình thích nghi:** đường học cá nhân hóa, ghép từ các Chuyên đề phù hợp hồ sơ, tự điều chỉnh theo tiến bộ.
- **Thành thạo (mastery):** mức nắm vững một kỹ năng/nội dung; đủ thành thạo thì được bỏ qua để khỏi học lại.
