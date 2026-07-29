---
name: grilling
description: Interview the user relentlessly about a plan or design, one question at a time, until the design tree is resolved. Does NOT write files. Use when the user wants to stress-test a plan before building, or uses any 'grill' trigger phrases.
---

# Entrevista

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing. Asking multiple questions at once is bewildering.

If a question can be answered by exploring the codebase, explore the codebase instead.

## Alcance

Esta skill **solo entrevista**. No escribe archivos, no genera specs, no propone tareas. Documentar es trabajo de `/spec`.

## Durante la entrevista

Lleva registro mental de tres cosas, porque `/spec` las va a necesitar:

1. **Decisiones cerradas** — qué se decidió y por qué, incluyendo las alternativas descartadas.
2. **Ramas abiertas** — lo que quedó sin resolver y qué lo desbloquea.
3. **Fuera de alcance** — lo que el usuario dijo explícitamente que no se hace en esta iteración.

## Cierre

Cuando ya no queden ramas abiertas, dilo y ofrece cerrar. Si el usuario cierra ("listo", "cerramos", "ya"), resume en el chat las decisiones cerradas y las ramas abiertas, y sugiere el siguiente paso:

```
/spec
```

Si el usuario pide el spec directamente durante la entrevista ("genera el spec"), no lo escribas tú — dile que corra `/spec` y que estás listo.
