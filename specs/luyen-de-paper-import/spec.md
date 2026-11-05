---
feature: luyen-de-paper-import
status: draft
created: 2026-08-17
owner_approved: 2026-08-17  # 6 quyết định schema đã chốt với owner
repos: [exe-api]
epic: PRD-151 (M7 Luyện đề)
---

# Spec — Import đề luyện thi từ canonical JSON (M7-16 Phase 1)

## 1. Mục tiêu & lý do

Hiện KHÔNG có đường sạch để nạp đề luyện thi thật vào hệ thống — chỉ seed script dev (synthetic) hoặc set `frozenItemIds` bằng tay qua admin-api (assemble chưa expose). Content team không tự nhập được.

Phase 1 giải quyết bằng **1 định dạng JSON chuẩn (canonical)** + **import CLI**: đưa file JSON đúng schema → tạo ngữ liệu + câu hỏi + đề (trạng thái `draft`) → examiner duyệt → publish. Đây là **core tái dùng** cho admin import UI (M7-16 phase sau) và cho việc mồi nội dung từ dataset mở (CC BY).

## 2. Phạm vi (Phase 1)

- **Canonical JSON schema** (chi tiết: `data-model.md`): 1 đề = `paper` + `sections[]` (stimulus passage/audio hoặc null) + `questions[]`.
- **BE (exe-api):**
  - Thêm subdoc `provenance {sourceRef, importBatch, attribution}` vào `AssessmentQuestion` + `AssessmentStimulus`.
  - `import.service.js` — `importPaper(canonical)`: validate → tạo Stimulus/Question (`draft`) → tạo TestPaper (`draft`) + `frozenItemIds` theo thứ tự section→question.
  - CLI `scripts/ops/import-paper.js <file.json>` → gọi service, in preview.
- **Stimulus 3 dạng:** `ref` (dùng lại stimulus có sẵn) · inline `audioUrl` (đã upload) · inline `audioFile` (mp3 mới → copy vào `STORAGE_LOCAL_ROOT`).
- **Vet gate:** import ra `draft`; publish thủ công (examiner kiểm đáp án/nội dung/leak).

## 3. Ngoài phạm vi (Phase 1)

- **Admin import UI** (exe-admin) — phase sau; CLI là core tái dùng.
- **Adapter DOCX/PDF/XLSX** — chỉ canonical JSON ở P1; adapter khác nạp sau (convert về canonical).
- **Center scoping / visibility** — P1 chỉ platform (`centerId=null`). Gác lại.
- **Validate theo blueprint** — free-form, số câu tuỳ ý (`blueprintKey` optional, không ép).
- **Idempotency/dedup** — import luôn tạo mới; admin tự xử trùng.
- **Auto-publish** — luôn ra draft, không tự publish.
- Không đụng `assemble` (import set `frozenItemIds` trực tiếp).

## 4. Acceptance Criteria

- [ ] AC1 — Import 1 file canonical hợp lệ → tạo đúng số Stimulus + Question + 1 TestPaper, tất cả `status='draft'`, `centerId=null`.
- [ ] AC2 — `frozenItemIds` của TestPaper đúng **thứ tự** section→question trong file.
- [ ] AC3 — `answerKey` sai shape theo `itemType` (vd `mcq` mà answerKey không phải số/string; `matching_headings` không phải mảng) → import **reject** với lỗi rõ ràng, KHÔNG tạo doc nào (atomic-ish: fail trước khi ghi).
- [ ] AC4 — `provenance.license` không map được sang `contentSource` hợp lệ (unknown/thiếu) → **reject** (không nạp nội dung không rõ nguồn).
- [ ] AC5 — Stimulus `ref` trỏ stimulus có sẵn → KHÔNG tạo stimulus mới, câu link đúng `stimulusId` đó.
- [ ] AC6 — Stimulus inline `audioFile` (mp3) → file được copy vào `STORAGE_LOCAL_ROOT`, `audioUrl` set đúng.
- [ ] AC7 — Import KHÔNG leak: response/preview + doc tạo ra không lộ `answerKey`/`acceptedVariants` ra client (giữ `select:false`); `provenance` lưu đúng.
- [ ] AC8 — `blueprintKey` vắng mặt → import vẫn chạy (free-form), không lỗi.
- [ ] AC9 — Full suite exe-api xanh, 0 regression engine assessment/test-paper.

## 5. Repos ảnh hưởng

- **exe-api** — toàn bộ P1 (model + service + CLI). exe-web/exe-admin: không đụng ở P1.

## 6. Liên kết

- Draft schema owner đã duyệt: `<API_REPO>/../plans/reports/canonical-schema-260817-1643-luyen-de-paper-import-report.md` (nội dung authoritative đã chuyển vào `data-model.md` này).
- Design + validation: `design.md` · Schema + mapping: `data-model.md` · Tasks: `tasks.md`.
- Epic Plane: PRD-151; sub-task import: PRD (8ded5208).
