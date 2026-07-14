<!-- Tiếng Việt — tài liệu onboarding cho THÀNH VIÊN team (con người). Theo .ai/project-context.md §8:
     nội dung mô tả quy trình cho người đọc → tiếng Việt. Đây là "cửa ngõ cho người",
     song song với .ai/project-context.md là "cửa ngõ cho AI agent". -->

# Hướng dẫn Team — studee-workspace

> **Đọc file này đầu tiên nếu bạn là người** (BA, Tech Lead, Developer, Reviewer) mới vào project.
> File này **tổng hợp và trỏ đường** — nó không chép lại nội dung các tài liệu gốc, mà cho bạn bức tranh
> tổng thể rồi chỉ tới đúng file cần đọc sâu.
>
> **Nếu bạn là AI agent**, cửa ngõ của bạn là `.ai/project-context.md` (tiếng Anh), không phải file này.

---

## 1. Workspace này là gì (bản đồ 30 giây)

Nền tảng ENOVA/studee (học tiếng Anh có AI) được tách thành **4 repo độc lập, clone cạnh nhau** trên máy —
không phải monorepo, không có git history chung. Mỗi repo có git riêng, PR riêng.

| Placeholder | Repo thật | Vai trò |
|---|---|---|
| `<API_REPO>` | `exe-api` | Backend (Node business + Python adaptive-testing + Python AI proxy) |
| `<WEB_REPO>` | `exe-web` | App học viên (Next.js) |
| `<ADMIN_REPO>` | `exe-admin` | Portal admin/trung tâm (Next.js) |
| `<WORKSPACE>` | **`studee-workspace`** (repo này) | Hạ tầng cộng tác AI-SDD: `.ai/`, `specs/`, `templates/`, `docs/adr/`, `docs/architecture/` |
| `<ENV_BUNDLE>` | `studee-env-setup` | Bundle `.env` để onboard local (không phải git repo) |

**Điểm mấu chốt:** `studee-workspace` **không chứa code sản phẩm**. Nó chứa *quy trình, template, đặc tả (spec)
và tài liệu* mà cả 4 repo dùng chung. Một feature đụng cả FE + BE được điều phối ở đây bằng **một spec duy nhất**,
nhưng code được implement bằng **các PR riêng trong từng repo**.

- Đọc `<...>` là placeholder → tự thay bằng đường dẫn thật trên máy bạn (vd `../exe-api/...` nếu các repo là
  thư mục sibling). Quy ước đầy đủ: `docs/cross-repo-linking.md`.
- Tổng quan sản phẩm/kiến trúc: `docs/project-analysis.md`, và `<API_REPO>/docs/ARCHITECTURE.md`.

---

## 2. Bốn vai trò — ai làm gì

Mỗi vai trò có thể là **con người** *hoặc* một **AI agent** đóng vai đó (file mô tả agent nằm ở `.ai/agents/`).
Con người luôn giữ quyền duyệt ở các cổng (gate); AI làm phần nặng nhọc và tự-kiểm trước.

| Vai trò | Con người làm gì | AI agent tương ứng | Output chính |
|---|---|---|---|
| **BA** | Xác nhận ý định nghiệp vụ, độ ưu tiên, ràng buộc; duyệt spec | `.ai/agents/ba-agent.md` | `specs/<feature>/spec.md`, `acceptance.md` |
| **Tech Lead** | Chốt kiến trúc, duyệt design, merge cuối (CODEOWNER) | `.ai/agents/techlead-agent.md` | `design.md`, `data-model.md`, `contracts/*`, `tasks.md` |
| **Developer** | Code từng task, gỡ vướng, PR | `.ai/agents/developer-agent.md` | Code + test, commit theo task |
| **Reviewer** | (AI đi trước) — người review là Tech Lead ở cổng cuối | `.ai/agents/reviewer-agent.md` | `specs/<feature>/review.md` |

**Nguyên tắc cộng tác** (lặp trong cả 4 file agent): agent sau **không tự diễn giải lại** quyết định của agent
trước — nếu spec/design sai thì **trả lại**, không âm thầm sửa lệch. Không AI nào tự duyệt việc của chính mình;
cổng duyệt cuối **luôn là người**.

---

## 3. Luồng AI — Idea → Merge

Đây là "xương sống" của workspace. Toàn bộ chi tiết từng bước (vai trò, tài liệu bắt buộc, cổng duyệt, exit
criteria) ở `.ai/workflows/feature-workflow.md`. Tóm tắt:

```
Idea → AI Brainstorm → BA Review → Specification → Tech Lead Design → Technical Review
     → Task Generation → Development → AI Self Review → Tech Lead Review → Merge
```

| # | Bước | Ai chịu trách nhiệm | Cổng duyệt (người) | Prompt dùng |
|---|---|---|---|---|
| 1 | Idea | Bất kỳ ai | — | (tạo GitHub issue) |
| 2 | AI Brainstorm | BA Agent | — | `.ai/prompts/brainstorm-feature.md` |
| 3 | BA Review | Người BA | ✅ đáng làm không? | — |
| 4 | Specification | BA Agent | ✅ duyệt `spec.md`+`acceptance.md` | `.ai/prompts/generate-spec.md` |
| 5 | Tech Lead Design | Tech Lead Agent | — (bản nháp) | `.ai/prompts/generate-design.md`, `generate-contract.md` |
| 6 | Technical Review | Người Tech Lead (+ Reviewer Agent phản biện) | ✅ duyệt design | — |
| 7 | Task Generation | Tech Lead Agent | — | `.ai/prompts/generate-tasks.md` |
| 8 | Development | Developer Agent / dev | — | `.ai/prompts/implement-task.md`, `generate-tests.md` |
| 9 | AI Self Review | Reviewer Agent | ✅ 0 lỗi blocking | `.ai/prompts/review-pr.md` |
| 10 | Tech Lead Review | Người Tech Lead (CODEOWNER) | ✅ **cổng chặn merge** | — |
| 11 | Merge | Người Tech Lead | — | — |

**Vì sao có 2 điểm review AI** (không phải 1): Technical Review bắt lỗi *thiết kế trước khi viết code* (rẻ nhất),
AI Self Review bắt lỗi *implement trước khi tốn thời gian người*. Tech Lead Review (người) là cổng **duy nhất**
thật sự chặn merge. Chi tiết: `.ai/workflows/review-workflow.md`.

> Feature nhỏ / fix vặt **không cần** chạy hết pipeline — xem `.ai/checklists/techlead-checklist.md`
> §When to skip the full pipeline. Pipeline dành cho việc có "diện tích" sản phẩm/kiến trúc thật.

---

## 4. Luồng GitHub

> ⚠️ Các quy tắc git/branch/PR/CODEOWNERS dưới đây thuộc về **các repo code** (`exe-api`, `exe-web`, `exe-admin`)
> — nơi có branch protection và CI thật. `studee-workspace` cũng có git riêng nhưng chỉ chứa tài liệu; commit
> spec/design vào đây theo cùng quy ước để đồng nhất. Nguồn gốc quy tắc: `<API_REPO>/docs/CONVENTIONS.md`.

### 4.1 Ba nhánh

| Nhánh | Mục đích | Ai push |
|---|---|---|
| `main` | Production | Chỉ Tech Lead merge |
| `develop` | Tích hợp | **Chỉ qua PR** |
| `feature/*`, `fix/*`, `chore/*`, `docs/*`, `refactor/*` | Nhánh làm việc | Dev push thẳng |

- **PR base LUÔN là `develop`, không bao giờ `main`.** Một GitHub Action tự đóng PR nhắm vào `main` nếu không
  phải Tech Lead. Đây là quy tắc cứng — đừng bao giờ vượt qua.
- **Commit theo Conventional Commits:** `feat|fix|chore|docs|refactor|test(<scope>): <mô tả>`.
- Commit `git add <file cụ thể>`, **không** `git add .`. **Không** thêm `Co-Authored-By` hay attribution AI
  (quy tắc cứng, `<API_REPO>/CLAUDE.md`).

### 4.2 Vòng đời một thay đổi

```
Issue (feature-request / bug-report)
   → nhánh feature/<slug> từ develop
   → commit theo task
   → mở PR base=develop
   → AI Self Review (Reviewer Agent) — phải sạch lỗi blocking
   → Tech Lead Review (CODEOWNER @dotuananhtb duyệt — branch protection)
   → squash-and-merge vào develop → xoá nhánh
   → release lên main/prod theo .ai/workflows/release-workflow.md
```

### 4.3 Issue templates

- Feature mới → `.github/ISSUE_TEMPLATE/feature-request.md` (đây là "Idea" ở bước 1 của luồng AI).
- Bug → `.github/ISSUE_TEMPLATE/bug-report.md`.

