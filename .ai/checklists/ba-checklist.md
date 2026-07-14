# BA Checklist — Specification phase

> Used by the human BA (and self-checked by `.ai/agents/ba-agent.md`) before marking a spec "approved — ready
> for design" in `.ai/workflows/feature-workflow.md` step 4.

## Before writing

- [ ] The idea has a single, clear goal — if it's actually 2+ features, split it now, not after the design is
      half-written.
- [ ] Checked `<API_REPO>/docs/ARCHITECTURE.md` and existing `<API_REPO>/services/api/src/modules/` for overlap — is this really new,
      or an extension of something that exists?
- [ ] Checked `docs/adr/` — does an existing decision constrain this feature?

## Spec content (`spec.md`)

- [ ] Goal is one unambiguous sentence.
- [ ] Affected repos/surfaces named explicitly (api / web / admin / mobile), even if only `api` will be built now.
- [ ] In-scope and out-of-scope both listed; every out-of-scope item has a stated reason.
- [ ] Business decisions that need a human call (pricing, compliance, data retention) are listed under
      "Quyết định nghiệp vụ cần chốt", not silently decided.
- [ ] No implementation detail leaked into the spec (routes, schema field names, library choices) — that's
      `design.md`'s job.
- [ ] Open questions section is either empty, or lists exactly what's blocking approval.

## Acceptance criteria (`acceptance.md`)

- [ ] Every in-scope item from `spec.md` has at least one matching, checkable acceptance criterion.
- [ ] Each criterion is checkable by a human or a test — no "works well", "is fast" without a number when
      performance actually matters.
- [ ] Error/edge-case criteria included, not just the happy path.
- [ ] IDs (`AC-1`, `AC-2`, ...) assigned so `design.md`/`tasks.md`/`review.md` can reference them.

## Before marking "approved"

- [ ] No open questions remain unresolved.
- [ ] A second person (or the human BA if the AI drafted it) has actually read the spec, not skimmed it.
- [ ] Content is in Vietnamese per `.ai/project-context.md` §8.
