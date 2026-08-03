# Spec: Checkpoint chặng/khóa + nâng cấp CEFR band (Checkpoint Band-up)

- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code)
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — checkpoint service, band-up core, testlet stimulus serving
  - `web` (`exe-web`) — checkpoint node (sidebar), result screen, competency history page
  - `admin` (`exe-admin`) — phase checkpoint authoring (pick quiz, set threshold)
- **Module liên quan (nếu là `exe-api`):**
  - `services/api/src/modules/checkpoint/` — checkpoint service, grading, verdict, events
  - `services/api/src/modules/adaptive/` — band-up core (`applyCheckpointResult`, band-history)
  - `services/api/src/modules/assessment/` — testlet stimulus model + serving
  - `services/api/src/modules/quiz/` — lesson quiz model, grading, resolveQuizItems
  - `services/api/src/modules/course-content/` — course/phase structure, progress gating
- **Trạng thái:** Đã build (spec hồi tố)

## 1. Mục tiêu

Cho phép học viên thi checkpoint (kiểm tra kỹ năng) ở cuối mỗi **chặng (phase)** và cuối **khóa (course)** để gating tiến độ (đi chặng kế chỉ nếu pass) + **nâng cấp trình độ CEFR band liên tục** (1.0–6.0, không +1 nguyên) cho kỹ năng tested, với phân bổ ngân sách đều theo chặng để cuối khóa land gần target.

## 2. Bối cảnh

### Vì sao cần

Chặng/khóa cần một cánh cổng học tập để học viên xác nhận nắm vững các kỹ năng của chặng đó. Toàn ngành ELTS đều có cuộc kiểm tra cuối chương (chapter exam / milestone test); để track tiến độ và xác định khi nào học viên sẵn sàng thử thi IELTS/TOEIC, hệ thống cần checkpoint. Checkpoint pass → nâng cấp band kỹ năng; cuối khóa, nếu pass được tất cả checkpoint → đạt target.

### Lịch sử quyết định + supersession

Năm ngoài (2026-07-29) ba plans phát triển checkpoint được duyệt:
- **Plan A (260729-1607):** design ban đầu — band-up `+1 CEFR/chặng` (nguyên), Đó là cơ bản: chặng + khóa checkpoint, mixed test, unlimited retry, no penalty.
- **Plan B (260729-checkpoint-vuot-chang-restructure):** **SUPERSEDES checkpoint UX của A** — thay đổi giao diện: checkpoint không inline dưới bài → node đặc biệt sidebar + content panel (study-room), 2-step gate (hoàn bài on-path → available checkpoint → pass → chặng kế mở), admin UI set checkpoint per chặng. **Toàn bộ đã code xong & deploy**.
- **Plan C (260730-band-up-per-chang-distribution):** **SUPERSEDES band-up của A** — thay `+1 CEFR nguyên` → **fractional `bandPoint` 1.0–6.0**, phân bổ budget đều theo số chặng dạy skill, credit khi pass checkpoint, cuối khóa land target. **Toàn bộ đã code xong & deploy**.
- **Plan D (260730-checkpoint-testlet-audio-serving):** testlet stimulus (audio listening/passage reading) serve cho checkpoint dùng 1:1 audio có sẵn, payload additive, FE reuse `TestletCard`. **Toàn bộ đã code xong & deploy**.

**Spec này NHẤT QUÁN với C + D (band-up & testlet — final state); B chỉ nêu lại cấu trúc UX đã fixed.**

### Hiện trạng đã kiểm tra trong code

**Band-up model (bandPoint):**
- `modules/adaptive/student-model.model.js:24` — `bandPoint` field (Number, 1.0–6.0) trên `SkillAbilitySchema`. `cefr` là floor để hiển thị; `bandPoint` là source of truth.
- `constants/cefr-mapping.js:1` — `CEFR_NUM = { A1:1, A2:2, B1:3, B2:4, C1:5, C2:6 }` (map để dịch).
- `student-model.model.js:27` — `creditedBoundaries: [String]` idempotent per-chặng credit.
- `adaptive-path.snapshot.schema` — `skillBudget` (per-skill per-chặng phân bổ) + `skillTarget` (bandPoint đóng băng) lưu lúc assign khóa.

