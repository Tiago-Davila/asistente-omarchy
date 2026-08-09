<!--
Sync Impact Report
==================
Version change: 1.0.0 → 2.0.0
Bump rationale: MAJOR. El Principio X se redefine de forma incompatible: pasa de
prohibir toda interfaz gráfica en fase 1 a permitir una superficie mínima de
estado bajo cinco condiciones acumulativas. Además se remueve el modelo de
lenguaje del alcance de fase 1, lo que restringe los Principios IV y V respecto
de la versión anterior. Ambos son cambios incompatibles hacia atrás según la
política de versionado de este documento.

Modified principles:
- IV. Determinismo Antes que Modelo — la resolución de intención de fase 1 pasa a
  ser enteramente determinística; el fallback por modelo queda como punto de
  extensión declarado y no implementado.
- V. Degradación Elegante — se ajusta la lista de componentes opcionales de fase 1
  (TTS y wake word); la cláusula sobre el modelo de lenguaje pasa a ser condicional
  a su futura incorporación.
- IX. Separación de Responsabilidades — la capa de interfaz incorpora la superficie
  mínima de estado.
- X. Alcance de Fase 1: Sin Interfaz Gráfica → X. Alcance de Fase 1: Interfaz Mínima
  de Estado. Redefinición incompatible.

Modified sections:
- Restricciones Técnicas y Presupuestos — la tabla se alinea con el texto del
  Principio VI: RAM del stack completo pasa de ≤ 3 GB a ≤ 1.5 GB, y la fila del
  modelo LLM pasa de "≤ 4B parámetros, cuantizado" a "fuera de la fase 1".
- Flujo de Desarrollo y Puertas de Calidad — la puerta "no se introduce código de
  interfaz gráfica" se reemplaza por la verificación de las condiciones del nuevo
  Principio X, y se agrega una puerta de determinismo para el Principio IV.

Added sections: ninguna
Removed sections: ninguna

Deferred TODOs: ninguno

Artefactos dependientes desincronizados por esta enmienda (fuera del alcance de
este comando, requieren actualización aparte):
- CLAUDE.md — repite los presupuestos viejos (≤ 3 GB, LLM ≤ 4B) y afirma que la
  fase 1 no lleva interfaz gráfica.
- specs/001-asistente-voz-eva/spec.md — su sección de conflicto constitucional
  queda obsoleta: el conflicto que documenta está resuelto por esta versión.
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

Las intenciones frecuentes DEBEN resolverse con reglas determinísticas. Si una intención se puede
expresar como regla, DEBE expresarse como regla. **En fase 1 la resolución de intención es
enteramente determinística: no hay modelo de lenguaje generativo.** El fallback por modelo es un
punto de extensión que DEBE quedar declarado y NO DEBE implementarse en esta fase; cuando se
incorpore, será fallback y nunca primera opción.

**Racional**: las reglas son más rápidas, más baratas, reproducibles y testeables que una
inferencia. Sacar el modelo de la fase 1 elimina de entrada al mayor consumidor de RAM y de
latencia del stack, que son los dos presupuestos más ajustados del proyecto.

### V. Degradación Elegante

Ningún componente opcional PUEDE ser condición de arranque del daemon. La ausencia de un
componente opcional se reporta, no aborta. En fase 1 los componentes opcionales son TTS y wake
word, ambos fuera de alcance. Cuando se incorpore el modelo de lenguaje, la capa determinística
DEBE seguir funcionando sin él.

**Racional**: el daemon corre en la laptop de trabajo del usuario; un fallo parcial no puede
volverse un fallo total.

### VI. Presupuesto de Recursos Explícito y Verificable

El sistema corre en una laptop que el usuario está usando para otra cosa. Restricciones duras:

- En reposo: menos de 3% de un core y menos de 250 MB de RSS.
- Stack completo cargado: no más de 1.5 GB de RAM.
- Los modelos DEBEN estar cuantizados. STT no mayor a `whisper small`. LLM fuera de la fase 1.
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
- **interfaz**: CLI, notificaciones y la superficie mínima de estado del Principio X

El dominio NO DEBE conocer audio, ni modelos, ni Hyprland. Las dependencias apuntan hacia
adentro; una violación de dirección de dependencia bloquea el merge.

**Racional**: es la condición estructural que hace cumplibles los principios II y VIII.

### X. Alcance de Fase 1: Interfaz Mínima de Estado

El feedback al usuario en fase 1 es por notificación del sistema, salida estándar, código de
retorno y **una única superficie gráfica mínima de estado**. Esa superficie está permitida solo
si cumple estas cinco condiciones, que son acumulativas y verificables:

1. Muestra únicamente el estado del turno, el texto entendido y la acción resuelta.
2. NO DEBE tomar el foco del teclado en ningún momento.
3. NO DEBE tapar ni desplazar la ventana en la que el usuario está trabajando.
4. DEBE ser visible solo cuando hay algo que comunicar, y desaparecer sola al terminar el turno.
5. El sistema DEBE seguir siendo completamente funcional con esa superficie deshabilitada,
   degradando el feedback a los canales no gráficos.

Si alguna de las cinco deja de cumplirse, la superficie deja de estar permitida.

Queda fuera de la fase 1 todo lo demás: barra de estado permanente, widgets, panel de
configuración, historial navegable, dashboards y cualquier ventana que el usuario pueda enfocar.
La arquitectura DEBE dejar esos puntos de extensión declarados, y NO DEBE implementarlos.

**Racional**: la UI es el sumidero de esfuerzo clásico que impide que el núcleo llegue a
funcionar, y por eso la fase 1 la prohibía por completo. La excepción se admite porque sin
feedback visible el usuario no sabe si el asistente lo escuchó, lo que anula la usabilidad de un
asistente activado por atajo. El costo queda acotado por las cinco condiciones: la condición 5 en
particular garantiza que el núcleo nunca dependa de la superficie gráfica, que es lo que el
principio original protegía.

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
| RAM stack completo | ≤ 1.5 GB | medición con todos los modelos cargados |
| Modelo STT | ≤ `whisper small`, cuantizado | declarado en el plan de la feature |
| Modelo LLM | fuera de la fase 1 | ausencia verificada en el plan de la feature |
| Superficie gráfica | solo la del Principio X, con sus 5 condiciones | revisión contra el Principio X |
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
- La resolución de intención no depende de un modelo generativo (IV).
- La superficie gráfica cumple las cinco condiciones del Principio X, y el sistema sigue siendo
  funcional con ella deshabilitada (X).

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

**Version**: 2.0.0 | **Ratified**: 2026-08-09 | **Last Amended**: 2026-08-09
