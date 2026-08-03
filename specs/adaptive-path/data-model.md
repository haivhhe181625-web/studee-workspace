<!-- Tiếng Việt — data-model Adaptive Path + Hồ sơ năng lực. Grounded: student-model.model.js, course-content.model.js,
     course-content.progress.model.js, roadmap.model.js, constants/programs.js. -->
# Data model: Hồ sơ năng lực & Lộ trình thích nghi

## M1 — Thêm `target` vào `StudentModel` (`modules/adaptive/student-model.model.js`)
Không collection mới cho hồ sơ — chỉ thêm 1 subdoc. Mọi trường còn lại **giữ nguyên**.

```js
TargetSchema { _id:false }
  program:     enum PROGRAM_KEYS,  default 'general'   // constants/programs.js
  targetScore: Number, default null                    // native unit (band/points); null cho 'general'
  cefr:        enum CEFR_LEVELS, default null           // CEFR đích (spine nội bộ)
  source:      enum ['onboarding','course','manual'], default 'onboarding'
  updatedAt:   Date, default null

// thêm field:
StudentModelSchema.target: { type: TargetSchema, default: () => ({}) }
```
- **Suy mặc định (resolveTarget):** ưu tiên `manual` đã lưu → else onboarding goal (`User.profile.mainGoal`) + `program`
  của khóa đang enroll → else `{program:'general', cefr: bước kế của level}`. Validate `targetScore` bằng
  `programs.validateTargetScore(program, targetScore)` (tái dùng 1.5 — general phải null; scaled: null hoặc đúng
  min/max/step).
- **gap** không lưu — tính khi đọc: `gap[skill] = cefrNum(target.cefr) − cefrNum(skill.cefr)`.

## M2 — Lộ trình thích nghi: tính **on-the-fly** (KHÔNG collection `adaptive_paths`)
**Q-AP5:** Lộ trình tính mỗi lần đọc `GET /path` (pure service, KHÔNG persist). Không tạo model `AdaptivePath` hay
collection `adaptive_paths`. Tái dùng `CourseStructure` (cây khóa publish sẵn) + `StudentModel` (hồ sơ) → builder
deterministic → danh sách **courses:[{ phases:[{ modules, allModules }] }]** (v3).

**Output shape (v3 — khớp adaptive-path-builder.js):**
```
{
  target, level, needsProgram,
  courses: [{
    slug, title, program, targetScore, cefrFrom, cefrTo,
    neededSkills, fitsLevel,
    totalLessons, customLessons, estimatedDays, fullEstimatedDays,
    phases: [{
      key, title, order, cefrFrom, cefrTo, goalNote,
      modules:    [...skill/category modules — custom path],
      allModules: [...tất cả modules — full path],
      lessonCount, totalLessons, estimatedDays
    }]
  }]
}
```

**Chọn khóa:** course `ceiling ∈ (band yếu nhất, target]` (xem design.md §3).
**Chọn chuyên đề:** kỹ năng yếu (`gap≥1`) + chưa mastery, nhưng **bỏ qua đã giỏi (mastery≥0.8)** (FR-B2 Q-AP9).

**Chỉ mục nội dung (Q-AP4):** aggregate on-the-fly từ `course_structures`. Thêm index `(category, cefrFrom, cefrTo)`
**chỉ khi** đo thấy chậm — không thêm sớm (YAGNI).

## M3 — Band History: MỞ RỘNG `LearnerProfile.history` (không tạo bảng mới)
**Hiện tại:** `LearnerProfile.history` chỉ snapshot từ assessment (`{assessmentId, attemptId, overallCefr, theta, takenAt}`).
**Mở rộng:** log band-change append-only, ghi tất cả nguồn (assessment + checkpoint + ops).

**Thêm vào `HistoryEntrySchema` (giữ nguyên field cũ):**
```js
sourceGroup: { type:String, enum:['assessment','checkpoint','skill_retest','ops_recompute','ops_revoke','ops_recalibrate'], default:'assessment' }
bandSource:  { type:String, enum:['assessment','checkpoint','skill_retest','recompute'], default:'assessment' }  // → profile.bandSource
resultId:    { type:Schema.Types.ObjectId, ref:'Result', default:null }
skillCefr:   { type:Map, of:String, default:undefined }  // snapshot band từng kỹ năng {listening:'A2',…}
changedSkills: { type:[String], default:[] }             // kỹ năng đổi band vs entry trước
actor:       { kind:{type:String,enum:['learner','system','admin'],default:'learner'}, id:{ObjectId} }
reason:      { type:String, default:null }
notifiedLearner: { type:Boolean, default:false }        // BR-12: phải báo khi band công bố bị đổi
```
- **before/after:** append-only — so entry hiện tại với entry liền trước cùng `userId` (tránh lưu "from" riêng).
- **Phục vụ:** Block A "Nguồn cập nhật" + §5.5 so sánh + biểu đồ tiến bộ + audit khi ops can thiệp.

## M4 — Mastery-skip & Checkpoint Dismiss
**Mastery-skip (FR-B2, Q-AP9):** kỹ năng coi là "đã giỏi" **mức kỹ năng** (granularity hiện tại) ⇔ tất cả subskill của
kỹ năng có `value≥0.8` và `weakSignal=false`. Builder chỉ bốc module kỹ năng **không mastery** → skip học lại (lọc ở
mức kỹ năng, không mức bài/chủ điểm — cần content-tagging trong track sau).

