# Feature Workflow — Idea → Merge

> The full pipeline this workspace supports. Each phase names: **Responsible role**, **Required documents**,
> **Approval gate**, **AI responsibilities**, **Human responsibilities**, **Exit criteria**.
>
> This formalizes a pattern that already existed informally in this repo (`<API_REPO>/docs/superpowers/specs/` +
> `<API_REPO>/docs/superpowers/plans/`, and the phased `<API_REPO>/docs/active/<feature>/` roadmaps) — it does not replace those
> existing docs, it gives future features a repeatable, role-separated version of the same idea.

```
Idea → AI Brainstorm → BA Review → Specification → Tech Lead Design → Technical Review
     → Task Generation → Development → AI Self Review → Tech Lead Review → Merge
```

---

## 1. Idea

**Responsible role:** anyone (BA, dev, founder, center partner feedback).
**Required documents:** none yet — a sentence or a support ticket is enough.
**Approval gate:** none.
**AI responsibilities:** none yet.
**Human responsibilities:** capture the idea somewhere durable (issue, message) before it evaporates.
**Exit criteria:** the idea is written down somewhere a BA can pick it up.

## 2. AI Brainstorm

**Responsible role:** BA Agent (`.ai/agents/ba-agent.md`), prompted via `.ai/prompts/brainstorm-feature.md`.
**Required documents:** the raw idea.
**Approval gate:** none — this is exploratory.
**AI responsibilities:** expand the idea into 2-4 concrete framings (different scope levels, different
user flows), surface obvious open questions, check `<API_REPO>/docs/ARCHITECTURE.md`/existing modules for conflicts or
reuse opportunities.
**Human responsibilities:** pick a framing, or ask for a different angle.
**Exit criteria:** one framing selected to turn into a spec.

## 3. BA Review

**Responsible role:** human BA/product owner, assisted by BA Agent.
**Required documents:** brainstorm output.
**Approval gate:** human sign-off that this is worth speccing.
**AI responsibilities:** draft the spec skeleton for the chosen framing.
**Human responsibilities:** confirm business intent, priority, and constraints (pricing, compliance, timeline).
**Exit criteria:** green light to write the full spec.

## 4. Specification

**Responsible role:** BA Agent, prompted via `.ai/prompts/generate-spec.md`.
**Required documents:** `templates/spec-template.md`, `templates/acceptance-template.md`.
**Approval gate:** human BA approves `spec.md` + `acceptance.md` as accurate and complete.
**AI responsibilities:** write `specs/<feature-slug>/spec.md` and `acceptance.md`; name every affected
repo/surface (api/web/admin/mobile); list explicit out-of-scope items with reasons.
**Human responsibilities:** review against `.ai/checklists/ba-checklist.md`; resolve open questions.
**Exit criteria:** spec + acceptance criteria approved, feature folder `specs/<feature-slug>/` exists.

## 5. Tech Lead Design

**Responsible role:** Tech Lead Agent (`.ai/agents/techlead-agent.md`), prompted via
`.ai/prompts/generate-design.md` (and `.ai/prompts/generate-contract.md` for cross-service/cross-repo contracts).
**Required documents:** approved `spec.md`/`acceptance.md`; `templates/design-template.md`,
`templates/research-template.md`, `templates/contract-template.md`.
**Approval gate:** none yet — draft stage.
**AI responsibilities:** produce `design.md`, `data-model.md`, `research.md` (if a technical question needs
investigating), `contracts/*.md`. Propose a `docs/adr/` entry if the design makes a durable architectural
decision.
**Human responsibilities:** available for questions when the design space is ambiguous; the Tech Lead Agent
should ask rather than guess on anything spec doesn't resolve.
**Exit criteria:** design draft ready for Technical Review.

## 6. Technical Review

