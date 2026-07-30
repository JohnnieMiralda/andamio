---
description: Turns a grilling interview into a spec under docs/specs/
argument-hint: [optional-feature-name]
allowed-tools: Read, Grep, Glob, Write
model: claude-sonnet-5
disable-model-invocation: true
---

# Spec from interview

**Feature:** $ARGUMENTS
(If not specified, derive it from the conversation.)

Write the spec for the feature just discussed to `docs/specs/<slug>-spec.md` (slug in kebab-case, create the folder if it doesn't exist).

## Source

The **current conversation** is the only source. This command documents an interview that already happened — typically via the `grilling` skill.

If there's no design interview in this conversation to write from, say so and stop. Suggest running `grilling` first. Don't invent a spec from scratch.

## Required structure

```markdown
# Spec: <Feature or product name>

> Generated from grilling session — <YYYY-MM-DD>
> Status: Draft

## Overview

<What it is, for whom, and what problem it solves. 3-6 lines.>

## Design decisions

<Each decision made in the interview with brief justification.
Include discarded alternatives and why.>

## Requirements

<Hierarchically numbered requirements. This numbering is the source of truth
that /plan will reference as _Requirements: N.M_.>

### 1. <Functional area>
- 1.1 <Atomic, verifiable requirement>
- 1.2 <Atomic, verifiable requirement>

### 2. <Functional area>
- 2.1 ...

## Out of scope

<What explicitly will NOT be done in this iteration.>

## Open questions

<Unresolved questions and who/what unblocks them.>
```

## Rules

- Every requirement is atomic and verifiable: it can be marked done or not done with no ambiguity.
- **Don't invent decisions that weren't discussed.** Anything left open goes in Open questions, not in Design decisions. A spec with honest gaps is useful; one with gaps filled in on your own judgment is a trap.
- If the spec already exists, don't silently overwrite it: show me the conceptual diff (which requirements change, get added, or go away) and wait for confirmation. Renumbering a requirement breaks the task file's traceability — if you need to add one, add it at the end (1.4, 1.5) instead of renumbering.
- You only write to `docs/specs/`. Don't touch code or `docs/tasks/`.
- When done, confirm the path and suggest: `/plan docs/specs/<slug>-spec.md`
