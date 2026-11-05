<!-- Sổ nháp ý tưởng phát triển KHUNG ĐỀ luyện thi (paper blueprint). Ghi kiểu SRS 1 dòng, chưa phải PRD chốt. Bàn sâu rồi mới nâng lên prd.md/spec.md. -->
# Luyện đề — Khung đề (config) · Sổ ý tưởng phát triển

> Mỗi mục 1 dòng kiểu SRS — *"Hệ thống cho phép / sẽ …"*. Trạng thái: 💡 ý tưởng · 🔎 đang bàn · ✅ chốt (→ đưa vào prd.md).
> Cách dùng: bạn ném ý tưởng, tôi note vào đây. Không tự mở rộng ngoài điều bạn nói.

## Hiện trạng (baseline — verified từ code 27/08)
- Khung đề = `PaperBlueprint`: cấp đề (`exam`, `skill`, `variant`, `totalItems`, `timeLimitSec`, `targetBandRange`, `status`) + `sections[]`.
- Section config: `cefr` (A1–C2, **filter cứng**) · `itemTypes[]` (filter cứng) · `itemCount` · `difficultyTier` (easy/mid/hard, **chỉ nhãn**) · `irtBand{min,max}` (lọc IRT `b`, **opt-in**) · `stimulusGrouping` (metadata, **chưa enforce**).
- Chỉ hỗ trợ `skill` = reading | listening. Bank có đủ 5 kỹ năng nhưng khung chưa dùng writing/speaking/use_of_english.

## Backlog ý tưởng
<!-- Thêm mục mới xuống dưới đây -->

### [B1] Khung đề CEFR kiểm tra đầu vào (dùng khung luyện đề làm nơi test trước) · 🔎
- **Mục tiêu:** Hệ thống cho phép tạo 1 khung đề kiểu CEFR để kiểm tra đầu vào các kỹ năng (như bản kiểm tra đầu vào đã có); làm & test trong phần **luyện đề** trước, rồi mới đưa vào bài kiểm tra đầu vào.
- **Phạm vi tạm:** CHƯA cần khung adaptive — khung luyện đề hiện tại đáp ứng đủ để dựng 1 khung này.

#### Cấu trúc khung (chốt dần)

```
KHUNG ĐỀ CEFR — KIỂM TRA ĐẦU VÀO
│
├─ [1] SKILLS  (taxonomy 2 tầng, KHÔNG ra câu hỏi riêng cho grammar/vocab)
│   ├─ Language Activities  — đo TRỰC TIẾP qua bài làm
│   │   ├─ Reading
│   │   ├─ Listening
│   │   ├─ Writing
│   │   └─ Speaking
│   └─ Language Resources   — đo GIÁN TIẾP (suy từ Activities qua tag item)
│       ├─ Grammar
│       └─ Vocabulary        (Phonology ✗ đã bỏ)
│
├─ [2+3] CEFR × COMPETENCY   (mỗi Skill × Level mở ra 1 chuỗi)
│   CEFR Level (A1–C1)
│     └─ Descriptor (can-do statement, gắn theo level)
│           └─ Competency (khả năng đo cụ thể)
│                 └─ map → tag item: subskill / grammarPoint  (+ irt.b = độ khó)
│
├─ [4] QUESTION DISTRIBUTION    — MA TRẬN (Skill × CEFR × Competency → số câu)  ✅ chốt dạng
├─ [5] DIFFICULTY DISTRIBUTION BY CEFR  — MA TRẬN (CEFR × Easy/Med/Hard %), áp THEO LEVEL  ✅ chốt
│        ├─ lưu %, KHÔNG lưu số câu · rounding+rải+fallback = Engine (là TARGET, không hard constraint)
│        └─ Difficulty = f(irt.b) TOÀN CỤC, độc lập CEFR (relative_to_cefr:false) → xem assembly-and-difficulty-allocation-logic.md
└─ [6] SCORING / CEFR MAPPING   — TÁCH khỏi blueprint; cut-point θ cứng (chuẩn 1 lần) + ngưỡng raw derive per-exam  🔎 đang bàn
```

#### Trạng thái từng trục

