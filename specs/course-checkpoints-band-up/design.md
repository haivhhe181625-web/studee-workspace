# Design: Checkpoint chặng/khóa + nâng cấp CEFR band

- **Spec:** `specs/course-checkpoints-band-up/spec.md`
- **Ngày:** 2026-08-03
- **Tác giả:** Tech Lead (verification từ code + plans)
- **Trạng thái:** Đã duyệt (spec hồi tố từ implement)
- **ADR liên quan:** 
  - `docs/adr/admin-api-migration.md` (AdminJS → exe-admin REST)
  - Band-up history: `adaptive-path/design.md` §Band history

## 1. Tóm tắt kiến trúc

Checkpoint là một bài kiểm tra ở **cấp boundary (chặng/khóa)** mục đích gating tiến độ + nâng cấp band kỹ năng. Hệ thống **tái dùng ~80%** LessonQuiz engine (curated quiz, grading, frozen item set), thêm:

1. **Boundary gate logic** — 2-step (on-path done → available → pass → unlock next) ở `course-content.progress.service.js`.
2. **Band-up refactor** — từ `+1 CEFR nguyên` → **fractional bandPoint 1.0–6.0** với per-chặng budget, frozen lúc assign lộ trình (StudentPath snapshot).
3. **Testlet stimulus payload** — audio/passage serve via `buildTestletStimuli`, additive `stimuli[]` + signed URLs.
4. **Result UI** — pass/fail screen ở web, competency history page `/adaptive/history`.

