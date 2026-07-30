# Tasks: Correcciones del harness Andamio — 2026-07-29

> Fuente: `este mismo archivo, sección "Hallazgos de origen"` (review de arquitectura en chat, 2026-07-29)
> Generado: 2026-07-29 · Actualizado: 2026-07-30 (Phase 1, Phase 2, Phase 3, Phase 4, Phase 5 y Phase 6 ejecutadas y revisadas · Prueba de fuego movida a Phase 7 · Phase 8 agregada con los hallazgos no bloqueantes del review de Phase 6)

## Overview

Correcciones al harness Andamio salidas de una revisión de arquitectura. No hay
un reporte de `/audit` porque la revisión fue conversacional — los hallazgos
viven en este archivo (ver **Hallazgos de origen** al final) para que la
trazabilidad resuelva sin abrir nada más.

Ocho fases ordenadas por dependencia real, no por severidad. La Phase 0 fue
primero porque su resultado **cambiaba el contenido de la Phase 1**: si
`${CLAUDE_PLUGIN_ROOT}` no substituía en el cuerpo de un command, la corrección
de ruta no era una corrección de ruta, era un cambio de arquitectura. Substituye.

- **Phase 0: Verificación** — saber qué está roto de verdad antes de arreglar nada · ✅
- **Phase 1: Bugs que rompen** — lo que impide que el harness corra, o que puedas probar un arreglo · ✅
- **Phase 2: Garantías reales** — convertir promesas de prompt en mecanismo · ✅
- **Phase 3: Ingeniería de contexto** — la palanca grande · ✅
- **Phase 4: Consistencia y honestidad** — lo que sostiene el post público · ✅
- **Phase 5: Internacionalización** — el harness se publica en inglés, en una sola versión · ✅
- **Phase 6: Correcciones de review** — los 🟡/🟢 que los reviews de las Phases 1-5 dejaron fuera · ✅
- **Phase 7: Prueba de fuego** — lo único que valida las siete anteriores
- **Phase 8: Correcciones de review** — los 🟡/🟢 que el review de la Phase 6 dejó fuera

**El orden numérico es el orden de ejecución.** La internacionalización va antes
de la prueba de fuego a propósito: la Phase 7 tiene que validar la versión en
inglés, no una que dejó de existir. Y la Phase 6 va antes de la 7 porque varios
de sus hallazgos son inconsistencias de documentación que la 5 va a reescribir de
todos modos — hacerlos primero evita tocar las mismas líneas dos veces.

**Cuidado con el orden dentro de la Phase 5:** 5.1 define el criterio que las
otras tres aplican, y 5.3 es un breaking change que depende de que 5.2 ya haya
movido `CONVENCIONES.md`. No paralelizar.

## Tasks

### Phase 0: Verificación

- [x]   1. Instalar el plugin y confirmar que los componentes cargan
    - `/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/andamio`
    - `/plugin install andamio@miralda` → `/reload-plugins`
    - **Resultado:** el plugin carga. Registrado en `installed_plugins.json` como `andamio@miralda` v1.0.0, con los 4 commands y la skill presentes en `~/.claude/plugins/cache/miralda/andamio/1.0.0/`
    - **Scope `local` deliberado** — instalado solo para este repo mientras se prueba. Se instalará en los repos que corresponda después, cuando el harness esté validado
    - **Salida no prevista:** dos hallazgos nuevos, `R-16` (rutas stale) y `R-17` (comandos namespaced), más `R-18` al inspeccionar cómo se resuelve la copia instalada
    - _Findings: R-16, R-17, R-18_
    - _Agent: human decision_
    - _Model: n/a — `/plugin` is an interactive panel_

- [x]   2. Confirmar si `${CLAUDE_PLUGIN_ROOT}` substituye en el cuerpo de un command
    - Correr `/andamio:audit README.md` en este repo (alcance mínimo, barato)
    - Observar la **primera llamada Read** del comando: ¿leyó una ruta absoluta bajo `~/.claude/plugins/cache/miralda/andamio/1.0.0/harness/`, o falló con el string literal `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`?
    - El modo de falla que se busca es *silencioso*: hay que ver la llamada Read, no el resultado final del comando
    - Interrumpir después de esa primera Read — la auditoría de un README no aporta nada y escribe dos archivos que habría que borrar
    - **Resultado:** confirmado. El comando cargó con la instrucción ya resuelta a `C:/Users/johnn/Documents/ExpeGit/andamio/plugins/andamio/harness/CONVENCIONES.md` — sin rastro del string literal `${CLAUDE_PLUGIN_ROOT}`. Interrumpido tras esa observación, sin generar `AUDIT-2026-07-29.md` ni su task file derivado
    - **Contingencia si falla:** la corrección no es de ruta. CONVENCIONES pasa a ser una skill (`skills/convenciones/SKILL.md`), donde el placeholder sí está verificado, y los comandos la invocan en vez de leer un archivo. Eso reescribe la tarea 1.1 y sube a `opus` porque cambia la arquitectura de referencia de tres componentes. Si se confirma roto: parar y replantear la Phase 1
    - _Findings: R-03_
    - _Agent: human decision_
    - _Model: n/a_

### Phase 1: Bugs que rompen

