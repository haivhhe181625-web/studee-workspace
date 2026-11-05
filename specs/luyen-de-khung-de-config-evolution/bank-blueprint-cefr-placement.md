<!-- Khung câu hỏi (authoring template) + yêu cầu coverage của item bank cho đề đầu vào CEFR.
     Đi kèm demo-cefr-placement-entry-test-spec.md (khung đề) + artifact review.
     "Xây lại khung bank" = nâng granularity coverage lên skill×CEFR×competency×irt.b tier. -->
# Bank Blueprint — CEFR Placement (khung câu hỏi + yêu cầu bank)

> Bổ trợ `demo-cefr-placement-entry-test-spec.md`. File kia = **khung đề** (1 bài thi gồm gì). File này = **khung bank** (kho item phải chứa gì để ráp được đề đó, lặp lại nhiều lần).

## 0. Vì sao phải "xây lại khung bank"
Công cụ coverage hiện có (`bank-coverage.js` + `POOL_MATRIX`) chỉ đếm **skill × CEFR × itemType**. Artifact yêu cầu mịn hơn: **skill × CEFR × competency (subskill) × dải `irt.b`**. → bank + tooling phải nâng độ mịn. Nếu không, "bank đủ" theo tool cũ vẫn có thể **thiếu đúng ô** đề cần (vd có nhiều câu B2 nhưng không câu nào là `attitude`, hoặc tất cả dồn 1 độ khó).

## 1. Khung 1 item — authoring template (mỗi câu R/L phải có)
| Field | Bắt buộc | Giá trị |
|---|---|---|
| `skill` | ✅ | reading / listening |
| `stimulusId` | ✅ | trỏ Passage/Audio (`AssessmentStimulus`) — item không đứng rời |
| `cefrLevel` | ✅ | A2 / B1 / B2 / C1 (mốc đề đo) |
| `subskill` | ✅ | **competency**: gist / detail / inference / attitude / **vocab_in_context** |
| `itemType` | ✅ | mcq (MVP) hoặc true_false_ng / matching… |
| `irt.b` | ✅ | **KHÔNG null** — nếu chưa calibrate thì gán provisional theo CEFR (A2 −1.2 · B1 −0.4 · B2 0.4 · C1 1.2), rơi đúng tier |
| `answerKey` (+`options`) | ✅ | `select:false` |
| `targetGoal` | ✅ | trong scope placement (general) |
| `contentSource` | ✅ | ≠ null (self-authored / licensed / ai-generated-reviewed) — null bị chặn khỏi pool |
| `status` | ✅ | active mới được ráp |

**W/S** khác: `skill` writing/speaking · `itemType` essay / speaking_prompt · `rubricId` ✅ · `cefrLevel` = band task · **không** `irt.b` (chấm rubric, ngoài θ).

## 2. Khung 1 đề — assembly target (24 slot R/L)
Recap từ khung đề (mỗi slot = 1 ô cần rút 1 item khớp): Reading 12 (3 passage) · Listening 12 (3 audio), mỗi câu có (CEFR, competency, tier `irt.b`). Chi tiết bảng: xem `demo-cefr-placement-entry-test-spec.md` §3.

## 3. Khung bank — coverage target (CÁI CẦN XÂY LẠI)
Đề chỉ rút **1** item/ô, nhưng bank phải chứa **nhiều** để (a) ráp được nhiều đề khác nhau, (b) tránh lộ đề / overexposure, (c) có chỗ thay khi item bị retire.

**Mục tiêu MVP: mỗi ô `(skill × CEFR × competency)` ≥ 5 item active**, và các item trong ô **rải đúng dải `irt.b` của tier** (không dồn 1 điểm).

### Reading — 12 ô × ≥5 = **pool ≥ 60**
| CEFR | competency | tier (irt.b) | pool ≥ |
|---|---|---|---|
| A2 | gist | easy (b<−0.5) | 5 |
| A2 | detail | easy | 5 |
| B1 | detail | mid (−0.5…0.5) | 5 |
| B1 | vocab_in_context | mid | 5 |
| B1 | inference | mid | 5 |
| B2 | detail | mid–hard | 5 |
| B2 | vocab_in_context | mid–hard | 5 |
| B2 | inference | hard (b>0.5) | 5 |
| B2 | attitude | hard | 5 |
| C1 | gist | hard | 5 |
| C1 | inference | hard | 5 |
| C1 | attitude | hard | 5 |

### Listening — 12 ô × ≥5 = **pool ≥ 60** (cấu trúc gương Reading; competency: gist/detail/inference/attitude/vocab)

### Stimulus (passage/audio)
- Mỗi tier (easy/mid/hard) mỗi skill cần **≥ vài passage/audio** để 4 câu/bài không phải lúc nào cũng cùng 1 bài → tránh lộ.
- Reading passage có `body`; Listening audio `maxPlays:1`, `transcript` ẩn tới khi nộp.

### W/S
- Mỗi `(skill × band task dùng)` ≥ **3 task** (để luân phiên). Writing task1/task2, Speaking part1/2/3 theo level.

## 4. Gaps phải xử TRƯỚC khi seed (chặn)
- **[G1] Enum thiếu competency.** `vocab_in_context` **chưa có** trong `SUBSKILLS` (`assessment-question.model.js`). Phải thêm. (attitude/gist/detail/inference đã có.)
- **[G2] `irt.b` bắt buộc.** Item placement phải có `irt.b` (provisional theo CEFR nếu chưa calibrate) — nếu null thì lọc theo tier vô nghĩa.
- **[G3] Coverage tooling mù competency + irt.b.** `POOL_MATRIX` (skill×CEFR) + `coverageGaps`/`bank-stats` phải nâng thành **skill×CEFR×competency×tier** thì mới kiểm được khung mới. Hiện `getBankStats` KHÔNG group theo subskill/irt.b.
- **[G4] Stimulus theo tier.** Đủ passage/audio mỗi tier, `cefrBandHint` khớp item.

## 5. Kiểm bank — cách + giới hạn
**Lệnh (chạy ở máy/WSL có DB):**
```
docker exec exe_api node scripts/ops/bank-status.js       # tổng quan skill + UoE/level + calibrate
# hoặc getBankStats() qua admin insights (coverage gaps vs POOL_MATRIX)
```
**Giới hạn hiện tại:** cả 2 chỉ báo **skill totals / UoE-by-level / skill×CEFR** — **KHÔNG** theo competency hay `irt.b` tier. → **không verify được khung placement mới** cho tới khi làm [G3].
**Ở bản Windows này:** không container nào chạy → DB không với tới → phải kiểm ở WSL/máy có DB.

## Đã chốt trong khung này
- Item authoring template §1 · pool depth MVP = **≥5/ô** · irt.b provisional theo CEFR để ráp được ngay.

## Câu hỏi mở
- **[bank-Q1]** Pool depth 5/ô đủ cho MVP hay muốn cao hơn (vd 8) để chống lộ đề tốt hơn?
- **[bank-Q2]** Có muốn tôi thực hiện luôn [G1] (thêm enum) + [G3] (nâng coverage tooling) không, hay chỉ giữ ở mức khung để bàn tiếp?
