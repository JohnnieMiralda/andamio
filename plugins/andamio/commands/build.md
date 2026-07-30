---
description: Runs the next pending phase of a task file (or a specific phase, or all of them), orchestrating subagents
argument-hint: [path-to-tasks.md] [optional-phase | all]
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task, Agent
model: claude-sonnet-5
disable-model-invocation: true
---

# Execution Orchestrator

**Scope:** $ARGUMENTS

Argument interpretation:
- `<file>` alone → **run the next phase with tasks in `[ ]`**, not the whole file. One context per phase keeps this same prompt's hard rules from getting under-weighted as the context grows with every subagent return, every diff, and every test output.
- `<file> "Phase 4"` or `<file> 4` → that specific phase, whether or not there's pending work before it.
- `<file> all` → run the whole file, phase after phase, in a single context. Requires the explicit argument — it's no longer the default behavior.
- No file → use the most recent one in `docs/tasks/` and **confirm which one with me before starting**.

**How you find "the next phase":** walk the file's `### Phase N:` sections in order and take the first one with at least one task in `- [ ]` (ignore optional `[ ]*` ones if everything else in that phase is already `[x]`). That "ignore optional" clause only breaks ties against *other* phases that still have non-optional work pending — it's not a permanent block. If every phase's only remaining `[ ]` tasks are optional ones, select the first such phase instead of reporting nothing pending. If no `[ ]` of any kind remains anywhere, say so — the file is complete.

**You are the orchestrator.** You run on sonnet. You don't spin up another orchestrator — you coordinate, delegate, verify, and report.

## Step 0 — Before touching code

1. Read the full task file and its source (the spec or audit report from `> Source:`). Without the source you don't understand the tasks' traceability.
2. **Read the harness conventions:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. They define the layout, the task file format, the `_Agent:_`/`_Model:_` assignment, and the `### Phase N: Review corrections` format. Mandatory.
   If the project has its own `.claude/harness/CONVENCIONES.md`, **that copy wins** — it's a deliberate override for that repo.
3. Check the repo's state with `git branch --show-current` and `git status --short --untracked-files=all`. If the branch is the main one, **tell me and wait** — no direct work on main.
   Since one phase per `/build` invocation is the default, the tree will usually still carry the previous phase's uncommitted work by the time you run the next one — that's the expected steady state here, not an anomaly. Show me that status output and **ask me once** whether it's from this same task file's earlier phase. Don't try to guess the origin on your own — asking is cheaper and more reliable. If I confirm yes, continue; if not, stop.
4. List in chat what you're about to run: the phase's tasks, with their `_Agent:_` and `_Model:_`. If the argument didn't specify a phase, say which one you picked (the first with tasks in `[ ]`) and why. If any task in scope is `_Agent: human decision_`, **ask me about it now**, not partway through execution.

## Step 1 — Execute, task by task

In file order. Phases are ordered by dependency; don't reorder them or parallelize tasks from different phases.

For each task, respect its `_Agent:_`:

| `_Agent:_` | How it's executed |
|---|---|
| `autonomous subagent` | Delegate with Task/Agent, using the model from its `_Model:_` |
| `supervised main agent` | **You do it yourself.** Don't delegate it — this is where regression risk requires someone with full context to watch the change |
| `human decision` | Stop and ask me. Never execute it on your own |

Delegating to subagents:
- One subagent per task. Self-contained prompt: which files to touch, which conventions to follow, the task's text, and its traceability (`_Requirements:_`/`_Findings:_`) so it knows what to validate against.
- Independent tasks in the same phase can run in parallel. **If two tasks touch the same file, run them in series** — two agents editing the same file will step on each other.
- The subagent returns: what changed (`file:line`), what it verified, and what it couldn't do. It doesn't return the full code.

Run the project's tests after every task that changes logic. If the project has no tests, verify the minimum runnable thing (that it compiles, that the module imports, that the endpoint responds).

If a task fails twice on the model from its assigned `_Model:_`, it escalates one tier up and gets a fresh 2-failure budget before invoking the Step 2 advisor — full rule, tier order, and run log recording in `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`, "## Model escalation".

Mark `[x]` in the task file **only** once the task is done and verified. Run optional (`*`) tasks if the rest of the phase came out green; if you skip them, say so.

## Step 2 — Opus advisor, when you get stuck

Invoke the **`andamio-advisor`** subagent (`${CLAUDE_PLUGIN_ROOT}/agents/andamio-advisor.md`). It's read-only by tool configuration, not just by prompt: it advises, it doesn't implement. You apply its recommendation.

Escalate when any of these happen — not before, not "just in case":

- The same test or error keeps failing **after the model-escalation ladder is exhausted** — 2 failures at the escalated tier, or 2 failures with the task already at the `opus` ceiling to begin with. See Step 1's model-escalation rule.
- The task requires a decision the spec didn't resolve (and it's not an obvious `human decision`, which would be for me).
- The change is expanding to more modules than the task described. That's a sign the plan assumed something false.
- Security, concurrency, or data-migration surface shows up in a task that wasn't marked `opus`.
- Two ways to implement the task with trade-offs you can't resolve.

