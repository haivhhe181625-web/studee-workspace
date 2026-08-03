<!-- Tiếng Việt — PRD/SRS cho Phase 4. Viết cho người KHÔNG chuyên kỹ thuật vẫn hiểu chức năng.
     Bản kỹ thuật chi tiết: specs/exercise-engines/{design,tasks}.md · framing: docs/plan-phase4-exercise-engines.md -->
# PRD — Phase 4: Engine bài tập tương tác

- **Ngày:** 2026-07-17 · **Trạng thái:** Nháp để duyệt · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật đi kèm:** `specs/exercise-engines/design.md` & `tasks.md`

> Tài liệu này mô tả **chức năng sẽ có** bằng ngôn ngữ dễ hiểu. Người không chuyên kỹ thuật đọc để biết "hệ thống
> làm được gì"; kỹ thuật đọc kèm bản spec để biết "làm thế nào".

---

## 1. Mục đích & bối cảnh
Hiện mỗi Bài học đã có **lý thuyết + video + audio**, nhưng phần **luyện tập tương tác** mới chỉ có **2 loại**:
Luyện phát âm và Nói chuyện với AI. Bốn nhóm kỹ năng còn lại — **Nghe, Đọc, Viết, Ngữ pháp, Từ vựng** — học viên
mới chỉ *đọc/xem* chứ **chưa làm bài để được chấm**.

**Phase 4 bổ sung khả năng luyện tập có chấm điểm cho tất cả các kỹ năng đó**, để "học đủ 4 kỹ năng" thực sự trở
thành *luyện tập tương tác*, không chỉ nội dung tĩnh.

## 2. Phạm vi
**TRONG phạm vi** — thêm **3 loại bài tập mới** gắn vào Bài học:
1. **Trắc nghiệm (Quiz)** — *(Xây mới)* cho Nghe / Đọc / Ngữ pháp / Từ vựng (máy tự chấm). Riêng **Nghe** gồm 2 hình
   thức: **(a) Nghe & trả lời câu hỏi** và **(b) Nghe chép chính tả** (gõ lại đúng lời nghe).
2. **Viết (Writing)** — *(**Tích hợp thêm** — lõi chấm AI đã có sẵn)* học viên viết, **AI chấm** cho điểm + nhận xét.
3. **Nói có chấm điểm (Speaking)** — *(**Tích hợp thêm** — lõi chấm AI đã có sẵn)* học viên ghi âm trả lời, **AI chấm**
   phát âm/ngữ pháp/độ trôi chảy.

**Trạng thái triển khai (để đánh giá công sức):**
- **Xây mới:** engine **Quiz** (gồm Nghe-trả lời, Chép chính tả, Đọc, Ngữ pháp, Từ vựng) — dùng lại **ngân hàng câu
  hỏi** và **bộ chấm khách quan** có sẵn, nhưng cần dựng phần phục vụ bài tập theo Bài.
- **Tích hợp thêm:** **Viết** và **Nói** — **lõi chấm AI đã có** trong nền tảng, Phase 4 chỉ **nối vào Bài học**
  (không dựng lại bộ chấm).
- **Tính năng mở rộng (tùy chọn):** **AI gợi ý cải thiện cho bài Chép chính tả** — điểm **vẫn chấm chính xác** bằng
  so khớp; AI chỉ **thêm gợi ý** (vốn từ, âm sắc…). Có thể làm sau, không chặn phần cốt lõi.

**NGOÀI phạm vi (để sau):** mô phỏng đề thi đầy đủ TOEIC Part 1–7 / IELTS đúng cấu trúc; tự sinh câu hỏi bằng AI; hệ
soạn nội dung kéo-thả. Phase 4 dùng lại **ngân hàng câu hỏi** và **hệ chấm AI** đã có sẵn trong nền tảng.

## 3. Người dùng
| Vai | Nhu cầu |
|---|---|
| **Học viên** | Làm bài tập trong Bài học, được chấm ngay, biết đúng/sai và điểm để cải thiện. |
| **Đội học thuật (admin)** | Tạo bộ bài tập (chọn câu từ ngân hàng, đặt điểm đạt), gắn vào Bài, xuất bản. |

