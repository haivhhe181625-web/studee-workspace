# Release Checklist

> Used by the human Tech Lead for `.ai/workflows/release-workflow.md`. Covers both a `develop` merge (auto-deploy
> to dev) and cutting a `main` release (prod).

## Before merging a PR to `develop`

- [ ] Tech Lead Review passed (CODEOWNER approval).
- [ ] AI Self Review findings all resolved or explicitly accepted.
- [ ] If a new env var was introduced: `.env.example` updated in this repo, and noted for
      `studee-env-setup` (out of this repo's scope to edit directly, but flag it — see
      `.ai/project-context.md` §0).
- [ ] If a cross-service contract changed (`api↔llm`/`api↔cat`): both sides are in this PR, or the paired PR is
      linked and merge order is stated.
- [ ] If a cross-repo contract changed (`api↔web`/`api↔admin`): the other repo's PR exists or is tracked, and
      `contracts/*.md` states which must deploy first.

## After merge to `develop`

- [ ] `deploy-dev.yml` completed successfully (check Actions tab).
- [ ] Dev environment smoke-tested for anything with real user-facing risk — don't assume auto-deploy = verified.
- [ ] If `acceptance.md` had criteria that need a live environment to check (not just unit/integration tests),
      verify them now on dev.

## Cutting a `main` release

- [ ] Only the human Tech Lead does this (`main` is lead-only per `.github/CODEOWNERS`).
- [ ] Reviewed the set of `develop` commits going into this release — anything half-finished or flagged as
      risky in its `review.md`?
- [ ] Tag follows `vX.Y.Z` semantic versioning.
- [ ] `deploy.yml` completed successfully.
- [ ] Prod health check passed (`curl` against `/health` per `<API_REPO>/docs/ONBOARDING.md`).

## Known gaps (do not assume otherwise)

- No CI job runs tests/lint on PRs today — a green PR check means nothing about test status; you must run tests
  yourself (`docs/project-analysis.md` §12).
- No documented rollback procedure beyond "redeploy the previous tag" — if something breaks in prod, this is a
  human Tech Lead judgment call, not an automated path.
