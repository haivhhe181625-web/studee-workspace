# Release Workflow

> What happens after a feature is merged to `develop`, through to production. This documents the existing
> release mechanics (`<API_REPO>/docs/CONVENTIONS.md`, `<API_REPO>/.github/workflows/*.yml`) in workflow-doc form — it does not change
> them.

## Current mechanics (as-is, unchanged by this workspace)

| Trigger | Target | What happens |
|---|---|---|
| Push to `develop` | Dev environment | `<API_REPO>/.github/workflows/deploy-dev.yml` — SSH deploy via `docker-compose.dev.yml` |
| Push tag `v*` | Production | `<API_REPO>/.github/workflows/deploy.yml` — SSH deploy via `docker-compose.prod.yml` |
| PR opened against `main` from a non-lead | — | `<API_REPO>/.github/workflows/restrict-main-pr.yml` auto-closes it |

There is currently **no CI job that runs tests/lint on PRs** (`docs/project-analysis.md` §12) — deploy triggers
are the only automation. Treat every merge to `develop` as auto-deployed to the dev server; do not merge
half-finished work expecting a staging gate to catch it.

## Release phases

### 1. Merge to `develop`

**Responsible:** human Tech Lead, after Tech Lead Review passes (`.ai/workflows/review-workflow.md`).
**Effect:** auto-deploys to the dev environment (`dev.studeeteam.cloud`, per `<ENV_BUNDLE>/README.md`).
**Human responsibility:** smoke-test the dev environment after deploy for anything with real user-facing risk.

### 2. Cut a release (`main`)

**Responsible:** human Tech Lead only (`main` merges are lead-only, enforced by `<API_REPO>/.github/CODEOWNERS` +
`restrict-main-pr.yml`).
**Required documents:** the set of merged `develop` commits being released; check `specs/*/acceptance.md` for
anything in this batch that needs a manual verification pass before prod.
**AI responsibilities:** on request, summarize what changed since the last tag (Conventional Commit messages
make this mechanical) — does not decide *when* to release.
**Human responsibilities:** decide release timing, merge `develop` → `main`, tag `vX.Y.Z`.
**Exit criteria:** tag pushed, `deploy.yml` completes, prod health check passes
(`curl` against `/health` per `<API_REPO>/docs/ONBOARDING.md`).

### 3. Cross-repo releases

Because `exe-api`, `exe-web`, `exe-admin` are independent repos with independent release cycles, a feature
spanning multiple repos needs its cross-repo contract (`specs/<feature>/contracts/*.md`) to specify **release
order** — e.g. "the `api` contract change must be deployed to dev before the `web` PR can be tested end-to-end."
The Tech Lead Agent should state this explicitly in `design.md` when it applies; it's easy to silently assume
simultaneity across repos that don't actually deploy together.

## Rollback

Not currently documented anywhere in the repo beyond "redeploy the previous tag." If a release needs rollback,
that's a human Tech Lead decision outside this workspace's scope — this workflow doc does not invent a rollback
procedure that doesn't exist in practice; flag this as a gap rather than papering over it (see
`docs/ai-workspace-report.md` §Known limitations).

## Exit criteria for the whole pipeline

A feature is "done" when: merged to `develop` → deployed to dev → (if applicable) verified against
`acceptance.md` on the dev environment → included in a subsequent `main` release at the Tech Lead's discretion.
