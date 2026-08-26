---
feature: flashcard-guided-learning-flow
status: draft
tier: lite
owner: BA
created: 2026-08-25
affects: [exe-api, exe-web]
brainstorm: plans/reports/from-brainstorm-to-plan-reading-progress-and-flashcard-learning-flow-report.md
---

# Spec — Luồng học flashcard có dẫn dắt

## 1. Mục tiêu & lý do
Luồng flashcard hiện tại KHÔNG thiếu tính năng (4 chế độ + SRS FSRS + gamification) nhưng **thiếu định hướng & phản hồi tiến bộ**: mở bộ → tự chọn 1/4 mode, tiến độ SRS chìm, không thấy "mình giỏi hơn tới đâu".

> **Đổi hướng 26/08 (bỏ bản funnel 25/08):** bản dẫn-dắt-1-nút (introduce→match→quiz→type→review) ép 1 khuôn, mất hồn lật thẻ, 1 phiên 20 từ ≈ 80 lượt. Thay bằng:

Feature này = **menu chọn cách học + thang màu theo năng lực**:
- Trang bộ thẻ là **menu**: Học từ mới · Ôn đến hạn · Kiểm tra / Luyện tập — người học tự chọn, KHÔNG ép chuỗi.
- Mỗi thẻ có **nấc màu = hoạt động khó nhất từng làm đúng** (Lật vàng < Ghép xanh mờ < Trắc nghiệm xanh lá < Tự luận tím); leo nấc khi làm bài khó hơn đúng.
- **Đến hạn thì tụt** (thẻ giữ nấc màu, tô sọc chồng lên); **ôn thì hết sọc**.
- Giữ nguyên engine FSRS + 4 runner; ModePicker cũ = "Luyện tự do".

## 2. Phạm vi

### Nấc màu năng lực (field mới `bestRecallLevel` 0–4 trên SRS row)
`0 chưa học` → `1 Lật (vàng)` → `2 Ghép (xanh mờ)` → `3 Trắc nghiệm (xanh lá) / Lật "Quá dễ"` → `4 Tự luận (tím)`.
- Chỉ LÊN khi trả lời đúng mode khó hơn (`max`); cập nhật bất kể cờ practice (nấc là trục riêng, không bơm FSRS).
- **Lapse** (ôn thật trả sai → relearning) → reset về nấc 1.
- "Giới thiệu 1 từ" = lần review đầu (new→learning) qua `applyReviewEvent`.

### Trang bộ thẻ = menu 4 hành động (KHÔNG ép chuỗi)
1. **Học từ mới** (≤ newLimit thẻ 'new', cỡ lô/lượt — KHÔNG trừ theo ngày): **Lật thẻ** 3 nút (Chưa thuộc/Đã nhớ/Quá dễ) → review đầu (`practice=false`); "Quá dễ" (easy) → nấc 3 + stability nhảy cao.
2. **Ôn đến hạn**: thẻ `due` — Lật thẻ nhắc lại (`practice=false`) → **lấp tụt** (làm mới `due`); quên → reset nấc 1.
3. **Kiểm tra / Luyện tập**: thẻ **đã học** (`getDue extra` lọc bỏ `state='new'`, gồm cả ⭐, trộn) → ModePicker chọn **Lật(học lại)/Ghép/Trắc nghiệm/Tự luận** → **leo nấc màu** (`practice=true`, KHÔNG đụng lịch/XP). Lật ở đây = ôn lại thẻ chưa tới hạn.

### Tụt & lấp
- **Đến hạn** (`due ≤ now`) → thẻ giữ nấc màu, **tô sọc chồng lên** cùng màu (nhị phân, không rời đoạn, không mờ opacity). Nhờ vậy Kiểm tra leo nấc vẫn thấy đổi màu dù thẻ đang tụt.
- Chỉ lượt đẩy FSRS (`practice=false`: Học mới, Ôn đến hạn) mới **lấp** (đẩy `due` ra → về đoạn màu). Kiểm tra/Luyện tự do leo màu nhưng không lấp.

