<!-- Tiếng Việt — PRD/SRS cho "Hồ sơ năng lực + Lộ trình thích nghi" (bản cạnh spec).
     Bổ sung: Snapshot lộ trình đã giao (tách khỏi re-plan) + đổi-lộ-trình-chủ-động.
     Bản trước (docs/prd-adaptive-path.md) là nháp nội dung nền; file này là bản ổn định cạnh spec.
     Chi tiết kỹ thuật: design.md, data-model.md. -->

# PRD — Hồ sơ năng lực & Lộ trình học thích nghi

**Trạng thái:** Sẵn sàng triển khai · **Phiên bản:** 1.0 · **Ngày:** 2026-08-03  
**Chi tiết kỹ thuật:** [design.md](./design.md) · [data-model.md](./data-model.md) · [contracts/adaptive-path.md](../contracts/adaptive-path.md)

> **Ghi chú:** File `docs/prd-adaptive-path.md` là bản nháp nội dung nền. File này (specs/adaptive-path/prd.md) là bản chính thức cạnh spec, **bổ sung Snapshot lộ trình đã giao (mục 6) và Hành động đổi-lộ-trình-chủ-động (mục 7)**.

---

## 1. Mục đích & bối cảnh

Lộ trình học hiện tại là **cố định** (A1→A2, hoặc IELTS 5.0→6.0). Nhưng mỗi học viên **mạnh yếu không đều**: có người Đọc tốt nhưng Nghe yếu, Ngữ pháp ổn nhưng Từ vựng kém. Ép mọi người theo **một đường** thì người giỏi phần này phải học lại cái đã biết; người yếu phần kia không được luyện đủ.

**Mục tiêu:** Sau khi làm **bài đánh giá năng lực**, hệ thống dựng một **Hồ sơ năng lực** (điểm mạnh/yếu **từng kỹ năng**), rồi từ đó **tự xây một lộ trình cá nhân hóa** — **bốc đúng những Chuyên đề** phù hợp kỹ năng & trình độ của người đó từ **kho khóa học có sẵn**. Học viên vẫn có thể chọn học **lộ trình cố định**; lộ trình thích nghi là lựa chọn thông minh hơn.

---

## 2. Phạm vi

**TRONG phạm vi:**

1. **Hồ sơ năng lực** — hồ sơ điểm mạnh/yếu từng kỹ năng sinh từ bài đánh giá, **cập nhật dần** khi học viên luyện tập.
2. **Lộ trình thích nghi** — hệ thống **bốc các Chuyên đề** khớp (kỹ năng × trình độ) từ các khóa đã xuất bản để tạo một lộ trình riêng, **ưu tiên chỗ yếu**, **tự điều chỉnh** khi năng lực thay đổi.
3. **Snapshot lộ trình đã giao** — hệ thống lưu lại **ảnh chụp (snapshot)** cấu trúc lộ trình tại thời điểm giao cho học viên, để khi **nâng trình độ** thì **các chặng đã học không biến mất hay thay đổi**.
4. **Đổi lộ trình chủ động** — học viên có thể **chủ động yêu cầu cập nhật lộ trình** bằng cách bấm nút hoặc thay đổi mục tiêu; **không tự động đổi** sau khi nâng trình độ.

**NGOÀI phạm vi (để sau):** Tự sinh nội dung bằng AI; lập lịch học chi tiết theo ngày/giờ; nhiều mục tiêu song song; so sánh nhiều thuật toán chọn nội dung (A/B). **Không** thay đổi/sửa nội dung khóa cố định — chỉ **đọc & bốc**.

---

## 3. Người dùng

| Vai | Nhu cầu |
|---|---|
| **Học viên** | Xem mình mạnh/yếu gì; có một lộ trình đúng trình độ, tập trung vào chỗ yếu; theo dõi tiến bộ; cập nhật lộ trình khi muốn; thấy các chặng đã học không bị mất sau khi nâng trình độ. |
| **Đội học thuật (admin)** | Không cần thao tác mới — chỉ cần **gắn đúng kỹ năng & khoảng CEFR** cho Chuyên đề/Chặng khi soạn khóa (đã có sẵn ở luồng import/sửa). |

