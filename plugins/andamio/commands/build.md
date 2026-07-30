---
description: Ejecuta la siguiente phase pendiente de un task file (o una phase puntual, o todas), orquestando subagentes
argument-hint: [ruta-al-tasks.md] [phase-opcional | todas]
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task, Agent
model: claude-sonnet-5
disable-model-invocation: true
---

# Orquestador de ejecución

**Alcance:** $ARGUMENTS

Interpretación del argumento:
- `<archivo>` solo → **ejecutas la siguiente phase con tareas en `[ ]`**, no el archivo completo. Un contexto por phase evita que las reglas duras de este mismo prompt se sub-ponderen a medida que el contexto crece con cada retorno de subagente, cada diff y cada salida de test.
- `<archivo> "Phase 4"` o `<archivo> 4` → esa phase puntual, exista o no trabajo pendiente antes.
- `<archivo> todas` → corre el archivo completo, phase tras phase, en un solo contexto. Requiere el argumento explícito — ya no es el comportamiento por defecto.
- Sin archivo → usa el más reciente de `docs/tasks/` y **confírmame cuál antes de arrancar**.

**Cómo encuentras "la siguiente phase":** recorre las `### Phase N:` del archivo en orden y toma la primera que tenga al menos una tarea en `- [ ]` (ignora las opcionales `[ ]*` si todo lo demás de esa phase ya está en `[x]`). Si no queda ninguna, dilo — el archivo está completo.

**Tú eres el orquestador.** Corres en sonnet. No levantas otro orquestador — coordinas, delegas, verificas y reportas.

## Paso 0 — Antes de tocar código

1. Lee el task file completo y su fuente (el spec o el reporte de auditoría del `> Fuente:`). Sin la fuente no entiendes la trazabilidad de las tareas.
2. **Lee las convenciones del harness:** `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. Definen el layout, el formato del task file, la asignación de `_Agente:_` y `_Modelo:_`, y el formato de `### Phase N: Correcciones de review`. Es obligatorio.
   Si el proyecto tiene su propio `.claude/harness/CONVENCIONES.md`, **esa copia gana** — es una sobreescritura deliberada para este repo.
3. Verifica el estado del repo: `git status`. Si estás en la rama principal, **dímelo y espera** — no trabajo directo sobre main.
   Si hay cambios sin commitear, muéstrame `git status --short` y **pregúntame una vez** si es trabajo de una phase anterior de este mismo task file. No intentes deducir la procedencia solo — preguntar es más barato y más confiable. Si confirmo que sí, sigue; si no, detente.
4. Lista en el chat qué vas a ejecutar: las tareas de la phase, con su `_Agente:_` y `_Modelo:_`. Si el argumento no especificaba phase, di cuál elegiste (la primera con tareas en `[ ]`) y por qué. Si alguna tarea del alcance es `_Agente: decisión humana_`, **pregúntame por ella ahora**, no a mitad de la ejecución.

## Paso 1 — Ejecutar, tarea por tarea

En el orden del archivo. Las fases están ordenadas por dependencia; no las reordenes ni paralelices tareas de fases distintas.

Por cada tarea, respeta su `_Agente:_`:

| `_Agente:_` | Cómo se ejecuta |
|---|---|
| `subagente autónomo` | Delega con Task/Agent, usando el modelo de su `_Modelo:_` |
| `agente principal supervisado` | **La haces tú.** No la delegues — es donde el riesgo de regresión requiere que alguien con el contexto completo vea el cambio |
| `decisión humana` | Detente y pregúntame. Nunca la ejecutes por tu cuenta |

Delegación a subagentes:
- Un subagente por tarea. Prompt autocontenido: qué archivos tocar, qué convenciones seguir, el texto de la tarea, y su trazabilidad (`_Requirements:_` / `_Findings:_`) para que sepa contra qué se valida.
- Tareas independientes de la misma phase pueden ir en paralelo. **Si dos tareas tocan el mismo archivo, van en serie** — dos agentes editando el mismo archivo se pisan.
- El subagente devuelve: qué cambió (archivo:línea), qué verificó, y qué no pudo hacer. No devuelve el código completo.

