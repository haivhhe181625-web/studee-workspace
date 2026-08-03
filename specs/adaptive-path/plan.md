<!-- Tiếng Việt — FRAMING (đặt vấn đề) cho "Hồ sơ năng lực + Lộ trình thích nghi". Chưa phải spec chi tiết.
     Giữ khóa cố định (CourseStructure) làm lộ trình cứng; thêm lộ trình cá nhân hóa trỏ vào cây khóa. -->
# Đặt vấn đề: Hồ sơ năng lực & Lộ trình học thích nghi

- **Ngày:** 2026-07-17
- **Tác giả:** Dev + Claude (framing)
- **Trạng thái:** Nháp phác thảo — chờ chốt §7 trước khi viết spec chi tiết
- **Ý tưởng:** Sau khi làm bài **đánh giá năng lực**, mỗi học viên có một **hồ sơ năng lực** (điểm mạnh/yếu từng kỹ
  năng). Từ hồ sơ, sinh một **lộ trình thích nghi** — thay vì chỉ lộ trình **cố định** (A1→A2, IELTS 5→6). Lộ trình
  thích nghi **bốc** các Chặng/Chuyên đề/Bài từ **khóa cố định** cho khớp điểm yếu của từng người.
- **Nguồn:** khảo sát `modules/adaptive` (StudentModel, getRecommendations), `modules/course-content`
  (CourseStructure), `modules/roadmap` (UserRoadmap).

---

## 1. Tầm nhìn & ranh giới
- **Giữ nguyên** khóa cố định (`CourseStructure` + enrollment Phase 3): vừa là **kho nội dung**, vừa là **lộ trình
  tuyến tính** cho ai muốn đi thẳng theo mức/kỳ thi.
- **Thêm** một **lớp phủ cá nhân hóa** (chỉ đọc): danh sách **có thứ tự** các node bốc từ nhiều khóa published, chọn
  theo hồ sơ. Không sửa khóa.
- Một Bài có thể xuất hiện ở **cả** lộ trình cứng lẫn thích nghi; tiến độ dùng chung tín hiệu engine.

## 2. Hồ sơ năng lực — đã có ~80% (`StudentModel`, 1 doc/user)
Sinh từ bài đánh giá (`Result`) + tinh chỉnh bởi `learning_events`.

| Thành phần | Nội dung |
|---|---|
| `level` | CEFR tổng (= `Result.overallCefr`) |
| `skill: Map` | 5 kỹ năng `reading/listening/writing/speaking/use_of_english` → `{theta, se, cefr}` — năng lực từng kỹ năng |
| `mastery: Map` | 9 subskill (gist/detail/inference/attitude/grammar_tense/collocation/word_form/pronunciation/fluency) → `{value 0..1, sampleCount, weakSignal}` — Knowledge Graph mạnh/yếu chi tiết |
| `weakPhonemes` | `[{phoneme, score}]` — phát âm yếu (đã deep-link IpaLesson) |
| `behavior`, `coach` | thói quen học + text AI coach |
| `lastResultId` | bài đánh giá gần nhất (re-tính khi thi lại) |

**Còn thiếu để "thích nghi có đích":** một **mục tiêu (target)** rõ (program + targetScore hoặc CEFR đích). Hiện chỉ
có level *hiện tại*. Cần: `gap = target − hiện tại` từng kỹ năng → **thứ tự ưu tiên**.

## 3. Cái đã có trong adaptive (tái dùng)
- `getStudentModel(userId)` → hồ sơ. `getRecommendations(userId)` → **weakSkills/targets** + `matchCourses(...)` +
  deep-link phát âm (`weakPhonemes`→IpaLesson). `computeMastery` cập nhật khi có `learning_events`.
- **Hạn chế:** `matchCourses` match model **`Course` (modules/course) cũ, coarse** — KHÔNG phải `CourseStructure`
  (khóa cố định mới); và ra **hoạt động rời rạc**, chưa phải **lộ trình có thứ tự**. → Đây là khoảng trống.

## 4. Lộ trình thích nghi — cơ chế bốc node
Cây khóa đã mang 2 chiều cần khớp: `module.category` = **kỹ năng**; `phase.cefrFrom→cefrTo` = **mức**;
`program/targetGoal` = kỳ thi. ⇒ index mọi Bài/Chuyên đề theo **(skill, cefr, program)** across khóa published.

