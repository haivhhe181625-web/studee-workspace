# Design: Admin Map Template Builder

- **Spec:** `specs/map-template-admin-builder/spec.md`
- **Ngày:** 2026-08-03
- **Tác giả:** AI (từ code 690df00 + 814be85 + 6600767)
- **Trạng thái:** Đã build (spec hồi tố)
- **ADR liên quan:** không (core domain pattern đã biết)

---

## 1. Tóm tắt kiến trúc

Hệ thống **CourseMapTemplate** tách riêng admin (tạo/sửa cấu trúc) với learner (tiêu thụ template đã publish). Kiến trúc:

1. **Data layer** — `CourseMapTemplate` collection (embedding node/edge, một doc = một template version).
2. **Business logic** — `map-template-writer.js` service (HARD RULE: admin-api chỉ delegate).
3. **Validation** — pure `map-template-validate.js` (itemIds async, structural-sig, cycle detect).
4. **Admin API** — bespoke `map-template.admin.js` router (KHÔNG factory vì hybrid versioning).
5. **Learner consumption** — `map-builder.js` `loadLatestTemplate` filter status + resolver template-selection.

**Tư tưởng core:** Shared template docs (read-mostly, cacheable) + per-user LearnerPath ref (templateKey, version) + resolver (program/band → template).

---

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Template sharing | Một CourseMapTemplate doc để mọi learner ref (KHÔNG copy per-user) | Giảm storage N倍, read-mostly cacheable, single-source-of-truth cho health check | Per-user snapshot (LearnerPath) khác → học viên cũ pinned, mới nhận version mới |
| Hybrid versioning | Live-safe (itemIds/pos/mastery) in-place; structural (nodes/edges/gating) → new-version | Nhanh iterate nội dung, structural safety (learner đang học KHÔNG vỡ mid-course) | Learner nên biết sửa cấu trúc hơi delayed |
| Status lifecycle | draft/published/archived (default published backward-compat) | Admin cần draft-edit → publish gate; learner xem published only; archived = soft-delete | Admin workflow thêm bước publish (tránh live draft vô ý) |
| Validator architecture | Pure sync + async itemIds (1 query gom) | Testable, reusable ở update/publish, dễ mock | Async block → 422 deferred (trên update/publish, không create) |
| Structural-sig canonicalize | Node sort nodeKey; edge as set; **KHÔNG** position/masteryThreshold/itemIds count | Reorder array vô hại (not structural), false-409 reduced | Cần dev hiểu sig + maintain sort logic |
| Permissions model | maptemplate:read/write/delete/manage (publish=manage only) | Content Team (`write`) soạn, admin (`manage`) publish | Thêm role management step |
| Program tag | Nullable enum, live-safe (NOT structural-sig) | Resolver match per-learner, fallback if-mismatch | Null ≠ mismatch (universal phủ tất cả band) — cần validation rõ |
| Template selection | Resolver match 1 template/learner, fallback demo-foundation | Single-source authoritative choice, learner pinned → no retroactive | Không multi-template composition / concurrent map exploration |
| Dev utility | Wipe learner + revert published→draft (gated prod) | Dev test free thoải mái | Destructive, phải prod-guard cứng |

---

## 3. Data model

Tham chiếu chi tiết ở `data-model.md`. Tóm tắt:

**CourseMapTemplate** (collection: `course_map_templates`)
- `_id` (ObjectId)
- `templateKey` (String, required, unique per key+version)
- `version` (Number, default 1)
- `status` (enum: draft|published|archived, default published)
- `cefrRange` (object: floor/ceiling ∈ CEFR_LEVELS | null)
- `program` (enum: PROGRAM_KEYS | null = universal)
- `nodes[]` — embedded MapNode (nodeKey, nodeKind, activityType, config, skillTag, subskillTags, prerequisites[], gatePolicy, position, masteryThreshold)
- `edges[]` — embedded Edge (from, to)
- `generated` (Boolean, default false — track course→map VIEW vs hand-authored)
- `sourceCourseSlug` (String | null — track View provenance)
- `timestamps` (createdAt, updatedAt)

**Unique index:** `(templateKey, version)` → 1 doc per key+version.

---

## 4. Luồng dữ liệu

### 4a. Admin tạo template

```
POST /api/admin/map-templates (body: {templateKey, cefrRange, program})
  → verifyToken → verifyPermission(maptemplate:write)
  → map-template-writer.createTemplate
    ├─ validate templateKey unique (find existing, throw E11000 catch → 409)
    ├─ insert v1 draft
    └─ trả DTO ({id, templateKey, version:1, status:draft, ...})
  → auditLog
  → 201 CREATED
```

