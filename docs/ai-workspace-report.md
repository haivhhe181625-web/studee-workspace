# Báo cáo — AI-SDD Workspace cho `exe-api` (pilot)

> ⚠️ **Ghi chú lịch sử (2026-07-14):** báo cáo này mô tả trạng thái **tại thời điểm workspace AI-SDD vừa được tạo
> bên trong `exe-api`** — cây thư mục ở mục 3 và mọi đường dẫn trong bài phản ánh layout của `exe-api` lúc đó,
> giữ nguyên làm bản ghi lịch sử, không viết lại theo placeholder. **Ngay sau đó cùng ngày**, `.ai/`, `specs/`,
> `templates/`, `docs/adr/`, `docs/architecture/` và chính file này đã được **chuyển sang repo `studee-workspace`**
> — nghĩa là mục 3 "Cấu trúc thư mục (kết quả)" dưới đây **không còn đúng với `exe-api` hiện tại**. Xem
> `docs/cross-repo-linking.md` cho quy ước link hiện hành, và trạng thái thư mục thật hiện tại là: `exe-api` chỉ
> còn code + docs gốc của nó; `.ai/`/`specs/`/`templates/`/`docs/adr/`/`docs/architecture/` nằm ở `studee-workspace`.

> **Ngày:** 2026-07-14 · **Phạm vi:** repo `exe-api` (pilot repo được chọn cho việc thử nghiệm quy trình AI-SDD
> trước khi nhân rộng). Báo cáo tổng kết những gì đã chuẩn bị — không có business logic nào bị thay đổi.

## 1. Thay đổi trong repository

Không có code nghiệp vụ (`services/api/src`, `services/cat/src`, `services/llm/src`) nào bị sửa. Toàn bộ thay
đổi là **thêm mới** tài liệu/hạ tầng cộng tác AI:

- Thêm `docs/project-analysis.md` — phân tích kiến trúc/stack/quy ước/điểm mạnh-yếu hiện có.
- Thêm folder `.ai/` (context, agent, workflow, prompt, checklist).
- Thêm folder `specs/` (kèm 1 ví dụ minh hoạ `example-feature/`).
- Thêm folder `templates/` (7 template).
- Thêm folder `docs/architecture/`, `docs/adr/` (kèm index + template ADR).
- Thêm `.github/ISSUE_TEMPLATE/` (feature-request, bug-report).
- Thêm báo cáo này (`docs/ai-workspace-report.md`).

**Không file nào đã tồn tại bị ghi đè.** Mọi thư mục mục tiêu (`.ai/`, `specs/`, `templates/`,
`docs/architecture/`, `docs/adr/`, `.github/ISSUE_TEMPLATE/`) đều chưa tồn tại trước khi bắt đầu (đã kiểm tra
trước khi tạo). Các tài liệu hiện có (`CLAUDE.md`, `docs/CONVENTIONS.md`, `docs/ARCHITECTURE.md`,
`docs/ONBOARDING.md`, `docs/ADR-admin-api-migration.md`, `docs/superpowers/*`, `docs/active/*`) được **tham
chiếu, không sao chép lại** — workspace mới trỏ tới chúng làm nguồn sự thật thay vì nhân bản nội dung.

## 2. Danh sách file đã tạo (39 file)

```
docs/project-analysis.md
docs/ai-workspace-report.md
docs/architecture/README.md
docs/adr/0000-index.md
docs/adr/adr-template.md

.ai/project-context.md
.ai/agents/{ba,techlead,developer,reviewer}-agent.md
.ai/workflows/{feature,review,release}-workflow.md
.ai/prompts/{brainstorm-feature,generate-spec,generate-design,generate-contract,
             generate-tasks,implement-task,review-pr,generate-tests,update-documentation}.md
.ai/checklists/{ba,techlead,developer,review,release}-checklist.md
.ai/context/README.md

templates/{spec,design,tasks,research,acceptance,review,contract}-template.md

specs/README.md
specs/example-feature/{spec,acceptance,design,data-model,research,tasks}.md
specs/example-feature/contracts/toggle-reminder.md
specs/example-feature/diagrams/README.md

.github/ISSUE_TEMPLATE/{feature-request,bug-report}.md
```

## 3. Cấu trúc thư mục (kết quả)

