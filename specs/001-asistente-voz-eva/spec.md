# Feature Specification: Eva — Asistente de Voz Local para Escritorio

**Feature Branch**: `001-asistente-voz-eva`

**Created**: 2026-08-09

**Status**: Draft — requiere `/speckit-clarify` antes de planificar

**Input**: Descripción del usuario: asistente de voz local para escritorio Linux que interpreta
comandos hablados en español rioplatense y ejecuta acciones sobre el entorno de escritorio,
sin conexión a internet y sin degradar el rendimiento del sistema.

---

## ⚠ Conflicto con la Constitución (resolver antes de `/speckit-plan`)

El requisito de **interfaz gráfica mínima** (US4, FR-034 a FR-039) contradice de forma directa el
**Principio X — Alcance de Fase 1: Sin Interfaz Gráfica** de
[.specify/memory/constitution.md](../../.specify/memory/constitution.md), que dice textualmente:
"Queda explícitamente fuera de esta fase todo lo relacionado con overlays, layer-shell, widgets,
barra de estado y OSD. La arquitectura DEBE dejar el punto de extensión declarado, y NO DEBE
implementarlo."

Esto no es una ambigüedad interpretable: son dos afirmaciones incompatibles sobre el mismo alcance.
La especificación se redacta **incluyendo** la interfaz gráfica porque el usuario la solicitó de
forma explícita y detallada, pero el conflicto DEBE resolverse antes de la fase de planificación
por una de estas dos vías:

- **Vía A — Enmendar la constitución**: redefinir el Principio X para admitir una interfaz mínima
  no intrusiva en fase 1. Es una redefinición incompatible de un principio, por lo que exige un
  bump **MAJOR** a la versión 2.0.0 vía `/speckit-constitution`.
- **Vía B — Recortar el alcance**: quitar US4 de esta feature y dejar el feedback en notificación
  del sistema, stdout y código de retorno, como manda hoy el Principio X. US4 pasaría a una feature
  posterior.

Ninguna otra restricción del usuario entra en conflicto con la constitución. Los presupuestos de
recursos que pide el usuario (<1% de un núcleo, <150 MB en reposo, ≤1,5 GB por turno) son **más
estrictos** que los constitucionales (<3%, <250 MB, ≤3 GB), y un límite más estricto cumple el
principio VI sin necesitar excepción.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ejecutar una acción escribiendo texto plano (Priority: P1)

El usuario envía al asistente una frase en texto plano —sin micrófono, sin audio— y el asistente
resuelve la intención, valida los parámetros, ejecuta la acción sobre el escritorio y devuelve el
resultado. Es el núcleo del sistema: el registro cerrado de acciones, la validación y la ejecución.

**Why this priority**: es el único camino que hace al sistema testeable de punta a punta sin
hardware de audio, y el Principio II lo exige como condición de diseño. Todo lo demás se apoya
acá. Entregado solo, ya es un lanzador de acciones por lenguaje natural utilizable.

**Independent Test**: se prueba enviando frases de texto al asistente y verificando que la acción
correcta se ejecutó con los parámetros correctos, en una máquina sin micrófono conectado.

**Acceptance Scenarios**:

1. **Given** el asistente corriendo y una configuración con la aplicación "navegador" declarada,
   **When** el usuario envía el texto "abrí el navegador", **Then** el asistente ejecuta la acción
   de abrir esa aplicación y reporta éxito.
2. **Given** el asistente corriendo, **When** el usuario envía "andá al escritorio 3", **Then** el
   asistente cambia al espacio de trabajo 3 y reporta éxito.
3. **Given** el asistente corriendo, **When** el usuario envía "buscá recetas de milanesas",
   **Then** el asistente ejecuta una búsqueda web con ese texto como término de búsqueda.
4. **Given** el asistente corriendo, **When** el usuario envía una frase que no corresponde a
   ninguna acción declarada, **Then** el asistente rechaza, no ejecuta nada, y reporta el motivo
   "acción desconocida".
5. **Given** el asistente corriendo, **When** el usuario envía "abrí" sin nombrar aplicación,
   **Then** el asistente rechaza por parámetro obligatorio faltante y no inventa un valor.
