# Phân tích Repository — `exe-api` (pilot cho AI-SDD workspace)

> ⚠️ **Ghi chú lịch sử (2026-07-14):** tài liệu này được viết **khi workspace AI-SDD còn nằm bên trong `exe-api`**,
> mọi đường dẫn trong bài (`docs/ARCHITECTURE.md`, `CLAUDE.md`, `services/api/...`, cây thư mục ở mục 3, v.v.) là
> đường dẫn **tương đối trong `exe-api` tại thời điểm đó** — giữ nguyên, không viết lại theo placeholder
> `<API_REPO>`/`<WORKSPACE>` vì đây là bản ghi hiện trạng tại một thời điểm, không phải tài liệu vận hành sống.
> Ngay sau khi viết xong, `.ai/`, `specs/`, `templates/`, `docs/adr/`, `docs/architecture/` và chính file này đã
> được **chuyển sang repo `studee-workspace`**. Quy ước link cross-repo hiện hành nằm ở `docs/cross-repo-linking.md`.

> **Ngày:** 2026-07-14 · **Phạm vi:** repo `exe-api` (được chọn làm pilot). Không sửa code trong tài liệu này.
> **Bối cảnh:** `exe-api` là 1 trong 3 git repo độc lập của hệ thống ENOVA/studee (`exe-api`, `exe-web`, `exe-admin`),
> cộng thêm `studee-env-setup` (bundle `.env`, không phải git repo). Ba repo **không** dùng chung version control,
> không có monorepo tooling (Nx/Turborepo/Lerna) nối chúng lại — mỗi repo có `CLAUDE.md`, git flow, CI riêng nhưng
> theo cùng một khuôn (rất có thể do cùng một quy trình chuẩn hoá). Tài liệu này chỉ phân tích `exe-api`; hai repo
> kia và `studee-env-setup` được nêu ở mức tối thiểu để làm rõ ranh giới hệ thống.

## 1. Tổng quan dự án

**ENOVA/studee** là nền tảng luyện thi/luyện nói tiếng Anh có AI: Speaking 1-1 với AI, luyện phát âm IPA, bài thi
đầu vào/đánh giá năng lực thích ứng (adaptive/MST), quản lý trung tâm (multi-tenant B2B). `exe-api` là backend,
tổ chức theo dạng **monorepo nhiều service** (không dùng công cụ monorepo, chỉ là 3 folder độc lập dưới
`services/`, orchestrate bằng Docker Compose):

| Service | Stack | Vai trò |
|---|---|---|
| `services/api` | Node 20 + Express 4 + MongoDB (Mongoose 8) + Redis + Socket.IO + BullMQ | Business logic chính — auth, user, assessment (MST/IRT), payment, course, IPA, admin REST |
| `services/cat` | Python (nội bộ, gọi qua `X-Service-Key`) | Adaptive Testing engine — IRT (`irt.py`) + Multi-Stage Testing (`mst.py`) |
| `services/llm` | Python 3.11 + FastAPI + httpx + Redis | Anti-Corruption Layer bọc OpenAI (chat/ASR/TTS/embeddings/moderation) |

## 2. Kiến trúc hiện tại

- **Triết lý:** "microservice nhẹ" — tách theo blast-radius + workload (Node cho I/O/business, Python cho
  ML/audio), không phải distributed monolith cũng không phải full microservice.
- **Giao tiếp:** REST thuần trên toàn hệ thống — không có GraphQL/gRPC. Cross-service (`api ↔ llm`, `api ↔ cat`)
  qua HTTP nội bộ (Docker DNS) xác thực bằng `X-Service-Key` (shared secret), khác với JWT dùng cho client-facing.
- **Auth:** Firebase Authentication (OAuth) ở FE → đổi lấy JWT nội bộ do `api` phát hành (access + refresh) →
  RBAC bằng registry phẳng `<resource>:<action>` (`src/constants/permissions.js`) + tenant scope qua
  `User.centerId` cho center staff.
- **Admin surface:** vừa trải qua một cuộc migration lớn — từ AdminJS (rule cũ) sang REST tại `/api/admin/*`
  + portal Next.js riêng `exe-admin` (xem `docs/ADR-admin-api-migration.md`, đã hoàn tất Phase 5, 03/07/2026).
  Admin REST bắt buộc **không chứa business logic**, phải delegate về `src/modules/<feature>/*.service.js`.
- **Module pattern nhất quán:** mỗi domain (`auth`, `user`, `assessment`, `payment`, `ipa`, `course`, `plan`,
  `subscription`, `talk`, `roadmap`, ...) theo đúng 5-file pattern `model / schema / service / controller / routes`.
