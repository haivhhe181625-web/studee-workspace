# Contract: Lấy tiến độ kỹ năng theo phase

- **Loại:** Cross-repo (api↔web)
- **Bên cung cấp (provider):** `exe-api` (module `adaptive`)
- **Bên tiêu thụ (consumer):** `exe-web` (section "Tiến độ kỹ năng" trong `/adaptive`)
- **Trạng thái:** Nháp — chờ Technical Review

## Endpoint

```
GET /api/adaptive/progress
```

- **Auth:** JWT user (`verifyToken` + `withTenant` — áp sẵn ở `adaptive.routes` level)
- **Permission required:** không (user-scoped, ownership qua `req.user.id`)

## Request

Không có body/query param. `userId` lấy từ token — **không** nhận từ client.

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| — | — | — | (không có input) |

## Response — thành công (200)

```json
{
  "roadmap": { "id": "665...", "cefrLevel": "A1-A2", "activePhaseId": 1 },
  "skills": [
    { "skill": "speaking", "cefr": "A2" },
    { "skill": "reading",  "cefr": "A2" }
  ],
  "phases": [
    {
      "phaseId": 1,
      "name": "Nền tảng phát âm & từ vựng cơ bản",
      "status": "unlocked",
      "practicedCount": 12,
      "avgScore": 72,
      "skills": [
        { "skill": "speaking", "practicedCount": 12, "avgScore": 72, "mastery": 0.61 }
      ]
    },
    {
      "phaseId": 2,
      "name": "Giao tiếp tình huống",
      "status": "locked",
      "practicedCount": 0,
      "avgScore": null,
      "skills": []
    }
  ],
  "generatedAt": "2026-07-14T10:00:00.000Z"
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `roadmap.id` | string (ObjectId) | Lộ trình active |
| `roadmap.cefrLevel` | string | Group CEFR (`A1-A2`/`B1-B2`/`C1+`) |
| `roadmap.activePhaseId` | number | Phase đang active (`resolveCurrentPhase`) |
| `skills[]` | array | Mức CEFR hiện tại mỗi kỹ năng (từ `student_models.skill`) |
| `skills[].skill` | string | ∈ reading/listening/writing/speaking/use_of_english |
| `skills[].cefr` | string \| null | CEFR hiện tại của kỹ năng |
| `phases[]` | array | Theo thứ tự phase của lộ trình |
| `phases[].phaseId` | number | |
| `phases[].name` | string | Tên phase |
| `phases[].status` | string | `locked`/`unlocked`/`in_progress`/`completed` |
| `phases[].practicedCount` | number | Số bài đã luyện thuộc phase (không tính gaming) |
| `phases[].avgScore` | number \| null | Điểm TB 0–100 của phase; **`null` nếu chưa có điểm** (không phải 0) |
| `phases[].skills[]` | array | Chỉ gồm kỹ năng đã có hoạt động trong phase |
| `phases[].skills[].mastery` | number \| null | 0..1 từ `student_models` (nếu có) |
| `generatedAt` | string (ISO) | Thời điểm tính |

> **Quy ước hiển thị (FE):** `avgScore = null` → render "Chưa có dữ liệu", **không** hiển thị `0`. Không có `%` hoàn thành phase (phase không có mẫu số — xem `design.md` §2 Q#2).

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 401 | `NOT_AUTHENTICATED` | Thiếu/sai token |
| 409 | `NEEDS_ASSESSMENT` | Chưa có `student_model`/roadmap active (chưa làm bài test) — FE hiện gate làm test, dùng lại pattern `StudentModelCard` |

*(Có roadmap nhưng chưa luyện bài nào **không** phải lỗi → 200 với `phases[].practicedCount=0`, `avgScore=null`.)*

## Versioning / breaking change

N/A — contract mới, endpoint mới. Không đụng response của các route `adaptive` hiện có.

## Sequencing triển khai (cross-repo)

`exe-api` deploy trước (endpoint chạy độc lập). `exe-web` deploy section tiến độ sau. Xem `.ai/workflows/release-workflow.md` §3.

## Ví dụ gọi thực tế

```bash
curl -X GET http://localhost:5050/api/adaptive/progress \
  -H "Authorization: Bearer <token>"
```
