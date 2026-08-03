<!-- Tiếng Việt — PRD/SRS cho "Theo dõi tiến độ khóa học" (Phase 3).
     Viết cho người KHÔNG chuyên kỹ thuật (PO/BA/đội học thuật).
     Bản kỹ thuật: design.md, data-model.md, contracts/course-progress.md. -->
# PRD — Theo dõi tiến độ học tập trong khóa

- **Ngày:** 2026-08-03 · **Trạng thái:** Nháp để duyệt · **Đối tượng đọc:** PO/BA/đội học thuật + kỹ thuật
- **Bản kỹ thuật/khung:** `specs/course-progress/design.md` · `specs/course-progress/data-model.md`
- **Chi tiết hợp đồng API:** `specs/course-progress/contracts/course-progress.md`

> Tài liệu mô tả **chức năng sẽ có** cho học viên theo dõi tiến độ khóa — bằng ngôn ngữ dễ hiểu. Đội kỹ thuật đọc kèm `design.md`.

---

## 1. Mục đích & bối cảnh
Hiện tại, khi học viên tham gia một khóa, hệ thống chỉ hiển thị nội dung tĩnh — không biết học viên đó đã học bao nhiêu bài, bài nào đã xong, bài nào còn phải làm. Không có cơ chế để:
- **Biết ai đã ghi danh khóa** (để tính học phí, theo dõi bộ phận học thuật)
- **Ghi nhận hoàn thành từng bài** (để biết học viên tiến độ ra sao)
- **Mở khóa bài kế tiếp hợp lý** (đừng để học viên học lộn xộn, phải đúng trình tự)
- **Tính chuỗi ngày học** (stream — khuyến khích học liên tục, giữ động lực)
- **Biết bài tiếp theo cần làm** (giúp học viên không bị lạc, không phải tự tìm)

**Mục tiêu:** sau khi học viên **ghi danh khóa**, hệ thống **tự động theo dõi tiến độ** — bài nào làm xong, bài nào được mở, chuỗi ngày học bao lâu, bài tiếp theo là gì. Giúp học viên **có kế hoạch rõ ràng** và **hệ thống biết được tình hình thực tế** để hỗ trợ tốt hơn (nâng cấp mục tiêu, điều chỉnh lộ trình, v.v. ở Phase sau).

## 2. Phạm vi
**TRONG phạm vi:**
1. **Ghi danh khóa** — học viên chọn "Ghi danh" khóa → hệ thống ghi nhận; học viên khi đó mới bị gate (chỉ được mở khóa bài nếu làm xong bài trước).
2. **Theo dõi tiến độ** — ghi nhận khi học viên xem/hoàn thành bài học; tính toán phần trăm hoàn thành khóa/từng giai đoạn.
3. **Mở khóa tuần tự** — bài học mở lần lượt (bài 1 mở ngay, bài 2 mở sau khi xong bài 1, v.v.). Chỉ áp dụng khi đã ghi danh; nếu chưa ghi danh thì xem tự do (như Phase 2).
4. **Hoàn thành bài** — ghi nhận bài xong dựa trên bài tập:
   - Bài có bài **Phát âm** → hoàn thành khi làm xong hết bài phát âm (hoặc đạt trình độ tốt).
   - Bài không có bài phát âm → hoàn thành khi xem.
   - Ghi nhận thời điểm hoàn thành (để theo dõi tiến độ).
5. **Chuỗi ngày học (Streak)** — đếm liên tục mỗi ngày học viên làm bài; chuỗi dài nhất từng có (motivation).
6. **Hiện bài tiếp theo** — trên màn hình, chỉ cho học viên "bài tiếp theo cần làm là gì", thay vì để họ tự tìm.

**NGOÀI phạm vi (để sau):** tự sinh nội dung; chấm bài Nói; tính điểm chi tiết (XP, huy hiệu); lập lịch học tự động; admin theo dõi tiến độ học viên. Không sửa/thay đổi nội dung khóa — chỉ đọc dữ liệu để ghi nhận.

## 3. Người dùng
| Vai | Nhu cầu |
|---|---|
| **Học viên** | Ghi danh khóa; xem tiến độ mình (bao nhiêu % xong, bài nào mở/khóa); biết bài tiếp theo; duy trì chuỗi ngày học; hoàn thành bài khi làm xong bài tập. |
| **Đội học thuật (admin)** | Biết ai đã ghi danh khóa nào (tính học phí, hỗ trợ học viên). Không cần thao tác gì mới — chỉ cần xem báo cáo (Phase 4+). |

