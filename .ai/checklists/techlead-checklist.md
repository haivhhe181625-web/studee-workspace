# Tech Lead Checklist — Design & Technical Review

> Used by the human Tech Lead (and self-checked by `.ai/agents/techlead-agent.md`) before marking a design
> "approved" in `.ai/workflows/feature-workflow.md` step 6, and to decide when the full pipeline is warranted.

## When to skip the full pipeline

Not every change needs `spec.md` → `design.md` → `tasks.md`. Use judgment:

- **Skip the pipeline** for: typo fixes, dependency bumps, config-only changes, one-line bug fixes with an
  obvious root cause, doc-only changes.
- **Use the full pipeline** for: new endpoints, new modules, schema changes, anything touching auth/permissions,
  anything touching a cross-service or cross-repo contract, anything a BA would want to know shipped.
- When unsure, err toward at least a `spec.md` + `design.md` — cheap insurance against scope creep.

## Design content (`design.md`)

- [ ] Every acceptance criterion in `acceptance.md` maps to something concrete in the design (§12 table filled).
- [ ] Extends an existing module where possible; new module choice is justified, not default.
- [ ] Data model changes list migration/backfill impact, if any.
- [ ] Permission strings (new `<resource>:<action>` entries) are named and will be added to
      `src/constants/permissions.js`.
- [ ] Tenant scoping addressed if the feature touches center-scoped data (`User.centerId`).
- [ ] External calls go through an adapter — no `require('openai')` (or equivalent) proposed directly in a
      service.
- [ ] Admin surface changes (if any) delegate to module services — no business logic proposed inside
      `src/admin-api/`.
- [ ] Cross-service (`api↔llm`/`api↔cat`) or cross-repo (`api↔web`/`api↔admin`) contracts are written to
      `contracts/*.md`, not left implicit in prose.
- [ ] Does this design conflict with an existing `docs/adr/` entry? If yes, a new ADR explicitly supersedes it
      (never silently contradicted).
- [ ] Does this design *make* a decision worth an ADR (new external dependency, new architectural pattern,
      reversal of a prior decision)? If yes, `docs/adr/NNNN-<slug>.md` drafted.

## Task generation (`tasks.md`)

- [ ] Each task is independently testable and sized for roughly one PR.
- [ ] Tasks are ordered so dependencies come first (e.g. schema before service before route before test-only
      task).
- [ ] Tests-first structure used where the touched code already does TDD (check for existing `__tests__`/`tests/`
      alongside the module being changed).
- [ ] Every task has a concrete verification command (`Run: ... Expected: ...`), not "make sure it works."

## Before marking design "approved"

- [ ] Reviewer Agent's Technical-Review findings (if any) are resolved or explicitly accepted with a stated
      reason.
- [ ] `research.md` questions (if any were opened) have a conclusion, not left "TBD."
