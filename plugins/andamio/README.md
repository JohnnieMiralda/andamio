# Andamio — spec-driven pipeline

A pipeline where each tool leaves a markdown artifact for the next one to consume. Each step does **one** thing: interview, document, plan, audit, execute. None does two.

## Flow

```
/andamio:grilling <topic>          interview, 1 question at a time. Writes nothing.
      │
      ▼
/andamio:spec                    ← documents the interview
      │
      ▼
docs/specs/<slug>-spec.md         ← numbered requirements (1.1, 2.3...)
      │
      ▼
/andamio:plan docs/specs/<slug>-spec.md
      │
      ▼
docs/tasks/<slug>-tasks.md        ← executable list, one file per spec


/andamio:audit [path]
      │
      ├──► docs/audit/AUDIT-<date>.md          ← findings with IDs (A-01...)
      │
      └──► docs/tasks/audit-<date>-tasks.md    ← executable list, one per run


/andamio:build docs/tasks/<file>-tasks.md ["Phase 4" | all]   ← executes the work
      │
      ▼
   code
```

## Execution

`/build` is the only command that writes code. It runs on **sonnet 5** and acts as orchestrator:

```
/andamio:build <task file> ["Phase N" | all]  ← by default, the next phase with [ ]
      │
      ├─ for each task, per its _Agent:_
      │     autonomous subagent       → delegates, with the task's _Model:_
      │     supervised main agent     → does it itself
      │     human decision            → stops and asks you
      │
      ├─ stuck? → agents/andamio-advisor.md (read-only, advises but doesn't implement)
      │
      ├─ when the phase closes → agents/andamio-reviewer.md (read-only, independent)
      │       🔴🟠 → an opus subagent fixes them (one single re-review round)
      │       🟡🟢 → adds them as a new phase to the task file
      │
      └─ and delivers the commit message ready to copy
```

One commit per phase, in Conventional Commits (`feat(scope): subject`), with a `Refs:` footer to the spec and the requirements it implements — so the harness's traceability reaches the git history. `/build` **doesn't commit**: it gives you the message and you decide.

The advisor isn't brought in "just in case": it escalates on concrete triggers — the same error fails after 2 attempts, the change expands to more modules than the task described, security or concurrency surface shows up where it wasn't marked, or there's a trade-off the spec didn't resolve.

Two sources (`specs/`, `audit/`), one destination (`tasks/`). The task file's name **mirrors** its source's, so the pair is found without opening anything.

Everything pending in the repo, with no index file:

```bash
grep -rn "^- \[ \]" docs/tasks/
```

## Traceability

Every task points to its origin: `/plan`'s point to `_Requirements: N.M_` from the spec, `/audit`'s to `_Findings: A-NN_` from the report. No origin, no task.

Every task also states **who** executes it (`_Agent:_`) and **with which model** (`_Model:_` — the cheapest one that completes it reliably).

The rules for layout, format, regeneration, agent, and model live in one single place: **[`harness/CONVENCIONES.md`](harness/CONVENCIONES.md)**. `/plan` and `/audit` read it via `${CLAUDE_PLUGIN_ROOT}`. If a rule changes, it changes there.

## Installation

Install it as a plugin, once, and it's there in all your projects:

```
/plugin marketplace add JohnnieMiralda/andamio
/plugin install andamio@miralda
```

**personal** scope in the install dialog: it stays in all your projects, without copying anything to any project's `.claude/`. **local** scope is a legitimate pattern while you're trying out a change in one specific repo — it installs only there.

Update, across all projects at once:

```
/plugin marketplace update
```

Publishing and versioning details in the [marketplace README](../../README.md).

### Project-specific conventions

Commands read `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md` — the copy that ships with the plugin. If a project needs different rules (a different `docs/` layout, different model criteria), put a `.claude/harness/CONVENCIONES.md` in that repo and **it wins**. Note at the top of the file why it differs, or in six months you won't know whether it's intentional or just stale.

## Typical usage

```bash
# 1. New feature: design interview
/andamio:grilling retry system for the Gupshup webhook
# ... you answer questions one at a time ...
# > "done, let's close it"

# 2. Document the interview
/andamio:spec
# → docs/specs/gupshup-webhook-retries-spec.md

# 3. Turn the spec into tasks
/andamio:plan docs/specs/gupshup-webhook-retries-spec.md
# → docs/tasks/gupshup-webhook-retries-tasks.md

# 4. Audit existing code (independent of the spec)
/andamio:audit src/handlers
# → docs/audit/AUDIT-2026-07-29.md
# → docs/tasks/audit-2026-07-29-tasks.md

# 5. Execute — with no phase, runs the next pending one, one per run (default)
/andamio:build docs/tasks/gupshup-webhook-retries-tasks.md
# → orchestrates subagents, runs tests, opus review, marks [x], and tells you which phase is next

# ...or one specific phase
/andamio:build docs/tasks/gupshup-webhook-retries-tasks.md "Phase 2"

# ...or the whole feature in one go, in a single context
/andamio:build docs/tasks/gupshup-webhook-retries-tasks.md all
```

Work on a branch. `/build` checks `git status` before starting: if you're on main it stops and tells you; if there are uncommitted changes it asks you once whether they're from an earlier phase of this same task file.

## System rules

- **One task file per source.** Never a file shared between features: it grows without a ceiling, costs tokens on every run, and collides across branches.
- **Regenerating doesn't erase progress.** If the spec changes and you run `/plan` again, it preserves the `[x]` of tasks that didn't change, adds new ones in `[ ]`, and moves ones that were already done and no longer apply to `## Obsolete`. It reports the delta in chat.
- **Checkboxes are only marked when executing**, never when generating.
- **Don't renumber requirements in an existing spec** — it breaks the task file's traceability. Add at the end (1.4, 1.5).
- **Creation order: source first, tasks after.** If it fails partway through, what's left is the expensive-to-reproduce part.
- Generation commands (`/spec`, `/plan`, `/audit`) have restricted `allowed-tools` in their main loop: it can't modify your code, only read and write the harness's markdown. That's a real permission. The subagents `/audit` spawns to explore are read-only by prompt instruction, not by that same restriction — they don't inherit `allowed-tools`. `/build` is the only main loop that writes code.
- **Whoever implements doesn't review.** The review and the advisor run as their own agents (`agents/andamio-reviewer.md`, `agents/andamio-advisor.md`), on a different model than the one that wrote the code, with no `Write`/`Edit`/`Bash` in their tool configuration — read-only by permission, not just by prompt. Sonnet reviewing sonnet's own work shares the same blind spots.
- **`/build` doesn't touch the source.** If the implementation reveals the spec is wrong, it tells you and suggests going back to `grilling` — it doesn't edit it on its own.
- `grilling` writes no files. If it asks you to generate a spec, that's a bug in the skill.
- If you're coming from the version with a single `docs/TASKS.md`: the commands no longer read or write it. Move it or delete it yourself — blindly migrating a file with checked-off checkboxes isn't a command's job.
