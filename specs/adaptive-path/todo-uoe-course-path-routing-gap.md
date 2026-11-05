# TODO — UoE (grammar/vocab) không được route vào course-path từ placement

> Trạng thái: **DEFERRED** (lộ trình chưa mở). **Phải fix TRƯỚC khi mở lộ trình cho user.**
> Ghi ngày 2026-09-03. Liên quan: `specs/placement-item-bank`, `specs/adaptive-path`.

## Vấn đề (đã truy dấu, xác nhận)
Bài đầu vào **gộp Use of English vào Reading** (item grammar_tense/word_form/collocation mang `skill='reading'`), nên `Result.skillScores` **không có entry `use_of_english`**. Hệ quả:

- `adaptive.service.js:340` `skillMapFromResult` → `sm.skill` chỉ có `reading`, `listening`.
- `adaptive.service.js:948` `getProfileView` → `use_of_english` có `hasData=false` (nhãn "chưa có dữ liệu").
- `adaptive-path-builder.js:202` `weak = skills.filter(hasData && gap>=1)` → **UoE bị loại** → `neededCats` (`:52`) **không thêm** `['grammar','vocabulary']` (map ở `:45`).
- **→ Khoá học grammar/vocab KHÔNG được xếp vào course-path từ placement.**

**Không mù hoàn toàn:** `seedFromResult` vẫn tạo LearningEvents từ attempt (`:332-334`) → `computeMastery` có mastery subskill grammar/vocab → **gợi ý BÀI LẺ (recommendation) vẫn bắt được**. Mù ở tầng **khoá học**, sáng ở tầng **bài lẻ**.

## Đo phân kỳ grammar/vocab vs đọc-hiểu (2026-09-03, dev data)
N=7 (nhỏ, nhiễu — dữ liệu test): UoE acc TB 0.326 vs đọc-hiểu 0.389 (gần nhau); **std lệch cá nhân 0.309; 57% thí sinh lệch >0.25; Pearson 0.47 (không cao)**.
→ Trung bình gộp không sai band, **nhưng cá nhân phân kỳ đáng kể** → chỉ dựa band Reading sẽ bỏ sót "đọc ổn nhưng grammar yếu". **Đo lại trên pilot** (N lớn) trước khi chốt.

## Fix (rẻ, đúng — làm trước khi mở lộ trình)
**Bắc cầu mastery-subskill → course-path**: cho `buildAdaptivePath` coi `use_of_english` là skill yếu khi các subskill của nó (grammar_tense/collocation/word_form) có mastery thấp — dù không có band. Dùng dữ liệu placement đã sinh (LearningEvents), **không đo thêm gì mong manh, không kéo dài đề**.
- Điểm sửa: chỗ dựng `skills`/`weak` (`getProfileView` + `adaptive-path-builder`) — thêm nguồn "weak-by-subskill" cho các skill không có band.

## Quyết định lớn hơn (để pilot)
Có nên **tách section UoE riêng** (engine hỗ trợ: `CAT_SKILLS` gồm `use_of_english`, pool 28, timer 8') thay vì gộp?
- Chỉ đáng nếu (a) cần báo cáo UoE như competency riêng, HOẶC (b) pilot xác nhận grammar phân kỳ mạnh khỏi reading.
- Chi phí: +~8' đề, phá cấu trúc bank hiện tại (UoE đang trong passage), cần calibrate cutoff riêng.
- **Khuyến nghị: KHÔNG tách bây giờ; sửa dây là đủ. Xét lại sau pilot bằng dữ liệu phân kỳ thật.**

## Trigger
- [ ] Trước khi mở adaptive-path cho user: làm "sửa dây" ở trên.
- [ ] Sau pilot: chạy lại phép đo phân kỳ (N lớn) → quyết fold vs tách section UoE.
