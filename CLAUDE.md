# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Estado actual del repositorio

**No hay código de aplicación todavía.** El repo contiene únicamente la constitución del
proyecto y el andamiaje de Spec Kit. No existen `package.json`, `pyproject.toml`, Makefile,
suite de tests ni comandos de build/lint/test — y **no deben inventarse**: la elección de
lenguaje, runtime y herramientas es una decisión de la fase `plan` de la primera feature, no
algo a improvisar.

Archivos que sí existen y hay que leer antes de trabajar:

- [.specify/memory/constitution.md](.specify/memory/constitution.md) — 17 principios
  vinculantes. Es la fuente de autoridad del proyecto.
- `.specify/templates/` — plantillas de spec, plan, tasks y checklist.
- `.specify/scripts/powershell/` — scripts de soporte del flujo (este proyecto está
  inicializado con `script: ps`, no bash).

## Qué es Eva

Asistente de voz **local** para escritorio Linux (Omarchy: Arch + Hyprland). Interpreta
lenguaje natural en **español rioplatense** y ejecuta acciones sobre el sistema operativo.
Corre como daemon de usuario en una laptop, sin conexión a internet.

La documentación del proyecto se escribe en español; la constitución y las specs siguen esa
convención.

## Restricciones que condicionan toda decisión técnica

Estas salen de la constitución y son las que más seguido se violan por inercia. Leer el
documento completo antes de planificar, pero como mínimo:

- **Registry cerrado (principio III)**: el LLM nunca genera ni ejecuta shell arbitrario. Elige
  una herramienta de un registro declarado y completa parámetros tipados, validados contra
  esquema antes de ejecutar. Herramienta desconocida o parámetro inválido → rechazo, nunca
  interpretación creativa. Esta es la frontera de seguridad; no se negocia por conveniencia.
- **Determinismo primero (IV)**: si una intención se puede expresar como regla, se expresa como
  regla. El LLM es fallback.
- **El núcleo no depende de audio (II)**: todo test de intención, ejecución y error corre sin
  micrófono. La voz es un adaptador de entrada.
- **Dirección de dependencias (IX)**: `dominio` (intenciones, herramientas, validación) →
  `aplicación` (orquestación del turno, máquina de estados) → `infraestructura` (audio,
  modelos, IPC, Hyprland) → `interfaz` (CLI, notificaciones). El dominio no importa audio, ni
  modelos, ni Hyprland.
- **Cero red en runtime (XI)**: cualquier dependencia de red descubierta es un defecto
  bloqueante.
- **Sin GUI en fase 1 (X)**: nada de overlays, layer-shell, widgets, barra de estado ni OSD.
  Se deja el punto de extensión declarado, sin implementar. El feedback es notificación del
  sistema, stdout y código de retorno.
- **Presupuestos duros (VI, VII)**: <3% de un core y <250 MB RSS en reposo; ≤3 GB con el stack
  completo; STT ≤ `whisper small` cuantizado; LLM ≤ 4B parámetros cuantizado; latencia por
  etapa declarada y medida. Toda excepción va documentada en el `plan.md` de la feature.
- **Falla ruidosa (XVI)** y **confirmación de acciones destructivas (XII)**: ante ambigüedad se
  pregunta o se rechaza; nunca se ejecuta la acción "más parecida". Cada herramienta declara si
  es destructiva; el default ante duda es que requiere confirmación explícita.
- **Toda herramienta del registry llega con tests (XIV)**: validación de parámetros + casos de
  rechazo (ambigüedad, herramienta inexistente, parámetro fuera de rango). Sin test no entra al
  registry.

## Flujo de trabajo: Spec Kit

El proyecto usa Spec Kit con la integración `claude` y separador `-`, por lo que los comandos
se invocan como `/speckit-specify`, `/speckit-plan`, etc. (no `/speckit.specify`).

Orden de fases: `constitution` → `specify` → `clarify` → `checklist` → `plan` → `tasks` →
`implement`.

**El principio XVII es la regla operativa más importante para vos**: durante `specify`,
`clarify`, `checklist`, `plan` y `tasks` **no se escribe código**, solo documentos.
`implement` es la única fase que toca código de aplicación.

### Estado de feature

Este proyecto **no deriva la feature activa del nombre de la rama de git**. La resuelve, en
orden de prioridad:

1. `$env:SPECIFY_FEATURE_DIRECTORY`
2. `.specify/feature.json` (clave `feature_directory`, persistida por `create-new-feature.ps1`)

Si ninguna existe, los scripts fallan con "Feature directory not found". Los artefactos de cada
feature viven en `specs/NNN-slug/`: `spec.md`, `plan.md`, `tasks.md`, `research.md`,
`data-model.md`, `quickstart.md`, `contracts/`.

### Scripts de soporte

Crear una feature nueva (numeración secuencial, genera el directorio y `spec.md` desde la
plantilla, y persiste `feature.json`):

```bash
pwsh .specify/scripts/powershell/create-new-feature.ps1 -Json -ShortName 'nombre-corto' 'Descripción de la feature'
```

Verificar precondiciones de la feature activa sin escribir nada:

```bash
pwsh .specify/scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly
```

`-PathsOnly` evita que la resolución de rutas persista `feature.json` y ensucie el árbol de
trabajo. `create-new-feature.ps1` acepta además `-DryRun`, `-AllowExistingBranch`, `-Number N`
y `-Timestamp`.

## Al agregar comandos de build/test

Cuando la primera feature defina el stack, actualizá este archivo con los comandos reales de
build, lint, test y **cómo correr un test individual** — es lo que más falta hace y hoy no
existe. Mantené la sección de restricciones sincronizada con la constitución si esta se
enmienda (la constitución versiona con semver; ver su sección `Governance`).

## Conventional Commits
- Hacer conventional commits (fix, test, feature, bug) cada vez que hay que subir un cambio. 
- No agregar coautoria a los commits. 
