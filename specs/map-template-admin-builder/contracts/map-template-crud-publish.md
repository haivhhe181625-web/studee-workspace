# Contract: Map Template CRUD & Publish

- **Loại:** Nội bộ module (`learning-map` service → admin-api)
- **Bên cung cấp (provider):** `exe-api` (admin-api `/api/admin/map-templates`)
- **Bên tiêu thụ (consumer):** Portal `exe-admin` (map-template-builder.tsx)
- **Trạng thái:** Live (commit 690df00 + 814be85)

---

## Base endpoint

```
GET|POST|PUT|DELETE /api/admin/map-templates
POST /api/admin/map-templates/:id/new-version
POST /api/admin/map-templates/:id/publish
POST /api/admin/map-templates/:id/dev-reset-progress
```

All return `{data: ...}` or `{error: ...}` (admin convention).

---

## GET / — List templates

```
GET /api/admin/map-templates?page=1&limit=20&status=published&q=ielts
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:read`

### Request

| Param | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `page` | Integer | ✗ | Default 1. Validate ≥1. |
| `limit` | Integer | ✗ | Default 20. Cap 100 (Math.min(100, incoming)). |
| `status` | String (enum) | ✗ | Filter: draft|published|archived. Coerce string, no injection. |
| `q` | String | ✗ | Search templateKey/title. Escape regex. |

### Response — thành công

```json
{
  "data": {
    "items": [
      {
        "id": "64abc...",
        "templateKey": "course-ielts-foundation",
        "version": 1,
        "status": "published",
        "cefrRange": { "floor": "A1", "ceiling": "B2" },
        "program": "ielts",
        "nodeCount": 12,
        "edgeCount": 13,
        "pinnedLearners": true,
        "updatedAt": "2026-08-01T15:30:00Z"
      }
    ],
    "total": 42,
    "page": 1,
    "limit": 20
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `id` | String (ObjectId) | Template _id as string. |
| `templateKey` | String | Stable template identifier. |
| `version` | Number | Current version. |
| `status` | String | draft|published|archived. |
| `cefrRange` | Object | {floor, ceiling} or null. |
| `program` | String | Program tag or null (universal). |
| `nodeCount` | Number | Count nodes[]. |
| `edgeCount` | Number | Count edges[]. |
| `pinnedLearners` | Boolean | ≥1 learner pinned to this version (health flag). |
| `updatedAt` | String (ISO) | Timestamp. |

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:read` |
| 400 | BAD_REQUEST | Invalid pagination (page < 1) |

---

## GET /:id — Detail (full nested doc)

```
GET /api/admin/map-templates/:id
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:read`

### Request

| Param | Kiểu | Ghi chú |
|---|---|---|
| `:id` | String (ObjectId) | Template _id. Validate ObjectId.isValid() → 404 nếu invalid. |

### Response — thành công

```json
{
  "data": {
    "id": "64abc...",
    "templateKey": "course-ielts-foundation",
    "version": 1,
    "status": "draft",
    "cefrRange": { "floor": "A1", "ceiling": "B2" },
    "program": "ielts",
    "nodes": [
      {
        "nodeKey": "p1:l1",
        "nodeKind": "CORE",
        "activityType": "MCQ_QUIZ",
        "activityRef": { "refCollection": null, "refId": null, "slug": null },
        "config": { "itemIds": ["63x1", "63x2"], "passRatio": 0.8 },
        "skillTag": "listening",
        "subskillTags": ["short-conversations"],
        "prerequisites": [],
        "gatePolicy": "HARD",
        "position": { "row": 0, "col": 0 },
        "masteryThreshold": 0.8
      }
    ],
    "edges": [
      { "from": "p1:l1", "to": "p1:l2" }
    ],
    "itemsHealth": [
      {
        "nodeKey": "p1:l1",
        "itemCount": 2,
        "minRequired": 5,
        "healthy": false
      }
    ],
    "updatedAt": "2026-08-01T15:30:00Z"
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:read` |
| 404 | NOT_FOUND | ID invalid hoặc doc not found |

