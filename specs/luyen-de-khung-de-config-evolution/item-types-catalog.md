<!-- Catalog dạng câu (item type) × kỹ năng × competency cho bài kiểm tra đầu vào CEFR (R/L + UoE).
     Bản markdown của artifact 77f642a1. Nguồn engine: assessment-question.model.js (ITEM_TYPES), grade.js. -->
# Item Types Catalog — CEFR Placement (R/L + Use of English)

> Cập nhật 2026-08-30: **2 chiều điểm** — Reading & Use of English (θ_R) + Listening (θ_L). Không còn θ_UoE riêng.
> Grammar/word_form/collocation nhúng trong **Reading** (skill=reading, dạng đo-thuần) → tính vào θ_R. `vocab_in_context` ở cả R lẫn L. Grammar KHÔNG vào Listening.
> Ký hiệu engine: **✓** có sẵn (grade.js chấm tự động) · **＋** cần thêm type + luật chấm.

## Reading → θ_R (cần ngữ cảnh bài đọc)
| Dạng | code | Đo competency | Engine |
|---|---|---|---|
| Trắc nghiệm 4 đáp án | `mcq` | gist · detail · inference · attitude | ✓ |
| Đúng / Sai / Không nêu | `true_false_ng` | detail · inference | ✓ |
| Ghép tiêu đề ↔ đoạn | `matching_headings` | gist (ý chính đoạn) | ✓ |
| Ghép thông tin / đặc điểm | `matching_information` · `matching_features` | detail (quét) | ✓ |
| Nối vế kết câu | `matching_sentence_endings` | inference · liên kết ý | ✓ |
| Điền hoàn thành câu | `sentence_completion` | detail (rút từ bài) | ✓ |
| Từ trong ngữ cảnh | `mcq` | **vocab_in_context** | ✓ |

VD vocab_in_context: *Từ "vital" (dòng 4) gần nghĩa nhất với: A. optional / **B. essential** / C. costly / D. temporary.*

## Listening → θ_L (nghe 1 lần, transcript ẩn tới khi nộp)
| Dạng | code | Đo competency | Engine |
|---|---|---|---|
| Trắc nghiệm 4 đáp án | `mcq` | gist · detail · inference · attitude | ✓ |
| Điền ghi chú / hoàn thành câu | `sentence_completion` | detail (bắt thông tin) | ✓ |
| Điền nhãn sơ đồ / bản đồ | `labeling` | detail (định vị) | ✓ |
| Nghe chép chính tả | `dictation` | detail · âm/chính tả | ✓ |
| Ghép người nói ↔ quan điểm | `matching_features` | attitude · purpose | ✓ |

## Use of English (nhúng trong Reading → θ_R; đo thuần)
| Dạng | code | Đo competency | Engine |
|---|---|---|---|
| Điền chỗ trống mở | `cloze` | grammar_tense · giới từ · mạo từ | ✓ |
| Điền từ cho sẵn (ngân hàng) | `fill_blank` + options | collocation · vocab | ✓ |
| Biến đổi dạng từ | `fill_blank` / `short_answer` | word_form | ✓ |
| Chọn lỗi sai | `mcq` | grammar (nhận diện lỗi) | ✓ |
| Sắp xếp từ thành câu | `sentence_ordering` | trật tự từ · cú pháp | **＋** |
| Biến đổi câu giữ nghĩa (key word) | `key_word_transform` | grammar tổng hợp | **＋** |

VD: cloze *"By the time we arrived, the film ___ already started." → **had***. word_form *"…very ___ (IMPRESS)" → **impressive***. ordering *"never / such / seen / I / have / a view" → **I have never seen such a view***.

## Bộ MVP (chốt lean — 0 dạng cần code mới)
- **Reading:** `mcq` · `true_false_ng` · 1× matching · vocab_in_context.
- **Listening:** `mcq` · `sentence_completion` · 1× `labeling` · `matching_features`.
- **Use of English:** `cloze` · `fill_blank` (banked) · `word_form`.
- **Hoãn phase 2:** `sentence_ordering`, `key_word_transform` (cần code type + grade; dạng có-sẵn đã phủ đủ competency).

## Nguyên tắc tag competency (tránh mis-tag)
- **inference ≠ detail:** inference chỉ đúng khi đáp án phải **suy từ dữ kiện gián tiếp**. Nếu lý do/thông tin được **nêu thẳng** trong bài (kể cả diễn giải lại) → đó là **detail/paraphrase**, KHÔNG phải inference.
- **attitude** = quan điểm/thái độ người nói, không phải sự kiện.
- (Nếu auto-gen: đưa 2 luật trên vào prompt.)

## Nguyên tắc dùng
- Mỗi câu gắn `irt.b` (feed θ) + `subskill` (competency) + `stimulusId` (R/L) / rời (UoE).
- Điều tiết level bằng assembly, không thêm dạng: A1/A2 → dạng dễ; matching/headings → B1+.
- UoE **không** dùng dạng "hỏi ngữ pháp về đoạn văn" (phụ thuộc bài → nhiễm θ_R).
