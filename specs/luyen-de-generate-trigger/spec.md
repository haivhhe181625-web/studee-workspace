---
feature: luyen-de-generate-trigger
status: draft
created: 2026-08-18
repos: [exe-api, exe-admin]
epic: PRD-151 (M7 Luyện đề)
relates: luyen-de-item-bank-generation (Bước 2), llm-direct-testlet-generation (③)
---

# Spec — Admin trigger sinh câu bù (BullMQ, chạy trên server)

## 1. Mục tiêu & lý do

Sinh câu (đặc biệt listening có audio) phải chạy **trên đúng server đích** để file audio lưu vào disk
server đó (khớp URL tương đối trong DB). Không có SSH → cần **kích hoạt qua admin UI**: admin bấm nút →
**api server đó** chạy generation → audio vào disk của chính nó. Generation dài (LLM) không được block HTTP
→ đẩy sang **BullMQ background job** (mirror `flashcard-generate`), UI poll trạng thái.

## 2. Phạm vi

- **BE (exe-api):**
  - Queue `exam-practice-generate` + worker (mirror `flashcard-generate.queue/worker`), register trong
    `server.js`. Job `exam-practice-generate:fill` → `{ surplus?, maxTotalCalls?, skill?, exam? }`.
  - `gen-fill.service`: thêm `onProgress` callback vào runGenerationBatch/fillBank (báo sau mỗi cell) + hỗ
    trợ filter `skill`/`exam` (chỉ sinh listening / TOEIC nếu chọn).
  - Service `enqueueFill(data)` (jobId cố định `exam-practice-fill` → 1 job tại 1 thời điểm; đang chạy →
    reuse) + `readFillJob(jobId)` (state + progress + result).
  - Admin endpoints (`test-paper.admin.js`): `POST /test-papers/generate-fill` (perm write) → 202
    `{jobId,status}`; `GET /test-papers/generate-fill/:jobId` (perm read) → `{state,progress,result}`.
- **FE (exe-admin):** trên coverage panel — nút "Sinh câu bù" → POST → poll GET → hiện progress (cell x/y,
  tokens, cost) + kết quả. Chọn filter skill/exam.

## 3. Ngoài phạm vi

- **Promotion dev→prod** (copy DB + audio) — feature ops riêng, không thuộc phase này.
- **Auto-vet / auto-active** — câu vẫn ra draft, vet thủ công.
- **Multi-worker/scale** — concurrency 1 (một lần fill; generation nặng + tránh rate-limit LLM).
- **Sinh trên local** — vẫn chạy được qua ops script; feature này là đường không-SSH cho server.

## 4. Acceptance Criteria

- [ ] AC1 — `POST /generate-fill` → 202 `{jobId, status}`; job vào queue; gọi 2 lần liên tiếp (đang chạy)
  → **reuse cùng job** (không đẻ job trùng).
- [ ] AC2 — Worker chạy → gọi `fillBank(data)`; job `completed` với `returnvalue` = report + usage + costUsd.
- [ ] AC3 — `GET /generate-fill/:jobId` trả `state` (waiting/active/completed/failed) + `progress`
  (cell hiện tại / tổng, đã sinh) + `result` khi xong.
- [ ] AC4 — Filter: `POST {exam:'toeic', skill:'listening'}` → chỉ sinh cell TOEIC listening.
- [ ] AC5 — Audio (listening) lưu vào disk server đang chạy worker (đúng nơi api serve) — verify qua
  `saveAssessmentAudio` path (không đổi).
- [ ] AC6 — Phân quyền: POST cần `test-paper:write`, GET cần `test-paper:read`; thiếu → 403.
- [ ] AC7 — Worker fail → job `failed`, GET trả state failed + code; không treo UI.
- [ ] AC8 — Test (mock queue + mock generate): enqueue dedup, worker chạy fillBank, status mapping. Full
  suite xanh, 0 regression.

## 5. Repos ảnh hưởng

- **exe-api** — queue + worker + gen-fill (onProgress/filter) + admin endpoints + server.js register.
- **exe-admin** — nút trigger + poll trên coverage panel.

## 6. Liên kết

- Pattern tái dùng: `src/queues/flashcard-generate.{queue,worker}.js`, `flashcard-generation.service`
  (enqueue/status), `server.js` (worker register).
- Core sinh: `gen-fill.service` (Bước 2) + `llm-generate-testlet` (③).

## Câu hỏi mở

- Progress granularity: sau mỗi cell là đủ, hay cần sau mỗi testlet? (đề xuất mỗi cell — KISS).
- Cancel job giữa chừng: có cần nút huỷ không? (đề xuất P2, chưa làm).
