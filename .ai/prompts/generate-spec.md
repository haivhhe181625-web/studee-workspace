# Prompt: Generate Spec

> Phase: Specification (`.ai/workflows/feature-workflow.md` step 4). Agent: `.ai/agents/ba-agent.md`.

## When to use

After BA Review has picked a framing (from brainstorm output or directly from a clear idea) and it's time to
write the actual spec the Tech Lead will design against.

## Prompt

```
You are acting as the BA Agent (.ai/agents/ba-agent.md). Read .ai/project-context.md first.

Framing to spec out: "<paste the chosen framing / idea>"

Do the following:
1. Create specs/<feature-slug>/spec.md from templates/spec-template.md. Fill every section — do not leave
   template placeholders in the output.
2. Create specs/<feature-slug>/acceptance.md from templates/acceptance-template.md, with checkable criteria
   (AC-1, AC-2, ...) covering every "in scope" item in spec.md, including error/edge cases.
3. Name every affected repo/surface explicitly (api/web/admin/mobile), even if only exe-api will be built now.
4. If anything is ambiguous, put it under "Câu hỏi mở" in spec.md instead of guessing — do not mark the spec
   ready for approval while open questions remain.
5. Self-check against .ai/checklists/ba-checklist.md before presenting the result.
6. Do NOT propose implementation details (routes, schemas, library choices) — that belongs to design.md, not
   spec.md.

Output in Vietnamese (per .ai/project-context.md §8).
```

## Expected output

`specs/<feature-slug>/spec.md` and `acceptance.md`, ready for human BA Review sign-off.