6. **Given** una misma intención expresada en cinco fraseos rioplatenses distintos (voseo,
   imperativo con enclítico, muletilla inicial), **When** se envía cada uno, **Then** los cinco
   resuelven a la misma acción con los mismos parámetros.

---

### User Story 2 - Ejecutar una acción hablando, con atajo de teclado (Priority: P1)

El usuario presiona un atajo de teclado global, habla una frase, deja de hablar, y el asistente
transcribe, resuelve y ejecuta la acción sin que el usuario tenga que indicar cuándo terminó.

**Why this priority**: es la propuesta de valor del producto. Sin esto, Eva es un lanzador de
comandos más. Depende de US1 pero se prueba de forma independiente sobre el mismo núcleo.

**Independent Test**: se prueba activando por atajo, reproduciendo audio grabado de frases de
referencia, y verificando que la acción ejecutada coincide con la esperada y que la captura se
detuvo sola al terminar el habla.

**Acceptance Scenarios**:

1. **Given** el asistente en reposo, **When** el usuario presiona el atajo global y dice "abrí
   la terminal", **Then** el asistente captura, detecta el fin del habla por su cuenta, transcribe,
   y ejecuta la acción de abrir la terminal.
2. **Given** el asistente en reposo, **When** el usuario presiona el atajo y no dice nada,
   **Then** el asistente descarta la activación sin ejecutar ninguna acción y vuelve a reposo.
3. **Given** una activación en curso con el usuario hablando, **When** el usuario cancela antes de
   que la acción se ejecute, **Then** no se ejecuta ninguna acción y el asistente vuelve a reposo.
4. **Given** el asistente en reposo y sin activación, **When** se inspecciona el estado del
   dispositivo de audio, **Then** el asistente no tiene el micrófono abierto ni está capturando.
5. **Given** la misma frase de referencia enviada por voz y por texto, **When** ambas se procesan,
   **Then** producen exactamente la misma acción con los mismos parámetros.

---

### User Story 3 - Confirmación explícita de acciones destructivas (Priority: P2)

Cuando la acción resuelta está marcada como destructiva o irreversible, el asistente muestra la
acción concreta que va a ejecutar y espera una confirmación explícita antes de ejecutarla.

**Why this priority**: el Principio XII lo exige y el canal de voz es ruidoso y ambiguo, por lo que
el costo de un falso positivo destructivo es asimétrico. No es P1 porque el conjunto inicial de
acciones puede entregarse marcando todas como no destructivas salvo el cierre de ventana.

**Independent Test**: se prueba pidiendo una acción declarada destructiva y verificando que nada
se ejecuta hasta que llega una confirmación explícita, y que una no-respuesta o una cancelación
dejan el sistema sin cambios.

**Acceptance Scenarios**:

1. **Given** una acción marcada como destructiva, **When** el usuario la pide, **Then** el
   asistente muestra la acción concreta con sus parámetros y queda esperando confirmación sin
   ejecutar nada.
2. **Given** el asistente esperando confirmación, **When** el usuario confirma explícitamente,
   **Then** la acción se ejecuta y se reporta el resultado.
3. **Given** el asistente esperando confirmación, **When** el usuario cancela, **Then** la acción
   no se ejecuta y el asistente vuelve a reposo.
4. **Given** el asistente esperando confirmación, **When** vence el tiempo de espera sin respuesta,
   **Then** la acción no se ejecuta y el asistente vuelve a reposo.
5. **Given** una acción cuya clasificación de destructividad no está declarada, **When** el usuario
   la pide, **Then** el asistente la trata como destructiva y pide confirmación.

---

### User Story 4 - Ver el estado del asistente sin cambiar de ventana (Priority: P2)

El asistente muestra en pantalla, de forma mínima y no intrusiva, en qué estado está, qué texto
entendió y qué acción va a ejecutar, sin robar el foco del teclado ni tapar la ventana de trabajo.

**Why this priority**: sin feedback visible el usuario no sabe si el asistente lo escuchó, y eso
rompe la usabilidad. Es P2 y no P1 porque US1 y US2 pueden entregar feedback por notificación del
sistema mientras tanto. **Bloqueada por el conflicto con el Principio X descrito arriba.**

**Independent Test**: se prueba activando el asistente mientras se escribe en un editor y
verificando que el cursor de texto no se pierde, que el editor mantiene el foco, y que la interfaz
aparece y desaparece sola.

