# Acceptance Criteria: Checkpoint chặng/khóa + nâng cấp CEFR band

- **Spec:** `specs/course-checkpoints-band-up/spec.md`
- **Ngày:** 2026-08-03

## Cách dùng file này

Mỗi tiêu chí có ID (AC-1, AC-2, …) map về mục "Trong phạm vi" ở spec.md. Format Given/When/Then khi hành vi state-dependent; checklist đơn giản khi checkable trực tiếp.

## Tiêu chí chức năng

### AC-1: Checkpoint node hiển thị + 3 trạng thái

- [ ] Sidebar có node Checkpoint cuối chặng (sau bài cuối cùng).
- [ ] Node Checkpoint ở khóa (after last phase) nếu khóa có checkpoint.
- **Given** node trạng thái "locked"
  - **When** chọn nó **Then** card hiển thị "Checkpoint available sau khi hoàn thành bài học" + disable runner.
- **Given** node trạng thái "available" (on-path done + chưa pass)
  - **When** chọn nó **Then** content panel = CheckpointInlineRunner, learner thấy đề + submit button.
- **Given** node trạng thái "passed"
  - **When** chọn nó **Then** result screen hiển thị (pass outcome) + link retake (vẫn bấm được).

### AC-2: 2-step gate — on-path complete → available

- **Given** learner trong course, chặng có checkpoint
  - **When** hoàn thành tất cả bài on-path chặng
  - **Then** checkpoint node = "available" (progressively ở sidebar)
- [ ] Non-on-path bài (từ adaptive path khác) KHÔNG block available (D1 260729).
- [ ] Chặng không có on-path module → checkpoint available ngay (vacuous step A).

### AC-3: 2-step gate — pass → unlock chặng kế

- **Given** checkpoint available, learner submit
  - **When** overall score ≥ passThreshold (default 0.8, admin config)
  - **Then** checkpoint node = "passed" ✓, chặng kế unlock, nav "Chặng kế" enable + clickable
- **When** overall score < threshold
  - **Then** checkpoint node vẫn "available", fail screen hiển thị, suggest retry.
- [ ] Khóa-level checkpoint pass → course = "completed" (nếu tất cả skill ≥ target hoặc passed khóa).

### AC-4: Nav-lock + tooltip

- [ ] Khi checkpoint NOT available: nút "Chặng kế"/"Bài tiếp" disable + tooltip "Hãy hoàn thành bài học đầu tiên".
- [ ] Khi checkpoint available nhưng NOT passed: nút "Chặng kế" disable + tooltip "Hãy pass checkpoint trước".
- [ ] UI KHÔNG cho phép bấm (onclick block, hoặc href KHÔNG navigate).

### AC-5: Unlimited retry, NO penalty

- [ ] Learner có thể submit checkpoint đạo 1, 2, 10 lần.
- [ ] Mỗi submit → best-score-wins (API trả passed=true nếu 1/N lần pass).
- [ ] Fail lần trước → KHÔNG deduct band, KHÔNG cooldown, KHÔNG "banned" message.
- [ ] Vẫn pass lần sau → cập nhật band (idempotent per boundary/skill via creditedBoundaries).

### AC-6: Mixed test MCQ per chặng

- [ ] Checkpoint quiz = LessonQuiz curated (quizType='curated', KHÔNG 'single_skill').
- [ ] Items trả về từ getCheckpointItems = gộp MCQ từ 2+ kỹ năng chặng (hoặc tất cả nếu 1 skill).
- [ ] Admin set via seed (data) hoặc admin UI (Phase Checkpoint Control) sau.
- [ ] getCheckpointItems trả { quizCode, passThreshold, items:[], stimuli:[] }.

### AC-7: Pass threshold admin-configurable

- [ ] CheckpointRef (phase.checkpoint + course.checkpoint) chứa `passThreshold` (Number, 0–1).
- [ ] getCheckpointItems trả `passThreshold` = `ref.passThreshold ?? lessonQuiz.passThreshold ?? 0.8`.
- [ ] Admin set qua endpoint `PUT /api/admin/courses/:id/phase-checkpoint` (body: `{ passThreshold: 0.75, … }`).
- [ ] Verify line: `checkpoint.service.js:77` — effectiveThreshold.

### AC-8: BandPoint 1.0–6.0 model

