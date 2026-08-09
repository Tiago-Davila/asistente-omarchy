# Feature Specification: Eva — Asistente de Voz Local para Escritorio

**Feature Branch**: `001-asistente-voz-eva`

**Created**: 2026-08-09

**Status**: Clarificada y auditada — lista para `/speckit-plan`

**Input**: Descripción del usuario: asistente de voz local para escritorio Linux que interpreta
comandos hablados en español rioplatense y ejecuta acciones sobre el entorno de escritorio,
sin conexión a internet y sin degradar el rendimiento del sistema.

---

## Alineación con la Constitución (v2.0.0)

Esta especificación está alineada con
[.specify/memory/constitution.md](../../.specify/memory/constitution.md) **v2.0.0**. No hay
conflictos abiertos.

**Superficie de estado.** La versión 1.0.0 de la constitución prohibía toda interfaz gráfica en
fase 1, lo que contradecía a US4 y a FR-034 a FR-039. La enmienda a v2.0.0 redefinió el Principio X,
que ahora permite **una única superficie gráfica mínima de estado** sujeta a cinco condiciones
acumulativas. Cada condición tiene su requisito en esta spec:

| Condición del Principio X | Requisito de esta spec |
| --- | --- |
| 1. Muestra únicamente estado del turno, texto entendido y acción resuelta | FR-034 |
| 2. No toma el foco del teclado en ningún momento | FR-036, FR-038 |
| 3. No tapa ni desplaza la ventana de trabajo | FR-035 |
| 4. Visible solo cuando hay algo que comunicar; desaparece sola | FR-037, NFR-022 |
| 5. El sistema sigue funcional con la superficie deshabilitada | FR-039, FR-033, SC-013 |

Las exclusiones del Principio X —barra de estado permanente, widgets, panel de configuración,
historial navegable, dashboards y cualquier ventana enfocable— están en FR-053 y en
*Fuera de alcance*.

**Modelo de lenguaje.** La v2.0.0 sacó el LLM generativo del alcance de fase 1 (Principio IV y tabla
de presupuestos). Coincide con lo que esta spec declara: FR-013 exige resolución determinística y el
LLM figura en *Fuera de alcance*.

**Presupuestos.** Los de esta spec son iguales o **más estrictos** que los constitucionales, y un
límite más estricto satisface el Principio VI sin necesitar excepción documentada:

| Dimensión | Constitución (piso) | Esta spec (gobierna) |
| --- | --- | --- |
| CPU en reposo | < 3% de un núcleo | **< 1%** (NFR-001) |
| RSS en reposo | < 250 MB | **< 150 MB** (NFR-002) |
| RAM pico por turno | ≤ 1.5 GB | ≤ 1,5 GB (NFR-003) |

---

## Glosario

Términos con significado técnico preciso en esta especificación. Cada uno tiene **un solo** nombre
canónico; los sinónimos listados como prohibidos no deben usarse en este documento ni en los
derivados.

| Término canónico | Definición | Sinónimos prohibidos |
| --- | --- | --- |
| **Catálogo de acciones** | El conjunto cerrado y declarado de acciones que el asistente sabe ejecutar, con sus parámetros tipados. Es el *registry* del Principio III. | "registro" a secas, "registro de acciones" |
| **Bitácora** | El registro estructurado de cada turno: texto, acción, parámetros, resultado y latencia por etapa. Es la observabilidad del Principio XV. | "registro" a secas, "registro de actividad", "log" |
| **Superficie de estado** | La única superficie gráfica permitida en fase 1 por el Principio X. | "interfaz gráfica", "la interfaz", "overlay", "OSD" |
| **Turno** | Una interacción completa, desde la activación hasta que el sistema vuelve a reposo. Incluye la ventana de confirmación si la hubo. | "sesión", "interacción" |
| **Espacio de trabajo** | La subdivisión de escritorios virtuales del gestor de ventanas. El usuario puede decirle "escritorio" o "pestaña" al hablar (AMB-04); el documento usa siempre "espacio de trabajo". | "escritorio", "pestaña", "workspace" |
| **Carga de referencia** | La condición de máquina bajo la que se miden todos los NFR: navegador con 10 pestañas abiertas y un editor de código con un proyecto cargado, sin compilación en curso. | — |
| **Conjunto de frases de referencia** | El conjunto de prueba definido en FR-063 a FR-067, base de NFR-014 a NFR-016. | "corpus", "dataset" |
| **Componente** | Cada una de las etapas intercambiables del pipeline según el Principio VIII: captura de audio, transcripción, resolución de intención, validación, ejecución, notificación y superficie de estado. | "módulo", "servicio" |

---

## Clarifications

### Session 2026-08-09

- Q: ¿Qué acciones del catálogo requieren que el asistente pida confirmación antes de ejecutarlas? → A: Cerrar la ventana activa (FR-021), más toda acción propia (FR-023) que no declare explícitamente `destructiva: false` en configuración.
- Q: Cuando el usuario dice el nombre de un sitio, ¿cómo decide Eva entre abrir el sitio y hacer una búsqueda web? → A: El verbo manda y el sitio debe estar declarado: "abrí X" abre X si está en sitios declarados y si no rechaza; "buscá X" siempre hace búsqueda web con X como término.
- Q: Si el usuario pide abrir una aplicación que ya está abierta, ¿se enfoca la existente o se abre otra instancia? → A: Siempre se enfoca la instancia existente; si hay varias, la más reciente. Nunca se abre una segunda instancia.
- Q: Si se pide abrir una aplicación en un espacio de trabajo determinado y ya está abierta en otro, ¿qué hace el sistema? → A: Mueve la ventana existente al espacio de trabajo pedido y la enfoca ahí.
- Q: ¿Cuánto espera el sistema una confirmación pendiente antes de cancelarla sola? → A: 10 segundos.
- Q: Si el texto encaja con dos acciones distintas del catálogo y ambas tienen parámetros válidos, ¿qué hace el sistema? → A: Rechaza siempre con motivo "intención ambigua", nombrando las acciones candidatas. No desempata por ningún criterio.
- Q: ¿Por qué canal se confirma o cancela una acción destructiva? → A: Por teclado o por voz, indistintamente: tecla dedicada de aceptar y de cancelar, o palabra declarada de aceptación y de cancelación.
- Q: ¿Cuánto permanece el sistema en estado escuchando si no detecta voz? → A: 10 segundos, tras los cuales abandona la activación.
- Q: ¿Cuál es el tope duro de captura cuando sí entra audio y la detección de fin de habla no corta? → A: 20 segundos desde la activación.
- Q: ¿Qué pasa si el usuario cancela cuando la acción ya empezó a ejecutarse? → A: La cancelación se ignora: la acción se completa, se informa que ya estaba en curso, y no se intenta revertir ni abortar a mitad.
- Q: ¿De qué tamaño es el conjunto de frases de referencia? → A: Al menos 300 entradas: unas 20 por acción del catálogo más al menos 80 marcadas "debe rechazarse".

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ejecutar una acción escribiendo texto plano (Priority: P1)

