# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Jerarquía de autoridad — leer esto antes que nada

**Este archivo no es la fuente de verdad. Es un resumen derivado, y está por debajo de los
documentos de especificación.** El orden de autoridad es:

1. **[.specify/memory/constitution.md](.specify/memory/constitution.md)** — la constitución del
   proyecto. Manda sobre todo lo demás, incluida cualquier feature. Hoy en **v2.0.0**.
2. **[specs/](specs/)** — la especificación de la feature activa, hoy
   [specs/001-asistente-voz-eva/spec.md](specs/001-asistente-voz-eva/spec.md). Manda sobre este
   archivo, y cede ante la constitución.
3. **CLAUDE.md** (este archivo) — último. Solo orienta.

Reglas operativas que se derivan de esto:

- **Ante cualquier duda sobre qué construir o qué está permitido, la primera respuesta se busca en
  la constitución y en la spec, no acá.** Este archivo se lee para ubicarse rápido, no para decidir.
- **Si este archivo contradice a la constitución o a la spec, gana el documento de arriba y este
  archivo está mal.** No es una discrepancia a interpretar: es un defecto de CLAUDE.md que hay que
  corregir en el momento en que se detecta.
- **Nada se decide enmendando este archivo.** Cambiar una restricción real exige
  `/speckit-constitution` (con bump de versión y Sync Impact Report) o `/speckit-specify` /
  `/speckit-clarify` sobre la spec. Editar CLAUDE.md no cambia ninguna regla, solo el resumen.
- Todo lo que sigue en este archivo lleva la referencia a su fuente entre paréntesis: el número de
  principio para la constitución, o el identificador `FR-`/`NFR-`/`SC-`/`AMB-` para la spec. Si un
  párrafo de acá no puede señalar su fuente, no es una regla del proyecto.

Este archivo se mantiene sincronizado a mano. Cuando la constitución o la spec cambien, actualizarlo
es parte del mismo trabajo, no una tarea posterior.

## Máquina objetivo

Eva corre en esta laptop, y todos los presupuestos de recursos se definen contra este hardware:

| Componente | Especificación |
| --- | --- |
| Procesador | AMD Ryzen 5 3450U — 4 núcleos, 8 hilos, hasta 3.5 GHz |
| Gráficos | AMD Radeon Vega 8 integrados (comparten memoria con el sistema) |
| Memoria RAM | 8 GB |
| Almacenamiento | SSD de 256 GB |
| Pantalla | 14 pulgadas (HD o FHD según sub-versión) |

Consecuencias que ya están escritas en las reglas y no son negociables acá:

- La máquina se usa **al mismo tiempo** para desarrollo, con navegador y editor abiertos. Eva es un
  accesorio, nunca el proceso principal (Principio VI).
- Los gráficos integrados comparten la RAM del sistema: los 8 GB no están todos disponibles.
- 4 núcleos físicos significa que el presupuesto de "dejar un núcleo libre" (NFR-004) es una
  restricción apretada, no holgada.
- El modelo de lenguaje generativo queda fuera de la fase 1 (Principio IV y tabla de presupuestos),
  en parte por este hardware.

## Qué es Eva

Asistente de voz **local** para escritorio Linux (Omarchy: Arch + Hyprland). Interpreta lenguaje
natural en **español rioplatense** y ejecuta acciones sobre el sistema operativo. Corre como daemon
de usuario, sin conexión a internet.

La documentación del proyecto se escribe en español; la constitución y las specs siguen esa
convención.

## Estado actual del repositorio

**No hay código de aplicación todavía.** El repo contiene la constitución, la especificación de la
feature 001 y el andamiaje de Spec Kit. No existen `package.json`, `pyproject.toml`, Makefile, suite
de tests ni comandos de build/lint/test — y **no deben inventarse**: la elección de lenguaje,
runtime y herramientas es una decisión de la fase `plan`, no algo a improvisar (Principio XVII).

Estado de la feature 001: **Draft**. Bloqueada para `/speckit-plan` hasta cerrar AMB-01, AMB-02 y
AMB-03 en `/speckit-clarify`.

## Restricciones que condicionan toda decisión técnica