- [ ] StudentModel.skill[X].bandPoint = Number (1.0–6.0) — NOT integer.
- [ ] CEFR display = `floor(bandPoint)` (A1: 1–1.99, A2: 2–2.99, …, C2: 6–6.xx).
- [ ] Per-chặng budget frozen vào StudentPath.skillBudget (phân bổ từ target−current ÷ N chặng).
- [ ] Per-skill target = `StudentPath.skillTarget[skill]` (không thay đổi sau assign).
- Verify: `student-model.model.js:24`, `adaptive-path-snapshot-view.js` skill target/budget.

### AC-9: Idempotent credit

- [ ] StudentModel.skill[X].creditedBoundaries = [String] (phaseKey list).
- [ ] submitCheckpoint pass → per-skill call applyCheckpointResult.
- [ ] applyCheckpointResult chỉ credit 1 lần/boundary/skill (if !creditedBoundaries.includes(boundaryKey)).
- [ ] Retake chặng → boundaryKey đã trong array → skip, KHÔNG stack 2x budget.
- Verify: `adaptive.service.js` applyCheckpointResult + creditedBoundaries check.

### AC-10: Testlet stimulus serving

- [ ] getCheckpointItems({ userId, slug, boundaryKey }) trả:
  ```json
  {
    "quizCode": "chkpt-phaseA",
    "passThreshold": 0.8,
    "items": [{ "id":"q1", "text":"…", "type":"mcq", "stimulusId":"stim-1", … }],
    "stimuli": [{ "id":"stim-1", "kind":"audio", "media":{ "audioUrl":"https://…?sig=…" } }]
  }
  ```
- [ ] Audio URL signed (TTL ~3h = COURSE_AUDIO_TTL_SEC = 10800 sec), KHÔNG raw path.
- [ ] Reading passage stimulus = kind='passage', `body` = text, KHÔNG transcript.
- [ ] Nếu stimulus retire/missing → `stimuli[]` vắng mặt, item vẫn có stimulusId → FE fallback render flat.
- Verify: `testlet-stimuli.js:18`, `checkpoint.service.js:93`.

### AC-11: Result screen — Pass

- [ ] Heading: "Chúc mừng! Bạn đã pass checkpoint."
- [ ] Per-skill score: "Listening: 0.85/1.0", "Reading: 0.72/1.0".
- [ ] Raised skills: "Listening: A2 → B1 ✓ (nâng cấp)" (highlight/badge).
- [ ] Weak scored-not-raised: "Writing: 0.6/1.0 (chưa đạt ngưỡng)" (shown, not raised).
- [ ] CTA: "[Tiếp tục]" → unlock chặng kế + navigate.
- [ ] Suggestions: link study weak subskills (nếu có).

### AC-12: Result screen — Fail

- [ ] Heading: "Bạn chưa pass. Hãy thử lại."
- [ ] Per-skill score breakdown.
- [ ] Suggestions: "Nghe chi tiết: 0.4/1.0 — hãy review [lesson link]".
- [ ] CTA: "[Quay về lộ trình]", "[Thi lại]".
- [ ] KHÔNG deduct band, KHÔNG "banned" msg.

### AC-13: Competency history page

- **Endpoint:** `GET /api/adaptive/band-history` → `{ history:[] }`
- **Page:** `/adaptive/history` (exe-web)
- [ ] Timeline hiển thị newest first (takenAt sort).
- [ ] Per entry:
  - Badge `sourceGroup` = "Đánh giá" (assessment) / "Checkpoint" / "Tái kiểm tra" (skill_retest).
  - `overallCefr` + `skillCefr` snapshot (listening: A2, reading: B1, …).
  - `changedSkills`: [skill names] raised.
  - Date + time.
- [ ] Baseline row (current model) if history empty.
- Verify: `adaptive.service.js:1196` getBandHistory + `learning-event.model.js` sourceGroup enum.

### AC-14: Admin authoring checkpoint

- [ ] Endpoint: `PUT /api/admin/courses/:id/phase-checkpoint`
- **Given** admin has `course_content:manage` permission
  - **When** set `{ phaseKey: 'phase-1', quizCode: 'quiz-123', passThreshold: 0.75 }`
  - **Then** validate LessonQuiz 'quiz-123' is published (else 409 QUIZ_CONFIG_NOT_PUBLISHED).
  - **Then** persist phase.checkpoint ref + save CourseStructure.
- [ ] admin UI = `exe-admin` PhaseCheckpointControl (pending full UI build; MVP = seed data).
- Verify: `course-content.admin.js` setPhaseCheckpoint + `checkpoint.service.js` loadCheckpoint validation.

### AC-15: Permission checks

