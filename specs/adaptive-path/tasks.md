<!-- Tiếng Việt — tasks Adaptive Path. design/data-model/contract cùng thư mục. M1 build ngay; M2 sau/song song Phase 4. -->
# Tasks: Hồ sơ năng lực & Lộ trình thích nghi

**Nhánh:** `exe-api` `feature/adaptive-competency-profile` (off `develop`); `exe-web` `feature/adaptive-profile-web`.
**Thứ tự:** M1 (T1–T4) triển khai NGAY (`StudentModel` đã ~80%). M2 (T5–T9) **sau/song song Phase 4** (Q-AP8 — cần
mastery giàu). Mỗi task: code → compile/jest → commit (Conventional, không AI-attribution).

**[SCOPE — chốt 2026-07-19]** Nguồn scope chính thức = `docs/feature-spec-ho-so-nang-luc.md` (7 khối A–G + 52 BR).
M1 mở rộng để phủ **khối A (đầy đủ), D2, E-tín-hiệu-yếu, F** (không chỉ A/B/C lát cắt cũ) — xem AP-T10. Phần cần
đo lường mới (checkpoint nâng band §4.4.5, suy giảm §4.4.4, đánh giá lại/biểu đồ tiến bộ, chủ điểm ngữ pháp cần gắn
thẻ ngân hàng đề) vẫn thuộc M2/Phase-4.

---

## M1 — Hồ sơ năng lực (buildable now)

### AP-T1 — `target` trên StudentModel + đặt/sửa (exe-api)
- `student-model.model.js`: thêm `TargetSchema` + field `target` (data-model §M1).
- `adaptive.service.js`: `resolveTarget(userId)` (manual → onboarding+program → default), `setTarget(userId, body)`
  (validate qua `programs.validateTargetScore`, lưu `source:'manual'`).
- `adaptive.schema.js`: validate body PUT /target. Controller `setTarget` + route `PUT /adaptive/target`.
- Test: general+score≠null → 422; ielts score sai step/range → 422; hợp lệ → lưu; suy mặc định khi chưa set.
- Commit: `feat(adaptive): learner competency target (program/score/cefr)`.

### AP-T2 — Hồ sơ trình bày tiếng Việt (exe-api)
- `adaptive.service.js`: `getProfileView(userId)` — gói `getStudentModel`+`getRecommendations`+`resolveTarget` thành
  shape `profile` (contract §A1): skills xếp yếu→mạnh + `gap`/`band`/`label` (tái dùng `prioritizeSkills`),
  `weakSubskills` (bỏ `weakSignal`), `weakPhonemes` deep-link (`findLessonsForPhonemes`).
- Controller `getProfile` + route `GET /adaptive/profile`. 409 `needsAssessment` khi chưa có model.
- Test: gap đúng dấu theo target; xếp yếu trước; 409 khi chưa đánh giá; weakPhonemes có route.
- Commit: `feat(adaptive): learner-facing competency profile view`.

### AP-T3 — exe-web Hồ sơ năng lực (feature learn)
- service + `useProfile`/`useSetTarget` (mutation invalidate profile); types `Profile`/`Target`.
- `ProfileCard`: mạnh/yếu từng kỹ năng (badge band), gap tới mục tiêu, điểm yếu chi tiết, âm phát âm yếu (link IPA);
  editor mục tiêu (program select + targetScore theo scale). Trạng thái chưa-đánh-giá → CTA làm đánh giá.
- Commit: `feat(learn): competency profile card + target editor`.

### AP-T4 — Verify M1
- exe-api `jest adaptive`; exe-web `tsc`+`lint`+`build`. Cập nhật spec (đánh dấu M1 xong). Báo cáo.

### AP-T10 — Đồng bộ khối A/B/E theo feature-spec (rẻ, dùng data sẵn có) — ✅ Xong 2026-07-19
- **exe-api `getProfileView`** đọc thêm `Result` (theo `lastResultId`) và phơi:
  - Khối A: `levelLabel`/`levelDescription` (§7.4), `confidence` (`Result.confidenceLabel` — BR-03),
    `confidenceInterval` (`overallCiLo/Hi`, guard `ciPlaceholder` — BR-04), `assessedAt`/`updatedAt`/`bandSource` (BR-33/34).
  - Khối B: **đủ 5 kỹ năng**, kỹ năng chưa đo → `hasData:false` "chưa có dữ liệu" (BR-06, KHÔNG quy band thấp nhất);
    thêm `ciLo/ciHi` per-skill.
  - Khối E: `weakSubskills` **giữ `weakSignal`** (gắn cờ, confirmed trước — BR-35/38, sửa correctness cũ);
    `weakTopics` từ `Result.topicBreakdown` (chẩn đoán chủ điểm, yếu nhất trước, `weakSignal` khi `total<4`).
  - **Khối F** (§3.6): `buildExamConversion` **tái dùng** `programs.scoreBandForCefr` (bảng §3.6.2 ở `cefr-mapping.js`,
    nguồn đơn BR-42). CEFR→dải điểm 1 chiều (BR-43/44) + miễn trừ (BR-13); mở dải theo CI (BR-45); biên vượt ngưỡng
    (BR-46, vd C2+TOEIC → `estimate:null`); cảnh báo lệch kỹ năng ≥2 bậc (BR-51); khoảng cách mục tiêu (`gapBands`).
    `null` khi `program='general'` (BR-14).