## 4. Tổng quan 3 loại bài tập (dễ hiểu)
- **Quiz:** một bộ câu hỏi (chọn đáp án, điền từ, nghe rồi trả lời, nối…). Làm xong bấm nộp → **máy chấm ngay**, hiện
  **điểm %** và **câu nào đúng/sai**. Dùng cho Nghe (có audio), Đọc (có đoạn văn), Ngữ pháp, Từ vựng.
  - **Nghe & trả lời câu hỏi:** nghe đoạn audio rồi chọn/điền đáp án cho các câu hỏi về nội dung.
  - **Nghe chép chính tả (dictation):** nghe (có giới hạn số lần phát) rồi **gõ lại đúng** câu/đoạn đã nghe; hệ thống
    **so khớp văn bản** (bỏ qua khác biệt nhỏ như hoa/thường, dấu câu, biến thể được chấp nhận). Vẫn là **máy tự chấm**.
    *(Mở rộng tùy chọn: sau khi có điểm chính xác, **AI đưa thêm gợi ý cải thiện** — vốn từ, âm sắc, lỗi hay gặp — mà
    **không** làm thay đổi điểm.)*
- **Viết:** *(tích hợp thêm)* đề bài yêu cầu viết một đoạn/bài. Học viên gõ bài → **AI chấm theo tiêu chí** (lõi có
  sẵn) → trả về **band điểm** + nhận xét từng tiêu chí.
- **Nói (chấm điểm):** *(tích hợp thêm)* đề bài yêu cầu nói (đơn thoại hoặc đọc theo mẫu). Học viên **ghi âm** → hệ
  thống chuyển giọng thành chữ và **AI chấm** phát âm/ngữ pháp/từ vựng/độ trôi chảy (lõi có sẵn) → **band điểm** + bản
  ghi lời nói. *(Khác "Nói chuyện với AI" hiện có — cái đó là hội thoại tự do, không chấm.)*

## 5. Yêu cầu chức năng (SRS)

**Nhóm A — Làm bài tập (Học viên)**
- **FR-A1** Trong một Bài học, học viên thấy danh sách bài tập kèm loại (Quiz / Viết / Nói) và mở được từng bài.
- **FR-A2 (Quiz)** Hệ thống hiển thị bộ câu hỏi (kèm audio/đoạn văn nếu có), **không lộ đáp án**. Học viên trả lời và nộp.
- **FR-A3 (Quiz)** Khi nộp, hệ thống **tự chấm** và hiển thị **điểm %**, số câu đúng, và **đánh dấu đúng/sai** từng câu.
- **FR-A3a (Nghe & trả lời)** Câu hỏi Nghe đi kèm **audio**; học viên nghe rồi trả lời. Có thể **giới hạn số lần phát** audio (do người tạo đặt).
- **FR-A3b (Nghe chép chính tả)** Học viên nghe rồi **gõ lại** nội dung; hệ thống **so khớp với đáp án**, chấp nhận
  khác biệt nhỏ (hoa/thường, dấu câu, các biến thể được duyệt) và **tự chấm** như các câu khách quan khác.
- **FR-A3c (Gợi ý cải thiện bằng AI — mở rộng, tùy chọn)** Sau khi bài Chép chính tả đã **có điểm chính xác** từ so
  khớp, hệ thống **có thể** gọi AI phân tích lỗi và **đưa gợi ý cải thiện** (ví dụ: vốn từ, âm sắc/trọng âm, lỗi lặp
  lại). **Điểm không phụ thuộc AI** — nếu AI không sẵn sàng thì vẫn có điểm và kết quả đúng/sai. *(Có thể triển khai
  sau phần cốt lõi.)*
- **FR-A4 (Viết — tích hợp thêm)** Hệ thống hiển thị đề; học viên nhập bài viết và nộp; **lõi chấm AI có sẵn** trả về
  **band điểm + nhận xét theo tiêu chí**. Bài quá ngắn sẽ được nhắc nhập thêm trước khi chấm.
- **FR-A5 (Nói — tích hợp thêm)** Hệ thống hiển thị đề; học viên **ghi âm** và nộp; **lõi chấm AI có sẵn** trả về
  **band điểm + bản ghi lời nói** (transcript). Ghi âm quá ngắn/không nghe được sẽ báo "chưa chấm được".
- **FR-A6** Mỗi loại bài tập có **ngưỡng đạt** do người tạo đặt. Đạt ngưỡng → bài tập tính là **hoàn thành**; vượt
  cao hơn → **thành thạo**. Học viên **làm lại được**; hệ thống giữ **kết quả tốt nhất** (điểm không bị tụt).

**Nhóm B — Gắn kết với khóa học & hồ sơ năng lực**
- **FR-B1** Bài học được tính **hoàn thành** khi **mọi** bài tập bắt buộc trong Bài đã đạt ngưỡng (kết hợp với luyện
  phát âm đã có). Hoàn thành Bài → **mở Bài kế** và **tăng % tiến độ khóa** (theo cơ chế tiến độ hiện có).
- **FR-B2** Kết quả mỗi bài tập **cập nhật hồ sơ năng lực** của học viên (điểm mạnh/yếu từng kỹ năng), phục vụ gợi ý
  và lộ trình thích nghi sau này.
