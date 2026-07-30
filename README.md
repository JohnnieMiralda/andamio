# Andamio

**Scaffolding, not a crutch: it doesn't write for you, it holds up your discipline.**

A spec-driven pipeline for Claude Code — temporary structure that holds while the building goes up. Nothing gets built without a spec, no task exists without an origin, and traceability reaches all the way into the commit message.

```
/andamio:grilling <topic>   design interview, 1 question at a time. Writes nothing.
      ↓
/andamio:spec            → docs/specs/<slug>-spec.md        requirements 1.1, 2.3...
      ↓
/andamio:plan            → docs/tasks/<slug>-tasks.md       tasks with agent and model
      ↓
/andamio:build           → code + independent review + commit message

/andamio:audit [path]    → docs/audit/AUDIT-<date>.md      findings A-01, A-02...
                         → docs/tasks/audit-<date>-tasks.md
```

Every task knows **where it comes from** (`_Requirements: 1.1_` / `_Findings: A-03_`), **who executes it** (subagent, supervised agent, or you), and **with which model** (the cheapest one that completes it reliably). That footer reaches the commit, so six months from now `git log` tells you which spec each line came from.

## Install

In Claude Code, from any project:

```
/plugin marketplace add JohnnieMiralda/andamio
/plugin install andamio@miralda
```

Choose **personal** scope in the dialog: it becomes available in all your projects without touching any project's `.claude/`. If you just want to try it in one specific repo, choose **local** — it installs only there, without affecting the rest.

> `/plugin` opens an interactive panel. If your session doesn't support it, run it from a terminal with `claude`.

Full pipeline documentation: [plugins/andamio/README.md](plugins/andamio/README.md).

## Update

```
/plugin marketplace update
```

Refreshes **all** your installs — nothing gets copied to any project. Plugins only receive the update when their `plugin.json`'s `version` bumps: you decide when there's a release, not every commit. That's for an already-published plugin — while iterating locally without a pinned `version` it's the other way around, see ["Testing without publishing"](#testing-without-publishing).

## Why it exists

Claude Code writes code fast. The problem isn't speed — it's that without structure you end up with features nobody specified, tasks with no origin, and a `git log` that explains nothing. Andamio applies five brakes:

- **No step does two things.** The interview doesn't write specs; the spec doesn't plan; the plan doesn't execute. Each artifact is reviewable on its own.
- **Generation commands can't touch your code.** The main loop has restricted `allowed-tools`: it reads and writes markdown, nothing else — that's a real permission. The subagents `/audit` spawns to explore are read-only by prompt instruction, not by that permission; they don't inherit `allowed-tools`. `/build` is the only main loop that writes code.
- **Whoever implements doesn't review.** The review and the advisor run as their own agents (`plugins/andamio/agents/`), on a different model than the one that wrote the code, with no `Write`/`Edit`/`Bash` in their configuration — read-only by permission, not just by prompt. Correlated blind spots are exactly what a review has to break.
- **Every `/build` run is a phase, not the whole file.** By default it runs the next phase with pending tasks and tells you which one's next — a genuinely fresh context per phase, not an accumulation of diffs and test output that ends up under-weighting its own prompt's hard rules. Running everything in one go requires asking for it explicitly (`all`).
- **Regenerating doesn't erase progress.** If the spec changes, the plan updates while preserving what you already marked done.

## Known limits

Naming them makes the harness more credible, not less — one that only lists virtues reads as marketing.

- **No pipeline status index.** To see pending tasks, see the command documented in [`harness/CONVENCIONES.md`'s "Regeneration" section](plugins/andamio/harness/CONVENCIONES.md#regeneration-dont-erase-progress). The index doesn't track which specs never made it to `/plan` or which audits never got a `/build` — at this scale over-engineering; the gap stays noted, not solved.
- **The ad hoc subagents `/audit` spawns don't inherit `allowed-tools`.** The reviewer and advisor in `plugins/andamio/agents/` ARE read-only by tool configuration (`tools: Read, Grep, Glob`); the subagents `/audit` spawns to explore code, on the other hand, are read-only only by prompt instruction — they don't inherit the main loop's restriction.
- **An interrupted run doesn't resume on its own.** The run log (`docs/tasks/<slug>-run-<date>.md`) leaves which task failed or got skipped, but resuming is still a manual read — there's no `--resume`, and the task where the run died partway through leaves no row (it only gets appended once it finishes).
- **Installed plugins are cached copies**, not live links to the repo — see ["Testing without publishing"](#testing-without-publishing) for how to sync changes.

## Publishing a change

```bash
# 1. edit the plugin
# 2. re-pin version in plugins/andamio/.claude-plugin/plugin.json (omitted while iterating)
# 3. commit + push
git add -A && git commit -m "feat(andamio): <what changed>" && git push
```

Semver: `patch` for wording, `minor` for a new command or rule, `major` when the `docs/` layout or the artifact format changes — that breaks existing task files.

## Repo structure

```
.claude-plugin/marketplace.json      marketplace catalog (what /plugin marketplace add reads)
plugins/andamio/
├── .claude-plugin/plugin.json       manifest: name, author (version omitted while iterating)
├── commands/*.md                    slash commands
├── agents/*.md                      review + advisor
├── skills/grilling/SKILL.md         skills
└── harness/CONVENCIONES.md          support files, via ${CLAUDE_PLUGIN_ROOT}
CLAUDE.md                            context for editing this repo
```

Components get namespaced with the plugin's name: the four commands and the skill are invoked as `/andamio:spec`, `/andamio:plan`, `/andamio:audit`, `/andamio:build`, `/andamio:grilling`. That avoids collisions with other plugins' commands and skills.

### Testing without publishing

Point the marketplace at the local path instead of the repo:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/andamio
/plugin install andamio@miralda
```

**The installed plugin is a cached copy, not a mirror of the working tree.** Editing the repo doesn't change what runs on its own.

While iterating, **omit `version` in `plugin.json`** — without the field, Claude Code falls back to the commit SHA as the version, so every commit counts as a new one. With a pinned `version` (as it should be when publishing), the plugin is pinned to that version: bumping it does deliver, but a re-sync without bumping it does nothing.

To bring a change (committed or not — the re-sync copies the working tree's content, a separate mechanism from the version identifier above, so there's no need to commit first) to the installed copy:

```bash
claude plugin update andamio@miralda --scope local
```

Then **restart your Claude Code session** — the command itself warns "Restart to apply changes"; an open session keeps serving whatever it loaded at startup.

## License

MIT