---

## 4. Tổng quan

- **(A) Hồ sơ năng lực:** một bản tóm tắt của mỗi học viên gồm: **trình độ tổng** (CEFR), **trình độ từng kỹ năng** (Nghe/Đọc/Viết/Nói/Ngữ pháp-Từ vựng), **các điểm yếu chi tiết** (vd: nghe ý chính ổn nhưng nghe chi tiết kém, thì quá khứ hoàn thành yếu…), và **các âm phát âm yếu**. Hồ sơ **tự cập nhật** khi học viên làm bài tập.

- **(B) Lộ trình thích nghi:** một danh sách **Chuyên đề có thứ tự**, được **bốc từ nhiều khóa** cho khớp hồ sơ. Ví dụ: học viên **Nghe** đang ở **A2** (đích B1) → hệ thống bốc **Chuyên đề Nghe ở Chặng A2→B1** từ các khóa và đưa vào lộ trình. Học xong, năng lực cập nhật → lộ trình **tự điều chỉnh** (thêm/bớt/đổi thứ tự).

- **(C) Snapshot lộ trình đã giao:** hệ thống **lưu ảnh chụp** cấu trúc lộ trình (danh sách các Chuyên đề, bài học, Chặng, mục tiêu) tại thời điểm **hệ thống giao lộ trình cho học viên**. Snapshot này **không thay đổi** khi học viên nâng trình độ (tím band-up); chỉ **thay đổi khi học viên chủ động cập nhật lộ trình** (bấm nút hoặc đổi mục tiêu).

- **(D) Đổi lộ trình chủ động:** học viên bấm nút **"Cập nhật lộ trình"** hoặc **thay đổi mục tiêu học tập** → hệ thống **tính lộ trình mới** dựa trên hồ sơ năng lực hiện tại + mục tiêu mới → **lưu ảnh chụp mới**, thay thế cái cũ. Cơ chế này đảm bảo học viên **luôn kiểm soát được quá trình thay đổi lộ trình**, không bị "lộ trình tự đổi" mà không nhận biết.

---

## 5. Quy tắc xây lộ trình (dễ hiểu)

Mỗi **Chuyên đề** trong khóa đã gắn sẵn **1 kỹ năng** và nằm trong **1 Chặng** có **1 khoảng trình độ** (vd A2→B1).
Hệ thống dùng đúng 2 thông tin đó để bốc:

> **Với mỗi kỹ năng của học viên:** lấy **trình độ hiện tại → bước kế** (vd Nghe A2 → B1) rồi **bốc các Chuyên đề của đúng kỹ năng đó, nằm trong Chặng có khoảng trình độ tương ứng**, từ **mọi khóa đã xuất bản** (ưu tiên khóa đúng kỳ thi mục tiêu nếu có). **Ưu tiên kỹ năng yếu nhất trước.**

- **Ví dụ (như yêu cầu):** kỹ năng bất kỳ ở mức **A2→B1** → bốc ra **Chuyên đề tương ứng với kỹ năng & mức đó** để đưa vào lộ trình.
- **Sắp thứ tự:** kỹ năng yếu nhất (cách đích xa nhất) trước → trình độ thấp trước (học nền trước) → **xen kẽ** các kỹ năng để không nhàm một mảng.
- **Bỏ qua** những Chuyên đề học viên đã **thành thạo** (theo hồ sơ) để không học lại.

---

## 6. Snapshot lộ trình đã giao

Tính năng **snapshot** là cơ chế để bảo vệ cấu trúc lộ trình **sau khi giao cho học viên**.

### Vấn đề được giải quyết

