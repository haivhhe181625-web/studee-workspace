<!-- Demo test-spec CỤ THỂ cho bài kiểm tra đầu vào CEFR, đủ 4 kỹ năng nhưng ngắn.
     Là bản instantiate của khung 6 trục (xem khung-de-config-development-notes.md) +
     logic ráp/chấm (assembly-and-difficulty-allocation-logic.md).
     TÀI LIỆU REVIEW — chưa wire code/seed. -->
# Demo Spec — CEFR Placement Entry Test (4 kỹ năng, bản ngắn)

> ⚠️ **ĐÃ ĐỔI HƯỚNG** (xem `placement-decisions-handoff.md`): bản dùng để làm giờ là **R/L + Use of English, tạm bỏ W/S**, overall = **avg-renormalize** (không phải min). File này giữ làm tham chiếu khung 4-kỹ-năng; các mục "min" / "đủ 4 kỹ năng" đã bị override.

> **Mục đích:** đề đầu vào — **nhanh** nhưng **bao quát R/L/W/S**, ra **CEFR đủ uy tín** để học viên thấy đúng trình độ.
> **Trạng thái:** bản demo để review, CHƯA wire code/seed.
> **Engine đích:** module `assessment`, `testType='placement'`, `assemblyMode='fixed'` (non-adaptive) — KHÔNG phải luyện-đề (`PaperBlueprint` chỉ R/L, không ra CEFR).

---

## 0. Chốt nền (từ phát hiện code)
- Đề đầu vào **đã có engine sẵn** ở `assessment`: chấm θ→band→CEFR (`band-scoring.js`), standard-setting (`angoff.service.js`), preset `CORE_PLACEMENT_PRESET`, chế độ `fixed` non-adaptive.
- Demo này = **thêm 1 preset placement** trên engine đó. Không viết engine mới.
- Grammar/Vocab (Resources) **đo gián tiếp qua tag** trên item R/L (đúng khung [1]) — KHÔNG mở section riêng → giữ đúng 4 kỹ năng, ngắn hơn. *(Muốn tín hiệu Grammar/Vocab sắc hơn thì tách 1 section Use-of-English sau — engine hỗ trợ.)*

## 1. Cấu hình tổng
| Field | Giá trị | Ghi chú |
|---|---|---|
| testType | `placement` | |
| assemblyMode | `fixed` | non-adaptive, số câu cố định |
| CEFR scope | **A1–C1** | khớp khung |
| Skills | Reading, Listening, Writing, Speaking | 4 kỹ năng |
| includeWritingSpeaking | `true` | |
| Tổng thời lượng mục tiêu | **~35–40 phút** | "đủ ngắn" |
| Output | CEFR per-skill + overall + CI + confidence + disclaimer | |

## 2. Phân bổ theo kỹ năng (thời lượng + số câu)
| Kỹ năng | Loại chấm | Số item | Thời lượng | Ghi chú |
|---|---|---|---|---|
| Reading | khách quan → θ | **12** | ~15 phút | 3 bài đọc · 5 competency · tag Grammar/Vocab |
| Listening | khách quan → θ | **12** | ~15 phút | 3 audio · 5 competency · tag Grammar/Vocab |
| Writing | rubric AI → band | **1 task** | ~9 phút | task theo level (email/opinion ngắn ~120–150 từ) |
| Speaking | rubric AI → band | **1 task** | ~6 phút | nói ~2 phút (giới thiệu + 1 chủ đề) |
| **Tổng** | | 24 obj + 2 task | **~45 phút** | |

**Vì sao 12 (không phải 8):** độ tin của θ ∝ tổng thông tin các câu, mà câu chỉ cho nhiều thông tin khi `irt.b` gần năng lực người làm. (1) Fixed form không dồn câu quanh sức người làm → phải rải nhiều `irt.b` phủ A2→C1 → cần nhiều câu hơn adaptive. (2) 8 câu/5 mốc → CI rộng ~2 mốc; 12 câu → 2–3 câu/vùng → CI về ~1 mốc. (3) 5 competency cần đủ mẫu để báo cáo per-competency có nghĩa. Đánh đổi: +10 phút.

## 3. Ma trận Question Distribution — trục [4] (Skill × CEFR × Competency → count)

**Cấu trúc bài (đã có sẵn trong code):** mỗi item R/L gắn vào 1 **Stimulus** (`AssessmentStimulus`, `kind: passage|audio`) qua `stimulusId`. Tức Reading = **các bài đọc**, mỗi bài kèm cụm câu hỏi; Listening = **các đoạn audio** (nghe 1 lần, transcript ẩn tới khi nộp). Mỗi Stimulus có `difficultyTier` (easy/mid/hard) + `cefrBandHint`.

