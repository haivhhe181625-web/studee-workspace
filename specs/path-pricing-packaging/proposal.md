<!-- Tài liệu ĐỀ XUẤT (brainstorm) cho BA tham khảo & lên kế hoạch.
     KHÔNG phải SRS. Chốt giá/gói cuối cùng do BA + PO quyết. -->

# Đề xuất — Bán lộ trình học theo Năng lực & Mục tiêu (Path Pricing & Packaging)

- **Ngày:** 2026-08-10 · **Trạng thái:** Draft (đề xuất) · **Đối tượng đọc:** BA / PO / đội kinh doanh
- **Mục đích:** đưa ra mô hình bán "lộ trình học" dựa trên điểm xuất phát (đánh giá năng lực) + mục tiêu người dùng, cách tính tiền theo số khóa, và một gói cao cấp (Premium) gồm các chức năng đề xuất thêm để BA đánh giá.
- **Ghi chú:** tài liệu ngắn gọn để tham khảo, không chốt con số giá. Phần offline là **giả định** để BA cân nguồn lực.

---

## 1. Ý tưởng cốt lõi

Người học **không mua từng khóa rời** — họ mua **một lộ trình** được hệ thống tự sinh từ:
- **Xuất phát:** band CEFR từ bài đánh giá đầu vào (M3) → lưu ở `StudentModel.band`.
- **Mục tiêu:** `targetCefr` / `targetScore` + chương trình (IELTS / TOEIC…).

Hệ thống tính lộ trình **đi qua bao nhiêu khóa** để đạt mục tiêu, rồi **tính tiền theo số khóa đó**. Càng xa mục tiêu → càng nhiều khóa → giá càng cao (minh bạch, người học hiểu vì sao).

---

## 2. Sinh lộ trình & tính GIÁ NỀN

### 2.1 Cách sinh danh sách khóa
Tái sử dụng cơ chế **Roadmap explorer** đã đặc tả (`course-driven-map`):
- Đầu vào: `program` + `cefrFrom` (band xuất phát) → `cefrTo` (mục tiêu).
- Đầu ra: **danh sách các khóa đã xuất bản phù hợp, đúng thứ tự** (vd A2→B1→B2).

Mỗi khóa (`CourseStructure`) đã có sẵn `program`, `phases[].cefrFrom/cefrTo` để lọc theo dải trình độ. → Chỉ cần **bổ sung trường giá cho khóa**.

### 2.2 Công thức giá nền
> **Giá nền lộ trình = Σ (giá từng khóa trong lộ trình)**

- Mỗi khóa có **giá riêng** (khóa nền rẻ, khóa nâng cao/dài đắt hơn) → phản ánh đúng khối lượng.
- Hiển thị minh bạch cho người mua:

```
Lộ trình của bạn: A2 → B2 (IELTS), đi qua 3 khóa
  • IELTS Foundation A2→B1      –  X₁
  • IELTS Intermediate B1→B2    –  X₂
  • IELTS Skills Booster        –  X₃
  ------------------------------------------
  Tổng (Gói Online):              X₁ + X₂ + X₃
```

### 2.3 Trường dữ liệu cần thêm (gợi ý cho kỹ thuật)
| Entity | Trường thêm | Ghi chú |
|---|---|---|
| `CourseStructure` | `price` (number, tiền tệ đơn) | giá bán 1 khóa |
| `CourseStructure` | `currency` (enum, mặc định 'VND') | đơn vị tiền |
| (tùy) `CourseStructure` | `originalPrice` / `discount` | phục vụ khuyến mãi sau này |

---

## 3. Hai gói bán

### Gói 1 — ONLINE (nền, giữ nguyên sản phẩm hiện có)
Toàn bộ giá trị số hóa đã/đang xây:
- Bản đồ game hợp nhất (`/adaptive`) gộp tất cả khóa trong lộ trình.
- Lý thuyết / video, quiz + flashcard **chấm tự động**.
- **Speaking 1-1 với AI**, chấm **Viết & Nói theo rubric bằng AI** (M3).
- **Đánh giá năng lực đầu vào** (M3) → xếp band + thẻ năng lực.
- **Lịch học theo ngày + nhắc học** (M4), thẻ năng lực tự cập nhật.

**Giá = Giá nền (mục 2.2).** Đây là gói mặc định mọi người học đều mua.