### 4.4 Điều cần biết về review/CI

- Mỗi PR **bắt buộc** có approval của CODEOWNER (`@dotuananhtb`) — enforce bằng branch protection.
- **CI hiện chỉ gate *deploy*** (SSH khi push `develop`/tag), **không** chạy test/lint trên PR. → PR "xanh"
  **không** có nghĩa là test đã chạy. Tự chạy `npm test` / `pytest` theo `<API_REPO>/TESTING.md`.

---

## 5. Prompt mẫu theo từng thành viên

Các prompt "chính chủ", đầy đủ nằm trong `.ai/prompts/*.md` — **luôn copy từ file gốc** để không lệch. Bảng dưới
map từng vai trò → prompt cần dùng, kèm phiên bản rút gọn để bạn dán nhanh vào một phiên AI mới.

> Mọi phiên AI đều mở đầu bằng câu: *"Read `.ai/project-context.md` first."* Thay `<...>` bằng giá trị thật.

### 👤 BA
| Khi nào | Prompt gốc |
|---|---|
| Mới có ý tưởng thô, chưa rõ ranh giới | `.ai/prompts/brainstorm-feature.md` |
| Đã chọn hướng, viết spec | `.ai/prompts/generate-spec.md` |

```
You are acting as the BA Agent (.ai/agents/ba-agent.md). Read .ai/project-context.md first.
Raw idea: "<dán ý tưởng>"
Làm theo .ai/prompts/brainstorm-feature.md — chỉ đưa ra 2-4 hướng + câu hỏi mở, CHƯA viết spec.
Output tiếng Việt.
```

### 👤 Tech Lead
| Khi nào | Prompt gốc |
|---|---|
| Thiết kế từ spec đã duyệt | `.ai/prompts/generate-design.md` |
| Feature cross-service/cross-repo | `.ai/prompts/generate-contract.md` |
| Chia task từ design đã duyệt | `.ai/prompts/generate-tasks.md` |

```
You are acting as the Tech Lead Agent (.ai/agents/techlead-agent.md). Read .ai/project-context.md first.
Spec đã duyệt: specs/<feature-slug>/spec.md + acceptance.md
Làm theo .ai/prompts/generate-design.md — viết design.md (+ data-model.md, contracts/* nếu cần).
Chỗ nào spec chưa rõ thì HỎI, đừng đoán. Output tiếng Việt.
```

### 👤 Developer
| Khi nào | Prompt gốc |
|---|---|
| Làm 1 task trong `tasks.md` (mỗi lần 1 task) | `.ai/prompts/implement-task.md` |
| Viết/bổ sung test | `.ai/prompts/generate-tests.md` |
| Cập nhật tài liệu sau khi code | `.ai/prompts/update-documentation.md` |

```
You are acting as the Developer Agent (.ai/agents/developer-agent.md).
Read .ai/project-context.md, <API_REPO>/docs/api/CONVENTIONS.md, và <API_REPO>/CLAUDE.md first.
Task: Task <N> trong specs/<feature-slug>/tasks.md. Design: specs/<feature-slug>/design.md
Làm theo .ai/prompts/implement-task.md: test-first → implement → chạy → commit. CHỈ đụng file task liệt kê.
Không sang task kế trong cùng lượt.
```

### 👤 Reviewer
| Khi nào | Prompt gốc |
|---|---|
| Tự review PR trước khi nhờ người | `.ai/prompts/review-pr.md` |

```
You are acting as the Reviewer Agent (.ai/agents/reviewer-agent.md). Mục tiêu: TÌM lỗi thật, mặc định hoài nghi.
Diff: <link PR hoặc "current working tree diff">
Spec: specs/<feature-slug>/spec.md, acceptance.md, tasks.md
Làm theo .ai/prompts/review-pr.md: chạy test thật, đối chiếu từng AC, phân loại BLOCKING/NON-BLOCKING,
viết specs/<feature-slug>/review.md. KHÔNG tự sửa code khi review.
```

---

## 6. Các bước tạo tài liệu cho một feature

Mỗi feature có một thư mục `specs/<feature-slug>/` (kebab-case, không kèm ngày). Cấu trúc đầy đủ và ví dụ mẫu:
`specs/README.md` + `specs/example-feature/`. Các bước tạo tài liệu:

1. **Tạo issue** từ `.github/ISSUE_TEMPLATE/feature-request.md` — ghi lại ý tưởng cho khỏi quên.
2. **Brainstorm** (BA): chạy prompt `brainstorm-feature` → chọn 1 hướng. *Chưa tạo file nào.*
3. **Viết spec** (BA): chạy `generate-spec` → tạo:
   - `specs/<slug>/spec.md`  ← từ `templates/spec-template.md` (mục tiêu, phạm vi in/out, câu hỏi mở)
   - `specs/<slug>/acceptance.md`  ← từ `templates/acceptance-template.md` (AC-1, AC-2… checkable)
   - **Người BA duyệt** → không còn câu hỏi mở mới sang bước sau.
4. **Design** (Tech Lead): chạy `generate-design` (+ `generate-contract` nếu cross-repo) → tạo:
   - `specs/<slug>/design.md`  ← `templates/design-template.md`
   - `specs/<slug>/data-model.md`  (nếu data model không đủ đơn giản để gộp vào design)
   - `specs/<slug>/research.md`  (chỉ khi có câu hỏi kỹ thuật cần điều tra) ← `templates/research-template.md`
   - `specs/<slug>/contracts/*.md`  (1 file / ranh giới cross-service|cross-repo) ← `templates/contract-template.md`
   - Cân nhắc 1 ADR trong `docs/adr/` nếu có quyết định kiến trúc lâu dài (dùng `docs/adr/adr-template.md`).
   - **Technical Review** (người Tech Lead, Reviewer Agent phản biện) → duyệt design.
5. **Chia task** (Tech Lead): chạy `generate-tasks` → tạo:
   - `specs/<slug>/tasks.md`  ← `templates/tasks-template.md` (task TDD-checkbox, mỗi task ~1 PR, có lệnh verify)
6. **Implement** (Developer): làm từng task theo `implement-task`, commit theo task.
7. **Review** (Reviewer): chạy `review-pr` → tạo `specs/<slug>/review.md` ← `templates/review-template.md`.
8. **Tài liệu vận hành** (nếu cần): cập nhật docs liên quan bằng `update-documentation`.

> Quy tắc ngôn ngữ tài liệu (`.ai/project-context.md` §8): **code + comment = tiếng Anh**;
> **`docs/`, `specs/`, `templates/` = tiếng Việt**; **`.ai/` (agent/workflow/prompt/checklist) = tiếng Anh**.

**Checklist tự kiểm trước khi bàn giao mỗi bước:** `.ai/checklists/{ba,techlead,developer,review,release}-checklist.md`.

---

## 7. Quy tắc vàng (đừng vi phạm)

- ❌ Không viết code nghiệp vụ như "tác dụng phụ" của việc chuẩn bị tài liệu/workspace.
- ❌ Không đặt PR base là `main`; luôn là `develop`.
- ❌ Không commit `.env`/secret; không dùng `git add .`; không thêm attribution AI vào commit/PR.
- ❌ Không để AI tự duyệt việc của chính nó — cổng cuối luôn là người (CODEOWNER).
- ❌ Đừng "tiện tay sửa" quy ước/tài liệu nghiệp vụ đang mâu thuẫn (vd `<API_REPO>/docs/api/CONVENTIONS.md` còn
  lỗi thời) — **báo Tech Lead**, đừng tự sửa (đã ghi nhận ở `docs/project-analysis.md` §12).
- ✅ Mọi phiên AI bắt đầu bằng đọc `.ai/project-context.md`. Mọi thành viên người bắt đầu bằng file này.

---

## 8. Bắt đầu từ đâu (cheat sheet)

| Tôi là… | Tôi mở… |
|---|---|
| Người mới (bất kỳ vai trò) | File này → `.ai/project-context.md` → `docs/project-analysis.md` |
| BA | `.ai/agents/ba-agent.md` + prompt `brainstorm-feature` / `generate-spec` |
| Tech Lead | `.ai/agents/techlead-agent.md` + prompt `generate-design` / `generate-tasks` |
| Developer | `.ai/agents/developer-agent.md` + `<API_REPO>/docs/api/CONVENTIONS.md` + prompt `implement-task` |
| Reviewer | `.ai/agents/reviewer-agent.md` + prompt `review-pr` |
| AI agent | `.ai/project-context.md` (KHÔNG phải file này) |
| Muốn hiểu toàn bộ pipeline | `.ai/workflows/feature-workflow.md` |