Nếu lộ trình **tính on-the-fly** (mỗi lần đọc) mà **không lưu snapshot**, thì:
- Học viên hoàn thành một Chuyên đề → năng lực thay đổi (lên A2 → B1) → lộ trình **tính lại**.
- Hệ thống bốc lại Chuyên đề → những Chuyên đề **đã học rồi (band-up) bị loại khỏi danh sách** vì được coi là "đã thành thạo" → **chặng hoặc bài học biến mất khỏi bản đồ**, gây nhầm lẫn.

**Snapshot giải quyết:** lộ trình **cố định tại thời điểm giao** (tính lần đầu + lưu snapshot) → chỉ **thay đổi khi học viên chủ động yêu cầu** (bấm nút "Cập nhật" hoặc đổi mục tiêu).

### Cách hoạt động

1. **Lần đầu tiên** học viên xem lộ trình:
   - Hệ thống **tính lộ trình thích nghi** dựa trên hồ sơ năng lực hiện tại.
   - **Lưu ảnh chụp (snapshot)** danh sách Chuyên đề, bài học, Chặng, mục tiêu vào cơ sở dữ liệu.
   - Học viên thấy danh sách lộ trình dựa trên **snapshot này**.

2. **Khi học viên hoàn thành Chuyên đề, nâng trình độ (band-up):**
   - Hệ thống **cập nhật hồ sơ năng lực** (thay đổi band).
   - **Snapshot lộ trình VẪN GIỮ NGUYÊN** (không tính lại).
   - FE hiển thị "đã đạt" cho các Chuyên đề đã mastery, nhưng **chặng vẫn visible** (không ẩn).

3. **Khi học viên chủ động cập nhật lộ trình** (bấm nút "Cập nhật lộ trình" hoặc đổi mục tiêu):
   - Hệ thống **tính lộ trình mới** dựa trên hồ sơ hiện tại + mục tiêu mới.
   - **Lưu ảnh chụp mới** (thay thế snapshot cũ).
   - Học viên thấy **danh sách lộ trình thay đổi** (có thể thêm/bớt/reorder Chuyên đề).

---

## 7. Hành động đổi-lộ-trình-chủ-động

Để đảm bảo học viên **luôn hiểu và kiểm soát** khi lộ trình thay đổi, việc cập nhật lộ trình **không tự động** — nó là **hành động chủ động**.

### Hai cách để học viên cập nhật lộ trình

1. **Đổi mục tiêu học tập** — học viên chỉnh sửa mục tiêu (ví dụ từ B1 lên B2) → **hệ thống tự động cập nhật lộ trình** cho khớp mục tiêu mới.

2. **Bấm nút "Cập nhật lộ trình"** — học viên bấm nút trên giao diện (trong phòng học/bản đồ lộ trình) → **hệ thống tính lộ trình mới dựa trên năng lực hiện tại** + mục tiêu hiện tại → **cập nhật snapshot**.

### Điều KHÔNG xảy ra tự động

- **Sau khi hoàn thành Chuyên đề:** lộ trình **không tự đổi** ngay. Snapshot **vẫn giữ**. Học viên tiếp tục thấy danh sách Chuyên đề như cũ (có thể thấy "đã đạt" kỹ năng nào, nhưng chặng vẫn có).
- **Sau khi band-up (nâng trình độ):** lộ trình **không tự tính lại**. Học viên phải chủ động bấm "Cập nhật lộ trình" nếu muốn.

### Lợi ích

- **Học viên kiểm soát:** không bị "lộ trình tự đổi" khiến bài học/chặng biến mất mà không nhận biết.
- **Rõ ràng & minh bạch:** khi cập nhật, hệ thống **thông báo rõ ràng** là lộ trình được cập nhật → các Chuyên đề mới được bổ sung hoặc điều chỉnh.
- **Bảo vệ tiến độ:** nếu học viên muốn **giữ nguyên lộ trình cũ** để hoàn thành hết các Chuyên đề đã bắt đầu, họ có thể chọn không cập nhật (hoặc cập nhật sau).

---

## 8. Yêu cầu chức năng (SRS)

