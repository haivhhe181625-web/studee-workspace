# Review Workflow

> Detail on the two review gates inside `feature-workflow.md` (Technical Review, step 6; AI Self Review + Tech
> Lead Review, steps 9-10) — what "review" actually means at each gate, so it doesn't collapse into a single
> rubber-stamp.

## Why two AI-assisted review points, not one

- **Technical Review** catches design mistakes *before* code is written — cheapest place to catch a wrong
  approach.
- **AI Self Review** catches implementation mistakes *before* the human Tech Lead's time is spent — cheapest
  place to catch a bug, a missed acceptance criterion, or a convention violation.
- **Tech Lead Review** is the only gate that actually blocks merge (branch protection via `.github/CODEOWNERS`).
  It should be fast because the two AI-assisted passes already did the expensive checking.

## Technical Review (design stage)

| | |
|---|---|
| **Responsible** | Human Tech Lead |
| **Assisted by** | Reviewer Agent, playing devil's advocate against `design.md` |
| **Input** | `spec.md`, `acceptance.md`, `design.md`, `data-model.md`, `contracts/*.md` |
| **Checklist** | `.ai/checklists/techlead-checklist.md` |
| **Pass condition** | Every acceptance criterion traces to the design; no contradiction with an existing ADR; cross-repo contracts are written down, not implicit |
| **Fail action** | Design goes back to Tech Lead Agent / human Tech Lead for another pass — do not proceed to Task Generation on a design with open blocking findings |

## AI Self Review (post-implementation)

| | |
|---|---|
| **Responsible** | Reviewer Agent, prompted via `.ai/prompts/review-pr.md` |
| **Input** | The diff, `acceptance.md`, `tasks.md`, `<API_REPO>/docs/CONVENTIONS.md` §Code review checklist |
| **Checklist** | `.ai/checklists/review-checklist.md` |
| **Pass condition** | Zero unresolved blocking findings; full test suite green; diff scope matches `tasks.md` |
| **Fail action** | Findings go back to Developer Agent / human dev. Re-run AI Self Review after fixes — do not skip straight to Tech Lead Review with unresolved blocking findings |
| **Output** | `specs/<feature>/review.md` (from `templates/review-template.md`), or the equivalent posted as PR comments |

### Finding severity

- **Blocking** — correctness bug, security/permission gap, missing acceptance criterion, broken test. Must be
  resolved before Tech Lead Review.
- **Non-blocking** — style preference, minor naming inconsistency, optional refactor. Note it, don't block on it
  (this repo has no linter to lean on, see `.ai/project-context.md` §4 — use judgment, not zealotry).

## Tech Lead Review (merge gate)

| | |
|---|---|
| **Responsible** | Human Tech Lead (CODEOWNER) |
| **Input** | PR, `review.md`, actual diff |
| **Checklist** | `.ai/checklists/review-checklist.md` (same one, human pass) |
| **Pass condition** | CODEOWNER approval |
| **Fail action** | Request changes on the PR as normal |

This is a **human-only** gate. AI agents prepare everything needed to make it fast, but do not approve on the
Tech Lead's behalf, and never merge without this approval (`<API_REPO>/docs/CONVENTIONS.md` — no self-merge, no bypassing
CODEOWNERS).

## What "AI Self Review" must never become

- A substitute for running the actual test suite ("tests should pass" is not a review finding).
- A rubber stamp because the Developer Agent and Reviewer Agent are, in a given session, effectively the same
  model — treat the self-review prompt as genuinely adversarial (`.ai/prompts/review-pr.md` is written this way
  on purpose).
- A gate that silently patches code while "reviewing" it — findings get reported, then a separate fix step
  addresses them, so there's a record of what was wrong.
