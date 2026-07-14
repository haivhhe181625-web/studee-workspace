# Prompt: Update Documentation

> Used at the tail end of Development (`.ai/workflows/feature-workflow.md` step 8) or as a standalone doc-only
> task. Agent: `.ai/agents/developer-agent.md` (for narrow, in-scope updates) or a human.

## When to use

A feature changed user-visible or API-visible behavior and the relevant doc (not this AI workspace's own
infrastructure) needs updating — e.g. a new endpoint should appear in `docs/api/<module>/`, or `README.md`'s
service table changed.

## Prompt

```
You are acting as the Developer Agent (.ai/agents/developer-agent.md).

Feature just shipped: specs/<feature-slug>/spec.md
Docs likely affected: <list, e.g. docs/api/<module>/, README.md, <API_REPO>/docs/ONBOARDING.md>

Do the following:
1. Update ONLY the docs directly describing behavior this feature changed — do not take this as license to
   rewrite or "clean up" unrelated documentation (<API_REPO>/CLAUDE.md §3 Surgical Changes applies to docs too).
2. Match the existing doc's format and language exactly (Vietnamese for /docs, per .ai/project-context.md §8).
3. If you find an existing doc that is already stale or contradicts <API_REPO>/CLAUDE.md/an ADR (e.g. the known
   <API_REPO>/docs/api/CONVENTIONS.md AdminJS-era section, see docs/project-analysis.md §5), do NOT silently rewrite it as
   a side effect of this task — flag it to the human Tech Lead as a separate follow-up instead.
4. If this feature is significant enough to warrant a docs/adr/ entry that wasn't already drafted during
   design, flag it rather than adding one unprompted.
```

## Expected output

Minimal, scoped doc updates matching what actually shipped — plus a short list of any stale-doc issues noticed
but deliberately not touched.
