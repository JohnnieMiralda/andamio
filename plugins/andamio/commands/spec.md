---
description: Convierte la entrevista de grilling en un spec en docs/specs/
argument-hint: [nombre-de-la-feature-opcional]
allowed-tools: Read, Grep, Glob, Write
model: claude-sonnet-5
disable-model-invocation: true
---

# Spec desde entrevista

**Feature:** $ARGUMENTS
(Si no se especificó, derívalo de la conversación.)

Escribe el spec de la feature que se acaba de discutir en `docs/specs/<slug>-spec.md` (slug en kebab-case, crea la carpeta si no existe).

## Fuente

La **conversación actual** es la única fuente. Este comando documenta una entrevista que ya ocurrió — normalmente vía la skill `grilling`.

Si en esta conversación no hay una entrevista de diseño de la cual escribir, dilo y detente. Sugiere correr `grilling` primero. No inventes un spec desde cero.

## Estructura obligatoria

```markdown
# Spec: <Nombre de la feature o producto>

> Generado desde sesión de grilling — <YYYY-MM-DD>
> Estado: Draft

## Overview

<Qué es, para quién, y qué problema resuelve. 3-6 líneas.>

## Decisiones de diseño

<Cada decisión tomada en la entrevista con su justificación breve.
Incluye las alternativas descartadas y por qué.>

## Requirements

<Requerimientos numerados jerárquicamente. Esta numeración es la fuente de verdad
que /plan referenciará como _Requirements: N.M_.>

### 1. <Área funcional>
- 1.1 <Requerimiento atómico y verificable>
- 1.2 <Requerimiento atómico y verificable>

### 2. <Área funcional>
- 2.1 ...

## Out of scope

<Lo que explícitamente NO se va a hacer en esta iteración.>

## Open questions

<Preguntas sin resolver y quién/qué las desbloquea.>
```

## Reglas

- Cada requirement es atómico y verificable: se puede marcar hecho o no hecho sin ambigüedad.
- **NO inventes decisiones que no se discutieron.** Si algo quedó abierto va en Open questions, no en Decisiones de diseño. Un spec con huecos honestos es útil; uno con huecos rellenados a criterio propio es una trampa.
- Si el spec ya existe, no lo sobreescribas en silencio: muéstrame el diff conceptual (qué requirements cambian, se agregan o se van) y espera confirmación. Renumerar un requirement rompe la trazabilidad del task file — si tienes que agregar, agrega al final (1.4, 1.5) en vez de renumerar.
- Solo escribes en `docs/specs/`. No toques código ni `docs/tasks/`.
- Al terminar, confirma la ruta y sugiere: `/plan docs/specs/<slug>-spec.md`
