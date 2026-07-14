# Prompt: Implement Task

> Phase: Development (`.ai/workflows/feature-workflow.md` step 8). Agent: `.ai/agents/developer-agent.md`.

## When to use

For each individual task in `tasks.md`, one at a time — do not batch multiple tasks into a single invocation.

## Prompt

```
You are acting as the Developer Agent (.ai/agents/developer-agent.md). Read .ai/project-context.md,
<API_REPO>/docs/api/CONVENTIONS.md, and <API_REPO>/CLAUDE.md first.

Task to implement: Task <N> from specs/<feature-slug>/tasks.md.
Design reference: specs/<feature-slug>/design.md

Do the following:
1. Read every file this task touches before editing — confirm the current structure matches what design.md
   assumed; if it doesn't, stop and report the discrepancy instead of guessing.
2. Follow tasks.md's steps in order: write the failing test (if applicable), run it and confirm the actual
   failure, implement, run and confirm the actual pass, then commit.
3. Follow the module pattern, naming, and middleware chain exactly as in <API_REPO>/docs/api/CONVENTIONS.md — do not
   introduce a different pattern even if it seems cleaner; consistency with the rest of the codebase wins.
4. Touch ONLY the files this task lists. If you find you need an unlisted file, stop and say so rather than
   silently expanding scope.
5. Commit with `git add <specific files>` (never `git add .`), Conventional Commit message, no AI attribution.
6. Check off the task's steps in tasks.md with the actual command output noted.

Do not move to the next task in this same turn — stop after this task is committed and verified.
```

## Expected output

The task's code + test changes, committed, with `tasks.md` updated to reflect what actually ran (not what was
expected to run).
