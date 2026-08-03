# Acceptance Criteria: Admin Map Template Builder

- **Spec:** `specs/map-template-admin-builder/spec.md`
- **Ngày:** 2026-08-03
- **Status code mapping:** 200 OK · 201 CREATED · 400 BAD_REQUEST · 403 FORBIDDEN · 404 NOT_FOUND · 409 CONFLICT · 422 UNPROCESSABLE_ENTITY

---

## Tiêu chí chức năng

### AC-1: Tạo template v1 draft

- **Given** user có quyền `maptemplate:write`
- **When** POST `/api/admin/map-templates` `{templateKey:"course-ielts-foundation", cefrRange:{floor:"A1",ceiling:"B2"}, program:"ielts"}`
- **Then** 201 CREATED, trả `{data:{id:"...", templateKey:"course-ielts-foundation", version:1, status:"draft", cefrRange:{...}, nodes:[], edges:[], program:"ielts", updatedAt:"..."}}`

### AC-2: TemplateKey dùng lại → 409

- **Given** template `course-ielts-foundation:v1` đã tồn tại (status=draft|published|archived)
- **When** POST `/api/admin/map-templates` `{templateKey:"course-ielts-foundation"}`
- **Then** 409 CONFLICT `{error:{code:"TEMPLATE_KEY_TAKEN", message:"Template key already exists"}}`

### AC-3: Sửa node/edge draft → in-place OK

- **Given** template `id="abc123"` status=`draft`, hiện có 2 node
- **When** PUT `/api/admin/map-templates/abc123` `{nodes:[{nodeKey:"n1", nodeKind:"CORE", activityType:"MCQ_QUIZ", ...}, {nodeKey:"n3", nodeKind:"PRACTICE", ...}], edges:[...]}`
- **Then** 200 OK, doc cập nhật in-place, trả updated DTO

### AC-4: Sửa node/edge published (structural) → 409

- **Given** template `id="abc123"` status=`published`, nodes hiện có `[{nodeKey:"n1",...},{nodeKey:"n2",...}]`
- **When** PUT `/api/admin/map-templates/abc123` `{nodes:[{nodeKey:"n1",...}]}` (xoá n2 — structural change)
- **Then** 409 CONFLICT `{error:{code:"TEMPLATE_STRUCTURAL_LOCKED", message:"Structural edits on published template require a new version"}}`

### AC-5: Sửa itemIds draft → in-place OK

- **Given** template draft với node MCQ `{nodeKey:"n1", config:{itemIds:[]}}`
- **When** PUT `{nodes:[{..., config:{itemIds:["q1","q2","q3"]}}]}`
- **Then** 200 OK, itemIds cập nhật ngay, học viên xem ngay

### AC-6: Sửa itemIds published (live-safe) → in-place OK

- **Given** template published, học viên chơi node với itemIds `["q1"]`
- **When** PUT `{nodes:[{nodeKey:"n1", config:{itemIds:["q2","q3"]}}]}`
- **Then** 200 OK. Node update ngay, attempt learner chôn ở câu cũ (q1) giữ lại, node bind nhất quán.

### AC-7: ItemIds chứa non-mcq item → 422

- **Given** item `id="non_mcq"` có `itemType:"quiz"` (không mcq)
- **When** PUT `{nodes:[{config:{itemIds:["non_mcq"]}}]}`
- **Then** 422 UNPROCESSABLE_ENTITY `{error:{code:"TEMPLATE_VALIDATION_FAILED", errors:[{path:"nodes[0].config.itemIds[0]", message:"Item must be MCQ type", code:"ITEM_TYPE_INVALID"}]}}`

### AC-8: ItemIds chứa inactive item → 422

- **Given** item `id="inactive_mcq"` có `status:"inactive"`
- **When** PUT `{nodes:[{config:{itemIds:["inactive_mcq"]}}]}`
- **Then** 422 `{errors:[{path:"nodes[0].config.itemIds[0]", message:"Item must have active status", code:"ITEM_STATUS_INVALID"}]}`

### AC-9: ItemIds chứa private center item → 422

- **Given** item `id="private_item"` có `centerId:"63abc...abc"` (KHÔNG null)
- **When** PUT với itemIds=`["private_item"]`
- **Then** 422 `{errors:[{path:"nodes[0].config.itemIds[0]", message:"Item must be shared bank only", code:"ITEM_CENTERFORBIDDEN"}]}`

### AC-10: MCQ itemIds rỗng draft → warn only (ok)

- **Given** node MCQ draft với `config:{itemIds:[]}`
- **When** PUT `{nodes:[{nodeKey:"n1", activityType:"MCQ_QUIZ", config:{itemIds:[]}}]}`
- **Then** 200 OK (warn ở response `_warnings` nếu trả, hoặc bỏ qua — draft cho phép rỗng)