- **Data:** MongoDB là datastore chính (không có DB quan hệ), Redis cho cache/rate-limit/Socket.IO/BullMQ, Qdrant
  làm vector store (dedup câu hỏi qua embedding).
- **Triển khai:** một VPS duy nhất, Docker Compose (`docker-compose.yml`/`.dev.yml`/`.prod.yml`), deploy qua
  GitHub Actions SSH khi push `develop` (dev) hoặc tag `v*` (prod). Không có Kubernetes/managed cloud.

## 3. Cấu trúc thư mục (rút gọn)

```
exe-api/
├── CLAUDE.md, README.md, TESTING.md, update-key.txt
├── docker-compose*.yml
├── docs/
│   ├── ARCHITECTURE.md, CONVENTIONS.md, ONBOARDING.md, ADMIN_GUIDE.md
│   ├── ADR-admin-api-migration.md          # 1 file, KHÔNG có folder docs/adr/
│   ├── api/ (ARCHITECTURE.md, CONVENTIONS.md, ONBOARDING.md, auth/, level-test/, onboarding/, roadmap/)
│   ├── llm/ (ARCHITECTURE.md, ONBOARDING.md)
│   ├── active/<feature>/          # roadmap nhiều giai đoạn đang chạy, có .status
│   ├── superpowers/
│   │   ├── specs/<date>-<feature>-design.md    # spec/design 1 file
│   │   └── plans/<date>-<feature>.md           # plan TDD checkbox, 1 file
│   ├── analysis/, security/
├── scripts/, storage/, tests/ (load, memory)
└── services/{api, cat, llm}/...
```

## 4. Tech stack

**services/api:** Express 4, Mongoose 8, Redis (`redis` + `@socket.io/redis-adapter`), Socket.IO, BullMQ,
Firebase Admin SDK, JWT, Joi, bcryptjs, Qdrant client, multer + sharp, Resend, Helmet, express-rate-limit,
Winston. Dev: Jest 30, Supertest, mongodb-memory-server, ioredis-mock, nodemon. **Không có ESLint/Prettier.**

**services/cat / services/llm:** Python, pytest, FastAPI (llm), IRT/MST thuần Python (cat). `llm` từng có adapter
Anthropic Claude + Azure Speech nhưng **đã gỡ** (scoring chuyển sang "scoring engine" repo riêng, không nằm trong
workspace này).

## 5. Tài liệu hiện có (đánh giá)

`exe-api` có tài liệu **tốt hơn mức trung bình** cho một dự án quy mô này:

- `CLAUDE.md` — quy tắc git, ngôn ngữ, "Karpathy" behavioral guidelines, hard rules cho admin surface.
- `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md`, `docs/ONBOARDING.md` (~10 phút setup, bảng troubleshoot) — chất
  lượng tốt, súc tích.
- `docs/ADR-admin-api-migration.md` — 1 ADR duy nhất, format tốt (bối cảnh/quyết định/lộ trình/hệ quả) nhưng
  **không nằm trong folder `docs/adr/`** và không có số thứ tự (ADR-0001...) — không scale nếu có ADR thứ 2.
- `docs/active/<feature>/` — roadmap nhiều giai đoạn (GĐ1-GĐ4) với file `.status` theo dõi tiến độ. Đây là một
  dạng "spec sống" phi chính thức, khá gần với khái niệm `specs/` mà AI workspace sẽ chuẩn hoá.
- `docs/superpowers/specs/` + `docs/superpowers/plans/` — **phát hiện quan trọng**: repo đã có sẵn một quy trình
  spec-driven kiểu Superpowers (1 file spec/design + 1 file plan dạng checklist TDD `- [ ] Step ... Run ...
  Expected ...` + mục Self-Review cuối plan đối chiếu lại spec). Tuy nhiên **chỉ có đúng 1 feature** đã dùng pattern
  này (`user-profile-edit-avatar`, 2026-06-15) — chưa được áp dụng lại, không có template, không tách rõ vai trò
  BA/Tech Lead/Dev, không có bước "BA review" hay "Technical review" độc lập trước khi viết plan.
- `docs/api/CONVENTIONS.md` — **đã lỗi thời**: vẫn mô tả rule cũ "KHÔNG viết API admin, mọi thứ qua AdminJS",
  trực tiếp mâu thuẫn với `CLAUDE.md` (đã supersede rule này 03/07/2026) và `docs/ADR-admin-api-migration.md`
  (AdminJS đã bị gỡ hoàn toàn). Đây là rủi ro thật — một dev hoặc AI agent đọc file này trước sẽ làm sai.