**Acceptance Scenarios**:

1. **Given** el usuario escribiendo en un editor, **When** activa el asistente, **Then** la
   interfaz aparece, el foco del teclado permanece en el editor, y lo que el usuario escriba sigue
   yendo al editor.
2. **Given** el asistente activo, **When** cambia de estado (escuchando → procesando → ejecutando),
   **Then** la interfaz refleja cada cambio de estado.
3. **Given** un turno terminado, **When** pasa el tiempo de permanencia definido, **Then** la
   interfaz desaparece sola sin intervención del usuario.
4. **Given** el asistente esperando confirmación, **When** el usuario confirma o cancela desde el
   teclado, **Then** la acción se resuelve sin que el foco haya cambiado de ventana.
5. **Given** el asistente en reposo sin nada que comunicar, **When** se observa la pantalla,
   **Then** no hay ningún elemento de interfaz visible.

---

### User Story 5 - Declarar qué conoce el asistente sin tocar código (Priority: P2)

El usuario declara en configuración qué aplicaciones, sitios y acciones propias conoce el
asistente, y qué sinónimos y alias corresponden a cada una, y aplica los cambios sin reiniciar
la máquina.

**Why this priority**: sin esto el asistente solo sirve para el entorno exacto de quien lo
programó, y el Principio XIII prohíbe hardcodear el entorno del usuario. Es P2 porque US1 puede
entregarse con una configuración inicial fija mientras se construye la recarga.

**Independent Test**: se prueba agregando una aplicación y un alias nuevos a la configuración,
recargando, y verificando que una frase que antes se rechazaba ahora resuelve correctamente.

**Acceptance Scenarios**:

1. **Given** una aplicación no declarada, **When** el usuario la pide por voz o texto, **Then** el
   asistente rechaza indicando que no está declarada.
2. **Given** esa misma aplicación agregada a la configuración y la configuración recargada,
   **When** el usuario la pide de nuevo, **Then** el asistente la abre correctamente.
3. **Given** un alias declarado para una acción, **When** el usuario usa el alias, **Then** resuelve
   a la misma acción que el nombre canónico.
4. **Given** una acción propia declarada por el usuario en configuración, **When** el usuario la
   pide, **Then** el asistente la ejecuta.
5. **Given** una configuración con errores, **When** el asistente la carga, **Then** reporta el
   error de forma visible y no queda operando en silencio con una configuración inválida.

---

### User Story 6 - Entender por qué el asistente hizo o no hizo algo (Priority: P3)

Cada interacción queda registrada con el texto transcripto, la acción resuelta, los parámetros,
el resultado y el tiempo de cada etapa, y el usuario puede consultar ese registro.

**Why this priority**: el Principio XV lo exige y es la única forma de ajustar reglas y
presupuestos de latencia con datos. Es P3 porque no bloquea el uso diario, pero sin esto no se
puede mejorar el sistema ni diagnosticar un rechazo inesperado.

**Independent Test**: se prueba ejecutando una serie de turnos —exitosos, rechazados y fallidos— y
verificando que cada uno dejó un registro consultable con todos los campos exigidos.

**Acceptance Scenarios**:

1. **Given** un turno ejecutado con éxito, **When** el usuario consulta el registro, **Then**
   encuentra el texto transcripto, la acción, los parámetros, el resultado y la latencia por etapa.
2. **Given** un turno rechazado por intención desconocida, **When** el usuario consulta el registro,
   **Then** encuentra el texto y el motivo del rechazo.
3. **Given** un turno que excedió el presupuesto de latencia, **When** el usuario consulta el
   registro, **Then** puede identificar qué etapa consumió el tiempo.

---

### Edge Cases

Cada caso borde tiene un comportamiento esperado declarado y verificable:

| # | Caso | Comportamiento esperado |
|---|------|-------------------------|
| EC-01 | El micrófono está ocupado por otra aplicación | El asistente no ejecuta ninguna acción, reporta el error de dispositivo ocupado de forma comprensible y vuelve a reposo. No reintenta en bucle. |
| EC-02 | No se detecta habla en la activación | Se descarta la activación en silencio o con señal mínima, no se transcribe, no se ejecuta nada, vuelve a reposo. |
| EC-03 | El usuario dice algo que no corresponde a ninguna acción conocida | Rechazo con motivo "acción desconocida". Nunca se ejecuta la acción más parecida. |
| EC-04 | El usuario nombra una aplicación no instalada o no declarada | Rechazo con motivo distinguible: "no declarada" (falta en configuración) o "no disponible" (declarada pero ausente del sistema). |
| EC-05 | El usuario indica un espacio de trabajo fuera del rango válido | Rechazo por parámetro fuera de rango, indicando el rango válido. No se recorta ni se aproxima al valor más cercano. |
| EC-06 | La aplicación pedida ya está abierta | Comportamiento definido por [AMB-02]; en todos los casos el resultado se reporta al usuario, sin fallar en silencio. |
| EC-07 | El usuario activa mientras un turno anterior está en curso | La segunda activación no corrompe el turno en curso. El sistema aplica una política única y declarada: ignorar la nueva activación mientras haya una en curso, informándolo. |
| EC-08 | El gestor de ventanas no responde o devuelve error | El turno termina en estado de error con motivo "fallo al ejecutar", el daemon sigue vivo, y el registro conserva el intento. |
| EC-09 | El sistema está bajo carga alta y se excede el presupuesto de latencia | La acción se completa igual si ya fue validada, el exceso se registra como tal, y el usuario recibe feedback de que el turno tardó más de lo previsto. Nunca se aborta a mitad de una acción ya iniciada. |
| EC-10 | La transcripción devuelve texto parcial o cortado | Se trata como cualquier texto: si resuelve a una acción con todos sus parámetros válidos, se ejecuta; si no, se rechaza. No se completa ni se adivina el texto faltante. |
| EC-11 | El usuario cancela mientras espera confirmación | La acción no se ejecuta, el sistema vuelve a reposo, y el registro deja constancia de la cancelación. |
| EC-12 | El dispositivo de audio cambia en medio de la captura | El turno en curso termina en error de captura sin ejecutar nada; el asistente se recupera y queda operativo para la siguiente activación sin reiniciar el daemon. |

---

## Requirements *(mandatory)*

### Functional Requirements

**Activación y captura**

- **FR-001**: El sistema MUST permitir al usuario iniciar un turno mediante un atajo de teclado global.
- **FR-002**: El sistema MUST capturar audio únicamente durante un turno activado, y MUST NOT capturar audio en ningún otro momento.
- **FR-003**: El sistema MUST permitir al usuario cancelar un turno en curso antes de que la acción se ejecute.
- **FR-004**: El sistema MUST liberar el dispositivo de audio al terminar el turno, sea cual sea el resultado.

**Transcripción**

- **FR-005**: El sistema MUST transcribir a texto en español el audio capturado durante el turno.
- **FR-006**: El sistema MUST detectar por su cuenta el fin del habla y dejar de capturar, sin acción del usuario.
- **FR-007**: El sistema MUST descartar los turnos en los que no se detectó habla, sin ejecutar ninguna acción.

**Resolución de intención**

- **FR-008**: El sistema MUST convertir el texto en una acción concreta con parámetros tipados, elegida de un registro cerrado y declarado de acciones.
- **FR-009**: El sistema MUST rechazar y avisar cuando el texto no corresponde a ninguna acción del registro, y MUST NOT ejecutar una acción aproximada.
- **FR-010**: El sistema MUST rechazar o solicitar el dato cuando falta un parámetro obligatorio, y MUST NOT inventar valores por defecto no declarados.
- **FR-011**: El sistema MUST resolver a la misma acción las variaciones naturales de fraseo de una misma intención, incluyendo voseo, imperativos con pronombre enclítico y muletillas iniciales.
- **FR-012**: El sistema MUST tolerar nombres de aplicaciones y sitios en inglés dentro de una frase en español.
- **FR-013**: El sistema MUST resolver las intenciones frecuentes de forma determinística, sin depender de un modelo generativo (ver *Fuera de alcance*).

**Acciones disponibles**

