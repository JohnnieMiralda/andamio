---
description: Auditoría de código - legibilidad, malas prácticas y seguridad
argument-hint: [ruta-o-glob-opcional]
allowed-tools: Read, Grep, Glob, Task, Write, Edit
model: claude-sonnet-5
disable-model-invocation: true
---

# Auditoría de Código

Identifica todo lo que hace el código engorroso, difícil de entender, frágil o inseguro, y produce un plan de acción ejecutable.

**Alcance solicitado:** $ARGUMENTS
(Si no se especificó alcance, audita todo el repositorio.)

**Lee primero las convenciones del harness:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. Definen el layout, el formato del task file y la asignación de `_Agente:_` y `_Modelo:_`. Es obligatorio.
Si el proyecto tiene su propio `.claude/harness/CONVENCIONES.md`, **esa copia gana** — es una sobreescritura deliberada para este repo.

## Dimensiones

1. **Legibilidad y mantenibilidad**: funciones excesivamente largas, nombres ambiguos o inconsistentes, anidación profunda, código duplicado, acoplamiento fuerte, responsabilidades mezcladas, falta de separación de capas, dead code, comentarios desactualizados o ausentes donde son críticos, y complejidad accidental (abstracciones innecesarias, sobre-ingeniería).

2. **Malas prácticas**: anti-patrones del stack en uso (revisa primero qué frameworks/lenguajes hay y aplica sus convenciones). Ejemplos: manejo de errores inexistente o catch silencioso, promesas sin await, mutación de estado compartido, magic numbers/strings, configuración hardcodeada, falta de tipado o `any`, lógica de negocio en handlers, ausencia de validación de inputs, dependencias innecesarias o desactualizadas.

3. **Seguridad**: secretos/credenciales/API keys hardcodeadas o commiteadas, inyección (SQL, command, template), falta de validación de inputs externos (webhooks, query params, payloads), auth/authz débil o ausente en endpoints, exposición de datos sensibles en logs o errores, CORS permisivo, dependencias con CVEs conocidos, manejo inseguro de tokens/sesiones, falta de rate limiting en endpoints públicos.

## Estrategia de ejecución

Usa subagentes (Task) para minimizar consumo del contexto principal:

- **Primero explora tú la estructura** (árbol de directorios, package.json/requirements, configs, entry points) para entender la arquitectura. No lances agentes a ciegas.
- **Subagentes en paralelo solo para lectura pesada**, divididos por área (handlers/API, lógica de negocio, infra/config/seguridad). Cada uno devuelve resumen compacto: hallazgo, `archivo:línea`, severidad, evidencia mínima — no bloques de código completos.
- **No uses agentes para pocos archivos o archivos pequeños** — léelos directamente.
- **Rutas/globs explícitos y sin solapamiento** por agente.
- Tú consolidas, deduplicas, priorizas y escribes los entregables. Los subagentes NO escriben archivos.

## Severidad

- 🔴 **Crítico**: riesgo de seguridad explotable, pérdida de datos, o bug latente grave.
- 🟠 **Alto**: mala práctica con impacto real en confiabilidad o mantenibilidad.
- 🟡 **Medio**: deuda técnica que ralentiza el desarrollo.
- 🟢 **Bajo**: mejora cosmética o de estilo.

## Entregables

Escríbelos **en este orden**: primero el reporte, después las tareas. El reporte es lo caro de reproducir.

### 1. `docs/audit/AUDIT-<YYYY-MM-DD>.md` — reporte

Crea la carpeta si no existe. Estructura:

- **Resumen ejecutivo**: estado general en 3-5 líneas, conteo por severidad.
- **Contexto del proyecto**: stack detectado, arquitectura observada, tamaño aproximado.
- **Hallazgos por dimensión**: cada uno con ID (`A-01`, `A-02`, ...), severidad, ubicación exacta (`archivo:línea`), por qué es un problema, y recomendación. Fragmentos de código solo si son indispensables y breves.
- **Lo que está bien**: brevemente, para no romperlo al refactorizar.
- **Priorización recomendada**: orden de ataque y justificación.

### 2. `docs/tasks/audit-<YYYY-MM-DD>-tasks.md` — tareas

Un archivo **por corrida** de auditoría; el nombre espeja el del reporte. Formato en `CONVENCIONES.md`, con `# Tasks: Audit <alcance> — <YYYY-MM-DD>` y `> Fuente: docs/audit/AUDIT-<YYYY-MM-DD>.md`.

Fases por severidad: **Fase 1** = críticos y seguridad, **Fase 2** = malas prácticas de alto impacto, **Fase 3** = legibilidad y deuda técnica.

Trazabilidad vía `_Findings: A-NN (archivo:línea)_`. Todo hallazgo 🔴 y 🟠 debe tener tarea; los 🟡/🟢 pueden agruparse en tareas de limpieza.

Si re-auditas un área ya auditada, esto genera un archivo nuevo — no toques ni fusiones los anteriores. Antes de escribir, revisa si en `docs/tasks/` hay tareas pendientes de una auditoría previa del mismo alcance y menciónalo en el chat; consolidar o borrar los viejos es decisión mía.

## Reglas finales

- NO modifiques ningún archivo de código. Solo lectura + el reporte + el task file.
- Sé específico: "`processWebhook` en `src/handlers/gupshup.ts:45` tiene 180 líneas y mezcla validación, parsing y persistencia" es útil; "hay funciones largas" no lo es.
- Si el repositorio es muy grande, prioriza: entry points, endpoints expuestos, lógica central, configuración/secretos. Indica en el reporte qué quedó fuera del alcance.
- Al terminar, dame en el chat los 3 hallazgos más críticos y la ruta de ambos entregables.