- [x]   1. Corregir la ruta de CONVENCIONES en `/build`
    - `plugins/andamio/commands/build.md:23` → `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md`
    - Agregar la línea de override de proyecto, idéntica a `plan.md:17` y `audit.md:17`
    - Eliminar la referencia a `~/.claude/harness/` — nada expande el `~`, y en Windows es ambiguo
    - _Findings: R-01 (build.md:23)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   2. Excepción de árbol sucio entre phases
    - `build.md` Paso 0.3 bloquea duro y contradice la regla dura "no commitees salvo que te lo pida": el flujo phase-por-phase que el README recomienda se auto-bloquea en la segunda corrida
    - Regla nueva: si el árbol está sucio, mostrar `git status --short` y **preguntar una vez** si es trabajo de una phase anterior de este mismo task file
    - **No** intentar detectar la procedencia automáticamente — preguntar es más barato y más confiable
    - Mantener el bloqueo duro solo para "estás en la rama principal"
    - _Findings: R-02 (build.md:24, build.md:92)_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   3. Actualizar el header de `CONVENCIONES.md`
    - `harness/CONVENCIONES.md:3` dice "Fuente única de verdad para `/plan` y `/audit`" — falta `/build`
    - `/build` depende de: tabla `_Agente:_`, tabla `_Modelo:_`, formato de `### Phase N: Correcciones de review`, convención `_Review:_`
    - _Findings: R-11 (CONVENCIONES.md:3)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   4. Corregir las rutas stale `ExpeGit/skills` → `ExpeGit/andamio`
    - `CLAUDE.md:52` y `README.md:84` documentan una ruta que ya no existe. El repo se renombró a `andamio` y las instrucciones de prueba local apuntan al nombre viejo
    - Se detectó en uso real: el primer `marketplace add` de la Phase 0 falló por esto
    - _Findings: R-16 (CLAUDE.md:52, README.md:84)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   5. Corregir el nombre de los comandos en la documentación
    - Los componentes de un plugin quedan namespaced con el nombre del plugin: son `/andamio:spec`, `/andamio:plan`, `/andamio:audit`, `/andamio:build`, `/andamio:grilling`. `commands/*.md` se registran como skills igual que `skills/*/SKILL.md`
    - `README.md:77` afirma que solo la skill queda namespaced. Falso: los cuatro comandos también
    - `plugins/andamio/README.md:100-127` — el bloque "Uso típico" usa `/spec`, `/plan`, `/audit`, `/build` sin prefijo. **Es el ejemplo que sigue quien instala el plugin por primera vez, y no funciona.** Prioridad alta por eso, no por severidad técnica
    - Revisar también los diagramas de flujo de ambos READMEs y de `CLAUDE.md`
    - Aprovechar para documentar que el scope de instalación puede ser personal (todos los proyectos) o local (un repo) — instalar por repo mientras se prueba es un patrón legítimo, hoy el README solo menciona personal
    - _Findings: R-17 (README.md:77, plugins/andamio/README.md:100-127)_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   6. Arreglar el loop de desarrollo local
    - El marketplace apuntado a una ruta local es un **puntero vivo** al working tree (`"source": "directory"`), pero el plugin instalado es una **copia en caché** bajo `cache/miralda/andamio/1.0.0/`
    - Con `version` pineado en `plugin.json`, la documentación es explícita: el usuario solo recibe la actualización cuando subes la versión. Editas el working tree → el plugin instalado sigue sirviendo la copia vieja. `/reload-plugins` recarga el caché, no lo re-sincroniza
    - Eso hace falsa la sección "Probar antes de publicar" de `CLAUDE.md:57` y `README.md:89`, que afirman que editar toma efecto de inmediato
    - Corrección: **omitir `version` mientras se itera.** Sin el campo, Claude Code cae al commit SHA y cada commit cuenta como versión nueva. Re-pinear al publicar (tarea 5.2)
    - Medir y documentar el ciclo real: ¿un reinstall recoge cambios **sin commitear**, o hay que commitear primero? La respuesta cambia el flujo de trabajo y hoy no está documentada en ninguna parte
    - **Bloquea la verificación de todo lo demás**: sin esto, editas y pruebas la copia vieja sin darte cuenta
    - Sustituye a la tarea que decía "bajar `version` a 0.1.0" — omitir es más útil que bajar
    - _Findings: R-18 (plugin.json:4, CLAUDE.md:47-57, README.md:79-89)_
    - _Agent: supervised main agent_
    - _Model: sonnet_

### Phase 2: Garantías reales

- [x]   1. Crear `agents/andamio-reviewer.md`
    - Mover los 4 puntos de verificación de `build.md` Paso 3 al agente. `build.md` pasa a **invocarlo**, no a re-describirlo (invariante 2)
    - `tools: Read, Grep, Glob` — **sin Bash**. Un reviewer con Bash no es read-only, es un agente con shell y una promesa. El orquestador ya corrió los tests: que le pase la salida en el prompt. Menos superficie y menos texto
    - Definir el formato de salida (severidad 🔴🟠🟡🟢, `archivo:línea`, bloquea sí/no) **en el agente**, para que el rigor no dependa de cómo el orquestador redacte el prompt esa corrida
    - Actualizar `build.md` Paso 3 y ambos READMEs para que apunten al agente
    - _Findings: R-05, R-13_
    - _Agent: supervised main agent_
    - _Model: opus_

- [x]   2. Crear `agents/andamio-advisor.md`
    - Mismo patrón que 2.1, ya establecido. `tools: Read, Grep, Glob`
    - Los 5 gatillos de escalada se quedan en `build.md` — son del orquestador. El agente define **cómo responde**: diagnóstico o decisión entre opciones, nunca implementación
    - _Findings: R-13_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]   3. Decidir quién corrige los hallazgos 🔴/🟠
    - Hoy opus detecta y sonnet —el que escribió el bug— corrige. El invariante 7 se honra en detección y se rompe en corrección
    - **Decidido: Opción A** — un subagente opus corrige los 🔴/🟠, no el modelo que escribió el bug. `build.md` Paso 3 actualizado
    - _Findings: R-06 (build.md:85)_
    - _Agent: human decision_
    - _Model: sonnet, once the option is chosen_

