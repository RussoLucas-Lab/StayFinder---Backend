# Flujo de trabajo

Se trabaja con **Spec-Driven Development** usando OpenSpec, con Claude Code como agente, Engram como memoria entre sesiones y la skill `karpathy-guidelines` para el estilo de trabajo ([ADR-0003](adr/0003-sdd-openspec-engram.md)).

## Setup (una sola vez)

### Requisitos

- Git y Docker. Python 3.12 o superior en el host es opcional (solo para el editor); el código corre en contenedores ([ADR-0010](adr/0010-ejecucion-en-contenedores.md)).
- Node.js 20.19 o superior (lo pide OpenSpec).

### 1. Repositorio

```bash
git init
```

### 2. OpenSpec

```bash
npm install -g @fission-ai/openspec@latest
openspec init        # elegir Claude Code como herramienta
```

`openspec init` genera su propio `openspec/config.yaml`. Reemplazalo por el de este repositorio (o copiá su `context` y sus `rules`). Si `init` agrega un bloque administrado a `CLAUDE.md`, conservá ese bloque junto con el contenido existente.

### 3. Engram

1. Instalar el binario según la [guía de instalación](https://github.com/Gentleman-Programming/engram/blob/main/docs/INSTALLATION.md) (en Windows no se usa Homebrew).
2. Conectarlo a Claude Code y reiniciar Claude Code:

   ```bash
   engram setup claude-code
   ```

3. Fijar el nombre del proyecto **después** de `git init` (inicializar Git puede cambiar el nombre que Engram detecta):

   ```bash
   engram init stayfinder-backend
   ```

### 4. Skill de Karpathy

En Claude Code:

```text
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
```

El repositorio se muestra hoy como `multica-ai/andrej-karpathy-skills`, pero sus instrucciones de instalación apuntan a `forrestchang`. Confirmá que ambos sean el mismo repositorio antes de instalar.

## Ciclo de una sesión

**Al empezar**

1. El agente recupera el contexto de Engram (`mem_context`).
2. Revisa `docs/change-map.md` y si hay un change activo en `openspec/changes/`.
3. Resume en dos o tres líneas en qué quedó el trabajo y qué sigue, y espera confirmación.

**Al terminar**

1. Las casillas de `tasks.md` del change activo reflejan el estado real.
2. El agente guarda un resumen con `mem_session_summary`: objetivo, descubrimientos, lo hecho, próximos pasos y archivos tocados.

## Ciclo de un change

Comandos del perfil `core` de OpenSpec (el conjunto exacto depende del perfil: `openspec config profile`).

| Paso | Comando | Resultado | Revisión humana |
|---|---|---|---|
| 1. Elegir | — | Próximo change del change map | Confirmar que sigue siendo el correcto |
| 2. Explorar (opcional) | `/opsx:explore` | Análisis sin archivos | Decidir el enfoque |
| 3. Proponer | `/opsx:propose <change-id>` | `proposal.md`, specs, `design.md`, `tasks.md` | **Leer y ajustar cada artefacto antes de seguir** |
| 4. Validar | `openspec validate <change-id> --strict` | Errores de formato | — |
| 5. Implementar | `/opsx:apply` | Código y tests, tarea por tarea | Revisar cada grupo de tareas |
| 6. Archivar | `/opsx:archive` | Specs fusionadas en `openspec/specs/` y change en `archive/` | Verificar que la spec final sea correcta |
| 7. Cerrar | — | Estado actualizado en el change map | — |

Reglas:

- Un change activo por vez, en el orden del change map, salvo indicación contraria.
- Los arreglos que restauran un comportamiento ya especificado, los typos y el formato se hacen directo, sin change.
- Un change que no cambia comportamiento (refactor, tooling, docs) declara `skip_specs: true` en su `.openspec.yaml`. No se inventan requisitos para pasar la validación.
- Si al implementar aparece una decisión transversal, se frena y se propone un ADR.

### Definición de terminado de un change

- [ ] Todas las tareas de `tasks.md` marcadas.
- [ ] Cada escenario de las specs del change tiene al menos un test, y todos pasan.
- [ ] `openspec validate <change-id> --strict` sin errores.
- [ ] Docs actualizados si el change alteró dominio, arquitectura o convenciones.
- [ ] Change archivado y change map actualizado.

## Formato de specs

Los artefactos se escriben en español, pero la estructura que OpenSpec analiza va en inglés y exacta:

- Encabezados de delta: `## ADDED Requirements`, `## MODIFIED Requirements`, `## REMOVED Requirements`, `## RENAMED Requirements`.
- `## Purpose` solo en capacidades nuevas (una o dos oraciones, al menos 50 caracteres).
- Requisitos: `### Requirement: <nombre>`, con `SHALL` o `MUST` en el texto. Uno por comportamiento.
- Escenarios: `#### Scenario: <nombre>` (exactamente cuatro `#`), con pasos `- **GIVEN**`, `- **WHEN**`, `- **THEN**`, `- **AND**`.
- `MODIFIED` repite el requisito completo, no solo la parte que cambia. `REMOVED` incluye **Reason** y **Migration**.
- Ruta de capacidad: `<modulo>/<capacidad>` en kebab-case (por ejemplo `reservations/holds`), o solo `<modulo>` si es chica. Si la versión instalada de OpenSpec no acepta rutas anidadas, usar `<modulo>-<capacidad>` y actualizar esta regla.

Ejemplo:

```markdown
## ADDED Requirements

### Requirement: Retención atómica de inventario
El sistema SHALL retener el inventario de todas las noches de la estadía en la misma transacción que crea el hold, o no retener ninguna.

#### Scenario: Último cupo bajo concurrencia
- **GIVEN** un tipo de habitación con 1 unidad disponible para todas las noches pedidas
- **WHEN** 50 viajeros solicitan un hold para esas noches al mismo tiempo
- **THEN** exactamente 1 hold se crea
- **AND** las otras 49 solicitudes reciben `409` con código `INVENTORY_UNAVAILABLE`
```

## Memoria con Engram

Engram da continuidad entre sesiones. No es fuente de verdad.

| Guardar | No guardar |
|---|---|
| Decisiones tomadas en la sesión (y proponer un ADR si son transversales) | Lo que ya está en docs o specs: se guarda la ruta, no el contenido |
| Descubrimientos y gotchas (comportamiento raro de una librería, un lock que no funcionó) | Secretos, tokens, credenciales, datos personales |
| Bugs resueltos, con su causa | Salidas completas de comandos o de tests |
| Estado al cerrar la sesión | Opiniones sin una decisión detrás |

- Títulos cortos y buscables. Para temas que evolucionan, usar un `topic_key` estable (por ejemplo `stayfinder/inventory-locking`) en lugar de crear memorias nuevas cada vez.
- **Promoción:** si una decisión guardada en Engram tiene que sobrevivir, se escribe como ADR o como spec. Nada importante vive solo en Engram.
