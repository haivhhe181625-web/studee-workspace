<!-- Tiếng Việt — thiết kế Adaptive Path + Hồ sơ năng lực. Grounded: modules/adaptive, course-content, roadmap.
     Framing: specs/adaptive-path/plan.md (§7 đã chốt). PRD: docs/prd-adaptive-path.md. Contract/Data cùng thư mục. -->
# Thiết kế: Hồ sơ năng lực & Lộ trình thích nghi

- **Ranh giới:** `exe-api` module `adaptive` (mở rộng, KHÔNG tạo module mới) + đọc `course-content`; `exe-web` feature
  `learn`. Không đụng exe-admin (admin chỉ gắn `category`/CEFR khi soạn khóa — đã có).
- **Quyết định:** đã chốt §7 `plan.md` (Q-AP1..8). **Contract:** `contracts/adaptive-path.md` · **Data:** `data-model.md`.
- **Hai lát cắt:** **M1 — Hồ sơ năng lực** (build ngay, `StudentModel` đã ~80%); **M2 — Lộ trình thích nghi**
  (sau/song song Phase 4, cần tín hiệu mastery giàu — Q-AP8). Tài liệu này scope cả hai; tasks tách M1 trước.

## 1. Nguyên tắc: tái dùng, không đo lại
Hồ sơ = `StudentModel` (1 doc/user, seed từ `Result` + `learning_events`) — **không** dựng model mới. Lộ trình thích
nghi = **lớp phủ chỉ-đọc** trỏ vào `CourseStructure`; tiến độ **dùng chung** cơ chế Phase 3 (`CourseEnrollment`),
không kho tiến độ mới. Chấm điểm vẫn ở engine (ipa/Phase 4) — adaptive chỉ **đọc** tín hiệu.

## 2. M1 — Hồ sơ năng lực (buildable now)
`StudentModel` đã có: `level`, `skill{theta,se,cefr}`, `mastery{value,sampleCount,weakSignal}`, `weakPhonemes`,
`coach`. **Thiếu 2 mảnh** để "thích nghi có đích":
1. **`target`** — mục tiêu `{program, targetScore, cefr}` (Q-AP1). Mặc định suy từ onboarding goal + `program` khóa
   đang theo; cho **nhập tay**. Validate `targetScore` bằng `constants/programs.validateTargetScore` (tái dùng 1.5).
2. **Trình bày dễ hiểu (FR-A2)** — endpoint learner gói `StudentModel` + `getRecommendations` thành hồ sơ tiếng
   Việt: mạnh/yếu từng kỹ năng, `gap = target.cefr − skill.cefr`, điểm yếu chi tiết (mastery thấp, bỏ `weakSignal`),
   âm phát âm yếu (deep-link IpaLesson — cơ chế `findLessonsForPhonemes` đã có).

Không tính lại năng lực ở M1 — chỉ **đọc + diễn giải**. Tự cập nhật (FR-A3) đã chạy qua `seedFromResult` (thi lại) +
`ingestIpaAttempt`/`computeMastery` (luyện tập) — không cần thêm.

## 3. M2 — Lộ trình thích nghi (v3): **courses → phases → {modules, allModules}**
**Cơ chế:** cây khóa đã có `{ course:{ phases:[ module:[] ] } }`. Builder tính:
1. **Chọn khóa:** ceiling (max cefrTo) ∈ (band yếu nhất, target] → gạt khóa đã qua + vượt đích.
2. **Chọn chuyên đề (kỹ năng yếu):** per khóa, tính `neededSkills` (kỹ năng `gap≥1` chưa mastery, bỏ đã giỏi ≥0.8).
3. **Lọc module theo skill:** mỗi phase → 2 danh sách:
   - `modules` = custom (chỉ kỹ năng cần + hết bài trong module)
   - `allModules` = full (mọi module của phase)
4. **Khóa đầu** lộ trình (`fitsLevel:true`) = điểm vào, chứ không theo level tổng.

**Join skill → category:**
| Skill | Category |
|---|---|
| reading | reading |
| listening | listening |
| writing | writing |
| speaking | speaking, pronunciation |
| use_of_english | grammar, vocabulary |

**Q-AP4:** aggregate on-the-fly từ `course_structures`. Index `(category,cefrFrom,cefrTo)` **chỉ khi** chậm.

## 4. Tiến độ dùng chung (Q-AP7) & Mastery-skip (Q-AP9)
**Tiến độ:** mỗi module = 1 Chuyên đề trong `courseId`. Hoàn thành ⇔ mọi Bài `completed` trong `CourseEnrollment`.
Đọc path → lazy-enroll → suy status từ enrollment.lessons.

**Mastery-skip (FR-B2):** kỹ năng `gap≥1` **nhưng** `mastery{value≥0.8, weakSignal=false} mọi subskill` → bỏ module,
mời checkpoint thay vì học lại (QA-AP9). Chỉ bốc module kỹ năng **chưa giỏi**.

**Học trong lộ trình = học chính Bài** → tiến độ dùng chung, không duplicate (FR-B6).

## 5. Unlock (tái dùng khung PHASE_STATUS)
Trạng thái item ∈ `locked|unlocked|in_progress|completed` (đã có ở `roadmap.model` PHASE_STATUS). Tính động khi đọc
(không lưu, như Phase 3): `unlocked[i] = i==0 || items[i-1].completed`. Sắp nền-trước-nâng-cao qua thứ tự §3.

## 6. Vòng lặp thích nghi (FR-B4)
Học → bài tập → `learning_events` → `computeMastery` cập nhật `StudentModel` (đã có) → **re-tính lộ trình**. Dựa
`generatedFrom{resultId, masteryAt}`: lộ trình cũ so `lastResultId`/`recomputedAt` → stale ⇒ regen (mẫu `ensureRoadmap`
đã có). ⚠️ Tín hiệu hiện mỏng (assessment + ipa) → mạnh nhất **sau Phase 4** (quiz/writing/speaking) — Q-AP8.

