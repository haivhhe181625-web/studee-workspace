---
feature: luyen-de-generate-trigger
status: done
created: 2026-08-18
repos: [exe-api, exe-admin]
---

# Tasks — Admin trigger sinh câu bù (BullMQ)

## BE (exe-api) — DONE
- [x] `gen-fill.service`: `loadCoverage`/`fillBank` nhận filter `skill`/`exam`; `runGenerationBatch` nhận
  `onProgress` (báo sau mỗi cell); thêm `enqueueFill`/`readFillJob`/`statusOf` (lazy-require queue → test
  không cần Redis).
- [x] Queue `exam-practice-generate.queue.js` (jobId cố định `exam-practice-fill` → dedup 1 job/lần).
- [x] Worker `exam-practice-generate.worker.js` (concurrency 1, `job.updateProgress`), register `server.js`.
- [x] Admin: `POST /test-papers/generate-fill` (write, 202) + `GET /test-papers/generate-fill/:jobId` (read).
- [x] Test `test-paper-admin-generate-fill` (mock queue): enqueue 202 + dedup, status+progress, 404, 403 → 4/4.

**Verify:** `npx jest test-paper` → 99/99 xanh.

## FE (exe-admin) — DONE
- [x] Service `triggerFill`/`getFillStatus` + type `FillJob`; hook `useTriggerFill`/`useFillStatus` (poll 2s,
  dừng khi done/failed).
- [x] `coverage-panel`: nút "Sinh câu bù" + select exam/skill + hiện progress (cell x/y, +câu) + cost khi xong.

**Verify:** `npm run build && lint` sạch.

## Deploy (ops — CHƯA làm)
- [ ] Deploy code ③ + trigger lên **server dev** (git push → dev pull/rebuild).
- [ ] Config llm dev: OPENAI key + service-key khớp.
- [ ] Test: admin dev bấm "Sinh câu bù" (skill=listening) → audio vào disk dev → nghe được trên dev.

## Còn lại
- [ ] Promotion dev→prod (feature ops riêng).
- [ ] (P2) Nút cancel job.