| # | Trục | Nội dung | Trạng thái |
|---|---|---|---|
| 1 | Skills | Activities (R/L/W/S) + Resources (Grammar, Vocab) | ✅ chốt taxonomy |
| 2 | CEFR | A1–C1, gắn Descriptor (không để trần) | ✅ chốt nguyên tắc |
| 3 | Competencies | Là đầu ra của Descriptor: Level→Descriptor→Competency | ✅ chốt chuỗi |
| 4 | Question distribution | **Ma trận** (Skill × CEFR × Competency → số câu), KHÔNG phải flat count | ✅ chốt dạng ma trận |
| 5 | **Difficulty distribution by CEFR** | **Ma trận CEFR × %(Easy/Med/Hard)** áp **theo level (a)**; lưu %, Engine làm tròn+rải+fallback; Difficulty=f(irt.b) toàn cục ([logic](assembly-and-difficulty-allocation-logic.md)) | ✅ chốt |
| 6 | Scoring / CEFR mapping | **Tách khỏi blueprint** (giai đoạn chấm, không phải ráp). Cut-point θ cứng chuẩn-1-lần + ngưỡng raw derive per-exam từ `irt.b`. MCQ vs rubric quy về cùng thang. MVP: báo CEFR per-skill, ngưỡng provisional dán nhãn "chưa chuẩn hóa" | 🔎 đang bàn |

**Bàn luận B1 — taxonomy Skills (Activities vs Resources):**
- ✅ Đồng ý: khớp mô hình CEFR gốc (communicative activities vs linguistic competences) VÀ khớp cấu trúc bank hiện có — item đã có tag cắt ngang: `subskill` (`grammar_tense`→Grammar, `collocation`/`word_form`→Vocabulary, `pronunciation`→Phonology; `gist`/`detail`/`inference`/`attitude`/`fluency`→Activities) + `grammarPoint`. → Đo Resources bằng tag trên item Activity, không cần section riêng cho grammar/vocab.
- ✅ Chốt: **BỎ Phonology**, Resources chỉ còn Grammar + Vocabulary (lý do: khó cải thiện qua học → không đo). Đồng thời giải quyết luôn caveat phủ-không-đều của Phonology.

**Bàn luận B1 — CEFR gắn descriptor:**
- ✅ Chốt phân cấp **CEFR Level → Descriptor → Competency**; trục "Competencies" gộp vào chuỗi này (không còn là trục rời).
- ⚠️ Quan sát data (gác tới lúc thiết kế schema): bank hiện có `subskill` là **enum phẳng, không gắn level** → không phân biệt B1/B2. "Descriptor" phải là **cấu trúc dữ liệu mới** (bộ can-do theo skill×level), rồi map descriptor ↔ competency ↔ `subskill`/`grammarPoint` của item.

**Ví dụ chuỗi CEFR Level → Descriptor → Competency → tag item (Reading):**

| CEFR | Descriptor (can-do) | Competency (khả năng đo) | Tag item (bank) |
|---|---|---|---|
| B1 | Understand main points | Nắm ý chính / gist | `subskill: gist` |
| B1 | Understand straightforward factual information | Định vị thông tin hiển ngôn | `subskill: detail` |
| B1 | Infer some meaning from context | Suy luận cơ bản theo ngữ cảnh | `subskill: inference` |
| B2 | Understand main ideas of complex texts | Nắm ý chính văn bản phức tạp/trừu tượng | `subskill: gist` + độ khó cao hơn (`irt.b`↑) |
| B2 | Understand writer's viewpoint | Nhận diện quan điểm/thái độ tác giả | `subskill: attitude` |
| B2 | Follow argumentation | Bám cấu trúc lập luận | `subskill: inference` |
| B2 | Infer implicit meaning | Suy luận hàm ẩn (sâu) | `subskill: inference` |
| B2 | Understand a wider range of vocabulary | Vốn từ rộng trong ngữ cảnh | Resource **Vocabulary** (`subskill: collocation`/`word_form`) |

