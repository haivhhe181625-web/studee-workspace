# M2 — Luyện phát âm IPA (refactor) · Spec pointer

> Spec đầy đủ ở Plane. Đây chỉ là con trỏ + acceptance rút gọn cho vòng code.

## Nguồn chuẩn (đọc trước khi code)
- **Plane spec v1.2 (gốc):** `SRS — M2: Luyện phát âm IPA` (page 23306346-…)
- **Plane v1.3 (cắt gọn theo data — ĐỌC CÁI NÀY cho phần đã đổi):** sub-page b88a1ebe-…
- **Report cơ sở:** `plans/reports/from-brainstorm-to-spec-revision-m2-ipa-data-driven-scope-cut-report.md`

## Scope (v1.3)
35 bài **published A/B/C/D**. **Defer** S6/S7 (E/F, 10 bài draft). L23 /v/ giữ draft.

## Worktree
- BE: `.worktrees/ipa-m2-exe-api` (branch `feat/ipa-m2-pronunciation-refactor`, off develop)
- FE: `.worktrees/ipa-m2-exe-web` (branch `feat/ipa-m2-pronunciation-refactor`, off develop)

## Acceptance (rút gọn — đối chiếu AC-001…014 trên Plane)
- [ ] `buildSteps()` ra đúng 3 màn, mọi template (AC-001)
- [ ] Luyện nói: chấm 1 mục không mất điểm mục khác, không rời màn (AC-002)
- [ ] Cảm âm dệt = minimal-pair 4 câu, **qua khi ≥3/4**, **có tính Mastery**; nhóm D (L31-36) không có cổng (D4/D5)
- [ ] Mastery: tổng ≥80% **và** câu ≥70 **và** qua cổng cảm âm (trừ D); ngưỡng từ `app-setting` (AC-005/006/011)
- [ ] SRS 2 nhịp: learning → ôn 1 ngày → ôn 7 ngày → mastered; trượt gắn `needsReview`, không thu hồi (D6)
- [ ] FE gọi ≤2 endpoint (`/start`, `/attempts`) (FR-006)
- [ ] Publish gate theo audio word/sentence/pair; **không** yêu cầu model 3-tốc-độ/L1 (D2)
- [ ] Migration 35 bài A-D: streak-mastered cũ → `mastered`; đang dở → về đầu bài mới, không kẹt (AC-009/010)
- [ ] Regression 50 audio lệch ≤ ±5 (AC-014)

## Ngoài scope
S6/S7 stress/intonation; audio model 3 tốc độ + `l1CompareAudio`; hợp nhất SRS với review layer (PRD-142).