## 4. Tổng quan
Theo dõi tiến độ khóa gồm các phần:

- **(A) Ghi danh:** học viên nhấn nút "Ghi danh khóa" → hệ thống lưu lại, kể từ đó áp dụng gate (chỉ mở bài nếu xong bài trước). Idempotent (nhấn lại không ghi danh lần 2).

- **(B) Tiến độ khóa:** tóm tắt bao nhiêu phần trăm khóa đã xong, bao nhiêu bài completed/mastered/still learning, phần trăm mỗi giai đoạn (ví dụ giai đoạn 1 xong 80%, giai đoạn 2 xong 30%).

- **(C) Trạng thái từng bài:**
  - `Chưa bắt đầu` — bài bị khóa (chưa làm xong bài trước).
  - `Đang học` — xem/làm bài nhưng chưa xong.
  - `Hoàn thành` — xong bài nhưng cần luyện thêm.
  - `Thành thạo` — làm rất tốt, không cần luyện thêm.
  - `Unlocked` — bài được mở khóa (xem được).

- **(D) Mở khóa bài kế:** khi hoàn thành bài, tự động mở bài tiếp theo (nếu có) để học viên không phải tìm.

- **(E) Chuỗi ngày học:** mỗi ngày học viên làm bài, chuỗi +1. Nếu không làm trong 1 ngày, chuỗi lại từ 1. Ghi nhận chuỗi hiện tại + chuỗi dài nhất từng có.

- **(F) Bài tiếp theo:** hiển thị bài unlocked đầu tiên chưa completed → học viên biết phải làm cái gì tiếp.

## 5. Quy tắc tiến độ (dễ hiểu)
Mỗi **bài học** nằm trong một **giai đoạn** (ví dụ giai đoạn A1, A2) và có **thứ tự** cụ thể.

Hệ thống mở bài theo thứ tự:
> **Bài 1 luôn được mở ngay khi ghi danh. Bài tiếp theo (2, 3, 4…) chỉ được mở khi bài trước đó đã `completed` hoặc `mastered`.**

- **Hoàn thành bài:** được ghi nhận từ **hoàn thành bài tập trong bài đó**:
  - Nếu bài có **Phát âm:** hoàn thành khi làm xong toàn bộ bài phát âm (theo tiến độ riêng của phát âm).
  - Nếu bài **không có phát âm:** hoàn thành ngay khi xem bài.
  - Nếu hoàn thành hết bài (thành thạo các phần), nâng lên trạng thái `mastered`.
  - Ghi nhận **thời điểm hoàn thành** (để báo cáo sau).

- **Chuỗi ngày học:** khi xong bài (bất cứ bài nào), chạm vào streak → ngày đó được đếm. Ngày hôm sau, nếu làm bài nữa → streak +1. Nếu không làm hôm sau → streak lại từ 1 hôm sau đó. Ghi nhận **streak hiện tại** (ngày liên tục) + **streak dài nhất** (động lực).

## 6. Yêu cầu chức năng (SRS)

**Nhóm A — Ghi danh khóa**
- **FR-A1** Học viên có nút **"Ghi danh"** trên trang chi tiết khóa (khi chưa ghi danh). Sau khi ghi danh → nút biến thành **"Đã ghi danh"** hoặc ẩn đi.
- **FR-A2** Nhấn "Ghi danh" → hệ thống lưu học viên đó vào khóa, từ đó áp dụng gate (chỉ mở bài nếu xong bài trước). Ghi danh là **idempotent** (nhấn lại không lỗi, vẫn trả trạng thái hiện tại).
- **FR-A3** Chỉ ghi danh được khóa **đã xuất bản** (live). Khóa nháp hoặc lưu trữ không cho ghi danh.

**Nhóm B — Tiến độ khóa**
- **FR-B1** Học viên **xem tiến độ khóa** (header trang chi tiết hoặc sidebar) gồm: **% hoàn thành tổng**, **số bài xong/tổng bài**, **% hoàn thành từng giai đoạn** (ví dụ: Giai đoạn 1: 80%, Giai đoạn 2: 30%).
- **FR-B2** Tiến độ **tự cập nhật** khi học viên hoàn thành bài (không cần reload, hoặc reload tự động).

