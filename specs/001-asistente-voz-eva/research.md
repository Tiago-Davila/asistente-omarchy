# Research — Eva, fase 0

**Feature**: `001-asistente-voz-eva` | **Date**: 2026-08-09 | **Plan**: [plan.md](plan.md)

Hardware objetivo de todas las decisiones: AMD Ryzen 5 3450U (Zen+, 4 núcleos), Radeon Vega
integrada **no utilizable para inferencia**, 8 GB DDR4 compartidos con la iGPU, Arch + Hyprland +
Wayland + PipeWire. Memoria realmente disponible ≈ 3 GB, núcleos realmente libres ≈ 1.

> **Sobre las mediciones.** El encargo exige mediciones tomadas en el hardware objetivo y prohíbe
> sustituirlas por documentación. Esta sesión de planificación corrió en Windows, sin acceso a la
> laptop. **Ninguna tabla de este documento contiene números medidos.** Donde hacía falta medir,
> hay una tabla vacía, un protocolo reproducible y una regla de decisión que selecciona la opción
> sin más criterio humano. Los rangos que sí aparecen están rotulados como *estimación de
> planificación* y existen solo para dimensionar presupuestos; ninguna decisión bloqueante se apoya
> en ellos.

---

## D-01 · Estructura de procesos

**Decisión**: cuatro procesos — `eva-daemon` (Java, residente), `eva-overlay` (Rust, residente
oculto), `pw-record` (bajo demanda), `eva-stt` (bajo demanda).

**Rationale**: dos restricciones fuerzan la separación y ninguna es negociable. Primero, ninguna
JVM puede crear una superficie `wlr-layer-shell` (ver D-10), así que la superficie sale sí o sí a
otro proceso. Segundo, NFR-002 acota el reposo a 150 MB, y cualquier motor de transcripción
residente consume entre el 40% y el 160% de ese presupuesto él solo, así que el STT sale de
residencia. Una vez aceptados esos dos, la captura también conviene afuera: si no existe proceso de
audio fuera del turno, FR-002 se cumple por construcción y no por disciplina de código, que es una
garantía mucho más fuerte y mucho más fácil de auditar.

**Alternativas consideradas**:

- *Proceso único Java*. Rechazada por las dos restricciones de arriba.
- *Daemon + un solo auxiliar multipropósito*. Rechazada: acopla el ciclo de vida de la superficie
  (residente) al del STT (efímero), y hace que un fallo del motor de transcripción se lleve puesta
  la superficie, violando el Principio V.
- *Todo bajo demanda, sin residentes*. Rechazada: el arranque de la JVM (300–800 ms) y el de
  cualquier toolkit gráfico (150–300 ms) exceden los 100 ms de NFR-008.

**Justificación de cada residente contra NFR-002** — ver la tabla de presupuesto en plan.md. El
argumento en una línea: los residentes suman 68–95 MB estimados sobre 150 MB, y los dos que
podrían haberlo roto (STT y superficie GTK) están explícitamente fuera por diseño.

---

## D-02 · Protocolo entre procesos

**Decisión**: sockets de dominio Unix en `$XDG_RUNTIME_DIR/eva/`, con **JSON Lines** —un objeto
JSON por línea terminada en `\n`— para control y estado; **PCM crudo por tubería** para audio.

**Rationale**: Java 21 habla AF_UNIX de forma nativa (`UnixDomainSocketAddress`, desde Java 16), así
que el IPC no introduce JNI ni dependencias nativas en el daemon. JSON Lines es delimitable sin
ambigüedad, legible en `socat` durante depuración, y trivial de generar desde Rust. Los permisos
del socket quedan en `0600` bajo `$XDG_RUNTIME_DIR`, que ya es privado del usuario: eso satisface
NFR-010 sin criptografía. El audio no pasa por JSON porque codificarlo sería puro desperdicio de
CPU en la máquina más ajustada del proyecto.

**Alternativas consideradas**:

- *D-Bus*. Rechazada: agrega una dependencia de sesión, una biblioteca cliente en Java que no es de
  primera línea, y latencia de intermediación, a cambio de un descubrimiento de servicios que con
  cuatro procesos y rutas fijas no hace falta.
- *gRPC / Protobuf*. Rechazada: la generación de código y el runtime pesan más que todo el
  protocolo de Eva, que tiene ~10 tipos de mensaje.
- *Sockets TCP en localhost*. **Rechazada por el Principio XI**: abrir un socket de red, aunque sea
  en loopback, contradice "ningún componente del asistente establece conexiones de red" (NFR-011) y
  hace la auditoría de SC-007 discutible en vez de binaria.
- *FIFOs con nartefacto de líneas*. Rechazada: no soportan múltiples clientes ni detectan
  desconexión limpiamente, que es lo que necesita la supervisión de auxiliares (FR-049).

---

## D-03 · Configuración de la JVM

