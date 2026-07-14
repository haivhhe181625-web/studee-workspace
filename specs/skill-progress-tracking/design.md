# Design: Theo dõi tiến độ kỹ năng theo phase của lộ trình

- **Spec:** `specs/skill-progress-tracking/spec.md`
- **Ngày:** 2026-07-14
- **Tác giả:** Tech Lead Agent
- **Trạng thái:** Chờ Technical Review
- **ADR liên quan:** `docs/adr/0001-learning-events-as-progress-source.md` (draft — tạo cùng design này)

> ⚠️ **3 câu hỏi mở của spec được chốt tạm ở §2 theo hướng best-practice** (user yêu cầu "cứ thiết kế chuẩn").
> Các quyết định này cần **Technical Review xác nhận** trước khi sang task — chúng đổi *dữ liệu hiển thị*, không đổi kiến trúc.

## 1. Tóm tắt kiến trúc

Feature này **mở rộng module `adaptive`** (không tạo module mới) — vì toàn bộ hạ tầng liên quan đã ở đó: `learning_events` (log bài đã chấm), `student_models` (năng lực đã tổng hợp), `study-schedule.service` (đã có `resolveCurrentPhase`). Tạo module `practice` mới là không cần thiết và trái nguyên tắc "ưu tiên mở rộng module có sẵn".

Ý tưởng cốt lõi (best-practice, mở-đóng để mở rộng): **biến `learning_events` thành tín hiệu "bài luyện đã chấm" phổ quát**, rồi xây một **tầng đọc tiến độ generic** đọc từ nó. Cụ thể:

1. **Ghi (write path):** mỗi bài luyện đã chấm ghi 1 `learning_event` kèm **điểm chuẩn hóa 0–100** và **ngữ cảnh phase** (`roadmapId` + `phaseId`) tại thời điểm học. Hôm nay chỉ IPA phát event luyện tập thật → wiring IPA trước; nguồn khác (course...) khi có chấm điểm chỉ cần emit event cùng dạng là **tự động** lên màn tiến độ, không phải sửa tầng đọc.
2. **Đọc (read path):** service mới `getSkillProgress(userId)` tổng hợp `learning_events` (scoped theo `userId`+`roadmapId`) theo `(phaseId × skill)`, ghép với `student_models` để có "mức hiện tại", trả DTO cho FE.
3. **Hiển thị:** thêm 1 `<section>` "Tiến độ kỹ năng" vào `AdaptiveContainer` (exe-web), theo đúng pattern card hiện có (`StudentModelCard` đã vẽ thanh tiến độ kỹ năng — tái dùng phong cách).

