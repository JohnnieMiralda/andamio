# Andamio

**Estructura temporal que sostiene mientras se levanta la obra.** Un pipeline spec-driven para Claude Code: nada se construye sin spec, ninguna tarea existe sin origen, y la trazabilidad llega hasta el mensaje del commit.

Andamio, no muleta: no escribe por ti, te sostiene la disciplina.

```
grilling <tema>   entrevista de diseño, 1 pregunta a la vez. No escribe nada.
      ↓
/spec             → docs/specs/<slug>-spec.md        requirements 1.1, 2.3...
      ↓
/plan             → docs/tasks/<slug>-tasks.md       tareas con agente y modelo
      ↓
/build            → código + review independiente + mensaje de commit

/audit [ruta]     → docs/audit/AUDIT-<fecha>.md      hallazgos A-01, A-02...
                  → docs/tasks/audit-<fecha>-tasks.md
```

Cada tarea sabe **de dónde viene** (`_Requirements: 1.1_` / `_Findings: A-03_`), **quién la ejecuta** (subagente, agente supervisado o tú) y **con qué modelo** (el más barato que la complete de forma confiable). Ese footer llega al commit, así que dentro de seis meses `git log` te dice de qué spec salió cada línea.

## Instalar

En Claude Code, desde cualquier proyecto:

```
/plugin marketplace add JohnnieMiralda/andamio
/plugin install andamio@miralda
```

Elige scope **personal** en el diálogo: queda disponible en todos tus proyectos sin tocar el `.claude/` de ninguno.

> `/plugin` abre un panel interactivo. Si tu sesión no lo soporta, córrelo desde una terminal con `claude`.

Documentación completa del pipeline: [plugins/andamio/README.md](plugins/andamio/README.md).

## Actualizar

```
/plugin marketplace update
```

Refresca **todas** tus instalaciones — no se copia nada a ningún proyecto. Los plugins solo reciben la actualización cuando sube el `version` de su `plugin.json`: tú decides cuándo hay release, no cada commit.

## Por qué existe

Claude Code escribe código rápido. El problema no es la velocidad, es que sin estructura terminas con features que nadie especificó, tareas sin origen, y un `git log` que no explica nada. Andamio pone cuatro frenos:

- **Ningún paso hace dos cosas.** La entrevista no escribe specs; el spec no planifica; el plan no ejecuta. Cada artefacto es revisable por separado.
- **Los comandos de generación no pueden tocar tu código.** `allowed-tools` restringido: leen y escriben markdown, nada más. `/build` es el único que escribe código.
- **Quien implementa no revisa.** El review corre en un subagente aparte y en un modelo distinto al que escribió el código. Puntos ciegos correlacionados es justo lo que un review debe romper.
- **Regenerar no borra progreso.** Si el spec cambia, el plan se actualiza preservando lo que ya marcaste como hecho.

## Publicar un cambio

```bash
# 1. edita el plugin
# 2. sube version en plugins/andamio/.claude-plugin/plugin.json
# 3. commit + push
git add -A && git commit -m "feat(andamio): <qué cambió>" && git push
```

Semver: `patch` para redacción, `minor` para un comando o regla nueva, `major` cuando cambia el layout de `docs/` o el formato de los artefactos — eso rompe los task files existentes.

## Estructura del repo

```
.claude-plugin/marketplace.json      catálogo (lo que lee /plugin marketplace add)
plugins/andamio/
├── .claude-plugin/plugin.json       manifiesto: nombre, versión, autor
├── commands/*.md                    slash commands
├── skills/grilling/SKILL.md         skills
└── harness/CONVENCIONES.md          archivos de apoyo, vía ${CLAUDE_PLUGIN_ROOT}
CLAUDE.md                            contexto para editar este repo
```

Los componentes quedan namespaced con el nombre del plugin: la skill `grilling` se invoca `/andamio:grilling`. Eso evita colisiones con skills de otros plugins.

### Probar sin publicar

Apunta el marketplace a la ruta local en vez de al repo:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/skills
/plugin install andamio@miralda
/reload-plugins
```

Editar un `SKILL.md` toma efecto de inmediato. Cambios a `commands/`, manifiestos o `harness/` requieren `/reload-plugins`.

## Licencia

MIT
