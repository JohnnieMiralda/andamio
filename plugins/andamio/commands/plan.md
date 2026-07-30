---
description: Generates the implementation plan for a spec under docs/tasks/
argument-hint: [path-to-spec.md]
allowed-tools: Read, Grep, Glob, Write, Edit
model: claude-sonnet-5
disable-model-invocation: true
---

# Implementation Plan from Spec

**Spec to process:** $ARGUMENTS
(If no path was specified, use the most recent file in `docs/specs/`.)

## Your task

1. **Read the harness conventions:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. They define the layout, the task file format, the regeneration rules, and the `_Agent:_`/`_Model:_` assignment. Mandatory.
   If the project has its own `.claude/harness/CONVENCIONES.md`, **that copy wins** — it's a deliberate override for that repo.
2. **Read the full spec.** If its "Open questions" section has points that block implementation, stop and ask me how to resolve them before generating the plan.
3. **Explore the codebase the minimum necessary** for the tasks to be concrete: which files/modules each task touches and what conventions already exist (folder structure, handler patterns, test framework). This isn't an audit — just enough to ground the tasks.
4. **Write `docs/tasks/<slug>-tasks.md`**, with the same slug as the spec. If the file already exists, apply the regeneration rules from `CONVENCIONES.md` — **preserve existing `[x]`**, don't rewrite from scratch.

## /plan-specific rules

- Traceability via `_Requirements: N.M_`. **EVERY requirement in the spec must be covered by at least one task.** If any isn't implementable, say so explicitly in chat.
- Order phases by dependency: schema/data first, core logic next, robustness (idempotency, errors, retries) last.
- Don't modify code in this session. Read-only + writing to `docs/tasks/` only.
- When done, summarize in chat: the file path, how many tasks and phases, which requirements went uncovered (if any), and — if it was a regeneration — what was added, changed, or made obsolete.