### AC-11: MCQ itemIds rỗng publish → 422 BLOCK

- **Given** node MCQ `{activityType:"MCQ_QUIZ", config:{itemIds:[]}}`
- **When** POST `/api/admin/map-templates/id/publish`
- **Then** 422 `{error:{code:"TEMPLATE_VALIDATION_FAILED", errors:[{path:"nodes[0].config.itemIds", message:"MCQ node requires at least one item", code:"ITEMCOUNT_INVALID"}]}}`

### AC-12: Edges không mirror prerequisites → 422

- **Given** node `n1` có `prerequisites:["n0"]`, nhưng edges KHÔNG chứa edge từ n0→n1
- **When** PUT `{nodes:[...], edges:[{from:"...",to:"..."}]}`
- **Then** 422 `{errors:[{path:"edges", message:"Edges must mirror prerequisites", code:"EDGE_PREREQ_MISMATCH"}]}`

### AC-13: Cycle prerequisite draft → warn (ok)

- **Given** nodes tạo vòng: n1→n2→n3→n1 (prerequisites)
- **When** PUT draft
- **Then** 200 OK (hoặc warn ở response, draft ok)

### AC-14: Cycle prerequisite publish → 422 BLOCK

- **Given** template draft với cycle (như AC-13)
- **When** POST `/publish`
- **Then** 422 `{errors:[{path:"nodes", message:"Prerequisite cycle detected", code:"CYCLE_INVALID"}]}`

### AC-15: NodeKey duplicate → 422

- **Given** nodes `[{nodeKey:"n1"}, {nodeKey:"n1"}]` (duplicate)
- **When** PUT
- **Then** 422 `{errors:[{path:"nodes[1].nodeKey", message:"NodeKey must be unique", code:"NODEKEY_DUPLICATE"}]}`

### AC-16: NodeKind/activityType invalid enum → 422

- **Given** node `{nodeKind:"INVALID_KIND", ...}`
- **When** PUT
- **Then** 422 `{errors:[{path:"nodes[0].nodeKind", message:"Invalid nodeKind", code:"ENUM_INVALID"}]}`

### AC-17: SkillTag ∈ ALL_CATEGORIES | null → 422 nếu lạ

- **Given** node `{skillTag:"unknown_category"}`
- **When** PUT
- **Then** 422 `{errors:[{path:"nodes[0].skillTag", message:"SkillTag not in allowed categories", code:"SKILLTAG_INVALID"}]}`

### AC-18: SubskillTags = free string[] (KHÔNG ràng SUBSKILLS) → OK

- **Given** node `{subskillTags:["present-simple", "daily-routines"]}` (seed dùng non-SUBSKILLS values)
- **When** PUT
- **Then** 200 OK (không 422 — free-string allowed, không ràng SUBSKILLS constant)

### AC-19: Publish gated maptemplate:manage

- **Given** user có `maptemplate:write` (KHÔNG `manage`)
- **When** POST `/:id/publish`
- **Then** 403 FORBIDDEN `{error:{code:"FORBIDDEN", message:"..."}}`

### AC-20: Publish gated maptemplate:manage (positive)

- **Given** user có `maptemplate:manage`, template draft hợp lệ
- **When** POST `/:id/publish`
- **Then** 200 OK, status → `published`

### AC-21: New-version clone từ doc được xem

- **Given** user view `/:id` (v1 draft), sau đó POST `/:id/new-version`
- **When** tạo version
- **Then** 201 CREATED, trả v2 draft (clone v1 toàn bộ nodes/edges/config, version=2)

### AC-22: New-version version conflict → retry E11000 → 409 (sau lần cuối)

- **Given** 2 request đồng thời POST `/:id/new-version` trên cùng v1
- **When** cả 2 cố gọi v = max + 1
- **Then** 1 thành công (201), 1 retry → 409 TEMPLATE_VERSION_CONFLICT hoặc cạnh tranh E11000 bị handle)

### AC-23: GET /:id list pagination

- **When** GET `/api/admin/map-templates?page=1&limit=20&status=published&q=ielts`
- **Then** 200 `{data:{items:[{id,templateKey,version,status,nodeCount,edgeCount,updatedAt},...], total:N, page:1, limit:20}}`

### AC-24: GET /:id detail đầy đủ

- **When** GET `/api/admin/map-templates/id`
- **Then** 200 `{data:{id,templateKey,version,status,cefrRange,program,nodes:[...],edges:[...],updatedAt,...}}`

### AC-25: DELETE (soft archive)

- **Given** template draft `id="abc123"`
- **When** DELETE `/api/admin/map-templates/abc123`
- **Then** 200 OK, status → `archived` (KHÔNG hard-delete)

### AC-26: DELETE published-cuối → 409 TEMPLATE_LAST_PUBLISHED