- **FR-014**: El sistema MUST poder abrir una aplicación declarada en configuración.
- **FR-015**: El sistema MUST poder abrir una aplicación declarada en un espacio de trabajo indicado.
- **FR-016**: El sistema MUST poder abrir una URL en el navegador declarado.
- **FR-017**: El sistema MUST poder abrir una URL en un espacio de trabajo indicado.
- **FR-018**: El sistema MUST poder realizar una búsqueda web con un texto dictado por el usuario.
- **FR-019**: El sistema MUST poder cambiar al espacio de trabajo indicado.
- **FR-020**: El sistema MUST poder mover la ventana activa a un espacio de trabajo indicado.
- **FR-021**: El sistema MUST poder operar sobre la ventana activa: cerrarla, ponerla en pantalla completa y alternar su estado flotante.
- **FR-022**: El sistema MUST poder controlar volumen y reproducción de medios.
- **FR-023**: El sistema MUST poder ejecutar una acción definida por el usuario, tomada exclusivamente de una lista declarada en configuración.

**Validación y seguridad**

- **FR-024**: El sistema MUST validar cada parámetro antes de ejecutar, comprobando tipo, rango y pertenencia al conjunto de valores permitidos.
- **FR-025**: El sistema MUST NOT ejecutar comandos arbitrarios provenientes de la transcripción ni de ninguna otra entrada del usuario.
- **FR-026**: El sistema MUST exigir confirmación explícita del usuario antes de ejecutar una acción marcada como destructiva o irreversible, mostrando previamente la acción concreta y sus parámetros.
- **FR-027**: El sistema MUST NOT inferir ni asumir la confirmación a partir del contexto, del turno anterior ni del silencio del usuario.
- **FR-028**: El sistema MUST tratar como destructiva toda acción cuya clasificación no esté declarada. [NEEDS CLARIFICATION: ¿qué acciones concretas del catálogo FR-014 a FR-023 se consideran destructivas? Ver AMB-01.]

**Feedback al usuario**

- **FR-029**: El sistema MUST exponer de forma observable su estado actual mientras está activo, distinguiendo al menos: en reposo, escuchando, procesando, ejecutando, esperando confirmación y error.
- **FR-030**: El sistema MUST mostrar al usuario el texto que entendió y la acción que va a ejecutar, antes o durante la ejecución.
- **FR-031**: El sistema MUST informar los errores distinguiendo al menos cuatro clases: no se entendió, acción desconocida, parámetro inválido y fallo al ejecutar.
- **FR-032**: El sistema MUST emitir una señal sonora breve al terminar el turno, distinta para éxito y para error, reconocible sin mirar la pantalla.
- **FR-033**: El sistema MUST reportar el resultado de cada turno también por un canal no gráfico, de modo que la funcionalidad siga siendo observable sin interfaz.

**Interfaz gráfica** *(condicionada al conflicto con el Principio X — ver arriba)*

- **FR-034**: El sistema MUST mostrar una interfaz gráfica mínima con el estado, la transcripción y la acción resuelta.
- **FR-035**: La interfaz MUST ocupar una porción reducida de la pantalla y MUST NOT tapar ni desplazar la ventana en la que el usuario está trabajando.
- **FR-036**: La interfaz MUST NOT tomar el foco del teclado en ningún momento.
- **FR-037**: La interfaz MUST ser visible solo cuando hay algo que comunicar y MUST desaparecer sola al terminar el turno.
- **FR-038**: El usuario MUST poder confirmar o cancelar una acción pendiente sin que cambie el foco de su ventana actual.
- **FR-039**: El sistema MUST seguir siendo completamente funcional con la interfaz gráfica deshabilitada, degradando el feedback a los canales de FR-033.

**Entrada por texto equivalente**

- **FR-040**: El sistema MUST aceptar comandos como texto plano y producir exactamente el mismo resultado que por voz.
- **FR-041**: Toda la funcionalidad del sistema MUST ser ejercitable sin micrófono ni dispositivos de audio presentes.

**Configuración**

- **FR-042**: El usuario MUST poder declarar en configuración qué aplicaciones, sitios y acciones propias conoce el asistente, sin modificar código.
- **FR-043**: El usuario MUST poder declarar sinónimos y alias que resuelvan a una misma acción.
- **FR-044**: El sistema MUST poder aplicar cambios de configuración sin reiniciar la máquina.
- **FR-045**: El sistema MUST reportar de forma visible los errores de configuración y MUST NOT operar en silencio con una configuración inválida.