- **exe-web**: `adaptive.types.ts` (+`Confidence`, `WeakTopicItem`, `ExamConversion`, mở rộng `SkillGap`/`WeakSubskillItem`/
  `CompetencyProfile`); `ProfileCard` render khối trình độ (CEFR+tên VN+độ tin cậy+CI+mốc), kỹ năng chưa đo, cờ
  weakSignal, chủ điểm cần ôn, **khối quy đổi điểm thi** (dải + mục tiêu + cảnh báo + miễn trừ).
- Test: `jest adaptive-target` **25 pass** (+7: khối A từ Result, guard CI, 5 khối F); `tsc`/`eslint` 0 lỗi; `vitest
  adaptive.service` 8 pass.
- Commit: `feat(adaptive): profile confidence, CI, all-5 skills, topic diagnosis + exam-score conversion`.

**M1 còn thiếu so với feature-spec (chưa làm — batch sau):**
- Khối A: **tiến độ trong band** ("B1·70% tới B2", tầng 2 §4.4.1) — cần định nghĩa công thức từ `mastery`.
- Khối D2: giải thích CEFR **per-skill** (vì sao band này / cần gì lên band kế) — mở rộng `getCoach`.
- Khối F nâng cao: Cambridge 3-tầng (BR-47/48, data đã có ở `cefr-mapping.CAMBRIDGE_CES_TO_CEFR`), TOEFL/PTE/DET/VSTEP
  (ngoài registry `program` general/ielts/toeic hiện tại), link "Xem quy đổi tham khảo" đầy đủ (§3.6.5e).
- Trạng thái "bài không hợp lệ" (BR-01) phân biệt với "chưa đánh giá"; "đang chấm" (BR-25).

---

## M2 — Lộ trình thích nghi

> **v1 — ✅ Xong 2026-07-19** (deterministic, không AI): `adaptive-path-builder.js` (PURE) + `getAdaptivePath` +
> `GET /adaptive/path`. Bốc **khóa toàn diện** + **chặng theo kỹ năng yếu** từ pool `CourseStructure` `published` của
> program mục tiêu; join kỹ năng→category + cửa sổ CEFR `[hiện tại, min(+1, target)]`. Đã seed ladder IELTS
> (`scripts/seed-ielts-course.js`: 3.5→5.0 · 4.0→5.0 · 5.0→6.0 · 6.0→7.0) và chạy thử thật OK. Test: `adaptive-path`
> **7 suites / 61 pass** (8 unit builder + 3 integration). Chưa làm: overlay tiến độ/status trên `modules`, `regenerate`,
> lưu `AdaptivePath`, lọc mastery (Q-AP9), chèn IPA. Chi tiết shape: contract §B1.
> Commit gợi ý: `feat(adaptive): deterministic adaptive path (whole-course + per-skill modules)`.

### Chi tiết thiết kế gốc (tham chiếu)

### AP-T5 — Model + bộ bốc Chuyên đề PURE (exe-api)
- `adaptive-path.model.js` (AdaptivePath, data-model §M2).
- `path-builder.js` PURE: `SKILL_TO_CATEGORIES` (design §3), `cefrOverlap(phase, lo, hi)`, `pickModules(pool, weak,
  target)`, `orderItems` (gap desc → cefr asc → interleave). Không DB.
- Test pure: join skill→category đúng; overlap CEFR đúng; xếp thứ tự + interleave; bỏ Chuyên đề đã thành thạo.
- Commit: `feat(adaptive): adaptive-path model + pure module picker`.

### AP-T6 — Generator + tiến độ dùng chung (exe-api)
- `adaptive-path.service.js`: `generatePath(userId)` (pool published, program khớp, chèn weakPhonemes IpaLesson,
  lưu `generatedFrom`), `getPath(userId)` (lazy-enroll Phase 3 + suy `status`/`percent` từ `CourseEnrollment`),
  `ensurePath` (regen khi stale theo `lastResultId`/`recomputedAt`).
- Controller `getPath`/`regeneratePath` + routes `GET /adaptive/path`, `POST /adaptive/path/regenerate` (cooldown).
- Test: sinh đúng Chuyên đề (skill×cefr) từ pool; unlock tuần tự; item completed khi Bài Chuyên đề completed; rỗng →
  gợi ý lộ trình cố định; stale → regen.