El usuario envía al asistente una frase en texto plano —sin micrófono, sin audio— y el asistente
resuelve la intención, valida los parámetros, ejecuta la acción sobre el escritorio y devuelve el
resultado. Es el núcleo del sistema: el catálogo de acciones, la validación y la ejecución.

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
4. **Given** el asistente corriendo, **When** el usuario envía una frase legible que no corresponde
   a ninguna acción del catálogo, **Then** el asistente rechaza, no ejecuta nada, y reporta el
   motivo "acción desconocida".
5. **Given** el asistente corriendo, **When** el usuario envía "abrí" sin nombrar aplicación,
   **Then** el asistente rechaza por parámetro obligatorio faltante y no inventa un valor.
6. **Given** una misma intención expresada en cinco fraseos rioplatenses distintos (voseo,
   imperativo con enclítico, muletilla inicial), **When** se envía cada uno, **Then** los cinco
   resuelven a la misma acción con los mismos parámetros.
7. **Given** un texto que resuelve a dos acciones del catálogo con parámetros válidos, **When** se
   envía, **Then** el asistente rechaza con motivo "intención ambigua", nombra las dos candidatas y
   no ejecuta ninguna.
8. **Given** la terminal ya abierta en el espacio de trabajo 1, **When** el usuario envía "abrí la
   terminal", **Then** la ventana existente queda enfocada y no existe una segunda ventana de
   terminal.
9. **Given** la terminal ya abierta en el espacio de trabajo 1, **When** el usuario envía "abrí la
   terminal en el escritorio 3", **Then** la ventana existente queda en el espacio 3 y enfocada, y
   no existe una segunda ventana de terminal.
10. **Given** un sitio no declarado en configuración, **When** el usuario envía "abrí Reddit",
    **Then** el asistente rechaza con motivo "sitio no declarado" y no ejecuta una búsqueda web.
11. **Given** el sitio "YouTube" declarado en configuración, **When** el usuario envía "buscá
    YouTube", **Then** el asistente ejecuta una búsqueda web con "YouTube" como término, y no abre
    el sitio declarado.
12. **Given** una frase con un nombre de aplicación en inglés dentro de una oración en español,
    **When** se envía, **Then** resuelve a la acción y los parámetros esperados declarados en el
    conjunto de frases de referencia.
13. **Given** un texto que no contiene ningún token del vocabulario declarado, **When** se envía,
    **Then** el asistente rechaza con motivo "no se entendió", distinto de "acción desconocida".
14. **Given** ninguna ventana activa en el espacio de trabajo actual, **When** el usuario envía
    "cerrá la ventana", **Then** el asistente rechaza con motivo "no hay ventana activa" y no opera
    sobre ninguna otra ventana.

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
2. **Given** el asistente en reposo, **When** el usuario presiona el atajo y no dice nada durante
   10 segundos, **Then** el asistente descarta la activación sin transcribir ni ejecutar nada,
   cierra la captura y vuelve a reposo.
3. **Given** una activación en curso con el usuario hablando, **When** el usuario cancela antes de
   que la acción se ejecute, **Then** no se ejecuta ninguna acción y el asistente vuelve a reposo.
4. **Given** el asistente en reposo y sin activación, **When** se inspecciona el estado del
   dispositivo de audio, **Then** el asistente no tiene el micrófono abierto ni está capturando.
5. **Given** la misma frase de referencia enviada por voz y por texto, **When** ambas se procesan,
   **Then** producen exactamente la misma acción con los mismos parámetros.
6. **Given** una activación con audio continuo entrando y sin fin de habla detectable, **When**
   pasan 20 segundos, **Then** la captura se cierra por tope duro y lo capturado se procesa como
   transcripción parcial.

---

### User Story 3 - Confirmación explícita de acciones destructivas (Priority: P2)

Cuando la acción resuelta está clasificada como destructiva, el asistente muestra la acción concreta
que va a ejecutar y espera una confirmación explícita antes de ejecutarla.

**Why this priority**: el Principio XII lo exige y el canal de voz es ruidoso y ambiguo, por lo que
el costo de un falso positivo destructivo es asimétrico. No es P1 porque el conjunto inicial de
acciones puede entregarse con una sola acción destructiva, el cierre de ventana.

**Independent Test**: se prueba pidiendo una acción declarada destructiva y verificando que nada
se ejecuta hasta que llega una confirmación explícita, y que una no-respuesta o una cancelación
dejan el sistema sin cambios.

**Acceptance Scenarios**:

1. **Given** una acción clasificada como destructiva, **When** el usuario la pide, **Then** el
   asistente muestra la acción concreta con sus parámetros y queda esperando confirmación sin
   ejecutar nada.
2. **Given** el asistente esperando confirmación, **When** el usuario presiona la tecla de aceptar
   **o** dice la palabra de aceptación declarada, **Then** la acción se ejecuta y se reporta el
   resultado.
3. **Given** el asistente esperando confirmación, **When** el usuario presiona la tecla de cancelar
   **o** dice la palabra de cancelación declarada, **Then** la acción no se ejecuta y el asistente
   vuelve a reposo.
4. **Given** el asistente esperando confirmación, **When** pasan 10 segundos sin respuesta,
   **Then** la acción no se ejecuta, el asistente vuelve a reposo y queda disponible para el
   siguiente turno.
5. **Given** una acción propia cuya clasificación de destructividad no está declarada, **When** el
   usuario la pide, **Then** el asistente la trata como destructiva y pide confirmación.
6. **Given** el asistente esperando confirmación, **When** llega cualquier entrada distinta de las
   cuatro declaradas en FR-057, **Then** la acción no se ejecuta, el plazo de 10 segundos no se
   reinicia, y el asistente sigue esperando hasta que expire.
7. **Given** una configuración que declara como palabra de aceptación una palabra que también activa
   una acción del catálogo, **When** el asistente la carga, **Then** rechaza la configuración
   informando la colisión (FR-058).

---

### User Story 4 - Ver el estado del asistente sin cambiar de ventana (Priority: P2)

El asistente muestra en pantalla, de forma mínima y no intrusiva, en qué estado está, qué texto
entendió y qué acción va a ejecutar, sin tomar el foco del teclado ni tapar la ventana de trabajo.

**Why this priority**: sin feedback visible el usuario no sabe si el asistente lo escuchó, y eso
rompe la usabilidad. Es P2 y no P1 porque US1 y US2 pueden entregar feedback por notificación del
sistema mientras tanto. Habilitada por el Principio X de la constitución v2.0.0, y acotada por sus
cinco condiciones.

**Independent Test**: se prueba activando el asistente mientras se escribe en un editor y
verificando, por consulta al gestor de ventanas, que el foco no cambió y que la superficie aparece y
desaparece sola.

**Acceptance Scenarios**:

1. **Given** el usuario escribiendo en un editor, **When** activa el asistente, **Then** la
   superficie de estado aparece, la ventana enfocada según el gestor de ventanas sigue siendo el
   editor, y los caracteres tecleados llegan al editor.
2. **Given** el asistente activo, **When** cambia de estado (escuchando → procesando → ejecutando),
   **Then** la superficie refleja cada cambio de estado.
3. **Given** un turno terminado, **When** pasan 3 segundos (NFR-022), **Then** la superficie deja de
   existir en el árbol de ventanas del gestor, sin intervención del usuario.
4. **Given** el asistente esperando confirmación, **When** el usuario confirma o cancela desde el
   teclado, **Then** la acción se resuelve y la ventana enfocada según el gestor de ventanas no
   cambió en ningún momento.
