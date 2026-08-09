# Data Model — Eva, fase 1

**Feature**: `001-asistente-voz-eva` | **Date**: 2026-08-09 | **Plan**: [plan.md](plan.md)

Modelo del **dominio**, en el sentido del Principio IX: nada de lo que sigue conoce audio, modelos
de inferencia, Hyprland ni el sistema de archivos. Todo lo que aparece acá vive en `eva-dominio` y
es construible desde un test sin ninguna dependencia.

---

## Entidades

### Turno

La unidad de interacción y la unidad de análisis de la bitácora. Un turno empieza con una activación
o con una entrada de texto, y termina cuando el sistema vuelve a reposo.

| Campo | Tipo | Reglas | Requisito |
|---|---|---|---|
| `id` | UUID | Único, generado al inicio | FR-046 |
| `inicio` | Instant | Reloj monótono para las duraciones, de pared para el registro | NFR-009 |
| `origen` | enum `VOZ` \| `TEXTO` | Determina si hubo captura | FR-040 |
| `textoEntrada` | String | Transcripción o texto plano recibido | FR-046 |
| `intencion` | Intencion? | Nulo si se rechazó antes de resolver | FR-008 |
| `accion` | AccionResuelta? | Nulo si se rechazó | FR-008 |
| `estadoFinal` | enum `EJECUTADO` \| `RECHAZADO` \| `CANCELADO` \| `FALLIDO` | Obligatorio al cerrar | FR-046 |
| `claseError` | ClaseError? | Obligatorio si `estadoFinal ≠ EJECUTADO` | FR-031, FR-047 |
| `etapaFallo` | Etapa? | Obligatorio si `estadoFinal ≠ EJECUTADO` | FR-047 |
| `duraciones` | Map\<Etapa, Duration\> | Una entrada por etapa atravesada | NFR-009 |
| `excedioPresupuesto` | boolean | Verdadero si la suma supera NFR-007 | EC-09 |

**Invariante**: un turno cerrado con `estadoFinal ≠ EJECUTADO` tiene siempre `claseError` y
`etapaFallo` no nulos. Es lo que hace cumplible a SC-010 sin depender de la disciplina de quien
escriba el código de error.

---

### AccionCatalogo

Una capacidad declarada. Es data cargada desde configuración, no una clase por acción.

| Campo | Tipo | Reglas | Requisito |
|---|---|---|---|
| `id` | String | `[a-z][a-z0-9_]*`, único en el catálogo | FR-008 |
| `patrones` | List\<PatronSlots\> | Al menos uno | FR-011 |
| `parametros` | List\<DefinicionParametro\> | Puede ser vacía | FR-008 |
| `destructiva` | boolean | Ver regla de resolución abajo | FR-028, FR-054 |
| `ejecutorId` | String | Debe existir en los despachadores conocidos | Principio III |
| `argv` | List\<String\>? | Solo para `ejecutorId = proceso`. **Arreglo declarado, nunca cadena partida ni compuesta con la transcripción** | FR-023, FR-025 |

**Resolución de destructividad** (FR-028, FR-054, FR-074), en este orden:

1. Si el `id` está en la lista cerrada de destructivas del núcleo (`cerrar_ventana`) → **destructiva,
   y la configuración no puede rebajarlo** (FR-074).
2. Si la configuración declara `destructiva = true` → destructiva (la configuración puede elevar).
3. Si es una acción propia (`ejecutorId = proceso`) y **no** declara `destructiva` → **destructiva**
   (FR-054, el default seguro).
4. En cualquier otro caso → no destructiva.

---

### DefinicionParametro

| Campo | Tipo | Reglas |
|---|---|---|
| `nombre` | String | Único dentro de la acción |
| `tipo` | enum `APP` \| `SITIO` \| `ENTERO` \| `TEXTO_LIBRE` \| `ENUM` | — |
| `obligatorio` | boolean | Si falta y es obligatorio → `parametro_invalido` (FR-010) |
| `dominio` | Dominio? | Rango para `ENTERO`, conjunto para `ENUM`, longitud para `TEXTO_LIBRE` |

**Dominio por tipo**:

- `APP` / `SITIO`: el valor debe resolver a una entidad declarada. Coincidencia exacta, después
  normalizada, y distancia de edición ≤ 1 **solo** para nombres de 5 caracteres o más. Por debajo
  del umbral se rechaza (FR-009): nunca se elige la entidad más parecida.
- `ENTERO`: rango cerrado. El rango de espacios de trabajo **se lee del gestor de ventanas**, no se
  declara como constante (Principio XIII, EC-05).
- `TEXTO_LIBRE`: 1 a 200 caracteres, escapado antes de usarse (FR-073).
- `ENUM`: pertenencia estricta al conjunto declarado.

---

### Intencion

Resultado intermedio de la resolución, antes de validar.

| Campo | Tipo |
|---|---|
| `accionId` | String |
| `valoresCrudos` | Map\<String, String\> |
| `patronQueEmparejo` | PatronSlots |

**No es una entidad persistida**: existe dentro del turno. Se modela aparte porque el resolutor
puede producir **varias** —y esa es exactamente la condición de FR-056.

---

### ResultadoResolucion

El tipo que hace implementable a FR-056. El resolutor **no** devuelve una intención: devuelve uno de
tres casos.

```
ResultadoResolucion =
  | Resuelta(Intencion)                       // exactamente un accionId distinto
  | Ambigua(List<String> accionesCandidatas)  // más de un accionId distinto → FR-056
  | NoResuelta(ClaseError)                    // no_entendido | accion_desconocida
```

**Regla de colapso**: dos intenciones con el mismo `accionId` **y** los mismos valores no son
ambigüedad; se colapsan a una. Dos con el mismo `accionId` y valores distintos **sí** son
ambigüedad. Dos con `accionId` distinto siempre son ambigüedad.

---

### ClaseError

Conjunto cerrado de cinco (FR-031). El tipo es un `enum`, no una cadena, para que el compilador
obligue a tratarlos todos.

| Valor | Cuándo aplica |
|---|---|
| `no_entendido` | Texto vacío o sin ningún token del vocabulario declarado (FR-072) |
| `accion_desconocida` | Texto con tokens declarados, sin coincidencia con ninguna acción (FR-072) |
| `intencion_ambigua` | Más de un `accionId` con parámetros válidos (FR-056) |
| `parametro_invalido` | Falta obligatorio, tipo incorrecto o fuera de dominio (FR-010, FR-024) |
| `fallo_al_ejecutar` | El ejecutor devolvió error (EC-08, EC-19, EC-22) |

---

### ConfiguracionUsuario

| Campo | Tipo | Requisito |
|---|---|---|
| `aplicaciones` | Map\<String, EntradaApp\> | FR-042 |
| `sitios` | Map\<String, EntradaSitio\> | FR-042 |
| `alias` | Map\<String, String\> | FR-043 |
| `muletillas` | List\<String\> | FR-011 |
| `reescriturasMorfologicas` | Map\<String, String\> | FR-011 |
| `accionesPropias` | List\<AccionCatalogo\> | FR-023 |
| `confirmacion` | EntradasConfirmacion | FR-057 |
| `superficieHabilitada` | boolean | FR-034, FR-039 |
| `atajos` | Map\<String, String\> | FR-001 |

**EntradasConfirmacion** son cuatro: `teclaAceptar`, `teclaCancelar`, `palabraAceptar`,
`palabraCancelar`. **Invariante de carga** (FR-058): ninguna de las dos palabras puede coincidir con
un token que active una acción del catálogo ni con un alias declarado. La colisión rechaza la
configuración.

---

### EntradaBitacora

