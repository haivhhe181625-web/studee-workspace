# Prompt: Generate Contract

> Phase: part of Tech Lead Design (`.ai/workflows/feature-workflow.md` step 5), whenever a feature crosses a
> service or repo boundary. Agent: `.ai/agents/techlead-agent.md`.

## When to use

The design touches `api↔llm`, `api↔cat` (cross-service, inside `exe-api`), or `api↔web`/`api↔admin`/mobile
(cross-repo). Use this any time a FE and BE (or two services) need to agree on a shape before either side is
built.

## Prompt

```
You are acting as the Tech Lead Agent (.ai/agents/techlead-agent.md). Read .ai/project-context.md and
<API_REPO>/docs/ARCHITECTURE.md §Cross-service contract / §Trust boundary first.

Boundary to contract: "<e.g. exe-api POST /api/ipa/score consumed by exe-web>"
Design reference: specs/<feature-slug>/design.md

Do the following:
1. Create specs/<feature-slug>/contracts/<name>.md from templates/contract-template.md.
2. State auth explicitly: JWT user vs. X-Service-Key vs. public — per <API_REPO>/docs/ARCHITECTURE.md's trust boundary
   rules. Never invent a new auth mechanism without flagging it as a design decision.
3. List every field in request/response with type and whether required — no "etc." or implicit fields.
4. List every error response (HTTP code + error code), not just the happy path.
5. If this changes an EXISTING contract (not a new one), explicitly call out what's breaking and the migration
   plan — do not treat a breaking change as routine.
6. State deployment sequencing if provider and consumer are in different repos (which must deploy first) —
   per .ai/workflows/release-workflow.md §3.
7. Include one realistic example request (curl or equivalent).

Output in Vietnamese (per .ai/project-context.md §8); keep field names/JSON as literal code (English, matching
the actual field names in the codebase).
```

## Expected output

One `contracts/<name>.md` per boundary, precise enough that the FE and BE (or two services) sides can be
implemented independently without further clarification.
