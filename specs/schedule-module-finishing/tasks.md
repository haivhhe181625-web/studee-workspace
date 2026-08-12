# Tasks: Hoàn thiện module Lịch học

**Spec:** `./spec.md` (đã duyệt, Q1a–Q4a chốt). **Chạy tuần tự Task 1→5** (Task 2 FE phụ thuộc Task 1 BE).
**Repos:** exe-api `feature/study-schedule-generation` · exe-web `feature/schedule-ui-integration`.

## Global Constraints
1. **exe-api:** English code/comment; Conventional Commits; `git add <files>` (không `git add .`); no Co-Authored-By/AI attribution; self-service JWT (`req.user.id`), no verifyPermission; coexist (không đụng `study-schedule.service` cũ). Test: `cd services/api && npx jest <p> -i`.
2. **exe-web:** theo `docs/patterns.md` (service→hook→query-keys→component; TanStack Query; `apiClient` auto-JWT); zod ở `src/utils/validation.ts`; KHÔNG sửa `src/components/ui/`; không logic trong `page.tsx`; no `any`/`@ts-ignore`. Verify: `npx tsc --noEmit` + `npm run lint` sạch; test vitest.
3. Adaptive/schedule endpoints trả **JSON trần** (không bọc `ApiResponse<T>`).
4. KHÔNG mở scope slice sau (e2e/M4-4/M4-5/deep-link/timeSlots/per-roadmap).
5. **Plan-taxonomy:** comment code + tên test KHÔNG nhắc `AC-n`/`IS-n`/`Qn` — mô tả hành vi.

---

## Task 1 (exe-api) — `GET /adaptive/schedule/prefs`
**Files:** modify `src/modules/adaptive/adaptive.controller.js`, `src/modules/adaptive/adaptive.routes.js`; test `src/__tests__/schedule-prefs-get.api.test.js`.
- [ ] **RED:** viết test trước.
- [ ] Controller `getStudyPrefs`: `const u = await User.findById(req.user.id).select('profile.studyPlan').lean(); res.status(HTTP.OK).json({ studyPlan: u?.profile?.studyPlan ?? null });`. Read-only, no LLM. Export handler.
- [ ] Route: `router.get('/schedule/prefs', asyncHandler(controller.getStudyPrefs));` — trong block `verifyToken, withTenant`; cùng path `/schedule/prefs` với PUT (khác method → OK). KHÔNG mở lại block comment cũ.
- [ ] Test (supertest + memory DB): user có studyPlan → 200 `{studyPlan}` khớp field đã seed; user chưa có → `{studyPlan:null}`; no JWT → 401.
- [ ] **GREEN:** `npx jest schedule-prefs-get -i` xanh; regression `npx jest schedule-prefs-save study-schedule.service -i` xanh.
- [ ] Commit `feat(adaptive): add GET /schedule/prefs to read current study preferences`.

## Task 2 (exe-web) — Đọc + sửa prefs trong UI
**Files:** `src/services/schedule.service.ts` (+`getStudyPrefs`), `src/hooks/use-schedule.ts` (+`useStudyPrefs`; `useSaveStudyPrefs` invalidate thêm prefs), `src/constants/query-keys.ts` (+`schedule.prefs`), `src/components/features/schedule/*` (thẻ tóm tắt prefs + prefill form), `src/types/schedule.types.ts` (nếu cần).
- [ ] `getStudyPrefs(): Promise<StudyPlan | null>` (GET `/adaptive/schedule/prefs`, trả `data.studyPlan`).
- [ ] `useStudyPrefs()` → `useQuery` (auth-gated như `use-adaptive.ts`), key `QUERY_KEYS.schedule.prefs`.
- [ ] `ScheduleContainer`: nút "Chỉnh sửa lịch học" → `SchedulePrefsForm` với `defaultValues` = prefs từ `useStudyPrefs`. Thẻ tóm tắt prefs nhỏ (thứ học/ôn · phút/ngày · hạn) từ dữ liệu này.
- [ ] `useSaveStudyPrefs` `onSuccess`: invalidate cả `schedule.all` lẫn `schedule.prefs`.
- [ ] Verify `npx tsc --noEmit` + `npm run lint` sạch; thêm case `getStudyPrefs` vào `schedule.service.test.ts`.
- [ ] Commit `feat(schedule): read and edit saved study preferences with prefill and summary`.

## Task 3 (exe-web) — Unit `schedule.utils.ts`
**Files:** test `src/utils/schedule.utils.test.ts`.
- [ ] Test (RED→GREEN): `groupDaysByWeek` gom đúng tuần ISO (nhiều tuần + tuần lẻ); `mondayOf` đúng; `todayLocalDateString`/`parseLocalDate` **không lệch ngày** (ngày biên, so Y/M/D local, không UTC); `itemTypeLabel` map đúng + fallback type lạ.
- [ ] Verify `npx vitest run schedule.utils` xanh; tsc/lint sạch.
- [ ] Commit `test(schedule): cover week-grouping and date/label utils`.

## Task 4 (exe-web) — Component `ScheduleContainer`
**Files:** test `src/components/features/schedule/ScheduleContainer.test.tsx`.
- [ ] Test (RTL, mock `useSchedule`/`useStudyPrefs`): (1) loading → skeleton; (2) error chung → thông báo; (3) 409 `SCHEDULE_NO_SOURCE` → empty-state + CTA; (4) `needsPrefs:true` → render form; (5) `schedule` → summary (số buổi/tổng phút/ngày xong) + warnings theo type + ngày có badge `Ôn tập`/`Nặng`.
- [ ] Verify vitest xanh; tsc/lint sạch.
- [ ] Commit `test(schedule): cover ScheduleContainer render branches`.

## Task 5 (exe-web) — Bổ sung test `SchedulePrefsForm`
**Files:** sửa `src/components/features/schedule/SchedulePrefsForm.test.tsx`.
- [ ] Thêm test: click preset `can_bang` → set `learningDows=[1,2,4,5]`, `reviewDow=6`, `focusDurationMin=60`; panel suggest (mock `useScheduleSuggest` trả presets) hiển thị "Dự kiến xong: …"; refine reject khi `reviewDow ≤ max(learningDows)`.
- [ ] Verify vitest xanh; tsc/lint sạch.
- [ ] Commit `test(schedule): cover preset quick-fill and suggestion preview`.

---
## Self-Review (điền khi xong)
AC-1→AC-5 map: Task 1 · Task 2 · Task 3 · Task 4 · Task 5. ⬜
