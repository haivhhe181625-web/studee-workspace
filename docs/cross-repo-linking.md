<!-- Tiếng Việt — dưới docs/, theo .ai/project-context.md §8 -->
# Quy ước link cross-repo

> **Bối cảnh:** Ngày 2026-07-14, toàn bộ hạ tầng AI-SDD (`.ai/`, `specs/`, `templates/`, `docs/adr/`,
> `docs/architecture/`, `docs/project-analysis.md`, `docs/ai-workspace-report.md`) được tạo thử nghiệm **bên
> trong `exe-api`**, sau đó **chuyển sang repo riêng `studee-workspace`** (repo này) cùng ngày. Việc chuyển làm
> hỏng mọi đường dẫn tương đối từng đúng khi mọi thứ nằm chung 1 repo. Tài liệu này định nghĩa quy ước link mới,
> repo-agnostic, để `studee-workspace` không phụ thuộc cứng vào layout của bất kỳ repo code nào.

## 1. Vì sao cần placeholder thay vì đường dẫn tương đối thật

`studee-workspace` điều phối **4 thư mục/repo độc lập** nằm cạnh nhau trên máy (không có git history chung,
không có monorepo tool):

```
<thư mục cha bất kỳ>/
├── exe-api/            ← <API_REPO>
├── exe-web/             ← <WEB_REPO>
├── exe-admin/            ← <ADMIN_REPO>
├── studee-workspace/      ← <WORKSPACE>  (repo này)
└── studee-env-setup/       ← <ENV_BUNDLE>  (không phải git repo)
```

Layout cạnh-nhau (sibling) này là **quy ước cục bộ hiện tại**, không phải điều gì `studee-workspace` có thể tự
đảm bảo — mỗi người/mỗi máy CI có thể clone 4 repo vào vị trí khác nhau. Vì vậy tài liệu trong `studee-workspace`
**không viết đường dẫn tương đối kiểu `../exe-api/docs/ARCHITECTURE.md`** (giả định vị trí cụ thể), mà dùng
**placeholder tượng trưng** — người đọc hoặc agent tự thay bằng đường dẫn thật trên máy của họ.

## 2. Bảng placeholder

| Placeholder | Trỏ tới | Ghi chú |
|---|---|---|
| `<API_REPO>` | `exe-api` | Backend — duy nhất, không mơ hồ |
| `<WEB_REPO>` | `exe-web` | Frontend learner — hiện chưa có tài liệu nào trong workspace trỏ tới path cụ thể bên trong (chỉ nhắc tên repo) |
| `<ADMIN_REPO>` | `exe-admin` | Frontend admin — tương tự `<WEB_REPO>` |
| `<WORKSPACE>` | `studee-workspace` (repo này) | Dùng khi một tài liệu **ngoài** `studee-workspace` (vd trong `<API_REPO>`) cần trỏ ngược vào đây |
| `<ENV_BUNDLE>` | `studee-env-setup` | Không phải git repo — bundle `.env` cục bộ dùng để onboard |

