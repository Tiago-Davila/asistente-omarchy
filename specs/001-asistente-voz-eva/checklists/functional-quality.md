# Auditoría de Calidad de la Especificación: Eva

**Objeto auditado**: [spec.md](../spec.md) — 55 FR, 19 NFR, 13 SC, 14 casos borde, 6 historias
**Fecha**: 2026-08-09
**Alcance**: la especificación como documento. No audita código ni plan técnico.
**Sesgo de riesgo**: (a) ejecutar una acción distinta de la pedida · (b) degradar el rendimiento de
la máquina · (c) requisitos no decidibles objetivamente.

**Resultado de la auditoría original**: 27 OK · 26 FALLA · 2 NO APLICA

> **Actualización 2026-08-09 — segunda sesión de `/speckit-clarify`.** Cinco decisiones del usuario
> cerraron seis ítems: **CHK014** (FR-056, EC-15, US1-7), **CHK018** (FR-057 a FR-060, US3-2/3/6),
> **CHK024** y **CHK025** (FR-061, FR-062, NFR-020, EC-16, US2-2/6), **CHK039** (FR-003, EC-17) y
> **CHK045** (FR-063 a FR-067). Estado actual: **33 OK · 20 FALLA · 2 NO APLICA**. Las 20 fallas
> restantes son todas de redacción y se corrigen editando la spec; ninguna requiere una decisión de
> producto. El veredicto del cierre queda vigente para esos 20 ítems.

---

## 1. Testeabilidad de requisitos funcionales