```
gap = target − skill.cefr  (mỗi kỹ năng)
với mỗi kỹ năng yếu (ưu tiên gap lớn):
   bốc Chuyên đề/Bài có category = kỹ năng
       AND cefr ≈ mức hiện tại kỹ năng đó (hoặc bước kế)
       AND program khớp (nếu nhắm IELTS/TOEIC)
       AND chưa mastered (theo mastery/ExerciseProgress)
xếp: gap desc → cefr asc (prereq) → interleave (không nhàm 1 kỹ năng)
```
**Độ mịn (đã chốt):** đơn vị bốc là **Chuyên đề (module)** — mỗi Chuyên đề gắn **1 kỹ năng** (`category`) trong **1
Chặng** có **1 khoảng CEFR** (vd A2→B1). ⇒ "kỹ năng X ở mức A2→B1" → bốc đúng **Chuyên đề của kỹ năng X trong Chặng
A2→B1** từ mọi khóa published. Phát âm yếu → chèn IpaLesson (cơ chế đã có).

**Vòng lặp thích nghi:** học node → bài tập → `learning_events` → `mastery` cập nhật (đã có) → **re-tính lộ trình**.
⚠️ Tín hiệu hiện **mỏng** (chỉ assessment + ipa) → **Phase 4** (quiz/writing/speaking) làm giàu hẳn ⇒ lộ trình mạnh
nhất **sau/song song Phase 4**.

## 5. Data model sketch (mới, nhẹ)
```
adaptive_paths (1/user)
  userId, target{program,targetScore,cefr}, generatedFrom{resultId,masteryAt}
  items: [ { courseId, phaseKey, moduleKey, lessonKey,     # trỏ vào CourseStructure
             targetSkill, targetSubskill, reason, cefr, order,
             status[locked|unlocked|in_progress|completed] } ]
  updatedAt
```
Tái dùng: khung unlock `UserRoadmap` (locked/unlocked/completed); `matchCourses`/`getRecommendations` (đổi nguồn
`Course`→`CourseStructure`); tín hiệu `ExerciseProgress`/`ipa_progress` (cross-course).

## 6. Ngoài phạm vi (phác thảo này)
Sinh nội dung bằng AI; đề xuất lịch học chi tiết; A/B thuật toán chọn; đa mục tiêu song song. → sau.

## 7. Quyết định đã chốt (theo đề xuất — 2026-07-17)
- **Q-AP1 (target):** mục tiêu lấy từ **onboarding goal + `program` khóa**, cho **nhập tay**. ✅
- **Q-AP2 (độ mịn):** đơn vị bốc là **Chuyên đề (module)** — 1 kỹ năng × 1 khoảng CEFR. ✅
- **Q-AP3 (pool):** bốc từ **mọi khóa published** (pool đa khóa), lọc theo program khi có mục tiêu kỳ thi. ✅
- **Q-AP4 (index):** **aggregate on-the-fly** trước; thêm index (skill,cefr,program) nếu chậm. ✅
- **Q-AP5 (model):** **KHÔNG tạo model `AdaptivePath`** — path tính **on-the-fly** mỗi lần đọc. Output v3:
  courses→phases→{modules,allModules}. ✅
- **Q-AP6 (hợp nhất course):** nguồn lộ trình = **`CourseStructure`** (khóa cố định published). ✅
- **Q-AP7 (chia sẻ tiến độ):** **dùng chung tín hiệu engine** (CourseEnrollment cross-course); enrollment
  khóa cố định để riêng. ✅
- **Q-AP8 (thứ tự):** làm **sau/song song Phase 4** (cần tín hiệu mastery đủ giàu). ✅
- **Q-AP9 (mastery-skip):** **đã chốt làm** — skip kỹ năng gap≥1 nhưng mastery≥0.8 (ngưỡng thạo). Lọc ở **mức kỹ
  năng** (chưa gắn thẻ chủ điểm — track sau). Thêm `readyForCheckpoint`. ✅

## 8. Bước tiếp theo
1. Chốt §7 (đặc biệt Q-AP2/Q-AP5/Q-AP6/Q-AP8).
2. (Sau Phase 4) viết spec `specs/adaptive-path/{design,data-model,contracts,tasks}.md`.
3. Triển khai: index nội dung theo (skill,cefr) → generator lộ trình → vòng lặp re-tính → UI exe-web.