- **Điểm mấu chốt của ví dụ:** cùng `subskill: gist` xuất hiện ở CẢ B1 và B2 → chứng minh vì sao Descriptor **phải gắn level**: tag item thô không đủ, level + descriptor mới phân biệt "gist văn bản đơn giản (B1)" vs "gist văn bản phức tạp (B2)". Difficulty (`irt.b`) là trục phụ tách trong cùng competency.
- ⚠️ Lưu ý 2 — **2 cơ chế đo khác nhau**: Reading/Listening = MCQ (đếm đúng/sai theo tag) → điểm Resource rạch ròi; Writing/Speaking = **rubric** (mở) → Grammar/Vocab/Phonology là *tiêu chí rubric*, không đếm câu. Scoring/CEFR mapping phải xử lý 2 nguồn này riêng.

**Bàn luận B1 — [4] Question distribution = ma trận:**
- ✅ Chốt: phân bổ số câu theo **ma trận (Skill × CEFR × Competency → Questions)**, KHÔNG dùng flat count theo skill (flat không kiểm soát được rải câu giữa level & competency).
- Ví dụ (ảnh): Reading/A2/Main idea→3 · Reading/A2/Detail→4 · Reading/B1/Inference→4 · Reading/B1/Vocabulary in context→3 · Reading/B2/Author's purpose→2 · Listening/A2/Specific information→4 · Listening/B1/Main idea→4 · Listening/B2/Inference→2.
- ⚙️ Hệ quả schema (gác tới lúc dựng): mỗi **hàng ma trận = 1 truy vấn bank** khi ráp đề (`skill + cefrLevel + competency→tag + count`) → ma trận **thay thế** khối `sections[]` thô hiện tại (`cefr`+`itemTypes`+`itemCount`). Điều kiện: mỗi Competency phải map được về tag item (`subskill`/`grammarPoint`) — dùng chung bảng map với Descriptor.

**Bàn luận B1 — [5] Difficulty distribution = lớp riêng, vuông góc CEFR:**
- ✅ Chốt: Difficulty là **lớp khác CEFR** — ma trận **CEFR × %(Easy/Medium/Hard)**. Ví dụ (ảnh): A1 80/20/0 · A2 40/50/10 · B1 20/60/20 · B2 10/50/40 · C1 0/40/60.
- ✅ Khớp code: `difficultyTier` (easy/mid/hard) + `TIER_B_BAND` → lọc `irt.b` (easy b<-0.5 · mid -0.5..0.5 · hard >0.5). Bank thiếu câu ở band nào đã có fallback top-up.
- ✅ **Q1 → chốt (a):** % khó áp **theo CEFR level** (trên tổng câu của level), KHÔNG mịn tới competency-cell. Thứ tự: Question Distribution (Skill×CEFR×Competency) → Difficulty Distribution (theo level) → Engine rải.
- ✅ **Nguyên tắc mới:** Difficulty lưu **% (tỷ lệ)**, KHÔNG lưu số câu cố định (số câu/level có thể lẻ, vd B1=7 → 1.4 câu). **Rounding + phân bổ = Test Assembly Engine**, Blueprint chỉ giữ tỷ lệ mong muốn.
- ⚙️ Logic Bước A (làm tròn largest-remainder) + Bước B (rải quota khó vào competency-cell — bài toán ma trận có biên + feasibility fallback) tách sang **[assembly-and-difficulty-allocation-logic.md](assembly-and-difficulty-allocation-logic.md)**.
- ✅ **Q1 fallback (thêm):** Difficulty là **TARGET, không hard constraint**. Thiếu item 1 ô → thang: (1) đúng target → (2) top-up cùng CEFR → (3) relax difficulty → (4) chọn câu gần nhất theo `irt.b`. Bắt buộc log lệch.
- ✅ **Q2 → chốt GLOBAL:** `Difficulty = f(irt.b)` trên **1 thang toàn cục**, độc lập CEFR (`type: global_irt, relative_to_cefr: false`). CEFR level ≠ statistical difficulty → "B2 Easy" có thể khó hơn "B1 Hard" (không phải bug). "Hard" = b cao toàn cục, KHÔNG phải "câu khó nhất người ở level đó gặp". Chưa tạo band relative-theo-level.
- ✅ **Đổi tên:** "Difficulty Distribution" → **"Difficulty Distribution by CEFR"** (nhấn: % theo từng level).