### Nhóm A — Hồ sơ năng lực

- **FR-A1** Sau khi hoàn tất bài đánh giá năng lực, hệ thống **tạo/cập nhật Hồ sơ năng lực** cho học viên: trình độ tổng + **trình độ từng kỹ năng** + **điểm yếu chi tiết** + **âm phát âm yếu**.

- **FR-A2** Học viên **xem được hồ sơ** của mình (mạnh/yếu ở đâu) bằng ngôn ngữ dễ hiểu.

- **FR-A3** Hồ sơ **tự cập nhật** khi học viên làm bài tập (kết quả bài tập làm rõ hơn điểm mạnh/yếu). Khi **thi lại**, hồ sơ được tính lại theo kết quả mới.

- **FR-A4** Học viên đặt/điều chỉnh **mục tiêu** (ví dụ đạt B1, hoặc IELTS 6.5); mục tiêu lấy mặc định từ **lựa chọn ban đầu (onboarding)** và **kỳ thi của khóa** đang theo, cho phép **sửa tay**.

### Nhóm B — Lộ trình thích nghi

- **FR-B1** Hệ thống **sinh lộ trình cá nhân** bằng cách **bốc các Chuyên đề** khớp **(kỹ năng × khoảng trình độ)** từ **các khóa đã xuất bản**, theo quy tắc ở §5.

- **FR-B2** Lộ trình được **sắp thứ tự ưu tiên** (kỹ năng yếu nhất trước, trình độ thấp trước, xen kẽ kỹ năng). **Bỏ qua bài học đã thành thạo:** kỹ năng gap≥1 nhưng thạo (mastery≥0.8 mọi subskill) → **KHÔNG học lại**, thay vào đó mời **kiểm tra nâng band** để kéo score lên ngay (thay vì lặp lại nội dung cũ).

- **FR-B3** Học viên **theo lộ trình**: mở Chuyên đề → học các Bài & làm bài tập trong đó (dùng chung nội dung với khóa cố định).

- **FR-B4** Khi năng lực thay đổi (làm bài tập/thi lại), hệ thống **cập nhật hồ sơ năng lực** (không tự động cập nhật lộ trình); học viên có thể **chủ động cập nhật lộ trình** (bấm nút hoặc đổi mục tiêu).

- **FR-B5** Lộ trình **mở khóa tuần tự** hợp lý (học nền trước, nâng cao sau); phần phát âm yếu được **chèn bài luyện phát âm** phù hợp.

- **FR-B6** **Tiến độ dùng chung** với nội dung gốc: hoàn thành một Chuyên đề/Bài trong lộ trình thích nghi được ghi nhận như hoàn thành chính nội dung đó (không phải làm lại ở nơi khác).

### Nhóm C — Snapshot & Đổi-lộ-trình-chủ-động

- **FR-C1** Hệ thống **lưu snapshot** lộ trình (ảnh chụp danh sách Chuyên đề, bài học, mục tiêu) tại thời điểm **giao lộ trình cho học viên** hoặc **cập nhật lộ trình**.

- **FR-C2** Snapshot **không thay đổi** khi học viên **hoàn thành Chuyên đề hoặc nâng trình độ** (band-up). Hệ thống **overlay thông tin mastery mới** (ví dụ "đã đạt kỹ năng X") nhưng **cấu trúc lộ trình vẫn giữ nguyên**.

- **FR-C3** Học viên **chủ động cập nhật lộ trình** bằng hai cách: (1) **đổi mục tiêu** → tự động cập nhật, hoặc (2) **bấm nút "Cập nhật lộ trình"** → tính lại dựa trên năng lực hiện tại.

- **FR-C4** Khi cập nhật lộ trình, hệ thống **tính lộ trình mới**, **lưu snapshot mới** (thay thế cũ), và **thông báo rõ ràng** cho học viên về các thay đổi (thêm/bớt/reorder Chuyên đề).

### Nhóm D — Quan hệ với lộ trình cố định