Không mâu thuẫn `<API_REPO>/docs/ARCHITECTURE.md`: giữ REST, user-scoped qua JWT, 5-file module pattern, không thêm dependency/service ngoài.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Module | Mở rộng `adaptive`, **không** tạo `practice` | Toàn bộ dữ liệu (events, student_model, roadmap resolver) đã ở `adaptive` | `adaptive` service phình thêm ~1 luồng đọc |
| Nguồn tiến độ | Tái dùng `learning_events` (append-only) + `student_models`, **không** tạo log mới | 1 source-of-truth; tránh 2 con số mâu thuẫn cho cùng kỹ năng | Phải bổ sung field vào `learning_events` |
| **[chốt Q#3]** Chuẩn hóa điểm | Thêm field `score` (0–100) ghi tại **write time**; IPA sẵn 0–100 map thẳng | "Điểm trung bình" giữa các loại bài chỉ có nghĩa khi cùng thang | Item chỉ có `correct` (không điểm số) → `score=null`, không vào avg |
| **[chốt Q#1]** Loại nội dung tính MVP | Mọi nguồn **đã emit `learning_event`**: assessment (baseline) + IPA (luyện speaking). Course **loại** (chưa có chấm điểm/route) | Trung thực với hiện trạng; kiến trúc mở để nguồn mới tự vào | MVP thực tế phủ chủ yếu speaking/pronunciation + baseline placement |
| **[chốt Q#2]** Đo tiến độ phase | Per-skill: **số bài đã luyện + điểm TB (0–100) + CEFR hiện tại**. **Không** hiển thị `%` | Phase không có danh sách bài cố định → không có mẫu số (đã verify `roadmap.model.js`) | Không có thanh "% hoàn thành" — dùng count + điểm thay thế |
| Gắn event ↔ phase | Ghi `phaseId` = phase active (`resolveCurrentPhase`) lúc write, **không** suy ra theo thời gian | Vòng đời phase (`startedAt`) chưa được set → suy theo thời gian không tin cậy | Event cũ (trước feature) `phaseId=null` — không quy về phase (chấp nhận) |
| Tính toán đọc | Aggregation MongoDB **scoped theo `userId`+`roadmapId`** (có index), theo tiền lệ `computeMastery` | Đáp ứng NFR "không full-collection scan"; đơn giản hơn materialize | Tính lúc đọc; nếu sau này nặng → materialize (ngoài phạm vi) |

## 3. Data model

Chi tiết đầy đủ: `data-model.md`. Tóm tắt thay đổi:

**`learning_events`** (`<API_REPO>/services/api/src/modules/adaptive/learning-event.model.js`) — thêm 3 field **nullable** (backward-compat, không cần migration):

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `score` | Number (0–100) | Không (default null) | Điểm chuẩn hóa của bài luyện; null nếu nguồn chỉ có `correct` |
| `roadmapId` | ObjectId ref `UserRoadmap` | Không (default null) | Lộ trình active lúc ghi event |
| `phaseId` | Number | Không (default null) | `phaseId` của phase active lúc ghi event |

+ Index tổng hợp mới: `{ userId: 1, roadmapId: 1, phaseId: 1, skill: 1 }` (phục vụ aggregate tiến độ scoped theo user).

Không tạo collection mới. `student_models`, `user_roadmaps` **không đổi schema** (chỉ đọc).

## 4. Luồng dữ liệu

**Write (khi học viên luyện IPA xong — nguồn thật hôm nay):**
```
ipa.service.recordIpaAttempt()  (đã có)
  -> adaptiveService.ingestIpaAttempt(attempt)   (SỬA)
       -> resolveActivePhaseContext(userId)  → { roadmapId, phaseId }   (MỚI, reuse resolveCurrentPhase)
       -> LearningEvent.create({ ...cũ, score: attempt.score(0-100), roadmapId, phaseId })
       -> computeMastery(userId)  (đã có, không đổi)
```

**Read (khi học viên mở màn Tiến độ):**
```
GET /api/adaptive/progress   (MỚI)
  router.use(verifyToken, withTenant)     (đã có, áp cho mọi route adaptive)
  -> asyncHandler(adaptiveController.getSkillProgress)   (MỚI, mỏng — lấy req.user.id)
       -> adaptiveService.getSkillProgress(userId)        (MỚI)
            -> UserRoadmap.findOne({ userId, isActive:true })      (không có → 409 NEEDS_ASSESSMENT)
            -> StudentModel.findOne({ userId })                    (mức CEFR/mastery hiện tại)
            -> LearningEvent.aggregate([$match {userId, roadmapId}, $group by (phaseId, skill):
                 count, avgScore = avg(score where score!=null && countsTowardMastery)])
            -> ghép roadmap.phases + aggregate + student_model → DTO
  -> res.json(toDto)   (không leak field nội bộ)
```

## 5. Contracts

- `contracts/get-skill-progress.md` — `GET /api/adaptive/progress` (**cross-repo api↔web**). Bắt buộc viết trước khi task FE/BE chạy (rule Tech Lead: không để contract ngầm trong task).

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `<API_REPO>/services/api/src/modules/adaptive/learning-event.model.js` | + `score`, `roadmapId`, `phaseId` + index | Sửa |
| `<API_REPO>/services/api/src/modules/adaptive/adaptive.service.js` | + `resolveActivePhaseContext()`, sửa `ingestIpaAttempt` (ghi score+phase), + `getSkillProgress()` | Sửa |
| `<API_REPO>/services/api/src/modules/adaptive/adaptive.controller.js` | + `getSkillProgress` (mỏng) | Sửa |
| `<API_REPO>/services/api/src/modules/adaptive/adaptive.routes.js` | + `GET /progress` | Sửa |
| `<API_REPO>/services/api/src/__tests__/adaptive.progress.service.test.js` | Unit test aggregate | Tạo |
| `<API_REPO>/services/api/src/__tests__/adaptive.progress.api.test.js` | API test endpoint | Tạo |
| `<WEB_REPO>/src/services/adaptive.service.ts` | + `getSkillProgress()` | Sửa |
| `<WEB_REPO>/src/hooks/use-adaptive.ts` | + `useSkillProgress()` | Sửa |
| `<WEB_REPO>/src/constants/query-keys.ts` | + `adaptive.progress()` key | Sửa |
| `<WEB_REPO>/src/types/adaptive.types.ts` | + type `SkillProgress` | Sửa |
| `<WEB_REPO>/src/components/features/adaptive/SkillProgressCard.tsx` | Card tiến độ theo phase×skill | Tạo |
| `<WEB_REPO>/src/components/features/adaptive/index.ts` | export card mới | Sửa |
| `<WEB_REPO>/src/components/features/adaptive/AdaptiveContainer.tsx` | render `<SkillProgressCard>` như 1 section | Sửa |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| Thiếu/sai token | 401 | `NOT_AUTHENTICATED` |
| Chưa có `student_model`/roadmap active (chưa làm assessment) | 409 | `NEEDS_ASSESSMENT` (dùng lại pattern có sẵn của adaptive) |
| Có roadmap nhưng chưa luyện bài nào | 200 | — (trả cấu trúc rỗng: `phases[].practicedCount=0`, `avgScore=null`) — FE render EmptyState |

## 8. Bảo mật & quyền

- **Không có permission string mới.** Endpoint user-scoped như mọi route `adaptive`: `router.use(verifyToken, withTenant)`, ownership ngầm qua `req.user.id` — **không** dùng `verifyPermission` (đúng pattern hiện tại).
- Service **chỉ** query theo `userId = req.user.id`; không nhận userId từ request (chống xem tiến độ người khác — AC-4).
- Không có field nhạy cảm; DTO chỉ chứa số liệu tiến độ của chính user. Không đụng `admin-api/` (feature thuần user-facing).

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Giới hạn phủ nội dung (đã biết, chấp nhận):** hôm nay chỉ IPA emit event luyện tập → per-phase progress chủ yếu là speaking/pronunciation; các skill khác chỉ có mốc baseline từ assessment. Đây là *giới hạn dữ liệu*, không phải lỗi thiết kế — tầng đọc đã generic, nguồn mới vào là tự hiển thị.
- **Phase progression chưa tồn tại (đã verify):** `startedAt/completedAt/in_progress` chưa bao giờ được set khi học → `resolveCurrentPhase` luôn trả phase unlocked đầu tiên, nên **thực tế mọi event luyện tập sẽ rơi vào cùng 1 phase** cho tới khi có logic thăng phase (feature riêng). Cấu trúc per-phase vẫn đúng và sẽ tự đầy khi progression được xây. **Không** kéo phase-progression vào MVP này (cần định nghĩa "hoàn thành phase" = chính là mẫu số mà Q#2 đã xác định là chưa có).
- Không có câu hỏi kỹ thuật cần `research.md` — mọi thứ đã verify trong code.

## 10. Testing strategy

- **Unit (Jest + mongodb-memory-server)** cho `getSkillProgress`: seed `learning_events` (có/không `score`, có/không `countsTowardMastery`, nhiều phase/skill) + `user_roadmaps` + `student_models`, assert đúng count/avgScore/loại trừ gaming (AC-2, AC-3, AC-E1) và cấu trúc rỗng (AC-E2).
- **Unit** cho `ingestIpaAttempt`: assert event ghi kèm `score` + `phaseId`/`roadmapId` (AC-1).
- **API (supertest)** cho `GET /api/adaptive/progress`: 401 khi thiếu token, isolation user A không thấy dữ liệu user B (AC-4), 409 khi chưa assessment (AC-E3), 200 happy path.
- **FE (vitest)** cho `adaptive.service.getSkillProgress` (mock axios) + smoke test render `SkillProgressCard` với data rỗng/đủ.
- Chi tiết case cụ thể nằm trong `tasks.md` (viết test-first).

## 11. Rollout / cross-repo sequencing

1. **`exe-api` trước:** schema `learning_events` (nullable → deploy an toàn, không migration) → `ingestIpaAttempt` ghi score+phase → endpoint `GET /progress`. Deploy độc lập, không phá gì.
2. **`exe-web` sau:** service + hook + card. Thiếu UI không ảnh hưởng API.
3. **Backfill:** không cần. Event cũ giữ `phaseId/score = null` (không quy về phase, không vào avg) — đã tính trong thiết kế.
4. Không đụng `exe-admin`. Theo `.ai/workflows/release-workflow.md` §3.

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion (từ `acceptance.md`) | Được đáp ứng bởi |
|---|---|
| AC-1 (ghi hoàn thành + điểm) | §4 write path + §3 field `score`/`phaseId`/`roadmapId`; `ingestIpaAttempt` (SỬA) |
| AC-2 (tiến độ theo kỹ năng của phase) | §4 read path — aggregate `(phaseId × skill)`: count + avgScore + CEFR |
| AC-3 (tiến độ theo từng phase) | §4 read path — DTO `phases[]` ghép `roadmap.phases`; contract §Response |
| AC-4 (chỉ xem của mình) | §8 — query theo `req.user.id`, không nhận userId từ request |
| AC-E1 (loại bài gaming/không điểm khỏi avg) | §4 — `$match countsTowardMastery` + `avg(score where score!=null)` |
| AC-E2 (trạng thái rỗng) | §7 — 200 + cấu trúc rỗng; FE EmptyState |
| AC-E3 (chưa có roadmap active) | §7 — 409 `NEEDS_ASSESSMENT`; FE gate như `StudentModelCard` hiện tại |
| NFR bảo mật/hiệu năng | §8 (user-scoped) + §2 (aggregate scoped theo index) |
