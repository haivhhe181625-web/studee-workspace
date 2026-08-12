# Tasks: Cấu hình lịch — tổng giờ khóa + gợi ý + hạn tự tính

**Spec:** `./spec.md` (đã duyệt, D1–D3 chốt). Chạy **Task 1 (BE) → Task 2 (FE)**.
**Repos:** exe-api `feature/study-schedule-generation` · exe-web `feature/schedule-ui-integration`.

## Global Constraints
1. **exe-api:** English code/comment; Conventional Commits; `git add <files>`; no AI attribution; self-service JWT; coexist. Jest: `cd services/api && npx jest <p> -i`.
2. **exe-web:** theo `docs/patterns.md`; zod ở `src/utils/validation.ts`; không sửa `components/ui/`; no `any`/`@ts-ignore`. Verify: `npx tsc --noEmit` + `npm run lint` sạch; vitest.
3. Không nhắc plan-taxonomy (AC-n/IS-n/Dn) trong comment/tên test.
4. Adaptive/schedule endpoints trả JSON trần.

---

## Task 1 (exe-api) — `suggest` trả thêm tổng giờ khóa + giờ ôn
**Files:** modify `src/modules/adaptive/adaptive.controller.js` (`suggestSchedule`); test `src/__tests__/schedule-suggest.api.test.js` (bổ sung).
- [ ] Trong `suggestSchedule` (hiện tính `totalLessonMin = Σ item.durationMin`): thêm `totalReviewMin = items.reduce((s,i)=> s + (i.exercises||[]).reduce((a,e)=> a + exerciseDurationOf(e.type), 0), 0)`. Import `exerciseDurationOf` từ `./schedule/lesson-duration`.
- [ ] Response: `res.status(HTTP.OK).json({ ...suggestPlans(totalLessonMin, asOfDate, deadline), totalLessonMin, totalReviewMin })`.
- [ ] Test (RED→GREEN): seed khóa có lesson kèm exercises (biết trước tổng) → response có `totalLessonMin` + `totalReviewMin` đúng (Σ heuristic exercise: quiz 8/flashcard 5/ipa 5/talk 10); `presets` vẫn còn. Giữ các case cũ xanh.
- [ ] `npx jest schedule-suggest.api -i` xanh; regression `npx jest schedule-suggest schedule-generate.api -i` xanh.
- [ ] Commit `feat(adaptive): return total course + review minutes from schedule suggest`.

## Task 2 (exe-web) — Hiển thị tổng giờ + gợi ý + hạn tự tính, bỏ ô chọn hạn
**Files:** `src/types/schedule.types.ts` (SuggestResponse presets-branch + `totalLessonMin`/`totalReviewMin`), `src/utils/schedule.utils.ts` (nếu cần helper cộng ngày / mirror projectFinish), `src/utils/validation.ts` (bỏ `deadline` khỏi `studyPrefsSchema`), `src/components/features/schedule/SchedulePrefsForm.tsx` + `SchedulePrefsSuggestPanel.tsx` (UI), test tương ứng.
- [ ] **Types:** thêm `totalLessonMin: number` + `totalReviewMin: number` vào nhánh presets của `SuggestResponse`.
- [ ] **Bỏ hạn:** xoá field `deadline` khỏi form + `studyPrefsSchema` (zod) + payload submit (`StudyPrefsInput` bỏ `deadline`, hoặc luôn gửi `deadline` không set). Kiểm tra `useSaveStudyPrefs`/`getStudyPrefs` không vỡ.
- [ ] **Hiển thị (suggest panel / config):**
  - "Tổng thời lượng khóa: {formatDurationMin(totalLessonMin+totalReviewMin)} (gồm ~{formatDurationMin(totalReviewMin)} ôn tập)".
  - "Gợi ý: ~{formatDurationMin(canBang.learningDays*canBang.minutesPerDay)}/tuần (nhịp Cân bằng)" — lấy preset `can_bang` từ response.
  - "Hạn hoàn thành dự kiến: {finish}" — tính live: `weeks = ceil(totalLessonMin / (watch(learningDows).length * watch(focusDurationMin)))`, `finish = addDaysLocal(today, weeks*7)`. **Mirror công thức BE `projectFinish`** (comment ghi rõ giữ đồng bộ). Ẩn/placeholder khi chưa đủ input (learningDows rỗng) hoặc suggest lỗi/no-source.
- [ ] Cập nhật test: `SchedulePrefsForm.test.tsx` (không còn field hạn; submit không kèm deadline), `SchedulePrefsSuggestPanel`/hoặc panel test (hiện tổng giờ + gợi ý + hạn tính đúng cho input mẫu), `schedule.service.test.ts` nếu đổi shape. Có thể thêm unit cho helper tính finish.
- [ ] Verify `npx tsc --noEmit` + `npm run lint` sạch; `npx vitest run SchedulePrefsForm SchedulePrefs schedule.service schedule.utils` xanh.
- [ ] Commit `feat(schedule): show total course hours, weekly suggestion and auto-projected finish; drop manual deadline`.

---
## Self-Review
AC-1→AC-3 map: Task 1 · Task 2. ⬜