**Cách đọc:** `<API_REPO>/docs/ARCHITECTURE.md` nghĩa là "file `docs/ARCHITECTURE.md` bên trong repo mà bảng trên
gọi là `<API_REPO>` (tức `exe-api`)" — thay bằng đường dẫn thật theo layout máy bạn, ví dụ `../exe-api/docs/
ARCHITECTURE.md` nếu 4 thư mục là sibling như minh hoạ ở mục 1.

**Đây KHÔNG phải link bấm được** trong Markdown/GitHub — kể cả khi cú pháp trông giống `[text](<API_REPO>/...)`
(xem ví dụ đã sửa ở `docs/adr/0000-index.md`). Đây là quy ước ký hiệu cho người đọc và cho AI agent tự resolve.

## 3. Quy tắc resolve theo cwd (quan trọng cho AI agent)

Tài liệu trong `.ai/` (agent, prompt, checklist) thường được một AI agent đọc **trong khi cwd của agent đó là một
repo code** (vd Developer Agent chạy `npm test` bên trong `<API_REPO>/services/api`), **không phải** trong
`studee-workspace`. Vì vậy:

- Mọi tham chiếu tới file/thư mục **bên trong repo code** (`docs/ARCHITECTURE.md`, `CLAUDE.md`,
  `services/api/...`, ...) đã được viết lại thành `<API_REPO>/...` trong đợt migration này (xem
  `docs/cross-repo-migration-report.md`) — vì các tham chiếu này **luôn** cần resolve về repo code, bất kể agent
  đang cwd ở đâu.
- Mọi tham chiếu tới file/thư mục **bên trong chính `studee-workspace`** (`.ai/...`, `templates/...`,
  `specs/...`, `docs/adr/...`, `docs/architecture/...`) **CHƯA** được viết lại thành `<WORKSPACE>/...` một cách
  máy móc — vì khi tài liệu đó được đọc từ bên trong `studee-workspace`, đường dẫn tương đối vẫn đúng nguyên
  trạng. Quy tắc thực tế: **nếu cwd của bạn là `studee-workspace`, đọc các tham chiếu này y nguyên (tương đối).
  Nếu cwd của bạn là một repo code khác (`<API_REPO>` chẳng hạn), tự thêm `<WORKSPACE>/` vào trước.**
- Lý do không mechanical-convert toàn bộ tham chiếu nội bộ: nhiều chuỗi như `specs/` là substring của cả tham
  chiếu nội bộ hợp lệ (`specs/<feature-slug>/spec.md`) lẫn tham chiếu cũ tới `<API_REPO>/docs/superpowers/specs/`
  — chuyển đổi máy móc toàn bộ có rủi ro double-prefix / sai ngữ cảnh cao hơn lợi ích, trong khi rule ở trên đã
  đủ rõ ràng cho người đọc và agent.

## 4. Trường hợp cố ý KHÔNG chuyển đổi (annotate thay vì convert)

Một số tham chiếu **mơ hồ theo thiết kế** — mỗi repo trong 4 repo đều có file cùng tên — nên KHÔNG được gán cứng
`<API_REPO>` (sẽ sai ngữ cảnh nếu ai đó áp dụng checklist/workflow này cho `<WEB_REPO>`/`<ADMIN_REPO>` sau này):

| Token | Vì sao mơ hồ | Xử lý |
|---|---|---|
| `README.md` | Cả 4 repo đều có README riêng | Giữ nguyên bare — người đọc tự hiểu theo ngữ cảnh câu văn |
| `.env.example` | Mỗi repo/service có `.env.example` riêng | Giữ nguyên bare |
| `.github/CODEOWNERS` (trong `.ai/workflows/feature-workflow.md`, `.ai/checklists/release-checklist.md`, `.ai/workflows/review-workflow.md`) | Cả 3 repo code đều enforce CODEOWNERS riêng theo cùng cơ chế — quy tắc là chung, không riêng 1 repo | Giữ nguyên bare (khác với trong `release-workflow.md`, nơi ngữ cảnh cụ thể là pipeline deploy thật của `<API_REPO>` — ở đó ĐÃ convert thành `<API_REPO>/.github/CODEOWNERS`) |
| `src/constants/permissions.js`, `src/admin-api/` (trong các file `.ai/agents/techlead-agent.md`, `.ai/checklists/techlead-checklist.md`, `.ai/agents/reviewer-agent.md`, ...) | Đường dẫn viết tắt tương đối tới `services/api/` (quy ước có sẵn từ `docs/api/CONVENTIONS.md` gốc, viết như đang đứng trong service đó) — không phải path đầy đủ từ root repo | Giữ nguyên như tài liệu gốc, KHÔNG viết lại thành `<API_REPO>/services/api/src/...` để tránh vượt phạm vi "chỉ thay placeholder cross-repo" sang "sửa nội dung convention" |

## 5. Đã tự động chuyển đổi (migration 2026-07-14)

Toàn bộ tham chiếu rõ ràng, không mơ hồ tới file/thư mục cụ thể bên trong `<API_REPO>` (`docs/ARCHITECTURE.md`,
`docs/CONVENTIONS.md`, `docs/ONBOARDING.md`, `docs/api/CONVENTIONS.md`, `docs/api/ARCHITECTURE.md`,
`docs/llm/ARCHITECTURE.md`, `docs/ADR-admin-api-migration.md`, `CLAUDE.md`, `TESTING.md`,
`docs/superpowers/...`, `docs/active/...`, `services/api/...`, `services/cat/...`, `services/llm/...`) đã được
chuyển thành `<API_REPO>/...`. Chi tiết đầy đủ, theo từng file: `docs/cross-repo-migration-report.md`.

**Ngoại lệ đã xử lý riêng (không mechanical):**
- `docs/project-analysis.md`, `docs/ai-workspace-report.md` — 2 báo cáo lịch sử, chỉ thêm banner ghi chú ở đầu
  file, **không** viết lại nội dung/cây thư mục bên trong (đó là bản ghi hiện trạng tại thời điểm viết).
- `docs/adr/0000-index.md` — sửa link Markdown tương đối bị hỏng (`../ADR-admin-api-migration.md` →
  `<API_REPO>/docs/ADR-admin-api-migration.md`, kèm ghi chú không bấm được).
- `.ai/project-context.md` §0 — viết lại (không chỉ thay placeholder) vì bảng mô tả repo đã sai hoàn toàn sau khi
  workspace chuyển ra khỏi `<API_REPO>`.
- `studee-env-setup/README.md` → `<ENV_BUNDLE>/README.md` (1 chỗ, trong `.ai/workflows/release-workflow.md`).

## 6. Quy ước cho tài liệu mới

Khi viết spec/design/prompt/checklist mới trong `studee-workspace`:

- Tham chiếu tới bất kỳ file nào **không nằm trong `studee-workspace`** → dùng placeholder (`<API_REPO>/...`,
  `<WEB_REPO>/...`, `<ADMIN_REPO>/...`, `<ENV_BUNDLE>/...`).
- Tham chiếu tới file **trong chính `studee-workspace`** → viết đường dẫn tương đối bình thường (`.ai/...`,
  `templates/...`, `specs/...`) — không cần `<WORKSPACE>/` trừ khi đoạn văn bản đó có khả năng cao bị đọc từ
  ngoài `studee-workspace` (vd nội dung sẽ được dán nguyên văn vào `CLAUDE.md` của `<API_REPO>`).
- Nếu một token có thể đúng với nhiều hơn 1 repo (như `README.md`, `.env.example`, `.github/CODEOWNERS`) → annotate
  bằng câu chữ ("repo đang làm việc", "repo tương ứng") thay vì gán cứng 1 placeholder sai.
