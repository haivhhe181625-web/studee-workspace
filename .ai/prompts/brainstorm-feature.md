# Prompt: Brainstorm Feature

> Phase: AI Brainstorm (`.ai/workflows/feature-workflow.md` step 2). Agent: `.ai/agents/ba-agent.md`.

## When to use

At the very start, when there's only a raw idea (a sentence, a support ticket, a piece of feedback) and it's not
yet clear what the actual feature boundaries should be.

## Prompt

```
You are acting as the BA Agent (.ai/agents/ba-agent.md). Read .ai/project-context.md first.

Raw idea: "<paste the idea here>"

Do the following:
1. Check <API_REPO>/docs/ARCHITECTURE.md and <API_REPO>/services/api/src/modules/ for anything that already does this or something
   close to it. Name the overlap if you find one.
2. Propose 2-4 distinct framings of this idea at different scope levels (e.g. minimal MVP vs. full version,
   or two genuinely different approaches to the same user need). For each, give: one-sentence description,
   rough affected repos/surfaces (api/web/admin/mobile), and the biggest open question.
3. List anything that looks like a business/product decision (pricing, compliance, data retention) rather than
   a scoping question — flag it, don't decide it.
4. Do NOT write spec.md yet. This is exploratory — output the framings and open questions only.

Output in Vietnamese (per .ai/project-context.md §8).
```

## Expected output

A short list of framings + open questions, not a spec. The human picks one (or asks for another angle) before
moving to BA Review.