### 4b. Admin sửa nodes/edges

```
PUT /api/admin/map-templates/:id (body: {cefrRange, program, nodes[], edges[]})
  → verifyToken → verifyPermission(maptemplate:write)
  → map-template-writer.updateTemplate(id, body)
    ├─ load doc từ DB
    ├─ validate(incoming)
    │  ├─ nodeKey unique, enum, skillTag, subskillTags
    │  ├─ prerequisites → node exist, no self
    │  ├─ edges → from/to exist, **mirror prerequisites**
    │  ├─ itemIds → mcq, active, centerId:null (async 1 query)
    │  ├─ MCQ itemIds rỗng → warn draft / ERROR publish
    │  └─ cycle → warn draft / ERROR publish
    ├─ IF status=published && structuralSig(incoming) ≠ structuralSig(stored):
    │   throw 409 TEMPLATE_STRUCTURAL_LOCKED
    ├─ ELSE (draft or live-safe):
    │   findByIdAndUpdate({$set: pick(incoming, [cefrRange,program,nodes,edges])})
    └─ trả updated DTO
  → auditLog
  → 200 OK
```

### 4c. Admin publish draft

```
POST /api/admin/map-templates/:id/publish
  → verifyToken → verifyPermission(maptemplate:manage)
  → map-template-writer.publishTemplate(id)
    ├─ load doc
    ├─ validate(doc) — re-check full:
    │  ├─ MCQ itemIds KHÔNG rỗng
    │  ├─ NO cycle
    │  └─ entry node tồn tại (root UNLOCKED)
    ├─ IF invalid → 422 TEMPLATE_VALIDATION_FAILED
    ├─ findByIdAndUpdate({$set: {status: published}})
    └─ trả updated DTO
  → auditLog
  → 200 OK
```

### 4d. Admin tạo version mới

```
POST /api/admin/map-templates/:id/new-version
  → verifyToken → verifyPermission(maptemplate:write)
  → map-template-writer.createNewVersion(id)
    ├─ load doc (viewed version, draft or not)
    ├─ v = max(version all docs templateKey) + 1
    ├─ clone → insert v+1 draft (catch E11000 → retry, finally 409 if repeat)
    └─ trả new DTO
  → auditLog
  → 201 CREATED
```

### 4e. Learner enrollment — template selection

```
POST /api/adaptive/learner-paths (create new learner path)
  → adaptive.service.ensureActivePath(studentModel)
    ├─ call resolveTemplatesForLearner(studentModel)
    │  ├─ find templates published || status=undefined (legacy)
    │  ├─ filter program match: exact (+2 score) | universal (+1) | skip
    │  ├─ filter cefrRange cover band: null (universal) | [floor,ceiling] | skip
    │  ├─ score rank: program-exact > universal; range-fit tightest > version high
    │  ├─ eligibility: entry node UNLOCKED
    │  └─ return [template] or []
    ├─ IF match: use returned template
    ├─ IF no match: fallback demo-foundation
    ├─ IF no fallback: 404
    └─ build path từ template

  LearnerPath.segments pin (templateKey, version)
  → học viên cũ KHÔNG ảnh hưởng resolve logic change
```

### 4f. Dev reset

```
POST /api/admin/map-templates/:id/dev-reset-progress
  → verifyToken → verifyPermission(maptemplate:manage)
  → IF NODE_ENV=production: throw 403 DEV_RESET_DISABLED
  → map-template-writer.devResetProgress(id)
    ├─ find doc (template)
    ├─ find LearnerPath bám (templateKey, version)
    ├─ delete LearnerPath, ActivityResult, RemediationLoop per learner
    ├─ findByIdAndUpdate doc {$set: {status: draft}}
    └─ trả {affectedLearners: N, deleted: {...}}
  → auditLog
  → 200 OK
```

---

## 5. Contracts

Tham chiếu:
- `contracts/map-template-crud-publish.md` — admin endpoints list/create/update/publish/delete
- `contracts/template-selection-resolver.md` — resolver boundary (StudentModel → template)