**Decisión**: HotSpot con `-XX:+UseSerialGC -Xms16m -Xmx96m -XX:MaxMetaspaceSize=64m
-XX:TieredStopAtLevel=1 -Xss512k` más **AppCDS** con archivo generado en instalación.
**No se usa imagen nativa en fase 1**, y se deja documentada como escalón de mitigación de R-02.

**Rationale**:

- *SerialGC* es la elección correcta acá por una razón que no es el tamaño del heap sino el conteo
  de núcleos. G1 arranca hilos de recolección paralelos y un hilo concurrente; en una máquina donde
  hay que dejar un núcleo libre (Principio VI, NFR-004), esos hilos compiten con el trabajo del
  usuario en el peor momento posible, que es durante el turno. Serial recolecta en el hilo que
  asigna, con pausas irrelevantes para un heap de decenas de MB y sin hilos de fondo.
- *`TieredStopAtLevel=1`* deja solo C1. Un daemon que hace ráfagas cortas de trabajo intermitente
  nunca alcanza el punto donde C2 amortiza su costo: pagaríamos hilos de compilación y code cache
  para código que se ejecuta unas decenas de veces por día. C1 compila rápido, barato y suficiente.
- *AppCDS* recorta el arranque y, más importante para NFR-002, permite que los metadatos de clase
  se mapeen desde archivo en vez de reconstruirse en memoria privada.
- *Heap máximo de 96 MB* con `Xms` bajo: el daemon no acumula estado grande; el catálogo compilado
  y la máquina de estados son kilobytes. El techo existe para que una fuga se manifieste como
  `OutOfMemoryError` ruidoso (Principio XVI) en vez de como una violación silenciosa de NFR-002.

**Sobre imagen nativa (GraalVM)**. Se evaluó explícitamente porque es la respuesta obvia a un
presupuesto de 150 MB. Lo que se gana: RSS de ~20–30 MB en vez de 60–80, y arranque de ~10 ms.
Lo que se pierde, y por qué pesa más hoy:

1. **Configuración de reflexión y recursos**: el cargador de configuración y la validación de
   esquema son exactamente el tipo de código que native-image rompe, y cada motor STT con JNI
   agrega su propia configuración. Es trabajo de integración que compite con construir la feature.
2. **JNI del motor de transcripción**: si D-05 elige un motor con binding JNI, native-image exige
   configuración adicional y complica el diagnóstico de fallos nativos.
3. **Ciclo de desarrollo**: compilaciones de minutos en una laptop de 4 núcleos, contra segundos en
   HotSpot. En la máquina objetivo, eso es un costo real de todos los días.
4. **No hace falta todavía**: el presupuesto estimado deja 55–82 MB de margen. Gastar complejidad
   para recuperar 40 MB que no necesitamos es optimización prematura.

La decisión se revierte automáticamente si el gate de R-02 falla: si el daemon medido supera
100 MB tras AppCDS y `jlink`, native-image entra como escalón 3.

**Alternativas consideradas**: G1 (rechazada por los hilos de fondo), ZGC/Shenandoah (rechazadas:
su footprint base excede el presupuesto entero), `-Xshare:auto` sin AppCDS a medida (menor
beneficio), OpenJ9 (menor RSS, pero es una JVM menos común en Arch y agrega riesgo de
disponibilidad sin resolver el problema estructural).

---

## D-04 · Captura de audio

**Decisión**: lanzar `pw-record` como proceso hijo por turno, escribiendo **PCM s16le 16 kHz mono**
a stdout, leído por el daemon. La captura se cierra matando el hijo.

**Rationale**: tres criterios, y `pw-record` gana los tres.

- *Confiabilidad ante cambio de dispositivo* (EC-12, FR-051): `pw-record` conectado al sink por
  defecto sigue el reencaminamiento de PipeWire; cuando el dispositivo desaparece de verdad, el
  proceso muere y el daemon lo ve como EOF, que es una señal de error limpia y trivialmente
  testeable. Java Sound sobre la capa de compatibilidad ALSA de PipeWire reporta cambios de
  dispositivo de forma inconsistente y a veces se cuelga en `read()`, que es el peor modo de fallo
  posible para un estado con salida acotada (NFR-023).
- *Cumplimiento de FR-002*: fuera del turno no existe proceso de audio. La garantía "el asistente no
  captura audio" pasa de ser una propiedad del código a ser verificable con `ps`, que es lo que
  pide el escenario US2-4.
- *Latencia de arranque*: el fork/exec de `pw-record` es del orden de decenas de milisegundos y
  ocurre **al inicio del turno**, no al final del habla, así que queda fuera del presupuesto de
  NFR-006. El arranque preciso se mide en Q1 del quickstart.

**Alternativas consideradas**:

- *`javax.sound.sampled` directo*. Rechazada por el manejo de cambio de dispositivo y porque
  mantener una línea de audio abierta desde la JVM hace de FR-002 una promesa en vez de un hecho.
