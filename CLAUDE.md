# Contexto — Marketplace de plugins

Este repo **no es una aplicación**: es un marketplace de plugins de Claude Code. El "código" son archivos markdown que Claude ejecuta como instrucciones.

Es el **origen**. Los proyectos no tienen copias — consumen el plugin instalado, así que una mejora aquí + `/plugin marketplace update` llega a todos. Nunca se edita en sentido contrario.

## Estructura

```
.claude-plugin/marketplace.json          catálogo del marketplace
plugins/andamio/
├── .claude-plugin/plugin.json           manifiesto (name, version, author)
├── skills/grilling/SKILL.md             entrevista de diseño (no escribe archivos)
├── commands/spec.md                     entrevista → docs/specs/<slug>-spec.md
├── commands/plan.md                     spec → docs/tasks/<slug>-tasks.md
├── commands/audit.md                    código → docs/audit/AUDIT-<fecha>.md
│                                                + docs/tasks/audit-<fecha>-tasks.md
├── commands/build.md                    task file → código (subagentes + review)
└── harness/CONVENCIONES.md              fuente única: layout, formato, agente, modelo
.claude/settings.json                    hooks de graphify de este repo (no es del plugin)
```

Flujo, instalación y publicación en [README.md](README.md).

## Reglas del repo como plugin

- **Los archivos de apoyo se referencian con `${CLAUDE_PLUGIN_ROOT}`**, no con rutas relativas ni `.claude/`. El placeholder se substituye dentro del contenido de skills y commands. Un plugin instalado vive en un directorio de caché — una ruta relativa al proyecto no lo encuentra.
- **Nada fuera del directorio del plugin.** Al instalar se copia solo `plugins/<nombre>/`; un `../algo-compartido` no viaja.
- **Subir `version` en `plugin.json` es lo que dispara la actualización** para quien ya lo tiene instalado. Un cambio sin bump no llega a nadie.
- `.claude/settings.json` es configuración de **este** repo, no del plugin. No se distribuye.

## Invariantes al editar

1. **Un paso, una responsabilidad.** `grilling` entrevista y nada más; `/spec` documenta; `/plan` planifica; `/audit` audita; `/build` ejecuta. Si una edición hace que un paso haga dos cosas, es la edición equivocada.
2. **Cero duplicación de reglas.** Layout, formato del task file, regeneración y criterios de `_Agente:_` / `_Modelo:_` viven **solo** en `harness/CONVENCIONES.md`. Los comandos lo referencian, no lo copian. El README tampoco lo repite.
3. **Un task file por fuente**, con nombre que espeja el de su fuente. Nunca un archivo compartido entre features.
4. **Regenerar no destruye trabajo hecho.** Todo comando que reescriba un task file existente preserva los `[x]` y reporta el delta. Los checkboxes solo se marcan al ejecutar, nunca al generar.
5. **Toda tarea generada es trazable** a un `_Requirements: N.M_` de un spec, un `_Findings: A-NN_` de una auditoría, o un `_Review: <phase>_` de un review.
6. **`allowed-tools` restringido.** Ningún comando de **generación** (`/spec`, `/plan`, `/audit`) puede tocar código del proyecto destino. `/build` es la única excepción — es el comando de ejecución — y por eso lleva sus propias reglas duras: no toca la fuente, no sale del alcance de la phase, no commitea sin permiso, no marca `[x]` sin verificar. Si agregas un comando de generación, restríngelo igual.
7. **Quien implementa no revisa.** El review de `/build` va en un subagente aparte y en un modelo distinto (opus) al que escribió el código (sonnet). Puntos ciegos correlacionados es justo lo que el review debe romper.
8. **Idioma: español.** Convención operativa interna.

## Al agregar un comando nuevo

Frontmatter mínimo: `description`, `argument-hint`, `allowed-tools` (mínimo necesario), `model`, `disable-model-invocation: true` — los comandos del harness se invocan a mano, no por decisión del modelo. Si escribe en `docs/tasks/`, que lea `${CLAUDE_PLUGIN_ROOT}/harness/CONVENCIONES.md` en el paso 1.

## Probar antes de publicar

Apunta el marketplace a la ruta local, no al repo remoto:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/skills
/plugin install andamio@miralda
/reload-plugins
```

Editar un `SKILL.md` toma efecto de inmediato. Cambios a `commands/`, manifiestos o `harness/` requieren `/reload-plugins`.