- [x]   4. Corregir el reclamo de `allowed-tools` en ambos READMEs
    - `README.md:50` y `plugins/andamio/README.md:138` afirman que los comandos de generación "no pueden tocar tu código"
    - Es cierto para el main loop; los subagentes de `/audit` (que tiene `Task`) no heredan la restricción. "Los subagentes NO escriben archivos" es un prompt, no un permiso
    - Redacción honesta que siga siendo fuerte: el main loop está restringido por `allowed-tools`; los subagentes son read-only por prompt. Nombrar la diferencia es más creíble que ocultarla
    - Es el reclamo que alguien técnico va a auditar primero al leer el post
    - _Findings: R-04 (README.md:50, plugins/andamio/README.md:138)_
    - _Agent: supervised main agent_
    - _Model: sonnet_

### Phase 3: Ingeniería de contexto

- [x]   1. `/build` ejecuta UNA phase por defecto
    - Hoy `/build <archivo>` sin phase corre todo en un solo contexto: el orquestador acumula cada retorno de subagente, cada salida de tests y cada diff, mientras las reglas duras quedan al inicio del contexto y se sub-ponderan progresivamente (lost-in-the-middle sobre las reglas que definen la corrección del harness)
    - Cambio: sin argumento de phase, ejecutar **la siguiente phase con tareas en `[ ]`** y al cerrarla indicar cuál sigue. Correr todo requiere argumento explícito (`todas`)
    - Es menos texto que un protocolo de re-anclaje y da un contexto realmente fresco por phase, no uno simulado. La phase ya es la unidad coherente de trabajo según el propio `build.md`
    - Actualizar: `build.md` (interpretación del argumento, Paso 0, reporte final), ambos READMEs, el ejemplo de uso típico
    - **Depende de 1.2** — sin la excepción de árbol sucio esto se bloquea solo
    - _Findings: R-08_
    - _Agent: supervised main agent_
    - _Model: opus_

- [x]   2. Bitácora de corrida
    - `docs/tasks/<slug>-run-<YYYY-MM-DD>.md`: append de una línea por tarea con tarea, agente, modelo, archivos tocados, estado de tests, veredicto
    - Hoy si `/build` muere a mitad de phase lo único que sobrevive son los `[x]` alcanzados y código sin commitear. Reanudar es re-derivar el estado desde `git diff` a mano — en un harness cuya tesis es trazabilidad
    - Agregar al layout de `CONVENCIONES.md`
    - _Findings: R-09_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]   3. Escalada de modelo cuando la tarea resiste
    - `CONVENCIONES.md:99` declara la política ("es más fácil escalar una tarea que falló que recuperar tokens quemados") y ningún paso la ejecuta
    - `build.md` Paso 1: si una tarea falla 2 veces en su `_Modelo:_` asignado, re-correrla un escalón arriba antes de invocar al advisor
    - Anotar la escalada en la bitácora (3.2): así la próxima regeneración de `/plan` tiene evidencia real para asignar modelos, en vez de estimar
    - **Depende de 3.2**
    - _Findings: R-07 (CONVENCIONES.md:99)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

### Phase 4: Consistencia y honestidad

- [x]   1. Sección "Límites conocidos" en el README raíz
    - Sin índice de estado del pipeline: `grep` da pendientes, no qué specs nunca se planificaron ni qué auditorías quedaron sin atender
    - Los subagentes no heredan `allowed-tools`
    - Sin resume automático de una corrida interrumpida
    - El plugin instalado es una copia en caché, no un espejo del working tree (ver 1.6)
    - Contraintuitivo pero cierto: esta sección hace el post **más** fuerte. Un harness que nombra sus techos se lee como ingeniería; uno que solo lista virtudes se lee como marketing
    - _Findings: R-10, R-04, R-18_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   2. Comentario en `.gitignore`
    - Aclarar que ignorar `docs/tasks/` y `docs/audit/` es porque **este** repo no los produce como parte del plugin. En un proyecto consumidor son el registro trazable y van commiteados
    - Decisión pendiente: este task file vive en `docs/tasks/` y por lo tanto no se versiona. Probablemente sí lo quieres en git — es el registro de por qué cambió el harness. Un `!docs/tasks/correcciones-harness-*.md` lo resuelve
    - _Findings: R-14 (.gitignore)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

### Phase 5: Internacionalización

Decisión del 2026-07-30: el harness se publica en inglés, en **una sola versión**.

Dos versiones en dos idiomas es un fork silencioso de seis archivos de prompts sin
compilador que detecte la divergencia. Cada corrección habría que aplicarla dos
veces, y la primera vez que se olvide una hay dos harnesses que se comportan
distinto sin forma de notarlo. Va contra el invariante 2 y el costo se paga cada
semana, no una vez.

