# Implementation Plan: Eva — Asistente de Voz Local para Escritorio

**Branch**: `001-asistente-voz-eva` | **Date**: 2026-08-09 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-asistente-voz-eva/spec.md`

---

## ⚠ Tres bloqueos que este plan no puede resolver por sí solo

**1. Las mediciones en hardware objetivo no se tomaron.** La sesión de planificación corre en
Windows; el objetivo es una laptop Arch + Hyprland. Toda medición pedida en el punto 5 del encargo
—memoria residente al cargar, tiempo de carga, tiempo de transcripción en español rioplatense,
hilos usados— **está sin tomar y no se sustituye por benchmarks publicados**.
[research.md](research.md) define el protocolo de medición, la tabla vacía a completar y la **regla
de decisión** que selecciona el motor automáticamente según los números. La elección de motor de
transcripción es la única decisión del plan que queda **abierta y bloqueada por medición**.

**2. Discrepancia en el conteo de hilos.** El encargo dice "4 núcleos, 4 hilos, sin SMT". El
Ryzen 5 3450U es un Picasso/Zen+ que sale de fábrica con SMT (4C/8T), y [CLAUDE.md](../../CLAUDE.md)
—escrito con datos tuyos— dice "4 núcleos, 8 hilos". O tenés SMT deshabilitado en BIOS, o hay un
error en uno de los dos lados. **Este plan asume 4 hilos**, que es la dirección conservadora: si
resultan 8, todos los presupuestos tienen más margen, nunca menos. Verificar con `lscpu` antes de
`/speckit-tasks` (ver [quickstart.md](quickstart.md), escenario Q0).

**3. NFR-004 de la spec es internamente contradictorio y además choca con el encargo.** Dice
literalmente: *"MUST NOT superar el 75% de la capacidad total de CPU —equivalente a 3 de los 4
núcleos—"* y en la oración siguiente *"Ningún núcleo MUST quedar por encima del 10% de utilización
atribuible al asistente"*. Las dos no pueden cumplirse a la vez. Además el encargo dice que **hay
como máximo un núcleo ocioso**, con lo cual 75% es un techo que degradaría el trabajo del usuario.
Este plan presupuesta contra **150% de CPU (1,5 de 4 núcleos)**, aplicado por `CPUQuota=150%` en el
slice de systemd, y lo registra en *Complexity Tracking* como desviación que **requiere enmienda de
spec** antes de `/speckit-implement`. El plan no edita la spec: eso es trabajo de
`/speckit-specify` o `/speckit-clarify`.

---

## Summary

Eva se implementa como un **daemon Java 21 residente** que orquesta el turno y contiene todo el
dominio, más **procesos auxiliares no-Java** para las tres capacidades donde la JVM es inviable o
claramente peor: captura de audio (PipeWire), transcripción (motor STT nativo) y superficie de
estado (Wayland layer-shell). Los auxiliares se comunican por sockets de dominio Unix con un
protocolo de líneas JSON, y ninguno es dependencia del dominio: cada uno entra por un puerto
declarado en `eva-dominio` (Principio VIII).

La decisión estructural que hace viable el presupuesto de latencia es que **el modelo de
transcripción se carga al activarse el turno, en paralelo con el habla del usuario**. Como
NFR-006/NFR-007 se miden desde el *fin* del habla, el tiempo de carga queda absorbido por la
duración del enunciado y no consume presupuesto. Eso permite mantener el STT fuera de residencia y
respetar los 150 MB de reposo (NFR-002) sin pagarlo en latencia.

---

## Technical Context

**Language/Version**: Java 21 (LTS) para dominio, aplicación e infraestructura. Rust (edición 2021)
para dos binarios auxiliares. Sin Python en el camino crítico — ver research.md, D-09.

**Primary Dependencies**: JDK 21 (`java.net.UnixDomainSocketAddress`, disponible desde 16, evita
JNI para IPC) · Jackson o `java.util.spi` para JSON — decidido en research.md D-12 · JUnit 5 ·
ArchUnit (gate del Principio IX) · Gradle 8 con toolchains. Motor STT: **abierto**, ver D-05.

**Storage**: Sistema de archivos. Configuración en `$XDG_CONFIG_HOME/eva/`, bitácora en
`$XDG_STATE_HOME/eva/bitacora/`, modelos en `$XDG_DATA_HOME/eva/models/`. Sin base de datos:
el volumen es de decenas de turnos por día (FR-046).

**Testing**: JUnit 5 para dominio y aplicación. Tests de pipeline con `PcmSource` de archivo WAV en
lugar de micrófono (FR-041). Conjunto de 300 frases de referencia versionado en el repo, ejecutado
por test parametrizado (FR-063 a FR-067). Arneses de presupuesto: RSS por `smaps_rollup`, latencia
por los timestamps de la bitácora.

**Target Platform**: Omarchy (Arch Linux), Hyprland sobre Wayland, PipeWire. AMD Ryzen 5 3450U
(Zen+, 4 núcleos; conteo de hilos a verificar), Radeon Vega integrada **sin uso para inferencia**,
8 GB DDR4 compartidos con la iGPU. Memoria realmente disponible ≈ 3 GB; núcleos realmente libres ≈ 1.

**Project Type**: Daemon de usuario de escritorio, multiproceso, cien por ciento local.

**Performance Goals**: fin del habla → acción ejecutada < 2 s p50 (NFR-006) y < 3,5 s p95
(NFR-007), sobre turnos sin confirmación. Cambio de estado observable < 100 ms (NFR-008).

**Constraints**: < 1% de un núcleo y < 150 MB RSS agregado en reposo (NFR-001, NFR-002) · ≤ 1,5 GB
de pico por turno (NFR-003) · ≤ 150% de CPU durante el turno (ver bloqueo 3) · cero conexiones de
red del asistente (NFR-011) · ningún estado sin salida acotada (NFR-023).

**Scale/Scope**: un usuario, un turno concurrente, catálogo de ~12 acciones base más acciones
propias declaradas, conjunto de referencia de 300 entradas.

---

## Constitution Check

*GATE: debe pasar antes de Phase 0 y volver a evaluarse tras Phase 1.*

Evaluado contra [constitution.md](../../.specify/memory/constitution.md) **v2.0.0**.

| Principio | Cómo lo satisface el diseño | Estado |
|---|---|---|
| I. La especificación manda | Cada decisión de este plan referencia su FR/NFR. La tabla de *Trazabilidad de decisiones* cierra el círculo. | ✅ |
| II. Núcleo desacoplado de la voz | `PcmSource` y `Transcriptor` son puertos; el orquestador acepta texto por `evactl` sin pasar por ninguno de los dos (FR-040, FR-041). | ✅ |
| III. Registry cerrado | El catálogo es data validada contra JSON Schema al cargar. No hay `Runtime.exec` con cadena construida: el ejecutor recibe una acción tipada y despacha por un `switch` exhaustivo sobre ids conocidos. Las acciones propias ejecutan un `argv[]` **declarado**, nunca compuesto desde la transcripción. | ✅ |
| IV. Determinismo antes que modelo | Resolución de intención por gramática de slots compilada desde configuración. Sin LLM en fase 1. | ✅ |
| V. Degradación elegante | Superficie de estado y STT son opcionales: si mueren, el daemon degrada a los canales de FR-033 y sigue aceptando texto. Ninguno es condición de arranque. | ✅ |
| VI. Presupuesto de recursos | STT bajo demanda; JVM con SerialGC y heap acotado; superficie en Rust en vez de GTK. Presupuesto de reposo desglosado abajo. | ⚠ Gated |
| VII. Latencia funcional | Presupuesto por etapa con suma cerrada, más abajo. Medición por bitácora. | ⚠ Gated |
| VIII. Puertos y adaptadores | Siete puertos declarados en `eva-dominio`, cero implementaciones. | ✅ |
| IX. Separación de responsabilidades | Cuatro módulos Gradle con dependencias dirigidas hacia adentro, verificado por ArchUnit en CI. | ✅ |
| X. Superficie mínima de estado | Un solo proceso de superficie, layer-shell sin foco, apagable por configuración. Las cinco condiciones mapeadas a FR-034…FR-039. | ✅ |
| XI. Cien por ciento local | Ningún componente abre sockets de red. Los modelos se descargan una vez, fuera de runtime, por un script de instalación. | ✅ |
| XII. Confirmación de destructivas | Estado `esperando confirmación` en la máquina de estados, cuatro entradas declaradas (FR-057). | ✅ |
| XIII. Configuración declarativa | Aplicaciones, sitios, alias, gramática, atajos y acciones propias son datos. Descubrimiento por XDG, nada hardcodeado. | ✅ |
| XIV. Tests obligatorios | Toda acción del catálogo entra con test de validación de parámetros y casos de rechazo; gate en CI. | ✅ |
| XV. Observabilidad por turno | Bitácora JSONL con un registro por turno y timing por etapa. | ✅ |
| XVI. Falla ruidosa | FR-056 (ambigüedad) y FR-009 implementados como rechazo explícito; sin desempate. | ✅ |
| XVII. Especificación antes que código | Este comando produce solo documentos. No se creó ningún módulo ni archivo fuente. | ✅ |

**Dos compuertas en amarillo (VI y VII)**: no son violaciones de diseño, son afirmaciones que
dependen de mediciones que todavía no existen. Se resuelven completando la tabla de research.md
D-05 en el hardware objetivo. Si ninguna alternativa entra, aplica el plan de contingencia de D-05.

---

## Arquitectura

### Estructura de procesos

Cuatro procesos. Solo dos son residentes.

| Proceso | Lenguaje | Residencia | RSS estimado (a medir) | Justificación de la residencia |
|---|---|---|---|---|
| `eva-daemon` | Java 21 | **Residente** | 60–80 MB | Es el núcleo: mantiene el catálogo compilado, la máquina de estados y el socket de control. Arrancarlo por turno costaría 300–800 ms de JVM, incompatible con NFR-008. |
| `eva-overlay` | Rust | **Residente, oculto** | 8–15 MB | Debe pintar en < 100 ms desde la activación (NFR-008). Un arranque en frío de cualquier toolkit gráfico supera ese presupuesto, así que el proceso vive y solo muestra u oculta la superficie. |
| `pw-record` | — (PipeWire) | **Bajo demanda** | ~5 MB mientras corre | Solo existe durante la captura. Fuera del turno no hay proceso de audio, lo que satisface FR-002 por construcción, no por disciplina. |
| `eva-stt` | motor a decidir | **Bajo demanda** | 60–250 MB mientras corre | Es el mayor consumidor del sistema. Se lanza en la activación y se descarga por inactividad. Ver *Política de carga* abajo. |

**Presupuesto de reposo (NFR-002, 150 MB)**: `eva-daemon` 60–80 + `eva-overlay` 8–15 = **68–95 MB**,
con 55–82 MB de margen. Los dos procesos bajo demanda aportan 0 en reposo. Este es el argumento
central por el que la superficie **no** se implementa en GTK4 (≈50–60 MB residentes consumirían la
mitad del presupuesto) ni el STT queda residente.

**Presupuesto de pico (NFR-003, 1,5 GB)**: 95 MB de residentes + hasta 250 MB de STT + ~5 MB de
captura = **≈350 MB**, muy por debajo. El techo de 1,5 GB solo se acercaría con whisper `small` sin
cuantizar, que la constitución ya prohíbe.

**Política de carga del STT**: se lanza al **inicio del turno**, no al final del habla. Como
NFR-006/NFR-007 se miden desde el fin del habla, la carga del modelo transcurre mientras el usuario
habla y **no consume presupuesto de latencia**. Se descarga tras **120 s** sin turnos. Un turno que
llega con el motor ya caliente ahorra la carga; uno que llega en frío la absorbe con el enunciado.
Enunciados muy cortos (< 1 s) son el peor caso y quedan cubiertos por la holgura del presupuesto.

### Comunicación entre procesos

Sockets de dominio Unix en `$XDG_RUNTIME_DIR/eva/`, con **JSON Lines** (un objeto JSON por línea,
terminado en `\n`). Java 21 habla AF_UNIX de forma nativa vía `UnixDomainSocketAddress`, sin JNI.

| Socket | Sentido | Contenido |
|---|---|---|
| `control.sock` | `evactl` → daemon | `activate`, `text`, `confirm`, `cancel`, `status`, `reload` |
| `overlay.sock` | daemon ↔ overlay | daemon → estado a pintar; overlay → eventos de tecla de confirmación |
| stdout/stdin | daemon ↔ `pw-record` | PCM crudo s16le 16 kHz mono |
| stdout/stdin | daemon ↔ `eva-stt` | daemon → PCM o ruta de archivo; motor → texto parcial y final |

El esquema completo está en [contracts/ipc-protocol.md](contracts/ipc-protocol.md).

### Módulos

```text
eva/
├── settings.gradle.kts
├── eva-dominio/            # Java puro, cero dependencias externas
│   └── src/main/java/ar/eva/dominio/
│       ├── modelo/         # Turno, Accion, Parametro, Intencion, EstadoAsistente, ClaseError
│       ├── puerto/         # PcmSource, DetectorFinHabla, Transcriptor, ResolutorIntencion,
│       │                   # ValidadorParametros, Ejecutor, Notificador, Bitacora, Reloj
│       └── regla/          # Normalizador, GramaticaSlots, ResolutorAmbiguedad
├── eva-aplicacion/         # depende solo de eva-dominio
│   └── OrquestadorTurno, MaquinaEstados, PoliticaConfirmacion, PoliticaConcurrencia
├── eva-infraestructura/    # depende de dominio; implementa los puertos
│   ├── audio/              # PwRecordPcmSource
│   ├── stt/                # adaptador del motor elegido (D-05)
│   ├── vad/                # VadEnergiaZcr (por defecto), VadSileroOnnx (opcional)
│   ├── hyprland/           # HyprlandIpcEjecutor, DescubridorAplicaciones (XDG)
│   ├── config/             # CargadorToml, ValidadorEsquema, CompiladorCatalogo
│   ├── bitacora/           # BitacoraJsonl
│   └── notificacion/       # NotificadorLibnotify, NotificadorStdout
├── eva-daemon/             # composition root: cablea puertos con adaptadores
└── helpers/                # crate Rust, dos binarios
    ├── src/bin/evactl.rs   # cliente de control y entrada de texto
    └── src/bin/overlay.rs  # superficie de estado layer-shell
