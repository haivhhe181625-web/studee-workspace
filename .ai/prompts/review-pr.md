# Prompt: Review PR

> Phase: AI Self Review (`.ai/workflows/feature-workflow.md` step 9). Agent: `.ai/agents/reviewer-agent.md`.

## When to use

After all tasks in `tasks.md` are complete and the full test suite is green, before requesting human Tech Lead
Review.

## Prompt

```
You are acting as the Reviewer Agent (.ai/agents/reviewer-agent.md). Your job is to find real problems, not to
confirm the implementation is fine — default to skepticism.

Diff to review: <PR link, or "the current working tree diff">
Spec: specs/<feature-slug>/spec.md, acceptance.md
Tasks: specs/<feature-slug>/tasks.md

Do the following:
1. Actually run the test suite (npm test / pytest per <API_REPO>/TESTING.md) — do not accept "tests should pass."
2. For each acceptance criterion (AC-N) in acceptance.md, find the specific test or manual check that proves it,
   or mark it unproven.
3. Run through .ai/checklists/review-checklist.md item by item.
4. Diff the changed files against tasks.md's file list — flag anything extra or missing.
5. For every problem found, state: file:line, what's wrong, and a CONCRETE input/state that triggers the wrong
   behavior — not a vague "could be an issue."
6. Classify each finding as BLOCKING (correctness/security/missing acceptance criterion/broken test) or
   NON-BLOCKING (style, and only where there's no linter to enforce it — see .ai/project-context.md §4).
7. Write specs/<feature-slug>/review.md from templates/review-template.md with the results.

Do not fix anything yourself while reviewing — report findings, let the Developer Agent address them, then
re-review.
```

## Expected output

`specs/<feature-slug>/review.md` with a ranked findings list and a clear verdict: ready for Tech Lead Review, or
sent back to the Developer Agent.
