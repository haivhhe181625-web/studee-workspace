---
feature: productive-skills-assessment
status: draft
owner: TechLead
created: 2026-09-14
depends-on: design.md
note: Đây LÀ implementation plan (TDD-checkbox). Không duy trì thêm plan superpowers song song.
---

# Tasks — Đề đánh giá Nói & Viết riêng + khoá kỹ năng chưa đánh giá

Thứ tự: **BE trước** (client cần goal/endpoint) → **Web** ∥ **Mobile**. Mỗi task 1 PR-worth, base `develop`, Conventional Commits, không AI attribution.

## A. Backend — exe-api (`services/api`)

- [x] **BE-0: Kiểm pool W/S** — Đếm item `skill∈{writing,speaking}`, `status='active'`, `targetGoal∈{general|null}`.
  - **Kết quả (dev enova_dev, 14/09/2026): writing=14 essay active, speaking=14 speaking_prompt active (targetGoal null=general). Đủ, KHÔNG cần author thêm.**
  - Lưu ý: `targetGoal` lưu là `null` (không phải chuỗi 'general') → BE-2 phải bao gồm null khi lọc general.
  - **Còn lại: kiểm prod parity** — xác nhận prod cũng có ≥1 active writing + speaking `general/null` trước khi release (mới đếm được dev).

- [x] **BE-1: Goal → testType `productive_skills`** — goalToTestType + intake goal enum + model testType enum. (commit 6a52f65)
- [x] **BE-2: Config skills + quota** — createFromIntake đặt skills=['writing','speaking'], fixed, đi `withQuota('assessment')`; test xác nhận remaining giảm + không bypass. (commit 6a52f65)
- [x] **BE-3: System-check giữ mic + consent** — productive_skills ≠ placement → đã bắt buộc mic/consent; test khoá. (commit 6a52f65)
- [x] **BE-4: Bypass cờ MST theo testType** — `mstAllowed` +productive_skills; `wsTasksForAttempt` load assessment trước → `mstAllowed`. ws-tasks cờ-tắt: productive OK, diagnostic 404. (commit dfefd12)
- [x] **BE-5: Regression** — jest assessment + integration + mst-integration → **189 passed / 21 suites**.

> Phase BE xong trên branch `feat/productive-skills-assessment` (base develop, 2 commit, **chưa push**). Prod parity W/S (BE-0) cần kiểm trước release.

## B. Web — exe-web (`src`)

- [x] **WEB-1: Entry bài Nói & Viết** — *Thiết kế lại vs plan:* W/S cần system-check mic/consent nên KHÔNG start ngầm bằng mutation; thay vào đó CTA deep-link vào intake preset `?goal=productive_skills`. IntakeWizard nhận `presetGoal` (từ page server-component đọc searchParams) → preset goal + bỏ step goal, vẫn chạy mic/consent. (commit 0654b13)
- [x] **WEB-2: Khoá row kỹ năng chưa đánh giá** — `SkillBars.tsx` render đủ core skill; skill chưa có band → hàng mờ + icon khoá + "Chưa đánh giá". (commit 0654b13)
- [x] **WEB-3: CTA "Đi đến đánh giá"** — `MissingSkillCta.tsx` → Link `ASSESSMENT_INTAKE?goal=productive_skills`; bỏ chặn `!isPlacement` ở CoreResultView/ResultContainer nên placement giờ hiện CTA. Giữ nút "Làm lại bài đánh giá" tổng. (commit 0654b13)

> Verify: `tsc --noEmit` sạch (chỉ còn lỗi IPA test pre-existing, không liên quan); vitest result+intake **74/74 passed**; eslint sạch. Trên branch `feat/productive-skills-assessment` (exe-web, base develop, chưa push).

## C. Mobile — studee-app (`lib/features/exam_mst`)

- [ ] **MOB-1: Map `skills` vào state**
  - `data/exam_mst_repository_impl.dart` + `state` (+ freezed): đọc `skills` từ start/attempt response.
  - Verify: `dart run build_runner build --delete-conflicting-outputs`; `flutter analyze` 0 lỗi; placement → skills=[R,L], productive → [W,S].

- [ ] **MOB-2: Fix placement skip W/S**
  - `bloc/exam_mst_bloc.dart` `_startWsPhase`: nếu không có W/S (theo skills) hoặc `ws-tasks` rỗng/lỗi → `add(ExamMstWsSubmitted())`. `mst_ws_phase_body.dart`: phòng thủ writing rỗng.
  - Verify: chạy placement (flavor develop) → **ra kết quả R+L**, không màn W/S trống. `flutter analyze` 0 lỗi.

- [ ] **MOB-3: Bài Nói & Viết (chỉ W/S) + entry**
  - `_mcqSkills` lọc theo skills (rỗng R/L → vào thẳng ws-tasks). `usecase/start_productive_skills_exam_usecase.dart` (goal `productive_skills`).
  - Verify: chạy bài Nói & Viết → làm Viết→Nói→nộp→ra CEFR. `flutter analyze` 0 lỗi.

- [ ] **MOB-4: Result locked row + CTA + l10n**
  - `mst_result_skill_breakdown_card.dart`: locked row + "Đi đến đánh giá" (navigate trong Bloc). `.arb` en/vi/ja.
  - Verify: sau placement → Viết/Nói mờ+khoá, bấm CTA → vào bài Nói & Viết. `flutter analyze` 0 lỗi.

## Gate
- Hoàn tất A trước khi merge B/C (client phụ thuộc endpoint/goal).
- Mỗi PR: AI Self Review (reviewer checklist) → CODEOWNER `@dotuananhtb` duyệt → merge `develop`.
- Map Acceptance: BE-1..5→AC1-5; WEB-1..3→AC6-8; MOB-1..4→AC9-12.
