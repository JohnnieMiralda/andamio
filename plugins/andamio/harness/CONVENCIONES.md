# Harness conventions

Single source of truth for `/plan`, `/audit`, and `/build`. If a rule changes, it changes here — not in the commands.

## Layout

```
docs/
├── specs/   <slug>-spec.md              ← /spec    (source)
├── audit/   AUDIT-<YYYY-MM-DD>.md       ← /audit   (source)
└── tasks/   <slug>-tasks.md             ← /plan    (derived)
          audit-<YYYY-MM-DD>-tasks.md  ← /audit   (derived)
          <slug>-run-<YYYY-MM-DD>.md   ← /build   (derived)
```

Two sources, one destination. **The task file's name mirrors its source's**, so the pair is found without opening anything.

**Creation order:** the source artifact first, then the derived tasks. If the process fails partway through, what's left is the expensive-to-reproduce part. `/audit` writes its report and *then* the task file; `/plan` only reads the spec and writes tasks.

Create the folders if they don't exist. One task file per source — never a file shared between features.

## Task file format

```markdown
# Tasks: <Name> — <YYYY-MM-DD>

> Source: `<path to the spec or audit report>`
> Generated: <YYYY-MM-DD> · Updated: <YYYY-MM-DD>

## Overview

<Brief description of the plan and its phases. The architectural narrative
goes here, if there is one — not in a separate document.>

- **Phase 1: <name>** - <description>
- **Phase 2: <name>** - <description>

## Tasks

### Phase 1: <name>

- [ ]   1. <Task title>
    - <Concrete step 1, mentioning real files/modules from the codebase>
    - <Concrete step 2>
    - _Requirements: 1.1, 2.3_          ← /plan
    - _Findings: A-01, A-03 (file:line)_   ← /audit
    - _Agent: <autonomous subagent | supervised main agent | human decision>_
    - _Model: <haiku | sonnet | opus>_

- [ ]   2. <Task title>
    - [ ] 2.1 <Subtask>
        - <Implementation detail>
        - _Requirements: ..._
        - _Agent: ..._
        - _Model: ..._
    - [ ]* 2.2 <Optional subtask, e.g. tests>
        - **Property N: <property to validate>**
        - **Validates: <Requirements N.M | A-NN>**
```

Hierarchical numbering for subtasks (2.1, 2.2). Tasks with `*` are optional (typically property tests). All are born in `[ ]` — checkboxes are only marked when **executing**, never when generating.

**Traceability**: every task points to its origin. `/plan` uses `_Requirements: N.M_` from the spec; `/audit` uses `_Findings: A-NN_` from the report; `/build` uses `_Review: <phase>_` for corrections coming out of a review. No origin, no task.

## Regeneration: don't erase progress

If the task file already exists, **don't overwrite it from scratch**. The source changed, your finished work didn't:

1. Read the existing file before writing.
2. **Preserve the `[x]` state** of every task whose title and traceability didn't change.
3. New tasks are born in `[ ]`.
4. A task that no longer applies: if it was in `[ ]`, it disappears. If it was in `[x]`, move it to an `## Obsolete` section at the end with a one-line reason — the record of work done doesn't get silently erased.
5. Update `Updated:` in the header.
6. Report in chat: how many tasks were added, how many changed, how many became obsolete.

Seeing "everything pending" in the repo doesn't need an index file:

```bash
grep -rn "^- \[ \]" docs/tasks/
```

## Run log

`/build` appends one line per executed task to `docs/tasks/<slug>-run-<YYYY-MM-DD>.md` — if `/build` dies partway through a phase, this is what lets you resume without re-deriving state from `git diff`. Create it on the run's first task:

```markdown
# Run: <slug> — <YYYY-MM-DD>

| Task | Agent | Model | Files | Tests | Verdict |
|---|---|---|---|---|---|
```

One row per executed task — **whatever its outcome**, not just the ones that ended up `[x]`: a task that failed or that you skipped is exactly what needs to be readable afterward. In this order: task (number + short title), agent, model, files touched, test status, verdict.

`Model` is the model it actually ran with; if the task escalated after failing, the escalation is visible in that same single row (`sonnet→opus (escalated, 2 failures)`), never in an extra row — so the next `/plan` regeneration sees which assignments fell short. `Verdict` takes one of three values: `verified`, `failed — <why or where to pick up>`, `skipped — <why>`. Example:

```markdown
| 2.1 Validate webhook input | autonomous subagent | haiku | `src/webhook/validate.ts` | ok (8/8) | verified |
| 2.2 Sign the outgoing payload | autonomous subagent | sonnet→opus (escalated, 2 failures) | `src/webhook/sign.ts` | ok (5/5) | verified |
| 3. Migrate retries table | supervised main agent | opus | — | fails (2/7) | failed — see advisor |
```

Known ceiling: the row gets appended when the task **finishes**, so a run that dies partway through a single task leaves no trace of that task. Accepted, not solved.

## Artifact language

Three surfaces, different criteria — not "one language for the repo":

- **Documentation** (both READMEs, the marketplace catalog) → English. It decides whether someone installs; a README that whoever's browsing the repo on GitHub can't read hides most of the point.
- **Prompts** (`commands/`, `agents/`, this file, `SKILL.md`) → English. Whoever contributes to the harness reads them first — they're the code.
- **Artifacts the harness produces** (spec, task file, commit message) → the target project's language. If the project's `CLAUDE.md` doesn't specify one, English by default.

Output language is parametrized, not forked: every command that writes to `docs/` reads the target project's `CLAUDE.md` — it already does, the internal standard requires it — and writes the artifact in that language. Zero new mechanism.

## `_Agent:_` assignment

- **autonomous subagent** — mechanical and bounded, success criteria verifiable with no debate.
- **supervised main agent** — architectural, or with regression risk that needs a human-context reviewer watching it happen.
- **human decision** — requires business judgment, or touches something the spec/audit left unresolved.

## `_Model:_` assignment

Always the **cheapest** model that can complete the task reliably.

| Model | When | Examples |
|---|---|---|
| `haiku` | Mechanical, unambiguous | renaming, moving files, extracting constants and magic numbers, obvious types, formatting, boilerplate from an existing pattern, updating imports, dead code, tests from a clear template |
| `sonnet` | Standard implementation | well-specified business logic, refactors within a module, splitting long functions, integration with documented APIs, adding error handling or validation following a pattern, tests that require designing cases. **Most of the time.** |
| `opus` | High cost of error | security (auth, injection, secrets), cross-cutting changes across several modules, schema/data migrations, race conditions and concurrency, design trade-offs the spec left open |

Practical rules:

- If torn between two, pick the cheaper one. It's easier to escalate a failed task than to recover burned tokens.
- If a task mixes trivial and complex work, split it into subtasks with different models instead of assigning the expensive model to all of it.

When executing, switch with `/model haiku` (or whichever the task indicates) before working on it — or include it in the subagent's prompt.
