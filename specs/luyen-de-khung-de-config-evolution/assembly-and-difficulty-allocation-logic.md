<!-- Logic ráp đề + phân bổ difficulty cho KHUNG ĐỀ CEFR đầu vào. Tách khỏi sổ chính (khung-de-config-development-notes.md) vì đây là thuật toán engine, không phải config. -->
# Assembly & Difficulty Allocation — Logic engine

> Bổ trợ cho `khung-de-config-development-notes.md` (trục [4] Question distribution + [5] Difficulty distribution).
> **Nguyên tắc nền:** Blueprint chỉ mô tả **ý muốn** (số câu theo competency-cell + **tỷ lệ %** khó theo level). Mọi **làm tròn & phân bổ** là việc của **Test Assembly Engine**, KHÔNG lưu vào Blueprint.

## 1. Thứ tự áp dụng

```
Question Distribution           (Blueprint — số câu cố định)
        ↓  ma trận Skill × CEFR × Competency
Difficulty Distribution         (Blueprint — % Easy/Med/Hard THEO CEFR level)  ← Q1 = (a)
        ↓  engine làm tròn + rải
Đề thực tế (frozen items)
```

- **[4]** cho số câu **cố định** từng ô `(Skill × CEFR × Competency)`. Tổng các ô trong 1 level = N(level).
- **[5]** cho **% khó theo level** (áp trên toàn bộ N(level) câu của level đó, KHÔNG mịn tới từng ô — quyết định (a)).

## 2. Vì sao difficulty lưu % chứ không lưu số câu

Số câu 1 level có thể lẻ khi nhân %. VD B1 = 7 câu, 20/60/20 → Easy 1.4 / Med 4.2 / Hard 1.4 → không tồn tại 1.4 câu.
→ Blueprint lưu:
```yaml
difficulty_distribution:   # theo từng CEFR level
  B1: { easy: 20, medium: 60, hard: 20 }   # %
```
→ Engine mới quy ra số câu nguyên lúc generate.

## 3. Bước A — quota khó theo level (largest remainder)

N(level) = tổng số câu các competency-cell trong level (từ [4]).
1. raw = N × %(band). VD N=7: Easy 1.4 / Med 4.2 / Hard 1.4.
2. Lấy phần nguyên: 1 / 4 / 1 = 6. Dư = N − 6 = 1.
3. Chia phần dư cho các band có **phần thập phân lớn nhất** (largest remainder): Easy .4 = Hard .4 > (Med .2 đã tie-break). → +1 cho Easy (hoặc quy tắc tie cố định).
4. Kết quả nguyên, tổng = N. VD → Easy 1 / Med 4 / Hard 2 (tổng 7). ✅

> Rounding là trách nhiệm của Engine, không phải Blueprint. Blueprint chỉ giữ %.

## 4. Bước B — rải quota khó vào các competency-cell  ⚠️ phần lõi cần thiết kế

Trong 1 level giờ có **2 ràng buộc biên** đồng thời:
- **Hàng** = mỗi competency-cell có số câu cố định (từ [4]). VD B1: Reading×Main idea 5 · Reading×Inference 4 · Reading×Vocabulary 3 · Listening×Main idea 4 · Grammar×Tense 4 (tổng 20).
- **Cột** = quota Easy/Med/Hard của level (từ Bước A). VD 20% / 60% / 20% → 4 / 12 / 4.

→ Cần điền lưới **(competency-cell × difficulty)** sao cho **tổng hàng = số câu cell** và **tổng cột = quota khó**. Đây là bài toán **ma trận có biên cho trước** (matrix balancing / transportation).

**Hướng giải đề xuất (chốt sau):**
1. Rải tỷ lệ: mỗi cell nhận `cell_count × %(band)` rồi largest-remainder **theo cột** để khớp đúng quota cột, đồng thời khớp tổng hàng.
2. Ưu tiên giữ **đúng số câu competency** (hàng) hơn là giữ đúng % khó (cột) khi xung đột — vì competency là mục tiêu đo chính; difficulty là **hình dạng phụ (target)**.

### 4.1 Feasibility fallback — Difficulty là TARGET, không phải hard constraint ✅

Bank có thể thiếu item ở tổ hợp `(skill, cefr, competency, band)` — VD B1 cần Hard=4 nhưng bank chỉ có Reading:3 / Listening:1 / Grammar:0. **KHÔNG fail ngay.** Thang xuống dần:

```
1. Try exact target                        (skill, cefr, competency, band)
2. Thiếu → top-up từ CÙNG CEFR             (bỏ ràng buộc competency-cell, GIỮ level)
3. Vẫn thiếu → relax difficulty constraint (nới band)
4. Chọn câu GẦN NHẤT theo irt.b
```

→ Rất quan trọng khi bank chưa đủ lớn. Tái dùng đúng tinh thần fallback `irtBand` đã có trong `test-paper.service.js` (top-up khi band quá chặt). **Bắt buộc log** phần lệch, không nuốt im lặng.

## 5. Difficulty semantics — global IRT (chốt Q2) ✅

```yaml
difficulty_model:
  type: global_irt
  source: irt.b
  relative_to_cefr: false     # KHÔNG chuẩn hóa theo level
```

- `Difficulty = f(irt.b)` trên **1 thang toàn cục** (Easy/Med/Hard cắt từ `irt.b` — xem §6 map band).
- CEFR level và Difficulty là **2 dimension độc lập**: 1 item có thể `CEFR=B1, Difficulty=Hard` (gắn B1 nhưng `b` cao toàn cục) hoặc `CEFR=B2, Difficulty=Easy`. → **"B2 Easy" có thể khó hơn "B1 Hard"** — không phải bug, vì CEFR level ≠ statistical difficulty.
- **"Hard" = absolute/statistical difficulty**, KHÔNG phải "perceived difficulty for learners at this CEFR level". "A1 + Hard" = item target ở A1 nhưng `b` tương đối cao toàn cục — KHÔNG phải "câu khó nhất một người A1 gặp phải".
- **Không** tạo `A1-relative / A2-relative / B1-relative difficulty` ở giai đoạn này (DB đã có sẵn `irt.b` global → dùng luôn, sạch).

## 6. Ánh xạ xuống code hiện có
- `%(band)` → `irt.b` qua `TIER_B_BAND` (easy b<−0.5 · mid −0.5..0.5 · hard >0.5) trong `paper-blueprint.js`.
- Competency → tag item (`subskill` / `grammarPoint`) — dùng chung bảng map với Descriptor (xem sổ chính).
- Truy vấn bank mỗi ô = `{ skill, cefrLevel, subskill|grammarPoint, 'irt.b' ∈ band }` + `$sample` — mở rộng từ vòng lặp section trong `assembleFromBlueprint`.

## Câu hỏi mở
- Quy tắc tie-break largest remainder cố định thế nào (ưu tiên band nào khi phần thập phân bằng nhau)?
- Bước B: chấp nhận sai lệch % khó tới đâu trước khi báo "bank không đủ" thay vì fallback im lặng? (cần log, không nuốt).
