# Andamio

**Estructura temporal que sostiene mientras se levanta la obra.** Un pipeline spec-driven para Claude Code: nada se construye sin spec, ninguna tarea existe sin origen, y la trazabilidad llega hasta el mensaje del commit.

Andamio, no muleta: no escribe por ti, te sostiene la disciplina.

```
/andamio:grilling <tema>   entrevista de diseño, 1 pregunta a la vez. No escribe nada.
      ↓
/andamio:spec            → docs/specs/<slug>-spec.md        requirements 1.1, 2.3...
      ↓
/andamio:plan            → docs/tasks/<slug>-tasks.md       tareas con agente y modelo
      ↓
/andamio:build           → código + review independiente + mensaje de commit

/andamio:audit [ruta]    → docs/audit/AUDIT-<fecha>.md      hallazgos A-01, A-02...
                         → docs/tasks/audit-<fecha>-tasks.md
```

Cada tarea sabe **de dónde viene** (`_Requirements: 1.1_` / `_Findings: A-03_`), **quién la ejecuta** (subagente, agente supervisado o tú) y **con qué modelo** (el más barato que la complete de forma confiable). Ese footer llega al commit, así que dentro de seis meses `git log` te dice de qué spec salió cada línea.

## Instalar

En Claude Code, desde cualquier proyecto:

```
/plugin marketplace add JohnnieMiralda/andamio
/plugin install andamio@miralda
```

Elige scope **personal** en el diálogo: queda disponible en todos tus proyectos sin tocar el `.claude/` de ninguno. Si solo quieres probarlo en un repo puntual, elige **local** — instala únicamente ahí, sin afectar el resto.

> `/plugin` abre un panel interactivo. Si tu sesión no lo soporta, córrelo desde una terminal con `claude`.

Documentación completa del pipeline: [plugins/andamio/README.md](plugins/andamio/README.md).

## Actualizar

```
/plugin marketplace update
```

Refresca **todas** tus instalaciones — no se copia nada a ningún proyecto. Los plugins solo reciben la actualización cuando sube el `version` de su `plugin.json`: tú decides cuándo hay release, no cada commit. Eso es para un plugin ya publicado — mientras se itera localmente sin `version` pineado es al revés, ver ["Probar sin publicar"](#probar-sin-publicar).

## Por qué existe

Claude Code escribe código rápido. El problema no es la velocidad, es que sin estructura terminas con features que nadie especificó, tareas sin origen, y un `git log` que no explica nada. Andamio pone cinco frenos:

- **Ningún paso hace dos cosas.** La entrevista no escribe specs; el spec no planifica; el plan no ejecuta. Cada artefacto es revisable por separado.
- **Los comandos de generación no pueden tocar tu código.** El main loop tiene `allowed-tools` restringido: lee y escribe markdown, nada más — eso es un permiso real. Los subagentes que `/audit` levanta para explorar son read-only por instrucción de prompt, no por ese permiso; no heredan `allowed-tools`. `/build` es el único main loop que escribe código.
- **Quien implementa no revisa.** El review y el advisor corren como agentes propios (`plugins/andamio/agents/`), en un modelo distinto al que escribió el código y sin `Write`/`Edit`/`Bash` en su configuración — read-only por permiso, no solo por prompt. Puntos ciegos correlacionados es justo lo que un review debe romper.
- **Cada corrida de `/build` es una phase, no el archivo entero.** Por defecto ejecuta la siguiente phase con tareas pendientes y te dice cuál sigue — un contexto fresco por phase, no una acumulación de diffs y salidas de test que termina sub-ponderando las reglas duras de su propio prompt. Correr todo de corrido requiere pedirlo explícito (`todas`).
- **Regenerar no borra progreso.** Si el spec cambia, el plan se actualiza preservando lo que ya marcaste como hecho.

## Publicar un cambio

```bash
# 1. edita el plugin
# 2. re-pinea version en plugins/andamio/.claude-plugin/plugin.json (se omite mientras se itera)
# 3. commit + push
git add -A && git commit -m "feat(andamio): <qué cambió>" && git push
```

Semver: `patch` para redacción, `minor` para un comando o regla nueva, `major` cuando cambia el layout de `docs/` o el formato de los artefactos — eso rompe los task files existentes.

## Estructura del repo

```
.claude-plugin/marketplace.json      catálogo (lo que lee /plugin marketplace add)
plugins/andamio/
├── .claude-plugin/plugin.json       manifiesto: nombre, autor (versión se omite mientras se itera)
├── commands/*.md                    slash commands
├── skills/grilling/SKILL.md         skills
└── harness/CONVENCIONES.md          archivos de apoyo, vía ${CLAUDE_PLUGIN_ROOT}
CLAUDE.md                            contexto para editar este repo
```

Los componentes quedan namespaced con el nombre del plugin: los cuatro comandos y la skill se invocan `/andamio:spec`, `/andamio:plan`, `/andamio:audit`, `/andamio:build`, `/andamio:grilling`. Eso evita colisiones con comandos y skills de otros plugins.

### Probar sin publicar

Apunta el marketplace a la ruta local en vez de al repo:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/andamio
/plugin install andamio@miralda
```

**El plugin instalado es una copia en caché, no un espejo del working tree.** Editar el repo no cambia lo que corre por sí solo.

Mientras iteras, **omite `version` en `plugin.json`** — sin el campo, Claude Code usa el commit SHA como versión, así que cada commit cuenta como una versión nueva. Con `version` fijo (como debe quedar al publicar), el plugin se pinea a esa versión: subirla sí entrega, pero un re-sync sin subirla no hace nada.

Para llevar un cambio (commiteado o no — no hace falta commitear primero) a la copia instalada:

```bash
claude plugin update andamio@miralda --scope local
```

Después, **reinicia la sesión de Claude Code** — el propio comando avisa "Restart to apply changes"; la sesión abierta sigue sirviendo lo que cargó al arrancar.

## Licencia

MIT
