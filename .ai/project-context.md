# AI Project Context — studee-workspace

> **Read this file first in every new AI session** — whether your working directory is this repo
> (`studee-workspace`) or a sibling code repo (`<API_REPO>` etc.) that you were pointed here from. It is the
> single entry point into how the whole ENOVA/studee multi-repo system works. It intentionally does not
> duplicate existing docs — it points to them and adds the context those docs don't cover (why this workspace
> exists, how to work here as BA/Tech Lead/Dev/Reviewer, and what's still evolving).

## 0. What this repo is, in the wider workspace

The ENOVA/studee platform is split across **four independent sibling folders/repos**, checked out side by side
locally (no shared parent git history, no monorepo tool):

| Placeholder | Repo | Role | Git |
|---|---|---|---|
| `<API_REPO>` | `exe-api` | Backend — Node business logic + Python adaptive-testing + Python AI proxy | own git history |
| `<WEB_REPO>` | `exe-web` | Learner-facing Next.js app | own git history |
| `<ADMIN_REPO>` | `exe-admin` | Admin/center-staff portal, Next.js | own git history |
| `<WORKSPACE>` | `studee-workspace` (**this repo**) | AI-SDD collaboration infra: `.ai/`, `specs/`, `templates/`, `docs/adr/`, `docs/architecture/` | own git history |
| `<ENV_BUNDLE>` | `studee-env-setup` | Loose `.env` bundle for local onboarding across the repos above | not a git repo |

A feature that touches FE and BE is coordinated across separate PRs in separate repos, not a single commit.

**How to read a placeholder like `<API_REPO>/docs/ARCHITECTURE.md` elsewhere in this workspace:** substitute the
repo name from the table above for the local checkout you're using (e.g. `../exe-api/docs/ARCHITECTURE.md` if
`exe-api` is a sibling folder of `studee-workspace`). These are **symbolic references, not clickable relative
links** — resolve them against your actual local layout. Full convention: `docs/cross-repo-linking.md`.

**Where you're reading this file from matters:** if your cwd is `studee-workspace` itself, references to
`.ai/...`, `templates/...`, `specs/...`, `docs/adr/...`, `docs/architecture/...` in these docs are plain
relative paths and resolve directly. If your cwd is a sibling repo (e.g. you're the Developer Agent working
inside `<API_REPO>` implementing a task), those same bare references mean `<WORKSPACE>/.ai/...`,
`<WORKSPACE>/templates/...`, etc. — this workspace was piloted inside `exe-api` and moved out to its own repo on
2026-07-14 (see `docs/ai-workspace-report.md` and `docs/cross-repo-linking.md`); most `.ai/`-internal
cross-references were left as bare relative paths rather than mechanically prefixed everywhere, precisely because
which one applies depends on your cwd, not on the text itself.

**This AI-SDD workspace was piloted inside `<API_REPO>`, then extracted into its own repo (this one).** Keep
every template and prompt in this workspace **repo-agnostic and cross-repo aware** — a spec should be able to
describe work spanning `<API_REPO>` + `<WEB_REPO>` + `<ADMIN_REPO>` even though today only `<API_REPO>` has
matching code-level conventions documented. See `docs/ai-workspace-report.md` for the original pilot rollout plan
and `docs/cross-repo-linking.md` for the extraction/linking convention.

## 1. Project overview

ENOVA/studee is an AI-assisted English learning platform: 1-on-1 AI speaking practice, IPA pronunciation
training, adaptive placement/assessment exams (IRT/MST), and B2B center management (multi-tenant). Full detail:
`docs/project-analysis.md` (repo analysis), `README.md` (quick start), `<API_REPO>/docs/ONBOARDING.md` (~10 min setup).

## 2. Tech stack

| Service | Stack |
|---|---|
| `<API_REPO>/services/api` | Node 20, Express 4, MongoDB (Mongoose 8), Redis, Socket.IO, BullMQ, Firebase Admin, JWT, Joi |
| `<API_REPO>/services/cat` | Python — IRT + Multi-Stage Testing adaptive item selection, internal-only |
| `<API_REPO>/services/llm` | Python 3.11, FastAPI — Anti-Corruption Layer wrapping OpenAI (chat/ASR/TTS/embeddings) |

No relational DB. Qdrant used as a vector store for question-embedding dedup. Full picture:
`<API_REPO>/docs/ARCHITECTURE.md` (cross-service) and `<API_REPO>/docs/api/ARCHITECTURE.md` / `<API_REPO>/docs/llm/ARCHITECTURE.md` (per-service).

## 3. Architecture style

- "Lightweight microservices" — 3 services split by workload (Node for I/O/business, Python for ML/audio), not a
  distributed monolith, not full microservices.
- REST everywhere, no GraphQL/gRPC. Cross-service calls use `X-Service-Key` shared secret over Docker DNS; JWT is
  only for client-facing auth.
- 5-file module pattern per domain: `<feature>.model.js / .schema.js / .service.js / .controller.js / .routes.js`
  under `<API_REPO>/services/api/src/modules/<feature>/`.
- External providers (OpenAI, Firebase, etc.) are always called through an adapter, never required directly in a
  service. Admin surface (`src/admin-api/`) must never contain business logic — it delegates to module services.
- Full detail and the anti-pattern table: `<API_REPO>/docs/ARCHITECTURE.md`.

## 4. Coding conventions

Authoritative source: `<API_REPO>/docs/CONVENTIONS.md` (monorepo-wide: git/commit/branch/PR) — **`<API_REPO>/docs/api/CONVENTIONS.md`
also exists but is partially stale** (it still describes the pre-ADR "no admin REST" rule; the current rule is in
`<API_REPO>/CLAUDE.md` §Admin Rules and `<API_REPO>/docs/ADR-admin-api-migration.md`). When the two disagree, `<API_REPO>/CLAUDE.md` and the ADR
win. This inconsistency is tracked in `docs/project-analysis.md` §12 — do not silently "fix" it by editing
business docs as a side effect of an unrelated task; flag it to the tech lead instead.