- *Binding JNI a `libpipewire`*. Rechazada: máximo control, máximo costo. Introduce código nativo en
  el daemon, que es justo lo que la separación de procesos evita, y no compra latencia relevante.
- *`parec` (PulseAudio)*. Rechazada: pasa por la capa de compatibilidad en vez de hablar con
  PipeWire de forma nativa, agregando un intermediario sin beneficio.

---

## D-05 · Transcripción — **DECISIÓN ABIERTA, BLOQUEADA POR MEDICIÓN**

Es la decisión de mayor riesgo del proyecto (R-01) y la única que este plan **no** cierra.

### Candidatos

| Id | Motor | Modelo | Modo | Integración con Java | Por qué está en la lista |
|---|---|---|---|---|---|
| **A** | Vosk (Kaldi) | `vosk-model-small-es-0.42` | **Streaming** | Binding JNI oficial en Maven Central | Decodifica mientras el usuario habla: al terminar el enunciado, casi todo el trabajo ya está hecho. Es el único candidato que apunta a la vía de streaming del presupuesto de latencia. |
| **B** | whisper.cpp | `ggml-base` cuantizado (q5_0) | Por lotes | Proceso separado, texto por stdout | Referencia de calidad razonable con huella chica. Sin dependencia de JNI. |
| **C** | whisper.cpp | `ggml-small` cuantizado (q5_0) | Por lotes | Proceso separado | Techo de calidad que permite la constitución (STT ≤ `whisper small`). Es el candidato más caro en memoria y tiempo. |
| D *(reserva)* | faster-whisper (CTranslate2) | `small` int8 | Por lotes | Proceso Python separado | Solo si A, B y C fallan. Introduce Python en el camino crítico, lo que exige justificación adicional. |

**Por qué Vosk encabeza la lista.** El presupuesto de NFR-006 se mide **desde el fin del habla**.
Un motor por lotes empieza a trabajar recién ahí y tiene que procesar el enunciado completo; un
motor con decodificación incremental llega al fin del habla con el grueso ya decodificado y solo
necesita finalizar. La diferencia en el presupuesto de plan.md es 150 ms contra 900 ms en p50. En
una máquina con un núcleo libre, esa diferencia es la que separa holgura de riesgo permanente.

**La contrapartida honesta**: el modelo chico de Vosk en español tendrá peor tasa de error que
whisper, sobre todo con nombres propios en inglés. Eso importa menos de lo que parece **porque el
modo de fallo del sistema es el rechazo**: una transcripción mala cae en FR-009 o FR-056 y termina
en rechazo, no en acción equivocada. NFR-015 acota lo que de verdad duele —ejecutar otra cosa— al
1%, mientras que NFR-014 tolera un 10% de fallos. El diseño ya está sesgado a preferir un motor
rápido que rechaza sobre uno lento que acierta más.

### Protocolo de medición

Se ejecuta **en la laptop objetivo**, bajo la **carga de referencia** (navegador con 10 pestañas y
editor con proyecto cargado, sin compilación en curso). Sin esto los números no valen.

1. **Corpus de medición**: 20 enunciados grabados por el usuario, de 2 a 5 segundos, en español
   rioplatense, tomados del conjunto de frases de referencia (FR-063). Al menos 5 deben contener un
   nombre de aplicación o sitio en inglés (FR-064). Se graban una sola vez con
   `pw-record --rate 16000 --channels 1 --format s16` y se reutilizan para los tres candidatos, de
   modo que la comparación sea sobre el mismo audio.
2. **Por cada candidato**, tres corridas completas del corpus, descartando la primera (caché de
   página frío). Se reporta la mediana y el p95 de las dos restantes.
3. **Métricas a registrar**:
   - `rss_carga_mb` — RSS del proceso tras cargar el modelo, antes de transcribir, por
     `/proc/<pid>/smaps_rollup` campo `Pss`.
   - `t_carga_ms` — desde el `exec` hasta que el motor acepta audio.
   - `t_transcripcion_ms` — **desde el fin acústico del habla hasta el texto final**. Para los
     motores por lotes coincide con el tiempo total de procesamiento; para el de streaming es solo
     la finalización. Esta es la métrica que entra en el presupuesto.
   - `hilos` — máximo de hilos del proceso durante la transcripción, por `/proc/<pid>/status`.
   - `cpu_pico_pct` — pico de CPU del conjunto, por `pidstat -p <pid> 1`.
   - `aciertos` — cuántos de los 20 enunciados producen la acción esperada al pasar por el
     resolutor de intención. Se mide **la acción resuelta, no la tasa de error de palabra**, porque
     es lo que le importa al producto.

### Tabla de resultados — **A COMPLETAR EN EL HARDWARE OBJETIVO**

| Candidato | `rss_carga_mb` | `t_carga_ms` | `t_transcripcion_ms` p50 | p95 | `hilos` | `cpu_pico_pct` | `aciertos` /20 |
|---|---|---|---|---|---|---|---|
| A · Vosk small-es | — | — | — | — | — | — | — |
| B · whisper.cpp base-q5_0 | — | — | — | — | — | — | — |
| C · whisper.cpp small-q5_0 | — | — | — | — | — | — | — |