- [x]   1. Política de idioma en `CONVENCIONES.md` y reescritura del invariante 8
    - No es un idioma, son **tres superficies** con criterios distintos:
        - **Documentación** (ambos READMEs, catálogo del marketplace) → inglés. Es lo que decide si alguien instala; un README en español tapa la mayor parte del alcance en GitHub. La norma interna ya contempla inglés para audiencia internacional
        - **Prompts** (`commands/`, `agents/`, `CONVENCIONES.md`, `SKILL.md`) → inglés. No por alcance: porque quien quiera modificar el harness tiene que leerlos, y son el código
        - **Artefactos que el harness produce** (spec, task file, mensaje de commit) → el idioma del proyecto. Si los prompts van a inglés sin más, los specs y los commits de un repo interno salen en inglés y los lee un equipo que opera en español
    - El idioma de salida se **parametriza, no se forkea**. Una línea en los prompts: escribir los artefactos en el idioma que usa el proyecto; si el `CLAUDE.md` del proyecto no lo especifica, inglés por defecto. Cero mecanismo nuevo — Claude ya lee el `CLAUDE.md` del proyecto destino, y la norma interna ya exige que exista antes de tocar código
    - `CLAUDE.md` invariante 8 pasa de "Idioma: español" a la regla de tres superficies. Sin eso el invariante contradice el repo el mismo día del cambio
    - **Primero esta tarea:** define el criterio que 5.2, 5.3 y 5.4 aplican
    - _Findings: R-19_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   2. Prompts a inglés
    - `commands/{spec,plan,audit,build}.md`, `harness/CONVENCIONES.md`, `agents/{andamio-reviewer,andamio-advisor}.md`
    - `skills/grilling/SKILL.md` **se queda como está**. El cuerpo ya está en inglés y es la redacción original y probada; "interview me relentlessly" no tiene equivalente exacto en español. Traducirlo era el riesgo que iba a introducir la tarea retirada 4.2
    - Los `description` del frontmatter quedan en inglés con frases gatillo en inglés: se matchean contra la intención del usuario
    - **No es trabajo mecánico.** Traducir un prompt cambia comportamiento — por eso no es `haiku` aunque parezca sustitución de texto
    - Depende de 5.1
    - _Findings: R-19_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   3. `_Agente:_` / `_Modelo:_` → `_Agent:_` / `_Model:_`
    - Campos en español dentro de prompts en inglés es peor que cualquiera de las dos opciones puras: quien contribuya no sabe si es intencional o quedó a medias
    - **Breaking change.** Rompe la trazabilidad de todo task file existente → `major` por la política de semver del README (ver 7.2)
    - **Migrar este archivo en el mismo cambio.** Si `CONVENCIONES.md` pasa a inglés y este task file mantiene los campos en español, `/andamio:build` deja de parsearlo. Es el único task file que existe hoy: hacerlo ahora sale gratis, en un mes es una migración real con `[x]` marcados que nadie quiere tocar a mano
    - Alcance: `CONVENCIONES.md` (formato y tablas), los 4 commands, `agents/*.md` si los mencionan, ambos READMEs, y este task file completo
    - Depende de 5.2
    - _Findings: R-19_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   4. Documentación a inglés
    - `README.md`, `plugins/andamio/README.md`, `CLAUDE.md`, la `description` de `.claude-plugin/marketplace.json` y la descripción del repo en GitHub
    - `CLAUDE.md` también, porque es contribuidor-facing: quien quiera modificar el harness lo lee primero
    - Conservar el juego de "Andamio, no muleta" — "Scaffolding, not a crutch" funciona en inglés y es la línea que carga el post
    - Incluye lo que la tarea 4.1 haya agregado como "Límites conocidos"
    - Depende de 5.1
    - _Findings: R-19_
    - _Agent: supervised main agent_
    - _Model: sonnet_

### Phase 6: Correcciones de review — 2026-07-29

Hallazgos 🟡/🟢 de los reviews independientes (opus) de la Phase 1, la Phase 2,
la Phase 3, la Phase 4 y la Phase 5, cada uno en una o dos rondas (la primera sobre las
tareas completas, la segunda verificando los arreglos 🟠 de esa primera ronda,
cuando hubo). Los 🔴/🟠 ya se arreglaron en la misma corrida de cada phase —
ver los commits correspondientes. Esto es lo que quedó fuera por no bloquear.

- [x]   1. Actualizar referencia stale en `plugins/andamio/README.md:75`
    - Dice "`/plan` y `/audit` lo leen vía `${CLAUDE_PLUGIN_ROOT}`" — `/build` también lo lee desde la tarea 1.1 de esta misma Phase 1, y el header de `CONVENCIONES.md:3` ya lo refleja (tarea 1.3). Esta línea quedó un nivel arriba sin corregir
    - _Review: Phase 1 (ronda 1, hallazgo 🟡6)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   2. Alinear el diagrama de `plugins/andamio/README.md` (líneas 8-20)
    - El `←` de la línea 11 queda en la columna 34 contra la 35 de las líneas 14 y 20 — desfase de una columna tras agregar el prefijo `/andamio:` en la tarea 1.5. Cosmético, pero es el primer bloque del README del plugin
    - _Review: Phase 1 (ronda 1, hallazgo 🟡7)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   3. Prefijo namespaced en `.claude-plugin/marketplace.json:12`
    - La descripción del plugin en el catálogo dice "grilling → /spec → /plan → /build, más /audit" sin `/andamio:`. Es la tarjeta que ve el panel `/plugin` — primer contacto de quien instala, mismo problema de fondo que R-17 pero en un archivo fuera del alcance literal de la tarea 1.5
    - _Review: Phase 1 (ronda 1, hallazgo 🟢12)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   4. Colapsar doble verificación de `git status` en `build.md:25-26`
    - El ítem 3 del Paso 0 corre `git status` y luego pide `git status --short` — dos llamadas donde una basta
    - _Review: Phase 1 (ronda 1, hallazgo 🟢13)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]*  5. Prefijo namespaced en el resto de `grilling/SKILL.md`
    - Quedan dos menciones en prosa de `/spec` sin prefijo (líneas ~16 y ~20): "Documentar es trabajo de `/spec`" y "porque `/spec` las va a necesitar". El segundo reviewer las clasificó 🟡 (no 🟠) porque son referencias a la etapa del pipeline, no instrucciones que el usuario copie literalmente — mismo registro que `CLAUDE.md` y ambos READMEs usan en prosa. Opcional: solo si se quiere el criterio 100% literal de namespacing en todo el repo
    - _Review: Phase 1 (ronda 2, hallazgo 🟡 residual)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   6. Aclarar la tensión de lectura en `README.md:90` vs `:92`
    - La línea 90 dice que la versión sale del commit SHA (implicando que sin commitear no cambia la versión); la línea 92 dice que `claude plugin update` recoge cambios sin commitear. Ambas son ciertas — son dos mecanismos distintos (identificador de versión vs. contenido copiado) — pero el texto no lo explicita y un lector técnico puede leerlo como contradicción. Una cláusula de media línea en 92 lo cierra
    - _Review: Phase 1 (ronda 2, hallazgo 🟡)_
    - _Agent: supervised main agent_
    - _Model: sonnet_