Trazadas a su fuente. La lista es un resumen, no un reemplazo: antes de planificar hay que leer la
constitución completa.

**De la constitución (v2.0.0)**

- **Registry cerrado (III, NO NEGOCIABLE)**: nunca se genera ni se ejecuta shell arbitrario. Se
  elige una herramienta de un registro declarado y se completan parámetros tipados, validados contra
  esquema antes de ejecutar. Herramienta desconocida o parámetro inválido → rechazo, nunca
  interpretación creativa. Es la frontera de seguridad; no se negocia por conveniencia.
- **Determinismo total en fase 1 (IV)**: la resolución de intención es **enteramente
  determinística**. No hay modelo de lenguaje generativo en esta fase. El fallback por modelo es un
  punto de extensión que se declara y **no** se implementa.
- **El núcleo no depende de audio (II)**: todo test de intención, ejecución y error corre sin
  micrófono. La voz es un adaptador de entrada, no el núcleo.
- **Dirección de dependencias (IX)**: `dominio` (intenciones, herramientas, validación) →
  `aplicación` (orquestación del turno, máquina de estados) → `infraestructura` (audio, modelos,
  IPC, Hyprland) → `interfaz` (CLI, notificaciones y la superficie mínima de estado). El dominio no
  importa audio, ni modelos, ni Hyprland.
- **Cero red en runtime (XI)**: cualquier dependencia de red descubierta es un defecto bloqueante.
- **Interfaz mínima de estado, y solo esa (X)**: en fase 1 se permite **una** superficie gráfica
  mínima, y únicamente si cumple las cinco condiciones acumulativas del principio: muestra solo
  estado/texto entendido/acción resuelta; no toma el foco del teclado; no tapa ni desplaza la
  ventana de trabajo; aparece solo cuando hay algo que comunicar y se va sola; y **el sistema sigue
  siendo completamente funcional con ella deshabilitada**. Si una condición deja de cumplirse, la
  superficie deja de estar permitida. Barra permanente, widgets, panel de configuración, historial
  navegable y dashboards siguen fuera de fase 1.
- **Presupuestos duros (VI, VII)**: <3% de un núcleo y <250 MB de RSS en reposo; **≤1.5 GB** con el
  stack completo cargado; STT ≤ `whisper small` cuantizado; **LLM fuera de la fase 1**; latencia por
  etapa declarada y medida. Toda excepción se documenta en el `plan.md` de la feature.
- **Falla ruidosa (XVI)** y **confirmación de acciones destructivas (XII)**: ante ambigüedad se
  pregunta o se rechaza; nunca se ejecuta la acción "más parecida". Cada herramienta declara si es
  destructiva; el default ante duda es que requiere confirmación explícita.
- **Toda herramienta del registry llega con tests (XIV)**: validación de parámetros + casos de
  rechazo (ambigüedad, herramienta inexistente, parámetro fuera de rango). Sin test no entra al
  registry.
- **Observabilidad por turno (XV)**: cada interacción deja registro estructurado de transcripción,
  intención, herramienta, resultado y latencia por etapa. Local (XI).
- **Configuración declarativa (XIII)**: navegador, terminal, aplicaciones, rutas y atajos se leen de
  configuración o del sistema. Un valor del entorno del usuario embebido en el código es un defecto.

**De la spec 001 — más estrictas que la constitución**

La spec aprieta tres presupuestos por encima del piso constitucional. **Para esta feature gobierna
el número de la spec**; el de la constitución queda como piso mínimo del proyecto:

| Dimensión | Constitución (piso) | Spec 001 (gobierna) |
| --- | --- | --- |
| CPU en reposo | < 3% de un núcleo | **< 1%** (NFR-001) |
| RSS en reposo | < 250 MB | **< 150 MB** (NFR-002) |
| RAM pico por turno | ≤ 1.5 GB | ≤ 1,5 GB (NFR-003) |

Un límite más estricto satisface el Principio VI sin necesitar excepción documentada.

**De la spec 001 — restricciones que la constitución no fija**

- **Latencia** (NFR-006, NFR-007): fin del habla → acción ejecutada en <2 s en el percentil 50 y
  <3,5 s en el percentil 95. Cambio de estado observable en <100 ms desde la activación (NFR-008).