**Competency (5) — câu đó đo gì + mức ưu tiên MVP** (subskill enum có sẵn):
| Competency | Đo gì | MVP |
|---|---|---|
| `gist` (main idea) | nắm ý tổng quát | Bắt buộc |
| `detail` | lấy thông tin trực tiếp | Bắt buộc |
| `inference` | suy luận điều không nói thẳng — phân biệt level cao | Bắt buộc |
| `attitude` / purpose | hiểu quan điểm / ý định người nói | Nên có |
| `vocab_in_context` | đoán nghĩa từ theo ngữ cảnh — phân loại tốt hơn B1–C1 | Thêm ✓ |

Để **phase sau** (tốn câu, lợi ích chủ yếu B2–C1): `reference/cohesion`, `organization/discourse`.
→ Mỗi câu = MCQ khách quan + gắn **`irt.b`** (độ khó toàn cục) → đúng/sai → feed θ có trọng số theo `irt.b`.

### Reading — 3 bài đọc, 12 câu (kèm `irt.b`)
| Bài (Stimulus) | Tier | CEFR | Competency | irt.b |
|---|---|---|---|---|
| **Passage 1** (đời thường) | easy | A2 | `gist` | −1.3 |
| | | A2 | `detail` | −1.0 |
| | | B1 | `detail` | −0.3 |
| | | B1 | `vocab_in_context` | −0.2 |
| **Passage 2** (học thuật nhẹ) | mid | B1 | `inference` | 0.0 |
| | | B2 | `detail` | 0.3 |
| | | B2 | `vocab_in_context` | 0.5 |
| | | B2 | `inference` | 0.6 |
| **Passage 3** (quan điểm) | hard | B2 | `attitude` | 0.7 |
| | | C1 | `gist` | 1.0 |
| | | C1 | `inference` | 1.2 |
| | | C1 | `attitude` | 1.3 |

### Listening — 3 đoạn audio, 12 câu (kèm `irt.b`)
| Đoạn (Stimulus) | Tier | CEFR | Competency | irt.b |
|---|---|---|---|---|
| **Audio 1** (hội thoại) | easy | A2 | `gist` | −1.2 |
| | | A2 | `detail` | −1.0 |
| | | B1 | `gist` | −0.3 |
| | | B1 | `detail` | −0.2 |
| **Audio 2** (độc thoại) | mid | B1 | `inference` | 0.1 |
| | | B2 | `detail` | 0.3 |
| | | B2 | `vocab_in_context` | 0.5 |
| | | B2 | `attitude` | 0.6 |
| **Audio 3** (thảo luận) | hard | B2 | `inference` | 0.7 |
| | | C1 | `gist` | 1.1 |
| | | C1 | `attitude` | 1.2 |
| | | C1 | `inference` | 1.3 |

### Resources đo qua tag (nhúng vào R/L, KHÔNG cộng thêm câu)
Một số item R/L trên gắn thêm tag để lấy tín hiệu Grammar/Vocab:
- `grammar_tense` ≥3 item (rải A2/B1/B2) · `word_form` ≥2 · `collocation` ≥2.
→ Báo cáo Grammar/Vocab = thống kê đúng/sai trên các tag này, không phải section riêng.

### Writing (1) / Speaking (1)
- Task **chọn theo level ước tính** (`WRITING_TASKS`/`SPEAKING_PARTS` đã có: A2→task1/part1 … C1→task2/part3).
- Chấm theo rubric (dimensions): Writing = task response · coherence · grammar range · lexical range. Speaking = fluency · pronunciation · lexical · grammar.

## 4. Difficulty Distribution by CEFR — trục [5] (chỉ áp cho R/L khách quan)
| CEFR | Easy | Medium | Hard |
|---|---|---|---|
| A2 | 60 | 40 | 0 |
| B1 | 30 | 50 | 20 |
| B2 | 10 | 50 | 40 |
| C1 | 0 | 40 | 60 |

- % trên tổng câu của level đó; Engine làm tròn (largest-remainder) rồi rải vào competency-cell (xem file logic).
- `difficulty = f(irt.b)` **toàn cục** (`relative_to_cefr:false`). Band cắt theo `irt.b`: **Easy `b<−0.5` · Medium `−0.5…0.5` · Hard `b>0.5`** (khớp `TIER_B_BAND`). → cột `irt.b` ở §3 chính là hiện thực của phân bổ này.
- **Target, không hard-constraint**: thiếu item → fallback 4 bước (đúng target → top-up cùng CEFR → nới band → gần nhất theo irt.b), **có log**.