## 6. API style

REST thuần, không GraphQL/gRPC. Router theo module (`<feature>.routes.js`), middleware chain chuẩn
(`verifyToken → verifyPermission → validate(schema) → asyncHandler(controller.fn)`). Admin REST dùng CRUD factory
generic (`_crud.factory.js` + config resource) thay vì hand-roll từng route.

## 7. Coding conventions

Không có ESLint/Prettier/editorconfig ở `exe-api` (khác với `exe-web`/`exe-admin` — cả hai đều có
`eslint.config.mjs`). Convention được ghi bằng văn xuôi trong `CLAUDE.md` + `docs/CONVENTIONS.md` +
`docs/api/CONVENTIONS.md`: naming table (kebab-case file, camelCase function, PascalCase model, UPPER_SNAKE_CASE
const/env), 5-file module pattern, response luôn qua `toDto()`, error qua `ApiError` + `asyncHandler`, external
call luôn qua adapter (không `require('openai')` trực tiếp trong service).

## 8. Testing framework

Jest 30 + Supertest + mongodb-memory-server cho `api` (259/259 test pass theo `TESTING.md`, coverage tập trung
vào module `assessment`); pytest cho `cat` (73/73) và `llm` (103/103), chạy trong container. k6 cho load test,
script Node `--expose-gc` cho memory-leak check. Không có test framework nào trong `exe-web`/`exe-admin` được cấu
hình liên kết CI với `exe-api` (E2E Playwright ở `exe-web` bị skip, cần backend sống).

## 9. Build tools / CI-CD

Docker (per-service `Dockerfile` + 3 `docker-compose*.yml`). GitHub Actions: `deploy-dev.yml` (push `develop` →
SSH deploy dev), `deploy.yml` (tag `v*` → SSH deploy prod), `restrict-main-pr.yml` (tự đóng PR vào `main` từ
non-lead). `.github/CODEOWNERS` yêu cầu approval `@dotuananhtb` cho mọi thứ. Không có workflow chạy test/lint
tự động trên PR (`npm test`/`pytest` không nằm trong bất kỳ `.github/workflows/*.yml` nào hiện có) — CI hiện tại
chỉ lo deploy, không gate chất lượng.

## 10. Branch strategy

3-branch flow: `main` (prod, chỉ lead) / `develop` (integration, PR-only) / `feature|fix|chore|docs|refactor/*`
(dev tự push). Conventional Commits. Remote chỉ có `main` + `develop` (sạch, không branch rác) — khác biệt rõ so
với `exe-web` (có ~35 branch feature còn sót lại chưa dọn).

---

## 11. Điểm mạnh (Strengths)

1. Tài liệu kiến trúc/convention/onboarding đã tồn tại và có chất lượng tốt — không phải xây từ số 0.
2. Đã có mầm mống quy trình spec-driven (`docs/superpowers/specs` + `plans`) và roadmap sống
   (`docs/active/<feature>/.status`) — chỉ cần chuẩn hoá & nhân rộng, không cần phát minh lại.
3. Quy tắc bảo mật/RBAC/tenant-scope được viết rõ ràng và nhất quán giữa `CLAUDE.md`, ADR, và code convention.
4. Git flow 3-nhánh + CODEOWNERS + auto-close PR sai nhánh đã enforce kỷ luật branch tốt.
5. Test suite Node + Python đã đạt coverage cao ở phần lõi (assessment/MST) — nền tảng tốt để AI agent tự tin
   thay đổi code có test guard.
6. Ranh giới service rõ (Anti-Corruption Layer cho `llm`, adapter pattern cho external call) — dễ viết spec/contract
   theo từng service.

## 12. Điểm yếu (Weaknesses)

1. **Tài liệu bị trùng lặp và lệch nhau**: `docs/CONVENTIONS.md` (monorepo) vs `docs/api/CONVENTIONS.md`
   (per-service) mô tả 2 phiên bản khác nhau của cùng một quy tắc; `docs/api/CONVENTIONS.md` còn chứa rule đã bị
   ADR supersede. Không có cơ chế nào đánh dấu file nào là "nguồn sự thật".
2. **Không có lint/format tự động** cho `services/api` (JS) hay 2 service Python — convention chỉ tồn tại dưới
   dạng văn bản, không được enforce bằng công cụ, dễ trôi theo thời gian.
