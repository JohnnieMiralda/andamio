# Convenciones del harness

Fuente única de verdad para `/plan` y `/audit`. Si cambia una regla, cambia aquí — no en los comandos.

## Layout

```
docs/
├── specs/   <slug>-spec.md              ← /spec    (fuente)
├── audit/   AUDIT-<YYYY-MM-DD>.md       ← /audit   (fuente)
└── tasks/   <slug>-tasks.md             ← /plan    (derivado)
          audit-<YYYY-MM-DD>-tasks.md  ← /audit   (derivado)
```

Dos fuentes, un destino. **El nombre del task file espeja el de su fuente**, así el par se encuentra sin abrir nada.

**Orden de creación:** primero el artefacto fuente, después las tareas derivadas. Si el proceso falla a mitad, te queda lo caro de reproducir. `/audit` escribe su reporte y *luego* el task file; `/plan` solo lee el spec y escribe tareas.

Crea las carpetas si no existen. Un task file por fuente — nunca un archivo compartido entre features.

## Formato del task file

```markdown
# Tasks: <Nombre> — <YYYY-MM-DD>

> Fuente: `<ruta al spec o al reporte de auditoría>`
> Generado: <YYYY-MM-DD> · Actualizado: <YYYY-MM-DD>

## Overview

<Descripción breve del plan y las fases. Aquí va la narrativa arquitectónica
si la hay — no en un documento aparte.>

- **Phase 1: <nombre>** - <descripción>
- **Phase 2: <nombre>** - <descripción>

## Tasks

### Phase 1: <nombre>

- [ ]   1. <Título de la tarea>
    - <Paso concreto 1, mencionando archivos/módulos reales del codebase>
    - <Paso concreto 2>
    - _Requirements: 1.1, 2.3_          ← /plan
    - _Findings: A-01, A-03 (archivo:línea)_   ← /audit
    - _Agente: <subagente autónomo | agente principal supervisado | decisión humana>_
    - _Modelo: <haiku | sonnet | opus>_

- [ ]   2. <Título de la tarea>
    - [ ] 2.1 <Subtarea>
        - <Detalle de implementación>
        - _Requirements: ..._
        - _Agente: ..._
        - _Modelo: ..._
    - [ ]* 2.2 <Subtarea opcional, ej. tests>
        - **Property N: <propiedad a validar>**
        - **Validates: <Requirements N.M | A-NN>**
```

Numeración jerárquica para subtareas (2.1, 2.2). Las tareas con `*` son opcionales (típicamente tests de propiedades). Todas nacen en `[ ]` — los checkboxes solo se marcan al **ejecutar**, nunca al generar.

**Trazabilidad**: toda tarea apunta a su origen. `/plan` usa `_Requirements: N.M_` del spec; `/audit` usa `_Findings: A-NN_` del reporte; `/build` usa `_Review: <phase>_` para las correcciones que salen de un review. Sin origen no hay tarea.

## Regeneración: no borres progreso

Si el task file ya existe, **no lo sobreescribas de cero**. La fuente cambió, tu trabajo hecho no:

1. Lee el archivo existente antes de escribir.
2. **Preserva el estado `[x]`** de toda tarea cuyo título y trazabilidad no cambiaron.
3. Las tareas nuevas nacen en `[ ]`.
4. Una tarea que ya no aplica: si estaba en `[ ]`, desaparece. Si estaba en `[x]`, muévela a una sección `## Obsoletas` al final con una línea de por qué — el registro de trabajo hecho no se borra en silencio.
5. Actualiza `Actualizado:` en el encabezado.
6. Reporta en el chat: cuántas tareas se agregaron, cuántas cambiaron, cuántas quedaron obsoletas.

Ver "todo lo pendiente" del repo no necesita archivo índice:

```bash
grep -rn "^- \[ \]" docs/tasks/
```

## Asignación de `_Agente:_`

- **subagente autónomo** — mecánico y acotado, criterio de éxito verificable sin discusión.
- **agente principal supervisado** — arquitectónico, o con riesgo de regresión que hay que ver pasar.
- **decisión humana** — requiere criterio de negocio, o toca algo que el spec/auditoría no resolvió.

## Asignación de `_Modelo:_`

Siempre el modelo **más barato** que pueda completar la tarea de forma confiable.

| Modelo | Cuándo | Ejemplos |
|---|---|---|
| `haiku` | Mecánico, sin ambigüedad | renombrar, mover archivos, extraer constantes y magic numbers, tipos obvios, formateo, boilerplate desde un patrón existente, actualizar imports, dead code, tests desde template claro |
| `sonnet` | Implementación estándar | lógica de negocio bien especificada, refactors dentro de un módulo, dividir funciones largas, integración con APIs documentadas, agregar manejo de errores o validación siguiendo un patrón, tests que requieren diseñar casos. **La mayoría.** |
| `opus` | Costo de error alto | seguridad (auth, inyección, secretos), cambios cross-cutting en varios módulos, migraciones de schema/datos, race conditions y concurrencia, trade-offs de diseño que el spec dejó abiertos |

Reglas prácticas:

- Si dudas entre dos, elige el más barato. Es más fácil escalar una tarea que falló que recuperar tokens quemados.
- Si una tarea mezcla trabajo trivial y complejo, divídela en subtareas con modelos distintos en vez de asignar el modelo caro a todo.

Al ejecutar, cambia con `/model haiku` (o el que indique la tarea) antes de trabajarla — o inclúyelo en el prompt del subagente.