```

**Structure Decision**: multi-módulo Gradle con la regla de dependencias del Principio IX
verificada por ArchUnit en CI. `eva-dominio` no declara ninguna dependencia y ArchUnit prohíbe que
importe `javax.sound`, `java.net`, `java.nio.file` o cualquier paquete de `eva.infraestructura`.
El único módulo que conoce a todos es `eva-daemon`, que solo cablea.

---

## Presupuesto de latencia por etapa

Reparto de NFR-006 (2 s p50) y NFR-007 (3,5 s p95). El cronómetro arranca en el **fin acústico del
habla**, no en el fin de la detección. Dos escenarios según el motor que gane en D-05.

| Etapa | p50 (ms) | p95 (ms) | Cómo se mide |
|---|---|---|---|
| Confirmación de fin de habla (hangover del VAD) | 250 | 350 | Marca de tiempo del último frame con voz vs. emisión del evento `fin_habla` |
| Finalización de la transcripción — **vía streaming** | 150 | 400 | `t_texto_final − t_fin_habla` en la bitácora |
| Finalización de la transcripción — **vía por lotes** | 900 | 2000 | ídem |
| Normalización + resolución de intención | 5 | 20 | Instrumentación en `OrquestadorTurno` |
| Validación de parámetros | 1 | 5 | ídem |
| Despacho al gestor de ventanas | 20 | 90 | `t_respuesta_socket − t_envio` |
| Feedback (notificación + señal sonora, asíncrono) | 15 | 40 | No bloquea; se mide para la bitácora |
| **Total vía streaming** | **441** | **905** | Holgura: 1559 ms p50 / 2595 ms p95 |
| **Total vía por lotes** | **1191** | **2505** | Holgura: 809 ms p50 / 995 ms p95 |

Ambas vías cierran contra NFR-006/NFR-007, pero **la vía por lotes deja menos de 1 s de margen en
p95**, y ese margen es lo primero que se come la carga del sistema bajo la carga de referencia. Es
la razón cuantitativa por la que D-05 prioriza un motor con decodificación incremental.

Turnos con confirmación: NFR-024 los mide en dos tramos y excluye el tiempo humano. El segundo
tramo —confirmación → ejecución— presupuesta 20 ms p50 / 90 ms p95, holgadamente bajo su techo de
500 ms.

---

## Decisiones tomadas

Resumen; el análisis de alternativas y la justificación completa están en
[research.md](research.md), con el identificador `D-nn`.

| # | Decisión | Elección | Motivada por |
|---|---|---|---|
| D-01 | Estructura de procesos | 4 procesos, 2 residentes | NFR-002, NFR-008 |
| D-02 | IPC | Sockets Unix + JSON Lines | NFR-011, Principio VIII |
| D-03 | Configuración de la JVM | SerialGC, heap 16–96 MB, C1, AppCDS. **HotSpot, no imagen nativa** | NFR-002, NFR-004 |
| D-04 | Captura de audio | `pw-record` como proceso hijo, PCM por stdout | FR-002, FR-051, EC-12 |
| D-05 | Transcripción | **ABIERTA — bloqueada por medición.** Tres candidatos con regla de decisión | NFR-006, NFR-002 |
| D-06 | Detección de fin de habla | Energía + tasa de cruces por cero en Java puro; Silero-ONNX como puerto alternativo | NFR-004, NFR-020 |
| D-07 | Resolución de intención | Gramática de slots compilada desde configuración | FR-008…FR-013, FR-056 |
| D-08 | Registry de acciones | TOML de usuario validado contra JSON Schema al cargar | FR-042, FR-045, Principio III |
| D-09 | Ejecución sobre el escritorio | Socket IPC de Hyprland directo, **sin `hyprctl`** | FR-014…FR-022, NFR-006 |
| D-10 | Superficie de estado | Binario Rust con `wlr-layer-shell`; `eww` como plan B | FR-034…FR-039, NFR-002 |
| D-11 | Atajo global y ciclo de vida | `bind` de Hyprland → `evactl`; unidad de usuario de systemd | FR-001, FR-049 |
| D-12 | Configuración y bitácora | TOML en XDG; bitácora JSONL con rotación diaria | FR-042…FR-048 |
| D-13 | Testing | JUnit + `PcmSource` de archivo + conjunto de 300 frases versionado | FR-041, FR-063…FR-067 |

---

## Trazabilidad de decisiones

Cada decisión técnica apunta al requisito que la obliga. Sin esta columna, la decisión es
preferencia, no ingeniería (Principio I).

| Decisión | Requisitos que la motivan |
|---|---|
| STT bajo demanda con carga en la activación | NFR-002 (150 MB reposo) + NFR-006 (2 s desde fin de habla) |
| Superficie en Rust y no en GTK | NFR-002; una superficie GTK4 residente consumiría ~40% del presupuesto |
| Superficie residente y oculta, no bajo demanda | NFR-008 (100 ms) — el arranque en frío no entra |
| Socket de Hyprland en vez de `hyprctl` | NFR-006 (ahorra ~15 ms/invocación) + FR-014 (leer `clients` para saber si ya está abierta) |
| `evactl` en Rust y no CLI de JVM | NFR-008 — un arranque de JVM por activación supera los 100 ms |
| SerialGC y heap chico | NFR-002 + NFR-004 (no competir por el único núcleo libre) |
| VAD en Java sin modelo | NFR-002 (evita ~20 MB residentes de ONNX Runtime) + NFR-004 |
| `pw-record` como hijo y no Java Sound | FR-002 (sin proceso, no hay captura posible), EC-12 (PipeWire reencamina solo) |
| Gramática como datos y no como código | Principio XIII + FR-042, FR-043 |
| Recolección de **todos** los candidatos antes de decidir | FR-056 — no se puede rechazar por ambigüedad si el matcher corta en el primer acierto |
| Bitácora JSONL con timing por etapa | FR-046, FR-047, NFR-009 |

---

## Riesgos

| # | Riesgo | Probabilidad | Impacto | Mitigación concreta |
|---|---|---|---|---|
| R-01 | **La transcripción no entra en el presupuesto de latencia en este procesador.** | Media | Alto — invalida NFR-006/007 | Priorizar motor con decodificación incremental, que finaliza en ~150 ms tras el fin del habla en vez de procesar el enunciado entero. Si los tres candidatos fallan, aplica la escalera de contingencia de D-05: (1) reducir tamaño de modelo, (2) acotar el vocabulario con gramática cerrada del catálogo, (3) recién entonces relajar el umbral de latencia. **La calidad de la transcripción se degrada antes que el umbral**, porque un fallo de transcripción se manifiesta como rechazo (FR-009), mientras que un umbral relajado degrada el producto de forma permanente. |
| R-02 | **La suma de residentes excede los 150 MB con navegador y editor abiertos.** | Media | Alto — invalida NFR-002 | El presupuesto estimado (68–95 MB) deja 55–82 MB de margen. Si la JVM no baja de 100 MB con la configuración de D-03, la escalera es: (1) AppCDS más agresivo y recorte de módulos con `jlink`, (2) `-XX:+AutoCreateSharedArchive`, (3) recién entonces GraalVM native-image, aceptando el costo de configuración de reflexión y JNI. Gate automatizado en CI: el test de presupuesto falla el build si `smaps_rollup` supera el techo. |
| R-03 | **La solución de ventana superpuesta resulta inmadura o frágil.** | Media | Medio — degrada US4, no el núcleo | El diseño ya asume que puede fallar: FR-039 exige que el sistema funcione con la superficie apagada, y el Principio V prohíbe que sea condición de arranque. Si el binario Rust resulta caro o inestable, el plan B es `eww` —ya empaquetado en Arch, layer-shell probado— con el mismo contrato de socket, de modo que el cambio no toca el daemon. Plan C: solo notificaciones (FR-033), perdiendo US4 pero no la feature. |
| R-04 | **La resolución determinística no cubre las variaciones naturales del habla.** | **Alta** | Medio — degrada NFR-014 (90%) | Es el riesgo más probable del proyecto y el conjunto de 300 frases existe justamente para medirlo temprano. Mitigación en tres tiempos: (1) el conjunto se escribe **antes** que el resolutor, así el objetivo es medible desde el día uno; (2) las reglas son datos, con lo cual una variación no cubierta se arregla editando configuración, no recompilando; (3) el modo de fallo es seguro por diseño — una frase no cubierta cae en rechazo (FR-009), nunca en acción equivocada, que es lo que NFR-015 protege con el umbral de <1%. |
| R-05 | El conteo de hilos real es 8 y no 4, o viceversa. | Media | Bajo | Se planificó contra 4. Verificación con `lscpu` en Q0 antes de `/speckit-tasks`. |
| R-06 | El modelo cuantizado degrada más de lo esperado con nombres propios en inglés dentro de frases en español. | Media | Medio | FR-064 obliga a que el conjunto de referencia incluya al menos una entrada con nombre en inglés por acción, así el problema aparece en la medición y no en producción. Mitigación: declarar variantes fonéticas como alias en configuración (FR-043). |

---

## Complexity Tracking

> Desviaciones respecto de la especificación o de la constitución que requieren acción explícita.

| Desviación | Por qué es necesaria | Alternativa más simple, y por qué se rechaza |
|---|---|---|
| **Presupuesto de CPU en 150% en vez del 75% de NFR-004** | NFR-004 es internamente contradictorio (75% del total vs. ningún núcleo sobre 10%) y el encargo declara que hay **un solo núcleo ocioso**. Un techo de 3 núcleos degradaría el trabajo del usuario, que es lo que el Principio VI protege. | Cumplir NFR-004 al pie de la letra es imposible: las dos cláusulas se excluyen. **Requiere enmienda de spec** vía `/speckit-clarify` antes de `/speckit-implement`. |
| **Cuatro procesos en vez de uno** | Ninguna JVM puede pintar una ventana `wlr-layer-shell` sin foco, y mantener el STT residente rompe NFR-002. | Un proceso único fue rechazado por las dos razones anteriores; ambas son restricciones de plataforma, no preferencias. |
| **Dos lenguajes (Java + Rust)** | Los toolkits gráficos de la JVM no soportan layer-shell, y un cliente de control en JVM no entra en los 100 ms de NFR-008. | Python + PyGObject fue evaluado y rechazado en D-10: ~45 MB residentes contra ~10 MB, sobre un presupuesto de 150 MB. |
| **D-05 sin resolver al cerrar el plan** | Requiere medición en hardware que esta sesión no tiene. | Elegir a ciegas contra benchmarks publicados fue rechazado: el encargo lo prohíbe explícitamente y Zen+ sin AVX-512 no se parece a las máquinas de los benchmarks habituales. |

---

## Artefactos generados

| Archivo | Contenido |
|---|---|
| [research.md](research.md) | Las 13 decisiones con alternativas evaluadas, protocolo de medición de D-05 y su regla de decisión |
| [data-model.md](data-model.md) | Modelo del dominio, entidades, máquina de estados con las 30 transiciones |
| [contracts/ipc-protocol.md](contracts/ipc-protocol.md) | Protocolo entre procesos, mensaje por mensaje |
| [contracts/registry-schema.json](contracts/registry-schema.json) | JSON Schema del catálogo de acciones |
| [contracts/config-schema.json](contracts/config-schema.json) | JSON Schema de la configuración del usuario |
| [quickstart.md](quickstart.md) | Escenarios de validación de punta a punta, incluida la vía sin audio |
