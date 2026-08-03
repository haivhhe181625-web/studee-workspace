# Spec: Admin Map Template Builder — Gamified Learning Map Authoring

- **Ngày:** 2026-08-03
- **Tác giả:** AI (hồi tố từ plans + code, commit 690df00 + 814be85 + 6600767)
- **Repos/surfaces ảnh hưởng:**
  - `api` (`exe-api`) — **xây trong scope này:** admin API + template engine, map-builder đọc template.
  - `admin` (`exe-admin`) — **xây ngoài scope này** (portal UI, tách spec riêng).
  - `web` (`exe-web`) — **KHÔNG scope này** (learner tiêu thụ map).
- **Module liên quan (nếu là `exe-api`):**
  - `services/api/src/modules/learning-map/` — **ĐÃ TẠƠI** (map-template.model, map-builder, map-template-validate, map-template-writer).
  - `services/api/src/admin-api/resources/map-template.admin.js` — **ĐÃ TẠO** (bespoke router).
  - `services/api/src/admin-api/resources/question-bank.admin.js` — **ĐÃ MỞ RỘNG** (picker by-ids).
  - `services/api/src/constants/permissions.js` — **ĐÃ THÊM** (maptemplate:read|write|delete|manage).
  - `modules/assessment/assessment-question.model.js` — tham chiếu chỉ đọc để validate itemIds (centerId:null, active mcq).
- **Trạng thái:** Đã build (spec hồi tố từ code + plans). Backend DONE (commit 690df00 + 814be85 + 6600767). Frontend ngoài scope docs này.

---

## 1. Mục tiêu

Cho phép Content Team (quyền `maptemplate:write`) **tạo và sửa CourseMapTemplate** (cấu trúc node/edge/gating của bản đồ gamified) và **gán câu hỏi thực từ ngân hàng AssessmentQuestion vào node MCQ** — thay thế hẳn việc seed bằng terminal script. Hệ thống phát hành **2 loại sửa:** (1) live-safe (itemIds, position, masteryThreshold) → in-place tức thì tới học viên; (2) structural (node/edge/gating) trên published → 409, buộc tạo version mới → chỉ học viên mới nhận. **Platform admin** (`maptemplate:manage`) đẩy version từ draft → published và xác nhận health.

---

## 2. Bối cảnh

**Vì sao cần:** Loạt vấn đề tiền tệ:
- Học viên chơi bản đồ gamified nhưng **nội dung (MCQ) dùng fake seed data** từ `seed-map-curated-mcq.js` → không dùng được bank thực curated. Cần cơ chế assign item.
- Admin phải **lên terminal seed** mỗi khi dựng map mới → công sức ops, khó scale, dễ sai.
- Template admin dựng **chỉ dùng được hardcode `demo-foundation`** (`ensureActivePath` build-once) → template mới không tới học viên.

**Hiện trạng đã kiểm tra trong code (không đoán):**
- Model `CourseMapTemplate` (commit 690df00) **ĐÃ TẠO** với fields `templateKey`, `version`, `status`, `nodes[]`, `edges[]`, `cefrRange`, `program` (`modules/learning-map/course-map-template.model.js:1-150`).
- Admin resource (`modules/learning-map/map-template.admin.js`) **ĐÃ BESPOKE** (NOT factory) với 7 endpoint: list, detail, create, update, publish, archive, dev-reset (line 21-195, 690df00).
- **Hybrid versioning** logic — updateTemplate (line 690df00 `map-template-writer.js:89-126`) **ĐÃ ĐÁP ỨNG**: structural-sig canonicalized, published + structural → 409, live-safe in-place.
- **Validator** (commit 690df00, `map-template-validate.js`) **ĐÃ ĐẠT:** itemIds (centerId:null, active mcq) line 93-100; edges-mirror-prereq (line 97-98); MCQ itemIds rỗng → warn draft/error publish (line 99-100); cycle → warn/error (line 98).
- **Template selection (Slice C, commit 6600767)** — `resolveTemplatesForLearner` đối chiếu program + cefrRange với learner target → điều phối enrollment (không hardcode `demo-foundation`). `template-resolver.js` **ĐÃ TẠO**.
- **Dev reset utility** (commit 814be85) `POST /:id/dev-reset-progress` **ĐÃ** (`map-template-writer.js:214-255`) — xoá LearnerPath/ActivityResult/RemediationLoop, hạ draft, gated prod.
- Permissions **ĐÃ THÊM** (`constants/permissions.js` commit 690df00): `maptemplate:read|write|delete|manage`.
- Học viên surface **ĐỦ NGON** (`learning-map.service.js:61-70` trên develop) — strip itemIds, answerKey khi trả học viên.