```
exe-api/
├── CLAUDE.md, README.md, TESTING.md                    # giữ nguyên
├── docs/
│   ├── project-analysis.md                              # MỚI — Phase 1
│   ├── ai-workspace-report.md                            # MỚI — báo cáo này
│   ├── architecture/README.md                            # MỚI — trống, chờ điền theo domain
│   ├── adr/{0000-index.md, adr-template.md}               # MỚI — ADR mới từ đây, ADR cũ giữ nguyên vị trí
│   ├── ARCHITECTURE.md, CONVENTIONS.md, ONBOARDING.md,     # giữ nguyên
│   │   ADR-admin-api-migration.md, api/, llm/, active/,
│   │   superpowers/, analysis/, security/
├── .ai/                                                   # MỚI — toàn bộ
│   ├── project-context.md
│   ├── agents/ (ba, techlead, developer, reviewer)
│   ├── workflows/ (feature, review, release)
│   ├── prompts/ (9 file)
│   ├── checklists/ (5 file)
│   └── context/README.md
├── templates/                                             # MỚI — 7 template
├── specs/                                                 # MỚI — README + example-feature/
├── .github/
│   ├── ISSUE_TEMPLATE/ (feature-request, bug-report)       # MỚI
│   ├── CODEOWNERS, workflows/                              # giữ nguyên
└── services/{api,cat,llm}/                                 # KHÔNG đụng tới
```

## 4. Từng vai trò làm việc như thế nào

- **BA** (người hoặc `.ai/agents/ba-agent.md`): nhận idea → chạy `.ai/prompts/brainstorm-feature.md` →
  `.ai/prompts/generate-spec.md` → tạo `specs/<feature>/spec.md` + `acceptance.md` → tự kiểm bằng
  `.ai/checklists/ba-checklist.md` → xin duyệt người.
- **Tech Lead** (người hoặc `.ai/agents/techlead-agent.md`): nhận spec đã duyệt → chạy
  `.ai/prompts/generate-design.md` (+ `generate-contract.md` nếu cross-service/cross-repo) →
  `.ai/checklists/techlead-checklist.md` → Technical Review với người → chạy `.ai/prompts/generate-tasks.md`.
- **Developer** (người hoặc `.ai/agents/developer-agent.md`): chạy `.ai/prompts/implement-task.md` cho từng task
  trong `tasks.md`, test-first theo đúng phong cách đã có ở `docs/superpowers/plans/`, tự kiểm bằng
  `.ai/checklists/developer-checklist.md`.
- **Reviewer** (`.ai/agents/reviewer-agent.md`, luôn là AI trước khi tới người): chạy
  `.ai/prompts/review-pr.md` → `specs/<feature>/review.md` → nếu sạch, chuyển cho Tech Lead Review (người,
  CODEOWNER) — gate cuối cùng, không thể bỏ qua (`.github/CODEOWNERS`).

Chi tiết đầy đủ từng gate/exit-criteria: `.ai/workflows/feature-workflow.md`.

## 5. AI agent cộng tác với nhau như thế nào

Bốn agent (`.ai/agents/*.md`) không làm việc độc lập — mỗi agent chỉ nhận input từ agent/gate trước và chỉ bàn
giao output cho agent/gate sau, theo đúng thứ tự pipeline:

```
BA Agent → (người duyệt spec) → Tech Lead Agent → (người duyệt design) → Developer Agent
   → Reviewer Agent (AI Self Review) → (người duyệt cuối — CODEOWNER)
```

Nguyên tắc cộng tác cốt lõi (lặp lại trong cả 4 file agent, phần "Collaboration rules"):
- Agent sau **không được** âm thầm diễn giải lại quyết định của agent trước — nếu spec/design có vấn đề, trả lại
  thay vì tự sửa lệch hướng.
- Reviewer Agent đóng vai trò phản biện ở **2 điểm** trong pipeline (Technical Review cho design, AI Self Review
  cho code) — không chỉ 1 lần cuối, để bắt lỗi càng sớm càng rẻ.
- Không AI agent nào tự duyệt việc của chính mình — gate duyệt cuối luôn là người (`.ai/workflows/review-workflow.md`
  §What "AI Self Review" must never become).

## 6. Quy trình làm việc hằng ngày khuyến nghị

1. Idea mới → tạo GitHub issue từ `.github/ISSUE_TEMPLATE/feature-request.md` (hoặc `bug-report.md` nếu là bug).
2. Nếu đủ nhỏ/rõ ràng (xem `.ai/checklists/techlead-checklist.md` §When to skip the full pipeline) → làm trực
   tiếp, PR bình thường theo `docs/CONVENTIONS.md`.
3. Nếu là feature thật → theo đúng `.ai/workflows/feature-workflow.md` từ đầu tới cuối, dùng
   `.ai/prompts/*.md` tương ứng từng bước.