### Regla de decisión

Se aplica en orden; el primer candidato que cumpla **todas** las condiciones de un escalón, gana.
La regla existe para que la elección no dependa de quién mire la tabla.

1. **Escalón preferente** — `t_transcripcion_ms` p95 ≤ 400 ms **y** `rss_carga_mb` ≤ 250 **y**
   `cpu_pico_pct` ≤ 150 **y** `aciertos` ≥ 18/20. Habilita la vía de streaming del presupuesto.
2. **Escalón aceptable** — p95 ≤ 2000 ms **y** `rss_carga_mb` ≤ 400 **y** `cpu_pico_pct` ≤ 150
   **y** `aciertos` ≥ 18/20. Habilita la vía por lotes, con menos de 1 s de holgura en p95.
3. **Ninguno cumple** → aplica la contingencia.

Si dos candidatos cumplen el mismo escalón, gana el de menor `rss_carga_mb`, porque la memoria es
el recurso más escaso de la máquina y el que la constitución acota con más dureza.

### Plan de contingencia si ningún candidato entra

Escalera en orden estricto. **Lo primero que se sacrifica es la calidad de transcripción; lo último,
el umbral de latencia.** El motivo es asimétrico y vale declararlo: un error de transcripción se
manifiesta como rechazo (FR-009, FR-056) y le cuesta al usuario repetir la frase; un umbral de
latencia relajado degrada cada turno para siempre y convierte el asistente en algo más lento que el
atajo de teclado que pretende reemplazar, que es el fin del producto.

1. **Bajar el tamaño del modelo** dentro del mismo motor (small → base → tiny). Cuesta aciertos,
   que el conjunto de referencia mide.
2. **Acotar el vocabulario**: los motores basados en Kaldi aceptan una gramática cerrada en la
   decodificación. Alimentar el decodificador con el vocabulario del catálogo de acciones más los
   nombres declarados en configuración reduce drásticamente el espacio de búsqueda y sube los
   aciertos sobre ese vocabulario, a costa de no reconocer nada fuera de él — que para un asistente
   de comandos con catálogo cerrado es una pérdida aceptable, salvo en el término libre de búsqueda
   de FR-018.
3. **Recortar FR-018**: si la búsqueda web con texto libre es lo único que exige vocabulario
   abierto, se puede degradar a un modo de dos pasos o sacarla de la fase 1 antes de tocar la
   latencia.
4. **Recién entonces, relajar NFR-006/NFR-007** vía `/speckit-clarify`, con el número medido a la
   vista y no como concesión anticipada.

### Residencia y descarga

**Bajo demanda**, lanzado al **inicio del turno** y descargado tras **120 s** de inactividad. La
carga transcurre en paralelo con el habla del usuario y por lo tanto no consume presupuesto de
NFR-006, que se mide desde el fin del habla. El peor caso es un enunciado más corto que el tiempo
de carga; con `t_carga_ms` medido se sabrá si hace falta ajustar el umbral de descarga hacia arriba
para que la mayoría de los turnos encuentren el motor caliente.

---

## D-06 · Detección de fin de habla

**Decisión**: VAD por **energía de corto plazo + tasa de cruces por cero**, en Java puro, con
ventana de 20 ms y *hangover* configurable (por defecto 250 ms). El puerto `DetectorFinHabla` deja
la puerta abierta a `VadSileroOnnx` como implementación alternativa.

**Rationale**: el criterio dominante es el presupuesto, no la exactitud. Un VAD neuronal en Java
exige ONNX Runtime, que agrega ~20 MB de biblioteca nativa **residente** en el daemon —más del 13%
del presupuesto total de NFR-002— para resolver un problema que en un escritorio, con el usuario
hablándole de cerca al micrófono, un detector clásico resuelve razonablemente. El costo de CPU de
energía + ZCR sobre ventanas de 20 ms es despreciable frente al núcleo libre de NFR-004.

Más importante: **el diseño no depende de que el VAD sea bueno**. FR-061 (10 s sin voz) y FR-062
(20 s de tope duro) son cortes por tiempo que garantizan la salida del estado `escuchando` aunque
el VAD falle por completo, y NFR-023 lo exige explícitamente. El VAD optimiza el caso normal; los
topes cubren el patológico.

**Alternativas consideradas**:

- *Silero VAD vía ONNX Runtime*. Mejor exactitud en ambientes ruidosos, ~1–2 MB de modelo, pero
  ~20 MB de runtime nativo residente. Queda como implementación alternativa detrás del puerto, a
  activar si el conjunto de referencia muestra que el corte prematuro o tardío es una causa real de
  fallo.
- *VAD dentro del proceso de captura*. Rechazada para fase 1: mueve lógica de dominio a un
  auxiliar y complica el testeo sin audio (FR-041). El VAD en Java se prueba alimentándolo con
  arreglos de PCM desde un test.