---

## 3. Phạm vi

### Trong phạm vi

**F-ADMIN-BUILDER (S-B):**
- **IS-1** — Endpoint POST `/api/admin/map-templates/` tạo CourseMapTemplate v1 draft với templateKey + cefrRange + program. Validate templateKey unique + value. 409 TEMPLATE_KEY_TAKEN nếu dùng lại.
- **IS-2** — Endpoint PUT `/:id` sửa toàn bộ doc (nodes, edges, cefrRange, program). Validate node structure, edges, itemIds. 422 TEMPLATE_VALIDATION_FAILED nếu invalid. 409 TEMPLATE_STRUCTURAL_LOCKED nếu published + đổi structural (nodes/edges/gating/skill/config.skill flip empty↔nonempty).
- **IS-3** — Endpoint GET `/` list template (pagination, filter status, q search). GET `/:id` chi tiết đầy đủ.
- **IS-4** — **Hybrid versioning:** Sửa live-safe (itemIds assign, position, masteryThreshold) → in-place ngay. Sửa structural trên published → 409, bắt POST `/:id/new-version` (clone → v+1 draft).
- **IS-5** — POST `/:id/publish` draft → published. Re-validate toàn bộ. Block MCQ itemIds rỗng + cycle. Perm: **`maptemplate:manage`** (chỉ platform admin). POST `/:id/new-version` write (Content Team).
- **IS-6** — DELETE `/:id` soft-archive (status: draft→archived). Block nếu published cuối cùng của key (load-latest-template=null → 404 onboarding).
- **IS-7** — Nodest validation: nodeKey unique trong template, enum (CORE/PRACTICE/CHECKPOINT/REMEDIATION/SIDE_QUEST), skillTag ∈ categories, subskillTags free-string (không ràng SUBSKILLS — là taxonomy node).
- **IS-8** — Edge validation: from/to trỏ node tồn tại. **Mirror prerequisites:** cần mỗi edge a→b, b.prerequisites chứa a (gating chạy prerequisites, không edge).
- **IS-9** — ItemIds validation (async): chỉ mcq, active, centerId:null (shared bank). 1 query gom không N+1. Lỗi → error path `nodes[i].config.itemIds[j]`.
- **IS-10** — Cycle detect: warn ở draft, **BLOCK** khi publish.
- **IS-11** — Perm `maptemplate:read|write|delete|manage`. Content Team = `write` (draft/itemIds). Platform admin = `manage` (publish/dev-reset).
- **IS-12** — Dev utility POST `/:id/dev-reset-progress` (commit 814be85): xoá learner LearnerPath + ActivityResult + RemediationLoop (map-only, giữ StudentModel), hạ draft. Gated `maptemplate:manage` + **403 DEV_RESET_DISABLED** nếu production.

**F-TEMPLATE-SELECTION (S-C):**
- **IS-13** — Resolver `resolveTemplatesForLearner(StudentModel)` → match program (exact +2 / universal +1) + cover band (null=universal; mismatch → fallback demo) → trả 1 template hoặc []. Eligibility: template entry node phải UNLOCKED.
- **IS-14** — Program tag trên template (nullable, enum PROGRAM_KEYS, live-safe). Plumb create/update/new-version body whitelist, validate.
- **IS-15** — `ensureActivePath` dùng resolver thay hardcode `demo-foundation` → new learner nhận đúng template per target.
- **IS-16** — Fallback: không match → demo-foundation nếu tồn tại, else 404. Learner hiện tại (đã pinned version) không đổi.

### Ngoài phạm vi