### Giữ nguyên / tái dùng
- 4 runner (Flip/Type/Quiz/Match) + scheduler FSRS + `POST /cards/:id/review`: KHÔNG đổi cơ chế (chỉ chèn cập nhật `bestRecallLevel`).
- `ModePicker` cũ = "Luyện tự do".
- **Luyện tập/Kiểm tra KHÔNG cộng XP/streak** — chỉ lượt thật (Học mới + Ôn đến hạn) mới thưởng.

## 3. Ngoài phạm vi
- KHÔNG đổi engine FSRS/scheduler, cơ chế chấm 4 runner, deck/card CRUD, AI generate, library. (`bestRecallLevel` là field thêm mới nhưng KHÔNG đổi state machine/scheduler FSRS.)
- KHÔNG làm cross-deck "học hôm nay" tổng (đợt này theo TỪNG bộ).
- Gamification chỉ đổi tối thiểu: bỏ thưởng khi `practice=true` + dayKey→vnDayKey. KHÔNG thêm collection/cơ chế mới.

## 4. Repo ảnh hưởng
| Repo | Thay đổi |
|---|---|
| exe-api | Endpoint "hôm nay" /bộ (newCards≤limit + due + progress thang màu) · field `bestRecallLevel` + cập nhật trong `applyReviewEvent` (leo nấc/lapse reset) · setting new-limit/user · đếm giới thiệu theo ngày VN · bỏ thưởng practice |
| exe-web | `StudyMenu` (Học mới/Ôn đến hạn/Kiểm tra/Luyện tự do) · Lật thẻ 3 nút (again/good/easy) · thanh 4 màu + đoạn "đang tụt" kẻ sọc · new-limit chọn-rồi-Bắt-đầu · gỡ funnel `GuidedStudyRunner` |

## 5. Acceptance Criteria
Backend:
- [ ] `GET /flashcards/decks/:deckId/today` trả `{ newLimit, newRemaining, newCards:[≤newRemaining thẻ state='new'], dueCards:[+bestRecallLevel], progress:{total, introduced, levels:{flip,match,quiz,type}, slipping, mastered} }`.
- [ ] `newRemaining = min(newLimit, số thẻ state='new')` — cỡ lô/lượt, KHÔNG trừ số đã học trong ngày ("20 luôn là 20").
- [ ] `progress.mastered` = số thẻ `isMastered` (stability≥21) — trục ⭐ riêng, không cộng vào levels. Review response trả `crossedMastery=true` khi thẻ lần đầu vượt mốc.
- [ ] Field `bestRecallLevel` (0–4) trên SRS row; `applyReviewEvent` cập nhật `max(cũ, nấc-đúng)`: flip good→1, flip easy→3, match→2, quiz→3, type→4; sai không nâng; cập nhật bất kể `practice`.
- [ ] **Lapse** (practice=false, quên→relearning) reset `bestRecallLevel=1`; Kiểm tra (practice=true) sai KHÔNG reset.
- [ ] `progress.levels` chỉ đếm thẻ **chưa tụt** (`due>now`); `slipping` = thẻ đã học **đến/quá hạn** (`due≤now`). Bất biến: `flip+match+quiz+type+slipping = introduced ≤ total`.
- [ ] new-limit (cỡ lô/lượt) đọc/ghi per-user qua `PATCH /users/me` (mặc định 10, hợp lệ 5/10/20); user chưa có studyPlan vẫn set được.
- [ ] Học mới (review đầu, `practice=false`) chuyển `new→learning` đúng FSRS, KHÔNG double-schedule; Lật "Quá dễ" (easy) đẩy stability cao.
- [ ] Lượt `practice=true` (Kiểm tra/Luyện tự do) KHÔNG cộng XP/quest/streak; Học mới + Ôn đến hạn vẫn thưởng; guest phòng học chung giữ thưởng. Bucket theo ngày VN.
- [ ] KHÔNG đổi engine/scheduler FSRS, cơ chế chấm 4 runner.