- *Delegar el fin de habla al motor STT*. Rechazada: acopla la detección al motor, que es
  justamente la decisión abierta de D-05, y rompe la intercambiabilidad del Principio VIII.

---

## D-07 · Resolución de intención

**Decisión**: **gramática de slots declarada en configuración** y compilada a un matcher
determinístico al cargar. Cuatro etapas: normalización → reescritura morfológica → emparejamiento
de patrones → extracción y tipado de parámetros.

**Diseño**:

1. **Normalización**: minúsculas; eliminación de tildes preservando `ñ`; colapso de espacios;
   remoción de muletillas iniciales y finales declaradas en configuración (`che`, `dale`, `eh`,
   `por favor`, `a ver`). La lista es data, no código (Principio XIII).
2. **Reescritura morfológica rioplatense**: tabla declarada que mapea formas de imperativo voseante
   a un lema canónico — `abrí|abrime|abrilo|abrila|abrí​melo → abrir`; `andá|andate|anda → ir`;
   `cerrá|cerrala|cerralo → cerrar`; `mové|movela → mover`; `buscá|buscame → buscar`;
   `poné|ponele → poner`. Los pronombres enclíticos (`me te le lo la se nos los las`) se despegan
   del final de la forma verbal antes de buscar el lema. Esto es lo que hace que FR-011 sea
   cumplible sin un modelo.
3. **Emparejamiento**: cada acción declara uno o más patrones con slots, por ejemplo
   `abrir {app}` y `abrir {app} en (el )?(escritorio|espacio|pestaña) {n}`. Los patrones se
   compilan a autómatas. **El matcher recolecta todos los patrones que emparejan, no corta en el
   primero**: sin esto FR-056 sería inimplementable, porque no se puede rechazar por ambigüedad si
   nunca se llega a ver el segundo candidato.
4. **Extracción y tipado**: cada slot declara su tipo (`app`, `sitio`, `entero`, `texto_libre`) y su
   validación. Los slots de entidad se resuelven contra las tablas declaradas: primero coincidencia
   exacta, después coincidencia normalizada, y **nunca coincidencia difusa que cambie de acción**.
   Se admite distancia de edición ≤ 1 solo para elegir *entre entidades del mismo tipo* y solo para
   nombres de 5 caracteres o más; por debajo del umbral se rechaza en vez de adivinar (FR-009).
5. **Nombres propios en inglés**: se resuelven por el mismo mecanismo de entidades, porque el nombre
   está declarado literalmente en configuración. Las variantes de pronunciación se agregan como
   alias (FR-043). No hace falta tratamiento fonético especial en fase 1; FR-064 obliga a que el
   conjunto de referencia contenga estos casos, así que si el enfoque no alcanza, se descubre
   midiendo.

**Decisión entre dos candidatas**: si tras el emparejamiento hay más de un **id de acción distinto**
con todos sus parámetros válidos, se rechaza con `intencion_ambigua` nombrando las candidatas
(FR-056). Dos patrones de la *misma* acción que producen los *mismos* parámetros no son ambigüedad
y se colapsan. Dos patrones de la misma acción con parámetros distintos **sí** son ambigüedad.

**Reglas como datos, no como código** (Principio XIII, FR-042): la gramática, las muletillas, la
tabla morfológica y los alias viven en TOML. Agregar un fraseo no recompila nada. La contrapartida
es que un error de configuración puede romper la resolución, y por eso FR-045 y el validador de
esquema son parte del mismo diseño.

**Alternativas consideradas**:

- *Expresiones regulares sueltas por acción*. Rechazada: no compone, no da parámetros tipados, y
  detectar ambigüedad entre regex es impracticable.
- *Clasificador estadístico de intención entrenado localmente*. Rechazada por el Principio IV: hay
  una expresión por reglas disponible, así que corresponde la regla.
- *Levenshtein global sobre la frase completa*. Rechazada frontalmente: es exactamente el
  "desempate por puntaje" que FR-056 prohíbe y el mecanismo por el cual un asistente ejecuta otra
  cosa.

---

## D-08 · Registry de acciones

**Decisión**: el catálogo se declara en TOML dentro de la configuración del usuario y se valida
contra un **JSON Schema** ([contracts/registry-schema.json](contracts/registry-schema.json)) al
cargar, tras convertir el TOML a un árbol JSON. Una acción nueva es una entrada nueva de TOML.

**Rationale**: TOML porque la configuración la edita una persona y necesita comentarios, que JSON no
tiene. JSON Schema porque la validación tiene que ser declarativa y producir errores localizados que
mapeen a las clases de FR-031 y al comportamiento de FR-045. Convertir TOML→JSON en memoria y
validar contra el esquema da lo mejor de los dos sin inventar un validador propio.