Si una tarea falla 2 veces en el modelo de su `_Modelo:_` asignado, reintenta un escalón arriba (`haiku`→`sonnet`, `sonnet`→`opus`; `opus` no escala, ya es el techo — tabla completa en `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`) **antes** de invocar al advisor del Paso 2. Si la tarea escalada tiene éxito, la escalada queda anotada en la **única** fila de esa tarea en la bitácora de corrida — con el modelo final y la escalada visible ahí, no en una fila extra (formato en `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`).

Marca `[x]` en el task file **solo** cuando la tarea está hecha y verificada. Tareas con `*` (opcionales) las ejecutas si el resto de la phase quedó verde; si las saltas, dilo.

Después de cada tarea, sea cual sea su resultado — verificada, fallida (tras agotar la escalada de modelo y, si aplica, el advisor del Paso 2) o salteada —, appendea una línea a la bitácora de corrida (`docs/tasks/<slug>-run-<YYYY-MM-DD>.md`, créala si no existe) con el formato definido en `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`. No dependas de que la tarea haya salido bien: una corrida que muere deja su valor justo en las filas de lo que no funcionó.

Corre los tests del proyecto después de cada tarea que cambie lógica. Si el proyecto no tiene tests, verifica lo mínimo ejecutable (que compile, que el módulo importe, que el endpoint responda).

## Paso 2 — Advisor opus, cuando te trabas

Invoca el subagente **`andamio-advisor`** (`${CLAUDE_PLUGIN_ROOT}/agents/andamio-advisor.md`). Es read-only por configuración de herramientas, no solo por prompt: aconseja, no implementa. Tú aplicas su recomendación.

Escala cuando pase cualquiera de estas — no antes, no "por si acaso":

- El mismo test o error sigue fallando **después de 2 intentos** de arreglo.
- La tarea requiere una decisión que el spec no resolvió (y no es `decisión humana` obvia, que sería para mí).
- El cambio se está expandiendo a más módulos de los que la tarea decía. Eso es señal de que el plan asumió algo falso.
- Aparece superficie de seguridad, concurrencia o migración de datos en una tarea que no estaba marcada `opus`.
- Dos formas de implementar la tarea con trade-offs que no sabes resolver.

El agente define qué espera recibir en el prompt y cómo responde — no se lo redactes de nuevo, dale lo que pide: el síntoma concreto, lo que ya intentaste y por qué falló, el código relevante mínimo, y la pregunta específica.

Si el advisor tampoco resuelve, o su recomendación cambia el alcance de la phase: párate y dime. No improvises un rediseño.

## Paso 3 — Review con agente independiente

Al terminar la phase (o el archivo completo si ejecutaste todo), invoca el subagente **`andamio-reviewer`** (`${CLAUDE_PLUGIN_ROOT}/agents/andamio-reviewer.md`). **No lo hagas tú** — revisar tu propio trabajo comparte los puntos ciegos. El agente define su propio criterio, orden de verificación y formato de salida — no se lo redactes de nuevo en el prompt, dale lo que pide:
- El diff de lo que cambió (`git diff`, o la lista de archivos si el diff es enorme).
- Las tareas de la phase con su trazabilidad.
- La ruta del spec o reporte fuente.
- La salida de los tests que ya corriste (el agente no tiene Bash, no los corre él).

### Qué haces con los hallazgos

- 🔴 y 🟠 → **los corrige un subagente opus**, nunca vos ni el mismo modelo que escribió el bug — el invariante "quien implementa no revisa" también aplica a la corrección, no solo a la detección. Una sola ronda de review después del arreglo, no un ciclo infinito. Si el segundo review sigue en 🔴, párate y dime.
- 🟡 y 🟢 → agrégalos al task file como una phase nueva al final: `### Phase N: Correcciones de review — <YYYY-MM-DD>`, con tareas en `[ ]`, su `_Agente:_` y `_Modelo:_`, y trazabilidad `_Review: <phase revisada>_`. No los arregles ahora: no estaban en el plan.