- **FR-D1** **Lộ trình cố định giữ nguyên**: học viên vẫn chọn học thẳng theo mức/kỳ thi nếu muốn.

- **FR-D2** Lộ trình thích nghi **chỉ đọc & bốc** từ kho khóa; **không sửa** nội dung khóa. Một Chuyên đề có thể xuất hiện ở **cả hai**.

### Nhóm E — Lịch sử & Kiểm tra

- **FR-E1 (Band History):** hệ thống **ghi nhật ký** mỗi lần band thay đổi (từ đánh giá, kiểm tra, hoặc can thiệp admin), kèm **nguồn thay đổi** (assessment / checkpoint / ops) để học viên/admin theo dõi quá trình.

- **FR-E2 (Checkpoint):** kỹ năng đã thạo có **lựa chọn kiểm tra nâng band** (short assessment) thay vì học lại, tối ưu thời gian → nếu pass, band tăng 1 bậc (không học cả chuyên đề).

---

## 9. Yêu cầu phi chức năng (SRS)

- **NFR-1 (Cần dữ liệu đánh giá):** lộ trình thích nghi chỉ có ý nghĩa sau khi học viên **đã làm bài đánh giá**; chưa có → mời làm đánh giá hoặc dùng lộ trình cố định.

- **NFR-2 (Chính xác dần):** càng luyện nhiều, hồ sơ & lộ trình càng sát — **giàu nhất sau Phase 4** (có đủ bài tập Nghe/Đọc/Viết/Nói/Ngữ pháp/Từ vựng để đo).

- **NFR-3 (Không trùng lặp công sức):** tái dùng hồ sơ năng lực & cơ chế gợi ý **đã có sẵn**; chỉ **chuyển nguồn** nội dung sang **kho khóa học chính thức** và **dựng lộ trình có thứ tự**.

- **NFR-4 (Không đụng khóa cố định):** đọc-bốc, không ghi/sửa nội dung khóa.

- **NFR-5 (Snapshot immutable):** snapshot **không thay đổi tự động** khi band-up; chỉ **thay đổi khi cập nhật chủ động** → đảm bảo cấu trúc ổn định & dự đoán được.

---

## 10. Luồng người dùng (tóm tắt)

1. Học viên **làm bài đánh giá** → xem **Hồ sơ năng lực** (mạnh/yếu).
2. Hệ thống **đề xuất Lộ trình thích nghi** (các Chuyên đề theo kỹ năng & trình độ) → **lưu snapshot**.
3. Học viên **theo lộ trình**, làm bài tập → **hồ sơ tự cập nhật**, snapshot **vẫn giữ**.
4. (Tùy chọn) Khi muốn, học viên **bấm "Cập nhật lộ trình"** hoặc **đổi mục tiêu** → hệ thống **tính lộ trình mới + lưu snapshot mới**.
5. (Tùy chọn) Học viên **thi lại** → đo tiến bộ & làm mới lộ trình (bước 4).

---

## 11. Tiêu chí hoàn thành (nghiệm thu)

- Sau đánh giá, học viên thấy **Hồ sơ năng lực** đúng (mạnh/yếu theo kỹ năng).
- Hệ thống sinh **Lộ trình thích nghi** gồm các **Chuyên đề đúng (kỹ năng × trình độ)** bốc từ khóa có sẵn, đúng ví dụ §5 (A2→B1 bốc Chuyên đề tương ứng).
- **Snapshot lộ trình được lưu** tại lần đầu xem + khi cập nhật chủ động; **không tự động thay đổi** khi band-up.
- Học viên **hoàn thành Chuyên đề** → hồ sơ cập nhật (mastery tăng) + snapshot **vẫn giữ nguyên**.
- Học viên **nâng trình độ (band-up)** → snapshot **vẫn giữ nguyên**; FE hiển thị "đã đạt" cho kỹ năng tương ứng, nhưng chặng vẫn visible.
- Học viên **bấm "Cập nhật lộ trình" hoặc đổi mục tiêu** → hệ thống tính & lưu snapshot mới → danh sách lộ trình thay đổi rõ ràng.
- Học trong lộ trình được **ghi nhận tiến độ chung** với nội dung gốc; sau khi luyện, học viên có **tùy chọn cập nhật lộ trình** (không bắt buộc).
- **Lộ trình cố định** vẫn hoạt động song song, không bị ảnh hưởng.

