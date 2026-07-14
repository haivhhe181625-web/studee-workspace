<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8 -->
# Báo cáo Migration — Cross-repo References (2026-07-14)

> Đi kèm `docs/cross-repo-linking.md` (quy ước). File này là **inventory chi tiết**: mọi tham chiếu bị "gãy" hoặc
> đặc thù-repo được phát hiện khi 39 file AI-SDD workspace chuyển từ `exe-api` sang `studee-workspace`, và xử lý
> cụ thể cho từng loại. Không dùng để tra cứu hằng ngày — dùng khi cần audit lại hoặc mở rộng sang `exe-web`/
> `exe-admin`.

## 1. Phương pháp

1. Grep toàn bộ 39 file vừa chuyển, tìm token khớp pattern đường dẫn/tên file đặc thù-repo (`docs/*.md`,
   `CLAUDE.md`, `TESTING.md`, `services/*`, `.github/*`, tên repo `exe-web`/`exe-admin`/`studee-env-setup`, và
   tham chiếu nội bộ `.ai/`/`templates/`/`specs/`/`docs/adr/`/`docs/architecture/`).
2. Phân loại mỗi token: **(A) rõ ràng thuộc `<API_REPO>`** → convert tự động; **(B) nội bộ `studee-workspace`**
   → không convert (xem lý do ở `docs/cross-repo-linking.md` §3); **(C) mơ hồ/dùng chung nhiều repo** → giữ
   nguyên + annotate; **(D) tên repo trần** (không kèm path) → giữ nguyên, không phải "reference" cần resolve;
   **(E) lịch sử** (2 báo cáo point-in-time) → banner, không convert nội dung.
3. Convert nhóm A bằng `sed` một lượt (14 pattern, đã xác minh không chồng lấn substring — xem §4), verify bằng
   grep tìm double-prefix (`<API_REPO>/<API_REPO>`) → không có kết quả, sạch.
4. Xử lý tay 5 trường hợp đặc biệt không thể mechanical: `docs/adr/0000-index.md` (link Markdown hỏng),
   `.ai/workflows/release-workflow.md` (`.github/workflows/*.yml` + `studee-env-setup/README.md`),
   `.ai/project-context.md` §0 (mô tả repo sai hoàn toàn sau khi workspace chuyển ra), 2 báo cáo lịch sử.

## 2. Kết quả — nhóm A (đã convert tự động)

**153 lần thay thế `<API_REPO>/...`** trên **34/39 file**. Không phát hiện double-prefix hay lỗi chồng lấn khi
verify.

| Pattern gốc | Thay bằng | Số file có mặt |
|---|---|---|
| `docs/ARCHITECTURE.md` | `<API_REPO>/docs/ARCHITECTURE.md` | nhiều |
| `docs/CONVENTIONS.md` | `<API_REPO>/docs/CONVENTIONS.md` | nhiều |
| `docs/ONBOARDING.md` | `<API_REPO>/docs/ONBOARDING.md` | ít |
| `docs/api/CONVENTIONS.md` | `<API_REPO>/docs/api/CONVENTIONS.md` | nhiều |
| `docs/api/ARCHITECTURE.md` | `<API_REPO>/docs/api/ARCHITECTURE.md` | 2 |
| `docs/llm/ARCHITECTURE.md` | `<API_REPO>/docs/llm/ARCHITECTURE.md` | 2 |
| `docs/ADR-admin-api-migration.md` | `<API_REPO>/docs/ADR-admin-api-migration.md` | 5 |
| `docs/superpowers/specs/`, `docs/superpowers/plans/` | `<API_REPO>/docs/superpowers/...` | 8 |
| `docs/active/<feature>/` | `<API_REPO>/docs/active/<feature>/` | 2 |
| `CLAUDE.md` | `<API_REPO>/CLAUDE.md` | nhiều |
| `TESTING.md` | `<API_REPO>/TESTING.md` | 5 |
| `services/api...`, `services/cat...`, `services/llm...` | `<API_REPO>/services/api...` (bare lẫn có path con) | nhiều |

Danh sách đầy đủ theo file (số lần `<API_REPO>` xuất hiện sau convert):

```
.ai/project-context.md            30   .ai/agents/developer-agent.md      13
.ai/workflows/feature-workflow.md  6   .ai/workflows/release-workflow.md   6
.ai/agents/techlead-agent.md       6   .ai/prompts/update-documentation.md 4
.ai/context/README.md              4   docs/adr/0000-index.md              4
.ai/prompts/generate-tests.md      3   docs/adr/adr-template.md            3
docs/architecture/README.md        3   templates/design-template.md        3
specs/example-feature/design.md    3   .github/ISSUE_TEMPLATE/bug-report.md 3
specs/README.md                    3   .ai/checklists/developer-checklist.md 2
.ai/prompts/generate-contract.md   2   .ai/prompts/implement-task.md       2
.ai/workflows/review-workflow.md   2   specs/example-feature/spec.md       2
specs/example-feature/tasks.md     2   templates/spec-template.md          2
templates/tasks-template.md        2   .ai/agents/ba-agent.md               2
.ai/agents/reviewer-agent.md       2   (còn lại: 1 mỗi file — 12 file khác)
```

## 3. Kết quả — nhóm B (nội bộ `studee-workspace`, không convert)

**~263 tham chiếu** tới `.ai/...`, `templates/...`, `specs/...`, `docs/adr/...`, `docs/architecture/...` trên 44
vị trí file. **Không convert máy móc** — lý do và quy tắc resolve đầy đủ ở `docs/cross-repo-linking.md` §3. Tóm
tắt: đúng nguyên trạng khi đọc từ trong `studee-workspace`; cộng thêm `<WORKSPACE>/` khi đọc từ repo khác.