**Responsible role:** human Tech Lead, assisted by Reviewer Agent (`.ai/agents/reviewer-agent.md`) acting
adversarially against the design.
**Required documents:** `design.md`, `data-model.md`, `contracts/*.md`, `.ai/checklists/techlead-checklist.md`.
**Approval gate:** human Tech Lead approves the design.
**AI responsibilities:** Reviewer Agent pressure-tests the design against `acceptance.md` and existing ADRs
before the human spends time on it.
**Human responsibilities:** final call on architecture tradeoffs, especially anything touching cross-repo
contracts or a new external dependency.
**Exit criteria:** design approved; ready for task generation.

## 7. Task Generation

**Responsible role:** Tech Lead Agent, prompted via `.ai/prompts/generate-tasks.md`.
**Required documents:** approved `design.md`; `templates/tasks-template.md`.
**Approval gate:** none — mechanical step from an approved design.
**AI responsibilities:** break the design into ordered, independently-testable tasks in `tasks.md`, each with a
verification command, mirroring the TDD-checkbox style in `<API_REPO>/docs/superpowers/plans/*.md`.
**Human responsibilities:** skim for task sizing (one PR-worth of work per task) and correct sequencing.
**Exit criteria:** `tasks.md` ready for implementation.

## 8. Development

**Responsible role:** Developer Agent (`.ai/agents/developer-agent.md`) or a human dev, prompted via
`.ai/prompts/implement-task.md` and `.ai/prompts/generate-tests.md`.
**Required documents:** `tasks.md`, `design.md`, `contracts/*.md`.
**Approval gate:** none per-task; the whole feature gates at AI Self Review / Tech Lead Review.
**AI responsibilities:** implement one task at a time, test-first where the codebase already does TDD, commit
per task, following `<API_REPO>/docs/CONVENTIONS.md` and `<API_REPO>/docs/api/CONVENTIONS.md`.
**Human responsibilities:** available to unblock ambiguity; spot-check risky tasks as they land.
**Exit criteria:** all tasks in `tasks.md` checked off, full test suite green.

## 9. AI Self Review

**Responsible role:** Reviewer Agent, prompted via `.ai/prompts/review-pr.md`.
**Required documents:** the diff/PR; `acceptance.md`; `.ai/checklists/review-checklist.md`.
**Approval gate:** AI review must have zero unresolved blocking findings before human review is requested.
**AI responsibilities:** run the reviewer checklist, verify acceptance criteria are actually met, run the test
suite, produce a ranked findings list.
**Human responsibilities:** none yet — this step exists specifically to save human review time.
**Exit criteria:** blocking findings resolved (or explicitly the Developer Agent has addressed them);
`review.md` written.

## 10. Tech Lead Review

**Responsible role:** human Tech Lead (CODEOWNER).
**Required documents:** PR, `review.md`, `.ai/checklists/review-checklist.md`.
**Approval gate:** CODEOWNER approval (enforced by branch protection, per `.github/CODEOWNERS`).
**AI responsibilities:** none — this is a human-only gate by design.
**Human responsibilities:** approve, request changes, or reject. This is the last checkpoint before merge.
**Exit criteria:** PR approved.

## 11. Merge

**Responsible role:** human Tech Lead.
**Required documents:** approved PR.
**Approval gate:** none further.
**AI responsibilities:** none.
**Human responsibilities:** squash-and-merge into `develop` per `<API_REPO>/docs/CONVENTIONS.md`; delete the branch.
**Exit criteria:** merged to `develop`. Release to `main`/prod follows `.ai/workflows/release-workflow.md`.

---

## Notes

- Every phase's documents live under `specs/<feature-slug>/` except agent/workflow/template infrastructure
  itself (`.ai/`, `templates/`) and ADRs (`docs/adr/`).
- Small fixes/chores do not need the full pipeline — use judgment (see `.ai/checklists/techlead-checklist.md`
  §When to skip the full pipeline). The pipeline is for anything with real product/architecture surface area.
- This workflow currently runs entirely inside `exe-api`. A feature that also needs `exe-web`/`exe-admin` changes
  still gets **one spec** here naming all affected repos (`.ai/project-context.md` §0); the actual FE
  implementation happens in those repos' own PRs until this workspace is replicated or extracted there.
