# Prompt: Generate Design

> Phase: Tech Lead Design (`.ai/workflows/feature-workflow.md` step 5). Agent: `.ai/agents/techlead-agent.md`.

## When to use

After `spec.md` and `acceptance.md` are BA-approved and it's time to decide *how* to build it.

## Prompt

```
You are acting as the Tech Lead Agent (.ai/agents/techlead-agent.md). Read .ai/project-context.md,
<API_REPO>/docs/ARCHITECTURE.md, <API_REPO>/docs/CONVENTIONS.md, <API_REPO>/docs/api/CONVENTIONS.md, and any relevant docs/adr/ entries first.

Spec to design against: specs/<feature-slug>/spec.md and acceptance.md.

Do the following:
1. Read the actual code for any module this feature extends before proposing changes to it — don't assume
   structure from the spec.
2. If there's an unresolved technical question that blocks a design decision, write specs/<feature-slug>/
   research.md from templates/research-template.md FIRST, answer it, then proceed.
3. Create specs/<feature-slug>/design.md from templates/design-template.md. Every acceptance criterion (AC-N)
   must map to something concrete in §12 of the design.
4. If the feature touches a cross-service (api↔llm, api↔cat) or cross-repo (api↔web, api↔admin) boundary,
   write specs/<feature-slug>/contracts/<name>.md from templates/contract-template.md for each boundary.
5. Create specs/<feature-slug>/data-model.md if the data model is non-trivial (otherwise inline it in design.md
   §3).
6. If this design makes a decision worth recording long-term (new dependency, new pattern, reversing a prior
   ADR), draft docs/adr/NNNN-<slug>.md — check docs/adr/0000-index.md for the next number.
7. Self-check against .ai/checklists/techlead-checklist.md before presenting the result.

Output in Vietnamese (per .ai/project-context.md §8).
```

## Expected output

`design.md` (+ `research.md`/`data-model.md`/`contracts/*.md`/new ADR as needed), ready for Technical Review.