**Registro de actividad**

- **FR-046**: El sistema MUST registrar cada interacción con: texto transcripto, acción resuelta, parámetros, resultado y tiempo de cada etapa.
- **FR-047**: El usuario MUST poder consultar ese registro para entender por qué el asistente hizo o no hizo algo.
- **FR-048**: El registro MUST permanecer en la máquina del usuario y MUST NOT enviarse a ningún destino externo.

**Robustez**

- **FR-049**: El fallo de un componente MUST NOT terminar el proceso del asistente.
- **FR-050**: El sistema MUST NOT dejar el escritorio en estado inconsistente si un turno falla a mitad de camino.
- **FR-051**: El sistema MUST recuperarse de un cambio de dispositivo de audio y quedar operativo para el siguiente turno sin reiniciar.
- **FR-052**: El sistema MUST aplicar una política única y declarada cuando llega una activación con un turno ya en curso.

### Key Entities

- **Turno**: una interacción completa de punta a punta. Atributos: identificador, momento de inicio, texto de entrada (transcripto o escrito), intención resuelta, acción elegida, parámetros, estado final (ejecutado / rechazado / cancelado / fallido), motivo si no se ejecutó, y duración por etapa.
- **Acción del registro**: una capacidad que el asistente sabe ejecutar. Atributos: nombre canónico, alias y sinónimos, lista de parámetros con su tipo y si son obligatorios, valores permitidos por parámetro, y clasificación de destructividad.
- **Parámetro**: un dato tipado que una acción necesita. Atributos: nombre, tipo, obligatoriedad, rango o conjunto de valores válidos.
- **Configuración del usuario**: la declaración de entorno. Contiene aplicaciones conocidas, sitios conocidos, acciones propias, alias, y el mapeo de atajos.
- **Estado del asistente**: el valor observable del sistema en un momento dado, dentro del conjunto cerrado: reposo, escuchando, procesando, ejecutando, esperando confirmación, error.
- **Frase de referencia**: un elemento del conjunto de prueba. Atributos: texto de la frase en español rioplatense, intención esperada y parámetros esperados, o la marca de que debe rechazarse.

### Non-Functional Requirements

**Consumo de recursos**

- **NFR-001**: En reposo el asistente MUST consumir menos del 1% de un núcleo, medido como promedio sostenido durante 10 minutos sin activaciones.
- **NFR-002**: En reposo el asistente MUST consumir menos de 150 MB de memoria residente, sumando todos sus procesos.
- **NFR-003**: Durante un turno completo, el consumo total de memoria del asistente y sus procesos auxiliares MUST NOT superar 1,5 GB en su pico.
- **NFR-004**: El asistente MUST dejar al menos un núcleo libre en todo momento; MUST NOT saturar los 4 núcleos disponibles.
- **NFR-005**: Mientras el asistente procesa un turno, la ventana en la que el usuario trabaja MUST NOT perder fluidez de forma perceptible.

**Latencia**

- **NFR-006**: Desde el fin del habla hasta la ejecución de la acción, el tiempo MUST ser menor a 2 segundos en el percentil 50 del conjunto de frases de referencia.
- **NFR-007**: Ese mismo tiempo MUST ser menor a 3,5 segundos en el percentil 95.
- **NFR-008**: El cambio de estado MUST reflejarse de forma observable en menos de 100 milisegundos desde la activación.
- **NFR-009**: Cada etapa del procesamiento MUST tener un presupuesto de tiempo declarado, y el tiempo real de cada etapa MUST quedar registrado por turno.

**Privacidad y operación offline**

- **NFR-010**: Ningún audio, transcripción, comando ni dato de uso MUST salir de la máquina.
- **NFR-011**: El sistema MUST funcionar completo sin conexión a internet; MUST NOT tener ninguna dependencia de red en tiempo de ejecución.

**Confiabilidad**

- **NFR-012**: El asistente MUST sobrevivir sin reiniciarse a: micrófono ocupado, cambio de dispositivo de audio, error del gestor de ventanas y configuración inválida.
- **NFR-013**: Tras cualquier fallo de turno, el asistente MUST volver al estado de reposo y quedar disponible para el siguiente turno.

**Precisión**

