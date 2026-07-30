---
description: Code audit - readability, bad practices and security
argument-hint: [optional-path-or-glob]
allowed-tools: Read, Grep, Glob, Task, Write, Edit
model: claude-sonnet-5
disable-model-invocation: true
---

# Code Audit

Identifies everything that makes the code cumbersome, hard to understand, fragile, or insecure, and produces an actionable plan.

**Requested scope:** $ARGUMENTS
(If no scope was specified, audit the whole repository.)

**Read the harness conventions first:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. They define the layout, the task file format, and the `_Agent:_`/`_Model:_` assignment. Mandatory.
If the project has its own `.claude/harness/CONVENCIONES.md`, **that copy wins** — it's a deliberate override for that repo.

## Dimensions

1. **Readability and maintainability**: excessively long functions, ambiguous or inconsistent names, deep nesting, duplicated code, tight coupling, mixed responsibilities, missing layer separation, dead code, stale or missing comments where they're critical, and accidental complexity (unnecessary abstractions, over-engineering).

2. **Bad practices**: anti-patterns for the stack in use (check first which frameworks/languages are present and apply their conventions). Examples: missing error handling or silent catch, unawaited promises, shared-state mutation, magic numbers/strings, hardcoded configuration, missing typing or `any`, business logic in handlers, missing input validation, unnecessary or outdated dependencies.

3. **Security**: hardcoded or committed secrets/credentials/API keys, injection (SQL, command, template), missing validation of external inputs (webhooks, query params, payloads), weak or missing auth/authz on endpoints, sensitive data exposed in logs or errors, permissive CORS, dependencies with known CVEs, insecure token/session handling, missing rate limiting on public endpoints.

## Execution strategy

Use subagents (Task) to minimize main-context consumption:

- **Explore the structure yourself first** (directory tree, package.json/requirements, configs, entry points) to understand the architecture. Don't launch agents blindly.
- **Parallel subagents only for heavy reading**, split by area (handlers/API, business logic, infra/config/security). Each returns a compact summary: finding, `file:line`, severity, minimal evidence — not full code blocks.
- **Don't use agents for few or small files** — read them directly.
- **Explicit, non-overlapping paths/globs** per agent.
- You consolidate, deduplicate, prioritize, and write the deliverables. Subagents do NOT write files.

## Severity

- 🔴 **Critical**: exploitable security risk, data loss, or a severe latent bug.
- 🟠 **High**: bad practice with real impact on reliability or maintainability.
- 🟡 **Medium**: technical debt that slows development down.
- 🟢 **Low**: cosmetic or style improvement.

## Deliverables

Write them **in this order**: report first, then tasks. The report is what's expensive to reproduce.

### 1. `docs/audit/AUDIT-<YYYY-MM-DD>.md` — report

Create the folder if it doesn't exist. Structure:

- **Executive summary**: overall state in 3-5 lines, count by severity.
- **Project context**: detected stack, observed architecture, approximate size.
- **Findings by dimension**: each with an ID (`A-01`, `A-02`, ...), severity, exact location (`file:line`), why it's a problem, and recommendation. Code snippets only if indispensable and brief.
- **What's working well**: briefly, so it doesn't get broken while refactoring.
- **Recommended prioritization**: order of attack and justification.

### 2. `docs/tasks/audit-<YYYY-MM-DD>-tasks.md` — tasks

One file **per audit run**; its name mirrors the report's. Format in `CONVENCIONES.md`, with `# Tasks: Audit <scope> — <YYYY-MM-DD>` and `> Source: docs/audit/AUDIT-<YYYY-MM-DD>.md`.

Phases by severity: **Phase 1** = critical and security, **Phase 2** = high-impact bad practices, **Phase 3** = readability and technical debt.

Traceability via `_Findings: A-NN (file:line)_`. Every 🔴 and 🟠 finding must have a task; 🟡/🟢 ones can be grouped into cleanup tasks.

If you re-audit an area that was already audited, this generates a new file — don't touch or merge with previous ones. Before writing, check whether `docs/tasks/` has pending tasks from a previous audit of the same scope and mention it in chat; consolidating or deleting the old ones is my call.

## Final rules

- Do NOT modify any code file. Read-only + the report + the task file.
- Be specific: "`processWebhook` in `src/handlers/gupshup.ts:45` is 180 lines and mixes validation, parsing, and persistence" is useful; "there are long functions" is not.
- If the repository is very large, prioritize: entry points, exposed endpoints, core logic, configuration/secrets. State in the report what was left out of scope.
- When done, give me in chat the 3 most critical findings and the path to both deliverables.