5. **Given** el asistente en reposo, **When** se consulta el árbol de ventanas del gestor, **Then**
   no existe ninguna ventana perteneciente al asistente.
6. **Given** la superficie de estado deshabilitada en configuración, **When** se ejecuta el catálogo
   completo de acciones, **Then** todas se ejecutan correctamente y el resultado de cada turno queda
   disponible por los canales no gráficos de FR-033.

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
5. **Given** una configuración con entradas inválidas, **When** el asistente la carga, **Then**
   reporta el error de forma visible, deja fuera del catálogo únicamente las entradas inválidas, y
   sigue operando con las válidas.
6. **Given** un turno en curso, **When** la configuración se recarga a mitad del turno, **Then** el
   turno se completa contra el catálogo vigente al resolver la intención, y la configuración nueva
   se aplica recién a partir del turno siguiente.

---

### User Story 6 - Entender por qué el asistente hizo o no hizo algo (Priority: P3)

Cada interacción queda registrada en la bitácora con el texto de entrada, la acción resuelta, los
parámetros, el resultado y el tiempo de cada etapa, y el usuario puede consultarla.

**Why this priority**: el Principio XV lo exige y es la única forma de ajustar reglas y
presupuestos de latencia con datos. Es P3 porque no bloquea el uso diario, pero sin esto no se
puede mejorar el sistema ni diagnosticar un rechazo inesperado.

**Independent Test**: se prueba ejecutando una serie de turnos —exitosos, rechazados y fallidos— y
verificando que cada uno dejó una entrada de bitácora con todos los campos exigidos.

**Acceptance Scenarios**:

1. **Given** un turno ejecutado con éxito, **When** se consulta la bitácora, **Then** la entrada
   contiene el texto de entrada, la acción, los parámetros, el resultado y la latencia por etapa.
2. **Given** un turno rechazado, **When** se consulta la bitácora, **Then** la entrada contiene el
   texto y el código de clase de rechazo, de las cinco de FR-031.
3. **Given** un turno que excedió el presupuesto de latencia, **When** se consulta la bitácora,
   **Then** la entrada identifica la etapa que consumió el tiempo y marca el exceso.

---

### User Story 7 - El asistente sigue vivo después de un fallo (Priority: P2)

Ante un fallo de cualquier componente —micrófono ocupado, cambio de dispositivo, error del gestor de
ventanas, configuración inválida— el daemon sigue corriendo, informa el fallo y queda operativo para
el turno siguiente sin reiniciarse.

**Why this priority**: el Principio V lo exige y es lo que separa un accesorio usable de uno que hay
que reiniciar a mano. Es P2 porque el sistema puede demostrarse sin ella, pero sin esto no es usable
durante una jornada real.

**Independent Test**: se prueba inyectando cada fallo externo por separado y verificando que el
identificador de proceso del daemon no cambia y que el turno siguiente se completa correctamente.

**Acceptance Scenarios**:

1. **Given** el micrófono ocupado por otra aplicación, **When** el usuario activa el asistente,
   **Then** el turno termina con error de dispositivo ocupado, el proceso del daemon sigue vivo, y
   el turno siguiente por texto plano se completa correctamente.
2. **Given** un turno con el gestor de ventanas devolviendo error, **When** se intenta ejecutar la
   acción, **Then** el turno termina con motivo "fallo al ejecutar", el proceso sigue vivo, y la
   bitácora conserva el intento.
3. **Given** el dispositivo de audio cambiando en medio de una captura, **When** el turno falla,
   **Then** el asistente vuelve a reposo sin reiniciarse y la activación siguiente captura por el
   dispositivo nuevo.
4. **Given** un turno en curso, **When** llega una segunda activación, **Then** se ignora e informa,
   y el turno en curso se completa sin alteración.
5. **Given** cualquier turno terminado en error, **When** pasan 3 segundos, **Then** el asistente
   está en reposo y acepta una activación nueva.

---

### Edge Cases

Cada caso borde tiene comportamiento esperado declarado y verificable.

| # | Caso | Comportamiento esperado |
|---|------|-------------------------|
| EC-01 | El micrófono está ocupado por otra aplicación | El asistente no ejecuta ninguna acción, reporta el error de dispositivo ocupado y vuelve a reposo. No reintenta en bucle. |
| EC-02 | No se detecta habla en la activación | A los 10 segundos sin habla detectada se abandona el turno (FR-061): no se transcribe, no se ejecuta nada, se cierra la captura y se vuelve a reposo, con la señal sonora de turno vacío (FR-032). |
| EC-03 | Texto legible que no corresponde a ninguna acción del catálogo | Rechazo con motivo "acción desconocida". Nunca se ejecuta la acción más parecida. |
| EC-04 | El usuario nombra una aplicación no instalada o no declarada | Rechazo con motivo distinguible: "no declarada" (falta en configuración) o "no disponible" (declarada pero ausente del sistema). |
| EC-05 | El usuario indica un espacio de trabajo fuera del rango válido | Rechazo por parámetro fuera de rango, indicando el rango válido. No se recorta ni se aproxima al valor más cercano. |
| EC-06 | La aplicación pedida ya está abierta | Se enfoca la instancia existente, la más reciente si hay varias, y nunca se abre una segunda (FR-014). El resultado se reporta, sin fallar en silencio. |
| EC-07 | Llega una activación con un turno ya en curso | Se ignora y se informa que hay un turno en proceso (FR-052), en todos los estados no-reposo. El turno en curso no se altera. La cancelación tiene su propio canal (FR-057), así que ignorar la activación no deja al usuario sin salida. |
| EC-08 | El gestor de ventanas no responde o devuelve error | El turno termina con motivo "fallo al ejecutar", el daemon sigue vivo, y la bitácora conserva el intento. |
| EC-09 | El sistema está bajo carga alta y se excede el presupuesto de latencia | La acción se completa si ya fue validada, el exceso se marca en la bitácora, y el usuario recibe feedback de que el turno tardó más de lo previsto. Nunca se aborta una acción ya iniciada. |
| EC-10 | La transcripción devuelve texto parcial o cortado | Se trata como cualquier texto: si resuelve a una acción con todos sus parámetros válidos, se ejecuta; si no, se rechaza. No se completa ni se adivina el texto faltante. |
| EC-11 | El usuario cancela mientras espera confirmación | La acción no se ejecuta, el sistema vuelve a reposo, y la bitácora deja constancia de la cancelación. |
| EC-12 | El dispositivo de audio cambia en medio de la captura | El turno termina en error de captura sin ejecutar nada; el asistente se recupera y queda operativo para la activación siguiente sin reiniciar el daemon. |
| EC-13 | El usuario pide abrir un sitio no declarado en configuración | Rechazo con motivo "sitio no declarado" (FR-016). No se sustituye por una búsqueda web, aunque el nombre sea buscable. |
| EC-14 | Se pide abrir en un espacio de trabajo una aplicación ya abierta en otro | La ventana existente —la más reciente si hay varias— se mueve al espacio pedido y se enfoca ahí (FR-015). No se abre una segunda instancia ni se descarta el espacio pedido. |
| EC-15 | El texto resuelve a dos o más acciones del catálogo con parámetros válidos | Rechazo con motivo "intención ambigua", nombrando las candidatas (FR-056). No se desempata por ningún criterio ni se ejecuta ninguna. |
| EC-16 | Entra audio continuo y la detección de fin de habla nunca corta | La captura termina por tope duro a los 20 segundos (FR-062). Lo capturado se trata como transcripción parcial (EC-10). |
| EC-17 | El usuario cancela con la acción ya en ejecución | La cancelación se ignora, la acción se completa y se informa que ya estaba en curso (FR-003). No se revierte ni se aborta a mitad. La bitácora deja constancia de la cancelación tardía. |
| EC-18 | Se pide operar sobre la ventana activa y no hay ninguna | Rechazo con motivo "no hay ventana activa" (FR-020, FR-021, FR-068, FR-069). No se opera sobre otra ventana. |
| EC-19 | Se pide control de volumen o medios y no hay dispositivo de salida disponible | Rechazo con motivo "fallo al ejecutar", indicando la ausencia del dispositivo. El daemon sigue vivo. |
| EC-20 | La configuración se recarga con un turno en curso | El turno se completa contra el catálogo vigente al momento de resolver la intención. La configuración nueva se aplica desde el turno siguiente (FR-044). |
| EC-21 | La configuración tiene entradas inválidas al arrancar | El daemon arranca, deja fuera del catálogo solo las entradas inválidas, reporta el error de forma visible y opera con las válidas. Si el archivo entero es ilegible, arranca con el catálogo vacío y lo reporta. Nunca arranca en silencio ni se niega a arrancar (FR-045). |
| EC-22 | Una acción propia declarada por el usuario falla al ejecutarse | El turno termina con motivo "fallo al ejecutar", nombrando la acción propia. El daemon sigue vivo y la bitácora conserva el intento. |
| EC-23 | El procesamiento no termina dentro de su presupuesto | A los 10 segundos el procesamiento aborta con motivo "tiempo agotado", sin ejecutar la acción (FR-070). |