**Nhóm C — Trạng thái & Mở khóa bài**
- **FR-C1** Mỗi bài hiển thị **trạng thái** (badge):
  - `Chưa bắt đầu` — bài bị khóa (chưa làm xong bài trước).
  - `Đang học` — đã xem nhưng chưa hoàn thành.
  - `Hoàn thành` — xong nhưng không phải mastered.
  - `Thành thạo` — làm rất tốt.
  - Bài khóa → **link bị disable** (không click vào được).
- **FR-C2** **Bài 1 luôn unlocked** (không khóa). Bài kế tiếp chỉ mở khi bài trước `completed` hoặc `mastered`.
- **FR-C3** Khi học viên xem bài (bất kể unlocked hay không), tự động đánh dấu `in_progress` (đang học).

**Nhóm D — Hoàn thành bài**
- **FR-D1** Khi học viên **hoàn thành bài tập trong bài:**
  - Bài có **bài Phát âm:** hoàn thành khi làm xong toàn bộ bài phát âm (dựa trên tiến độ phát âm có sẵn, không chấm lại).
  - Bài **không có phát âm:** hoàn thành ngay khi xem bài.
- **FR-D2** Khi bài chuyển sang `completed`, **tự động mở bài kế tiếp** (nếu có). Bài được mở → học viên có thể click vào.
- **FR-D3** Ghi nhận **thời điểm hoàn thành** (để báo cáo tiến độ sau này).
- **FR-D4** Nếu **tất cả bài** đều đạt `completed` hoặc `mastered` → khóa chuyển sang trạng thái **`completed`** (xong khóa).

**Nhóm E — Chuỗi ngày học (Streak)**
- **FR-E1** Hệ thống ghi nhận **streak hiện tại** (số ngày liên tục học viên làm bài) + **streak dài nhất** (motivation).
- **FR-E2** Khi học viên **hoàn thành bài bất kỳ**, chạm vào streak:
  - Nếu hôm nay là ngày đầu tiên làm bài → streak = 1.
  - Nếu hôm qua đã làm bài, hôm nay làm lại → streak +1.
  - Nếu không làm bài trong 1 ngày → ngày hôm sau, streak lại từ 1.
  - Cập nhật `longest` = max(current, longest).
- **FR-E3** Hiển thị streak trên UI (đằng sau mục tiêu, phần thành tích, v.v.).

**Nhóm F — Bài tiếp theo**
- **FR-F1** Hiển thị **bài tiếp theo cần làm** (bài unlocked đầu tiên chưa completed) trên trang chi tiết khóa, kèm nút **"Học tiếp"** hoặc link trực tiếp.
- **FR-F2** Nếu đã xong hết khóa → thay bằng **"Bạn đã hoàn thành khóa"** (không có bài tiếp theo).
- **FR-F3** Cập nhật khi bài hoàn thành (không cần reload).

**Nhóm G — Gate và quyền truy cập**
- **FR-G1** **Chưa ghi danh** → xem tự do (giữ hành vi Phase 2), không tiến độ, không gate, tất cả bài mở.
- **FR-G2** **Đã ghi danh** → áp dụng gate, chỉ bài unlocked mới click được, bài khóa bị disable + hiển thị lý do ("Bạn cần hoàn thành bài X trước").

## 7. Yêu cầu phi chức năng (SRS)
- **NFR-1 (Gate chỉ sau ghi danh):** trước khi ghi danh, xem tự do (không gate). Gate bắt đầu sau khi ghi danh.
- **NFR-2 (Nhanh & chính xác):** tiến độ cập nhật nhanh khi hoàn thành bài; không có delay lớn (< 1 giây).
- **NFR-3 (Tái dùng phát âm):** không chấm lại bài phát âm — dùng kết quả có sẵn từ phát âm (không bản ghi riêng).
- **NFR-4 (Không đụng nội dung):** chỉ đọc tiến độ bài tập phát âm; không sửa/xóa nội dung khóa hoặc bài tập.
- **NFR-5 (Bảo vệ cơ cấu khóa):** khi khóa **đã có học viên ghi danh**, admin **không được** phép gỡ xuất bản → sửa cấu trúc (để bảo vệ tiến độ đã ghi).