**Checkpoint infrastructure:**
- `modules/checkpoint/checkpoint-attempt.model.js` — `{ userId, courseId, boundaryKey, quizCode, score, passed, perItem, skillScores:[{skill, score, raised}] }`. `boundaryKey` = `phaseKey` (chặng) hoặc literal `'course'` (khóa).
- `modules/checkpoint/checkpoint.service.js:77` — `effectiveThreshold = ref.passThreshold ?? lessonQuiz.passThreshold ?? 0.8`. Default 80% (admin config per checkpoint).
- `checkpoint.service.js:159` → `submitCheckpoint` → grade qua `gradeItem`, compute verdict, **per-skill if sub-score ≥ threshold call `adaptive.applyCheckpointResult(skill, true)`** → credit band + log history.
- `modules/course-content/course-content.progress.service.js` — `buildProgress` extend để checkpoint là node boundary; phải pass để unlock chặng kế (2-step gate).

**Testlet stimulus:**
- `modules/assessment/testlet-stimuli.js:48` — `buildTestletStimuli(items)` → tìm unique `stimulusId` từ items, load `AssessmentStimulus` (status='active'), sign audio (TTL=3h cho course), return `{ id, kind, skill, body, media.audioUrl }`.
- `checkpoint.service.js:93` — `getCheckpointItems()` → `resolveQuizItems` + `buildTestletStimuli` → return `{ items[], stimuli[] }`.
- `resolveQuizItems` (modules/quiz/quiz.grading.js) → thêm `stimulusId` vào từng item từ question schema.

**Competency history:**
- `modules/adaptive/adaptive.service.js:1196+` — `getBandHistory(userId)` → `LearnerProfile.history` (append-only log), sort newest first, mỗi entry: `{ takenAt, sourceGroup, bandSource, overallCefr, skillCefr, changedSkills }`.
- `sourceGroup` enum: `'assessment'` / `'checkpoint'` / `'skill_retest'` / `'ops_recompute'` / `'ops_revoke'` / `'ops_recalibrate'` — **checkpoint tạo entry với `sourceGroup:'checkpoint'`**.
- `bandSource` → source nào gây ra band change (used in profile view BR-33).

## 3. Phạm vi

### Trong phạm vi

- **IS-1** — Checkpoint = bài kiểm tra cuối chặng (phaseKey) + cuối khóa (literal `'course'`), lưu ref `phase.checkpoint` + `course.checkpoint`.
- **IS-2** — Checkpoint node ở sidebar (study-room) = node đặc biệt, 3 trạng thái (khóa/sẵn sàng/đạt). Chọn → content panel = checkpoint runner (KHÔNG route mới; tái dùng `CheckpointInlineRunner`).
- **IS-3** — 2-step gate:
  - **Bước A:** Hết tất cả bài on-path chặng → checkpoint available.
  - **Bước B:** Pass checkpoint (overall score ≥ admin threshold, default 0.8) → chặng kế mở, khóa completed nếu khóa pass.
- **IS-4** — Gating locked: nút "Bài tiếp"/"Chuyên đề tiếp"/"Chặng kế" disable + tooltip khi chưa đủ điều kiện.
- **IS-5** — Unlimited retry, **NO cooldown, NO band penalty on fail**. Fail → sở hữu pass → không đổi band, suggest study.
- **IS-6** — **Mixed test per chặng:** 1 `LessonQuiz` curated chứa MCQ từ các kỹ năng của chặng (admin pick qua seed / admin UI later), serve đầy đủ.
- **IS-7** — **Pass threshold admin-configurable:** `CheckpointRef.passThreshold` (default 0.8 từ LessonQuiz); admin set per chặng.
- **IS-8** — **Band-up model = fractional bandPoint (1.0–6.0):** 
  - Mỗi chặng có ngân sách `skillBudget[skill]` (phân bổ từ target−current ÷ số chặng dạy skill).
  - Pass checkpoint chặng → **per-skill sub-score ≥ threshold** → credit `bandPoint += skillBudget[skill]` (clamped ≤ target).
  - `cefr` hiển thị = `floor(bandPoint)` (A1=1–1.99, A2=2–2.99, …, C2=6–6.xx).
  - Chặng khóa (course) **KHÔNG credit band**, chỉ gate completion.
