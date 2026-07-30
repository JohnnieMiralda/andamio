# Andamio — pipeline spec-driven

Pipeline donde cada herramienta deja un artefacto md que la siguiente consume. Cada paso hace **una** cosa: entrevistar, documentar, planificar, auditar, ejecutar. Ninguno hace dos.

## Flujo

```
/andamio:grilling <tema>          entrevista, 1 pregunta a la vez. No escribe nada.
      │
      ▼
/andamio:spec                    ← documenta la entrevista
      │
      ▼
docs/specs/<slug>-spec.md         ← requirements numerados (1.1, 2.3...)
      │
      ▼
/andamio:plan docs/specs/<slug>-spec.md
      │
      ▼
docs/tasks/<slug>-tasks.md        ← lista ejecutable, un archivo por spec


/andamio:audit [ruta]
      │
      ├──► docs/audit/AUDIT-<fecha>.md          ← hallazgos con IDs (A-01...)
      │
      └──► docs/tasks/audit-<fecha>-tasks.md    ← lista ejecutable, una por corrida


/andamio:build docs/tasks/<archivo>-tasks.md ["Phase 4" | todas]   ← ejecuta el trabajo
      │
      ▼
   código
```

## Ejecución

`/build` es el único comando que escribe código. Corre en **sonnet 5** y actúa como orquestador:

```
/andamio:build <task file> ["Phase N" | todas]  ← por defecto, la siguiente phase con [ ]
      │
      ├─ por cada tarea, según su _Agente:_
      │     subagente autónomo        → delega, con el _Modelo:_ de la tarea
      │     principal supervisado     → la hace él mismo
      │     decisión humana           → se detiene y te pregunta
      │
      ├─ trabado? → agents/andamio-advisor.md (read-only, aconseja pero no implementa)
      │
      ├─ al cerrar la phase → agents/andamio-reviewer.md (read-only, independiente)
      │       🔴🟠 → los corrige un subagente opus (una sola ronda de re-review)
      │       🟡🟢 → los agrega como phase nueva al task file
      │
      └─ y te entrega el mensaje de commit listo para copiar
```

Un commit por phase, en Conventional Commits (`feat(scope): asunto`), con footer `Refs:` al spec y los requirements que implementa — así la trazabilidad del harness llega al historial de git. `/build` **no commitea**: te da el mensaje y decides tú.

El advisor no se levanta "por si acaso": escala con gatillos concretos — el mismo error falla tras 2 intentos, el cambio se expande a más módulos de los que decía la tarea, aparece superficie de seguridad o concurrencia donde no estaba marcada, o hay un trade-off que el spec no resolvió.

Dos fuentes (`specs/`, `audit/`), un destino (`tasks/`). El nombre del task file **espeja** el de su fuente, así el par se encuentra sin abrir nada.

Todo lo pendiente del repo, sin archivo índice:

```bash
grep -rn "^- \[ \]" docs/tasks/
```

## Trazabilidad

Cada tarea apunta a su origen: las de `/plan` a `_Requirements: N.M_` del spec, las de `/audit` a `_Findings: A-NN_` del reporte. Sin origen no hay tarea.

Cada tarea indica además **quién** la ejecuta (`_Agente:_`) y **con qué modelo** (`_Modelo:_` — el más barato que la complete de forma confiable).

Las reglas de layout, formato, regeneración, agente y modelo viven en un solo lugar: **[`harness/CONVENCIONES.md`](harness/CONVENCIONES.md)**. `/plan` y `/audit` lo leen vía `${CLAUDE_PLUGIN_ROOT}`. Si cambia una regla, cambia ahí.

## Instalación

Se instala como plugin, una vez, y queda en todos tus proyectos:

```
/plugin marketplace add JohnnieMiralda/andamio
/plugin install andamio@miralda
```

Scope **personal** en el diálogo de instalación: queda en todos tus proyectos, sin copiar nada al `.claude/` de ninguno. Scope **local** es un patrón legítimo mientras pruebas un cambio en un repo puntual — instala solo ahí.

Actualizar, en todos los proyectos a la vez:

```
/plugin marketplace update
```

Detalles de publicación y versionado en el [README del marketplace](../../README.md).

### Convenciones propias por proyecto

Los comandos leen `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md` — la copia que viene en el plugin. Si un proyecto necesita reglas distintas (otro layout de `docs/`, otros criterios de modelo), pon un `.claude/harness/CONVENCIONES.md` en ese repo y **esa gana**. Anota arriba del archivo por qué difiere, o en seis meses no vas a saber si es intencional o quedó viejo.

## Uso típico

```bash
# 1. Feature nueva: entrevista de diseño
/andamio:grilling sistema de reintentos para el webhook de Gupshup
# ... respondes preguntas una a una ...
# > "listo, cerramos"

# 2. Documentar la entrevista
/andamio:spec
# → docs/specs/reintentos-webhook-gupshup-spec.md

# 3. Convertir el spec en tareas
/andamio:plan docs/specs/reintentos-webhook-gupshup-spec.md
# → docs/tasks/reintentos-webhook-gupshup-tasks.md

# 4. Auditar código existente (independiente del spec)
/andamio:audit src/handlers
# → docs/audit/AUDIT-2026-07-29.md
# → docs/tasks/audit-2026-07-29-tasks.md

# 5. Ejecutar — sin phase corre la siguiente pendiente, una por corrida (por defecto)
/andamio:build docs/tasks/reintentos-webhook-gupshup-tasks.md
# → orquesta subagentes, corre tests, review opus, marca [x], y te dice qué phase sigue

# ...o una phase puntual
/andamio:build docs/tasks/reintentos-webhook-gupshup-tasks.md "Phase 2"

# ...o el feature completo de corrido, en un solo contexto
/andamio:build docs/tasks/reintentos-webhook-gupshup-tasks.md todas
```

Trabaja en una rama. `/build` verifica `git status` antes de arrancar: si estás en la principal se detiene y te avisa; si hay cambios sin commitear te pregunta una vez si son de una phase anterior de este mismo task file.

## Reglas del sistema

- **Un task file por fuente.** Nunca un archivo compartido entre features: crece sin techo, cuesta tokens en cada ejecución y choca entre ramas.
- **Regenerar no borra progreso.** Si el spec cambia y corres `/plan` de nuevo, preserva los `[x]` de las tareas que no cambiaron, agrega las nuevas en `[ ]`, y manda a `## Obsoletas` las que ya estaban hechas y dejaron de aplicar. Reporta el delta en el chat.
- **Los checkboxes solo se marcan al ejecutar**, nunca al generar.
- **No renumeres requirements en un spec existente** — rompe la trazabilidad del task file. Agrega al final (1.4, 1.5).
- **Orden de creación: fuente primero, tareas después.** Si falla a mitad, te queda lo caro de reproducir.
- Los comandos de generación (`/spec`, `/plan`, `/audit`) tienen `allowed-tools` restringido en su main loop: no puede modificar tu código, solo leer y escribir los md del harness. Eso es un permiso real. Los subagentes que `/audit` levanta para explorar son read-only por instrucción de prompt, no por esa misma restricción — no heredan `allowed-tools`. `/build` es el único main loop que escribe código.
- **Quien implementa no revisa.** El review y el advisor corren como agentes propios (`agents/andamio-reviewer.md`, `agents/andamio-advisor.md`), en un modelo distinto al que escribió el código y sin `Write`/`Edit`/`Bash` en su configuración de herramientas — read-only por permiso, no solo por prompt. Sonnet revisando lo de sonnet comparte los puntos ciegos.
- **`/build` no toca la fuente.** Si la implementación revela que el spec está mal, te lo dice y sugiere volver a `grilling` — no lo edita por su cuenta.
- `grilling` no escribe archivos. Si te pide generar un spec, es un bug de la skill.
- Si vienes de la versión con `docs/TASKS.md` único: los comandos ya no lo leen ni lo escriben. Muévelo o bórralo tú — migrar a ciegas un archivo con checkboxes marcados no es trabajo de un comando.