The agent defines what it expects in the prompt and how it responds — don't redraft it, give it what it asks for: the concrete symptom, what you already tried and why it failed, the minimal relevant code, and the specific question.

If the advisor doesn't resolve it either, or its recommendation changes the phase's scope: stop and tell me. Don't improvise a redesign.

## Step 3 — Review with an independent agent

When the phase is done, invoke the **`andamio-reviewer`** subagent (`${CLAUDE_PLUGIN_ROOT}/agents/andamio-reviewer.md`). This runs once per phase even under `all` — never once for the whole file — so findings and the commit message below stay scoped to the same unit of work. **Don't do it yourself** — reviewing your own work shares the same blind spots. The agent defines its own criteria, verification order, and output format — don't redraft it in the prompt, give it what it asks for:
- The diff of what changed (`git diff`, or the file list if the diff is huge).
- The phase's tasks with their traceability.
- The path to the source spec or report.
- The output of the tests you already ran (the agent has no Bash, it doesn't run them itself).

### What you do with the findings

- 🔴 and 🟠 → **an opus subagent fixes them**, never you or the model that wrote the bug — the "whoever implements doesn't review" invariant also applies to the fix, not just to detection. One single review round after the fix, not an infinite loop. If the second review is still at 🔴, stop and tell me.
- 🟡 and 🟢 → add them to the task file as a new phase at the end: `### Phase N: Review corrections — <YYYY-MM-DD>`, with tasks in `[ ]`, their `_Agent:_` and `_Model:_`, and `_Review: <reviewed phase>_` traceability. Don't fix them now — they weren't in the plan.

## Hard rules

- **Don't touch the source.** The spec and the audit report are read-only here. If the implementation revealed the spec is wrong, say so in chat and suggest running `grilling` again — don't edit it.
- **Don't go out of scope.** If you're running "Phase 4", don't get ahead on Phase 5 tasks "since you're already there."
- **Don't commit** unless asked to. Report what changed and let me decide.
- **Don't mark `[x]` anything you didn't verify.** A false checkbox is worse than a pending task.
- If a task turns out to be impossible, don't mark it: leave it in `[ ]`, note it in chat, and record it in the run log with the `failed — <why>` verdict.
- If a task was already done, don't mark it either: leave it in `[ ]`, note it in chat, and record it in the run log with the `already done — <why it wasn't needed>` verdict.

## Final report

When done, in chat:

1. Phases and tasks completed, task file path.
2. Files touched.
3. Test status.
4. If you escalated to the advisor: why, and what it decided.
5. Review verdict: findings fixed, and the ones that became new tasks.
6. What's left pending and why.
7. If no explicit phase or `all` was passed: which phase has pending `[ ]` tasks next, so you know what to invoke after this.
8. **The commit message** (see below). Deliver it ready to copy — don't commit.

## Commit message

One commit per phase: the phase is the coherent unit of work. If you ran several phases, deliver one message per phase, in order.

[Conventional Commits](https://www.conventionalcommits.org/) format, in a code block ready to copy:

```
<type>(<scope>): <subject in imperative mood, lowercase, no trailing period>

<Body: what changed and why. One line per completed task.
The diff already shows the "what" — this is the "why".>

Refs: <source spec or report> · <Requirements N.M | Findings A-NN>
```

Types, based on what the phase did:

| Type | When |
|---|---|
| `feat` | New user-visible functionality |
| `fix` | Fixes a bug or a 🔴/🟠 audit finding |
| `refactor` | Restructures without changing behavior |
| `perf` | Performance improvement |
| `test` | Tests only (the `*` tasks, if in a separate commit) |
| `docs` | Documentation only |
| `chore` | Dependencies, configuration, tooling |

Rules:

- **The work determines the type, not the command.** An `/audit` phase that fixes security is `fix`, not `chore`; one that only splits long functions is `refactor`.
- `scope` = the module or area touched (`webhook`, `auth`, `handlers`). If the phase touches several with no clear center, omit it.
- Subject ≤ 72 characters, imperative ("add", not "added" or "adding"), in the project's language.
- If the phase introduces a breaking change: `!` after the scope and a `BREAKING CHANGE: <what breaks and what to do>` footer.
- The `Refs:` footer is mandatory — it's the harness's traceability reaching the git history. Without it the commit loses its origin.
- If a phase mixes different kinds of work (unrelated feature + refactor), say so and propose separate commits with which files go in each.

Example:

```
feat(webhook): add exponential backoff retries to Gupshup

Failed webhooks were lost with no record. Now they retry up to 5 times
with backoff, and exhausted ones land in the dead-letter queue.

- webhook_retries table indexed by status and next_attempt_at
- Retry worker with 2^n backoff, 15-min ceiling
- Idempotency by message_id to avoid duplicating on retry

Refs: docs/specs/gupshup-webhook-retries-spec.md · Requirements 1.1-1.4, 2.1
```