- **IS-9** — Idempotent credit: `creditedBoundaries[skill]` track chặng đã credit → retake KHÔNG stack budget 2 lần.
- **IS-10** — **Testlet stimulus serving:** checkpoint serve listening audio (AssessmentStimulus → signable URL, TTL~3h) + reading passages, payload `{ items[], stimuli[] }` additive (items phẳng không đổi, stimuli[] top-level).
- **IS-11** — **Result screen (pass & fail):**
  - **Pass:** per-skill score, which raised (highlighted), weak scored-not-raised (shown), "Tiếp tục" unlock next chặng.
  - **Fail:** per-skill score, weak subskills (study suggestions), "[Quay về lộ trình]" / "[Thi lại]".
- **IS-12** — **Competency history page** `/adaptive/history` — timeline `sourceGroup` badge (assessment/checkpoint), `skillCefr` snapshot per entry, `changedSkills`, `takenAt`.
- **IS-13** — **Admin authoring:** `PUT /api/admin/courses/:id/phase-checkpoint` set checkpoint ref per chặng (validate published LessonQuiz, persist `phase.checkpoint`).
- **IS-14** — **Phân quyền:** 
  - Checkpoint submit: `courses:read` (implicitly via enrollment).
  - Admin set checkpoint: `course_content:manage` + center scope.

### Ngoài phạm vi

- "1 audio : N câu" testlet (author riêng checkpoint pool, sửa CAT query) — **Lý do:** hoãn initiative A sau; hiện phục vụ 1:1 audio có sẵn.
- Non-MCQ testlet (true/false, matching…) — **Lý do:** chỉ MCQ curated; non-MCQ sau.
- Chặn tốt nghiệp hoặc bài tập bù nếu cuối khóa dưới target — **Lý do:** band-up framework chỉ credit; chặn/remediation là initiative sau.
- Điểm thi native (IELTS/TOEIC score) → chỉ map CEFR hiển thị, không lưu − **Lý do:** out of scope; sau.
- Phân tích per-bài/chuyên đề (analytics) — **Lý do:** checkpoint tổng hợp skill; chi tiết sau.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Học viên (Learner) | Hoàn thành hết bài on-path chặng → checkpoint available → làm bài kiểm tra chặng → xem kết quả (pass/fail) | Xác nhận nắm kỹ năng, nâng cấp band, đi chặng kế, track tiến độ qua history timeline |
| Học viên (fail) | Xem lại đề yếu, thi lại unlimited (KHÔNG phạt) | Cải thiện score, pass, nâng band cuối cùng |
| Admin nội dung | Pick published LessonQuiz, set passThreshold per chặng (admin UI hoặc seed) | Kiểm soát độ khó checkpoint + điều chỉnh ngưỡng vượt qua |
| Tech Lead / Ops | Có band-change log append-only (checkpoint entries sourceGroup='checkpoint') | Audit band-up, điều tra nếu band bất thường, hỗ trợ learner dispute |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Lựa chọn đề xuất | Người quyết |
|---|---|---|
| Pass boundary checkpoint **LOCK chặng kế** hay chỉ **UNLOCK KHÔNG THƯỜNG XUYÊN**? | **Chặng kế LOCK** cho đến pass. 2-step gate quy định rõ. | ✓ Đã chốt (owner 260729) |
| Fail checkpoint: **cộng thêm attempt hoặc bỏ qua (best-score-wins)**? | Best-score-wins, **retry unlimited KHÔNG cool-down KHÔNG phạt**. | ✓ Đã chốt (BR-31) |
| Band-up: **+1 CEFR nguyên/chặng hay fractional/liên tục**? | **Fractional bandPoint 1.0–6.0**, phân bổ đều per-chặng. | ✓ Đã chốt (owner 260730) |
| Course boundary (khóa) checkpoint: **credit band như chặng hay chỉ gate**? | **Chỉ gate completion** (KHÔNG credit band per-skill; chặng là nơi credit). | ✓ Đã chốt (D4 260730) |
| Testlet stimulus: **author riêng hay reuse có sẵn**? | **Reuse 1:1 audio có sẵn** (scope B); "1 audio N câu" = initiative A sau. | ✓ Đã chốt (scope B 260730) |

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map về một mục "Trong phạm vi".

