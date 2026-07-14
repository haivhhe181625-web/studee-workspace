# Review Checklist — AI Self Review & Tech Lead Review

> Shared checklist for both review gates in `.ai/workflows/review-workflow.md`. The Reviewer Agent runs this
> during AI Self Review (step 9); the human Tech Lead runs the same list during Tech Lead Review (step 10) —
> by the time it reaches the human, everything here should already be green.

## Correctness

- [ ] Every acceptance criterion (`AC-N`) in `acceptance.md` is traced to a passing test or an explicit manual
      verification note — not assumed.
- [ ] Full test suite passes (`npm test` / `pytest`), actually run, output checked — not "should pass."
- [ ] Diff scope matches `tasks.md`'s file list; any extra file changed is explained.
- [ ] No regression in adjacent, unrelated tests.

## Security / permissions (per `<API_REPO>/docs/CONVENTIONS.md` §Code review checklist)

- [ ] Permission check present at backend for every new/changed endpoint — not just hidden in the FE.
- [ ] `req.body` doesn't leak into role/owner/tenant-sensitive fields.
- [ ] Response goes through `toDto()` — no `passwordHash`, `__v`, or other internal field exposed.
- [ ] Sensitive fields (`select: false`) only returned with the documented `?includeSensitive=1` +
      `<resource>:manage` pattern, if applicable.
- [ ] Tenant scope (`centerId`) enforced server-side, never trusted from the client.

## Architecture conformance

- [ ] Service throws `ApiError`; controller wrapped in `asyncHandler`.
- [ ] Socket emit happens in the service, after DB commit.
- [ ] External calls go through an adapter, not a direct SDK `require`.
- [ ] Admin surface changes contain no business logic outside `src/admin-api/` delegating to module services.

## Hygiene

- [ ] No secret or `.env` file in the diff.
- [ ] Commit messages are Conventional Commits, no AI attribution.
- [ ] No debug `console.log`/commented-out code left in.
- [ ] Docs updated if the change is user- or API-visible (e.g. `docs/api/<module>/`).

## Finding quality (for whoever wrote `review.md`)

- [ ] Every blocking finding names a concrete input/state and the resulting wrong behavior — not a vague
      "could be improved."
- [ ] Non-blocking (style) findings are labeled as such, not blocking merge in a repo with no linter to lean on.