---

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa | Commit |
|---|---|---|---|
| `modules/learning-map/course-map-template.model.js` | CourseMapTemplate schema, NODE_KINDS/ACTIVITY_TYPES/GATE_POLICIES const | Tạo | 690df00 |
| `modules/learning-map/map-template-validate.js` | Pure validator (structural-sig, itemIds async, cycle, edges-mirror) | Tạo | 690df00 |
| `modules/learning-map/map-template-writer.js` | Service: create/get/list/update/publish/archive/new-version/dev-reset | Tạo | 690df00 + 814be85 |
| `modules/learning-map/template-resolver.js` | Resolver: StudentModel → CourseMapTemplate match logic | Tạo | 6600767 |
| `modules/learning-map/map-builder.js` | loadLatestTemplate filter $nin:[draft,archived]; plumb resolver | Sửa | 690df00 + 6600767 |
| `admin-api/resources/map-template.admin.js` | Bespoke router: 7 endpoint (list/create/update/new-version/publish/delete/dev-reset) | Tạo | 690df00 + 814be85 |
| `admin-api/resources/question-bank.admin.js` | Extend: GET /by-ids (perm:question:read, centerId:null lock, ObjectId-cap) | Sửa | 690df00 |
| `admin-api/_router.js` | Mount map-template router | Sửa | 690df00 |
| `constants/permissions.js` | Add MAPTEMPLATE_READ/WRITE/DELETE/MANAGE | Sửa | 690df00 |
| `constants/permission-groups.js` | Add labels + group (CI catalog test) | Sửa | 690df00 |
| `constants/http-status.js` | Add 409 CONFLICT (nếu chưa có) | Sửa | 690df00 |
| `constants/error-codes.js` | Add ERR codes (TEMPLATE_KEY_TAKEN, TEMPLATE_STRUCTURAL_LOCKED, ..., DEV_RESET_DISABLED) | Sửa | 690df00 + 814be85 |
| `__tests__/admin-map-template.test.js` | Supertest 7 endpoint + 403 perm + injection guard | Tạo | 690df00 |
| `__tests__/map-template-validate.test.js` | Jest validator (nodeKey unique, edges-mirror, itemIds, cycle) | Tạo | 690df00 |
| `__tests__/learning-map-template-resolver.test.js` | Jest resolver (program match, band cover, fallback) | Tạo | 6600767 |

---

## 7. Xử lý lỗi

| Tình huống | HTTP | Error code | Message |
|---|---|---|---|
| TemplateKey dùng lại | 409 | TEMPLATE_KEY_TAKEN | "Template key already exists" |
| Structural edit published | 409 | TEMPLATE_STRUCTURAL_LOCKED | "Structural edits require new version" |
| Last published archive | 409 | TEMPLATE_LAST_PUBLISHED | "Cannot delete last published version" |
| Validation fail (itemIds/cycle/edge) | 422 | TEMPLATE_VALIDATION_FAILED | "Template validation failed" (errors[] chi tiết) |
| ItemIds invalid (type/status/center) | 422 | (trong errors[] code) | ITEM_TYPE_INVALID / ITEM_STATUS_INVALID / ITEM_CENTERFORBIDDEN |
| Cycle | 422 | (trong errors[]) | CYCLE_INVALID |
| Edge lệch | 422 | (trong errors[]) | EDGE_PREREQ_MISMATCH |
| Program invalid | 422 | PROGRAM_INVALID | "Invalid program tag" |
| No entry node | 422 | NO_ENTRY_NODE | "Template has no entry node" |
| Dev-reset production | 403 | DEV_RESET_DISABLED | "Dev reset is disabled in production" |
| Permission denied | 403 | FORBIDDEN | (standard) |
| Template not found | 404 | NOT_FOUND | "Template not found" |
| Invalid ObjectId | 404 | NOT_FOUND | "Template not found" |
| Bad request | 400 | BAD_REQUEST | (vd: invalid JSON) |

---

## 8. Bảo mật & quyền

**Permission model:**
- `maptemplate:read` — GET list/detail. Quyền default platform admin + content team review.
- `maptemplate:write` — POST create, PUT edit, POST new-version. Content Team soạn draft + itemIds.
- `maptemplate:delete` — DELETE (soft-archive). Cleanup.
- `maptemplate:manage` — POST publish, POST dev-reset. Platform admin only.

**Derived by role:**
- Platform Admin → `maptemplate:*` (admin).
- Content Team (bundle tạo sau) → `maptemplate:read|write`.
- Reviewer (question:read qua picker) → không maptemplate perm, chỉ question:read.

**Sensitive fields (select: false nếu cần):**
- Không. Map public structure, không secret. ItemIds/answerKey + config.theory strip khi trả learner.

**Tenant scope:**
- Map KHÔNG có centerId → platform global. Learner surface chỉ read public node structure (itemIds/answerKey/theory stripped).

**Audit:**
- Mỗi mutation (create/update/publish/archive/dev-reset) log `admin.command.executed` với command=`maptemplate.{action}`.

---

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

**Risk — loadLatestTemplate filter mổi học viên cũ:**
- **Mitigation:** `$nin:[draft,archived]` bao doc cũ status=undefined (legacy) vẫn served. Test regression learning-map must pass.