- **NFR-014**: Sobre el conjunto de frases de referencia, el sistema MUST resolver la intención y los parámetros correctos en al menos el 90% de los casos.
- **NFR-015**: Sobre ese mismo conjunto, la tasa de ejecución de una acción distinta de la pedida MUST ser menor al 1%. Un fallo debe manifestarse como rechazo, no como acción equivocada.
- **NFR-016**: El sistema MUST rechazar el 100% de las frases del conjunto de referencia marcadas como "debe rechazarse".

**Usabilidad**

- **NFR-017**: El usuario MUST poder ejecutar cada acción del catálogo sin memorizar una sintaxis exacta: al menos 3 fraseos naturales distintos por acción MUST resolver correctamente.
- **NFR-018**: El usuario MUST poder determinar el estado del asistente sin interrumpir su trabajo ni cambiar de ventana.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El usuario ejecuta cualquier acción del catálogo hablando, sin sacar las manos del teclado y sin cambiar de ventana, en menos de 3,5 segundos desde que termina de hablar, en el 95% de los intentos.
- **SC-002**: El asistente resuelve correctamente la intención y los parámetros en al menos el 90% de las frases del conjunto de referencia en español rioplatense.
- **SC-003**: Menos del 1% de las frases del conjunto de referencia producen una acción distinta de la pedida.
- **SC-004**: El 100% de las frases marcadas como "debe rechazarse" son rechazadas, con un motivo comprensible para el usuario.
- **SC-005**: El asistente corre durante una jornada de trabajo completa consumiendo en reposo menos del 1% de un núcleo y menos de 150 MB, medido a lo largo de la jornada.
- **SC-006**: El usuario no percibe pérdida de fluidez en su ventana de trabajo durante el procesamiento de un turno.
- **SC-007**: Ninguna transcripción, comando ni dato de uso abandona la máquina, verificado con la interfaz de red inhabilitada durante una sesión completa de uso.
- **SC-008**: El asistente completa una jornada de trabajo sin reiniciarse, incluyendo al menos un evento de micrófono ocupado y un cambio de dispositivo de audio.
- **SC-009**: El usuario agrega una aplicación nueva y un alias a su configuración y los usa correctamente en menos de 2 minutos, sin tocar código ni reiniciar la máquina.
- **SC-010**: Ante cualquier turno que no hizo lo esperado, el usuario puede determinar la causa consultando el registro, sin reproducir el problema.
- **SC-011**: El 100% de las acciones clasificadas como destructivas quedan sin ejecutarse hasta recibir confirmación explícita.
- **SC-012**: El conjunto completo de pruebas de intención, validación, rechazo y ejecución corre de punta a punta en una máquina sin dispositivos de audio.

---

## Ambigüedades identificadas

El usuario pidió marcar las ambigüedades de forma explícita. Se identificaron 7. La plantilla de
Spec Kit limita a 3 los marcadores `[NEEDS CLARIFICATION]` incrustados en los requisitos, así que
las 3 de mayor impacto quedan marcadas como bloqueantes y las 4 restantes se resuelven con un
supuesto razonable documentado, revisable en `/speckit-clarify`. Ninguna se perdió.

### Bloqueantes — requieren decisión antes de `/speckit-plan`

- **AMB-01** *(referida en FR-028)* — **¿Qué acciones se consideran destructivas?** Del catálogo
  FR-014 a FR-023, "cerrar la ventana activa" es la única candidata evidente, pero las acciones
  propias del usuario (FR-023) pueden serlo y el sistema no puede saberlo solo. Impacta seguridad y
  define si cada acción propia debe declarar su destructividad de forma obligatoria.
- **AMB-02** *(referida en FR-014 y EC-06)* — **Si la aplicación pedida ya está abierta: ¿enfocar
  la instancia existente o abrir una nueva?** Cambia el resultado observable de la acción más usada
  del catálogo, y determina si la acción necesita un parámetro adicional para elegir entre ambos
  comportamientos.
- **AMB-03** *(referida en FR-016 y FR-018)* — **¿Cómo se distingue una búsqueda web de la apertura
  de un sitio cuando el usuario dice el nombre de un sitio?** "abrí YouTube" y "buscá YouTube"
  pueden resolver a acciones distintas, y sin una regla declarada el sistema tendría que adivinar,
  lo que el Principio XVI prohíbe.