1. Checkpoint node ở sidebar (3 trạng thái) + content panel run test. *(IS-2)*
2. 2-step gate: on-path hết → available; pass → chặng kế mở. *(IS-3)*
3. Nav-lock: nút disable + tooltip khi chưa đủ. *(IS-4)*
4. Unlimited retry, no penalty, fail→no band change. *(IS-5)*
5. Mixed test per chặng (MCQ) + admin threshold setting. *(IS-6, IS-7)*
6. BandPoint 1.0–6.0, phân bổ per-chặng, credit khi pass. *(IS-8, IS-9)*
7. Testlet stimulus serving (audio + passage), payload additive. *(IS-10)*
8. Result screen pass/fail (per-skill, raised/weak, suggestions). *(IS-11)*
9. Competency history page timeline (sourceGroup, skillCefr, changedSkills). *(IS-12)*
10. Admin set checkpoint per chặng (LessonQuiz published, threshold). *(IS-13)*
11. Phân quyền (course read, course-content manage). *(IS-14)*

## 7. Câu hỏi mở

Spec này hồi tố **TÌNH TRẠNG ĐÃ BUILD** — không còn câu hỏi vì 4 plans đã duyệt + code đã implement. Để ghi lại: **không có thắc mắc chặn approval.**

## 8. Ghi chú cho Tech Lead Design

Ràng buộc đã xác minh (không phải giải pháp kỹ thuật):

- **Band model:** Anchor `bandPoint = CEFR_NUM[cefr]` đầu band (A1=1, …, C2=6). Fractional liên tục 1.0–6.0 (không integer). Floor để hiển thị.
- **Per-chặng budget:** Frozen lúc assign lộ trình (StudentPath snapshot); **NOT read from course live** (tránh race khi admin sửa chặng). Đóng băng `skillBudget + skillTarget` trên snapshot.
- **Idempotent credit:** `creditedBoundaries` track boundary key → chỉ credit 1 lần/chặng/skill, dù retake 100 lần. Nếu bỏ field này → risk stack budget.
- **Gate = 2 bước (Bước A + B):** Không phải logic chỉnh sửa trong progress — chỉnh rõ gì là "on-path", "available", "passed" trước khi code.
- **Testlet payload additive:** items[] phẳng không thêm `media`, chỉ `stimulusId` (inert cho lesson-quiz). stimuli[] top-level. Grading/submit contract KHÔNG đổi.
- **Audio TTL dài:** Course-side 3h (COURSE_AUDIO_TTL_SEC = 10800 sec, testlet-stimuli.js:18) vs exam 30' — vì learner không persist state reload mất đáp án.
- **McQ-only serve:** Checkpoint serve lọc MCQ; chấm KHÔNG lọc → non-MCQ sẽ invisible input nhưng count SAI. Ghi rõ ràng buộc để dev test.
- **Learner profile history:** append-only (KHÔNG overwrite). Each checkpoint pass → log `HistoryEntry { sourceGroup:'checkpoint', takenAt, skillCefr:{}, changedSkills:[] }`. Baseline row (model hiện tại) nếu history empty.