**Config chốt (skeleton — số minh hoạ, chốt số thật sau):**
```yaml
cefr_distribution:                 # DERIVE (chỉ đọc) — hệ tự cộng từ question_distribution để hiển thị/validate.
  # A1: 15%, A2: 25% ...            # KHÔNG phải ô nhập. Top-down (nhập % → auto sinh ô) CHƯA build.
question_distribution:             # [4] NGUỒN SỰ THẬT (input) — ma trận Skill × CEFR × Competency → count
  - { skill: reading, cefr: B1, competency: main_idea, count: 5 }
  - { skill: reading, cefr: B1, competency: inference, count: 4 }
difficulty_distribution:           # [5] by CEFR, lưu %  (số minh hoạ theo ảnh chốt gần nhất)
  A1: { easy: 70, medium: 30, hard: 0 }
  A2: { easy: 40, medium: 50, hard: 10 }
  B1: { easy: 20, medium: 60, hard: 20 }
  B2: { easy: 10, medium: 50, hard: 40 }
  C1: { easy: 0,  medium: 40, hard: 60 }
difficulty_model:                  # semantics Difficulty
  type: global_irt
  source: irt.b
  relative_to_cefr: false
```

**Bàn luận B1 — [6] Scoring / CEFR mapping:**
- ✅ **Tách khỏi Blueprint.** Blueprint = *đề cấu trúc thế nào* (dùng lúc RÁP, input). Scoring = *điểm nào → CEFR nào* (dùng lúc CHẤM, output). Blueprint KHÔNG chứa bảng ngưỡng.
- ✅ **Mapping KHÔNG phải bảng raw cứng toàn cục** (raw 70 ở đề dễ ≠ đề khó → dùng chung bảng là sai đo lường). NHƯNG cũng KHÔNG phải "người soạn tự đặt ngưỡng theo ý đồ đề" (vẫn tùy tiện, "B1" vô căn cứ).
- ✅ **Cách đúng — 2 lớp:** (1) **Cut-point CEFR trên thang năng lực θ** (cùng thang `irt.b`) = **cứng, chuẩn hóa 1 lần** bằng *standard setting* (Angoff/Bookmark/dựa dữ liệu). (2) **Ngưỡng raw derive PER-EXAM** từ θ-cut + `irt.b` các câu của đề (IRT test-equating) → so sánh được giữa các đề. ⇒ trả lời câu hỏi user: mapping *được tính ra cho từng đề*, nhưng NEO vào cut-point cứng, không tự do theo ý đồ.
- ⛓️ **Phụ thuộc [R1]:** làm chuẩn cần `irt.b` đã calibrate. Chưa calibrate → mọi mapping chỉ là **phỏng đoán chuyên gia, KHÔNG phải CEFR chính thức**.
- ✅ **Lưu ý 2 (giải quyết tại đây):** 2 nguồn điểm khác cơ chế — MCQ (Reading/Listening/Grammar/Vocab, đếm đúng/θ khách quan) vs **rubric** (Writing/Speaking, band mở). Scoring phải **quy 2 nguồn về cùng thang** trước khi gộp, hoặc **báo CEFR theo từng skill** rồi mới tổng hợp. KHÔNG cộng thẳng số-câu-đúng với điểm-rubric.
- 💡 **Đề xuất MVP (chưa calibrate):** báo **CEFR per-skill** trước; ngưỡng dùng tạm **provisional/expert, dán nhãn "ước lượng — chưa chuẩn hóa"** (không gọi CEFR chính thức); sau khi bank đủ lượt → calibrate → chuyển θ-based cut-scores. (Khớp ưu tiên "lấp đầy dữ liệu" đã chốt ở [R1].)

## Bàn luận / Quyết định
<!-- Ghi kết luận sau khi bàn: cái gì chốt, cái gì gác lại, lý do -->