Frontend:
- [ ] `/flashcards/[deckId]/study` = **menu 3 hành động**: Học từ mới · Ôn đến hạn · Kiểm tra / Luyện tập — user tự chọn, KHÔNG ép chuỗi.
- [ ] **Học từ mới**: bấm vào phiên Lật thẻ; nhãn "{newRemaining} từ". Control **"Từ mới/lượt"** (5/10/20) chỉ đặt cỡ lô, KHÔNG trừ theo ngày.
- [ ] **Lật thẻ** 3 nút: Chưa thuộc(again) / Đã nhớ(good) / Quá dễ(easy). Dùng cho cả Học mới lẫn Ôn đến hạn (`practice=false`).
- [ ] **Kiểm tra / Luyện tập**: mở thẳng ModePicker trên thẻ **đã học** (loại `state='new'`; gồm cả ⭐), `practice=true` → leo nấc màu. KHÔNG đi vòng qua màn "thẻ đến hạn". `StudyRunnerContainer` bị xóa.
- [ ] Thanh (độ sâu) **mỗi nấc 1 đoạn màu** (vàng/teal/xanh lá/tím); phần `slipping` của nấc **tô sọc chồng lên cùng màu** (không rời đoạn). Chip (hoạt động) = `activity` (số thẻ TỪNG làm đúng mỗi mode, tích lũy — KHÔNG về 0 khi leo nấc) + "Đang tụt" (`slipping`) + **⭐ Đã nhớ** (`mastered`); "còn X từ mới · Y thẻ đến hạn ôn".
- [ ] Khi review trả `crossedMastery=true` → hiện chúc mừng "🎉 Đã nhớ từ '…'". Nấc 4 chỉ khác màu, không hiệu ứng.
- [ ] Gỡ funnel `GuidedStudyRunner`/`buildGuidedPlan`; test 4 runner cũ vẫn xanh.

## 6. Quyết định đã chốt (PO 26/08 — thay bản 25/08)
- **Menu 3 hành động** (Học từ mới · Ôn đến hạn · Kiểm tra/Luyện tập) — bỏ funnel ép chuỗi; xóa `StudyRunnerContainer`.
- **Lật thẻ 3 mức**: Chưa thuộc / Đã nhớ / Quá dễ (→ FSRS easy, nhảy nấc 3).
- **Màu = nấc năng lực**: Lật vàng · Ghép xanh mờ · Trắc nghiệm xanh lá · Tự luận tím; field `bestRecallLevel` 0–4.
- **Đến hạn thì tụt** = sọc chồng lên màu nấc (nhị phân theo `due`, giữ nấc, không mờ dần); **ôn thì hết sọc**. Lapse → về nấc 1.
- **Ôn = làm mới** (hết tụt); **Kiểm tra = leo nấc** (không đụng lịch). Đi theo ngày VN.
- **⭐ "Đã nhớ" = `isMastered` (stability≥21)** — trục riêng chồng lên màu + chúc mừng khi vượt mốc. Sai sau ⭐ → xây lại nhanh theo FSRS (không code thêm).
- **Kiểm tra** lấy thẻ đã học (loại `state='new'`, gồm cả ⭐), trộn ngẫu nhiên; nấc 4 chỉ khác màu (KISS).
- new-limit = **cỡ lô/lượt** (mặc định 10, 5/10/20), lưu `studyPlan.flashcardNewPerDay` qua `PATCH /users/me`; `newRemaining=min(newLimit, thẻ 'new')`, KHÔNG trừ theo ngày. Nhãn "Từ mới/lượt".
- Practice/Kiểm tra KHÔNG cộng XP/quest/streak.

## 7. Câu hỏi mở
- Không còn — chi tiết kỹ thuật ở `design.md`.
