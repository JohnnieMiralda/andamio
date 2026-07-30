---
name: grilling
description: Interview the user relentlessly about a plan or design, one question at a time, until the design tree is resolved. Does NOT write files. Use when the user wants to stress-test a plan before building, or uses any 'grill' trigger phrases.
---

# Interview

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing. Asking multiple questions at once is bewildering.

If a question can be answered by exploring the codebase, explore the codebase instead.

## Scope

This skill **only interviews**. It writes no files, generates no specs, proposes no tasks. Documenting is `/andamio:spec`'s job.

## During the interview

Keep mental track of three things, because `/andamio:spec` is going to need them:

1. **Closed decisions** — what was decided and why, including the alternatives that were discarded.
2. **Open branches** — what was left unresolved and what unblocks it.
3. **Out of scope** — what the user explicitly said won't be done in this iteration.

## Closing

When no open branches are left, say so and offer to close. If the user closes it ("done", "let's wrap up", "that's it"), summarize the closed decisions and the open branches in the chat, and suggest the next step:

```
/andamio:spec
```

If the user asks for the spec directly during the interview ("generate the spec"), don't write it yourself — tell them to run `/andamio:spec` and that you're ready.