- **Precisión** (NFR-014 a NFR-016): ≥90% de intención y parámetros correctos sobre el conjunto de
  frases de referencia; <1% de ejecución de una acción distinta de la pedida; 100% de rechazo sobre
  las frases marcadas como "debe rechazarse". **Ejecutar una acción incorrecta es más grave que no
  ejecutar ninguna.**
- **Concurrencia** (NFR-004): al menos un núcleo libre en todo momento; nunca saturar los 4.
- **Catálogo cerrado de acciones** (FR-014 a FR-023): abrir aplicación, abrir aplicación en espacio
  de trabajo, abrir URL, abrir URL en espacio de trabajo, búsqueda web, cambiar de espacio de
  trabajo, mover ventana activa, operar ventana activa (cerrar / pantalla completa / flotante),
  volumen y medios, y acciones propias declaradas en configuración.
- **Fuera de alcance de la fase 1**: wake word, LLM generativo, síntesis de voz, conversación
  multi-turno con memoria, consultas que devuelven información, control de aplicaciones más allá de
  abrirlas, múltiples usuarios, empaquetado y distribución.

**Decisiones abiertas — no asumir un comportamiento**

Tres ambigüedades siguen sin resolver y bloquean `/speckit-plan`. No inventar una respuesta:

- **AMB-01**: qué acciones concretas del catálogo se consideran destructivas.
- **AMB-02**: si la aplicación pedida ya está abierta, enfocar la existente o abrir otra instancia.
- **AMB-03**: cómo se distingue una búsqueda web de la apertura de un sitio cuando el usuario dice
  el nombre de un sitio.

Otras cuatro (AMB-04 a AMB-07) están resueltas con un supuesto documentado en la spec, no con una
decisión firme. Antes de apoyarse en cualquiera, leer su fundamento en la spec.

## Flujo de trabajo: Spec Kit

El proyecto usa Spec Kit con la integración `claude` y separador `-`, por lo que los comandos se
invocan como `/speckit-specify`, `/speckit-plan`, etc. (no `/speckit.specify`).

Orden de fases: `constitution` → `specify` → `clarify` → `checklist` → `plan` → `tasks` →
`implement`.

**El Principio XVII es la regla operativa más importante para vos**: durante `specify`, `clarify`,
`checklist`, `plan` y `tasks` **no se escribe código**, solo documentos. `implement` es la única
fase que toca código de aplicación.

### Estado de feature

Este proyecto **no deriva la feature activa del nombre de la rama de git**. La resuelve, en orden de
prioridad:

1. `$env:SPECIFY_FEATURE_DIRECTORY`
2. `.specify/feature.json` (clave `feature_directory`, persistida por `create-new-feature.ps1`)

Si ninguna existe, los scripts fallan con "Feature directory not found". Los artefactos de cada
feature viven en `specs/NNN-slug/`: `spec.md`, `plan.md`, `tasks.md`, `research.md`,
`data-model.md`, `quickstart.md`, `contracts/`, `checklists/`.

Feature activa hoy: `specs/001-asistente-voz-eva`.

### Scripts de soporte

Crear una feature nueva (numeración secuencial, genera el directorio y `spec.md` desde la plantilla,
y persiste `feature.json`):

```bash
pwsh .specify/scripts/powershell/create-new-feature.ps1 -Json -ShortName 'nombre-corto' 'Descripción de la feature'
```

Verificar precondiciones de la feature activa sin escribir nada:

```bash
pwsh .specify/scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly
```

`-PathsOnly` evita que la resolución de rutas persista `feature.json` y ensucie el árbol de trabajo.
`create-new-feature.ps1` acepta además `-DryRun`, `-AllowExistingBranch`, `-Number N` y `-Timestamp`.

Este proyecto está inicializado con `script: ps`: los scripts de soporte son PowerShell, no bash.

## Al agregar comandos de build/test

Cuando la fase `plan` defina el stack, actualizar este archivo con los comandos reales de build,
lint, test y **cómo correr un test individual** — es lo que más falta hace y hoy no existe.

## Conventional Commits

- Hacer conventional commits (fix, test, feature, bug) cada vez que hay que subir un cambio.
- No agregar coautoría a los commits.