- Commit: `feat(adaptive): generate adaptive path + shared progress`.

### AP-T7 — exe-web Lộ trình thích nghi (feature learn)
- service + `useAdaptivePath`/`useRegeneratePath`; type `AdaptivePath`.
- `PathTimeline`: item có thứ tự (khoá/mở + %), lý do bốc, điều hướng vào Bài gốc (dùng chung tiến độ); nút làm mới.
- Commit: `feat(learn): adaptive path timeline`.

### AP-T8 — Verify M2 + vòng lặp thích nghi
- Kiểm end-to-end: học Chuyên đề → progress cập nhật → `computeMastery` → `regenerate` đổi thứ tự/bớt item.
- exe-api `jest`; exe-web `tsc`+`lint`+`build`.

### AP-T9 — Chốt + docs
- Cập nhật `plan.md` (đánh dấu spec xong), PRD nghiệm thu §9. Báo cáo. Câu hỏi mở (nếu có).

---

## Phụ thuộc & lưu ý
- **M2 cần Phase 4** cho tín hiệu mastery đủ giàu (Q-AP8) — M1 độc lập, chạy trước.
- Tái dùng tối đa: `programs.validateTargetScore` (1.5), `CourseEnrollment`/`enroll` (Phase 3), `prioritizeSkills`/
  `findLessonsForPhonemes`/`ensureRoadmap` pattern (adaptive) — **không** kho tiến độ/model mới ngoài `adaptive_paths`.
- Không đụng exe-admin (Q-AP: admin chỉ gắn `category`/CEFR khi soạn khóa — đã có ở import/sửa).

---

## Kết quả M1 — ✅ Xong (chưa commit/push)

| Task | Nội dung | Repo |
|---|---|---|
| AP-T1 | `target` trên StudentModel + `resolveTarget`/`setTarget` + `PUT /adaptive/target` | exe-api `feature/adaptive-competency-profile` (off develop) |
| AP-T2 | `getProfileView` + `GET /adaptive/profile` (diễn giải tiếng Việt) | exe-api (cùng nhánh) |
| AP-T3 | types/service/hooks + `ProfileCard`/`TargetEditor` + tích hợp `AdaptiveContainer` | exe-web `feature/course-progress-web` ⚠️ (chưa tách nhánh riêng) |
| AP-T4 | Verify | — |
| AP-T10 | Khối A(độ tin cậy/CI/mốc) + B(đủ 5 kỹ năng/hasData) + E(weakSignal/weakTopics) + **F(quy đổi điểm thi)** từ `Result`/`cefr-mapping` | exe-api (cùng nhánh) + exe-web (cùng nhánh ⚠️) |

**Verify:**
- exe-api `jest adaptive-target` → **25 pass** (13 target + 12 profile: khối A/B/E + 5 khối F).
- exe-web `tsc` **0 lỗi** · `eslint` (file đổi) **0 lỗi** · `vitest adaptive.service` **8 pass**.

**Quyết định khi code (khác spec, đã đồng bộ):**
- Lỗi program/score dùng **400 BAD_REQUEST** (codebase không có 422; khớp Phase 1.5 import) — contract §A2 đã sửa.
- Band kỹ năng **target-driven**: `gap≥2 yếu · ==1 trung bình · ≤0 mạnh` (giải mâu thuẫn text/example trong contract §A1).
- exe-web mở rộng feature `adaptive` sẵn có (KHÔNG tạo feature `learn`); CEFR picker **A1–C1** (web `SelfLevel` không C2).
- Đổi program → reset `targetScore` cũ về null (điểm khác thang không so được).

**⚠️ Nợ kỹ thuật:** AP-T3 (exe-web) đang nằm trên `feature/course-progress-web`, lẫn với Phase 3 web — cần tách
`feature/adaptive-profile-web` off develop khi commit (cherry-pick) hoặc chấp nhận gộp.

**M2 (AP-T5..T9):** chưa làm — chờ Phase 4 (Q-AP8). Còn 2 câu hỏi mở Q-AP9/Q-AP10.

## Câu hỏi mở
- **Q-AP9 (ngưỡng "thành thạo" để bỏ Chuyên đề):** dùng `mastery.value ≥ ?` hay item completed đủ? Đề xuất: completed
  **hoặc** mọi subskill kỹ năng `value ≥ 0.7` & `!weakSignal`. Chốt khi vào AP-T5.
- **Q-AP10 (bước CEFR khi bốc):** chỉ `[hiện tại, +1]` hay tới thẳng `target.cefr`? Đề xuất: `[hiện tại, min(+1,
  target)]` (nền trước), lộ trình tự nối bước kế sau mỗi lần regen. Chốt khi vào AP-T5.