## 8. Luồng người dùng (tóm tắt)
1. Học viên **vào trang khóa** → thấy nút **"Ghi danh"**.
2. Nhấn **"Ghi danh"** → hệ thống lưu, nút biến thành **"Đã ghi danh"**.
3. Từ đó, **lần đầu vào trang** → thấy:
   - **Thanh % tiến độ** (ví dụ 0% xong).
   - **Bài 1 unlocked** (xem được).
   - **Bài 2+ khóa** (không click được).
   - **"Bài tiếp theo: Bài 1"** + nút **"Học tiếp"**.
4. Nhấn **"Học tiếp"** → vào bài 1, **tự động xem đánh dấu `in_progress`**.
5. Làm xong bài 1 (bài phát âm) → **tự động chuyển `completed`**, hệ thống:
   - **Mở bài 2**.
   - **Cập nhật % tiến độ** (ví dụ 25% xong).
   - **Cập nhật streak** (+1).
   - **Cập nhật bài tiếp theo** → hiển thị bài 2.
6. Lặp lại quá trình với bài 2, 3, v.v. cho đến khi **xong hết khóa** → trạng thái khóa = `completed`.

## 9. Tiêu chí hoàn thành (nghiệm thu)
- Học viên có nút **"Ghi danh"** khi chưa ghi danh; **nhấn ghi danh** → nút ẩn/đổi (chỉ ghi danh 1 lần).
- Sau ghi danh, **gate hoạt động:** bài 1 unlocked, bài 2+ khóa; khóa link khi khóa → disable link.
- Khi xem bài → **tự động `in_progress`** (không cần gọi API riêng).
- Khi xong bài tập (phát âm) → **tự động `completed`**, mở bài kế, cập nhật %, streak +1, bài tiếp theo cập nhật.
- **Tiến độ % chính xác:** `completed_lessons / total_lessons` (làm tròn).
- **Phần trăm từng giai đoạn** tính đúng (ví dụ giai đoạn 1 có 3 bài, 2 xong → 67%).
- **Streak hoạt động:** mỗi ngày làm bài, streak +1; không làm trong 1 ngày → streak reset lại từ 1. Hiển thị streak hiện tại + dài nhất.
- **Bài tiếp theo đúng:** luôn chỉ bài unlocked đầu tiên chưa completed; khi bài xong → cập nhật ngay.
- **Guard:** nếu khóa đã có ghi danh → admin không được gỡ xuất bản (gặp lỗi 409 kèm thông báo rõ ràng).

## 10. Ngoài phạm vi
Chấm bài Nói; tính điểm chi tiết (XP, huy hiệu); lập lịch học tự động (ngày/giờ); admin theo dõi tiến độ học viên (dashboard báo cáo, v.v.); điều chỉnh lộ trình dựa trên tiến độ (Phase 4+). → tài liệu/giai đoạn sau.

## 11. Thuật ngữ (cho người không chuyên)
- **Ghi danh:** đăng ký tham gia khóa (như "join class"). Từ đó, gate áp dụng.
- **Gate / Mở khóa:** chỉ cho phép xem bài nếu làm xong bài trước (tuần tự).
- **Tiến độ:** phần trăm + số bài xong trong khóa; cập nhật khi hoàn thành bài.
- **Trạng thái bài:** chưa bắt đầu (khóa) / đang học / hoàn thành / thành thạo.
- **Hoàn thành bài:** ghi nhận bài xong (dựa trên bài tập; nếu là phát âm → khi xong phát âm).
- **Thành thạo:** bài làm rất tốt (không cần luyện thêm); nâng cấp của `hoàn thành`.
- **Unlocked:** bài được mở khóa (xem được).
- **Locked:** bài bị khóa (không click vào).
- **Giai đoạn:** nấc trình độ của khóa (ví dụ A1, A2, B1).
- **Bài học:** đơn vị nhỏ nhất trong giai đoạn (một bài phát âm, một bài ngữ pháp, v.v.).
- **Bài tiếp theo:** bài unlocked đầu tiên chưa hoàn thành → gợi ý học viên học cái gì tiếp.
- **Streak / Chuỗi ngày học:** số ngày liên tục học viên làm bài; nhằm khuyến khích tính nhất quán. Chuỗi **hiện tại** = từ ngày nào tính. Chuỗi **dài nhất** = kỷ lục (motivation).
- **Phát âm:** bài tập phát âm tiếng Anh (IPA), có chấm tự động. Hoàn thành bài Phát âm = xong toàn bộ exercises phát âm trong bài đó.