4. Mọi phiên AI mới bắt đầu bằng việc đọc `.ai/project-context.md` — đây là điểm vào duy nhất, không cần đọc lại
   toàn bộ `CLAUDE.md`/`docs/*` mỗi lần (project-context.md đã trỏ đúng chỗ cần).
5. Trước khi merge: `.ai/checklists/review-checklist.md` phải sạch, `.ai/checklists/release-checklist.md` chạy
   trước khi deploy dev/prod.

## 7. Giới hạn hiện tại (Known limitations)

- **Chỉ có ở `exe-api`.** `exe-web`, `exe-admin` chưa có `.ai/`/`specs/`/`templates/` riêng — feature cross-repo
  vẫn spec ở đây (`.ai/project-context.md` §0) nhưng phần implementation ở 2 repo kia vẫn chạy theo quy trình cũ
  của họ (`CONTRIBUTING.md`/`AGENTS.md` riêng).
- **Chưa có CI gate thật** cho quy trình mới — không có workflow nào tự động kiểm tra `specs/<feature>/` có đủ
  file/đủ mục trước khi merge; toàn bộ dựa vào kỷ luật con người + checklist thủ công.
- **Chưa qua thử lửa** — pipeline mới hoàn toàn viết trên giấy, `specs/example-feature/` là ví dụ hư cấu, chưa
  có 1 feature thật nào chạy hết từ Idea đến Merge qua quy trình này để kiểm chứng thời lượng/độ ma sát thực tế.
- **`docs/api/CONVENTIONS.md` vẫn còn lỗi thời** (rule AdminJS cũ, xem `docs/project-analysis.md` §5/§12) —
  workspace này chỉ *phát hiện và ghi nhận*, cố tình không tự sửa vì đó là nội dung nghiệp vụ/quy ước hiện có,
  ngoài phạm vi "chỉ chuẩn bị hạ tầng cộng tác AI".
- **Không có công cụ enforce** (lint, CI check) cho bất kỳ quy ước mới nào trong `.ai/`/`templates/` — giống tình
  trạng chung của `services/api` (không ESLint), rủi ro trôi theo thời gian nếu không có kỷ luật review.
- **Rollback quy trình release** vẫn chưa có tài liệu chính thức (ghi nhận lại ở
  `.ai/workflows/release-workflow.md` §Rollback, không tự bịa ra quy trình chưa tồn tại).

## 8. Cải tiến tương lai (Future improvements)

1. **Chạy thử 1 feature thật, nhỏ, qua toàn bộ pipeline** — đo thời gian, ma sát ở từng gate, chỉnh template dựa
   theo thực tế trước khi nhân rộng.
2. **Nhân rộng sang `exe-web`/`exe-admin`** — sau khi pipeline ổn định ở `exe-api`, quyết định giữa 2 hướng:
   (a) copy `.ai/`/`templates/` vào từng repo, hay (b) tách thành 1 repo workspace dùng chung, 3 repo còn lại
   chỉ chứa code + link tới workspace đó. Quyết định này nên là 1 ADR khi tới lúc.
3. **Thêm CI job nhẹ** kiểm tra cấu trúc `specs/<feature>/` (đủ file bắt buộc, không còn placeholder `<...>`)
   trước khi cho phép merge PR gắn với feature đó — biến checklist thủ công thành gate tự động một phần.
4. **Dọn `docs/CONVENTIONS.md` vs `docs/api/CONVENTIONS.md`** (đã ghi nhận ở `docs/project-analysis.md`) —
   khuyến nghị tech lead xử lý riêng, ngoài phạm vi workspace này.
5. **Cân nhắc thêm `.ai/context/` cho domain `assessment`/MST-IRT** khi có feature động tới nó — đây là domain
   phức tạp nhất trong repo, giá trị context-file cao nhất.
6. **Đánh số lại `docs/ADR-admin-api-migration.md`** thành `docs/adr/0001-...md` một khi có thể chấp nhận sửa
   link cũ (hiện tại workspace này giữ nguyên để không phá tính tương thích ngược).

---

## Tóm tắt cho người dùng

Repo `exe-api` hiện đã có đầy đủ hạ tầng để chạy thử quy trình AI-SDD Idea → Merge: 4 agent, 3 workflow, 9
prompt, 5 checklist, 7 template, 1 ví dụ spec đầy đủ, cấu trúc ADR có đánh số, và issue template. Không có code
nghiệp vụ nào bị đụng tới. Sẵn sàng để chạy thử với **một feature thật, nhỏ** nhằm kiểm chứng quy trình trước khi
quyết định nhân rộng sang `exe-web`/`exe-admin` hoặc tách thành workspace repo riêng.