## Rủi ro / Tiền đề
- **[R1] Trục Difficulty phụ thuộc calibrate `irt.b` — hiện chưa đáng tin.**
  - `irt.b` = độ khó thống kê trên thang toàn cục (b thấp=dễ, b cao=khó; b cao ↔ giữ cùng nhóm người làm thì tỷ lệ đúng thấp → đa số sai).
  - **Hiện trạng:** `b` là **provisional do chuyên gia/AI gán tạm** (item `ai-generated-reviewed` cũng vậy), CHƯA calibrate từ dữ liệu thật. Code ghi rõ MVP b tạm, sau pilot mới chuẩn.
  - **Điều kiện để tin được:** cần **người dùng thật làm** (~100+ lượt/câu, field `n_responses`) rồi chạy `rasch-calibrate.js` → b mới phản ánh độ khó thực.
  - **Hệ quả cho khung đầu vào:** khi bank chưa đủ lượt trả lời, filter Easy/Med/Hard (theo band `irt.b`) sai lệch → **fallback 4 bước chạy nhiều**, phân bổ difficulty lệch target. Phải lường trước khi đưa khung vào bài kiểm tra đầu vào thật; cân nhắc chỉ coi Difficulty là "định hình mềm" cho tới khi calibrate xong.
  - **✅ Hướng xử lý (user chốt):** **ưu tiên LẤP ĐẦY dữ liệu bank** hơn là xây fallback phức tạp cho tình huống thiếu. → fallback 4 bước giữ tối thiểu (đủ để không fail), không đầu tư thêm; mitigation thật = content pipeline sinh/nhập đủ item + thu lượt trả lời để calibrate. Vấn đề "thiếu bank / thiếu tổ hợp" cứ note, xử lý sau.

## Câu hỏi mở
- ~~[5-Q1] % Difficulty áp cấp level hay cấp ô?~~ ✅ **chốt (a) — theo CEFR level.**
- ~~[5-Q2] "Hard" toàn cục hay tương đối level?~~ ✅ **chốt GLOBAL (`relative_to_cefr: false`).**
- **[cfg-Q1] `cefr_distribution` vai trò?** → User làm rõ: `cefr_distribution` = **% mục tiêu tổng số câu theo CEFR level**; `question_distribution` = phân bổ chi tiết Skill×CEFR×Competency **và là nguồn số câu thực tế**. `cefr_distribution` **KHÔNG bắt buộc** nếu đã có `question_distribution`.
  - **Hai chiều nhập:** *Bottom-up* (nhập ô chi tiết → derive % CEFR) vs *Top-down* (nhập % CEFR + tổng câu → hệ tự sinh ô, cần thuật toán tự chia level→skill→competency).
  - **Rủi ro:** để cả 2 làm input = 2 nguồn sự thật → mâu thuẫn. Phải chọn 1 gốc.
  - ✅ **CHỐT (đề xuất A):** (1) `question_distribution` = **nguồn sự thật duy nhất (input)**; (2) `cefr_distribution` = **derive, chỉ đọc** (hiển thị + validate, KHÔNG phải ô nhập); (3) **CHƯA build Top-down** — cần trọng số tự chia skill/competency mỗi level mà content team chưa định → YAGNI, để dành khi đề lớn thực sự phiền.
- ~~[cfg-Q2] Số A1 chốt là gì?~~ ✅ **chỉ là minh hoạ** — số % difficulty thật chốt sau, không phải cam kết.
- **[logic-Q]** tie-break largest-remainder + ngưỡng sai lệch % khó trước khi báo thiếu bank — xem file logic.
- **[6-Q1]** Tổng hợp CEFR toàn bài từ per-skill thế nào? (lấy min? trung bình có trọng số? báo profile từng skill không gộp?) — ảnh hưởng cách trình bày kết quả đầu vào.
- **[6-Q2]** Phương pháp standard-setting cho θ-cut khi có dữ liệu: Angoff / Bookmark / anchor theo đề chuẩn? — chốt lúc chuẩn hóa, sau calibrate.
- **[6-Q3]** MVP có **chặn hiển thị "CEFR chính thức"** cho tới khi calibrate xong không, hay vẫn cho hiện kèm nhãn "ước lượng"? — quyết định sản phẩm.
