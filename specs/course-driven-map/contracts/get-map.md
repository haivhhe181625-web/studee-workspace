# Contract: getMap — Learner-facing Game Map

- **Loại:** Nội bộ module (learning-map → adaptive surface)
- **Bên cung cấp (provider):** `exe-api` (learning-map.service)
- **Bên tiêu thụ (consumer):** `exe-web` (AdaptiveMapContainer), internal calls
- **Trạng thái:** Live (spec hồi tố)

## Endpoint

```
GET /api/adaptive/map
```

- **Auth:** JWT learner token (verifyToken)
- **Permission required:** Không (self-only; userId từ JWT)

## Request

Không có body. Query params (tuỳ chọn):

```json
{}
```

| Param | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `includeConfig` | Boolean | | default false; true = lộ config (test only, require permission) |

## Response — thành công

```json
{
  "mapId": "507f1f77bcf86cd799439011",
  "empty": false,
  "needsAssessment": false,
  "nodes": [
    {
      "nodeKey": "ielts-5-6:P1:M1:L1",
      "type": "CONTENT",
      "title": "Reading Strategies",
      "order": 0,
      "description": "Learn key strategies for IELTS reading",
      "config": {
        "theory": "# Strategies\n...",
        "media": [
          {
            "type": "video",
            "url": "https://youtube.com/...",
            "label": "Expert Video",
            "order": 0
          }
        ]
      },
      "gating": {
        "type": "SOFT",
        "prereqs": []
      },
      "publishedAt": "2026-08-03T10:00:00Z"
    },
    {
      "nodeKey": "ielts-5-6:P1:M1:L2",
      "type": "MCQ",
      "title": "Reading Comprehension Quiz 1",
      "order": 1,
      "config": {
        "itemIds": ["Q1","Q2","Q3"],
        "passThreshold": 0.6,
        "media": []
      },
      "gating": {
        "type": "SOFT",
        "prereqs": ["ielts-5-6:P1:M1:L1"]
      },
      "publishedAt": "2026-08-03T10:00:00Z"
    }
  ],
  "nodeStates": [
    {
      "nodeKey": "ielts-5-6:P1:M1:L1",
      "status": "unlocked",
      "attempt": 0,
      "startedAt": "2026-08-03T11:00:00Z",
      "completedAt": null,
      "score": null,
      "passed": null
    },
    {
      "nodeKey": "ielts-5-6:P1:M1:L2",
      "status": "locked",
      "attempt": 0,
      "startedAt": null,
      "completedAt": null,
      "score": null,
      "passed": null
    }
  ],
  "currentSegmentIndex": 0,
  "segments": [
    {
      "courseSlug": "ielts-5-6",
      "courseMapVersion": 1,
      "status": "active",
      "nodeCount": 15,
      "completedNodeCount": 1
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `mapId` | String (ObjectId) | CourseMapTemplate._id; client dùng cho /api/adaptive/map/node/:nodeKey/start |
| `empty` | Boolean | true = no active path or no template (demo archive fallback); false = có nodes |
| `needsAssessment` | Boolean | true = no StudentModel; precondition placement (CTA: làm bài kiểm tra) |
| `nodes[]` | Array | flat node list từ CourseMapTemplate; **config.theory/itemIds/deckRefId KHÔNG lộ nếu node status=LOCKED** (anti-leak) |
| `nodes[].nodeKey` | String | immutable lesson.uid; dùng cho start/submit endpoint |
| `nodes[].type` | String | 'CONTENT', 'MCQ', 'FLASHCARD', 'VIDEO' |
| `nodes[].config` | Object | type-specific; theory only if CONTENT; itemIds only if MCQ + UNLOCKED |
| `nodes[].gating` | Object | {type: 'SOFT'/'HARD', prereqs: [nodeKey list], minMastery: number\|null} |
| `nodeStates[]` | Array | progress per node; order matches nodes[] |
| `nodeStates[].status` | String | 'unlocked', 'locked', 'in_progress', 'completed', 'failed' |
| `currentSegmentIndex` | Number | index vào segments[]; nếu multi-segment roadmap, render segment này |
| `segments[]` | Array | course segments enrolled; metadata cho roadmap multi-segment rendering |

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 409 | NEEDS_ASSESSMENT | no StudentModel; client route tới placement + return {empty:true, needsAssessment:true} |
| 404 | LEARNER_PATH_NOT_FOUND | (rare) LearnerPath không tạo được; fallback empty |
| 500 | RESOLVER_ERROR | template resolver fail; log + return {empty:true} |
| 401 | UNAUTHORIZED | no JWT or token invalid |

## Thêm ghi chú

### Anti-leak (Bảo mật)

- **Config strip**: nếu node status='LOCKED' → KHÔNG trả theory, itemIds, deckRefId trong response (client không biết nội dung chưa mở).
- **Verify**: roadmap-preview test `JSON.stringify(response)` chứa 0 "theory" / "itemIds" / "deckRefId" fields.

### Caching & Freshness

- **Server-side cache**: LearnerPath + nodeStates cache; invalidate khi submit node (applyResult).
- **Client-side cache**: React Query / TanStack Query; revalidate on mount + 5 min stale.
- **Reconcile**: ensureActivePath() auto-heal nếu segment missing (concurrent enroll lag).

### Dual-Progress Note

- Response là **game-map view** (LearnerPath.nodeStates).
- Learner progress trên course-detail endpoint (CourseEnrollment) = separate document.
- Sync = deferred (V4); current document limitation (separate tracking systems).

### Empty-state CTA

```json
{
  "empty": true,
  "needsAssessment": false,
  "nodes": [],
  "nodeStates": [],
  "segments": []
}
```

- FE render "Ghi danh khóa học" button → POST /api/courses/:slug/enroll
- Nếu needsAssessment=true → CTA "Làm bài kiểm tra năng lực" → /assessment/intake

---

## Versioning / Breaking Change

**N/A — contract mới** (spec hồi tố, endpoint live).

---

## Sequencing triển khai

1. **BE endpoint** live + test (unit: getMap + reconcile; integration: enroll→getMap).
2. **FE AdaptiveMapContainer** merge (hook use-adaptive-map::useGetMap).
3. **Route** `/adaptive` → AdaptiveMapContainer.

---

## Ví dụ gọi thực tế

```bash
# Learner authenticated (JWT in header)
curl -X GET http://localhost:3000/api/adaptive/map \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json"

# Response (success)
{
  "mapId": "507f1f77bcf86cd799439011",
  "empty": false,
  "needsAssessment": false,
  "nodes": [...],
  "nodeStates": [...],
  "currentSegmentIndex": 0
}

# Response (empty — learner mới)
{
  "empty": true,
  "needsAssessment": false,
  "nodes": [],
  "nodeStates": [],
  "segments": []
}

# Response (pre-placement)
{
  "empty": true,
  "needsAssessment": true,
  "nodes": [],
  "nodeStates": [],
  "segments": []
}
```