- **FR-B3** Bài tập được gắn **kỳ thi mục tiêu** (Tổng quát / IELTS / TOEIC). Khóa theo kỳ thi nào sẽ ưu tiên bài tập
  phù hợp kỳ thi đó.

**Nhóm C — Soạn bài tập (Admin/Đội học thuật)**
- **FR-C1** Admin tạo **bộ Quiz** bằng cách **chọn câu hỏi từ ngân hàng có sẵn** theo kỹ năng / trình độ (CEFR) / kỳ
  thi; đặt **ngưỡng đạt**; **xuất bản**.
- **FR-C2** Admin tạo **bài Viết / Nói**: nhập đề, chọn **bộ tiêu chí chấm**, đặt **ngưỡng đạt**; **xuất bản**.
- **FR-C3** Admin **gắn** bộ bài tập đã xuất bản vào Bài học (khi soạn/sửa khóa). Chỉ bộ đã xuất bản mới gắn được;
  gắn sai (bộ chưa tồn tại/chưa xuất bản) sẽ báo lỗi khi kiểm tra khóa.

## 6. Yêu cầu phi chức năng (SRS)
- **NFR-1 (Không lộ đáp án):** đáp án, transcript mẫu, tiêu chí chấm **không bao giờ** gửi xuống máy học viên; chấm ở
  máy chủ.
- **NFR-2 (Kiểm soát chi phí AI):** chấm Viết/Nói dùng dịch vụ AI trả phí → **giới hạn theo hạn mức (quota)** mỗi tài
  khoản, như luyện phát âm hiện tại.
- **NFR-3 (Chạy được khi chưa có AI):** môi trường phát triển/thử nghiệm **không cần khóa dịch vụ AI** — hệ thống trả
  kết quả mô phỏng hợp lệ để kiểm thử.
- **NFR-4 (Tận dụng nền tảng):** dùng lại **ngân hàng câu hỏi**, **hệ chấm khách quan** và **hệ chấm AI viết/nói** đã
  có; **không** dựng lại từ đầu.
- **NFR-5 (Tính đúng):** Bài học đã hoàn thành/điểm đã đạt **không bị tụt** khi làm lại (chỉ tăng hoặc giữ nguyên).

## 7. Luồng người dùng (tóm tắt)
**Học viên:** vào Bài học → chọn bài tập → làm (chọn đáp án / viết / ghi âm) → nộp → xem điểm & đúng-sai/nhận xét →
đạt ngưỡng → Bài được đánh dấu hoàn thành → mở Bài kế.
**Admin:** tạo bộ bài tập (chọn câu/nhập đề + ngưỡng) → xuất bản → gắn vào Bài học → xuất bản khóa.

## 8. Tiêu chí hoàn thành (nghiệm thu)
- Học viên làm được **Quiz** và nhận điểm + đúng/sai ngay; làm **Viết** nhận band + nhận xét; làm **Nói** nhận band +
  transcript.
- Hoàn thành đủ bài tập trong Bài → Bài chuyển **hoàn thành**, **% khóa tăng**, **Bài kế mở**.
- Admin tạo & xuất bản được 3 loại bộ bài tập và gắn vào Bài; đáp án không lộ; chấm AI có giới hạn hạn mức; môi trường
  thử nghiệm chạy không cần khóa AI.

## 9. Ngoài phạm vi Phase 4 (nêu rõ để tránh hiểu nhầm)
Mô phỏng cấu trúc đề thi TOEIC Part 1–7 / IELTS task đầy đủ; tự sinh câu hỏi bằng AI; hệ soạn nội dung kéo-thả; đề
xuất bài tập thích ứng theo hồ sơ (→ tài liệu `specs/adaptive-path/plan.md`, làm sau/song song).

## 10. Thuật ngữ (cho người không chuyên)
- **Quiz:** bài trắc nghiệm/điền/nối, máy tự chấm.
- **Nghe chép chính tả (dictation):** nghe audio rồi gõ lại đúng lời nghe; máy so khớp văn bản để chấm.
- **Band điểm:** thang điểm kỹ năng (ví dụ IELTS 0–9) do AI ước lượng — *ước tính nội bộ, không phải điểm thi chính thức*.
- **Ngưỡng đạt:** mức điểm tối thiểu để tính "hoàn thành" một bài tập.
- **Ngân hàng câu hỏi:** kho câu hỏi có sẵn của nền tảng (đã phân loại theo kỹ năng/trình độ/kỳ thi).
- **Hồ sơ năng lực:** bản tóm tắt điểm mạnh/yếu từng kỹ năng của học viên, dùng để cá nhân hóa việc học.
- **CEFR:** khung trình độ A1–C2.