Proyección serializable del turno. Un objeto JSON por línea. Esquema en
[contracts/ipc-protocol.md](contracts/ipc-protocol.md#bitácora).

---

### FraseReferencia

| Campo | Tipo | Reglas |
|---|---|---|
| `texto` | String | La frase tal como se diría |
| `accionEsperada` | String? | Excluyente con `rechazoEsperado` |
| `parametrosEsperados` | Map\<String,String\>? | Presente si hay `accionEsperada` |
| `rechazoEsperado` | ClaseError? | Excluyente con `accionEsperada` |
| `etiquetas` | List\<String\> | `voseo`, `enclitico`, `muletilla`, `idioma_mixto` — permiten verificar las cuotas de FR-064 |

**Invariante**: exactamente uno de `accionEsperada` y `rechazoEsperado` está presente (FR-066).

---

## Máquina de estados

Seis estados (FR-029). La matriz reproduce la de [spec.md § Máquina de estados](spec.md) y agrega la
columna de qué se registra en la bitácora, que es lo que hace auditable cada transición.

### Estados

| Estado | Significado | Salida garantizada |
|---|---|---|
| `REPOSO` | Sin turno. Sin captura de audio. | — (estado estable) |
| `ESCUCHANDO` | Captura abierta, esperando fin de habla | 10 s sin voz (FR-061) · 20 s tope duro (FR-062) |
| `PROCESANDO` | Transcribiendo, resolviendo y validando | 10 s (FR-070) |
| `ESPERANDO_CONFIRMACION` | Acción destructiva presentada, sin ejecutar | 10 s (FR-055, NFR-019) |
| `EJECUTANDO` | Acción despachada al ejecutor | Acotada por el ejecutor; su fallo lleva a `ERROR` |
| `ERROR` | Turno cerrado con fallo, informando | 3 s (FR-071) |

NFR-023 exige que ningún estado carezca de salida acotada, y la columna de la derecha es la prueba.

### Transiciones por entrada del usuario

| Estado | `activar` | `texto` | `aceptar` | `cancelar` | Vencimiento |
|---|---|---|---|---|---|
| `REPOSO` | → `ESCUCHANDO` | → `PROCESANDO` | ignora | ignora | — |
| `ESCUCHANDO` | ignora e informa | ignora e informa | ignora | → `REPOSO` | 10 s → `REPOSO`; 20 s → `PROCESANDO` (parcial) |
| `PROCESANDO` | ignora e informa | ignora e informa | ignora | → `REPOSO` | 10 s → `ERROR` |
| `ESPERANDO_CONFIRMACION` | ignora e informa | ignora e informa | → `EJECUTANDO` | → `REPOSO` | 10 s → `REPOSO` |
| `EJECUTANDO` | ignora e informa | ignora e informa | ignora | ignora e informa | — |
| `ERROR` | ignora e informa | ignora e informa | ignora | ignora | 3 s → `REPOSO` |

`ignora e informa` significa: el estado no cambia, el turno en curso no se altera, y el usuario
recibe feedback de por qué no pasó nada (FR-052, EC-07).

### Transiciones internas

| Desde | Hacia | Condición | Requisito |
|---|---|---|---|
| `ESCUCHANDO` | `PROCESANDO` | El VAD detecta fin de habla | FR-006 |
| `PROCESANDO` | `ESPERANDO_CONFIRMACION` | Acción resuelta y clasificada destructiva | FR-026, FR-028 |
| `PROCESANDO` | `EJECUTANDO` | Acción resuelta, no destructiva, parámetros válidos | FR-024 |
| `PROCESANDO` | `ERROR` | Cualquiera de las cinco clases de `ClaseError` | FR-031 |
| `EJECUTANDO` | `REPOSO` | El ejecutor reporta éxito | — |
| `EJECUTANDO` | `ERROR` | El ejecutor reporta fallo | EC-08 |

### Invariantes de la máquina

1. **La captura de audio está abierta si y solo si** el estado es `ESCUCHANDO` o
   `ESPERANDO_CONFIRMACION` (FR-002, FR-060). En los otros cuatro no existe proceso de captura.
   Verificable con `ps`, no solo por revisión de código.
2. **Nunca hay dos turnos activos.** Toda entrada de activación en un estado distinto de `REPOSO`
   se ignora (FR-052).
3. **De `ESPERANDO_CONFIRMACION` no se sale ejecutando por vencimiento**: el vencimiento va a
   `REPOSO` sin ejecutar (FR-055). Es la garantía de que el silencio nunca confirma (FR-027).
4. **`ERROR` es transitorio**, no absorbente: siempre vuelve a `REPOSO` en 3 s (FR-071).
5. **Toda transición a un estado terminal de turno** (`REPOSO` desde error/cancelación, o tras
   `EJECUTANDO`) escribe exactamente una entrada de bitácora (FR-046).

### Cobertura de prueba

SC-014 exige que las 30 celdas de la tabla de entradas produzcan la transición declarada. El test es
parametrizado sobre estado × entrada y corre por la vía de texto plano, sin audio (FR-041).
