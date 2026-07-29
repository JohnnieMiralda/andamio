---
description: Genera el plan de implementación de un spec en docs/tasks/
argument-hint: [ruta-al-spec.md]
allowed-tools: Read, Grep, Glob, Write, Edit
model: claude-sonnet-5
disable-model-invocation: true
---

# Plan de Implementación desde Spec

**Spec a procesar:** $ARGUMENTS
(Si no se especificó ruta, usa el archivo más reciente en `docs/specs/`.)

## Tu tarea

1. **Lee las convenciones del harness:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. Definen el layout, el formato del task file, las reglas de regeneración y la asignación de `_Agente:_` y `_Modelo:_`. Es obligatorio.
   Si el proyecto tiene su propio `.claude/harness/CONVENCIONES.md`, **esa copia gana** — es una sobreescritura deliberada para este repo.
2. **Lee el spec completo.** Si su sección "Open questions" tiene puntos que bloquean la implementación, detente y pregúntame cómo resolverlos antes de generar el plan.
3. **Explora el codebase lo mínimo necesario** para que las tareas sean concretas: qué archivos/módulos toca cada tarea y qué convenciones existen (estructura de carpetas, patrones de handlers, framework de tests). No es una auditoría — solo lo necesario para aterrizar las tareas.
4. **Escribe `docs/tasks/<slug>-tasks.md`**, con el mismo slug del spec. Si el archivo ya existe, aplica las reglas de regeneración de `CONVENCIONES.md` — **preservas los `[x]` existentes**, no reescribes de cero.

## Reglas específicas de /plan

- Trazabilidad vía `_Requirements: N.M_`. **TODO requirement del spec debe estar cubierto por al menos una tarea.** Si alguno no es implementable, dilo explícitamente en el chat.
- Ordena las fases por dependencia: schema/datos primero, lógica core después, robustez (idempotencia, errores, retries) al final.
- NO modifiques código en esta sesión. Solo lectura + escritura en `docs/tasks/`.
- Al terminar, resume en el chat: la ruta del archivo, cuántas tareas y fases, qué requirements quedaron sin cubrir (si alguno), y —si fue regeneración— qué se agregó, cambió o quedó obsoleto.