### Resueltas con supuesto documentado — confirmar en `/speckit-clarify`

- **AMB-04** — **"Pestaña" en el habla del usuario.** Supuesto: significa siempre *espacio de
  trabajo del gestor de ventanas*. Fundamento: el control de pestañas del navegador exige operar
  la aplicación por dentro, y "control de aplicaciones específicas más allá de abrirlas" está
  explícitamente fuera de alcance, así que la otra lectura no tiene acción que la respalde.
- **AMB-05** — **Espacio de trabajo no indicado.** Supuesto: se usa el espacio de trabajo actual,
  salvo que la aplicación declare uno propio en configuración, en cuyo caso gana el declarado.
  Fundamento: es el comportamiento menos sorprendente y no requiere que el sistema busque un
  espacio "libre", noción que además no está definida.
- **AMB-06** — **Interfaz gráfica por pedido explícito.** Supuesto: la interfaz aparece únicamente
  por el ciclo automático del turno; no hay una acción para invocarla por separado. Fundamento:
  FR-037 exige que sea visible solo cuando hay algo que comunicar, y una invocación manual sin
  turno en curso no tendría contenido que mostrar.
- **AMB-07** — **Configuración con errores al arrancar.** Supuesto: el daemon arranca igual, deja
  fuera del registro únicamente las entradas inválidas, reporta el error de forma visible y sigue
  operando con las entradas válidas. Si el archivo entero es ilegible, arranca con el registro
  vacío y lo reporta; nunca arranca en silencio ni se niega a arrancar. Fundamento: combina el
  Principio V (degradación elegante) con el XVI (falla ruidosa).

---

## Fuera de alcance

Queda explícitamente fuera de esta feature:

- Palabra de activación por voz (wake word).
- Modelo de lenguaje generativo para interpretar frases no reconocidas. La resolución de intención
  de esta fase es enteramente determinística.
- Síntesis de voz para respuestas habladas.
- Conversación de múltiples turnos con memoria de contexto. Cada turno es independiente; la única
  excepción es el par acción-confirmación de US3, que no constituye memoria conversacional.
- Consultas que devuelven información en lugar de ejecutar acciones.
- Control de aplicaciones específicas más allá de abrirlas.
- Múltiples usuarios o perfiles.
- Instalación empaquetada o distribución a terceros.

La arquitectura DEBE dejar declarado el punto de extensión para el fallback por modelo de lenguaje
(Principio IV) y para la síntesis de voz (Principio VIII), sin implementarlos.

---

## Assumptions

- **Entorno**: el usuario corre Omarchy (Arch + Hyprland) en una laptop de 4 núcleos y 8 GB de RAM
  con gráficos integrados, usada simultáneamente para desarrollo. El asistente es siempre un
  proceso accesorio, nunca el principal.
- **Actor único**: hay un solo usuario humano, dueño de la máquina, sin necesidad de autenticación,
  autorización ni separación de permisos.
- **Sesión gráfica**: el asistente corre dentro de una sesión gráfica del usuario ya iniciada, con
  el gestor de ventanas disponible. No cubre el arranque del sistema ni la pantalla de login.
- **Atajo de teclado**: el atajo global se registra a través del entorno de escritorio y se declara
  en configuración; el sistema no lo hardcodea (Principio XIII).
- **Conjunto de frases de referencia**: existe y lo provee el usuario, con la intención y los
  parámetros esperados por frase, incluidas las frases que deben rechazarse. Es la base de
  NFR-014 a NFR-016 y de SC-002 a SC-004. Sin este conjunto, esos criterios no son verificables.
- **Idioma**: la entrada es español rioplatense, admitiendo nombres propios de aplicaciones y
  sitios en inglés. Ningún otro idioma está soportado.
- **Espacios de trabajo**: el rango válido lo determina el entorno de escritorio del usuario y se
  lee del sistema o de la configuración, no se fija en el código.
- **Persistencia del registro**: el registro de actividad se conserva localmente. La política de
  retención y rotación no está especificada y se define en la fase de planificación.
- **Turnos concurrentes**: se asume un único turno activo a la vez. FR-052 exige declarar la
  política ante una activación concurrente; el supuesto es ignorar la nueva activación e informarlo.
