# Tasks: Luồng học hợp nhất v1

> ✅ **ĐÃ DUYỆT — owner "clear, mặc định hết" (2026-08-05).** D1–D5 chốt; chấp nhận 2 giới hạn: (a) cổng cứng cơ bản đã có sẵn ở study-space — v1 chỉ surface checkpoint, không build gating; (b) deep-link mức **module** (không tới từng lesson). Ngoài phạm vi: M4-10 remediation · tick per-exercise ôn · talk completion · gating/route mới.
>
> **▶ RESUME (sau khi clear context) — bắt đầu code ngay:** không cần lịch sử chat. Chạy `/spec-workflow --code --resume development unified-study-flow` **hoặc** `/superpowers execute studee-workspace/specs/unified-study-flow/tasks.md`. Đọc `./spec.md` + `./design.md` là đủ ngữ cảnh; scout đã xong (deep-link routes, `useCourseProgress`, checkpoint endpoints ghi trong design §5–6). Implementer đọc file thật khi làm.
> **Branch (bắt đầu từ HEAD hiện tại — verify bằng `git rev-parse HEAD`):** exe-api `feature/study-schedule-generation` · exe-web `feature/schedule-ui-integration` (cùng nhánh chuỗi lịch học hiện có, đều local).

**Spec:** `./spec.md` · **Design:** `./design.md` (đã duyệt). Chạy **BE (1→2) rồi FE (3→5)**.
**Repos:** exe-api `feature/study-schedule-generation` · exe-web `feature/schedule-ui-integration`.

## Global Constraints
1. **exe-api:** English code/comment; Conventional Commits; `git add <files>`; self-service JWT; coexist; **KHÔNG đụng `buildProgress`/study-space (chỉ đọc)**. Jest `cd services/api && npx jest <p> -i`.
2. **exe-web:** theo `docs/patterns.md`; không sửa `components/ui/`; no `any`/`@ts-ignore`. Verify `npx tsc --noEmit` + `npm run lint` sạch; vitest.
3. Không nhắc plan-taxonomy (AC-n/IS-n/Dn) trong comment/tên test. Endpoints trả JSON trần.
4. Không mở scope: M4-10 remediation · tick per-exercise ôn · gating mới · route mới · deep-link mức lesson.

---

## Task 1 (exe-api) — Enrich item với context đường dẫn (KEYSTONE)
**Files:** `schedule/lesson-list.resolver.js`, `lesson-schedule.model.js`, `schedule/lesson-schedule.generator.js`; tests tương ứng.
- [ ] `lesson-schedule.model.js`: `days[].items` thêm `kind` (default `'lesson'`), `courseSlug`, `phaseKey`, `moduleKey` (optional để back-compat doc cũ).
- [ ] Resolver `buildLessonItem`: emit thêm `courseSlug, phaseKey, moduleKey` (đã có khi duyệt — ngừng drop). Áp cho cả nhánh StudentPath lẫn CourseStructure.
- [ ] Generator: mang `courseSlug/phaseKey/moduleKey` qua item học; `buildReviewDay` gắn `courseSlug/phaseKey/moduleKey/lessonKey` cho mỗi exercise item (từ lesson gốc trong tuần).
- [ ] Tests: resolver item có đủ context; generateSchedule item học + item ôn có context; back-compat (item cũ không kind vẫn đọc được).
- [ ] Verify `npx jest lesson-list.resolver schedule-generator lesson-schedule.model -i` xanh (cập nhật assertion cũ). Commit `feat(adaptive): carry course/phase/module context on schedule items`.

