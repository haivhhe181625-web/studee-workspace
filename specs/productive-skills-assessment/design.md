---
feature: productive-skills-assessment
status: draft
owner: TechLead
created: 2026-09-14
depends-on: spec.md
---

# Thiết kế — Đề đánh giá Nói & Viết riêng + khoá kỹ năng chưa đánh giá

Nguồn sự thật code: `exe-api/services/api/src/modules/assessment/`, `exe-web/src/components/features/assessment/result/`,
`studee-app/lib/features/exam_mst/`. Bản nháp trước: `plans/260914-1100-productive-skills-assessment/` (được thay bằng spec này làm canonical).

## 1. Nguyên tắc thiết kế

- **Không xây engine mới.** Tận dụng đúng 3 khối đã có: `config.skills` (scope kỹ năng), `withQuota` (giới hạn tại intake trước khi chạy AI), Result per-skill.
- Bài Nói & Viết = **một assessment mới khác goal**, không phải flow mới. Toàn bộ chấm AI dùng lại `wsTasksForAttempt` + `finalizeResult` sẵn có.
- "Chưa đánh giá" **suy ra ở client** từ `skillScores` (skill vắng / `cefrLevel==null`) — **không thêm field backend**.

## 2. Backend (exe-api) — `modules/assessment`

### 2.1 Goal/testType mới
- `goalToTestType` (assessment.service.js): `productive_skills → 'productive_skills'`.
- Intake Joi schema: thêm `productive_skills` vào enum `goal`.
- `Assessment` model: thêm `productive_skills` vào enum `testType`.
- `createFromIntake`: nhánh cho goal mới đặt `config.skills=['writing','speaking']`, `assemblyMode='fixed'`. **Không** rơi vào `if (intake.goal==='placement')` (để KHÔNG bypass quota) → tự động đi nhánh `withQuota('assessment')` (D2, D4).
- `failedSystemChecks`: goal mới **không** nằm trong nhánh loại trừ mic/consent → giữ yêu cầu mic + recordingConsent (D-spec AC3).

### 2.2 Bypass cờ MST theo testType (D3)
Hiện `mstAllowed(assessment) = env.ASSESSMENT_MST_ENABLED || testType==='placement'`.
- Mở rộng: `|| testType==='productive_skills'`.
- `wsTasksForAttempt`: đổi guard đầu hàm từ `if(!env.ASSESSMENT_MST_ENABLED) throw` → load assessment trước rồi `if(!mstAllowed(assessment)) throw` (giống `submitModuleForAttempt`/`nextModuleForAttempt` đã làm).
- `submitModuleForAttempt` đã dùng `mstAllowed` → tự bao trùm goal mới.

> Lưu ý: bài Nói & Viết **không có phase MCQ R/L** → không gọi `next-module`. Luồng: start → `ws-tasks` → upload speaking → submit → finalize. Vì vậy điểm chạm cờ chỉ là `ws-tasks` + `submit`.

### 2.3 Quota (D2, D4)
- Dùng lại `withQuota(user, 'assessment', idemKey, createAssessment)` — không thêm bucket.
- Enforce ngay: hết quota → 402 trước khi tạo attempt (chống lạm dụng chấm AI). Giá trị limit theo cấu hình quota hiện hành; tinh chỉnh khi entitlement/quota-paid xong (ngoài phạm vi).

### 2.4 Kết quả
- Không đổi `finalizeResult`. Result per-skill có sẵn writing/speaking; `validationStatus='uncalibrated'` (đã có).

### 2.5 Contract phía client
- **Start/attempt response phải trả `skills`** (mảng kỹ năng của assessment) để client biết bài gồm gì. Đây là hợp đồng đã có (server-driven skills, dùng cho web trước đó) — xác nhận `productive_skills` cũng trả đúng `['writing','speaking']`, placement trả `['reading','listening']`.

## 3. Web (exe-web) — `features/assessment/result`

### 3.1 Phát hiện kỹ năng chưa đánh giá
- Danh sách kỹ năng chuẩn (Listening/Reading/Writing/Speaking/UoE) − những skill có trong `result.skillScores` (cefrLevel≠null) = **missing**. Tái dùng logic `missing` đang cấp cho `MissingSkillCta`.