---

## Máquina de estados

Los seis estados del asistente y sus transiciones. Cada celda declara qué hace el sistema al recibir
esa entrada en ese estado. `ignora e informa` significa que el estado no cambia y el usuario recibe
feedback de por qué no pasó nada.

| Estado | Activación (atajo) | Texto plano | Aceptar | Cancelar | Vencimiento |
| --- | --- | --- | --- | --- | --- |
| **reposo** | → escuchando | → procesando | ignora | ignora | — |
| **escuchando** | ignora e informa | ignora e informa | ignora | → reposo | 10 s sin habla → reposo (FR-061); 20 s tope → procesando con texto parcial (FR-062) |
| **procesando** | ignora e informa | ignora e informa | ignora | → reposo | 10 s → error "tiempo agotado" (FR-070) |
| **esperando confirmación** | ignora e informa | ignora e informa | → ejecutando | → reposo | 10 s → reposo sin ejecutar (FR-055) |
| **ejecutando** | ignora e informa | ignora e informa | ignora | ignora e informa (FR-003) | — |
| **error** | ignora e informa | ignora e informa | ignora | ignora | 3 s → reposo (FR-071) |

Transiciones adicionales, no gobernadas por una entrada del usuario:

- `procesando` → `esperando confirmación` cuando la acción resuelta está clasificada como destructiva (FR-028).
- `procesando` → `ejecutando` cuando la acción resuelta no es destructiva y sus parámetros son válidos.
- `procesando` → `error` ante cualquiera de las cinco clases de rechazo de FR-031.
- `ejecutando` → `reposo` al completarse la acción con éxito.
- `ejecutando` → `error` si la ejecución falla.

Ningún estado carece de salida acotada en el tiempo (NFR-020, NFR-023).

---

## Requirements *(mandatory)*

### Functional Requirements

**Activación y captura**

- **FR-001**: El sistema MUST permitir al usuario iniciar un turno mediante un atajo de teclado global.
- **FR-002**: El sistema MUST capturar audio únicamente durante un turno activado, y MUST NOT capturar audio en ningún otro momento.
- **FR-003**: El sistema MUST permitir al usuario cancelar un turno en curso antes de que la acción se ejecute. Una cancelación recibida **después** de iniciada la ejecución MUST ignorarse: el sistema MUST completar la acción, MUST informar al usuario que ya estaba en curso, y MUST NOT intentar revertirla ni abortarla a mitad.
- **FR-004**: El sistema MUST liberar el dispositivo de audio al terminar el turno, sea cual sea el resultado. Verificable inspeccionando el estado del dispositivo tras cada clase de resultado (US2-4).

**Transcripción**

- **FR-005**: El sistema MUST transcribir a texto en español el audio capturado durante el turno.
- **FR-006**: El sistema MUST detectar por su cuenta el fin del habla y dejar de capturar, sin acción del usuario.
- **FR-007**: El sistema MUST descartar los turnos en los que no se detectó habla, sin ejecutar ninguna acción.
- **FR-061**: Si tras la activación no se detecta habla durante 10 segundos consecutivos, el sistema MUST abandonar el turno sin transcribir ni ejecutar nada, cerrar la captura y volver a reposo.
- **FR-062**: La captura MUST terminar como máximo a los 20 segundos desde la activación, aunque siga entrando audio y no se haya detectado el fin del habla. Lo capturado MUST tratarse como transcripción parcial, con el comportamiento de EC-10.

**Resolución de intención**

- **FR-008**: El sistema MUST convertir el texto en una acción concreta con parámetros tipados, elegida del catálogo de acciones.
- **FR-009**: El sistema MUST rechazar y avisar cuando el texto no corresponde a ninguna acción del catálogo, y MUST NOT ejecutar una acción aproximada.
- **FR-010**: El sistema MUST rechazar o solicitar el dato cuando falta un parámetro obligatorio, y MUST NOT inventar valores por defecto no declarados.
- **FR-011**: El sistema MUST resolver a la misma acción los fraseos de una misma intención declarados en el conjunto de frases de referencia, incluyendo voseo, imperativos con pronombre enclítico y muletillas iniciales.
- **FR-012**: Para cada aplicación y sitio declarados con nombre en inglés, el sistema MUST resolver la acción y los parámetros esperados cuando ese nombre aparece dentro de una frase en español, verificado sobre las entradas del conjunto de referencia que incluyen nombres en inglés (FR-064).
- **FR-013**: El sistema MUST resolver las intenciones de forma determinística, sin depender de un modelo generativo (ver *Fuera de alcance*).
- **FR-056**: Si el texto resuelve a más de una acción del catálogo con parámetros válidos, el sistema MUST rechazar con motivo "intención ambigua" y MUST nombrar las acciones candidatas en el feedback. El sistema MUST NOT desempatar por puntaje de coincidencia, orden de declaración, especificidad, frecuencia de uso ni ningún otro criterio implícito.
- **FR-072**: El sistema MUST distinguir dos clases de rechazo por reconocimiento fallido: "no se entendió" MUST aplicar cuando el texto está vacío o no contiene ningún token del vocabulario declarado; "acción desconocida" MUST aplicar cuando el texto contiene tokens declarados pero no coincide con ninguna acción del catálogo.

**Acciones disponibles**

