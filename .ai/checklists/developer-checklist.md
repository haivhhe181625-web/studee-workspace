# Developer Checklist — Implementation

> Used by the human dev (and self-checked by `.ai/agents/developer-agent.md`) while working through
> `tasks.md` in `.ai/workflows/feature-workflow.md` step 8, before handing off to AI Self Review.

## Per task

- [ ] Failing test written and confirmed failing (with the actual error, not assumed) before implementing —
      where the touched code already follows TDD.
- [ ] Implementation matches the task's file list; any extra file touched is called out, not silent.
- [ ] Test confirmed passing after implementation, with real command output, not assumed.
- [ ] Commit is scoped to this task only (`git add <specific files>`, never `git add .`).
- [ ] Commit message follows Conventional Commits, no `Co-Authored-By`/AI attribution.

## Convention compliance (per `<API_REPO>/docs/api/CONVENTIONS.md`, `<API_REPO>/docs/CONVENTIONS.md`, `<API_REPO>/CLAUDE.md`)

- [ ] File/variable/class naming matches the naming table.
- [ ] Middleware chain order: `verifyToken → verifyPermission → validate(schema) → asyncHandler(handler)`.
- [ ] Response goes through `toDto()` — never `res.json(rawDbDoc)`.
- [ ] Service throws `ApiError`; controller uses `asyncHandler`, no manual try/catch for expected errors.
- [ ] Socket emits happen in the service, after DB commit — not in controller/route.
- [ ] External provider calls go through an adapter in `src/adapters/`.
- [ ] `req.body` never trusted for `role`/`permissions`/`isActive` or any owner/tenant field — always from
      `req.user`.
- [ ] New admin capability (if any): REST lives only in `src/admin-api/`, no business logic there — delegates to
      a module service.
- [ ] New permission strings registered in `src/constants/permissions.js`.

## Before requesting AI Self Review

- [ ] Every checkbox in `tasks.md` is checked off.
- [ ] Full relevant test suite run (`npm test` / `pytest`, per `<API_REPO>/TESTING.md`), not just the new test — no
      regressions.
- [ ] `git status` reviewed — no unrelated files, no `.env`, no stray debug files staged.
- [ ] If a new env var was introduced, `.env.example` updated to match.
- [ ] If the change affects a cross-service contract (`api↔llm`/`api↔cat`), both sides updated in this PR (or the
      other side's PR is linked).