- [ ] POST /courses/:slug/checkpoint/submit — requires `courses:read` (implicit via enrollment assertEnrolled).
- [ ] PUT /api/admin/courses/:id/phase-checkpoint — requires `course_content:manage` + center scope.
- [ ] GET /api/adaptive/band-history — requires `verifyToken` (self-service, userId from JWT).

---

## Tiêu chí lỗi / edge case

### AC-E1: Checkpoint not found

- **Given** phase/course không có checkpoint config
  - **When** call getCheckpointItems
  - **Then** 404 CHECKPOINT_NOT_FOUND "Chặng/khóa này chưa cấu hình checkpoint"

### AC-E2: Quiz not published

- **Given** checkpoint ref quizCode tìm được nhưng quiz status ≠ 'published'
  - **When** call getCheckpointItems
  - **Then** 409 QUIZ_CONFIG_NOT_PUBLISHED "Quiz checkpoint chưa được publish"

### AC-E3: Not enrolled

- **Given** learner not enrolled in course
  - **When** call getCheckpointItems
  - **Then** 403 (enrollment check trước via assertEnrolled)

### AC-E4: No StudentModel (pre-assessment)

- **Given** learner chưa làm đánh giá (KHÔNG StudentModel)
  - **When** call getBandHistory
  - **Then** 409 NEEDS_ASSESSMENT "Hãy hoàn thành đánh giá trước"

### AC-E5: Stimulus retire/missing

- **Given** item có stimulusId tham chiếu stimulus đã xóa/retire (status ≠ 'active')
  - **When** getCheckpointItems
  - **Then** stimuli[] không include nó; item vẫn trả; FE fallback render item flat (KHÔNG crash)
- Verify: testlet-stimuli.js — filter `status:'active'` + map fallback.

### AC-E6: Audio URL outside signable prefix

- **Given** AssessmentStimulus.media.audioUrl ngoài prefix signed (vd random CDN)
  - **When** buildTestletStimuli sign it
  - **Then** signAudioSafe return null (KHÔNG try serve unsigned); item.media.audioUrl = null
- Verify: testlet-stimuli.js:30 — check signMediaUrl === url → return null (warn log).

### AC-E7: Sign error (soft fail)

- **Given** signMediaUrl throw (vd key issue)
  - **When** buildTestletStimuli call sign
  - **Then** catch error, return null, warn log; stimulus vẫn có id/kind/body (partial), audioUrl=null
- FE shows "audio unavailable".

### AC-E8: Boundary key không hợp lệ

- **Given** call submitCheckpoint với boundaryKey='invalid-key' (không match phase hoặc 'course')
  - **When** loadCheckpoint → resolveCheckpointRef
  - **Then** 404 CHECKPOINT_NOT_FOUND

---

## Tiêu chí phi chức năng

| Loại | Tiêu chí | Cách đo |
|---|---|---|
| Bảo mật | checkpoint payload KHÔNG leak answer key (resolveQuizItems select:false) | Read quiz-grading.js SELECT clause; verify hidden `correctAnswer` |
| Bảo mật | transcript NEVER loaded từ stimulus (select:false) | Read testlet-stimuli.js `.select('kind skill body media')`  — không 'transcript' |
| Bảo mật | audio URL signed, không public raw path serve | Read media-sign.js; verify signMediaUrl() call trước return |
| Bảo mật | Permission enforce (course:read, course_content:manage) | Read adaptive.controller, admin-api assert verifyPermission |
| Hiệu năng | resolveQuizItems (resolve curated itemIds) < 500ms cho 50 items | Benchmark với N=50 curated items; measure query + gradeItem loop |
| Hiệu năng | buildTestletStimuli (unique stimuli + sign) < 200ms cho 40 items (10 stimuli) | Benchmark; include AssessmentStimulus.find + map(sign) |
| Hiệu năng | getBandHistory sort + return < 100ms | Trace query time (history append-only, sort in-memory) |
| Khả dụng | Audio TTL 3h sufficient (course checkpoint no time limit) | Verify COURSE_AUDIO_TTL_SEC = 10800; no learner reload mid-attempt timeout expected |

---

## Ngoài phạm vi kiểm thử

Phần cố ý không viết acceptance (liệt kê ngoài phạm vi ở spec.md):
- "1 audio : N câu" testlet (hoãn initiative A).
- Non-MCQ testlet (true/false, matching).
- Chặn tốt nghiệp nếu dưới target cuối khóa.
- Native exam scores (IELTS/TOEIC → chỉ map hiển thị).
- Per-bài/chuyên đề analytics.
