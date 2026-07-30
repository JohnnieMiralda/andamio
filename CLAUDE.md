# Context — Plugin marketplace

This repo **is not an application**: it's a Claude Code plugin marketplace. The "code" is markdown files that Claude executes as instructions.

It's the **origin**. Projects don't have copies — they consume the installed plugin, so an improvement here + `/plugin marketplace update` reaches everyone. Never edited in the reverse direction.

## Structure

```
.claude-plugin/marketplace.json          marketplace catalog
plugins/andamio/
├── .claude-plugin/plugin.json           manifest (name, author — version omitted while iterating)
├── skills/grilling/SKILL.md             design interview (writes no files)
├── commands/spec.md                     interview → docs/specs/<slug>-spec.md
├── commands/plan.md                     spec → docs/tasks/<slug>-tasks.md
├── commands/audit.md                    code → docs/audit/AUDIT-<date>.md
│                                                + docs/tasks/audit-<date>-tasks.md
├── commands/build.md                    task file → code (subagents + review)
└── harness/CONVENCIONES.md              single source: layout, format, agent, model
.claude/settings.json                    this repo's graphify hooks (not part of the plugin)
```

Flow, installation, and publishing in [README.md](README.md).

## Rules for this repo as a plugin

- **Support files are referenced with `${CLAUDE_PLUGIN_ROOT}`**, not relative paths or `.claude/`. The placeholder is substituted inside the content of skills and commands. An installed plugin lives in a cache directory — a path relative to the project won't find it.
- **Nothing outside the plugin's directory.** Installing only copies `plugins/<name>/`; a `../something-shared` doesn't travel.
- **Bumping `version` in `plugin.json` is what triggers the update** for anyone who already has it installed. A change with no bump reaches no one. While iterating locally the field is left out — see "Testing before publishing" below.
- `.claude/settings.json` is configuration for **this** repo, not for the plugin. It isn't distributed.

## Invariants when editing

1. **One step, one responsibility.** `grilling` interviews and nothing else; `/spec` documents; `/plan` plans; `/audit` audits; `/build` executes. If an edit makes a step do two things, it's the wrong edit.
2. **Zero rule duplication.** Layout, task file format, regeneration, and the `_Agent:_`/`_Model:_` criteria live **only** in `harness/CONVENCIONES.md`. The commands reference it, they don't copy it. The README doesn't repeat it either.
3. **One task file per source**, named to mirror its source. Never a file shared between features.
4. **Regenerating doesn't destroy finished work.** Every command that rewrites an existing task file preserves the `[x]` and reports the delta. Checkboxes are only marked when executing, never when generating.
5. **Every generated task is traceable** to a `_Requirements: N.M_` from a spec, a `_Findings: A-NN_` from an audit, or a `_Review: <phase>_` from a review.
6. **Restricted `allowed-tools`.** No **generation** command (`/spec`, `/plan`, `/audit`) can touch the target project's code. `/build` is the only exception — it's the execution command — and that's why it carries its own hard rules: it doesn't touch the source, doesn't go outside the phase's scope, doesn't commit without permission, doesn't mark `[x]` without verifying. If you add a generation command, restrict it the same way.
7. **Whoever implements doesn't review.** `/build`'s review runs in a separate subagent, on a different model (opus) than the one that wrote the code (sonnet). Correlated blind spots are exactly what the review has to break.
8. **Language: three surfaces, not one.** Documentation (both READMEs, the marketplace catalog) and prompts (`commands/`, `agents/`, `CONVENCIONES.md`, `SKILL.md`) go in English — they decide whether someone installs and are what whoever contributes to the harness reads. The artifacts the harness produces (spec, task file, commit message) go in the target project's language — English by default if its `CLAUDE.md` doesn't specify one. This repo declares its own: the artifacts the harness produces about this codebase (specs, task files, commits) go in Spanish, as the existing ones under `docs/` already reflect. Details in `harness/CONVENCIONES.md`.

## When adding a new command

Minimum frontmatter: `description`, `argument-hint`, `allowed-tools` (minimum necessary), `model`, `disable-model-invocation: true` — harness commands are invoked by hand, not by the model's own decision. If it writes to `docs/tasks/`, it should read `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md` in step 1.

## Testing before publishing

Point the marketplace at the local path, not the remote repo:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/andamio
/plugin install andamio@miralda
```

The real local development cycle — why the installed plugin isn't a mirror of the working tree, and how to force a re-sync — lives in [README.md, "Testing without publishing" section](README.md#testing-without-publishing). Don't duplicate it here.