**Checkpoint Dismiss:** điều kiện "không hối thúc" (BR-30) → thêm field:
```js
StudentModelSchema.checkpointDismissed: { type:[String], default:[] }  // skill slugs đã dismiss lời mời checkpoint
```
Eligibility sau này trừ các skill đã dismiss.

## M5 — StudentPath: Snapshot lộ trình ĐÃ GÁN (IMMUTABLE với band-up)

<!-- 2026-08-03: bổ sung StudentPath snapshot (tách assignment khỏi live replan) -->

**Vấn đề gốc:** `getAdaptivePath` (on-the-fly) tính lại lộ trình mỗi request → skill đã mastery (≥0.8) bị gạt khỏi module → chặng rớt module → biến mất khỏi bản đồ + phòng học sau band-up.

**Giải pháp:** Model mới `StudentPath` (collection `student_paths`, file `student-path.model.js`): snapshot MỘT LẦN cấu trúc **ĐÃ GÁN** (courses→chặng→modules gán + lessonKeys + checkpoint ref + target), IMMUTABLE với band-up. `getAdaptivePath` đọc **cấu trúc từ snapshot**, overlay mastered flag (StudentModel) + progress (CourseEnrollment) → FE hiện "đã đạt" thay vì ẩn. Re-plan là **HÀNH ĐỘNG TƯỜNG MINH** (đổi target nút "Cập nhật lộ trình") — archive + tạo snapshot mới.

### Tách bạch DB: 4 stores

| Store | Model | Nội dung | Thay đổi | Quyền sở hữu |
|-------|-------|----------|---------|-------------|
| **CONTENT** (shared) | `CourseStructure` | course→phase→module→lesson + checkpoint ref | không (publish) | admin |
| **ASSIGNMENT** (per learner) | `StudentPath` (mới) | snapshot chặng/module ĐÃ GÁN + target | re-plan thủ công | system → learner trigger |
| **PROGRESS** (per learner) | `CourseEnrollment` | trạng thái lesson (Map pathKey) | học → update | system (lesson hoàn thành) |
| **KNOWLEDGE** (per learner) | `StudentModel` | θ/cefr/mastery/target/band history | band-up, assessment | system (assessment, checkpoint) |

### Schema `StudentPath` (collection `student_paths`)
```js
{
  userId: ObjectId(User),
  isActive: Boolean [partial unique: true when isActive=true],
  assignedAt: Date,
  archivedAt: Date|null,
  sourceResultId: ObjectId(Result)|null,  // Result sinh snapshot (phát hiện retake)
  target: {
    program: enum PROGRAM_KEYS,
    targetScore: Number|null,
    cefr: String|null
  },
  courses: [{
    slug, title, program, targetScore, order,
    cefrFrom, cefrTo,
    checkpoint: { quizCode, passThreshold, label }|null,  // khóa boundary
    skillTarget: Map<skill, bandPoint>,  // frozen target per-skill tại gán
    phases: [{
      key, title, order, cefrFrom, cefrTo, goalNote,
      checkpoint: { quizCode, passThreshold, label }|null,
      skillBudget: Map<skill, bandPoint>,  // frozen gain distribute từng chặng
      modules: [{
        moduleKey, category, skill, moduleTitle,
        lessonKeys: [String],  // join key → CourseEnrollment.pathKey
        lessonSummary: [{ key, title, type }]
      }]
    }]
  }]
}
```

**Chi tiết:**
- **`skillTarget`, `skillBudget`** (band-point): frozen assign-time để band-up (đổi StudentModel chỉ) không thay đổi cấu trúc. `skillTarget[S]` = min(course.cefrTo, learner.target) (course-level, 1 target cho mọi chặng). `skillBudget[S]` per chặng = gain / # chặng dạy S (distribute đều).
- **`lessonKeys`** PHẢI snapshot: là khóa nối với `CourseEnrollment.pathKey` (`phaseKey/moduleKey/lessonKey`) để overlay progress.
- **`checkpoint` (phase, course)** PHẢI snapshot: gate chặng/khóa đọc từ đây; nếu admin đổi quizCode sau, learner đã gán không lệch.
- **1 user = 1 active**: partial unique index `{userId, isActive}` như `UserRoadmap` (giữ history via `archivedAt`).
- **`allModules`** KHÔNG lưu: tính live từ `CourseStructure` ở read time (Phase 02) — cho phép content edit phản chiếu, DRY.

### Service `student-path.service.js`
**Hàm chính:**
- `buildSnapshotFromPath(pathDto, {startBandPoints, targetBandPoint})` — PURE: map DTO → shape snapshot + compute `skillBudget`.
- `createStudentPath(userId, {sourceResultId, pathDto, ...})` — insert active (idempotent: E11000 → trả bản hiện có).
- `replaceStudentPath(userId, {...})` — archive active + create mới (atomic: updateMany isActive=false rồi insert; race-safe).
- `getActiveStudentPath(userId)` — lean read bản active (nguồn cho Phase 02 `getAdaptivePath`).

**Gọi từ `adaptive.service.js`:**
- Lần đầu đọc path: `getAdaptivePath` → lazy-create snapshot via `createStudentPath`.
- Thay target / nút "Cập nhật lộ trình": `replanStudentPath()` → `replaceStudentPath()` → `getAdaptivePath()`.

## Không đổi
`learning_events`, `mastery`, `weakPhonemes`, `coach`, `CourseEnrollment`, `UserRoadmap` — giữ nguyên. Hồ sơ tự cập
nhật qua `seedFromResult`/`ingestIpaAttempt`/`computeMastery` sẵn có (FR-A3).