- **Portal UI** (`exe-admin` map-template-builder.tsx) — **Tách spec riêng** (FE chỉ mock BE contract, không scope docs này).
- **Player engine thực** (MCQ/video/IPA/flashcard/writing/speaking) — **Ngoài scope**, tham chiếu chỉ đọc `activityRef` + `config`. Adapter plugin từng loại.
- **Course→Map generator** (course-driven generated map) — **Ngoài scope**, separate feature (`sourceCourseSlug` field chỉ để tracking provenance).
- **Enrollment entity** (learner chọn khóa) — **Ngoài scope** (Slice C dùng StudentModel.target preset, không FE chọn).
- **Versioning nâng cao** (undo, export/import, diff) — **Ngoài scope** Phase 1; admin re-seed nếu cần rollback.
- **Quantitative cap** (số template/node limit) — **Ngoài scope** (hiệu năng phase 3).
- **Xoá cứng** (hard delete) — **Ngoài scope** (soft archive chỉ).

---

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Content Team (quyền `maptemplate:write`) | Tạo `CourseMapTemplate` draft mới rồi soạn nodes/edges/itemIds trên portal | Dựng lộ trình gamified mà học viên chơi mà không cần dev terminal |
| Content Team | Gán câu hỏi từ bank vào node MCQ qua picker (chọn multi-id, validate) | Tái dùng curated MCQ thật thay vì seed fake |
| Content Team | Sửa itemIds / position / mastery-threshold sống — KHÔNG vợ 409 | Nhanh iterate nội dung live còn học viên học |
| Platform Admin (quyền `maptemplate:manage`) | Review draft, phát hành `POST /:id/publish` trên BE (UI nút Publish) | Đẩy template mới từ Content Team tới learning-map mà tất cả học viên mới nhận |
| Platform Admin | Xem warning (itemIds rỗng, cycle, lệch band) trên detail template | Chấp nhận hoặc sửa trước publish |
| Platform Admin | Post dev-reset trên map published (wipe learner + force draft) | Thoải mái sửa cấu trúc khi test |
| New learner (onboarding) | Hệ thống tự chọn template đúng per target (program+band) | Nhận map phù hợp tự động, không hardcode demo-foundation |
| Tech Lead (debug) | Xem health: item active >= MIN_ITEMS per node MCQ | Phát hiện node "hụt item" sau khi bank retire |

---

## 5. Quyết định nghiệp vụ đã chốt (owner, red-team, commit logs)

| Câu hỏi | Lựa chọn đã chốt | Verification |
|---|---|---|
| Sửa structural trên published: đè overwrite, tạo version, hay từ chối? | **Tạo version mới** (409 + buộc POST /new-version) — chỉ học viên mới nhận, học viên cũ pinned | plan.md LOCKED DECISIONS, red-team F5 chốt scope, 690df00 updateTemplate structural-sig logic |
| Publish gate: `write` hay `manage`? | **`manage`** (Content Team `write` chỉ draft, admin `manage` live) — đảm bảo platform control | plan-backend.md F20 chốt, 690df00 publisher route |
| ItemIds validation: centerId scope? | **centerId:null only** (shared bank) — chặn rò rỉ private cross-tenant | red-team F1 Crit, 690df00 validator line 99 |
| Flip curated↔simulator (itemIds rỗng←→không)? | **Structural → 409** (cấm live-flip trên published) | plan-backend.md F11/F3 chốt, validator warn draft/error publish empty MCQ |
| Template không matched learner: fallback? | **demo-foundation** (fallback default, hoặc 404 nếu cả demo hỏng) | plan.md "fallback demo-foundation", 6600767 resolver null check |
| Program tag scope (per-center hay global)? | **Global** (map KHÔNG có centerId, program = tag để resolver match) | plan-backend.md scope chốt, model.js default null (universal) |
| Edge chỉ để hiển thị hay gating? | **Render only** (gating chạy prerequisites, không edge). Edge phải mirror prereq. | red-team F2 Crit, validator edges-mirror-prereq check |
| Dev-reset prod-safe? | **Hard-block production** (403 DEV_RESET_DISABLED) dù quyền | plan.md LOCKED DECISIONS wipe, 814be85 line NODE_ENV check |

---

## 6. Acceptance criteria (tóm tắt)

Chi tiết ở `acceptance.md`. Mỗi dòng map `IS-N`.

