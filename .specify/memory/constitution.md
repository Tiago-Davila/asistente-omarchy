<!--
Sync Impact Report
==================
Version change: TEMPLATE (sin versionar) → 1.0.0
Bump rationale: Primera ratificación. Se sustituyen todos los placeholders del
template por los 17 principios obligatorios definidos para el proyecto Eva.

Modified principles:
- [PRINCIPLE_1_NAME] → I. La Especificación Manda
- [PRINCIPLE_2_NAME] → II. Núcleo Desacoplado de la Voz
- [PRINCIPLE_3_NAME] → III. Ejecución por Registry Cerrado (NO NEGOCIABLE)
- [PRINCIPLE_4_NAME] → IV. Determinismo Antes que Modelo
- [PRINCIPLE_5_NAME] → V. Degradación Elegante
- (nuevos) VI–XVII: Presupuesto de Recursos, Latencia, Puertos y Adaptadores,
  Separación de Responsabilidades, Alcance de Fase 1, Cien por Ciento Local,
  Confirmación de Acciones Destructivas, Configuración Declarativa, Tests
  Obligatorios, Observabilidad por Turno, Falla Ruidosa, Especificación Antes
  que Código

Added sections:
- Restricciones Técnicas y Presupuestos (reemplaza [SECTION_2_NAME])
- Flujo de Desarrollo y Puertas de Calidad (reemplaza [SECTION_3_NAME])

Removed sections: ninguna

Deferred TODOs: ninguno
-->

# Constitución de Eva

Eva es un asistente de voz local para escritorio Linux (Omarchy: Arch + Hyprland) que
interpreta lenguaje natural en español rioplatense y ejecuta acciones sobre el sistema
operativo del usuario. Corre como daemon de usuario en una laptop, sin conexión a internet.

Las palabras **DEBE**, **NO DEBE** y **PUEDE** se interpretan como requisitos normativos.
Todo incumplimiento sin excepción documentada bloquea la aceptación del trabajo.

## Core Principles

### I. La Especificación Manda

La especificación tiene precedencia sobre la implementación. NO DEBE programarse
funcionalidad que no esté trazada a una historia de usuario o requisito explícito. Todo
artefacto de código DEBE poder señalar el requisito que lo justifica; el código sin
trazabilidad se elimina o se especifica antes de seguir.

**Racional**: sin trazabilidad no hay forma de decidir qué es alcance y qué es deriva.

### II. Núcleo Desacoplado de la Voz

El sistema DEBE ser completamente operable y testeable enviando texto plano al daemon, sin
micrófono ni audio. La voz es un adaptador de entrada, no el núcleo. Todo test de intención,
ejecución y manejo de errores DEBE poder correr sin dispositivos de audio presentes.

**Racional**: acoplar la lógica al audio vuelve el sistema no testeable en CI y frágil ante
cambios de motor.

### III. Ejecución por Registry Cerrado (NO NEGOCIABLE)

El modelo de lenguaje NO DEBE generar ni ejecutar comandos de shell arbitrarios. Solo elige
una herramienta de un registro explícito y declarado, y completa parámetros tipados. Toda
salida del modelo DEBE validarse contra un esquema antes de ejecutarse. Herramienta
desconocida o parámetro inválido implica rechazo; NO DEBE haber interpretación creativa,
corrección automática ni ejecución aproximada.

**Racional**: es la frontera de seguridad del sistema; un agente con shell libre sobre la
máquina del usuario no es aceptable bajo ninguna justificación de conveniencia.

### IV. Determinismo Antes que Modelo

Las intenciones frecuentes DEBEN resolverse con reglas determinísticas. El modelo de lenguaje
es fallback, no primera opción. Si una intención se puede expresar como regla, DEBE expresarse
como regla.

**Racional**: las reglas son más rápidas, más baratas, reproducibles y testeables que una
inferencia.

### V. Degradación Elegante

Si el modelo de lenguaje no está disponible, la capa determinística DEBE seguir funcionando.
Ningún componente opcional (LLM, TTS, wake word) PUEDE ser condición de arranque del daemon.
La ausencia de un componente opcional se reporta, no aborta.

**Racional**: el daemon corre en la laptop de trabajo del usuario; un fallo parcial no puede
volverse un fallo total.

### VI. Presupuesto de Recursos Explícito y Verificable

El sistema corre en una laptop que el usuario está usando para otra cosa. Restricciones duras:

- En reposo: menos de 3% de un core y menos de 250 MB de RSS.
- Stack completo cargado: no más de 3 GB de RAM.
- Los modelos DEBEN estar cuantizados. STT no mayor a `whisper small`. LLM no mayor a 4B
  parámetros.
- Los componentes pesados se cargan bajo demanda, o se declaran explícitamente como residentes
  con su costo justificado.

Cualquier decisión de diseño que viole este presupuesto DEBE rechazarse, o documentarse como
excepción con justificación en el plan de la feature.

**Racional**: un asistente que degrada la máquina que asiste es un costo neto, no una ayuda.

### VII. La Latencia es un Requisito Funcional

Cada etapa del pipeline DEBE tener un presupuesto de tiempo declarado y medible. El total desde
fin del habla hasta acción ejecutada DEBE estar acotado por un número explícito. La latencia se
mide, no se estima; superar el presupuesto es un defecto funcional, no una optimización
pendiente.

**Racional**: por encima de cierto umbral el asistente deja de ser usable aunque sea correcto.

### VIII. Arquitectura de Puertos y Adaptadores

Cada etapa se define como interfaz con implementación intercambiable: captura de audio, wake
word, transcripción, parser de intención, ejecutor, notificador, síntesis de voz. Cambiar un
motor NO DEBE requerir tocar el núcleo.

**Racional**: los motores locales (STT, LLM, TTS) van a cambiar más rápido que el dominio.

### IX. Separación de Responsabilidades

Las capas y sus responsabilidades son:

- **dominio**: intenciones, herramientas, reglas de validación
- **aplicación**: orquestación del turno y máquina de estados
- **infraestructura**: audio, modelos, IPC, integración con Hyprland
- **interfaz**: CLI y notificaciones

El dominio NO DEBE conocer audio, ni modelos, ni Hyprland. Las dependencias apuntan hacia
adentro; una violación de dirección de dependencia bloquea el merge.

**Racional**: es la condición estructural que hace cumplibles los principios II y VIII.

### X. Alcance de Fase 1: Sin Interfaz Gráfica

El feedback al usuario en fase 1 es por notificación del sistema, salida estándar y código de
retorno. Queda explícitamente fuera de esta fase todo lo relacionado con overlays, layer-shell,
widgets, barra de estado y OSD. La arquitectura DEBE dejar el punto de extensión declarado, y
NO DEBE implementarlo. La interfaz gráfica se especificará como fase aparte.

**Racional**: la UI es el sumidero de esfuerzo clásico que impide que el núcleo llegue a
funcionar.

### XI. Cien por Ciento Local

Ninguna transcripción, comando ni telemetría sale de la máquina. NO DEBE existir ninguna
dependencia de red en tiempo de ejecución. Toda dependencia de red descubierta en una
implementación es un defecto bloqueante.

**Racional**: es la premisa de privacidad del producto, no una preferencia de despliegue.

### XII. Confirmación Explícita de Acciones Destructivas

Las acciones irreversibles o destructivas DEBEN requerir confirmación explícita del usuario,
mostrando la acción concreta a ejecutar antes de ejecutarla. La confirmación NO DEBE inferirse
del contexto, del tono, ni de una interacción previa. Cada herramienta del registry declara si
es destructiva; el default ante duda es "requiere confirmación".

**Racional**: la voz es un canal con ruido y ambigüedad; el costo de un falso positivo
destructivo es asimétrico.

### XIII. Configuración Declarativa

Nada del entorno del usuario se hardcodea: navegador, terminal, aplicaciones, rutas y atajos se
leen de configuración o del sistema. El comportamiento se cambia editando configuración, no
código. Un valor específico del entorno embebido en el código fuente es un defecto.

**Racional**: Omarchy es un entorno personalizable; el asistente tiene que seguir al usuario.

### XIV. Tests Obligatorios sobre lo que Puede Fallar en Silencio

DEBEN existir tests para:

- un conjunto de referencia de frases en español rioplatense con la intención y los parámetros
  esperados
- cada herramienta del registry, con validación de parámetros
- casos de rechazo: intención ambigua, herramienta inexistente, parámetro fuera de rango

Una herramienta sin test NO DEBE entrar al registry.

**Racional**: la interpretación de lenguaje falla en silencio y produce acciones equivocadas
que el usuario descubre tarde.

### XV. Observabilidad Mínima por Turno

Cada interacción DEBE dejar registro estructurado de: transcripción, intención resuelta,
herramienta elegida, resultado y latencia por etapa. Los registros son locales (ver principio
XI). Un turno sin registro estructurado es un defecto de implementación.

