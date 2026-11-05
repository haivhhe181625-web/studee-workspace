---
feature: luyen-de-paper-import
status: draft
created: 2026-08-17
role: tech-lead
---

# Design — Import đề luyện thi từ canonical JSON

> Schema đầy đủ + mapping + bảng answerKey: `data-model.md`. File này: cách tiếp cận, quyết định, validation, luồng, rủi ro.

## 1. Cách tiếp cận

```
File nguồn (JSON | sau: DOCX/PDF/XLSX)
   │  ── adapter ──►  CANONICAL JSON  ── import.service ──►  DB (draft)  ── vet ──►  publish
```

- **Canonical = 1 hợp đồng duy nhất.** P1 chỉ có adapter JSON (identity — file đã là canonical). Adapter khác (DOCX/PDF) convert-về-canonical ở phase sau → import core KHÔNG đổi.
- **Import ≠ assemble.** `assemble` tự bốc item `active` từ bank theo blueprint. Import **biết chính xác** câu nào → set `frozenItemIds` **trực tiếp** theo thứ tự file. KHÔNG gọi assemble, KHÔNG lọc status.
- **Vet gate ở tầng PAPER.** Import ra `TestPaper.status='draft'` (+ item/stimulus `draft`). `listPublished` chỉ hiện `published` → học viên không thấy đề chưa duyệt. Examiner preview (M7-15) → `publish`.
- **5-file module.** Đặt ở `modules/test-paper/` (đề đã có module này): `import.service.js` + validators. CLI ở `scripts/ops/`.

## 2. Quyết định đã chốt (owner, 2026-08-17)

| # | Quyết định |
|---|---|
| 1 | Vet gate: GIỮ. Import→`draft`, publish thủ công. Không autoPublish. |
| 2 | Free-form: `blueprintKey` optional, KHÔNG validate cấu trúc/số câu. |
| 3 | Center: GÁC. P1 chỉ platform (`centerId=null`). |
| 4 | Audio cả 2 (+ref): stimulus = `ref` \| inline `audioUrl` \| inline `audioFile`(copy storage). |
| 5 | Provenance: THÊM subdoc vào Question + Stimulus. |
| 6 | Idempotency: BỎ. |

## 3. Validation (fail-fast, TRƯỚC khi ghi DB)

Thứ tự kiểm — lỗi đầu tiên → throw `ApiError`, KHÔNG ghi doc nào:
1. `schemaVersion` hỗ trợ (`"1.0"`).
2. `paper.exam`/`skill` ∈ enum; `sections` không rỗng.
3. Mỗi question: `itemType` ∈ `ITEM_TYPES`; `answerKey` **đúng shape theo itemType** (bảng ở `data-model.md` §answerKey — khớp `grade.js`).
4. `contentSource` resolve từ `provenance.license` (self-authored/licensed/ai-generated-reviewed). Thiếu/unknown → reject.
5. Stimulus: nếu `ref` → stimulus tồn tại; nếu inline audio → có `audioUrl` hoặc `audioFile` (file tồn tại).

> Validate là **pure** (`import-validate.js`, không DB) để unit-test không cần Mongo — trừ bước 5 `ref` (cần DB, kiểm trong service).

## 4. Luồng importPaper(canonical)

1. `validateCanonical(canonical)` (pure) → ném lỗi nếu sai.
2. Với mỗi section:
   - stimulus `ref` → dùng `_id` đó (không tạo). inline → `AssessmentStimulus.create({... status:'draft', centerId:null, contentSource, provenance})`; `audioFile` → copy vào `STORAGE_LOCAL_ROOT` + set `media.audioUrl`.
   - questions → `AssessmentQuestion.create({... cefrLevel:section.cefr, itemType, stem, options, answerKey, acceptedVariants, explanation, stimulusId, order, skill, targetGoal:exam, status:'draft', centerId:null, contentSource, provenance})`.
3. `TestPaper.create({... status:'draft', frozenItemIds: <ID câu theo thứ tự section→question>, centerId:null})`.
4. Trả `{ paperId, counts:{stimuli,questions}, warnings[] }` cho preview. KHÔNG publish.

> **Atomic:** Mongo standalone không có multi-doc txn. Mitigation: validate TOÀN BỘ trước (bước 1) nên phần lớn lỗi bắt trước khi ghi. Nếu lỗi giữa chừng khi ghi (hiếm) → để lại doc draft mồ côi; chấp nhận P1 (draft, examiner/admin dọn được; không ảnh hưởng học viên vì chưa publish). Ghi rõ trong AC3 là "reject trước khi ghi" cho lỗi validate — lỗi validate không ghi gì.

## 5. Leak-control

- `answerKey`/`acceptedVariants` vẫn `select:false` (model có sẵn) — import chỉ set giá trị, không đổi cơ chế.
- `explanation` không select:false (controller gate theo phase — đã có). Import chỉ ghi.
- Preview trả cho admin CÓ thể gồm answerKey (admin cần duyệt) nhưng đó là surface admin-api (`test-paper:manage`), KHÔNG phải learner. P1 CLI in ra console (dev) — không phải endpoint learner.

## 6. Rủi ro & mitigation

| Rủi ro | Mitigation |
|---|---|
| answerKey sai → chấm sai | Validate shape theo itemType (§3.3); examiner vet trước publish |
| Nội dung không rõ bản quyền | contentSource reject unknown (§3.4); provenance bắt buộc |
| Ghi dở giữa chừng (no txn) | Validate-all-first; doc mồ côi chỉ ở draft, không tới học viên |
| audioFile path không tồn tại/không mount | Validate file tồn tại trước copy (§3.5); lỗi rõ ràng |
| `provenance` field mới vỡ code cũ | Additive subdoc, default null — không đụng path hiện có |

## 7. Không làm (chống scope-creep)

Admin UI, adapter DOCX/PDF, center scoping, blueprint enforcement, idempotency, auto-publish, đụng assemble/CAT/MST.