1. Tạo template mới draft → hệ thống lưu + trả DTO complete. *(IS-1)*
2. Sửa itemIds/position/mastery trên draft → in-place OK. *(IS-2, IS-4)*
3. Sửa itemIds/position/mastery trên published → in-place OK (live-safe). *(IS-2, IS-4)*
4. Sửa node/edge structure trên draft → in-place OK. *(IS-2)*
5. Sửa node/edge structure trên published → 409 TEMPLATE_STRUCTURAL_LOCKED. *(IS-2, IS-4)*
6. ItemIds không-mcq hoặc inactive hoặc private (centerId≠null) → 422 validation error chỉ rõ path. *(IS-9)*
7. MCQ itemIds rỗng ở draft → warn (ok). Ở publish → 422 TEMPLATE_VALIDATION_FAILED. *(IS-10)*
8. Cycle prerequisite ở draft → warn. Ở publish → 422. *(IS-10)*
9. Edges không mirror prerequisites → 422. *(IS-8)*
10. POST publish gated `maptemplate:manage`; POST new-version gated `write`. *(IS-5, IS-11)*
11. DELETE archived (soft). Nếu published-cuối → 409 TEMPLATE_LAST_PUBLISHED. *(IS-6)*
12. Dev-reset xoá LearnerPath (map-only), hạ draft, giữ StudentModel. 403 nếu prod. *(IS-12)*
13. New learner → resolver chọn template per target (program+band), không hardcode. *(IS-15, IS-16)*
14. Template program-mismatch band → fallback demo, KHÔNG force map sai band. *(IS-13)*
15. Learner cũ (pinned version) KHÔNG ảnh hưởng khi admin đổi resolver logic. *(IS-16)*

---

## 7. Câu hỏi mở

Không còn (ĐÃ CHỐT hết red-team findings + owner decisions, spec hồi tố từ code thực).

---

## 8. Ghi chú cho xây dựng chức năng

> Ràng buộc kỹ thuật đã biết, KHÔNG phải giải pháp.

- **All-or-nothing validate**: 422 lỗi, 409 lock — KHÔNG viết riêng bộ nào. Đặc tả ở acceptance.md + design.md error-code mapping.
- **Structural-signature hash** (canonicalize trước, F9): node sort nodeKey, edges as set. Live-safe ≠ structural. Xem `map-template-writer.js:structural-sig logic + test case`.
- **Learner path pinning**: learner cũ ref (templateKey, version) snapshot — publish new-version KHÔNG retroactive (học viên mới nhận v+1, cũ ở v1). Verify `map-builder.loadLatestTemplate($nin draft/archived)`.
- **Shared bank only**: itemIds picker `centerId:null` + validator `ObjectId.isValid` cap. By-ids endpoint test cross-tenant rejection (6600767 by-ids FE trà answerKey safe ⟹ learner surface strip).
- **Dev-reset scope**: MAP-ONLY (LearnerPath/ActivityResult/RemediationLoop), GIỮ StudentModel (submit sau vẫn resolve). XOÁ LearningEvent khiến submit vỡ, KHÔNG xoá.
- **Template entry-node health**: publish-validator block no-root (`NO_ENTRY_NODE`). List/detail trả `itemsHealth` per node (item-count / MIN_ITEMS) để admin thấy retire-impact.
- **Program enum**: PROGRAM_KEYS (`constants/programs.js`), default null (universal). Match resolver: exact+2 > universal+1 > fallback.
- **CEFR range**: floor/ceiling ∈ CEFR_LEVELS | null. null = cover-all band.

---

## 9. Cross-reference (sibling specs)

- **`specs/learning-map-content-players/`** — Node activity player engines (MCQ/video/IPA/flashcard thực). Map template chỉ ref `activityRef` + `config`, không ownership activity.
- **`specs/course-driven-map/`** — Course→Map generator (sourceCourseSlug track). S-B admin là **hand-authored map** (generated=false).
- **`specs/adaptive-learning-path/`** — StudentModel band/target resolve logic. S-C resolver phụ thuộc StudentModel.target.program/cefr.

---

## 10. Unresolved (từ plan, không blocking)

- (Owner action) Tạo role bundle "Content Team" gán `maptemplate:write` ở ops step (BE chỉ thêm perm constant).