3. **CI không gate chất lượng** — không workflow nào chạy test/lint trên PR; `TESTING.md` mô tả số liệu test
   nhưng đó là kết quả chạy thủ công, không phải bằng chứng tái lập được từ CI.
4. **Spec-driven workflow chưa được thể chế hoá** — chỉ 1 feature từng dùng pattern spec+plan, không có template,
   không có vai trò BA tách biệt, không có gate review trước khi code.
5. **ADR không có nơi chốn chuẩn** — 1 file rời trong `docs/`, không đánh số, sẽ không scale khi có ADR thứ 2, 3.
6. **Không có issue template** trên GitHub — không có cấu trúc chuẩn để BA/PM mở yêu cầu tính năng hay bug report.
7. **Ranh giới cross-repo (`exe-api` ↔ `exe-web`/`exe-admin`) không có tài liệu hợp đồng chính thức** — API
   contract giữa BE và 2 FE chỉ nằm rải rác trong `docs/api/<endpoint>/` và trong code, không có
   `contracts/` tập trung, không versioning rõ ràng khi breaking change.

## 13. Thiếu tài liệu (Missing documentation)

- Không có tài liệu **BA-facing** (spec template, acceptance criteria template) — mọi tài liệu hiện có đều là
  tài liệu kỹ thuật viết bởi/cho dev.
- Không có **API contract tập trung** (OpenAPI/JSON schema) cho REST `services/api` — chỉ có markdown rải rác.
- Không có tài liệu **cross-repo** giải thích quan hệ `exe-api` ↔ `exe-web` ↔ `exe-admin` ở một chỗ (mỗi repo tự
  mô tả 1 phần, không repo nào có bức tranh tổng).
- `docs/ADMIN_GUIDE.md`, `docs/analysis/dgnl`, `docs/security/audit-2026-06-09.md` tồn tại nhưng không được liệt
  kê ở README/ONBOARDING — dev mới dễ bỏ sót.

## 14. Thiếu chuẩn (Missing standards)

- Không có ESLint/Prettier cho `services/api`.
- Không có convention đặt tên cho file spec/plan (file hiện tại dùng `<date>-<slug>[-design].md`, chưa chuẩn hoá,
  chưa có trạng thái máy đọc được ngoài `.status` tự do trong `docs/active/`).
- Không có template review/PR có cấu trúc máy đọc được (hiện là 1 khối markdown trong `docs/CONVENTIONS.md`).
- Không có ADR numbering/index.
- Không có chính sách rõ ràng cho "khi nào cần spec trước khi code" — hiện tại tuỳ nghi (feature nhỏ như avatar
  upload có spec, nhiều PR khác không có gì ngoài mô tả trong commit).

## 15. Khuyến nghị (Recommendations)

1. **Chuẩn hoá, không thay thế** quy trình `docs/superpowers/specs`+`plans` đã có — nâng thành `specs/<feature>/`
   với spec/design/tasks/research/acceptance/contracts tách file, có template, áp dụng vai trò BA → Tech Lead →
   Dev → Reviewer rõ ràng (xem `.ai/workflows/feature-workflow.md`).
2. **Gộp `docs/CONVENTIONS.md` và `docs/api/CONVENTIONS.md`** thành một nguồn sự thật, xoá phần rule AdminJS đã
   lỗi thời khỏi `docs/api/CONVENTIONS.md` (việc này KHÔNG nằm trong phạm vi tài liệu này — chỉ ghi nhận, không
   sửa; xem Phase ngoài phạm vi ở `docs/ai-workspace-report.md`).
3. Tạo `docs/adr/` có đánh số (`0001-...md`) và index; giữ nguyên `docs/ADR-admin-api-migration.md` tại chỗ, chỉ
   thêm liên kết từ index mới — không di chuyển file cũ (giữ backward compatibility).
4. Thêm `.github/ISSUE_TEMPLATE/` cho feature-request (liên kết tới spec-template) và bug-report.
5. Cân nhắc (ngoài phạm vi AI workspace, để tech lead quyết định riêng) thêm ESLint + 1 GitHub Actions job chạy
   `npm test`/`pytest` trên PR để CI thực sự gate chất lượng thay vì chỉ gate deploy.
6. Khi pattern ổn định trên `exe-api`, nhân rộng `.ai/`/`templates/`/`specs/` sang `exe-web`/`exe-admin` (hoặc
   tách thành 1 repo workspace dùng chung) — xem lộ trình ở `docs/ai-workspace-report.md`.

*(Mục 5–6 là khuyến nghị, không phải hành động đã thực hiện — tài liệu này không sửa code hay conventions hiện
có.)*