- [x]   7. Agregar `agents/*.md` a los diagramas de estructura
    - `README.md:69-73` y `CLAUDE.md` (sección Estructura) no listan `plugins/andamio/agents/`, que desde la Phase 2 es un componente de primera clase (`andamio-reviewer.md`, `andamio-advisor.md`) y que `README.md:51` ya referencia por ruta. Quien audite ese reclamo va al bloque de estructura y no encuentra el directorio
    - _Review: Phase 2 (ronda 2, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   8. Consolidar los 5 gatillos de escalada del advisor en una sola fuente
    - Hoy viven completos en `build.md` Paso 2 (el hogar pactado en la tarea 2.2), se reenumeran completos en `agents/andamio-advisor.md` justo después de decir "no son asunto tuyo", y se resumen de nuevo en `plugins/andamio/README.md`. Riesgo de drift entre las tres copias — invariante 2 de `CLAUDE.md` pide una sola fuente
    - _Review: Phase 2 (ronda 1 y 2, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]   9. Agregar nota de verificación de namespacing de agentes a la prueba de fuego
    - `build.md` Paso 2 y Paso 3 invocan a los agentes reviewer/advisor por nombre pelado (`andamio-advisor`, `andamio-reviewer`), sin haber verificado si los agentes de un plugin quedan namespaced igual que los comandos (R-17) — mismo modo de falla silenciosa que R-01/R-03, que la Phase 0 sí verificó para el caso de los commands
    - **Hecho:** la línea vive en la tarea 7.1 (la prueba de fuego se movió de Phase 5 a Phase 7 al insertar la internacionalización)
    - _Review: Phase 2 (ronda 2, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  10. Reordenar la prosa del Paso 1 para que siga el orden de ejecución real
    - `build.md:49-55` describe la escalada de modelo, el marcado de `[x]` y el append a la bitácora antes del párrafo que corre los tests — pero es la salida de los tests la que produce la señal de fallo de la que dependen la escalada y la verificación. El texto se lee al revés de como se ejecuta
    - _Review: Phase 3 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  11. Definir el presupuesto de fallos del intento escalado
    - `build.md:49` no dice si el intento ya escalado de modelo tiene su propio margen de 2 fallos antes de ir al advisor, o si una sola falla ahí ya dispara el Paso 2. Acotado por el techo `opus` (no hay loop infinito), pero queda a criterio de cada corrida en vez de estar definido
    - _Review: Phase 3 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]  12. Resolver la tarea opcional inalcanzable bajo el default de "siguiente phase"
    - `build.md:19` excluye las tareas `[ ]*` opcionales al buscar "la siguiente phase con tareas en `[ ]`" si el resto de esa phase ya está en `[x]` — pero `build.md:51` dice que las opcionales se ejecutan "si el resto de la phase quedó verde". Con el default de una phase por corrida, una phase cuyo único pendiente es su tarea opcional nunca se selecciona sola: solo se alcanza nombrando la phase explícitamente, lo que contradice la intención de `:51`
    - _Review: Phase 3 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]  13. Corregir la referencia rota a la "tabla completa" de escalada de modelo
    - `build.md:49` remite a `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md` como si tuviera una tabla de escalada de modelo. Lo que existe ahí es la tabla de **asignación** de `_Modelo:_` (`CONVENCIONES.md:109-113`), no una escalera `haiku→sonnet→opus` con el umbral de 2 fallos — esa escalera vive solo, inline, en `build.md`. La referencia apunta a algo que no está, y por el invariante de una sola fuente la escalera debería vivir en CONVENCIONES.md, no en build.md
    - _Review: Phase 3 (ronda 1 y 2, hallazgo 🟡 — mismo hallazgo confirmado en ambas rondas)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]  14. Documentar el nombre de archivo de la bitácora para task files de `/audit`
    - `CONVENCIONES.md:13` da `<slug>-run-<YYYY-MM-DD>.md`, que para un task file de `/audit` resuelve a `audit-<fecha>-run-<fecha>.md` (dos fechas en el mismo nombre). Se deriva de "el nombre espeja el de su fuente", pero nunca se dice explícito — vale una línea de ejemplo para que no se lea como error tipográfico
    - _Review: Phase 3 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  15. Reencuadrar el árbol sucio entre phases como el camino normal, no la excepción
    - `build.md:29` (Paso 0.3) y `plugins/andamio/README.md:132` siguen redactando "si hay cambios sin commitear, pregúntame una vez" como si fuera un caso especial. Con una phase por corrida como default (Phase 3, tarea 1), encontrar el árbol sucio de la phase anterior es lo que pasa en toda corrida después de la primera — no una excepción
    - _Review: Phase 3 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]  16. Resolver la tensión entre el review de `todas` y "un commit por phase"
    - `build.md:75` (Paso 3) dice "al terminar la phase (o el archivo completo si ejecutaste todo)" — un solo review al final de una corrida con `todas`. Eso choca con "un commit por phase: la phase es la unidad coherente de trabajo" del Mensaje de commit. Preexistente a esta phase, pero el default nuevo (una phase por corrida) hace que `todas` sea la excepción explícita y vuelve más visible la inconsistencia
    - _Review: Phase 3 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [x]  17. Mapear el caso "tarea ya estaba hecha" a un valor de `Veredicto` de la bitácora
    - `build.md:92` ("si una tarea resulta imposible o ya estaba hecha, no la marques, anótalo en el chat y sigue") y el Reporte final no mencionan la bitácora de corrida. El caso "ya estaba hecha" no mapea limpio a ninguno de los tres valores documentados (`verificado` / `falló` / `salteada`) — forzarlo a `salteada` pierde la distinción de por qué se saltó
    - _Review: Phase 3 (ronda 2, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  18. Acotar la negación de `.gitignore` si no se quiere versionar la bitácora
    - `!docs/tasks/correcciones-harness-*.md` (tarea 4.2) matchea también `correcciones-harness-2026-07-29-run-<fecha>.md` — el patrón del slug de la bitácora de `CONVENCIONES.md:13` — no solo el task file, que es lo único que el comentario de al lado dice que se versiona. Si versionar la bitácora también es deseable, decirlo explícito en el comentario; si no, acotar a `!docs/tasks/correcciones-harness-*-tasks.md`
    - _Review: Phase 4 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  19. Quitar la duplicación del comando `grep` entre `README.md` y `CONVENCIONES.md`
    - `README.md:59` (bullet "No hay índice de estado del pipeline") repite el mismo `grep -rn "^- \[ \]" docs/tasks/` que ya vive en `CONVENCIONES.md:79` — contra el invariante 2 ("el README tampoco lo repite"). Referenciar la sección de CONVENCIONES.md en vez de repetir el comando
    - _Review: Phase 4 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  20. Quitar la oración duplicada en `README.md`
    - El bullet "El plugin instalado es una copia en caché..." (`README.md:62`) repite palabra por palabra la misma oración en negrita de `README.md:98`, en el mismo archivo. El bullet ya linkea a esa sección — con el link basta
    - _Review: Phase 4 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  21. Recortar el comentario de `.gitignore` a lo que es contexto de este repo
    - `.gitignore:4-12` tiene 9 líneas de comentario para 3 patrones, con un mini-tutorial de mecánica de git (por qué `docs/tasks/*` en vez de `docs/tasks/`) que no es contexto del repo, es documentación de cómo funciona `.gitignore` en general. Además la línea 5 dice "docs/tasks/" cuando el patrón real ya es `docs/tasks/*` — texto stale desde la propia tarea 4.2
    - _Review: Phase 4 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  22. Traducir el heading `# Entrevista` en `skills/grilling/SKILL.md:6`
    - Es el único residuo en español que queda en el archivo tras la corrección de la tarea 5.4 (que tradujo el cuerpo completo, `## Alcance`/`## Durante la entrevista`/`## Cierre`, dejando explícitamente sin tocar las líneas 1-13 por estar fuera del alcance literal del fix). `CONVENCIONES.md:110` clasifica `SKILL.md` como superficie de prompt en inglés — este heading es lo único que falta para que el archivo lo cumpla de verdad
    - _Review: Phase 5 (ronda 2, hallazgo 🟡 residual)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]  23. Actualizar referencias de línea y vocabulario desactualizadas por la traducción de Phase 5
    - La traducción de `build.md`, `README.md` y `CONVENCIONES.md` reflowó esos archivos e invalidó los `archivo:línea` que citan seis tareas pendientes de esta misma Phase 6: la 4 (`build.md:25-26` → ahora `:28-29`), la 6 (`README.md:90`/`:92` → ahora `:100`/`:102`), la 7 (`README.md:69-73` → ahora `:77-85`), la 13 (`CONVENCIONES.md:109-113` → ahora `:125-129`, y el hallazgo mismo habla de `_Modelo:_` cuando el campo ya es `_Model:_`), la 16 (`build.md:75` → ahora `:73`) y la 17 (`build.md:92` → ahora `:90`, y los tres valores que cita — `verificado`/`falló`/`salteada` — ahora son `verified`/`failed`/`skipped` en `CONVENCIONES.md`)
    - Además, el propio texto de la tarea 5.2 (ya `[x]`) afirma que `skills/grilling/SKILL.md` "se queda como está" porque "el cuerpo ya está en inglés" — la Phase 5 (ronda 1) encontró que era falso para la mayor parte del cuerpo, corregido en la tarea 22 de esta Phase 6. Y la tarea 5 de esta misma Phase 6 cita dos frases en español de ese archivo que ya no existen — la sustancia de la tarea sigue vigente (el `/spec` sin prefijo sobrevive en las mismas líneas, ahora en inglés), solo el texto citado quedó stale
    - _Review: Phase 5 (ronda 1, hallazgo 🟡; ronda 2, hallazgo 🟡 residual)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  24. Documentar el idioma de la bitácora de corrida en "Artifact language"
    - La sección "## Artifact language" de `CONVENCIONES.md` (agregada en la tarea 5.1) enumera "spec, task file, commit message" como los artefactos que produce el harness, pero omite la bitácora de corrida (`docs/tasks/<slug>-run-<YYYY-MM-DD>.md`), que la misma sección "## Run log" del archivo define como una salida de `/build` con sus propios headers de columna y enum de veredicto. Es exactamente el vacío que dejó a la bitácora de este repo con headers en español bajo una política ya en inglés
    - _Review: Phase 5 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [x]  25. Pulir 3 pasajes de la traducción a inglés
    - `CONVENCIONES.md:109` traduce "tapa la mayor parte del alcance en GitHub" como "hides most of the point" — pierde el sentido original (alcance/audiencia perdida) por uno más débil
    - `README.md:47` — "Andamio puts in five brakes" es un calco literal de "pone cinco frenos"; un lector nativo espera algo como "puts five brakes in place"
    - `CLAUDE.md` ("Testing before publishing") y `README.md` ("### Testing without publishing") quedaron como dos nombres casi idénticos para dos secciones que se linkean entre sí — fieles a la divergencia original en español ("Probar antes de publicar" / "Probar sin publicar"), pero en inglés la distinción que el español sí transmitía se perdió
    - _Review: Phase 5 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