- **FR-014**: El sistema MUST poder abrir una aplicación declarada en configuración. Si la aplicación ya tiene una ventana abierta, el sistema MUST enfocar la instancia existente —la más reciente si hay varias— y MUST NOT abrir una segunda instancia.
- **FR-015**: El sistema MUST poder abrir una aplicación declarada en un espacio de trabajo indicado. Si la aplicación ya está abierta en otro espacio, el sistema MUST mover al espacio indicado la ventana existente —la más reciente si hay varias, con el mismo criterio de FR-014— y enfocarla ahí; MUST NOT abrir una segunda instancia ni descartar el espacio pedido.
- **FR-016**: El sistema MUST poder abrir una URL en el navegador declarado. La intención de apertura se resuelve **únicamente por el verbo**: si el sitio nombrado no está declarado en configuración, el sistema MUST rechazar con motivo "sitio no declarado" y MUST NOT sustituir la apertura por una búsqueda web.
- **FR-017**: El sistema MUST poder abrir una URL en un espacio de trabajo indicado. Si el navegador ya tiene una ventana abierta en otro espacio, aplica la regla de FR-015: la URL se abre en el navegador existente y su ventana se mueve al espacio indicado y se enfoca. **Consecuencia aceptada**: el resto de las pestañas abiertas en esa ventana se mueven con ella.
- **FR-018**: El sistema MUST poder realizar una búsqueda web con un texto dictado por el usuario. La intención de búsqueda MUST resolverse siempre a búsqueda web con el texto dictado como término, aunque ese texto coincida con un sitio declarado.
- **FR-073**: El término de búsqueda de FR-018 MUST validarse como texto plano de entre 1 y 200 caracteres y MUST escaparse antes de componer la URL de búsqueda. MUST NOT interpretarse como parte de la estructura de la URL ni como argumento de línea de comandos.
- **FR-019**: El sistema MUST poder cambiar al espacio de trabajo indicado. Si el espacio está fuera del rango válido, MUST rechazar por parámetro fuera de rango (EC-05).
- **FR-020**: El sistema MUST poder mover la ventana activa a un espacio de trabajo indicado. Si no hay ventana activa, MUST rechazar con motivo "no hay ventana activa" y MUST NOT operar sobre otra ventana.
- **FR-021**: El sistema MUST poder cerrar la ventana activa. Es una acción **destructiva** (FR-028). Si no hay ventana activa, MUST rechazar con motivo "no hay ventana activa".
- **FR-068**: El sistema MUST poder alternar el estado de pantalla completa de la ventana activa. No es destructiva. Si no hay ventana activa, MUST rechazar con motivo "no hay ventana activa".
- **FR-069**: El sistema MUST poder alternar el estado flotante de la ventana activa. No es destructiva. Si no hay ventana activa, MUST rechazar con motivo "no hay ventana activa".
- **FR-022**: El sistema MUST poder controlar volumen y reproducción de medios. Si no hay dispositivo de salida disponible, MUST rechazar con motivo "fallo al ejecutar" indicando la ausencia del dispositivo.
- **FR-023**: El sistema MUST poder ejecutar una acción definida por el usuario, tomada exclusivamente de una lista declarada en configuración. Si la acción propia falla al ejecutarse, el turno MUST terminar con motivo "fallo al ejecutar" nombrando la acción, sin terminar el proceso del asistente.

**Validación y seguridad**

- **FR-024**: El sistema MUST validar cada parámetro antes de ejecutar, comprobando tipo, rango y pertenencia al conjunto de valores permitidos.
- **FR-025**: El sistema MUST NOT ejecutar comandos arbitrarios provenientes de la transcripción ni de ninguna otra entrada del usuario.
- **FR-026**: El sistema MUST exigir confirmación explícita del usuario antes de ejecutar una acción clasificada como destructiva, mostrando previamente la acción concreta y sus parámetros.
- **FR-027**: El sistema MUST NOT inferir ni asumir la confirmación a partir del contexto, del turno anterior ni del silencio del usuario.
- **FR-028**: El sistema MUST clasificar como destructivas exactamente estas acciones, y MUST pedir confirmación antes de ejecutarlas: cerrar la ventana activa (FR-021), y toda acción propia (FR-023) que no declare explícitamente `destructiva: false` en configuración. El resto del catálogo base (FR-014 a FR-020, FR-022, FR-068, FR-069) MUST ejecutarse sin confirmación.
- **FR-074**: La configuración MUST poder elevar a destructiva cualquier acción del catálogo base, y MUST NOT poder rebajar a no destructiva ninguna acción listada como destructiva en FR-028.
- **FR-054**: La configuración MUST permitir declarar la destructividad de cada acción propia. Ante ausencia de esa declaración, el sistema MUST tratar la acción como destructiva.
- **FR-055**: El sistema MUST cancelar automáticamente una confirmación pendiente tras 10 segundos sin respuesta del usuario, sin ejecutar la acción y volviendo a reposo.
- **FR-057**: La confirmación y la cancelación MUST poder realizarse indistintamente por dos canales, ambos activos durante toda la ventana de FR-055: (a) **teclado**, con una tecla dedicada para aceptar y otra para cancelar; (b) **voz**, con una palabra de aceptación y una de cancelación. Las cuatro entradas MUST declararse en configuración.
- **FR-058**: Las palabras de confirmación y cancelación por voz MUST NOT coincidir con ninguna palabra que active una acción del catálogo ni con ningún alias declarado. El sistema MUST rechazar la configuración que las haga coincidir (FR-045).
- **FR-059**: Cualquier entrada recibida durante la ventana de confirmación que no sea una de las cuatro declaradas en FR-057 MUST tratarse como no-respuesta: MUST NOT ejecutar la acción y MUST NOT reiniciar el plazo de 10 segundos.
- **FR-060**: La ventana de confirmación forma parte del turno activado a efectos de FR-002: el sistema MUST mantener la captura de audio abierta mientras espera confirmación, y MUST cerrarla al resolverse o expirar la ventana.

**Feedback al usuario**

- **FR-029**: El sistema MUST exponer de forma consultable su estado actual, dentro del conjunto cerrado de seis: reposo, escuchando, procesando, ejecutando, esperando confirmación, error.
- **FR-030**: El sistema MUST mostrar al usuario el texto que entendió y la acción que va a ejecutar, antes o durante la ejecución.
- **FR-031**: El sistema MUST emitir, en cada turno no exitoso, un **código de clase de error** de este conjunto cerrado de cinco valores: `no_entendido`, `accion_desconocida`, `intencion_ambigua`, `parametro_invalido`, `fallo_al_ejecutar`. El mensaje asociado MUST nombrar la acción o el parámetro involucrado.
- **FR-032**: El sistema MUST emitir una señal sonora breve al terminar el turno, con tres variantes distinguibles: éxito, error y turno vacío. Las tres MUST durar menos de 300 ms y MUST diferir entre sí en frecuencia fundamental en al menos una tercera mayor.
- **FR-033**: El sistema MUST reportar el resultado de cada turno por un canal no gráfico —notificación del sistema, salida estándar y código de retorno—, de modo que el resultado sea observable sin la superficie de estado.

**Superficie de estado**