## 7. StudentPath: Snapshot ĐÃ GÁN + Overlay Live Knowledge (tách assignment khỏi replan)

<!-- 2026-08-03: bổ sung snapshot để chặng đã gán KHÔNG biến mất sau band-up -->

**Quyết định:** Persist `StudentPath` (snapshot cấu trúc ĐÃ GÁN) khác với live `getAdaptivePath` (on-the-fly). Tách bạch:
- **Snapshot (StudentPath):** courses→chặng→module gán + lessonKeys + checkpoint ref + frozen `skillTarget`/`skillBudget` (assign-time band ceilings). IMMUTABLE → band-up (đổi StudentModel chỉ).
- **Live overlay (getAdaptivePath):** đọc snapshot cấu trúc + overlay `mastered` flag (StudentModel) + `allModules` (CourseStructure) + progress (CourseEnrollment).
- **`mastered` flag:** skill đạt target band-point được frozen lúc gán → không bị loại module. FE hiện "đã đạt", không ẩn chặng.

**Data flow:**
```
Lần đầu: CourseEnrollment đăng ký khóa
         → buildAdaptivePath (xây path mới)
         → createStudentPath (snapshot gán) [lazy-create on first getAdaptivePath read]
         → getAdaptivePath (đọc snapshot + overlay live mastery+progress)

Band-up: checkpointPass → applyCheckpointResult (cập StudentModel: skill.cefr, mastery) [KHÔNG đụng snapshot]
         → getAdaptivePath (re-read snapshot + overlay NEW mastery → "đã đạt" hiện)

Re-plan (explicit): setTarget hoặc nút "Cập nhật lộ trình"
                   → replanStudentPath
                   → replaceStudentPath (archive cũ + tạo snapshot mới) [COMMIT re-plan]
                   → getAdaptivePath (đọc snapshot KHÁC)
```

**Re-plan trigger (explicit):**
1. `PUT /adaptive/target` (đổi target) → auto replan (xem contracts §A2)
2. Nút "Cập nhật lộ trình" (FE: `PathMapContainer.tsx`) → POST `/adaptive/path/replan`
3. **KHÔNG auto** khi band-up hoặc khóa mới → snapshot giữ tối đa (chỉ mastered thay đổi → flag)

**Không thay đổi:**
- Band-up chỉ cập StudentModel (skill.cefr, mastery, band history) → snapshot giữ nguyên → chặng đã gán vẫn hiện.
- `CourseEnrollment` progress dùng `pathKey` từ snapshot (single source of truth on-path — `student-path.service.js` + `course-content.progress.service.js:164-181`).
- Content edit không ảnh hưởng snapshot; `allModules` live derived từ CourseStructure.

---

## 8. File (Phạm vi Phase 1)
```
exe-api modules/adaptive/ (mở rộng):
  student-model.model.js      + TargetSchema {program,targetScore,cefr}; +checkpointDismissed:[]
  adaptive.service.js         + resolveTarget, setTarget, getProfileView (M1)
                              + getAdaptivePath (gọi builder + truyền mastery) (M2)
  adaptive-path-builder.js    PURE: SKILL_TO_CATEGORIES, buildAdaptivePath{...masteredSkills}
  adaptive.controller.js      + getProfile, setTarget (M1); getPath (M2); getBandHistory
  adaptive.routes.js          + GET /profile, PUT /target (M1); GET /path (M2); GET /band-history
  adaptive.schema.js (sửa)    setTarget validate

modules/assessment/learner-profile.model.js
  HistoryEntrySchema          + sourceGroup, bandSource, resultId, skillCefr, changedSkills, actor, reason, notifiedLearner

exe-web feature learn:
  services/adaptive.service.ts   + getProfile, setTarget (M1); getAdaptivePath (M2); getBandHistory
  hooks/use-adaptive.ts          + useProfile, useSetTarget (M1); useAdaptivePath (M2)
  types/adaptive.types.ts        + Profile, Target, AdaptivePath, BandHistoryEntry
  components/features/learn/     ProfileCard (M1); PathTimeline (M2)
```

**Ghi chú M2:** `adaptive_paths` (collection) **không tạo** — path tính on-the-fly.
`adaptive-path.model.js`, `adaptive-path.service.js` **không cần** (v3 on-the-fly).

**Ghi chú M5 (snapshot):** Đã thêm (Phase 01–05 xong):
```
exe-api modules/adaptive/ (mở rộng):
  student-path.model.js       StudentPath schema + partial unique index (userId, isActive)
  student-path.service.js     build/create/replace/get snapshot + compute skillBudget
  adaptive.service.js         + replanStudentPath, getAdaptivePath (read snapshot + overlay)
  adaptive.controller.js      + replanPath (POST /path/replan)
  adaptive.routes.js          + POST /path/replan

__tests__/:
  student-path.service.test.js          build/create/replace/idempotent
  student-path.integration.service.test.js   end-to-end (create→archive→create)
  student-path.onpath-invariant.service.test.js   onPathModuleKeys single source
  student-path.budget.service.test.js    skillBudget frozen
  student-path.read.service.test.js      getAdaptivePath overlay

scripts/:
  backfill-student-paths.js    lazy-create snapshot cho user cũ
```

## 9. Ngoài phạm vi (→ sau)
Sinh nội dung/câu hỏi bằng AI; lập lịch ngày-giờ; đa mục tiêu song song; A/B thuật toán chọn; ghi/sửa khóa cố định
(chỉ đọc-bốc, NFR-4).