---

## POST / — Create v1 draft

```
POST /api/admin/map-templates
Content-Type: application/json

{
  "templateKey": "course-ielts-foundation",
  "cefrRange": { "floor": "A1", "ceiling": "B2" },
  "program": "ielts"
}
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:write`

### Request

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `templateKey` | String | ✓ | Non-empty. Unique per version. |
| `cefrRange` | Object | ✗ | {floor, ceiling} ∈ CEFR_LEVELS or null. Default {}. |
| `program` | String (enum) | ✗ | PROGRAM_KEYS or null (universal). Default null. |

### Response — thành công (201 CREATED)

```json
{
  "data": {
    "id": "64abc123...",
    "templateKey": "course-ielts-foundation",
    "version": 1,
    "status": "draft",
    "cefrRange": { "floor": "A1", "ceiling": "B2" },
    "program": "ielts",
    "nodes": [],
    "edges": [],
    "updatedAt": "2026-08-01T15:30:00Z"
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:write` |
| 409 | TEMPLATE_KEY_TAKEN | TemplateKey (v1) đã tồn tại |
| 400 | BAD_REQUEST | templateKey empty |

---

## PUT /:id — Update whole doc (nodes, edges, config, range, program)

```
PUT /api/admin/map-templates/:id
Content-Type: application/json

{
  "cefrRange": { "floor": "A1", "ceiling": "B2" },
  "program": "ielts",
  "nodes": [ ... ],
  "edges": [ ... ]
}
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:write`

### Request

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `cefrRange` | Object | ✗ | {floor?, ceiling?} ∈ CEFR_LEVELS or null. |
| `program` | String (enum) | ✗ | PROGRAM_KEYS or null. |
| `nodes` | Array | ✗ | Full node list (replace entirely, not merge). |
| `edges` | Array | ✗ | Full edge list. |

**Whitelist cứng (KHÔNG accept):** `status, version, templateKey, createdAt, updatedAt`.

### Response — thành công (200 OK)

```json
{
  "data": {
    "id": "64abc...",
    "templateKey": "...",
    "version": 1,
    "status": "draft",
    "cefrRange": { ... },
    "program": "ielts",
    "nodes": [ ... ],
    "edges": [ ... ],
    "updatedAt": "2026-08-01T15:30:00Z"
  }
}
```

### Response — validation failed (422)

```json
{
  "error": {
    "code": "TEMPLATE_VALIDATION_FAILED",
    "message": "Template validation failed",
    "errors": [
      {
        "path": "nodes[0].config.itemIds[0]",
        "message": "Item must be MCQ type",
        "code": "ITEM_TYPE_INVALID"
      },
      {
        "path": "edges[2].to",
        "message": "Target node does not exist",
        "code": "NODE_NOT_FOUND"
      }
    ]
  }
}
```

| Field | Ghi chú |
|---|---|
| `code` | TEMPLATE_VALIDATION_FAILED |
| `errors[]` | Array of {path, message, code}. |
| `path` | Dot notation (e.g., `nodes[0].nodeKey`, `edges[1].from`). |
| `code` | Error code (ITEM_TYPE_INVALID, CYCLE_INVALID, ...). |

### Response — structural lock (409)

**Published template + structural change:**

```json
{
  "error": {
    "code": "TEMPLATE_STRUCTURAL_LOCKED",
    "message": "Structural edits on published template require a new version. Please create a new version and edit there."
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:write` |
| 404 | NOT_FOUND | Template not found |
| 409 | TEMPLATE_STRUCTURAL_LOCKED | Published + structural change |
| 422 | TEMPLATE_VALIDATION_FAILED | Validation error (errors[] detail) |

---

## POST /:id/new-version — Clone & create v+1 draft

```
POST /api/admin/map-templates/:id/new-version
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:write`

### Request

No body. Clone viewed doc (`:id`).

### Response — thành công (201 CREATED)