- **FR-034**: Cuando la superficie de estado está habilitada en configuración, el sistema MUST mostrar **una única** superficie, y esta MUST mostrar únicamente el estado del turno, el texto entendido y la acción resuelta. La superficie MUST poder deshabilitarse desde configuración.
- **FR-035**: La superficie MUST ocupar una porción reducida de la pantalla y MUST NOT tapar ni desplazar la ventana en la que el usuario está trabajando.
- **FR-036**: La superficie MUST NOT tomar el foco del teclado en ningún momento, verificable consultando la ventana enfocada del gestor de ventanas antes y después de que aparezca.
- **FR-037**: La superficie MUST ser visible solo cuando hay algo que comunicar y MUST desaparecer sola al terminar el turno, dentro del plazo de NFR-022.
- **FR-038**: El usuario MUST poder confirmar o cancelar una acción pendiente sin que cambie la ventana enfocada del gestor de ventanas.
- **FR-039**: El sistema MUST seguir siendo completamente funcional con la superficie deshabilitada, degradando el feedback a los canales de FR-033.
- **FR-053**: El sistema MUST NOT introducir ninguna otra superficie gráfica: quedan excluidas barra de estado permanente, widgets, panel de configuración, historial navegable, dashboards y cualquier ventana que el usuario pueda enfocar.

**Entrada por texto equivalente**

- **FR-040**: El sistema MUST aceptar comandos como texto plano y producir exactamente el mismo resultado que por voz.
- **FR-041**: La resolución de intención, la validación de parámetros, la clasificación de destructividad, el rechazo y la ejecución de acciones MUST ser ejercitables sin micrófono ni dispositivos de audio presentes. El feedback sonoro (FR-032) y la superficie de estado (FR-034) MUST verificarse por separado, por sus propios canales observables.

**Configuración**

- **FR-042**: El usuario MUST poder declarar en configuración qué aplicaciones, sitios y acciones propias conoce el asistente, sin modificar código.
- **FR-043**: El usuario MUST poder declarar sinónimos y alias que resuelvan a una misma acción.
- **FR-044**: El sistema MUST poder aplicar cambios de configuración sin reiniciar la máquina. Un turno en curso MUST completarse contra el catálogo vigente al momento de resolver la intención; la configuración nueva MUST aplicarse a partir del turno siguiente.
- **FR-045**: El sistema MUST reportar de forma visible los errores de configuración y MUST NOT operar en silencio con una configuración inválida. Ante entradas inválidas MUST dejarlas fuera del catálogo y seguir operando con las válidas; si el archivo entero es ilegible MUST arrancar con el catálogo vacío y reportarlo. MUST NOT negarse a arrancar.

**Bitácora**

- **FR-046**: El sistema MUST registrar en la bitácora cada turno con: texto de entrada, acción resuelta, parámetros, resultado y tiempo de cada etapa.
- **FR-047**: Para todo turno no exitoso, la entrada de bitácora MUST contener el código de clase de error de FR-031 y el identificador de la etapa en la que se produjo, de modo que la causa sea determinable sin reproducir el turno.
- **FR-048**: La bitácora MUST permanecer en la máquina del usuario y MUST NOT enviarse a ningún destino externo.

**Conjunto de frases de referencia**

- **FR-063**: El conjunto de frases de referencia MUST contener al menos 300 entradas, con al menos 20 fraseos distintos por cada acción del catálogo y al menos 80 entradas marcadas "debe rechazarse".
- **FR-064**: De los fraseos de cada acción, al menos 5 MUST estar en registro rioplatense con voseo o pronombre enclítico, y al menos 1 MUST incluir un nombre de aplicación o sitio en inglés dentro de la frase (FR-012).
- **FR-065**: Las 80 entradas de rechazo MUST cubrir las cinco clases de FR-031, con al menos 10 entradas por clase.
- **FR-066**: Cada entrada MUST declarar el texto de la frase y, o bien la acción y los parámetros esperados, o bien la marca "debe rechazarse" con su clase de rechazo esperada.
- **FR-067**: La precisión de NFR-014 a NFR-016 MUST medirse ejecutando el conjunto completo por la vía de texto plano (FR-040) y comparando, entrada por entrada, la acción y los parámetros resueltos contra los esperados.

**Robustez**

- **FR-049**: El fallo de cualquier componente —según la definición del glosario— MUST NOT terminar el proceso del asistente. Verificable comprobando que el identificador de proceso no cambia tras inyectar cada fallo.
- **FR-050**: Si un turno falla después de haber iniciado la ejecución, el sistema MUST dejar el escritorio en uno de dos estados observables: la acción aplicada por completo, o ninguna parte de la acción aplicada. MUST NOT dejar estados intermedios, como una ventana movida pero no enfocada.
- **FR-051**: El sistema MUST recuperarse de un cambio de dispositivo de audio y quedar operativo para el turno siguiente sin reiniciar.
- **FR-052**: Cuando llega una activación con un turno en curso, el sistema MUST ignorarla e informar al usuario que hay un turno en proceso, en todos los estados distintos de reposo. La cancelación dispone de su propio canal (FR-057).
- **FR-070**: El procesamiento MUST abortar a los 10 segundos, terminando el turno con motivo "tiempo agotado" sin ejecutar la acción.
- **FR-071**: El estado `error` MUST durar como máximo 3 segundos, tras los cuales el sistema MUST volver a reposo.

### Key Entities

- **Turno**: una interacción completa de punta a punta. Atributos: identificador, momento de inicio, texto de entrada (transcripto o escrito), intención resuelta, acción elegida, parámetros, estado final (ejecutado / rechazado / cancelado / fallido), código de clase de error si no se ejecutó, etapa en la que falló, y duración por etapa.
- **Acción del catálogo**: una capacidad que el asistente sabe ejecutar. Atributos: nombre canónico, alias y sinónimos, lista de parámetros con su tipo y obligatoriedad, valores permitidos por parámetro, y clasificación de destructividad.
- **Parámetro**: un dato tipado que una acción necesita. Atributos: nombre, tipo, obligatoriedad, rango o conjunto de valores válidos.
- **Configuración del usuario**: la declaración de entorno. Contiene aplicaciones conocidas, sitios conocidos, acciones propias con su destructividad, alias, el mapeo de atajos, las cuatro entradas de confirmación de FR-057, y el interruptor de la superficie de estado.
- **Estado del asistente**: el valor observable del sistema en un momento dado, dentro del conjunto cerrado de seis: reposo, escuchando, procesando, ejecutando, esperando confirmación, error. Sus transiciones están en *Máquina de estados*.
- **Frase de referencia**: un elemento del conjunto de prueba definido en FR-063 a FR-067. Atributos: texto de la frase en español rioplatense; y o bien la intención y los parámetros esperados, o bien la marca "debe rechazarse" junto con la clase de rechazo esperada (una de las cinco de FR-031).

### Non-Functional Requirements

Salvo indicación explícita en contrario, **todos los NFR de consumo de recursos y latencia se miden
bajo la carga de referencia** definida en el glosario.

**Consumo de recursos**