## Reglas duras

- **No toques la fuente.** El spec y el reporte de auditoría son read-only aquí. Si la implementación reveló que el spec está mal, dilo en el chat y sugiere `grilling` de nuevo — no lo edites.
- **No salgas del alcance.** Si ejecutas "Phase 4", no adelantas tareas de la 5 "porque ya estabas ahí".
- **No commitees** salvo que te lo pida. Reporta qué cambió y déjame decidir.
- **No marques `[x]` lo que no verificaste.** Un checkbox falso es peor que una tarea pendiente.
- Si una tarea resulta imposible o ya estaba hecha, no la marques: déjala en `[ ]`, anótalo en el chat y sigue.

## Reporte final

Al terminar, en el chat:

1. Phases y tareas completadas, ruta del task file.
2. Archivos tocados.
3. Estado de tests.
4. Si escalaste al advisor: por qué y qué decidió.
5. Veredicto del review: hallazgos arreglados, y los que quedaron como tareas nuevas.
6. Qué quedó pendiente y por qué.
7. Si no se pasó phase explícita ni `todas`: qué phase sigue con tareas en `[ ]`, para que sepas qué invocar después.
8. **El mensaje de commit** (ver abajo). Lo entregas listo para copiar — no commiteas.

## Mensaje de commit

Un commit por phase: la phase es la unidad coherente de trabajo. Si ejecutaste varias phases, entrega un mensaje por cada una, en orden.

Formato [Conventional Commits](https://www.conventionalcommits.org/), en un bloque de código para copiar directo:

```
<tipo>(<scope>): <asunto en imperativo, minúscula, sin punto final>

<Cuerpo: qué cambió y por qué. Una línea por tarea completada.
El "qué" ya está en el diff — aquí va el "por qué".>

Refs: <spec o reporte fuente> · <Requirements N.M | Findings A-NN>
```

Tipos, por lo que hizo la phase:

| Tipo | Cuándo |
|---|---|
| `feat` | Funcionalidad nueva visible para el usuario |
| `fix` | Corrige un bug o un hallazgo 🔴/🟠 de auditoría |
| `refactor` | Reestructura sin cambiar comportamiento |
| `perf` | Mejora de rendimiento |
| `test` | Solo tests (las tareas `*` si van en commit aparte) |
| `docs` | Solo documentación |
| `chore` | Dependencias, configuración, tooling |

Reglas:

- **El tipo lo determina el trabajo, no el comando.** Una phase de `/audit` que corrige seguridad es `fix`, no `chore`; una que solo divide funciones largas es `refactor`.
- `scope` = el módulo o área tocada (`webhook`, `auth`, `handlers`). Si la phase toca varios sin un centro claro, omítelo.
- Asunto ≤ 72 caracteres, en imperativo ("agrega", no "agregado" ni "agregando"), español.
- Si la phase introduce un breaking change: `!` después del scope y footer `BREAKING CHANGE: <qué se rompe y qué hacer>`.
- El footer `Refs:` es obligatorio — es la trazabilidad del harness llegando al historial de git. Sin él el commit pierde su origen.
- Si una phase mezcla trabajo de tipos distintos (feature + refactor no relacionado), dilo y propón commits separados con qué archivos van en cada uno.

Ejemplo:

```
feat(webhook): agrega reintentos con backoff exponencial a Gupshup

Los webhooks fallidos se perdían sin registro. Ahora se reintentan
hasta 5 veces con backoff y los agotados quedan en la dead-letter queue.

- Tabla webhook_retries con índice por status y next_attempt_at
- Worker de reintentos con backoff 2^n, techo de 15 min
- Idempotencia por message_id para no duplicar en reintento

Refs: docs/specs/reintentos-webhook-gupshup-spec.md · Requirements 1.1-1.4, 2.1
```
