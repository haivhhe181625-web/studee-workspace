# Prompt: Generate Tests

> Used within Development (`.ai/workflows/feature-workflow.md` step 8), either as part of a task's test-first
> step or to backfill coverage the Reviewer Agent flagged as missing. Agent: `.ai/agents/developer-agent.md`.

## When to use

- As the "write a failing test" step inside `implement-task.md`.
- When AI Self Review (`review-pr.md`) finds an acceptance criterion with no test proving it.

## Prompt

```
You are acting as the Developer Agent (.ai/agents/developer-agent.md).

Behavior to test: "<describe the acceptance criterion or task step>"
Existing test file (if extending): <path, or "none — new file">

Do the following:
1. Match the existing test style for this module exactly — look at a neighboring test file in the same
   directory (e.g. <API_REPO>/services/api/src/__tests__/*.test.js uses Jest + Supertest + mongodb-memory-server;
   <API_REPO>/services/cat and <API_REPO>/services/llm use pytest) before writing anything.
2. Cover: the happy path, the documented error/edge cases from acceptance.md, and permission/auth checks if the
   behavior is an endpoint.
3. Use real assertions on response status/body/DB state — not `expect(true).toBe(true)`-style placeholders.
4. If mocking an adapter/external call, mock at the adapter boundary (per <API_REPO>/docs/api/CONVENTIONS.md's adapter
   pattern), not by reaching into the provider SDK.
5. Run the new test and confirm it currently FAILS for the expected reason (the feature isn't implemented yet)
   before implementing anything.
```

## Expected output

A test file (new or extended) that fails for the right reason before implementation, and the exact command used
to run it.