### Phase 7: Prueba de fuego

- [ ]   1. Correr el pipeline completo contra un proyecto real
    - `/andamio:grilling` → `/andamio:spec` → `/andamio:plan` → `/andamio:build "Phase 1"` → `/andamio:build "Phase 2"` → `/andamio:audit`
    - Feature pequeña y real, no un ejemplo de juguete
    - Requiere instalar el plugin en ese repo — scope local por repo, como se hizo aquí
    - **Verificar el namespacing de los agentes:** `build.md` invoca `andamio-reviewer` y `andamio-advisor` por nombre pelado. Si los agentes de un plugin quedan namespaced como los comandos, esas invocaciones fallan — mismo modo de falla silenciosa que R-01 y R-03 (absorbe la tarea 6.9)
    - **Verificar el idioma de los artefactos:** correrlo en un repo cuyo `CLAUDE.md` diga español y confirmar que el spec, el task file y el mensaje de commit salen en español aunque los prompts estén en inglés. Es lo único que prueba que la parametrización de la tarea 5.1 funciona en vez de solo estar escrita
    - Anotar cada punto donde el harness se comportó distinto a lo documentado: ese es el input de la siguiente corrida de correcciones
    - Es lo único que valida las fases 0-6. Todo lo anterior es markdown que se *lee* bien
    - _Findings: R-03, R-19_
    - _Agent: human decision_
    - _Model: n/a_