**Racional**: sin esta traza no se pueden ajustar reglas, prompts ni presupuestos de latencia.

### XVI. Falla Ruidosa

El sistema NO DEBE ejecutar una aproximación cuando no entendió. Ante ambigüedad: pregunta o
rechaza, con un mensaje que indique qué no se entendió. NO DEBE adivinar, ni elegir la opción
más probable, ni ejecutar la acción "más parecida".

**Racional**: un asistente que adivina obliga al usuario a verificar todo, lo que anula su
propósito.

### XVII. Especificación Antes que Código

NO DEBE implementarse código durante las fases de especificación, aclaración, checklist,
planificación y generación de tareas. En esas fases solo se crean o actualizan los documentos
correspondientes. La implementación empieza cuando la fase de implementación empieza.

**Racional**: escribir código durante la especificación convierte las decisiones abiertas en
hechos consumados.

## Restricciones Técnicas y Presupuestos

Estas restricciones son verificables y DEBEN comprobarse antes de aceptar una feature:

| Dimensión | Límite duro | Cómo se verifica |
| --- | --- | --- |
| CPU en reposo | < 3% de un core | medición sostenida del daemon ocioso |
| RSS en reposo | < 250 MB | medición del proceso daemon ocioso |
| RAM stack completo | ≤ 3 GB | medición con todos los modelos cargados |
| Modelo STT | ≤ `whisper small`, cuantizado | declarado en el plan de la feature |
| Modelo LLM | ≤ 4B parámetros, cuantizado | declarado en el plan de la feature |
| Latencia por etapa | presupuesto declarado por etapa | registro estructurado por turno (XV) |
| Latencia total | cota explícita fin de habla → acción | registro estructurado por turno (XV) |
| Dependencias de red en runtime | cero | revisión de dependencias e integración |

Todo componente pesado DEBE clasificarse como *bajo demanda* o *residente*. Los residentes
DEBEN justificar su costo de memoria en el plan de la feature.

Cualquier excepción a esta tabla DEBE registrarse en el documento de plan de la feature
correspondiente, con la justificación y el costo aceptado. Una excepción no documentada es un
incumplimiento.

## Flujo de Desarrollo y Puertas de Calidad

El proyecto se desarrolla con el flujo Spec Kit. Las fases son secuenciales y cada una produce
documentos, no código, hasta llegar a implementación:

1. `constitution` → principios de gobierno (este documento)
2. `specify` → especificación de la feature
3. `clarify` → resolución de ambigüedades sobre la especificación
4. `checklist` → verificación de completitud de la especificación
5. `plan` → diseño técnico, presupuestos y contratos de puertos
6. `tasks` → descomposición en tareas ordenadas por dependencia
7. `implement` → única fase en la que se escribe código

Puertas de calidad para aceptar una implementación:

- Cada cambio de código traza a un requisito o historia de usuario (I).
- El camino ejecutado por los tests no requiere audio (II).
- Toda herramienta nueva llega con su test de validación de parámetros y sus casos de rechazo
  (XIV), y con su clasificación de destructividad (XII).
- El presupuesto de recursos y de latencia se mide, no se asume (VI, VII).
- No se introducen dependencias de red en runtime (XI).
- El dominio no importa audio, modelos ni Hyprland (IX).
- No se introduce código de interfaz gráfica (X).

## Governance

Esta constitución tiene precedencia sobre cualquier otra práctica, preferencia de estilo o
conveniencia de implementación del proyecto. Ante conflicto entre un documento de feature y
esta constitución, manda la constitución.

**Enmiendas**: toda modificación DEBE hacerse editando este archivo mediante el comando
`/speckit-constitution`, incluir un Sync Impact Report al inicio del archivo, y actualizar la
versión y la fecha de última enmienda en la misma operación.

**Versionado semántico** de esta constitución:

- **MAJOR**: eliminación o redefinición incompatible de un principio o de una regla de gobierno.
- **MINOR**: incorporación de un principio o sección nueva, o ampliación material de una guía.
- **PATCH**: aclaraciones, redacción, correcciones no semánticas.

**Cumplimiento**: toda revisión de especificación, plan, tareas o código DEBE verificar
explícitamente el cumplimiento de los principios aplicables. Un incumplimiento sin excepción
documentada bloquea la aceptación. Las excepciones se documentan en el plan de la feature
afectada, con justificación y alcance acotado; una excepción no se generaliza a otras features.

**Version**: 1.0.0 | **Ratified**: 2026-08-09 | **Last Amended**: 2026-08-09