- **Given** template với key="course-ielts" chỉ có 1 doc published (v1)
- **When** DELETE
- **Then** 409 `{error:{code:"TEMPLATE_LAST_PUBLISHED", message:"Cannot delete the last published version"}}`

### AC-27: Dev-reset wipe learner + hạ draft

- **Given** template published, có 10 learner pinned (template, v1)
- **When** POST `/:id/dev-reset-progress` (perm=manage, env≠prod)
- **Then** 200 `{data:{templateKey:"...", revertedToDraft:true, affectedLearners:10, deleted:{learnerPaths:10,activityResults:X,remediationLoops:Y}}}`
  - Verify: LearnerPath xoá, ActivityResult xoá, RemediationLoop xoá
  - Verify: StudentModel KHÔNG xoá
  - Verify: LearningEvent KHÔNG xoá
  - Verify: template status → draft

### AC-28: Dev-reset production → 403 DEV_RESET_DISABLED

- **Given** NODE_ENV='production'
- **When** POST `/:id/dev-reset-progress` (dù có quyền manage)
- **Then** 403 `{error:{code:"DEV_RESET_DISABLED", message:"Dev reset is disabled in production"}}`

### AC-29: Program tag live-safe (sửa published ok)

- **Given** template published `program:"ielts"`
- **When** PUT `{program:"toeic"}`
- **Then** 200 OK (không 409 — live-safe, KHÔNG structural)

### AC-30: Template resolver match program+band → learner mới

- **Given** StudentModel `{target:{program:"ielts", cefr:"B1"}}`
- **When** ensureActivePath gọi resolveTemplatesForLearner
- **Then** trả template ielts cefrRange cover B1 (hoặc universal), KHÔNG demo-foundation (nếu match)

### AC-31: No match → fallback demo-foundation

- **Given** StudentModel `{target:{program:"toeic", cefr:"C1"}}`, KHÔNG template toeic cover C1
- **When** ensureActivePath call resolver
- **Then** trả demo-foundation (fallback), KHÔNG 404

### AC-32: Learner cũ pinned → KHÔNG ảnh hưởng resolver

- **Given** LearnerPath.segments pin `(templateKey,version)` đã build
- **When** admin đổi resolver logic hay publish template mới
- **Then** learner xem cũ (pinned version), KHÔNG retroactive

---

## Tiêu chí lỗi / edge case

### AC-E1: ItemIds N+1 query → 1 query

- **Given** 100 itemIds đưa vào node (worst case nhiều node)
- **When** validate (PUT)
- **Then** 1 database query gom `find({_id:{$in:[...]}})`, KHÔNG N+1

### AC-E2: ObjectId invalid → ignore + cap

- **Given** POST by-ids với `?ids=invalid,63abc...abc,another_junk`
- **When** call `/api/admin/question-bank/by-ids`
- **Then** chỉ trả valid ObjectId (invalid skip), cap ~100, KHÔNG 500

### AC-E3: Learner surface itemIds stripped

- **Given** learner gọi GET `/api/learning-map/:id` (student surface, KHÔNG admin)
- **When** lấy template
- **Then** KHÔNG có `nodes[].config.itemIds` (stripped), KHÔNG answerKey

### AC-E4: Filter injection ?status[$ne]= & ReDoS ?q=

- **Given** `GET /?status[$ne]=draft&q=(a|a)*b`
- **When** list
- **Then** chỉ string-coerce status, escape regex q, KHÔNG server-side ReDoS / NoSQL inject

---

## Tiêu chí phi chức năng

| Loại | Tiêu chí | Cách kiểm tra |
|---|---|---|
| Bảo mật | Endpoint yêu cầu đúng perm (read/write/delete/manage) | Supertest 403 thiếu perm mỗi route (S8 red-team finding) |
| Bảo mật | ItemIds chỉ shared bank (centerId:null) | Test cross-tenant rejection in by-ids; learner surface strip |
| Bảo mật | Dev-reset hard-block production | Test NODE_ENV='production' → 403 |
| Audit | Mỗi mutation (create/update/publish/archive/dev-reset) audit-log | Verify `auditLog` gọi với `maptemplate.{command}` |
| Dữ liệu | Structural-sig canonicalized (node sort, edge as set) | Test node reorder → KHÔNG 409 false-positive |
| Dữ liệu | Version increment atomic (E11000 retry) | Test 2 concurrent new-version → 1 ok, 1 conflict |

---

## Ngoài phạm vi kiểm thử

- **Portal UI logic** — tách spec `exe-admin` map-template-builder.tsx (sync, state, network retry).
- **Learner playable map** — player engines tách spec learning-map-content-players.
- **Course→Map generation** — tách spec course-driven-map (sourceCourseSlug only tracking).
- **Performance cap** — >1000 template, >500 node/template — ngoài scope phase 1 (no quantitative requirement).
- **Undo/export/import** — ngoài scope; re-seed nếu cần.