**Risk — structural-sig false-positive:**
- **Mitigation:** Canonicalize node sort nodeKey, edge as set. Test node reorder KHÔNG 409.

**Risk — itemIds validate N+1:**
- **Mitigation:** 1 query `find({_id:{$in:allIds}})`. Profile slow query nếu >10K items.

**Risk — dev-reset destructive:**
- **Mitigation:** Hard-block NODE_ENV=production. Audit log mỗi call. FE nút confirm 2-lần.

**Risk — template no entry node:**
- **Mitigation:** Publish validator block `NO_ENTRY_NODE`. List health show node status.

**Đánh đổi — new-version từ viewed doc (KHÔNG latest published):**
- User view v1 (mà latest published v2 tồn tại), post new-version → clone v1. FE UX cần cho user biết "new-version từ doc được xem".

---

## 10. Testing strategy

**Unit (Jest):**
- `map-template-validate.test.js` (18 test)
  - Structural-sig canonical
  - ItemIds validation (mcq/active/centerId:null)
  - Edges mirror prerequisites
  - Cycle detect
  - NodeKey unique
  - Enum validation
  - SubskillTags free-string (KHÔNG ràng SUBSKILLS)

**Integration (Supertest):**
- `admin-map-template.test.js` (24 test)
  - Create v1 draft
  - Update draft full / published live-safe / published structural → 409
  - Publish re-validate
  - New-version
  - Delete soft-archive
  - Dev-reset (non-prod)
  - **Per-route 403 test** (bespoke router KHÔNG factory drift-guard, test tay)
  - Injection guard (status coerce, q escape, ObjectId valid)
  - By-ids cross-tenant rejection + cap

**Integration (Resolver):**
- `learning-map-template-resolver.test.js`
  - Program match (exact > universal)
  - Band cover (null universal, overlap, mismatch → fallback)
  - Entry node UNLOCKED check
  - Fallback demo-foundation
  - Learner pinned (no retroactive)

**Regression:**
- Full `learning-map-*` test suite (157 test xanh trước + sau).
- S-A learner surface KHÔNG ảnh hưởng (itemIds/answerKey strip).

---

## 11. Rollout / cross-repo sequencing

**S-B only (exe-api):** KHÔNG cross-repo dependency (learner surface đã strip).

**Sequence:**
1. **Commit 690df00** — Model + validator + writer + admin-api + perms (DONE BE).
2. **Commit 814be85** — Dev-reset endpoint (DONE).
3. **Commit 6600767** — Resolver + template-selection (DONE).
4. **FE xe-admin** — Portal builder UI (separate spec/track, KHÔNG blocking docs này).

**Live rollout:**
- BE deployed → FE portal live → admin start creating → learner auto-enroll via resolver.

---

## 12. Đối chiếu Acceptance criteria

| AC | Được đáp ứng bởi |
|---|---|
| AC-1 POST create v1 draft | map-template-writer.createTemplate + admin route POST / |
| AC-2 TemplateKey dùng lại 409 | createTemplate catch E11000 → 409 TEMPLATE_KEY_TAKEN |
| AC-3 Sửa draft in-place | updateTemplate (draft → $set, no 409) |
| AC-4 Sửa published structural 409 | updateTemplate structural-sig diff → 409 TEMPLATE_STRUCTURAL_LOCKED |
| AC-5 Sửa itemIds draft in-place | updateTemplate (live-safe KHÔNG structural) |
| AC-6 Sửa itemIds published in-place | updateTemplate live-safe (học viên attempt cũ bind) |
| AC-7-9 ItemIds validation 422 | map-template-validate itemIds async check + by-ids query |
| AC-10 MCQ rỗng draft ok | validator KHÔNG error draft (warn only) |
| AC-11 MCQ rỗng publish 422 | publishTemplate re-validate → error |
| AC-12 Edge lệch 422 | validator edges-mirror-prerequisites check |
| AC-13-14 Cycle draft/publish | validator cycle detect (warn draft, error publish) |
| AC-15-18 Node validation | map-template-validate enum/unique/skillTag/subskillTags |
| AC-19-20 Publish perm manage | admin route POST /publish verifyPermission(manage) |
| AC-21-22 New-version clone | createNewVersion clone v+1 (E11000 retry) |
| AC-23-24 List/detail GET | admin route GET / & /:id + DTO mapper |
| AC-25-26 Delete/archive | admin route DELETE soft-archive, block last-published |
| AC-27-28 Dev-reset | devResetProgress (wipe learner, prod-guard, audit) |
| AC-29 Program live-safe | updateTemplate (program NOT structural) |
| AC-30-32 Template resolver | resolveTemplatesForLearner (program match, band cover, fallback) |