- **NFR-001**: En reposo, el conjunto de procesos del asistente MUST consumir menos del 1% de un núcleo, medido como promedio sostenido durante 10 minutos sin activaciones.
- **NFR-002**: En reposo, el conjunto de procesos del asistente MUST mantener un RSS agregado menor a 150 MB, medido como máximo observado en una ventana de 10 minutos sin activaciones, contabilizando las páginas compartidas una sola vez.
- **NFR-003**: Durante un turno completo, el consumo total de memoria del asistente y sus procesos auxiliares MUST NOT superar 1,5 GB en su pico, medido con muestreo de al menos 10 Hz.
- **NFR-004**: Durante el procesamiento de un turno, el conjunto de procesos del asistente MUST NOT superar el 75% de la capacidad total de CPU —equivalente a 3 de los 4 núcleos—, medido como promedio en ventanas de 200 ms. Ningún núcleo MUST quedar por encima del 10% de utilización atribuible al asistente durante más de dos ventanas consecutivas.
- **NFR-005**: Mientras el asistente procesa un turno bajo la carga de referencia, el tiempo entre cuadros de la ventana enfocada MUST mantenerse por debajo de 33 ms en el percentil 99, y el asistente MUST NOT provocar más de un cuadro descartado por turno.
- **NFR-021**: El plan de la feature MUST declarar, por cada componente pesado, si es residente o de carga bajo demanda, con su costo de memoria y su tiempo de carga medidos. La suma de los residentes MUST caber en NFR-002, y el tiempo de carga de los componentes bajo demanda MUST estar incluido dentro del presupuesto de NFR-006.

**Latencia**

- **NFR-006**: Desde el fin del habla hasta la ejecución de la acción, el tiempo MUST ser menor a 2 segundos en el percentil 50, medido sobre los turnos que **no** requieren confirmación.
- **NFR-007**: Ese mismo tiempo MUST ser menor a 3,5 segundos en el percentil 95, con el mismo alcance de medición que NFR-006.
- **NFR-008**: El cambio de estado MUST reflejarse de forma consultable en menos de 100 milisegundos desde la activación, medido bajo la carga de referencia.
- **NFR-009**: El plan de la feature MUST declarar un presupuesto de tiempo por cada etapa del pipeline. La suma de los presupuestos por etapa MUST ser menor o igual a 2 s para el percentil 50 y a 3,5 s para el percentil 95, de modo que NFR-006 y NFR-007 se satisfagan por construcción. El tiempo real por etapa MUST quedar registrado en la bitácora de cada turno.
- **NFR-024**: En los turnos con confirmación, el presupuesto MUST aplicarse en dos tramos: fin del habla → presentación de la confirmación, con los techos de NFR-006 y NFR-007; y confirmación del usuario → ejecución, con un techo de 500 ms. El tiempo de respuesta humana MUST quedar excluido de ambos tramos.

**Privacidad y operación offline**

- **NFR-010**: Ningún audio, transcripción, comando ni dato de uso MUST salir de la máquina.
- **NFR-011**: Ningún componente del asistente MUST establecer conexiones de red. Entregar una URL o un término de búsqueda al navegador declarado **no** constituye una dependencia de red del asistente; el uso posterior de la red es del navegador.

**Confiabilidad**

- **NFR-012**: El asistente MUST sobrevivir sin reiniciarse a: micrófono ocupado, cambio de dispositivo de audio, error del gestor de ventanas, ausencia de dispositivo de salida y configuración inválida.
- **NFR-013**: Tras cualquier fallo de turno, el asistente MUST volver al estado de reposo dentro de los 3 segundos de FR-071 y quedar disponible para el turno siguiente.

**Precisión**

- **NFR-014**: Sobre el conjunto de frases de referencia, el sistema MUST resolver la intención y los parámetros correctos en al menos el 90% de las entradas.
- **NFR-015**: Sobre ese mismo conjunto, la tasa de ejecución de una acción distinta de la pedida MUST ser menor al 1%. Un fallo MUST manifestarse como rechazo, no como acción equivocada.
- **NFR-016**: El sistema MUST rechazar el 100% de las entradas marcadas "debe rechazarse", con la clase de rechazo esperada declarada en FR-066.

**Usabilidad**

- **NFR-017**: El usuario MUST poder ejecutar cada acción del catálogo sin memorizar una sintaxis exacta: al menos 3 fraseos naturales distintos por acción MUST resolver correctamente.
- **NFR-018**: El estado actual del asistente MUST estar disponible por un canal consultable en menos de 100 ms, sin que el usuario cambie la ventana enfocada.
- **NFR-019**: Una confirmación pendiente MUST expirar a los 10 segundos sin respuesta. El asistente MUST NOT quedar bloqueado en espera de confirmación más allá de ese plazo.
- **NFR-020**: El estado `escuchando` MUST terminar en 20 segundos como máximo bajo cualquier condición de entrada, incluidas ausencia total de habla (10 s, FR-061) y audio continuo sin fin de habla detectado (20 s, FR-062).

**Salidas acotadas y superficie**

- **NFR-022**: La superficie de estado MUST desaparecer a los 3 segundos de terminado el turno, verificable por consulta al árbol de ventanas del gestor.
- **NFR-023**: Ningún estado del asistente MUST carecer de salida acotada en el tiempo. Los topes son: `escuchando` 20 s (NFR-020), `procesando` 10 s (FR-070), `esperando confirmación` 10 s (NFR-019, FR-055), `error` 3 s (FR-071).

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El usuario ejecuta cualquier acción no destructiva del catálogo hablando, sin sacar las manos del teclado y sin cambiar de ventana, en menos de 3,5 segundos desde que termina de hablar, en el 95% de los intentos bajo la carga de referencia.
- **SC-002**: El asistente resuelve correctamente la intención y los parámetros en al menos el 90% de las 300 entradas del conjunto de referencia.
- **SC-003**: Menos del 1% de las entradas del conjunto de referencia producen una acción distinta de la pedida.
- **SC-004**: El 100% de las entradas marcadas "debe rechazarse" son rechazadas con la clase de rechazo esperada.
- **SC-005**: El asistente corre durante una jornada de trabajo completa consumiendo en reposo menos del 1% de un núcleo y menos de 150 MB, medido a lo largo de la jornada bajo la carga de referencia.
- **SC-006**: Durante el procesamiento de un turno, la ventana enfocada mantiene un tiempo entre cuadros por debajo de 33 ms en el percentil 99, con como máximo un cuadro descartado por turno.
- **SC-007**: Con la interfaz de red inhabilitada, el asistente completa todas sus etapas hasta la entrega al navegador sin errores propios y sin intentos de conexión atribuibles al asistente, durante una sesión completa de uso.
- **SC-008**: El asistente completa una jornada de trabajo sin reiniciarse, incluyendo al menos un evento de micrófono ocupado y un cambio de dispositivo de audio, verificado porque el identificador de proceso no cambia.
- **SC-009**: El usuario agrega una aplicación nueva y un alias a su configuración y los usa correctamente en menos de 2 minutos, sin tocar código ni reiniciar la máquina.
- **SC-010**: Para el 100% de los turnos no exitosos, la entrada de bitácora contiene el código de clase de error y el identificador de la etapa donde se produjo, sin necesidad de reproducir el turno.
- **SC-011**: El 100% de las acciones clasificadas como destructivas quedan sin ejecutarse hasta recibir confirmación explícita por uno de los cuatro canales de FR-057.
- **SC-012**: El conjunto completo de pruebas de intención, validación, rechazo y ejecución corre de punta a punta en una máquina sin dispositivos de audio.
- **SC-013**: El asistente ejecuta correctamente el catálogo completo de acciones con la superficie de estado deshabilitada, usando solo los canales no gráficos de FR-033.
- **SC-014**: Las 30 combinaciones de la tabla de *Máquina de estados* producen la transición declarada, verificado por prueba automatizada sobre la vía de texto plano.

