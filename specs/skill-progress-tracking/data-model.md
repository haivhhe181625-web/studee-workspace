# Data model: Theo dõi tiến độ kỹ năng theo phase của lộ trình

- **Design:** `specs/skill-progress-tracking/design.md`
- **Ngày:** 2026-07-14

Feature này **không tạo collection mới**. Chỉ mở rộng 1 collection và định nghĩa 1 DTO đọc.

## 1. Thay đổi collection `learning_events` (SỬA)

File: `<API_REPO>/services/api/src/modules/adaptive/learning-event.model.js`

`LearningEvent` hiện có: `userId, source, refId, questionId, skill, subskill, itemCefr, correct, band, durationSec, countsTowardMastery` + timestamps. Thêm **3 field nullable**:

| Field | Kiểu | Default | Ràng buộc | Ghi chú |
|---|---|---|---|---|
| `score` | Number | `null` | `min:0, max:100` | Điểm chuẩn hóa 0–100 của bài luyện. `null` khi nguồn chỉ có `correct` (item objective) hoặc chưa chấm điểm số. **Chỉ field này vào "điểm trung bình".** |
| `roadmapId` | ObjectId (ref `UserRoadmap`) | `null` | — | Lộ trình active của user lúc ghi event. `null` nếu chưa có roadmap (vd event `assessment` baseline sinh trước khi roadmap tồn tại). |
| `phaseId` | Number | `null` | — | `phaseId` của phase active (`resolveCurrentPhase`) lúc ghi event. `null` ⇒ không quy về phase nào. |

### Index mới

```
{ userId: 1, roadmapId: 1, phaseId: 1, skill: 1 }
```

Phục vụ truy vấn tiến độ: luôn `$match` theo `userId` (+ `roadmapId`) rồi `$group` theo `(phaseId, skill)` — không full-collection scan (NFR). Giữ nguyên 2 index cũ (`userId+skill+createdAt`, `userId+subskill+createdAt`).

### Migration / backfill

- **Không cần migration.** Cả 3 field nullable, thêm vào schema là đủ; document cũ đọc ra `undefined/null`.
- **Không backfill** event cũ: chúng giữ `phaseId=null` (không thuộc phase nào) và `score=null` (không vào avg). Đây là hành vi đã chấp nhận ở `design.md` §11 — tiến độ tính từ thời điểm feature live trở đi.

### Ghi field ở đâu (write path)

| Nguồn | Hàm | `score` lấy từ | `roadmapId`/`phaseId` |
|---|---|---|---|
| IPA (`speaking_drill`) | `adaptive.service.ingestIpaAttempt` (SỬA) | `attempt.score` (0–100, hiện đang bị bỏ) | `resolveActivePhaseContext(userId)` |
| Assessment | `adaptive.service.buildAssessmentEvents` (không đổi ở MVP) | không set (`null`) | thường `null` (roadmap chưa tồn tại lúc seed) |
| Nguồn tương lai (course/quiz...) | khi có chấm điểm → emit event cùng dạng | điểm 0–100 của nguồn đó | `resolveActivePhaseContext` |

> `resolveActivePhaseContext(userId)` (MỚI): đọc `UserRoadmap` active + `resolveCurrentPhase()` (tái dùng từ `study-schedule.service`) → `{ roadmapId, phaseId }` hoặc `{ null, null }` nếu chưa có roadmap. Best-effort: lỗi resolve **không** được làm hỏng việc ghi event (ghi với `phaseId=null`).

## 2. DTO đọc — `SkillProgress` (không lưu DB)

Kết quả của `getSkillProgress(userId)`, trả qua `GET /api/adaptive/progress`. Chi tiết field ở `contracts/get-skill-progress.md`. Hình dạng:

```
SkillProgress {
  needsAssessment: boolean            // true → FE hiện gate làm bài test (như StudentModelCard)
  roadmap: { id, cefrLevel, activePhaseId } | null
  skills: [                           // mức hiện tại toàn cục (từ student_models.skill)
    { skill, cefr }                   // vd { "speaking", "A2" }
  ]
  phases: [
    {
      phaseId, name, status,          // từ user_roadmaps.phases[]
      practicedCount,                 // Σ event có (roadmapId, phaseId) này
      avgScore,                       // avg(score) trên event score!=null; null nếu chưa có
      skills: [
        { skill, practicedCount, avgScore, mastery }  // mastery 0..1 từ student_models (nếu có)
      ]
    }
  ]
  generatedAt
}
```

### Nguồn từng phần

| Phần DTO | Đọc từ |
|---|---|
| `needsAssessment`, gate | không có `student_model`/roadmap active → 409 (không trả DTO) |
| `roadmap`, `phases[].{phaseId,name,status}` | `user_roadmaps` (active) |
| `skills[].cefr`, `phases[].skills[].mastery` | `student_models` (`.skill` θ/cefr, `.mastery` per-subskill) |
| `phases[].practicedCount/avgScore`, `phases[].skills[].{practicedCount,avgScore}` | aggregate `learning_events` scoped `{userId, roadmapId}` |

## 3. Bất biến / ràng buộc dữ liệu

- `avgScore` **chỉ** trung bình trên event có `score != null` **và** `countsTowardMastery != false` (loại gaming — AC-E1). Không có event hợp lệ ⇒ `avgScore = null` (không phải `0` — tránh hiểu nhầm "điểm kém", AC-E2).
- `practicedCount` đếm event `countsTowardMastery != false` (kể cả `score=null`) — "đã luyện bao nhiêu lần", tách khỏi "điểm".
- Không có field nào của DTO cho phép truy ra dữ liệu user khác: mọi query khóa theo `userId` server-side.
