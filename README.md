# Marketplace de plugins — Experenta

Catálogo de plugins de Claude Code. Se instala una vez y se actualiza en **todos** los proyectos con un comando.

| Plugin | Qué hace |
|---|---|
| [`ai-harness`](plugins/ai-harness/README.md) | Pipeline spec-driven: `grilling` → `/spec` → `/plan` → `/build`, más `/audit`. Un artefacto md por paso, trazabilidad hasta el commit |

## Instalar

En Claude Code, desde cualquier proyecto:

```
/plugin marketplace add JohnnieMiralda/claude-harness
/plugin install ai-harness@experenta
```

Elige scope **personal** en el diálogo de instalación: queda disponible en todos tus proyectos sin tocar el `.claude/` de ninguno.

> `/plugin` abre un panel interactivo. Si tu sesión no lo soporta (Claude Code web o desktop en algunos modos), córrelo desde una terminal con `claude`.

## Actualizar

Cuando publiques cambios aquí:

```
/plugin marketplace update
```

Eso refresca el catálogo en **todas** las instalaciones — no hay que copiar nada a ningún proyecto.

Los plugins solo reciben la actualización cuando sube el `version` de su `plugin.json`. Es a propósito: tú decides cuándo hay release, no cada commit.

## Publicar un cambio

```bash
# 1. edita el plugin
# 2. sube la versión en plugins/<plugin>/.claude-plugin/plugin.json
# 3. commit + push
git add -A && git commit -m "feat(ai-harness): <qué cambió>" && git push
```

Semver: `patch` para correcciones de redacción, `minor` para un comando o regla nueva, `major` cuando cambia el layout de `docs/` o el formato de los artefactos (rompe los task files existentes).

## Estructura

```
.claude-plugin/marketplace.json      catálogo (lo que lee /plugin marketplace add)
plugins/<plugin>/
├── .claude-plugin/plugin.json       manifiesto: nombre, versión, autor
├── commands/*.md                    slash commands
├── skills/<nombre>/SKILL.md         skills
└── harness/                         archivos de apoyo, vía ${CLAUDE_PLUGIN_ROOT}
CLAUDE.md                            contexto para editar este repo
```

Los componentes de un plugin quedan namespaced con su nombre: la skill `grilling` de `ai-harness` se invoca `ai-harness:grilling`. Eso evita colisiones con skills de otros plugins.

## Agregar un plugin nuevo

1. `plugins/<nombre>/.claude-plugin/plugin.json` con al menos `name`.
2. Sus `commands/` y `skills/`.
3. Agrégalo al array `plugins` de `.claude-plugin/marketplace.json` con su `source`.
4. Commit + push, y `/plugin marketplace update`.

## Probar sin publicar

Apunta el marketplace a la ruta local en vez de al repo:

```
/plugin marketplace add C:/Users/johnn/Documents/ExpeGit/skills
/plugin install ai-harness@experenta
/reload-plugins
```

Los cambios a un `SKILL.md` toman efecto de inmediato. Para todo lo demás — `commands/`, manifiestos, `harness/` — corre `/reload-plugins`.