**Rủi ro đã tránh bằng cách không convert nhóm này:** nhiều file (`specs/README.md`, `templates/spec-template.md`,
`templates/design-template.md`, `.ai/project-context.md`, `.ai/workflows/feature-workflow.md`) chứa **cả** tham
chiếu tới `docs/superpowers/specs/` (nhóm A, thuộc `<API_REPO>`) **và** tham chiếu tới `specs/<feature-slug>/...`
(nhóm B, thuộc `<WORKSPACE>`) trong cùng file — 2 chuỗi này chia sẻ substring `specs/`. Một sed pass toàn cục cho
`specs/` → `<WORKSPACE>/specs/` sẽ vô tình khớp cả bên trong `<API_REPO>/docs/superpowers/specs/` đã convert,
tạo ra `<API_REPO>/docs/superpowers/<WORKSPACE>/specs/` — sai. Đây là lý do chính khiến nhóm B được xử lý bằng
quy tắc-đọc-theo-cwd thay vì convert chuỗi.

## 4. Kết quả — nhóm C (mơ hồ, giữ nguyên + annotate)

| Token | Xuất hiện ở | Vì sao không convert |
|---|---|---|
| `README.md` | `.ai/prompts/update-documentation.md` | Cả 4 repo đều có README riêng |
| `.env.example` | `.ai/checklists/developer-checklist.md`, `.ai/checklists/release-checklist.md` | Mỗi repo/service có file riêng |
| `.github/CODEOWNERS` | `.ai/workflows/feature-workflow.md`, `.ai/checklists/release-checklist.md`, `.ai/workflows/review-workflow.md` | Quy tắc chung cho mọi repo, không riêng 1 cái (khác `release-workflow.md` §2 — ở đó đã convert vì ngữ cảnh cụ thể là pipeline `<API_REPO>`) |
| `src/constants/permissions.js`, `src/admin-api/`, `src/adapters/` | Rải rác trong `.ai/agents/*`, `.ai/checklists/*` | Quy ước path-rút-gọn có sẵn từ tài liệu gốc (ngầm định "trong `services/api/`") — giữ nguyên để không lấn sang sửa nội dung convention, ngoài phạm vi migration này |

Đầy đủ vị trí từng token: chạy `grep -rn "<token>" docs/ .ai/ templates/ specs/ .github/` trong `studee-workspace`.

## 5. Trường hợp xử lý tay (không thuộc pattern chung)

| File | Vấn đề | Xử lý |
|---|---|---|
| `docs/adr/0000-index.md` | Link Markdown `[...](../ADR-admin-api-migration.md)` — đúng khi `docs/adr/` còn trong `exe-api`, giờ trỏ sai (resolve về `studee-workspace/ADR-admin-api-migration.md`, không tồn tại) | Đổi target thành `<API_REPO>/docs/ADR-admin-api-migration.md` + thêm ghi chú "không bấm được" |
| `.ai/workflows/release-workflow.md` | `.github/workflows/*.yml`, `.github/workflows/deploy-dev.yml`, `deploy.yml`, `restrict-main-pr.yml`, `.github/CODEOWNERS` (§2 Cut a release) | Convert thành `<API_REPO>/...` — ngữ cảnh cụ thể (pipeline deploy thật), khác nhóm C |
| `.ai/workflows/release-workflow.md` | `studee-env-setup/README.md` | Convert thành `<ENV_BUNDLE>/README.md` |
| `.ai/project-context.md` §0 | Toàn bộ mô tả "repo này là `exe-api`, 1 trong 3 repo..." sai hoàn toàn sau khi workspace chuyển ra khỏi `exe-api` | Viết lại (không chỉ thay placeholder) — thêm dòng `<WORKSPACE>` vào bảng repo, giải thích quy tắc resolve theo cwd |
| `docs/project-analysis.md` | Toàn bộ nội dung (cây thư mục, path) mô tả `exe-api` tại thời điểm viết | Thêm banner lịch sử ở đầu, KHÔNG sửa nội dung bên trong |
| `docs/ai-workspace-report.md` | Tương tự — mục 3 "Cấu trúc thư mục (kết quả)" mô tả layout đã lỗi thời ngay trong ngày viết | Thêm banner lịch sử ở đầu, KHÔNG sửa nội dung bên trong |

## 6. Không đụng tới (đúng, không cần xử lý)

Tên repo trần trong văn xuôi (`exe-api`, `exe-web`, `exe-admin`, `studee-env-setup` không kèm path phía sau — vd
bảng tech stack, checkbox "- [ ] `exe-web`" trong issue template, participant name trong sequence diagram ở
`specs/example-feature/diagrams/README.md`) — đây là **tên định danh**, không phải path cần resolve, nên giữ
nguyên theo đúng tinh thần "chỉ annotate/convert reference, không đổi tên repo thành placeholder ở mọi chỗ".

## 7. Việc còn lại (không nằm trong migration này)

- `<WEB_REPO>`/`<ADMIN_REPO>` hiện chưa có tham chiếu thật nào dùng tới (chưa có file nào trong workspace trỏ
  path cụ thể vào `exe-web`/`exe-admin`) — placeholder đã định nghĩa sẵn ở `docs/cross-repo-linking.md`, sẽ dùng
  khi workspace mở rộng sang 2 repo đó.
- `docs/api/CONVENTIONS.md` trong `<API_REPO>` vẫn còn lỗi thời (rule AdminJS cũ) — đã ghi nhận từ
  `docs/project-analysis.md`, không thuộc phạm vi migration link.