- [ ] **CHK001** ¿Cada FR tiene al menos un criterio de aceptación con entrada y salida concretas? — **FALLA**
  - **Afectado**: FR-004, FR-012, FR-017, FR-020, FR-021, FR-022, FR-025, FR-044, FR-048, FR-049, FR-050, FR-051, FR-052, FR-053.
  - **Por qué**: hay 55 FR y 29 escenarios de aceptación repartidos en 6 historias. Al menos 14 FR no
    tienen ningún escenario que los ejercite. `FR-021` ("cerrarla, ponerla en pantalla completa y
    alternar su estado flotante") es el caso más grave: es la única acción destructiva del catálogo
    base y no tiene un solo escenario.
  - **Redacción alternativa**: agregar a US1 escenarios por cada acción del catálogo sin cobertura,
    y una historia nueva "US7 — Robustez del daemon" que dé escenarios a FR-049 a FR-052. Como
    mínimo: *"**Given** una ventana activa marcada como destructiva, **When** el usuario envía
    'cerrá la ventana', **Then** el asistente muestra la acción y espera confirmación sin cerrarla."*

- [ ] **CHK002** ¿Cada criterio de aceptación se evalúa verdadero/falso sin juicio humano? — **FALLA**
  - **Afectado**: US4 escenario 3 y US4 escenario 5.
  - **Por qué**: US4-3 dice *"**When** pasa el tiempo de permanencia definido"* — ese tiempo **no
    está definido en ninguna parte de la spec**. Un criterio que referencia un valor inexistente no
    es evaluable. US4-5 dice *"**When** se observa la pantalla, **Then** no hay ningún elemento de
    interfaz visible"*: exige que una persona mire.
  - **Redacción alternativa**: definir el valor y usar el estado del sistema como proxy. *"US4-3:
    **When** pasan 3 segundos desde que el turno terminó, **Then** la superficie deja de estar
    presente en el árbol de ventanas del gestor."* / *"US4-5: **Given** el asistente en reposo,
    **When** se consulta el árbol de ventanas del gestor, **Then** no existe ninguna ventana
    perteneciente al asistente."* Agregar el valor como `NFR-020: la superficie MUST desaparecer a
    los 3 s de terminado el turno`.

- [ ] **CHK003** ¿Hay FR con verbos no observables sin definir cómo se comprueban? — **FALLA**
  - **Afectado**: FR-012 ("tolerar"), FR-050 ("estado inconsistente"), FR-047 ("entender"), FR-049
    ("componente").
  - **Por qué**: `FR-012` — *"MUST **tolerar** nombres de aplicaciones y sitios en inglés"* — usa
    exactamente uno de los verbos no observables, sin criterio ni escenario. `FR-050` — *"MUST NOT
    dejar el escritorio en **estado inconsistente**"* — no define qué es un estado inconsistente, así
    que nadie puede decidir si se cumple. `FR-049` habla de "un componente" sin que la spec enumere
    los componentes.
  - **Redacción alternativa**: *"FR-012: para cada aplicación y sitio declarados con nombre en
    inglés, el sistema MUST resolver la intención correcta cuando el nombre aparece dentro de una
    frase en español, verificado sobre las entradas del conjunto de referencia marcadas
    `idioma_mixto`."* / *"FR-050: si un turno falla después de haber iniciado la ejecución, el
    sistema MUST dejar el escritorio en uno de dos estados observables: la acción aplicada por
    completo, o ninguna parte de la acción aplicada. MUST NOT dejar estados intermedios (por
    ejemplo, ventana movida pero no enfocada)."*

- [ ] **CHK004** ¿Cada FR indica camino feliz **y** camino de fallo? — **FALLA**
  - **Afectado**: FR-019, FR-020, FR-021, FR-022, FR-023.
  - **Por qué**: las cinco acciones declaran solo el camino feliz. No está escrito qué pasa si
    `FR-020`/`FR-021` se piden **sin ventana activa**, ni si `FR-022` se pide sin dispositivo de
    salida de audio, ni si una acción propia de `FR-023` falla al ejecutarse.
  - **Redacción alternativa**: añadir a cada uno la cláusula de fallo. *"FR-021: … Si no hay ventana
    activa, el sistema MUST rechazar con motivo 'no hay ventana activa' y MUST NOT operar sobre otra
    ventana."*

- [x] **CHK005** ¿Los FR usan forma normativa consistente (MUST / MUST NOT)? — **OK**
  - Los 55 FR usan MUST/MUST NOT de manera uniforme. No aparecen "should" ni "podría".

- [ ] **CHK006** ¿Los FR agrupan una sola capacidad verificable por identificador? — **FALLA**
  - **Afectado**: FR-021.
  - **Por qué**: `FR-021` agrupa tres acciones distintas (cerrar, pantalla completa, flotante) bajo
    un identificador, y una sola de las tres es destructiva. `FR-028` tiene que desambiguar por
    fuera diciendo "cerrar la ventana activa (FR-021)". Un identificador cuya clasificación de
    seguridad aplica solo a un tercio de su contenido es una trampa para quien implemente.
  - **Redacción alternativa**: separar en `FR-021a` cerrar ventana activa (destructiva), `FR-021b`
    alternar pantalla completa, `FR-021c` alternar flotante, y ajustar la referencia de FR-028.

---

## 2. Testeabilidad de requisitos no funcionales

- [ ] **CHK007** ¿Cada NFR declara métrica, unidad, umbral **y** condición de medición? — **FALLA**
  - **Afectado**: NFR-002, NFR-004, NFR-006, NFR-007, NFR-008.
  - **Por qué**: `NFR-001` es el único completo (métrica, unidad, umbral, ventana de 10 min, estado
    "sin activaciones"). `NFR-002` da umbral pero no ventana de medición. `NFR-006`/`NFR-007` dan
    percentil y umbral pero **no dicen bajo qué carga de la máquina se miden** (ver CHK008).
    `NFR-008` da 100 ms sin condición alguna.
  - **Redacción alternativa**: *"NFR-002: en reposo, el conjunto de procesos del asistente MUST
    mantener un RSS agregado menor a 150 MB, medido como máximo observado en una ventana de 10
    minutos sin activaciones, excluyendo páginas compartidas contabilizadas más de una vez."*

- [ ] **CHK008** ¿Se declara bajo qué carga del sistema se mide cada umbral? — **FALLA**
  - **Afectado**: NFR-001 a NFR-008 (todos los presupuestos de recursos y latencia).
  - **Por qué**: la sección *Assumptions* dice que la máquina se usa "simultáneamente para
    desarrollo, con navegador y editor abiertos", pero **ningún NFR referencia esa condición**. Un
    p95 de 3,5 s medido en una máquina ociosa y uno medido con el navegador consumiendo RAM son
    números distintos, y la spec no dice cuál cuenta. Es el hueco central del riesgo (b): el
    presupuesto que protege al usuario se mide justo en el escenario donde el usuario no está.
  - **Redacción alternativa**: agregar una definición única y referenciarla. *"**Carga de
    referencia**: navegador con 10 pestañas abiertas y un editor de código con un proyecto cargado,
    sin compilación en curso. Todos los NFR-001 a NFR-008 se miden bajo la carga de referencia,
    salvo indicación explícita en contrario."*

- [ ] **CHK009** ¿Se declara con qué instrumento o procedimiento se mide cada NFR? — **FALLA**
  - **Afectado**: NFR-002, NFR-003, NFR-004, NFR-006, NFR-007, NFR-008.
  - **Por qué**: ninguno indica el procedimiento. Para memoria importa especialmente: "memoria
    residente sumando todos sus procesos" (`NFR-002`) da resultados distintos según se cuenten o no
    páginas compartidas entre procesos. Sin fijar el método, dos mediciones honestas pueden diferir
    en decenas de MB y ambas "cumplir".
  - **Redacción alternativa**: agregar una subsección *Procedimiento de medición* que fije, por
    dimensión, la fuente del dato, la frecuencia de muestreo y la ventana de agregación.

- [ ] **CHK010** ¿Hay NFR que dependan de percepción subjetiva sin proxy objetivo? — **FALLA**
  - **Afectado**: NFR-005, NFR-018, SC-006, SC-010; también FR-031 ("comprensible") y FR-032
    ("reconocible sin mirar la pantalla").
  - **Por qué**: `NFR-005` — *"la ventana en la que el usuario trabaja MUST NOT perder fluidez de
    forma **perceptible**"* — no es decidible: dos personas pueden discrepar y ambas tener razón.
    `SC-006` repite el mismo problema como criterio de éxito. Esto es el riesgo (b) escrito de forma
    que nadie puede reprobarlo.
  - **Redacción alternativa**: sustituir percepción por un proxy medible. *"NFR-005: mientras el
    asistente procesa un turno bajo la carga de referencia, el tiempo entre cuadros de la ventana
    enfocada MUST mantenerse por debajo de 33 ms en el percentil 99, y el asistente MUST NOT
    provocar más de 1 cuadro descartado por turno."* Para FR-032: *"la señal de éxito y la de error
    MUST diferir en frecuencia fundamental en al menos una tercera mayor, y ambas MUST durar menos
    de 300 ms."*

- [x] **CHK011** ¿Los presupuestos de recursos distinguen reposo, pico y sostenido? — **OK**
  - `NFR-001`/`NFR-002` cubren reposo sostenido; `NFR-003` cubre pico durante turno. La distinción
    existe y está bien marcada. (Ver CHK012 para el hueco de CPU sostenido bajo turno.)

- [ ] **CHK012** ¿Existe un techo de CPU durante el turno, no solo en reposo? — **FALLA**
  - **Afectado**: [Gap] entre NFR-001 y NFR-004.
  - **Por qué**: `NFR-001` acota la CPU en reposo y `NFR-004` exige "dejar al menos un núcleo libre",
    pero **no hay techo de CPU sostenido durante el procesamiento**. Además "núcleo libre" no está
    definido: ¿0% de uso, o por debajo de algún umbral? Con 4 núcleos físicos y gráficos integrados
    compartiendo RAM, esta es la restricción que más directamente protege al usuario, y es la peor
    especificada de las cinco.
  - **Redacción alternativa**: *"NFR-004: durante el procesamiento de un turno, el conjunto de
    procesos del asistente MUST NOT superar el 75% de la capacidad total de CPU (equivalente a 3 de
    4 núcleos), medido como promedio en ventanas de 200 ms. Ningún núcleo MUST quedar por encima del
    10% de utilización atribuible al asistente durante más de 2 ventanas consecutivas."*

- [ ] **CHK013** ¿Los presupuestos de latencia por etapa suman como máximo el total declarado? — **FALLA**
  - **Afectado**: NFR-009.
  - **Por qué**: **no se puede evaluar**, porque los presupuestos por etapa no existen en el
    documento. `NFR-009` es un meta-requisito: *"Cada etapa del procesamiento MUST tener un
    presupuesto de tiempo declarado"* — exige que exista una declaración, sin declararla y sin
    exigir que la suma respete `NFR-006`/`NFR-007`. Tal como está, un plan podría declarar etapas
    que sumen 6 s y cumplir NFR-009 al pie de la letra mientras viola NFR-007.
  - **Redacción alternativa**: *"NFR-009: la spec de plan MUST declarar un presupuesto de tiempo por
    cada etapa del pipeline. La suma de los presupuestos por etapa MUST ser menor o igual a 2 s para
    el percentil 50 y a 3,5 s para el percentil 95, de modo que NFR-006 y NFR-007 se satisfagan por
    construcción. El tiempo real por etapa MUST quedar registrado por turno."*

---

## 3. Seguridad de ejecución

- [ ] **CHK014** ¿Está especificado el comportamiento ante **acción ambigua entre dos candidatas**? — **FALLA** ⛔
  - **Afectado**: [Gap]. `FR-009` cubre "no corresponde a ninguna acción"; nada cubre "corresponde a
    dos".
  - **Por qué**: es el hueco más caro del documento y ataca de lleno el riesgo (a). La spec
    especifica las cuatro clases de fallo que declara `FR-031` (no se entendió, acción desconocida,
    parámetro inválido, fallo al ejecutar) pero **la ambigüedad entre candidatas no es ninguna de
    las cuatro**. El Principio XVI de la constitución dice "Ante ambigüedad: pregunta o rechaza" —
    ofrece dos salidas y la spec no elige ninguna. Sin una regla escrita, quien implemente va a
    desempatar por puntaje, que es exactamente ejecutar una acción distinta de la pedida.
  - **Redacción alternativa**: *"FR-056: si el texto resuelve a más de una acción del registro con
    parámetros válidos, el sistema MUST rechazar con motivo 'intención ambigua', MUST nombrar las
    acciones candidatas en el feedback, y MUST NOT desempatar por puntaje, orden de declaración,
    frecuencia de uso ni ningún otro criterio implícito."* Agregar `EC-15` y un escenario en US1,
    más una clase de error en `FR-031` (pasa de cuatro a cinco).

- [ ] **CHK015** ¿Está definida la frontera entre "no se entendió" y "acción desconocida"? — **FALLA**
  - **Afectado**: FR-031, FR-009, EC-03.
  - **Por qué**: `FR-031` obliga a distinguir las dos clases, pero la spec nunca define qué separa
    una de otra. `FR-009` y `EC-03` describen ambas como "el texto no corresponde a ninguna acción".
    Un requisito que exige distinguir dos categorías sin definirlas no es verificable.
  - **Redacción alternativa**: *"'No se entendió' MUST aplicar cuando la transcripción está vacía o
    no contiene ningún token reconocible del vocabulario declarado. 'Acción desconocida' MUST
    aplicar cuando la transcripción es legible pero no coincide con ninguna acción del registro."*

- [x] **CHK016** ¿Está declarado explícitamente que ante duda el sistema rechaza y no aproxima? — **OK**
  - `FR-009` ("MUST NOT ejecutar una acción aproximada"), `EC-03` ("Nunca se ejecuta la acción más
    parecida"), `EC-10` ("No se completa ni se adivina el texto faltante") y `EC-05` ("No se recorta
    ni se aproxima al valor más cercano") lo declaran con criterio asociado. Bien cubierto.

- [x] **CHK017** ¿Está definido qué hace destructiva a una acción, con lista o regla? — **OK**
  - `FR-028` da la lista exacta y la regla; `FR-054` fija el default ante ausencia de declaración.
    No queda librado al criterio de quien implemente.

- [ ] **CHK018** ¿Está especificado **qué cuenta como aceptar y qué cuenta como cancelar** una confirmación? — **FALLA** ⛔
  - **Afectado**: FR-026, FR-027, FR-038, US3 escenarios 2 y 3.
  - **Por qué**: la spec define qué se muestra (`FR-026`) y qué pasa si nadie responde (`FR-055`,
    10 s), pero **nunca define el mecanismo de confirmación**. US3-2 dice *"el usuario confirma
    explícitamente"* sin decir cómo: ¿por voz? ¿por tecla? ¿qué palabras? Es un hueco de seguridad,
    no solo de redacción: si "sí" por voz cuenta, una conversación de fondo puede confirmar el
    cierre de una ventana con trabajo sin guardar. `FR-027` prohíbe inferir la confirmación del
    contexto, pero sin canal declarado esa prohibición no se puede comprobar.
  - **Redacción alternativa**: *"FR-057: la confirmación MUST realizarse por un canal explícito y
    declarado en configuración, distinto del canal de voz del turno. Aceptar MUST requerir una
    pulsación de tecla dedicada; cancelar MUST requerir otra. Ninguna entrada de voz MUST poder
    confirmar una acción destructiva. Cualquier entrada distinta de las dos teclas declaradas MUST
    tratarse como no-respuesta y MUST NOT ejecutar la acción."*

- [ ] **CHK019** ¿Hay requisitos que permitan ejecutar algo derivado de la transcripción sin validar? — **FALLA**
  - **Afectado**: FR-018.
  - **Por qué**: `FR-025` prohíbe comandos arbitrarios y `FR-024` exige validar parámetros, pero
    `FR-018` toma **texto libre dictado** y lo pasa como término de búsqueda. La spec no declara
    ninguna restricción sobre ese texto (longitud máxima, caracteres admitidos, escapado antes de
    componer la URL). Es el único punto del sistema donde entrada no acotada llega a un ejecutor.
  - **Redacción alternativa**: *"FR-018: … El término de búsqueda MUST validarse como texto plano de
    entre 1 y 200 caracteres, MUST escaparse antes de componer la URL de búsqueda, y MUST NOT
    interpretarse como parte de la estructura de la URL ni como argumento de línea de comandos."*

- [x] **CHK020** ¿Está prohibida la ejecución de comandos arbitrarios con requisito explícito? — **OK**
  - `FR-025` es explícito y `FR-023` acota las acciones propias a "una lista declarada en
    configuración". El registro cerrado del Principio III está bien reflejado.

---

## 4. Completitud de estados y transiciones

- [x] **CHK021** ¿Están enumerados todos los estados del asistente? — **OK**
  - `FR-029` y la entidad *Estado del asistente* declaran el mismo conjunto cerrado de seis: reposo,
    escuchando, procesando, ejecutando, esperando confirmación, error. Coinciden entre sí.

- [ ] **CHK022** ¿Para cada estado está definido qué entradas acepta y cuáles ignora? — **FALLA** ⛔
  - **Afectado**: [Gap]. No existe tabla de transiciones en la spec.
  - **Por qué**: hay seis estados declarados y ninguna matriz que diga, para cada uno, qué entradas
    procesa y qué entradas descarta. Lo único que existe es `EC-07` (activación durante turno) y
    `FR-052`. Con seis estados y al menos cuatro clases de entrada (activación, fin de habla,
    confirmación, cancelación) hay 24 combinaciones, de las cuales la spec define 3.
  - **Redacción alternativa**: agregar una sección *Máquina de estados* con una matriz
    estado × entrada, donde cada celda diga "procesa y transiciona a X", "ignora" o "rechaza con
    motivo Y". Es el artefacto que hace testeable a US2 y US3 en conjunto.

- [ ] **CHK023** ¿Está definido qué pasa si llega una activación **en cada** estado? — **FALLA**
  - **Afectado**: EC-07, FR-052, US3 escenario 3.
  - **Por qué**: `EC-07` da una regla única ("ignorar la nueva activación mientras haya una en
    curso") que, aplicada a "esperando confirmación", produce una contradicción con US3-3: el
    usuario debe poder cancelar, pero si toda activación se ignora mientras hay un turno en curso,
    no hay canal para cancelar. La spec no dice si cancelar usa el atajo de activación u otro.
  - **Redacción alternativa**: distinguir por estado. *"EC-07: una activación recibida en estado
    escuchando, procesando o ejecutando MUST ignorarse e informarse. Una activación recibida en
    estado esperando confirmación MUST interpretarse como cancelación de la acción pendiente."*

- [ ] **CHK024** ¿Está definido cómo se sale de cada estado, incluyendo por error y por tiempo agotado? — **FALLA** ⛔
  - **Afectado**: estados `escuchando`, `procesando` y `error`.
  - **Por qué**: tres de los seis estados no tienen salida acotada.
    · **escuchando**: `FR-006` corta por detección de fin de habla, pero **no hay duración máxima de
    captura**. Si la detección falla, el estado no termina nunca — y además la captura sin techo
    rompe el presupuesto de memoria (riesgo b).
    · **procesando**: no hay timeout. `NFR-006`/`NFR-007` son objetivos de percentil, no cortes.
    · **error**: la spec nunca dice cuánto dura ni cómo se sale. `NFR-013` dice que tras un fallo se
    vuelve a reposo, pero no si "error" es un estado con permanencia o una transición instantánea.
  - **Redacción alternativa**: *"FR-058: la captura de audio MUST terminar como máximo a los 15
    segundos desde la activación, aunque no se haya detectado fin de habla; en ese caso el turno
    MUST tratarse como transcripción parcial (EC-10). FR-059: el procesamiento MUST abortar a los 10
    segundos, terminando el turno con motivo 'tiempo agotado' sin ejecutar la acción. FR-060: el
    estado error MUST durar como máximo 3 segundos, tras los cuales el sistema MUST volver a
    reposo."*

- [ ] **CHK025** ¿Hay algún estado del que no esté declarada la salida? — **FALLA**
  - Consecuencia directa de CHK024: `escuchando`, `procesando` y `error` carecen de salida
    garantizada. Con la corrección de CHK024 este ítem se cierra.

---

## 5. Consistencia interna

- [ ] **CHK026** ¿Hay FR que se contradigan entre sí? — **FALLA**
  - **Afectado**: FR-034 vs FR-039.
  - **Por qué**: `FR-034` dice *"El sistema **MUST mostrar** una única superficie gráfica mínima"*
    —obligación incondicional— mientras `FR-039` dice *"El sistema MUST seguir siendo completamente
    funcional **con la superficie deshabilitada**"*. Si mostrarla es obligatorio, deshabilitarla es
    un incumplimiento. La condición 5 del Principio X depende de que se pueda apagar.
  - **Redacción alternativa**: *"FR-034: cuando la superficie gráfica está habilitada en
    configuración, el sistema MUST mostrar una única superficie mínima, y esta MUST mostrar
    únicamente el estado del turno, el texto entendido y la acción resuelta. La superficie MUST
    poder deshabilitarse desde configuración."*

- [ ] **CHK027** ¿Hay NFR cuyos umbrales sean incompatibles con algún FR? — **FALLA** ⛔
  - **Afectado**: NFR-006 y NFR-007 vs FR-026 y FR-055.
  - **Por qué**: `NFR-006`/`NFR-007` miden *"desde el fin del habla hasta **la ejecución de la
    acción**"* con techos de 2 s y 3,5 s. Pero `FR-026` obliga a esperar confirmación humana en las
    acciones destructivas, y `FR-055` admite hasta 10 s de espera. Un turno destructivo puede tardar
    11 s legítimamente y violar NFR-007 sin que nada esté mal. Los dos requisitos no pueden ser
    ciertos a la vez sobre el mismo conjunto de turnos.
  - **Redacción alternativa**: *"NFR-006 / NFR-007: … medido sobre los turnos que no requieren
    confirmación. En turnos con confirmación, el presupuesto MUST aplicarse en dos tramos: fin del
    habla → presentación de la confirmación (mismos techos de 2 s y 3,5 s), y confirmación del
    usuario → ejecución (techo de 500 ms). El tiempo de respuesta humana MUST quedar excluido."*

- [ ] **CHK028** ¿Los presupuestos de memoria son compatibles entre sí y con los de latencia? — **FALLA** ⛔
  - **Afectado**: NFR-002 vs NFR-003 vs NFR-006.
  - **Por qué**: `NFR-002` fija 150 MB en reposo y `NFR-003` fija 1,5 GB de pico por turno. La
    diferencia de un orden de magnitud implica que el modelo de transcripción **no puede estar
    residente**: tiene que cargarse por turno. Pero `NFR-006` exige p50 menor a 2 s desde el fin del
    habla, y ese presupuesto tendría que absorber la carga del modelo desde un SSD además de la
    inferencia. La spec **no declara si el modelo es residente o bajo demanda**, que es justo la
    clasificación que el Principio VI exige ("los componentes pesados se cargan bajo demanda, o se
    declaran explícitamente como residentes con su costo justificado"). Tal como está, los tres
    números pueden ser individualmente razonables y conjuntamente imposibles.
  - **Redacción alternativa**: agregar el requisito que fuerza la decisión al plan de forma
    verificable. *"NFR-021: la spec de plan MUST declarar, por cada componente pesado, si es
    residente o de carga bajo demanda, con su costo de memoria y su tiempo de carga medidos. La suma
    de los residentes MUST caber en NFR-002, y el tiempo de carga de los de demanda MUST estar
    incluido dentro del presupuesto de NFR-006."*

- [ ] **CHK029** ¿Los casos borde contradicen alguna regla declarada en los FR? — **FALLA**
  - **Afectado**: EC-02 vs FR-032.
  - **Por qué**: `EC-02` dice que una activación sin habla *"se descarta **en silencio o** con señal
    mínima"* — una disyunción sin criterio, que además choca con `FR-032`, que obliga a emitir señal
    sonora al terminar **cada** turno distinguiendo éxito de error. Un turno descartado no es ni una
    cosa ni la otra, y la spec ofrece dos comportamientos incompatibles sin decir cuál rige.
  - **Redacción alternativa**: *"EC-02: se descarta la activación emitiendo la señal sonora de
    turno vacío, distinta de las de éxito y error (FR-032). No se transcribe, no se ejecuta nada, y
    se vuelve a reposo."* Y extender FR-032 a tres señales.

- [ ] **CHK030** ¿Se usa la misma terminología para el mismo concepto en toda la spec? — **FALLA** ⛔
  - **Afectado**: la palabra "registro", usada para dos conceptos distintos.
  - **Por qué**: `FR-008` habla del *"**registro** cerrado y declarado de acciones"* (el catálogo de
    capacidades) y la entidad se llama *"Acción del **registro**"*. Pero `FR-046` a `FR-048` y toda
    US6 usan *"**registro** de actividad"* para la bitácora de turnos. Es la misma palabra para el
    catálogo y para el log. Un lector que encuentre "el registro conserva el intento" (`EC-08`) o
    "consultar el registro" (`FR-047`) tiene que inferir cuál de los dos es. Esta clase de colisión
    produce implementaciones equivocadas.
  - **Redacción alternativa**: fijar dos términos disjuntos y usarlos en todo el documento:
    **catálogo de acciones** para lo de FR-008 y la entidad, y **bitácora** para lo de FR-046 a
    FR-048, EC-08, EC-11 y US6. Agregar ambos al glosario.

- [ ] **CHK031** ¿Se usa un solo término para el elemento gráfico? — **FALLA**
  - **Afectado**: "interfaz" vs "superficie".
  - **Por qué**: tras la alineación con la constitución, `FR-034` a `FR-039` dicen "superficie", pero
    el encabezado del grupo dice "**Interfaz** gráfica", US4 dice "la **interfaz** aparece" en cuatro
    escenarios, y `AMB-06` se titula "**Interfaz** gráfica por pedido explícito". Dos nombres para
    una sola cosa.
  - **Redacción alternativa**: normalizar a **superficie de estado** en todo el documento, con una
    única mención "(anteriormente denominada interfaz gráfica)" en el glosario.

- [ ] **CHK032** ¿Hay un glosario de términos canónicos? — **FALLA**
  - **Afectado**: [Gap]. No existe sección de glosario.
  - **Por qué**: "turno", "espacio de trabajo", "superficie de estado", "catálogo de acciones",
    "bitácora", "carga de referencia" y "conjunto de frases de referencia" son términos con
    significado técnico preciso en esta spec y ninguno está definido en un lugar único. Los
    problemas de CHK030 y CHK031 son síntomas de esta ausencia.
  - **Redacción alternativa**: agregar una sección *Glosario* antes de *Requirements* con una
    entrada por término, y marcar los sinónimos prohibidos ("escritorio" por espacio de trabajo,
    "interfaz" por superficie, "registro" a secas).

---

## 6. Cierre de ambigüedades

- [ ] **CHK033** ¿Cada ambigüedad resuelta quedó como FR numerado **con criterio de aceptación**? — **FALLA** ⛔
  - **Afectado**: las decisiones de AMB-02, AMB-03 y la del espacio de trabajo (sesión 2026-08-09).
  - **Por qué**: las cinco decisiones de `/speckit-clarify` se convirtieron en FR y casos borde, pero
    **tres de las cinco no tienen ningún escenario de aceptación**: AMB-02 → `FR-014`+`EC-06` sin
    escenario; AMB-03 → `FR-016`/`FR-018`+`EC-13` sin escenario; espacio de trabajo ocupado →
    `FR-015`/`FR-017`+`EC-14` sin escenario. Solo la de los 10 segundos llegó a un escenario
    (US3-4). Es exactamente el patrón "decisión escrita sin comportamiento observable asociado".
  - **Redacción alternativa**: agregar tres escenarios a US1. *"**Given** la terminal ya abierta en
    el espacio 1, **When** el usuario envía 'abrí la terminal en el escritorio 3', **Then** la
    ventana existente queda en el espacio 3 y enfocada, y no existe una segunda ventana de
    terminal."* / *"**Given** un sitio no declarado en configuración, **When** el usuario envía
    'abrí Reddit', **Then** el asistente rechaza con motivo 'sitio no declarado' y no ejecuta una
    búsqueda web."*

- [ ] **CHK034** ¿Quedó alguna ambigüedad resuelta a medias? — **FALLA**
  - **Afectado**: FR-015 y FR-017.
  - **Por qué**: `FR-014` resuelve el caso de varias instancias diciendo "la más reciente si hay
    varias". `FR-015` hereda la regla de mover al espacio pedido pero **no repite el criterio de
    desempate**: si la aplicación está abierta en tres espacios y el usuario nombra un cuarto, la
    spec no dice cuál ventana se mueve.
  - **Redacción alternativa**: *"FR-015: … Si hay varias ventanas de la aplicación, MUST moverse la
    más reciente, con el mismo criterio de FR-014."*

- [ ] **CHK035** ¿Aparecieron ambigüedades nuevas al resolver las anteriores? — **FALLA**
  - **Afectado**: FR-017 y FR-028.
  - **Por qué**: dos casos nuevos, ambos creados por las resoluciones.
    · `FR-017` extiende a URLs la regla de mover la ventana: pedir una URL "en el escritorio 3"
    **mueve toda la ventana del navegador**, incluidas las demás pestañas que el usuario tenía en su
    espacio original. La spec adopta ese comportamiento sin declararlo como consecuencia aceptada.
    · `FR-028` fija que las acciones propias sin declaración son destructivas, y `FR-054` obliga a
    permitir la declaración. No está dicho qué pasa si la configuración declara `destructiva: true`
    para una acción del catálogo base (por ejemplo, marcar "cambiar de espacio" como destructiva):
    ¿gana la configuración del usuario o la lista cerrada de FR-028?
  - **Redacción alternativa**: para el primero, declararlo explícitamente en FR-017 y agregar un caso
    borde. Para el segundo: *"FR-028: … La configuración del usuario MUST poder elevar a destructiva
    cualquier acción del catálogo base, y MUST NOT poder rebajar a no destructiva ninguna acción
    listada como destructiva en este requisito."*

---

## 7. Cobertura de casos borde

- [x] **CHK036** ¿Cada caso borde declarado tiene comportamiento esperado explícito? — **OK**
  - Los 14 casos usan formato de tabla con columna de comportamiento esperado; ninguno se queda en
    la descripción del caso. (La disyunción de EC-02 se trata en CHK029.)

- [ ] **CHK037** ¿Hay caminos de fallo mencionados en los FR sin caso borde? — **FALLA**
  - **Afectado**: [Gap] para FR-020/FR-021 (sin ventana activa), FR-022 (sin dispositivo de salida),
    FR-044 (recarga de configuración con un turno en curso), FR-023 (acción propia que falla).
  - **Por qué**: cuatro caminos de fallo derivables de los FR no tienen fila en la tabla. El de
    `FR-044` es el más relevante para consistencia: si la configuración se recarga a mitad de un
    turno, la acción resuelta contra el catálogo viejo puede ejecutarse contra el nuevo.
  - **Redacción alternativa**: agregar `EC-16` a `EC-19`. Para el de recarga: *"El turno en curso
    MUST completarse contra el catálogo vigente al momento de resolver la intención. La
    configuración nueva MUST aplicarse recién a partir del turno siguiente."*

- [ ] **CHK038** ¿Están cubiertos los fallos de recursos externos? — **FALLA** (parcial)
  - **Afectado**: configuración inválida.
  - **Por qué**: dispositivo de audio (`EC-01`, `EC-12`), gestor de ventanas (`EC-08`) y sistema
    saturado (`EC-09`) tienen fila propia. **Configuración inválida no la tiene**: su comportamiento
    vive en US5-5 y en `AMB-07`, que es un supuesto no confirmado, no una regla. `NFR-012` la lista
    como fallo que el asistente debe sobrevivir, sin caso borde que lo describa.
  - **Redacción alternativa**: promover AMB-07 a `EC-20` con el comportamiento ya redactado allí
    (arrancar dejando fuera las entradas inválidas, reportar de forma visible, nunca arrancar en
    silencio ni negarse a arrancar).

- [ ] **CHK039** ¿Está cubierta la cancelación **durante** la ejecución? — **FALLA**
  - **Afectado**: FR-003, US2 escenario 3.
  - **Por qué**: `FR-003` acota la cancelación a "**antes de que la acción se ejecute**", y US2-3
    repite el recorte. La spec no dice qué pasa si el usuario cancela con la acción ya en curso: ¿se
    ignora la cancelación? ¿se intenta revertir? Se conecta con `FR-050` (no dejar estado
    inconsistente), que tampoco está definido.
  - **Redacción alternativa**: *"FR-003: … Una cancelación recibida después de iniciada la ejecución
    MUST ignorarse, y el sistema MUST informar que la acción ya estaba en curso. El sistema MUST NOT
    intentar revertir una acción parcialmente aplicada."*

- [x] **CHK040** ¿Están cubiertas las activaciones solapadas? — **OK**
  - `EC-07` y `FR-052` cubren el caso. (La política concreta se discute en CHK041 y CHK023.)

- [ ] **CHK041** ¿Los requisitos declaran la política, o solo exigen que exista una? — **FALLA**
  - **Afectado**: FR-052.
  - **Por qué**: `FR-052` dice *"MUST aplicar una política única y declarada"* sin declararla. La
    política real solo existe en la columna de comportamiento de `EC-07`. Un requisito que exige que
    algo esté declarado, sin declararlo, no es verificable por sí mismo. Mismo defecto de forma que
    `NFR-009` (CHK013).
  - **Redacción alternativa**: *"FR-052: cuando llega una activación con un turno en curso, el
    sistema MUST ignorarla e informar al usuario que hay un turno en proceso, salvo en el estado
    esperando confirmación, donde MUST interpretarse como cancelación (EC-07)."*

---

## 8. Verificabilidad sin dependencias

- [ ] **CHK042** ¿Se puede verificar **toda** la funcionalidad sin micrófono, audio ni intervención manual? — **FALLA**
  - **Afectado**: FR-041, SC-012 vs FR-032, US4 escenarios 1, 3 y 5.
  - **Por qué**: `FR-041` afirma que *"**toda** la funcionalidad MUST ser ejercitable sin micrófono
    ni dispositivos de audio"*, pero `FR-032` exige una señal sonora, que requiere salida de audio, y
    los escenarios de US4 requieren mirar la pantalla y comprobar el foco del teclado. La afirmación
    es más amplia que lo alcanzable, así que como está escrita es falsa.
  - **Redacción alternativa**: acotar el alcance de la afirmación. *"FR-041: la resolución de
    intención, la validación de parámetros, la clasificación de destructividad, el rechazo y la
    ejecución de acciones MUST ser ejercitables sin micrófono ni dispositivos de audio presentes. El
    feedback sonoro (FR-032) y la superficie de estado (FR-034) MUST verificarse por separado
    mediante sus propios canales observables."*

- [ ] **CHK043** ¿Hay requisitos cuya verificación exija que una persona escuche, mire o juzgue? — **FALLA**
  - **Afectado**: FR-031, FR-032, NFR-005, NFR-018, SC-006, SC-010, US4-5.
  - **Por qué**: siete puntos exigen percepción humana sin proxy. `FR-031` pide errores
    "comprensibles"; `SC-010` pide que el usuario "pueda determinar la causa"; `NFR-018` pide que
    "pueda determinar el estado". Ninguno es reprobable objetivamente.
  - **Redacción alternativa**: proxies concretos. *"FR-031: cada error MUST emitirse con un código de
    clase de un conjunto cerrado de cinco valores y un mensaje que nombre el parámetro o la acción
    involucrada."* / *"SC-010: para el 100% de los turnos no exitosos, la entrada de bitácora MUST
    contener el código de clase de error y el identificador de la etapa donde se produjo, sin
    necesidad de reproducir el turno."* / *"NFR-018: el estado actual MUST estar disponible por un
    canal consultable en menos de 100 ms sin cambiar el foco de ventana."*

- [x] **CHK044** ¿Está especificado cómo se verifica la operación sin conexión? — **OK**
  - `SC-007` declara el procedimiento ("verificado con la interfaz de red inhabilitada durante una
    sesión completa de uso") y `NFR-011` fija la regla. Es de los criterios mejor construidos del
    documento. (Ver CHK048 por su choque con FR-018.)

- [ ] **CHK045** ¿Está definido qué constituye el conjunto de frases de referencia y cómo se mide contra él? — **FALLA** ⛔
  - **Afectado**: NFR-014, NFR-015, NFR-016, SC-002, SC-003, SC-004, NFR-006, NFR-007, y la entidad
    *Frase de referencia*.
  - **Por qué**: es el artefacto más cargado de la spec —ocho requisitos dependen de él— y lo único
    que se dice es que "existe y lo provee el usuario" (*Assumptions*). **No hay tamaño mínimo, ni
    composición por acción, ni proporción de frases que deben rechazarse, ni procedimiento de
    medición.** La consecuencia es concreta y grave: `NFR-015` exige una tasa de error menor al 1%,
    pero con un conjunto de menos de 100 frases ese umbral no se puede distinguir de cero. Ocho
    requisitos numéricos apoyados en un conjunto sin cardinalidad definida no son falsables. Es el
    riesgo (c) en su forma más pura.
  - **Redacción alternativa**: *"El conjunto de frases de referencia MUST contener al menos 300
    entradas, con un mínimo de 10 fraseos distintos por cada acción del catálogo, al menos 3 de ellos
    en registro rioplatense con voseo o enclítico. Al menos 60 entradas MUST estar marcadas 'debe
    rechazarse', cubriendo las cinco clases de rechazo. Cada entrada MUST declarar texto, acción
    esperada y parámetros esperados. La precisión MUST medirse ejecutando el conjunto completo por
    la vía de texto plano y comparando acción y parámetros resueltos contra los esperados, entrada
    por entrada."* Además, promover el conjunto de *Assumption* a entregable con requisito propio.

---

## 9. Alcance

- [x] **CHK046** ¿Hay decisiones de tecnología, lenguaje, biblioteca o modelo filtradas en la spec? — **OK**
  - No hay lenguaje, framework, biblioteca ni modelo concreto en el documento. Las menciones a
    Omarchy, Arch y Hyprland están en *Assumptions* como contexto de entorno del usuario, no como
    elección de implementación. La spec respeta el límite con la fase de planificación.

- [ ] **CHK047** ¿Hay elementos fuera de alcance que en realidad hagan falta para un requisito incluido? — **FALLA**
  - **Afectado**: *Fuera de alcance* ("consultas que devuelven información") vs FR-047.
  - **Por qué**: la spec excluye "consultas que devuelven información en lugar de ejecutar acciones",
    pero `FR-047` exige que *"el usuario MUST poder consultar ese registro"*, que es literalmente una
    consulta que devuelve información. La exclusión probablemente apunta a consultas **por voz**,
    pero no lo dice, así que se contradice con un requisito incluido.
  - **Redacción alternativa**: *"Fuera de alcance: consultas **por voz** que devuelven información en
    lugar de ejecutar acciones. La consulta de la bitácora por canales no vocales (FR-047) sí está
    incluida."*

- [ ] **CHK048** ¿Hay requisitos incompatibles con la premisa de operación sin conexión? — **FALLA**
  - **Afectado**: FR-016, FR-018 vs SC-007 y NFR-011.
  - **Por qué**: `SC-007` fija que la verificación de privacidad se hace *"con la interfaz de red
    inhabilitada durante una sesión completa de uso"*, pero `FR-018` (búsqueda web) y `FR-016` (abrir
    una URL) no pueden producir un resultado útil sin red. La spec nunca distingue entre "el
    asistente no usa la red" y "las aplicaciones que el asistente lanza no usan la red" — y esa
    distinción es justamente la que hace que ambos requisitos convivan.
  - **Redacción alternativa**: *"NFR-011: ningún componente del asistente MUST establecer conexiones
    de red. Entregar una URL o un término de búsqueda al navegador declarado NO constituye una
    dependencia de red del asistente; el uso posterior de la red es del navegador."* Y en SC-007:
    *"… verificado inhabilitando la red y comprobando que el asistente completa todas sus etapas
    hasta la entrega al navegador, sin errores propios ni intentos de conexión atribuibles al
    asistente."*

- [x] **CHK049** ¿Los elementos fuera de alcance tienen su punto de extensión declarado? — **OK**
  - La sección cierra exigiendo puntos de extensión declarados y no implementados para el fallback
    por modelo (Principio IV), la síntesis de voz (Principio VIII) y las superficies excluidas
    (Principio X). Consistente con la constitución v2.0.0.

---

## 10. Trazabilidad

- [ ] **CHK050** ¿Cada FR se remonta a una historia de usuario? — **FALLA**
  - **Afectado**: FR-049, FR-050, FR-051, FR-052; parcialmente FR-004 y FR-053.
  - **Por qué**: los cuatro requisitos del grupo *Robustez* no pertenecen a ninguna de las seis
    historias. No hay una historia de usuario sobre supervivencia del daemon, así que son requisitos
    sin narrativa que los justifique ni escenarios que los prueben. El Principio I exige que todo
    artefacto pueda señalar el requisito que lo justifica; acá el eslabón que falta es el de arriba.
  - **Redacción alternativa**: agregar *"US7 — El asistente sigue vivo después de un fallo (P2)"* con
    escenarios para micrófono ocupado, cambio de dispositivo, error del gestor de ventanas y
    activación concurrente, y asociarle FR-049 a FR-052.

- [x] **CHK051** ¿Cada historia de usuario tiene al menos un FR que la implemente? — **OK**
  - Las seis historias tienen FR asociados: US1→FR-008 a FR-024 y FR-040, US2→FR-001 a FR-007,
    US3→FR-026 a FR-028 y FR-054/055, US4→FR-034 a FR-039 y FR-053, US5→FR-042 a FR-045,
    US6→FR-046 a FR-048.

- [x] **CHK052** ¿Cada NFR se remonta a una restricción real declarada del contexto? — **OK**
  - Los presupuestos de recursos trazan al hardware de *Assumptions* y al Principio VI; los de
    privacidad al Principio XI; los de precisión al Principio XVI. La tabla de la sección
    *Alineación* hace explícita la traza de los tres presupuestos más estrictos.

- [ ] **CHK053** ¿Existe una matriz de trazabilidad FR ↔ historia ↔ criterio de aceptación? — **FALLA**
  - **Afectado**: [Gap]. No existe.
  - **Por qué**: con 55 FR, 19 NFR y 29 escenarios, la cobertura solo se puede auditar leyendo el
    documento entero y cruzando a mano — que es como se detectaron CHK001, CHK033 y CHK050. Sin
    matriz, la próxima incorporación de requisitos va a volver a quedar sin escenario y nadie lo va a
    notar.
  - **Redacción alternativa**: agregar al final una tabla de tres columnas (FR · Historia · Escenarios
    que lo ejercitan) con una fila por FR, y marcar explícitamente las filas sin escenario.

- [x] **CHK054** ¿Cada requisito tiene identificador único y estable? — **OK**
  - No hay identificadores duplicados ni reutilizados. FR-053 a FR-055 se agregaron por anexión en
    lugar de renumerar, lo que preserva las referencias cruzadas existentes. La numeración fuera de
    orden dentro de los grupos es un costo aceptable frente a romper referencias.

- [ ] **CHK055** ¿Hay requisitos huérfanos que no respondan a ninguna necesidad declarada? — **NO APLICA**
  - Revisados los 55 FR y 19 NFR, no se encontró ninguno que no responda a una necesidad del
    documento de entrada, a un principio de la constitución o al hardware declarado. El problema de
    CHK050 es de trazabilidad hacia arriba (falta la historia), no de requisitos sin propósito.

- [ ] **CHK056** ¿Están declaradas las dependencias externas del proyecto? — **NO APLICA**
  - El sistema es cien por ciento local y sin dependencias de servicios externos por diseño
    (Principio XI). La única dependencia real —el conjunto de frases de referencia— se audita en
    CHK045.

---

## Veredicto

**NO LISTA PARA PLANIFICAR.**

La especificación está bien estructurada y tiene virtudes reales: el registro cerrado está bien
protegido (CHK016, CHK017, CHK020), la alineación con la constitución es explícita y verificable, y
no hay filtraciones de tecnología. Pero **26 de 55 ítems fallan**, y siete de esas fallas atacan
directamente los tres modos de fallo más caros del proyecto.

### Bloqueantes antes de `/speckit-plan`

Ordenados por riesgo. Los siete primeros son los que harían que el plan se construya sobre
requisitos indecidibles.

| # | Ítem | Qué falta | Riesgo |
|---|------|-----------|--------|
| 1 | CHK014 | Comportamiento ante **acción ambigua entre dos candidatas**. No es ninguna de las cuatro clases de error declaradas. Sin regla, se desempata por puntaje. | (a) |
| 2 | CHK018 | **Qué cuenta como aceptar y cancelar** una confirmación. Sin canal declarado, una voz de fondo puede confirmar una acción destructiva. | (a) |
| 3 | CHK045 | Definición del **conjunto de frases de referencia** (tamaño, composición, procedimiento). Ocho requisitos numéricos dependen de él; NFR-015 (<1%) es inmedible con menos de 100 entradas. | (c) |
| 4 | CHK027 | **NFR-006/007 contradicen FR-026/FR-055**: el techo de latencia incluye hasta 10 s de respuesta humana. | (c) |
| 5 | CHK028 | **Residencia del modelo de transcripción** sin declarar. NFR-002 (150 MB), NFR-003 (1,5 GB) y NFR-006 (2 s) pueden ser conjuntamente imposibles. | (b) |
| 6 | CHK008 | **Carga del sistema bajo la que se miden** los NFR. Sin ella, todo presupuesto de recursos y latencia se puede validar en una máquina ociosa. | (b), (c) |
| 7 | CHK010 | **NFR-005 y SC-006 sin proxy objetivo** ("fluidez perceptible"). La protección del usuario está escrita de forma que nadie puede reprobarla. | (b), (c) |
| 8 | CHK022, CHK024, CHK025 | **Máquina de estados**: sin matriz de transiciones; `escuchando`, `procesando` y `error` sin salida acotada; sin duración máxima de captura. | (a), (b) |
| 9 | CHK030 | **Colisión terminológica de "registro"** (catálogo de acciones vs bitácora). | (a) |
| 10 | CHK033 | Tres de las cinco decisiones de clarificación **sin escenario de aceptación**. | (c) |
| 11 | CHK048 | **FR-018 (búsqueda web) vs SC-007** (verificación con la red inhabilitada). | (c) |
| 12 | CHK013, CHK041 | **Meta-requisitos que exigen una declaración sin declararla** (NFR-009, FR-052). | (c) |
| 13 | CHK002, CHK026 | **US4-3 referencia un valor inexistente**; **FR-034 contradice FR-039**. | (c) |
| 14 | CHK019 | **Texto dictado de FR-018 sin validación** declarada. | (a) |
| 15 | CHK023, CHK039 | **Cancelación**: sin canal en estado esperando confirmación; sin regla para cancelación durante la ejecución. | (a) |

### Mejoras recomendadas, no bloqueantes

- **CHK032** — Glosario de términos canónicos. Es la causa raíz de CHK030 y CHK031.
- **CHK053** — Matriz de trazabilidad FR ↔ historia ↔ escenario. Es lo que evita que CHK001 y
  CHK033 se repitan en la próxima ronda.
- **CHK001, CHK004** — Escenarios y caminos de fallo para los 14 FR sin cobertura.
- **CHK050** — Historia US7 de robustez, para adoptar FR-049 a FR-052.
- **CHK031** — Normalizar "interfaz" a "superficie de estado".
- **CHK006** — Separar FR-021 en tres requisitos, uno por acción.
- **CHK003** — Reescribir FR-012 y FR-050 con verbos observables.
- **CHK037, CHK038** — Casos borde faltantes: sin ventana activa, sin dispositivo de salida, recarga
  de configuración durante un turno, configuración inválida al arrancar.
- **CHK034, CHK035** — Criterio de desempate en FR-015; precedencia entre configuración y FR-028;
  consecuencia declarada del movimiento de ventana en FR-017.
- **CHK042, CHK043** — Acotar el alcance de FR-041 y dar proxies objetivos a los siete requisitos
  que hoy dependen de percepción humana.
- **CHK047** — Acotar la exclusión de "consultas que devuelven información" a la vía vocal.

### Camino sugerido

Los bloqueantes 1, 2, 8 y 15 son decisiones de producto y conviene resolverlos con
`/speckit-clarify` en una segunda sesión. Los bloqueantes 3, 4, 5, 6 y 7 son de redacción de
requisitos y se pueden corregir editando la spec directamente con las alternativas propuestas acá.
El resto son correcciones mecánicas.