### Gói 2 — PREMIUM (các chức năng ĐỀ XUẤT THÊM — BA đánh giá)
> Premium = tất cả của Online **+** các chức năng dưới đây. Danh sách là **đề xuất để BA chọn/loại và định giá**, chưa cam kết triển khai.

**A. Tăng cường ONLINE (làm sâu giá trị số):**
1. **Mock test online mô phỏng** đầy đủ 4 kỹ năng theo định dạng thi thật (IELTS/TOEIC), có phân tích chi tiết.
2. **Người thật chữa Viết & Nói** (bắc cầu sau chấm AI) — feedback cá nhân hóa từ giáo viên.
3. **Báo cáo tiến độ nâng cao + dự báo band** đạt mục tiêu theo tốc độ hiện tại.
4. **AI tutor hỏi-đáp** giải thích lỗi sai, gợi ý học tiếp.
5. **Ưu tiên nội dung / mở khóa sớm** khóa nâng cao, tài nguyên bổ sung.

**B. Lợi ích OFFLINE (giả định — cần BA cân cơ sở/giáo viên):**
6. **Học offline tại trung tâm** — lớp nhóm theo chặng/level trong lộ trình.
7. **Thi thử offline mô phỏng phòng thi thật** (định kỳ theo tiến độ).
8. **Meeting với giáo viên** — 1-1 hoặc nhóm nhỏ: tư vấn lộ trình, chữa Speaking/Writing trực tiếp.
9. **Workshop kỹ năng / CLB tiếng Anh** offline định kỳ.
10. **Cam kết đầu ra / học lại miễn phí** nếu chưa đạt (tùy chính sách).

**Gợi ý cách định giá Premium (để BA cân):**
- **Cộng phụ phí theo số khóa** (khuyến nghị): Premium = Giá nền + (phụ phí × số khóa) — vì số buổi offline/thi thử/meeting **tỉ lệ với độ dài lộ trình**.
- Hoặc **nhân hệ số** trên giá nền (dự phòng, đơn giản hơn nhưng kém sát chi phí offline).

---

## 4. Edge cases cần BA xử lý (nghiệp vụ bán)
1. **Khóa trùng giữa các lần mua / mục tiêu:** người học đã sở hữu khóa → **không tính tiền lại** khi khóa đó nằm trong lộ trình mới.
2. **Nâng mục tiêu giữa chừng** (vd B1→B2 lên B1→C1): chỉ **tính bù (delta) các khóa mới**, không bán lại khóa đã mua.
3. **Giá khóa cố định vs khuyến mãi / combo:** có áp giảm giá theo lộ trình dài không?
4. **Năng lực & logistics offline:** số lớp, giáo viên, lịch, sức chứa — quyết định Premium có khả thi & quy mô nào.
5. **Hoàn tiền / chuyển nhượng / hết hạn:** chính sách cho gói đã mua.
6. **Đơn vị tiền & thuế / xuất hóa đơn.**

---

## 5. Vì sao mô hình này hợp lý
- **Minh bạch:** người học thấy rõ đi qua mấy khóa, trả bấy nhiêu.
- **Tận dụng hạ tầng sẵn có:** lộ trình + roadmap resolver + thẻ năng lực đã đặc tả (`course-driven-map`, M3, M4) — chỉ thêm lớp giá + gói.
- **KISS:** 2 gói dễ hiểu, dễ so sánh, dễ truyền thông; Premium là "nâng cấp giá trị" chứ không phải sản phẩm khác.
- **Mở rộng được:** sau này thêm add-on lẻ hoặc gói doanh nghiệp (B2B trung tâm) mà không phá cấu trúc.

---

## 6. Câu hỏi mở (cho BA / PO)
- [ ] Giá mỗi khóa do ai đặt và theo tiêu chí gì (độ dài, cấp độ, chi phí sản xuất)?
- [ ] Premium tính **phụ phí theo số khóa** hay **hệ số** trên giá nền?
- [ ] Trong danh sách Premium (mục 3.B/3.A), chức năng nào **giữ**, nào **loại**, nào **giai đoạn sau**?
- [ ] Offline: tự vận hành, hợp tác đối tác, hay thuê ngoài? (ảnh hưởng chia doanh thu & khả thi)
- [ ] Có gói **B2B cho trung tâm** (mua sỉ lộ trình cho học viên) không?
- [ ] Chính sách khi người học **không đạt mục tiêu** (cam kết đầu ra)?