---

## Trazabilidad

Cada requisito funcional traza a una historia de usuario y a los escenarios que lo ejercitan.

| Requisitos | Historia | Escenarios |
| --- | --- | --- |
| FR-008 a FR-013, FR-056, FR-072 | US1 | US1-4, US1-5, US1-6, US1-7, US1-12, US1-13 |
| FR-014 a FR-023, FR-068, FR-069, FR-073 | US1 | US1-1, US1-2, US1-3, US1-8 a US1-11, US1-14 |
| FR-024, FR-025 | US1 | US1-5, US1-14 |
| FR-040, FR-041 | US1 | US1-1 a US1-14, US2-5 |
| FR-001 a FR-007, FR-061, FR-062 | US2 | US2-1 a US2-6 |
| FR-026 a FR-028, FR-054, FR-055, FR-057 a FR-060, FR-074 | US3 | US3-1 a US3-7 |
| FR-029 a FR-039, FR-053 | US4 | US4-1 a US4-6 |
| FR-042 a FR-045 | US5 | US5-1 a US5-6 |
| FR-046 a FR-048 | US6 | US6-1 a US6-3 |
| FR-049 a FR-052, FR-070, FR-071 | US7 | US7-1 a US7-5 |
| FR-063 a FR-067 | Transversal (base de NFR-015 a NFR-017) | SC-002 a SC-004 |

Los NFR trazan a restricciones declaradas: NFR-001 a NFR-005 y NFR-021 al hardware de *Assumptions*
y al Principio VI; NFR-006 a NFR-009, NFR-024, NFR-022 y NFR-023 al Principio VII; NFR-010 y NFR-011
al Principio XI; NFR-012 y NFR-013 al Principio V; NFR-014 a NFR-016 al Principio XVI; NFR-017 a
NFR-020 a la usabilidad declarada en el objetivo.

---

## Ambigüedades identificadas

Se identificaron 7 ambigüedades durante `/speckit-specify`. Las 3 bloqueantes (AMB-01, AMB-02,
AMB-03) quedaron resueltas por decisión del usuario en la primera sesión de `/speckit-clarify`, y
5 decisiones más se tomaron en la segunda sesión tras la auditoría. Todas están en *Clarifications*
e incorporadas a los requisitos. **No quedan ambigüedades bloqueantes.**

Las 4 restantes siguen apoyadas en un supuesto documentado, no en una decisión firme. Antes de
apoyarse en cualquiera, leer su fundamento.

### Resueltas con supuesto documentado — no confirmadas por el usuario

- **AMB-04** — **"Pestaña" en el habla del usuario.** Supuesto: significa siempre *espacio de
  trabajo*. Fundamento: el control de pestañas del navegador exige operar la aplicación por dentro,
  y "control de aplicaciones específicas más allá de abrirlas" está fuera de alcance, así que la
  otra lectura no tiene acción que la respalde.
- **AMB-05** — **Espacio de trabajo no indicado.** Supuesto: se usa el espacio de trabajo actual,
  salvo que la aplicación declare uno propio en configuración, en cuyo caso gana el declarado.
  Fundamento: es el comportamiento menos sorprendente y no requiere buscar un espacio "libre",
  noción que además no está definida. Nota: si la aplicación ya está abierta, FR-014 manda y se
  enfoca la instancia existente donde esté; el espacio declarado solo aplica a la primera apertura.
- **AMB-06** — **Superficie de estado por pedido explícito.** Supuesto: aparece únicamente por el
  ciclo automático del turno; no hay una acción para invocarla por separado. Fundamento: FR-037
  exige que sea visible solo cuando hay algo que comunicar, y una invocación manual sin turno en
  curso no tendría contenido que mostrar.
- **AMB-07** — **Configuración con errores al arrancar.** Resuelto en FR-045 y EC-21 con el
  comportamiento que este supuesto proponía: arrancar dejando fuera las entradas inválidas y
  reportarlo. Se conserva la traza del origen; el requisito ya no depende del supuesto.

---

## Fuera de alcance

Queda explícitamente fuera de esta feature:

- Palabra de activación por voz (wake word).
- Modelo de lenguaje generativo para interpretar frases no reconocidas. La resolución de intención
  de esta fase es enteramente determinística.
- Síntesis de voz para respuestas habladas.
- Conversación de múltiples turnos con memoria de contexto. Cada turno es independiente; la única
  excepción es el par acción-confirmación de US3, que no constituye memoria conversacional.
- Consultas **por voz** que devuelven información en lugar de ejecutar acciones. La consulta de la
  bitácora por canales no vocales (FR-047) sí está incluida.
- Control de aplicaciones específicas más allá de abrirlas.
- Múltiples usuarios o perfiles.
- Instalación empaquetada o distribución a terceros.
- Toda superficie gráfica que no sea la del Principio X (FR-053): barra de estado permanente,
  widgets, panel de configuración, historial navegable, dashboards y cualquier ventana enfocable.

La arquitectura DEBE dejar declarados los puntos de extensión para el fallback por modelo de
lenguaje (Principio IV), para la síntesis de voz (Principio VIII) y para las superficies gráficas
excluidas (Principio X), sin implementar ninguno.

---

## Assumptions

- **Entorno**: el usuario corre Omarchy (Arch + Hyprland) en una laptop AMD Ryzen 5 3450U de
  4 núcleos y 8 hilos, con gráficos integrados Radeon Vega 8 que comparten la RAM del sistema,
  8 GB de RAM y SSD de 256 GB, usada simultáneamente para desarrollo. El asistente es siempre un
  proceso accesorio, nunca el principal.
- **Actor único**: hay un solo usuario humano, dueño de la máquina, sin necesidad de autenticación,
  autorización ni separación de permisos.
- **Sesión gráfica**: el asistente corre dentro de una sesión gráfica del usuario ya iniciada, con
  el gestor de ventanas disponible. No cubre el arranque del sistema ni la pantalla de login.
- **Atajo de teclado**: el atajo global se registra a través del entorno de escritorio y se declara
  en configuración; el sistema no lo hardcodea (Principio XIII).
- **Conjunto de frases de referencia**: lo provee el usuario y es un entregable de la feature, no
  una precondición externa. Su composición está especificada en FR-063 a FR-067. Es la base de
  NFR-015 a NFR-017 y de SC-002 a SC-004; con menos de 100 entradas el umbral de NFR-016 (<1%) no
  se distinguiría de cero.
- **Idioma**: la entrada es español rioplatense, admitiendo nombres propios de aplicaciones y
  sitios en inglés. Ningún otro idioma está soportado.
- **Espacios de trabajo**: el rango válido lo determina el entorno de escritorio del usuario y se
  lee del sistema o de la configuración, no se fija en el código.
- **Persistencia de la bitácora**: se conserva localmente. La política de retención y rotación no
  está especificada y se define en la fase de planificación.
- **Turnos concurrentes**: un único turno activo a la vez, garantizado por FR-052 y por la tabla de
  *Máquina de estados*.