**Cómo se agrega una acción sin tocar el núcleo**: una acción del catálogo declara `id`, `patrones`,
`parametros` (con tipo, obligatoriedad y dominio), `destructiva`, y un `ejecutor` que referencia un
despachador conocido por id. Las **acciones propias** (FR-023) usan el ejecutor `proceso`, cuyo
`argv` está **declarado como arreglo en la configuración** — nunca como cadena que se parte, y nunca
compuesto con texto de la transcripción. Eso mantiene el Principio III: el modelo elige un id y
llena parámetros tipados; jamás produce una línea de comandos.

**Validación previa a la ejecución** (FR-024): tres capas, en orden. (1) Esquema, al cargar la
configuración: estructura y tipos. (2) Tipado del slot, al resolver: el valor extraído se convierte
al tipo declarado o falla. (3) Dominio, antes de ejecutar: rango para enteros —el rango de espacios
de trabajo se **lee de Hyprland**, no se hardcodea— y pertenencia al conjunto para enumerados. Un
fallo en cualquiera produce `parametro_invalido` y aborta el turno.

**Alternativas consideradas**: YAML (rechazada por sus trampas de tipado implícito, que en un
archivo de seguridad son un riesgo innecesario); JSON puro (rechazada por la ausencia de
comentarios); acciones como clases Java con anotaciones (rechazada por el Principio XIII: obligaría
a recompilar para agregar una aplicación).

---

## D-09 · Ejecución sobre el entorno de escritorio

**Decisión**: hablar **directamente con el socket IPC de Hyprland**
(`$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket.sock`) desde Java por AF_UNIX. **No se
invoca `hyprctl`.**

**Rationale**: tres criterios, y el socket gana los tres.

- *Costo por invocación*: `hyprctl` es un cliente del mismo socket. Invocarlo cuesta un fork/exec
  (~10–20 ms) por comando, y FR-015 necesita **dos o tres** operaciones en un solo turno (consultar
  clientes, mover ventana, enfocar). El socket directo evita 30–60 ms de puro overhead sobre un
  presupuesto de despacho de 90 ms en p95.
- *Lectura de estado*: FR-014 y FR-015 requieren saber si la aplicación ya está abierta y dónde. El
  socket responde `j/clients`, `j/activeworkspace` y `j/monitors` en JSON. El vínculo entre "la
  aplicación que el usuario nombró" y "esta ventana" se hace por el campo `class` del cliente
  contra `StartupWMClass` de la entrada `.desktop`. Ese es el detalle que hace implementable la
  política de instancia única, y conviene dejarlo escrito acá porque no es obvio.
- *Manejo de errores*: el socket devuelve `ok` o una cadena de error, distinguible sin parsear
  salida de terminal ni interpretar códigos de retorno.

**Descubrimiento sin hardcodear** (Principio XIII, FR-042):

- *Aplicaciones*: se recorren las entradas `.desktop` de `$XDG_DATA_DIRS/applications` y
  `$XDG_DATA_HOME/applications`, tomando `Name`, `Exec`, `StartupWMClass` y `NoDisplay`. La
  configuración del usuario declara **alias hacia el id de la entrada**, no rutas de binarios.
- *Navegador por omisión*: `xdg-settings get default-web-browser`, con respaldo en el handler de
  `x-scheme-handler/https` de `mimeapps.list`. Nunca un nombre de navegador en el código.
- *Rango de espacios de trabajo*: de `j/monitors` y la configuración de Hyprland, no una constante.

**Alternativas consideradas**: `hyprctl` (rechazada por el costo por invocación y por tener que
parsear salida); `wlr-foreign-toplevel-management` vía Wayland (rechazada: da listado de ventanas
pero no las operaciones de espacio de trabajo que necesita el catálogo, y exigiría un cliente
Wayland en el daemon); D-Bus con un servicio de escritorio (rechazada: Hyprland no expone ahí lo que
hace falta).

---

## D-10 · Superficie de estado

**Decisión**: binario **Rust** residente usando `smithay-client-toolkit` con el protocolo
`wlr-layer-shell`, anclado a un borde, con `keyboard-interactivity: none`. Plan B declarado:
**`eww`**, con el mismo contrato de socket.

**Rationale**: la restricción de plataforma es dura y conviene enunciarla sin rodeos. Una superficie
anclada que no toma el foco en Wayland requiere el protocolo `wlr-layer-shell`, que **ningún toolkit
de la JVM implementa**: ni AWT, ni Swing, ni JavaFX exponen la capacidad, porque el protocolo no
tiene equivalente en X11 ni en el modelo de ventanas que esos toolkits abstraen. Java, acá, no es
peor: es inviable. Por eso el encargo autoriza otro lenguaje y por eso este componente sale a un
proceso.

Elegido Rust sobre las otras rutas viables por **memoria residente**, que es el recurso que
gobierna. Estimaciones de planificación, a confirmar en Q2 del quickstart:

| Opción | RSS estimado | Madurez | Costo de integración | Veredicto |
|---|---|---|---|---|
| **Rust + smithay-client-toolkit** | 8–15 MB | Alta — es la base de varios compositores y clientes | Media: hay que escribir el render | **Elegida** |
| `eww` (Rust, ya empaquetado) | 20–35 MB | Alta, en uso amplio en Hyprland | Baja: se configura, no se programa | **Plan B** |
| Python + GTK4 + `gtk4-layer-shell` | 40–60 MB | Alta | Baja | Rechazada: ~30% del presupuesto de NFR-002 en un solo auxiliar |
| Binding JNI a `gtk4-layer-shell` desde la JVM | 50–70 MB + JNI | Media | **Alta** | Rechazada: paga el costo de GTK *y* mete código nativo en el daemon, que es lo que la separación de procesos evita |
| `wlroots` C a mano | 5–10 MB | Alta | Alta | Rechazada: el ahorro sobre Rust no justifica escribir C |

`eww` es plan B y no primera opción porque, aunque cuesta menos trabajo, duplica el consumo y
agrega una dependencia externa con su propio lenguaje de configuración. Si el desarrollo del binario
Rust se estanca (R-03), el cambio a `eww` no toca el daemon: ambos consumen el mismo
`overlay.sock`.

**Contrato daemon ↔ superficie** — detalle completo en
[contracts/ipc-protocol.md](contracts/ipc-protocol.md):

- *Daemon → superficie*: mensajes `estado` con el estado actual de los seis (FR-029), el texto
  entendido y la acción resuelta con sus parámetros (FR-030); mensaje `ocultar` al terminar el turno
  (FR-037, NFR-022).
- *Superficie → daemon*: eventos `confirmar` y `cancelar`.
- *Cómo se confirma sin tomar el foco*: la superficie se declara con
  `keyboard-interactivity: none`, así que **no recibe teclas**. Las teclas de confirmación se
  registran como `bind` de Hyprland que invocan `evactl confirm` / `evactl cancel`, exactamente
  igual que el atajo de activación. El teclado nunca cambia de dueño y FR-036 y FR-038 se cumplen
  por construcción: no hay una ventana que pueda robar el foco porque la ventana nunca lo pide.
  Esta es la parte del diseño que más fácil se hace mal, y por eso la ruta de teclas **no pasa por
  la superficie**.

---

## D-11 · Atajo global y ciclo de vida

**Decisión**: el atajo se declara como `bind` de Hyprland que ejecuta `evactl activate`. El daemon
corre como **unidad de usuario de systemd** con `Restart=on-failure`. Los auxiliares los supervisa
el propio daemon.

**Rationale**:

- *Por qué `evactl` en Rust y no una CLI de JVM*: NFR-008 exige que el cambio de estado se observe
  en menos de 100 ms desde la activación. Un arranque de JVM consume ese presupuesto entero antes de
  hacer nada. `evactl` abre un socket, escribe una línea y termina; su costo es el fork/exec.
- *Por qué systemd y no `exec-once` de Hyprland*: `exec-once` lanza, pero no supervisa ni reinicia.
  Una unidad de usuario da reinicio con backoff, journal para diagnóstico, y —relevante para
  NFR-004— el slice donde se aplica **`CPUQuota=200%`**, que es el mecanismo que hace **exigible**
  el techo de CPU en vez de aspiracional. `CPUQuota` cuenta hilos lógicos: en esta máquina de 8
  hilos, 200% son dos hilos lógicos, es decir **un núcleo físico**, que es lo que pide NFR-004.
- *Supervisión de auxiliares*: el daemon los lanza y observa su tubería. Si la superficie muere, ve
  EOF, lo registra en la bitácora, **degrada a los canales de FR-033** y reintenta con backoff
  exponencial acotado. El Principio V prohíbe que un componente opcional sea condición de arranque:
  si la superficie no levanta al inicio, el daemon arranca igual y lo reporta.

**Orden de arranque**: la unidad depende de `graphical-session.target`, porque necesita
`$HYPRLAND_INSTANCE_SIGNATURE` y `$XDG_RUNTIME_DIR` para encontrar los sockets.

**Alternativas consideradas**: capturar el teclado global desde el daemon (rechazada: en Wayland no
existe atajo global para un cliente cualquiera, es prerrogativa del compositor — otra restricción de
plataforma, no una preferencia); un envoltorio de shell en vez de `evactl` (rechazada: `socat` o
`nc` como dependencia dura, y peor manejo de errores).

---

## D-12 · Configuración y bitácora

**Configuración**: TOML en `$XDG_CONFIG_HOME/eva/config.toml`, más `catalogo.toml` y
`gramatica.toml`. Se valida al cargar contra
[contracts/config-schema.json](contracts/config-schema.json).

**Comportamiento ante configuración inválida** (FR-045, EC-21): entrada por entrada. Las válidas
entran al catálogo, las inválidas quedan fuera y se reportan con su ruta dentro del archivo. Si el
archivo entero es ilegible, el daemon arranca con catálogo vacío y lo reporta. **Nunca se niega a
arrancar** y **nunca arranca en silencio**: es la combinación del Principio V con el XVI.