---

## 12. Ngoài phạm vi

Tự sinh nội dung/câu hỏi bằng AI; lập lịch học theo ngày; nhiều mục tiêu song song; tối ưu thuật toán chọn nâng cao. → tài liệu/giai đoạn sau.

---

## 13. Thuật ngữ (cho người không chuyên)

- **Hồ sơ năng lực:** bản tóm tắt điểm mạnh/yếu từng kỹ năng của học viên, sinh từ bài đánh giá và cập nhật dần khi học viên làm bài tập.

- **Kỹ năng:** Nghe / Đọc / Viết / Nói / Ngữ pháp / Từ vựng.

- **CEFR:** khung trình độ A1–C2 (A1, A2, B1, B2… là các mức). Mỗi mức tương ứng với "band".

- **Chặng / Chuyên đề / Bài học:** các tầng của một khóa học.
  - **Chặng:** một nấc trình độ (ví dụ A2→B1 là 1 chặng).
  - **Chuyên đề:** một mảng kỹ năng trong 1 chặng (ví dụ "Nghe ý chính ở mức A2→B1").
  - **Bài học:** đơn vị học nhỏ nhất (ví dụ một video + bài tập).

- **Lộ trình cố định:** đường học tuyến tính có sẵn (A1→A2→B1, hoặc IELTS 5→6→7) — học viên đi lần lượt từ đầu đến cuối mà không có lựa chọn.

- **Lộ trình thích nghi:** đường học cá nhân hóa, ghép từ các Chuyên đề phù hợp hồ sơ năng lực + mục tiêu, tự điều chỉnh theo tiến bộ học viên.

- **Thành thạo (mastery):** mức nắm vững một kỹ năng hoặc nội dung; khi học viên thành thạo (điểm mastery ≥0.8) thì không cần học lại, có thể bỏ qua.

- **Snapshot (ảnh chụp lộ trình):** tấm hình "đóng băng" cấu trúc lộ trình (danh sách Chuyên đề, bài học, mục tiêu) tại thời điểm hệ thống giao lộ trình cho học viên. Snapshot **không tự động thay đổi** khi học viên nâng trình độ; chỉ **thay đổi khi học viên chủ động cập nhật** lộ trình (bấm nút hoặc đổi mục tiêu).

- **Cập nhật lộ trình chủ động:** hành động học viên chủ động bấm nút "Cập nhật lộ trình" hoặc thay đổi mục tiêu học tập → hệ thống tính lộ trình mới dựa trên năng lực hiện tại → lưu snapshot mới, thay thế snapshot cũ. **Không xảy ra tự động.**

- **Band-up (nâng trình độ):** học viên hoàn thành đủ chuyên đề / bài tập → trình độ tăng (ví dụ A2 → B1) → hồ sơ năng lực cập nhật, snapshot vẫn giữ.

- **Mastery-skip / Bỏ qua đã thành thạo:** khi hệ thống phát hiện kỹ năng đã được mastery (≥0.8) thì không bốc các chuyên đề của kỹ năng đó vào lộ trình; thay vào đó mời học viên kiểm tra nâng band (short assessment) để kéo điểm lên ngay.

---

## 14. Ngoài phạm vi & Ghi chú phiên bản

**File trước đây:** `docs/prd-adaptive-path.md` (bản nháp, bổ sung nên §6.Snapshot + §7.Đổi-lộ-trình-chủ-động).

**Phiên bản này:** bản ổn định cạnh spec, sẵn sàng triển khai. Chi tiết kỹ thuật xem design.md + data-model.md.