No ESLint/Prettier is configured for `<API_REPO>/services/api`. Conventions are enforced by review, not tooling — be extra
careful to actually read `<API_REPO>/docs/CONVENTIONS.md` and `<API_REPO>/CLAUDE.md` rather than assume a linter would catch drift.

## 5. Naming conventions

| Thing | Convention | Example |
|---|---|---|
| File | kebab-case | `auth.routes.js`, `user.service.js` |
| Variable / function | camelCase | `createUser` |
| Class / Model | PascalCase | `User`, `ApiError` |
| Constant / env var | UPPER_SNAKE_CASE | `JWT_SECRET` |
| Permission string | `<resource>:<action>` | `user:read`, `lesson:manage` |
| Socket room / event | `<scope>:<id>` / `<resource>:<action>` | `user:<userId>`, `conversation:token` |

Full table: `<API_REPO>/docs/api/CONVENTIONS.md` §Naming (this section of that file is still current).

## 6. Branch conventions

3-branch flow: `main` (prod, tech-lead-only merges) / `develop` (integration, PR-only) /
`feature/*`, `fix/*`, `chore/*`, `docs/*`, `refactor/*` (dev pushes directly). Conventional Commits
(`feat|fix|chore|docs|refactor|test(<scope>): <desc>`). PR base is always `develop`, never `main` — a GitHub
Action auto-closes non-lead PRs targeting `main`. Full rules: `<API_REPO>/docs/CONVENTIONS.md`.

## 7. Review policy

- Every PR needs the CODEOWNER (`@dotuananhtb`) approval — enforced by branch protection.
- Reviewer checklist (existing, see `<API_REPO>/docs/CONVENTIONS.md` §Code review checklist) covers: permission check at
  backend, no trusting `req.body` for role/owner fields, response goes through `toDto()`, service throws
  `ApiError` + controller uses `asyncHandler`, socket emit happens in the service after DB commit, external calls
  go through an adapter, smoke test matches the code.
- This AI workspace adds an **AI Self Review** step before human review (see
  `.ai/workflows/review-workflow.md` and `.ai/agents/reviewer-agent.md`) — it does not replace the human
  CODEOWNER approval, it precedes it.
- CI currently only gates *deploy* (SSH on push to `develop` / tag), not test/lint on PRs — see
  `docs/project-analysis.md` §12 for the gap. Do not assume a green PR check means tests passed; run
  `npm test` / `pytest` yourself per `<API_REPO>/TESTING.md`.

## 8. Documentation policy

- **Code and comments:** English only.
- **`/docs`, `specs/`, `templates/` content:** Vietnamese (matches existing `<API_REPO>/docs/CONVENTIONS.md`,
  `<API_REPO>/docs/ADR-admin-api-migration.md`, `<API_REPO>/docs/superpowers/specs/*`).
- **`<API_REPO>/CLAUDE.md`, `.ai/` (this folder — agents, workflows, prompts, checklists):** English, mirroring the existing
  `<API_REPO>/CLAUDE.md` precedent for AI-facing instruction files, and to keep this workspace portable if it's later
  extracted into a shared repo or copied into `exe-web`/`exe-admin`.
- **Chat with the human dev/BA:** Vietnamese, per `<API_REPO>/CLAUDE.md`. (This context file itself is English because it's
  operational instruction for AI agents, not conversation.)
- Never add `Co-Authored-By` or AI attribution to commits or PRs (hard rule, `<API_REPO>/CLAUDE.md`).

## 9. AI workflow (Idea → Merge)

This repo now supports the full pipeline:

```
Idea → AI Brainstorm → BA Review → Specification → Tech Lead Design → Technical Review
     → Task Generation → Development → AI Self Review → Tech Lead Review → Merge
```

- Roles and rules: `.ai/agents/{ba,techlead,developer,reviewer}-agent.md`
- Phase-by-phase process, gates, exit criteria: `.ai/workflows/feature-workflow.md`,
  `.ai/workflows/review-workflow.md`, `.ai/workflows/release-workflow.md`
- Reusable prompts for each phase: `.ai/prompts/*.md`
- Document templates: `templates/*.md`
- Checklists per role: `.ai/checklists/*.md`
- Feature specs live in `specs/<feature-name>/` (see `specs/example-feature/` for the expected shape). This
  formalizes the pattern that already existed ad hoc in `<API_REPO>/docs/superpowers/specs/` + `<API_REPO>/docs/superpowers/plans/` —
  those two existing files are the closest real precedent for `spec.md`/`design.md` + `tasks.md`; read them once
  to calibrate tone and level of detail before writing a new spec.
- Existing `<API_REPO>/docs/active/<feature>/` (adaptive-learning, ipa-pronunciation-scoring, ...) are pre-existing
  multi-phase roadmaps with a `.status` file. They predate this workspace and are **not** migrated into
  `specs/` retroactively — leave them where they are. New work should use `specs/`.

## 10. Repository conventions this workspace must never violate

- Never write application/business code as a side effect of preparing this AI workspace.
- Never move or rename `<API_REPO>/docs/ADR-admin-api-migration.md` — new ADRs go in `docs/adr/` alongside it; the old file
  stays where it is for backward compatibility with existing links.
- Never commit `.env` files or secrets (`.gitignore` already blocks these — don't add exceptions).
- Never bypass the `develop`-only PR-base rule, even for AI-workspace-only changes.
