# Prompt: Generate Tasks

> Phase: Task Generation (`.ai/workflows/feature-workflow.md` step 7). Agent: `.ai/agents/techlead-agent.md`.

## When to use

After `design.md` has passed Technical Review, break it into an ordered, implementable task list.

## Prompt

```
You are acting as the Tech Lead Agent (.ai/agents/techlead-agent.md).

Design to break down: specs/<feature-slug>/design.md (approved).

Do the following:
1. Create specs/<feature-slug>/tasks.md from templates/tasks-template.md.
2. Order tasks so dependencies come first (schema before service before route before test-only task).
3. Size each task to roughly one PR-worth of work, independently testable.
4. For code the touched module already tests (check for an existing __tests__/tests/ folder next to it), write
   each task test-first: a failing-test step, a "run + confirm FAIL" step with the actual expected failure
   reason, an implement step, a "run + confirm PASS" step, a commit step — mirroring the style already used in
   <API_REPO>/docs/superpowers/plans/2026-06-15-user-profile-edit-avatar.md. Do not write vague steps like "implement the
   feature" — every step has a concrete action and a concrete verification command.
5. Add a final task that runs the full relevant suite and does a Self-Review pass tracing every acceptance
   criterion (AC-N) to the task that covers it.
6. Self-check against .ai/checklists/techlead-checklist.md §Task generation before presenting the result.

Output in Vietnamese (per .ai/project-context.md §8); code/commands stay as literal code.
```

## Expected output

`specs/<feature-slug>/tasks.md`, ready to be handed to the Developer Agent / a human dev, one task at a time.