### Phase 8: Correcciones de review — 2026-07-30

Hallazgos 🟡/🟢 del review independiente (opus) de la Phase 6. Los 5 🟠 que
encontró ya se arreglaron en la misma corrida — ver la bitácora
`docs/tasks/correcciones-harness-2026-07-29-run-2026-07-30.md`. Esto es lo
que quedó fuera por no bloquear.

- [ ]   1. Re-agregar el recordatorio de apéndice a la bitácora en el Paso 1 de `build.md`
    - El reorder de la tarea 6.10 (Phase 6) movió la prosa para que seguir el orden real de ejecución (tests → escalada → marcar `[x]`), pero en el camino la oración que instruye appendear la fila a la bitácora de corrida desapareció del Paso 1. Sigue siendo obligatorio vía la lectura de `CONVENCIONES.md` en el Paso 0.2, pero el recordatorio en el punto de ejecución se perdió, y el Reporte final tampoco la menciona
    - _Review: Phase 6 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   2. Resolver el riesgo de que una corrida `todas` ejecute en el mismo run las correcciones que su propio review acaba de crear
    - La tarea 6.16 hizo que el review corra una vez por phase incluso bajo `todas` — cada phase agrega sus hallazgos 🟡/🟢 como una `### Phase N: Correcciones de review` nueva al final del mismo archivo que el recorrido de `todas` sigue atravesando. Eso significa que una corrida `todas` puede alcanzar y ejecutar tareas que su propio review acaba de crear en esa misma corrida, contra la regla "no los arregles ahora: no estaban en el plan"
    - _Review: Phase 6 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: sonnet_