## Task 2 (exe-api) — Checkpoint thành nhiệm vụ trong lịch
**Files:** `schedule/lesson-list.resolver.js`, `schedule/lesson-schedule.generator.js`, `lesson-schedule.model.js`; tests.
- [ ] Model: item hỗ trợ `kind:'checkpoint'` + `boundaryKey`.
- [ ] Resolver: sau lesson cuối mỗi phase, nếu `phase.checkpoint` → emit marker `{ kind:'checkpoint', boundaryKey: phase.key, courseSlug, phaseKey: phase.key, moduleKey: <module cuối phase>, title: checkpoint.label ?? 'Vượt chặng' }`. Cuối course: `course.checkpoint` → `boundaryKey:'course'`.
- [ ] Generator: band-filter chỉ áp `kind==='lesson'`; checkpoint marker **độc chiếm 1 ngày** (`kind:'checkpoint'`, `durationMin` = hằng ~30, không oversized/heavy), đặt đúng vị trí path. Determinism giữ nguyên (vị trí suy từ path+asOfDate).
- [ ] Tests: phase có checkpoint → đúng 1 item checkpoint sau bài cuối phase, đúng `boundaryKey`; phase không có → không; course.checkpoint cuối khóa; determinism (cùng input→cùng output).
- [ ] Verify `npx jest lesson-list.resolver schedule-generator -i` xanh. Commit `feat(adaptive): insert phase/course checkpoint tasks into the schedule`.

## Task 3 (exe-web) — Item bấm được + deep-link
**Files:** `types/schedule.types.ts` (item +`kind`/`courseSlug`/`phaseKey`/`moduleKey`/`boundaryKey`), `utils/schedule.utils.ts` (hàm `scheduleItemHref(item)`), `components/features/schedule/ScheduleDayCard.tsx`; tests.
- [ ] Types: cập nhật `ScheduleItem` theo model BE.
- [ ] `scheduleItemHref(item)`: lesson/review → `ROUTES.ADAPTIVE_PATH_STUDY(courseSlug)?phase=&module=`; checkpoint → `+&checkpoint=1`; thiếu context → `null` (không bấm được). Talk (`type==='talk'`) → fallback/disable.
- [ ] `ScheduleDayCard`: item có href → render `<Link>`; không có → `<span>` (như cũ). Giữ badge oversized/review; thêm badge "Vượt chặng" cho `kind:'checkpoint'`.
- [ ] Tests unit `scheduleItemHref` (3 kind + thiếu context + talk); component item render link đúng.
- [ ] Verify tsc+lint sạch; `npx vitest run schedule.utils ScheduleDayCard SchedulePrefs` xanh. Commit `feat(schedule): deep-link schedule items into the study space`.

## Task 4 (exe-web) — Tick tiến độ từ `useCourseProgress`
**Files:** `hooks/use-schedule.ts` hoặc container, `ScheduleDayCard.tsx`, tests.
- [ ] Container/hook: theo `courseSlug` của item, đọc `useCourseProgress(slug)` → map done: lesson theo `"{phase}/{module}/{lesson}"` status ∈ {completed,mastered}; checkpoint theo `boundaryKey ∈ passedCheckpoints`; locked theo status `'locked'`.
- [ ] `ScheduleDayCard`: hiển thị trạng thái **xong** (check) / **khóa** (mờ + không bấm) / **chưa** cho item. Buổi ôn: không tick (D3).
- [ ] Tests: mock `useCourseProgress` → item done/locked/chưa render đúng; checkpoint done theo passedCheckpoints.
- [ ] Verify tsc+lint; vitest xanh. Commit `feat(schedule): reflect lesson/checkpoint completion on schedule items`.

## Task 5 (exe-web) — Bề mặt "Nhiệm vụ hôm nay"
**Files:** `ScheduleContainer.tsx` (nâng khối "hôm nay"), tests.
- [ ] Nâng khối "Kế hoạch hôm nay": liệt kê item hôm nay **bấm được + trạng thái** (dùng Task 3/4), làm CTA chính; nếu hôm nay không có việc → trạng thái rỗng thân thiện.
- [ ] Tests: container render "hôm nay" với item bấm được + trạng thái (mock schedule + progress).
- [ ] Verify tsc+lint; `npx vitest run ScheduleContainer` xanh. Commit `feat(schedule): today's tasks surface as the guided entry point`.

---
## Self-Review
AC-1→AC-5 map: Task 1 · Task 2 · Task 3 · Task 4 · Task 5. ⬜
