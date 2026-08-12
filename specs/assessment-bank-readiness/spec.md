# Spec: M3 Fixed Mode — Bước 1: Nhãn phân loại & độ sẵn sàng ngân hàng câu hỏi

- **Ngày:** 2026-08-10
- **Tác giả:** AI Brainstorm (chốt scope với dev)
- **Repos/surfaces ảnh hưởng:** `exe-api` (services/api — module `assessment`), `exe-admin` (mở field nhập liệu cho testlet). `exe-api/services/cat` chỉ **đọc** nhãn từ Mongo — không sửa ở bước này.
- **Module liên quan:** `exe-api/services/api/src/modules/assessment/`
- **Trạng thái:** Nháp — chờ BA review

## 1. Mục tiêu

Chuẩn bị ngân hàng câu hỏi đủ **nhãn phân loại** và đủ **độ phủ (coverage)** để bộ lắp đề động (bước sau) lắp được một đề hợp lệ cho đường Core — Reading, Listening, Use of English.

## 2. Bối cảnh

Đã kiểm code (không đoán), hiện trạng:

**Đã có:**
- `assessment-question.model.js` — nhãn Question rất đầy đủ: `skill`, `subskill`, `grammarPoint`, `cefrLevel`, `itemType` (14 loại), `topic` (enum 8), `targetGoal`, `status` (có `quarantined`), `stimulusId`, `irt`, `exposureCount`.
- `assessment-stimulus.model.js` — nhãn testlet: `skill`, `difficultyTier` (easy/mid/hard), `cefrBandHint`, `kind`, `itemTypesCovered[]`, `contentSource`, `status`, `exposureCount`.
- `item-type-blueprint.js` — blueprint dạng câu: reading 5 dạng, listening 3 dạng.
- `bank-coverage.js` — toàn bộ math coverage: `summarizeBank`, `coverageGaps`, `tierGaps` (mặc định ≥8/tier), `recommendGeneration`, `UOE_TARGET` (A1 60·A2 60·B1 60·B2 52·C1 30).
- `gen-testlet.service.js` + `gen-quality.service.js` + `expert-review.service.js` — pipeline AI sinh testlet + duyệt.

**Chưa có (phần bước này giải quyết):**
- Nhãn cấp testlet thiếu `passageTheme` → chưa enforce được "không lặp 1 chủ điểm >2 lần/kỹ năng" (bộ lắp đề bước sau cần).
- Chưa đánh dấu **câu neo** (`isAnchor`) khi soạn — cần cho equating/adaptive về sau, rẻ khi gắn ngay lúc seed.
- `fixedModuleCount` mặc định `=1` (SRS: Đọc 7 · Nghe 8 · UoE 10) — chưa set số thật.
- Ngưỡng coverage tối thiểu để "đủ lắp đề" (OQ-3 trong SRS) chưa chốt; chưa biết kho hiện đủ/thiếu bao nhiêu.

## 3. Phạm vi

### Trong phạm vi
- Thêm `passageTheme` (enum, tái dùng bộ `TOPICS`) và `isAnchor` (boolean, default `false`) vào model `Stimulus`; mở 2 field này cho nhập liệu trong `exe-admin` (admin resource whitelist).
- Khảo sát kho hiện có: đếm testlet/item **active** theo `(skill × tier)` cho Reading + Listening và theo `(skill × cefrLevel)` cho UoE, dùng `bank-coverage.js`.
- Chốt ngưỡng coverage tối thiểu + set `fixedModuleCount` thật (số đã duyệt).
- Lấp phần thiếu tới ngưỡng bằng pipeline AI-gen sẵn có + expert review duyệt sang `active`.

