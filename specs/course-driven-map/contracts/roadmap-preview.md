# Contract: roadmapPreview — Pre-placement Explorer (Read-only)

- **Loại:** Cross-repo (exe-api learner → exe-web explorer UI)
- **Bên cung cấp (provider):** `exe-api` (learning-map.service, roadmap-preview endpoint)
- **Bên tiêu thụ (consumer):** `exe-web` (RoadmapExplorer component)
- **Trạng thái:** Live (spec hồi tố)

## Endpoint

```
GET /api/adaptive/map/roadmap-preview
```

- **Auth:** JWT learner token (verifyToken)
- **Permission required:** Không (learner self-only; no StudentModel gate)

## Request

Query params (bắt buộc):

```json
{
  "program": "ielts",
  "fromCefr": "A1",
  "toCefr": "B2"
}
```

| Param | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `program` | String enum | ✓ | 'ielts', 'toeic', 'efset', ... (từ PROGRAM_KEYS) |
| `fromCefr` | String enum | ✓ | ['A1','A2','B1','B2','C1','C2'] (start band learner chọn) |
| `toCefr` | String enum | ✓ | ['A1','A2','B1','B2','C1','C2'] (target band); phải ≥ fromCefr |

## Response — thành công

```json
{
  "program": "ielts",
  "fromCefr": "A1",
  "toCefr": "B2",
  "courses": [
    {
      "slug": "ielts-1-2",
      "title": "IELTS 1.0–2.0",
      "program": "ielts",
      "targetScore": null,
      "bandRange": ["A1", "A1"],
      "phases": [
        {
          "key": "P1",
          "title": "Foundation",
          "order": 0,
          "modules": [
            {
              "key": "M1",
              "title": "Listening Basics",
              "order": 0,
              "nodes": [
                {
                  "key": "L1",
                  "title": "Sound Recognition",
                  "activityType": "CONTENT"
                },
                {
                  "key": "L2",
                  "title": "Quiz: Listening #1",
                  "activityType": "MCQ"
                }
              ]
            },
            {
              "key": "M2",
              "title": "Reading Intro",
              "order": 1,
              "nodes": [
                {
                  "key": "L3",
                  "title": "Reading Strategies",
                  "activityType": "CONTENT"
                }
              ]
            }
          ]
        },
        {
          "key": "P2",
          "title": "Advanced",
          "order": 1,
          "modules": [ ... ]
        }
      ]
    },
    {
      "slug": "ielts-3-5",
      "title": "IELTS 3.0–5.0",
      "bandRange": ["A2", "B1"],
      "phases": [ ... ]
    }
  ]
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `courses[]` | Array | CourseStructure{program, cefrFrom≤fromCefr, cefrTo≥toCefr}; order by bandRange ascending |
| `courses[].slug` | String | unique identifier cho "Ghi danh khóa học" link |
| `courses[].title` | String | course title |
| `courses[].bandRange` | [String, String] | [cefrFrom, cefrTo] of first + last phase (thông tin) |
| `courses[].phases[]` | Array | phân tầng course (NOT flattened; tree structure) |
| `phases[].key` | String | phase.key (identifier) |
| `phases[].title` | String | phase.title |
| `phases[].order` | Number | thứ tự |
| `phases[].modules[]` | Array | chuyên đề trong phase |
| `modules[].key` | String | module.key |
| `modules[].title` | String | module.title |
| `modules[].order` | Number | thứ tự |
| `modules[].nodes[]` | Array | node tóm tắt (KHÔNG config; chỉ metadata) |
| `nodes[].key` | String | lesson.key (identifier tuỳ chọn; để tham chiếu trong UI) |
| `nodes[].title` | String | lesson.title |
| `nodes[].activityType` | String | 'CONTENT' (theory), 'MCQ', 'FLASHCARD', 'VIDEO' |

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | INVALID_QUERY | fromCefr/toCefr ∉ CEFR_LEVELS; fromCefr > toCefr |
| 404 | NO_COURSES_FOUND | không có course matching band range + program |
| 401 | UNAUTHORIZED | no JWT or invalid token |

## Anti-leak guarantee

**Sensitive fields KHÔNG trả về**:
- `lesson.theory` (text nội dung)
- `lesson.exercises[].refId` (quiz ID)
- `lesson.config.itemIds` (exam question list)
- `lesson.config.deckRefId` (flashcard deck ID)
- `lesson.media[].url` (video/audio link)

**Verify**: test: `JSON.stringify(response)` KHÔNG chứa keys ["theory","refId","itemIds","deckRefId"]; values KHÔNG chứa URL http(s).

## Versioning / Breaking Change

**N/A — contract mới** (spec hồi tố).

---

## Sequencing triển khai

1. **BE service** `getRoadmapPreview()` + endpoint route.
2. **FE hook** `use-adaptive-map::useRoadmapPreview()` + RoadmapExplorer component.
3. **Wire** AdaptiveMapContainer pre-placement branch (no StudentModel) → render RoadmapExplorer.

---

## Ví dụ gọi thực tế

```bash
# Learner chưa có StudentModel; duyệt IELTS A1→B2 courses
curl -X GET "http://localhost:3000/api/adaptive/map/roadmap-preview?program=ielts&fromCefr=A1&toCefr=B2" \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json"

# Response
{
  "program": "ielts",
  "fromCefr": "A1",
  "toCefr": "B2",
  "courses": [
    {
      "slug": "ielts-1-2",
      "title": "IELTS 1.0–2.0",
      "program": "ielts",
      "bandRange": ["A1", "A1"],
      "phases": [
        {
          "title": "Foundation",
          "modules": [
            {
              "title": "Listening",
              "nodes": [
                {
                  "title": "Sound Recognition",
                  "activityType": "CONTENT"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}

# Request error: invalid CEFR
curl -X GET "http://localhost:3000/api/adaptive/map/roadmap-preview?program=ielts&fromCefr=X1&toCefr=B2" \
  -H "Authorization: Bearer <JWT>"

# Response 400
{
  "code": "INVALID_QUERY",
  "message": "fromCefr must be a valid CEFR level"
}
```

---

## Usage Context (FE)

```typescript
// AdaptiveMapContainer.tsx
const { data: preview } = useRoadmapPreview({
  program: selectedProgram,
  fromCefr: selectedFromCefr,
  toCefr: selectedToCefr,
  enabled: !studentModel // only fetch pre-placement
});

if (!studentModel) {
  return <RoadmapExplorer courses={preview?.courses || []} />;
}
```

---

## Notes

- **Band range filter**: course.phases[] cefrFrom ≤ fromCefr AND cefrTo ≥ toCefr (band-overlap logic).
- **Order**: courses ordered by bandRange (lowest → highest CEFR).
- **Caching**: FE cache 5 min; refetch on band-picker change.
- **Empty response**: learner pick A1→A1 → courses IELTS 1-2 only; no match → empty array (not error).