**Không tạo collection/model mới cho course-side checkpoint:** reuse `CheckpointAttempt` (mirrors `QuizAttempt` nhưng keyed by `boundaryKey` không `lessonPathKey`), store `phase.checkpoint` ref trực tiếp trên `PhaseSchema` (course-content.model).

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| **Checkpoint boundaries** | Chặng (phaseKey) + Khóa (literal 'course') | MVP scope; module/chuyên đề tính sau nếu cần | Không support granular checkpoint per chuyên đề (chưa cần) |
| **Band-up model** | Fractional bandPoint 1.0–6.0 | Liên tục, không nguyên; smooth progression; cuối khóa land target | +1 CEFR nguyên (cũ) = quá coarse, dồn cục early → dropped |
| **Budget allocation** | Chia đều per-chặng dạy skill: `budget[skill] = (target[skill] − current[skill]) / N[skill]` | Fair, predictable; frozen at assign-time → no race with live edit | Dynamic replan = complex; static allocation simpler |
| **Credit trigger** | Pass checkpoint → per-skill if sub-score ≥ threshold | Skill-level credit (not whole test); weak skills shown not raised | All-or-nothing (mọi skill cùng lúc) = coarse |
| **Idempotent credit** | `creditedBoundaries[skill]` track boundary keys | Retake KHÔNG stack budget 2x; dễ debug | Phức tạp hơn "check last raised" (cần track full list) |
| **Gate = 2-step** | Bước A: on-path done → available; Bước B: pass → unlock | Clear semantics; on-path-aware (defer non-on-path); hard gate chặn next | 1-step (chỉ pass gate) = KHÔNG enforce on-path→available clarity |
| **Course-level checkpoint** | NO per-skill credit (chỉ gate completion) | Boundary khóa là "finish line"; band-up = per-chặng | Could credit như chặng (overlap scope) |
| **Quiz curated MCQ-only** | Checkpoint serve lọc MCQ; grade KHÔNG lọc → non-MCQ invisible | Clean scope B; avoid server mutation (CAT pool); simple serve | Non-MCQ testlet (matching, etc) hoãn; users see only MCQ |
| **Testlet stimulus** | Payload additive: items[] phẳng + stimuli[] top-level | Không đổi grading/submit contract; FE render logic chỉ thêm | Grading complexity KHÔNG tăng |
| **Audio TTL** | Long TTL (~3h) cho course-side (vs 30' exam) | Learner không persist state → reload = mất đáp án; need long link life | Token expiry risk low (internal signing) |
| **History log append-only** | HistoryEntry sourceGroup enum (assessment/checkpoint/…) | Audit trail; no overwrite; per-entry tracking | Storage (log grows) — mitigated by trim-old later |

## 3. Data model

Chi tiết ở `data-model.md`. Tóm tắt:

### M1 — Checkpoint ref + Attempt

```javascript
// CourseStructureSchema.phases[].checkpoint (hoặc CourseStructure.checkpoint cho khóa)
CheckpointRefSchema {
  quizCode: String,                 // tham chiếu LessonQuiz published
  passThreshold: Number (0–1),      // admin set (default 0.8)
  label: String?                    // "Chapter 1 Exam" (optional)
}

// CheckpointAttempt — one learner submission
{
  userId: ObjectId,
  courseId: ObjectId,
  boundaryKey: String,              // phaseKey hoặc 'course'
  quizCode: String,
  score: Number (0–1),              // overall
  passed: Boolean,
  perItem: [{ questionId, skill, correct }],
  skillScores: [{ skill, score (0–1), raised }],  // per-skill verdict
  createdAt: Date
}
```

### M2 — Band model (StudentModel.skill)

```javascript
SkillAbilitySchema {
  theta: Number?,                   // từ assessment CAT
  se: Number?,
  cefr: String (enum CEFR_LEVELS),  // display (floor of bandPoint)
  bandPoint: Number,                // 1.0–6.0 (source of truth)
  creditedBoundaries: [String]      // boundary keys đã credit skill này
}
```

### M3 — StudentPath snapshot

```javascript
// Frozen at assign-time (buildStudentPath → saveSnapshot)
snapshot {
  courses: [{
    skillBudget: { skill: Number, … },  // per-chặng phân bổ per-skill
    skillTarget: { skill: Number, … },  // đóng băng target bandPoint per-skill
    phases: [{
      key: String,
      skillBudget: { skill: Number, … }
    }]
  }]
}
```

### M4 — History entry

```javascript
LearnerProfile.history [] {
  takenAt: Date,
  sourceGroup: String (enum: assessment/checkpoint/skill_retest/…),
  overallCefr: String,
  skillCefr: Map<skill, cefr>,
  changedSkills: [String],
  …(existing fields)
}
```

## 4. Luồng dữ liệu

### Track A: Learner submit checkpoint → grade → band-up → history

```
Learner POST /courses/:slug/checkpoint/submit { answers:[] }
  ↓
  submitCheckpoint({ userId, slug, boundaryKey, answers })
  ├─ loadCheckpoint → resolve CheckpointRef + LessonQuiz + threshold
  ├─ fetchGradableItems → decrypt answers, fetch question bank
  ├─ gradeItem (via gradeItem registry) → per-item correct/incorrect
  ├─ computeVerdict → overall score + per-skill sub-scores
  ├─ persist CheckpointAttempt { userId, boundaryKey, score, skillScores }
  │
  └─ IF passed (score ≥ threshold):
      ├─ resolveBandBudget(userId, slug, boundaryKey) → StudentPath.phases[boundaryKey].skillBudget
      ├─ FOR EACH skill: IF sub-score ≥ threshold AND !creditedBoundaries.includes(boundaryKey):
      │   ├─ adaptive.applyCheckpointResult(userId, skill, true, attemptId)
      │   │   ├─ fetch StudentModel.skill[skill].bandPoint + creditedBoundaries
      │   │   ├─ bandPoint += skillBudget[skill] (clamped ≤ target)
      │   │   ├─ add boundaryKey → creditedBoundaries (idempotent)
      │   │   ├─ log LearnerProfile.history { sourceGroup:'checkpoint', skillCefr, changedSkills:[] }
      │   │   └─ emit checkpoint.passed event
      │   └─ flag skill.raised = true
      │
      ├─ write LearningEvent { source:'checkpoint', userId, … } (per item)
      ├─ ingestQuizAttempt + computeMastery
      └─ emit checkpoint.passed, band.changed
  
  ↓ response { score, passed, skillScores:[{skill,score,passed,raised}], raisedSkills, suggestions }
```

### Track P: Progress gate — 2-step

```
buildProgress(course, enrollment, passedBoundaries, onPathModuleKeys?)
  ├─ FOR EACH phase:
  │   ├─ Step A: allPhaseLessonsCompleted (on-path only)
  │   ├─ Step B: if checkpoint exist: passedThisPhase (CHECK checkpoint_attempts, boundaryKey=phaseKey, passed=true)
  │   ├─ phaseBlocked = !step_A OR (checkpoint exists AND !step_B)
  │   └─ nextPhase.locked = phaseBlocked
  │
  └─ FOR course-level:
      ├─ courseBlocked = IF course.checkpoint: !passedCourseBoundary ELSE false
      └─ course.locked = courseBlocked
```

### Track B: Web UI — sidebar node + content panel

```
PathFocusStudyContainer
  ├─ loadProgress() → buildProgress
  ├─ FOR EACH node in sidebar:
  │   ├─ IF node.type='checkpoint':
  │   │   ├─ checkpointNodeState() → derive state (locked/available/passed)
  │   │   ├─ render CheckpointSidebarNode (3-state style)
  │   │   └─ IF selected:
  │   │       ├─ activeView = 'checkpoint'
  │   │       ├─ content panel = CheckpointPanel (wrapper)
  │   │       └─ CheckpointPanel = CheckpointInlineRunner (reuse quiz runner)
  │   │
  │   ├─ IF node.type='lesson': render LessonNode
  │   └─ …
  │
  ├─ lesson-content-panel.tsx
  │   ├─ nextLesson nav button
  │   ├─ IF nextLesson is checkpoint:
  │   │   ├─ label = "Làm bài kiểm tra chặng"
  │   │   └─ onClick = setActiveView('checkpoint')
  │   ├─ IF checkpoint available OR locked:
  │   │   ├─ nextDisabled = true, nextTooltip = "Hãy pass checkpoint trước"
  │   └─ ELSE enable
  │
  └─ NextChapterButton
      ├─ disabled if phaseBlocked (checkpoint available OR not passed)
      └─ tooltip accordingly
```

### Track S: FE result screen (both pass/fail)

```
CheckpointResult { outcome: 'pass' | 'fail', skillScores, raisedSkills, suggestions }
  ├─ IF pass:
  │   ├─ heading: "Chúc mừng! Bạn đã pass checkpoint."
  │   ├─ skillScores map: "Listening: 0.85/1.0 ✓ raised" + "Writing: 0.6/1.0 (weak not raised)"
  │   ├─ CTA: "Tiếp tục" (navigate to next phase)
  │   └─ suggestions: weak subskill links
  │
  └─ ELSE fail:
      ├─ heading: "Bạn chưa pass. Hãy thử lại."
      ├─ skillScores map + suggestions
      └─ CTA: "[Quay về lộ trình]" / "[Thi lại]"
```

## 5. Contracts

Liệt kê contract nào cần viết ở `contracts/`:
- `contracts/checkpoint-items-submit.md` — POST /courses/:slug/checkpoint/{items,submit}
- `contracts/checkpoint-band-distribution.md` — internal: StudentPath.skillBudget + band credit logic (có thể gộp vào checkpoint-items-submit nếu ngắn)

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `modules/checkpoint/checkpoint-attempt.model.js` | CheckpointAttempt schema (Mongoose) | Tạo |
| `modules/checkpoint/checkpoint.service.js` | getCheckpointItems, submitCheckpoint, grading + band-up | Tạo |
| `modules/checkpoint/checkpoint.controller.js` | HTTP handlers (GET items, POST submit) | Tạo |
| `modules/checkpoint/checkpoint.routes.js` | Route mount (POST /checkpoint/{items,submit}) | Tạo |
| `modules/checkpoint/checkpoint.graders.js` | gradeItem registry (objective → gradeObjective) | Tạo |
| `modules/checkpoint/checkpoint.verdict.js` | computeVerdict (overall + per-skill) | Tạo |
| `modules/checkpoint/checkpoint.events.js` | emit events (checkpoint.passed, band.changed) | Tạo |
| `modules/assessment/testlet-stimuli.js` | buildTestletStimuli, signAudioSafe (reusable) | Tạo |
| `modules/adaptive/student-model.model.js` | Thêm bandPoint, creditedBoundaries fields | Sửa |
| `modules/adaptive/adaptive.service.js` | applyCheckpointResult refactor (band-up logic) | Sửa |
| `modules/course-content/course-content.model.js` | Thêm CheckpointRef vào PhaseSchema, CourseStructureSchema | Sửa |
| `modules/course-content/course-content.progress.service.js` | buildProgress extend (checkpoint gate) | Sửa |
| `admin-api/resources/course-content.admin.js` | setPhaseCheckpoint action (validate + persist) | Sửa |
| `exe-web/app/services/checkpoint.service.ts` | getCheckpointItems, submitCheckpoint stubs + query keys | Tạo |
| `exe-web/app/components/checkpoint-sidebar-node.tsx` | 3-state node render (sidebar) | Tạo |
| `exe-web/app/components/checkpoint-panel.tsx` | Runner wrapper (content panel) | Tạo |
| `exe-web/app/components/checkpoint-result.tsx` | Result screen (pass/fail) | Tạo |
| `exe-web/app/pages/adaptive/history.tsx` | Competency history timeline page | Tạo |
| `exe-admin/app/pages/courses/[id]/page.tsx` | Thêm PhaseCheckpointControl (set quiz, threshold) | Sửa |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code | Ghi chú |
|---|---|---|---|
| Checkpoint không cấu hình cho boundary | 404 | CHECKPOINT_NOT_FOUND | Boundary (phase/course) không có ref |
| LessonQuiz referenced not found | 404 | CHECKPOINT_NOT_FOUND | quizCode không map → quiz |
| LessonQuiz not published | 409 | QUIZ_CONFIG_NOT_PUBLISHED | status ≠ 'published' |
| Learner not enrolled | 403 | NOT_ENROLLED | assertEnrolled fail (reuse existing code) |
| No StudentModel (pre-assessment) | 409 | NEEDS_ASSESSMENT | getBandHistory, applyCheckpointResult (reuse) |
| Invalid boundaryKey | 404 | CHECKPOINT_NOT_FOUND | resolveCheckpointRef return null |
| Stimulus missing/retired | 200 | N/A | buildTestletStimuli filter status='active', omit from stimuli[] |
| Audio sign failure (soft) | 200 | N/A | signAudioSafe catch → return null, warn log |
| Audio URL outside signable prefix | 200 | N/A | signMediaUrl !== url → return null (not served) |

## 8. Bảo mật & quyền

**Permission model:**
- `courses:read` — implicit via CourseEnrollment (assertEnrolled); learner can submit checkpoint
- `course_content:manage` — admin set checkpoint ref (PUT /api/admin/courses/:id/phase-checkpoint)

**Data protection:**
- Answer key + transcript → `select:false` trên LessonQuiz + AssessmentStimulus (never loaded)
- Audio URL → signed (signature expires TTL, KHÔNG long-term public)
- Band-change audit → append-only history (KHÔNG overwrite)

**Tenant scope:**
- Checkpoint per course; course enrollment scoped per center (existing).
- Admin action: verifyPermission enforces centerId scope.

## 9. Rủi ro / đánh đổi

| Rủi ro | Mitigated by | Residual |
|---|---|---|
| Course-side 3h audio TTL — link could leak? | Signed URL (internal key); non-signable prefix blocked (NULL); learner can't control signing | Low — signing key rotation elsewhere |
| Budget frozen at assign → live course edit mismatch | Snapshot copy; replan = rebuild snapshot | Acceptable — replan rare; non-learners see new budget after replan |
| Non-MCQ items invisible but count SAI | Constraint: checkpoint serve lọc MCQ; test must verify | Medium — ghi rõ constraint để dev test |
| MCQ-only = fewer questions → easier? | Admin control threshold (0.8 default tunable); Monitor pass rate | Acceptable — MBA handles policy |
| On-path ambiguity (non-on-path bypass?) | D1 260729: off-path bài bị bỏ khỏi `prevDone`; phaseHasOnPathModule check | Low — tested |
| Network: learner reload mid-attempt → state lost | Checkpoint state NOT persist (learner re-submit from start) | Acceptable — learner experience (UX can improve later) |

## 10. Testing strategy

**Unit tests (modules/checkpoint/*.test.js):**
- gradeItem per type (mcq → gradeObjective)
- computeVerdict (overall + per-skill, sub-score ≥ threshold)
- applyCheckpointResult (band-up, clamp, creditedBoundaries idempotent)
- buildProgress gate (2-step: on-path → available → passed)

**Integration tests:**
- submitCheckpoint full flow (resolve → grade → verdict → band-up → history)
- buildProgress with/without checkpoint
- buildTestletStimuli (filter status, sign, handle missing)

**E2E (manual browser):**
- Sidebar node (3-state transition)
- Content panel runner (same as lesson quiz)
- Result screen (pass/fail, raised skills)
- Retake idempotent (1st pass → raise; 2nd pass → no re-raise)
- Progress gate (next chapter unlock after pass)
- Competency history page timeline

## 11. Rollout / cross-repo sequencing

**Merge order** (main branch):
1. **exe-api:** checkpoint model + service (standalone, tested)
2. **exe-api:** progress.service extend (gate logic)
3. **exe-web:** checkpoint UI (sidebar + panel + result)
4. **exe-admin:** set checkpoint action

**Deploy:**
- exe-api first (handlers ready before web uses endpoints)
- exe-web next (can call api without stalling)
- exe-admin last (authoring UI nice-to-have; seed data sufficient for MVP)

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion (từ acceptance.md) | Được đáp ứng bởi (phần design này) |
|---|---|
| AC-1 Checkpoint node + 3 state | Track B (sidebar-node component + checkpointNodeState logic) |
| AC-2 2-step gate on-path | Track P (buildProgress on-path-aware) |
| AC-3 2-step gate pass → unlock | Track P (phaseBlocked check passed_checkpoint) + Track S (nextDisabled logic) |
| AC-4 Nav-lock + tooltip | Track S (lesson-content-panel nextDisabled+tooltip) |
| AC-5 Unlimited retry no penalty | Track A (best-score-wins + applyCheckpointResult idempotent) |
| AC-6 Mixed test MCQ | Track A (resolveQuizItems curated select) |
| AC-7 Pass threshold | checkpoint.service.js:77 (effectiveThreshold resolution) |
| AC-8 BandPoint 1.0–6.0 | M2 (SkillAbilitySchema.bandPoint) + Track A (credit += budget, clamp ≤ target) |
| AC-9 Idempotent credit | M2 (creditedBoundaries) + Track A (skip if boundaryKey in array) |
| AC-10 Testlet stimulus | Track A (buildTestletStimuli) + getCheckpointItems return stimuli[] |
| AC-11 Result pass | Track S (CheckpointResult outcome=pass render) |
| AC-12 Result fail | Track S (CheckpointResult outcome=fail render) |
| AC-13 Competency history | M4 (HistoryEntry extend) + Track A (log on band-up) |
| AC-14 Admin authoring | admin-api setPhaseCheckpoint + PhaseCheckpointControl |
| AC-15 Permissions | assertEnrolled (courses:read) + verifyPermission (course_content:manage) |