**Recarga sin reiniciar** (FR-044, EC-20): `evactl reload` releva la configuración y compila un
catálogo nuevo en memoria; el intercambio es una asignación de referencia atómica. Un turno en curso
**conserva la referencia al catálogo con el que resolvió la intención**, así que se completa contra
el catálogo viejo y el nuevo rige desde el turno siguiente. Esto es lo que hace correcta a EC-20 sin
bloqueos ni copias.

**Bitácora**: JSONL en `$XDG_STATE_HOME/eva/bitacora/AAAA-MM-DD.jsonl`, un objeto por turno, con
rotación diaria. Un objeto por turno y no por evento porque la unidad de análisis de FR-046 es el
turno. Campos y ejemplo en [contracts/ipc-protocol.md](contracts/ipc-protocol.md#bitácora).
Retención: **decisión diferida**, ver abajo.

---

## D-13 · Estrategia de testing

| Nivel | Qué prueba | Cómo | Requisito |
|---|---|---|---|
| Dominio | Normalización, gramática, tipado, clasificación de destructividad, ambigüedad | JUnit puro, sin E/S. `eva-dominio` no tiene dependencias que mockear porque no depende de nada | FR-041, Principio II |
| Arquitectura | Dirección de dependencias entre capas | ArchUnit: falla el build si el dominio importa infraestructura, `java.net`, `java.nio.file` o `javax.sound` | Principio IX |
| Catálogo | Cada acción con validación de parámetros y sus casos de rechazo | Test parametrizado sobre el catálogo cargado; **una acción sin test no entra** | Principio XIV |
| Pipeline sin audio | Turno completo por texto plano | Se inyecta el puerto `PcmSource` con una implementación de archivo; el orquestador no se entera | FR-040, FR-041, SC-012 |
| Pipeline con audio pregrabado | Captura → VAD → STT → intención | `PcmSource` lee un WAV en vez de `pw-record`. Mismo camino de código, sin micrófono | US2, SC-012 |
| Conjunto de referencia | Precisión y rechazo | 300 entradas en `tests/referencia/frases.jsonl`, versionadas en el repo. Test parametrizado que reporta aciertos, acciones equivocadas y rechazos, contrastando contra NFR-014/015/016 | FR-063…FR-067 |
| Presupuesto de memoria | NFR-001, NFR-002, NFR-003 | Arnés que arranca el daemon, espera reposo, muestrea `Pss` de `smaps_rollup` cada segundo durante 10 min y falla si supera el techo | NFR-001, NFR-002 |
| Presupuesto de latencia | NFR-006, NFR-007 | Se ejecuta el conjunto de referencia por la vía de audio pregrabado y se calculan p50/p95 desde los timestamps por etapa de la bitácora | NFR-006, NFR-007, NFR-009 |

**Construcción y versionado del conjunto de frases**: archivo JSONL, una entrada por línea, en el
repositorio junto al código, revisado como código. Cada entrada declara `texto`, y o bien
`accion_esperada` + `parametros_esperados`, o bien `rechazo_esperado` con su clase (FR-066). El
formato JSONL se elige para que el diff de un agregado sea una línea. Un test de estructura verifica
las cuotas de FR-063 a FR-065 —300 entradas, 20 por acción, 80 de rechazo, 10 por clase— y falla si
el conjunto se degrada.

---

## Decisiones diferidas

Cosas que este plan **no** decide, con el motivo y el momento en que corresponde decidirlas.

| # | Diferida | Por qué | Cuándo se decide |
|---|---|---|---|
| 1 | **Motor y modelo de transcripción (D-05)** | Requiere medición en hardware objetivo que esta sesión no puede hacer | Antes de `/speckit-tasks`, completando la tabla y aplicando la regla de decisión |
| 2 | Biblioteca JSON del daemon | No afecta arquitectura; es una elección de dependencia entre Jackson, `jsonp` y un parser mínimo propio. Depende de si D-05 trae Jackson por transitividad | Al escribir las tareas de infraestructura |
| 3 | ~~Retención y rotación de la bitácora~~ | — | **Cerrada** en la clarificación del 2026-08-09: rotación diaria, retención de 30 días, plazo configurable (FR-076) |
| 4 | ~~Colisión de alias duplicados~~ | — | **Cerrada** en la clarificación del 2026-08-09: se rechazan todas las entradas en conflicto al cargar, el resto sigue operando (FR-075, EC-24) |
| 5 | Empaquetado y distribución | Fuera de alcance de la fase 1 según la spec | Fase posterior |

---

## Riesgos

Los cuatro riesgos que el encargo pide declarar están desarrollados con su mitigación en
[plan.md § Riesgos](plan.md#riesgos), como R-01 a R-04, más dos adicionales que aparecieron al
planificar (R-05, conteo de hilos; R-06, nombres en inglés bajo cuantización). No se duplican acá
para que exista una sola fuente.