- [ ]   3. Sincronizar la enumeración de artefactos entre `CLAUDE.md` invariante 8 y `CONVENCIONES.md` "Artifact language"
    - La tarea 6.24 agregó la bitácora de corrida a la lista de `CONVENCIONES.md` ("spec, task file, run log, commit message"), pero `CLAUDE.md` invariante 8 sigue enumerando solo "spec, task file, commit message" — las dos listas gemelas que definen la misma política de idioma quedaron desincronizadas
    - _Review: Phase 6 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   4. Quitar la duplicación restante del comando `grep` en `plugins/andamio/README.md`
    - La tarea 6.19 quitó la duplicación del `grep -rn "^- \[ \]" docs/tasks/` en el README raíz, pero `plugins/andamio/README.md` todavía repite el mismo comando literal que ya vive en `CONVENCIONES.md` — el invariante 2 ("el README tampoco lo repite") sigue roto en el otro README
    - _Review: Phase 6 (ronda 1, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   5. Corregir 2 tareas ya `[x]` cuyo texto afirma hechos ahora falsos sobre `grilling/SKILL.md`
    - Distinto del caso de coordenadas de línea stale (tratado como historial aceptable): la tarea 5.2 (Phase 5, `[x]`) afirma que `skills/grilling/SKILL.md` "se queda como está" porque "el cuerpo ya está en inglés" — falso, es la razón por la que existió la tarea 6.22. Y la tarea 6.5 (`[x]`) cita dos frases en español de ese archivo que ya no existen tras la traducción. El review de Phase 6 distinguió esto explícitamente de la tarea 23 (moot): una premisa de hecho incorrecta en un registro `[x]` puede inducir a error a una futura regeneración de `/plan` o auditoría que lo lea, no es solo un número de línea desactualizado
    - _Review: Phase 6 (ronda 2, hallazgo 🟡)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   6. Aclarar qué "resto de la phase quedó verde" significa para una tarea opcional heredada de una corrida anterior
    - La tarea 6.12 resolvió que una phase cuyo único pendiente es su tarea opcional se vuelve alcanzable una vez que no queda ninguna otra phase con trabajo no-opcional pendiente. Pero `build.md` (Paso 1, "Run optional (`*`) tasks if the rest of the phase came out green") no tiene una señal verde que evaluar en la corrida actual cuando el resto de esa phase se puso verde en una corrida anterior — la lógica de la 6.12 es sólida, pero la condición que dispara esa oración queda ambigua en ese escenario
    - _Review: Phase 6 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   7. Traducir o documentar explícitamente el idioma de los comentarios de `.gitignore`
    - Los comentarios de `.gitignore` siguen en español mientras toda la documentación y los prompts pasaron a inglés. Las 3 superficies de "Artifact language" en `CONVENCIONES.md` no cubren comentarios de archivos de configuración del propio repo — es un vacío de la política, no una violación, pero las tareas 6.18 y 6.21 reescribieron esas líneas sin notarlo
    - _Review: Phase 6 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   8. Quitar la última duplicación del resumen de escalada de modelo en `build.md`
    - La tarea 6.13 movió la escalera completa de escalada de modelo a `harness/CONVENCIONES.md` ("## Model escalation"), pero `build.md` (Paso 1) todavía repite una oración-resumen que restablece la sustancia de la regla en vez de solo referenciar la sección — el invariante 2 apunta exactamente a esto, y es parte de por qué el hallazgo 🟠 de la tensión Paso 1/Paso 2 (bitácora, fila sin número) pudo aparecer
    - _Review: Phase 6 (ronda 1, hallazgo 🟢)_
    - _Agent: autonomous subagent_
    - _Model: haiku_

- [ ]   2. Re-pinear versión y publicar
    - Volver a poner `version` en `plugin.json` (lo quitó la tarea 1.6 para el loop de desarrollo)
    - **`major`**, no `minor`: la tarea 5.3 renombra `_Agente:_`/`_Modelo:_` y eso rompe todo task file existente — es exactamente el caso que la política de semver del README marca como `major`
    - `1.0.0` solo después de dos features reales de punta a punta. Antes de eso, `0.x`
    - Recordar: sin bump, el cambio no llega a nadie que ya lo tenga instalado
    - _Findings: R-18, R-19_
    - _Agent: human decision_
    - _Model: n/a_

## Hallazgos de origen

Revisión de arquitectura del 2026-07-29. Sustituye al reporte de `/audit` que no
existe — la revisión fue conversacional. R-16 a R-18 salieron de la instalación
real durante la Phase 0.

| ID | Sev | Ubicación | Hallazgo |
|---|---|---|---|
| R-01 | 🔴 | `build.md:23` | Ruta vieja `.claude/harness/CONVENCIONES.md`. Instalado como plugin no existe: el único comando que escribe código corre sin convenciones, y falla en silencio |
| R-02 | 🔴 | `build.md:24` + `:92` | "Detente si hay cambios sin commitear" contra "no commitees salvo que te lo pida": el flujo phase-por-phase que el README recomienda se auto-bloquea en la segunda corrida |
| R-03 | 🔴 | 3 de 5 componentes | `${CLAUDE_PLUGIN_ROOT}` verificado en contenido de skills y agents; **no** verificado empíricamente en el cuerpo de un command |
| R-04 | 🟠 | `README.md:50`, `plugins/andamio/README.md:138` | El reclamo "los comandos de generación no pueden tocar tu código" solo aplica al main loop. Los subagentes de `/audit` no heredan `allowed-tools` |
| R-05 | 🟠 | `build.md:66-81` | Cero agentes reusables: el rigor del reviewer varía corrida a corrida según cómo el orquestador redacte el prompt del que va a criticar su propio trabajo |
| R-06 | 🟠 | `build.md:85` | Opus detecta los 🔴/🟠 pero sonnet —el que escribió el bug— los corrige. El invariante 7 se rompe en la corrección |
| R-07 | 🟠 | `CONVENCIONES.md:99` | La política de escalada de modelo está declarada y ningún paso la ejecuta |
| R-08 | 🟠 | `build.md` completo | Sin frontera de contexto por phase. El contexto del orquestador crece monótonamente y las reglas duras se sub-ponderan progresivamente |
| R-09 | 🟠 | `build.md` Paso 1 | La ejecución no deja bitácora. Una corrida interrumpida no se puede reanudar sin re-derivar el estado a mano |
| R-10 | 🟡 | `README.md` | Sin índice de estado del pipeline. Correcto a esta escala (YAGNI), pero es un techo sin nombrar |
| R-11 | 🟡 | `CONVENCIONES.md:3` | El header omite `/build`, que sí depende del archivo |
| R-13 | 🟡 | falta `agents/` | Advisor y reviewer son read-only por prompt, no por configuración de herramientas |
| R-14 | 🟢 | `.gitignore` | Ignora `docs/tasks/` y `docs/audit/`: correcto aquí, contradice la tesis si se copia el patrón a un proyecto consumidor |
| R-16 | 🔴 | `CLAUDE.md:52`, `README.md:84` | Rutas stale `ExpeGit/skills`: el repo se renombró a `andamio`. Detectado en uso real — el primer `marketplace add` de la Phase 0 falló por esto |
| R-17 | 🔴 | `README.md:77`, `plugins/andamio/README.md:100-127` | Los comandos quedan namespaced (`/andamio:audit`), no sin prefijo. El bloque "Uso típico" es el primer contacto de quien instala el plugin y no funciona |
| R-18 | 🟠 | `plugin.json:4`, `CLAUDE.md:47-57`, `README.md:79-89` | El plugin instalado es una copia en caché y `version` está pineado: editar el working tree no cambia lo que corre. "Probar antes de publicar" es falso |
| R-19 | 🟠 | todo el repo | El harness está en español y la meta es distribución pública. Pero el idioma de los prompts determina el idioma de los artefactos, así que "traducirlo todo" rompe el uso interno. Hay que separar superficies y parametrizar el idioma de salida, no forkear el repo en dos idiomas |

**Retirado:** `R-15` (`version: 1.0.0` en software sin ejecutar). Lo absorbe R-18,
que es el mismo campo por una razón más grave: rompe el ciclo de desarrollo, no
solo exagera la madurez.

**Invertido:** `R-12` decía "cuerpo de `grilling` en inglés contra el invariante 8".
El invariante era el equivocado, no el archivo. Lo sustituye R-19: `grilling` se
queda en inglés y el resto del harness se le suma.

**No se convirtió en tarea (deuda aceptada):** R-10. A esta escala un índice de
estado del pipeline es sobre-ingeniería; queda documentado como techo conocido
en la tarea 4.1 en vez de resolverse.