## 5. Scoring / CEFR mapping — trục [6] (tái dùng, KHÔNG viết mới)
```
R/L:  đúng/sai item → θ (IRT) → thetaToBand(θ) → bandToCefr → CEFR/skill
W/S:  rubric AI → band → bandToCefr → CEFR/skill
Overall: MIN(CEFR các kỹ năng) → CEFR overall + CI 95%   ← chốt: min, KHÔNG phải 25% avg
```
> **Overall = min (chốt):** trình độ tổng = mức **thấp nhất** trong 4 kỹ năng. Bảo thủ — không thổi phồng, xếp lớp an toàn.
> ⚠ Đánh đổi: min rất **nhạy với kỹ năng yếu/đo kém nhất**. Kết hợp với [V-ws] (chấm AI W/S chưa chắc) → 1 điểm W/S sai có thể kéo tụt cả kết quả. Cần chặn bằng confidence/flag ở kỹ năng variance cao.
> **Lệch code:** `computeOverallBand` hiện tính **trung bình 25%/kỹ năng** → khi wire phải **override sang min** (hàm mới hoặc tham số).
- Ngưỡng θ→band hiện là **linear anchor provisional** (chưa calibrate) → output **gắn nhãn "ước tính" + disclaimer**, KHÔNG gọi "CEFR chính thức".
- Sau pilot: calibrate `irt.b` (`export-calibration-data.js` + Rasch, n≥100/item) → Angoff panel chốt θ-cut → thay bảng → mới bỏ nhãn ước tính.

## 6. Yêu cầu thêm đã gộp vào demo (uy tín)
1. **Confidence/CI** hiển thị (`ciLoBand/Hi`, `confidenceCap`) — trung thực về độ chắc.
2. **Disclaimer "ước tính"** tới khi calibrate (`MAPPED_SCORE_DISCLAIMER`).
3. **Chống làm ẩu / bấm bừa** (`response-confidence.js` + `scoring-invalid-low-effort`) — chặn CEFR rác.
4. **Profile per-skill** (R=B1, L=A2, W=…, S=…) — học viên thấy đúng trình độ nhờ chi tiết, không chỉ 1 nhãn.
5. **Prior từ intake** (`starting-point.engine`) — khai điểm cũ → neo prior → ít câu hơn, vẫn tin.
6. **Ghi kết quả vào hồ sơ năng lực** (chốt) — placement KHÔNG tự chọn khóa học. Nó **ghi CEFR (per-skill + overall) vào hồ sơ năng lực** người dùng (learner/competency profile). **Module lộ trình** đọc hồ sơ → tự lấy khóa học theo level → xây lộ trình học. Tách concern: khung đề = *đo & ghi*; lộ trình = *tiêu thụ*.

## 7. Output mẫu (học viên thấy)
```
Trình độ ước tính: B1  (= MIN 4 kỹ năng; khoảng tin cậy B1–B2, độ tin: trung bình)
  Reading    B2
  Listening  B1  ◂ min
  Writing    B1
  Speaking   B1
⚠ Điểm ước tính nội bộ — không phải chứng chỉ chính thức.
→ Ghi vào hồ sơ năng lực → lộ trình học tự dựng từ hồ sơ này.
```

## 8. Success criteria (định nghĩa "review được / uy tín")
- Ráp được form đủ 24 obj + 2 task từ bank; mỗi ô có item ở đúng dải `irt.b` (không thiếu ô nghiêm trọng).
- Mỗi kỹ năng ra CEFR + CI; overall + confidence.
- Học viên bấm bừa bị flag, không ra CEFR "đẹp giả".
- Thời lượng thực ≤ ~50 phút.
- Output kèm disclaimer + routing.

## 9. Rủi ro / vướng mắc
- **[V-data]** `irt.b` provisional + bank có thể mỏng ở vài (skill×CEFR) → fixed-form ráp hụt. **Phải chạy `bank-coverage` trước khi seed.** (chính là [R1])
- **[V-ws]** Chấm rubric AI cho W/S **độ tin tự nó chưa chắc** (có `aiScoreVariance`) → W/S nên hiển thị confidence thấp hơn R/L, hoặc gắn cờ "cần người review" khi variance cao.
- **[V-agg]** Overall = **min** (chốt) nhưng code mặc định 25% avg → **cần override khi wire**. Min nhạy với kỹ năng đo kém → phải kèm confidence/flag.

## Đã chốt
- Overall = **min** các kỹ năng.
- W/S = **1 task**/kỹ năng.
- **Không** làm biến thể "bỏ W/S" — luôn đủ 4 kỹ năng.
- Kết quả **ghi vào hồ sơ năng lực**; lộ trình học tiêu thụ hồ sơ để lấy khóa học (khung đề không tự map khóa học).

## Câu hỏi mở (còn lại)
- Không còn — spec demo đã đóng để review.