```json
{
  "data": {
    "id": "64new...",
    "templateKey": "course-ielts-foundation",
    "version": 2,
    "status": "draft",
    "cefrRange": { "floor": "A1", "ceiling": "B2" },
    "program": "ielts",
    "nodes": [ ... ],  // cloned from v1
    "edges": [ ... ],  // cloned
    "updatedAt": "2026-08-01T15:30:00Z"
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:write` |
| 404 | NOT_FOUND | Template not found |
| 409 | TEMPLATE_VERSION_CONFLICT | Concurrent create race (E11000 after retries) |

---

## POST /:id/publish — Draft → Published (re-validate, block if invalid)

```
POST /api/admin/map-templates/:id/publish
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:manage` **(ADMIN ONLY)**

### Request

No body.

### Response — thành công (200 OK)

```json
{
  "data": {
    "id": "64abc...",
    "templateKey": "course-ielts-foundation",
    "version": 1,
    "status": "published",
    "cefrRange": { ... },
    "program": "ielts",
    "nodes": [ ... ],
    "edges": [ ... ],
    "updatedAt": "2026-08-01T15:30:00Z"
  }
}
```

### Response — validation failed (422)

**MCQ itemIds empty, cycle, no entry node:**

```json
{
  "error": {
    "code": "TEMPLATE_VALIDATION_FAILED",
    "message": "Template cannot be published",
    "errors": [
      {
        "path": "nodes[2].config.itemIds",
        "message": "MCQ node requires at least one item",
        "code": "ITEMCOUNT_INVALID"
      },
      {
        "path": "nodes",
        "message": "Prerequisite cycle detected",
        "code": "CYCLE_INVALID"
      }
    ]
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:manage` |
| 404 | NOT_FOUND | Template not found |
| 422 | TEMPLATE_VALIDATION_FAILED | Invalid (errors[] detail) |

---

## DELETE /:id — Soft archive

```
DELETE /api/admin/map-templates/:id
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:delete`

### Request

No body.

### Response — thành công (200 OK)

```json
{
  "data": {
    "id": "64abc...",
    "status": "archived"
  }
}
```

### Response — last published (409)

```json
{
  "error": {
    "code": "TEMPLATE_LAST_PUBLISHED",
    "message": "Cannot delete the last published version of a template"
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:delete` |
| 404 | NOT_FOUND | Template not found |
| 409 | TEMPLATE_LAST_PUBLISHED | Only published version |

---

## POST /:id/dev-reset-progress — Wipe learners + revert draft (DEV ONLY)

```
POST /api/admin/map-templates/:id/dev-reset-progress
```

- **Auth:** JWT user
- **Permission required:** `maptemplate:manage`
- **Prod guard:** 403 if NODE_ENV='production'

### Request

No body.

### Response — thành công (200 OK)

```json
{
  "data": {
    "templateKey": "course-ielts-foundation",
    "revertedToDraft": true,
    "affectedLearners": 10,
    "deleted": {
      "learnerPaths": 10,
      "activityResults": 47,
      "remediationLoops": 3
    }
  }
}
```

### Response — production blocked (403)

```json
{
  "error": {
    "code": "DEV_RESET_DISABLED",
    "message": "Dev reset is disabled in production"
  }
}
```

### Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 403 | FORBIDDEN | User không có `maptemplate:manage` OR NODE_ENV=production |
| 404 | NOT_FOUND | Template not found |

---

## Audit logging

Mỗi mutation audit-log:
```
admin.command.executed {
  adminUser: userId,
  command: maptemplate.{create|update|new-version|publish|archive|dev-reset-progress},
  ip: ip,
  id: templateId,
  target: templateKey (nếu context),
  version: (nếu publish/new-version)
}
```

---

## Versioning / breaking change

**N/A — contract mới (commit 690df00), no breaking.**

---

## Sequencing triển khai (cross-repo)

1. **BE live** (690df00 commit merged) — admin-api active, permissions added.
2. **FE live** (exe-admin portal) — Portal UI consumes endpoints.
3. **Learner auto-enroll** — Template resolver active, new learners pick template per target.

