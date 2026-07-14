# .ai/context/

Optional, deeper context files for a single complex domain — used when `.ai/project-context.md` (the single
high-level entry point) isn't enough detail for an agent to work confidently in that area without re-reading a
lot of source first.

## When to add a file here

Add `.ai/context/<domain>.md` when a domain is complex enough that every agent working in it re-derives the same
background (e.g. the assessment/MST-IRT scoring pipeline, the RBAC/tenant-scoping model, the cross-service
auth handshake). Don't create one preemptively for a simple module — `docs/api/<module>/` and the code itself
are enough for most areas.

## What goes in a context file

- The mental model a new agent needs before touching this domain (not a restatement of the code — point to
  files, don't paste them).
- Non-obvious invariants and gotchas (the kind of thing that caused a past bug — see
  `docs/project-analysis.md`/`<API_REPO>/TESTING.md` for two documented ones in `assessment`).
- Links to the authoritative docs (`<API_REPO>/docs/ARCHITECTURE.md`, `docs/adr/`, relevant `docs/api/<module>/`) rather than
  duplicating them.

## Currently empty

No domain has needed one yet — this workspace is still in its pilot phase (`docs/ai-workspace-report.md`). The
first candidate, if one is needed, is likely the assessment/MST-IRT pipeline (`<API_REPO>/services/api/src/modules/
assessment/`, `<API_REPO>/services/cat/`) given its size and the two correctness bugs already documented in `<API_REPO>/TESTING.md`.