### Ngoài phạm vi
- **Bộ lắp đề động** (snapshot `assembledForm`, ràng buộc 25/50/25, `difficultyMean`, `fallbacksUsed`) — **Lý do:** là bước 2, bước này chỉ chuẩn bị nguyên liệu.
- **Writing / Speaking**: bank + rubric (3 file Test) — **Lý do:** vướng quyết định cấu trúc rubric (thang 5-mức vs descriptor-CEFR) và tính nhất quán CEFR-gốc; tách sang bước riêng.
- Thêm `targetCefr` 6 mức trên testlet — **Lý do:** `difficultyTier` (easy/mid/hard) đã đủ cho phân bổ 25/50/25; thêm 6 mức là thừa cho fixed mode.
- Sửa logic chấm điểm/band (bước 6) và sửa `services/cat` — **Lý do:** ngoài trục "chuẩn bị ngân hàng".

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Nhân viên nội dung (center/nội bộ) | Gắn `passageTheme` + đánh dấu câu neo khi tạo testlet trong admin | Đủ nhãn để bộ lắp đề chọn đúng chỗ, tránh trùng chủ điểm |
| Vận hành nội dung | Xem báo cáo coverage & được cảnh báo cell nào thiếu | Biết cần sinh thêm testlet ở đâu |
| Hệ thống (bộ lắp đề — bước sau) | Truy vấn testlet theo skill/tier/theme/blueprint | Lắp một đề hợp lệ, đa dạng chủ điểm |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| Ngưỡng coverage tối thiểu mỗi `(skill × tier)` (R/L) — OQ-3 | ≥ 2× số testlet phục vụ/đề mỗi tier (chốt sau khi có số inventory) | Tech Lead / Product |
| `fixedModuleCount` thật | Đọc 7 · Nghe 8 · UoE 10 (theo SRS §10.1) | Product |
| Mục tiêu UoE | Dùng `UOE_TARGET` hiện có (A1 60·A2 60·B1 60·B2 52·C1 30) | Product |
| `passageTheme` enum | Tái dùng `TOPICS` (8 giá trị) hiện có | Tech Lead |

## 6. Acceptance criteria (tóm tắt)

1. Model `Stimulus` có `passageTheme` (enum, có thể null) và `isAnchor` (boolean, default `false`); do strict/strictQuery, cả hai được **khai báo trong schema** (không bị drop âm thầm).
2. Nhân viên nội dung tạo/sửa được `passageTheme` + `isAnchor` của testlet qua `exe-admin` (field nằm trong writeFields/listFields của admin resource; admin-api không chứa business logic).
3. Có **báo cáo inventory**: bảng đếm testlet/item `active` theo `(skill × tier)` cho R/L và `(skill × cefrLevel)` cho UoE, sinh từ `bank-coverage.js` — không phải số đoán.
4. Ngưỡng coverage tối thiểu được chốt & ghi lại; `fixedModuleCount` set đúng số đã duyệt (không còn `=1`).
5. Sau khi lấp thiếu: mỗi `(skill × tier)` của R/L đạt ≥ ngưỡng, UoE đạt `UOE_TARGET`; không cell nào rỗng (`tierGaps`/`coverageGaps` trả rỗng cho R/L/UoE).
6. Chỉ testlet/item `status === 'active'` và `contentSource` hợp lệ được tính vào coverage (guard `isPoolEligible` giữ nguyên).
7. Không thay đổi logic chấm điểm, bộ lắp đề, hay `services/cat`.

## 7. Câu hỏi mở

- [ ] **OQ-3:** Ngưỡng tối thiểu cụ thể mỗi `(skill × tier)` — **chặn tới khi có kết quả inventory** (Task khảo sát cần MongoDB dev chạy).
- [ ] `passageTheme` có cần giá trị nào ngoài 8 `TOPICS` hiện có không (vd `news`, `narrative`)?
- [ ] UoE lấp bằng AI-gen tới `UOE_TARGET`, hay giữ nguyên nếu kho đã đạt?

## 8. Ghi chú cho Tech Lead Design

- **Strict schema:** app chạy `strict + strictQuery` — path không khai báo bị drop khỏi cả update lẫn query filter (xem comment `assessment-question.model.js:64-67`). Bắt buộc khai báo `passageTheme`/`isAnchor` trong `StimulusSchema`, không dựa vào mixed.
- **Admin:** field mới phải vào whitelist `writeFields` của resource (`src/admin-api/resources/assessment.admin.js`); mọi side-effect (nếu có) phải delegate về service module — admin-api không chứa business logic (CLAUDE.md §Admin Rules).
- **Trục độ khó:** `difficultyTier` là trục chính; ánh xạ `CEFR_TIER` đã có trong `bank-coverage.js`.
- **UoE** là item rời (`stimulusId = null`), không thuộc testlet → coverage UoE đo ở cấp Question (`UOE_TARGET`), không dùng `tierGaps` testlet.
- **Phụ thuộc:** Task khảo sát kho cần MongoDB dev (connection string / docker compose). Nếu chưa có, các task schema/admin/config vẫn chạy được; task inventory + lấp thiếu chờ DB.