### 3.2 UI khoá + CTA
- `SkillBars.tsx`: skill ∈ missing → render **locked** (mờ opacity, icon khoá, nhãn "Chưa đánh giá" thay band).
- `MissingSkillCta.tsx`: đổi hành vi CTA — thay vì `useRetake` (làm lại cả bài), gọi **`useStartProductiveSkills`** (khởi tạo bài Nói & Viết) → `router.push(ROUTES.ASSESSMENT_EXAM(attemptId))`. Nếu missing chứa cả W/S → 1 CTA "Đi đến đánh giá" (D1 gộp).
- Giữ nút tổng "Làm lại bài đánh giá" (retake) như hiện tại.

### 3.3 Service/Hook
- `assessment.service.ts`: `startProductiveSkillsAssessment()` — POST intake `goal='productive_skills'` (dùng đúng endpoint intake hiện có).
- `use-assessment.ts`: `useStartProductiveSkills()` (mutation) — onSuccess điều hướng exam.
- Tuân `exe-web/docs`: service→hook→component, không sửa `components/ui/`, Tailwind token, TanStack Query.

## 4. Mobile (studee-app) — `features/exam_mst`

### 4.1 Skills-driven flow (fix bug + hỗ trợ bài W/S)
- `data/exam_mst_repository_impl.dart`: map `skills` từ start/attempt response → Entity; giữ `goal` param (thêm `productive_skills`).
- `state`: thêm `skills`.
- `bloc`:
  - Placement (`skills=[reading,listening]`): sau R+L, `_startWsPhase` **guard** — nếu skills không có W/S **hoặc** `ws-tasks` trả rỗng/lỗi → `add(ExamMstWsSubmitted())` (submit thẳng, không vào phase trống). (AC9)
  - Bài Nói & Viết (`skills=[writing,speaking]`): `_mcqSkills` lọc theo skills → rỗng phần R/L → vào thẳng `ws-tasks`. (AC10)
- `mst_ws_phase_body.dart`: phòng thủ writing rỗng (không render "0/0").

### 4.2 Entry + result
- `domain/usecase/start_productive_skills_exam_usecase.dart` (song song `start_new_mst_exam_usecase.dart`): intake `goal='productive_skills'`.
- `mst_result_skill_breakdown_card.dart`: locked row (mờ+khoá) + nút "Đi đến đánh giá" (navigate trong Bloc) cho kỹ năng chưa có.
- l10n `.arb` (en/vi/ja): "Chưa đánh giá", "Đi đến đánh giá".
- Tuân rule app: token màu/size/style/string; iOS mic native theo `platform-native.md` (chỉ Dart + fallback nếu máy không archive iOS).

## 5. Rủi ro & giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| Bank W/S `general` rỗng → attempt không có task | Task đầu Phase BE: đếm item writing/speaking `targetGoal∈{general}`; nếu thiếu → tạo/nạp trước (chặn merge). |
| `ws-tasks` guard đổi làm hỏng luồng cũ | Load assessment trước rồi mới `mstAllowed`; test placement + diagnostic(cờ tắt→404) + productive(cờ tắt→OK). |
| Quota chặn nhầm khi limit paid chưa wire | D4: enforce theo cấu hình hiện hành; số cụ thể tinh chỉnh sau. Ghi rõ trong AC2. |
| iOS speaking mic (native) | Chỉ sửa Dart, fallback `MissingPluginException`; spec native nếu máy không build iOS. |
| Hai "đánh giá" gây rối UX | Nhãn rõ: "Làm lại (tổng)" vs "Đi đến đánh giá (phần thiếu)". |

## 6. Cross-repo

Một spec, 3 PR (mỗi repo 1 PR, base `develop`), theo `.ai/project-context.md §0`. Không có contract nhị phân mới —
chỉ dựa `skills` trong response (đã tồn tại). Không cần `data-model.md` / `contracts/*.md` riêng.

## 7. Điểm cần người quyết trước khi code (nếu có)

- Số lượng task Writing/Speaking mỗi bài (preset) — nếu khác preset hiện hành, chốt con số ở Task BE-2.
