# Contrato: protocolo entre procesos

**Feature**: `001-asistente-voz-eva` | **Date**: 2026-08-09 | **Decisión**: [research.md D-02](../research.md)

Cuatro procesos (plan.md § Estructura de procesos). Transporte: **sockets de dominio Unix** bajo
`$XDG_RUNTIME_DIR/eva/`, permisos `0600`. Codificación: **JSON Lines** — un objeto JSON por línea
terminada en `\n`, UTF-8, sin saltos de línea dentro del objeto.

Ningún socket es de red (NFR-011). Verificable con `ss -tulpn` sin salida atribuible a Eva (SC-007).

---

## 1. `control.sock` — `evactl` → daemon

Socket de petición y respuesta. El cliente abre, escribe una línea, lee una línea, cierra.

### Peticiones

| `cmd` | Campos | Efecto | Requisito |
|---|---|---|---|
| `activate` | — | Inicia un turno de voz | FR-001 |
| `text` | `texto` (string) | Inicia un turno de texto | FR-040 |
| `confirm` | — | Acepta la confirmación pendiente | FR-057 |
| `cancel` | — | Cancela | FR-003, FR-057 |
| `status` | — | Devuelve el estado actual, sin efectos | FR-029 |
| `reload` | — | Recarga la configuración | FR-044 |

```json
{"cmd":"text","texto":"abrí la terminal en el escritorio 3"}
{"cmd":"activate"}
{"cmd":"status"}
```

### Respuestas

```json
{"ok":true,"turno":"7f3a…","estado":"escuchando"}
{"ok":false,"clase":"accion_desconocida","mensaje":"no hay ninguna acción para 'hacé un café'"}
{"ok":true,"estado":"reposo","catalogo":{"acciones":12,"cargado":"2026-08-09T14:02:11Z"}}
```

`clase` es siempre uno de los cinco valores de FR-031: `no_entendido`, `accion_desconocida`,
`intencion_ambigua`, `parametro_invalido`, `fallo_al_ejecutar`.

Ante `intencion_ambigua` la respuesta incluye las candidatas, porque FR-056 exige nombrarlas:

```json
{"ok":false,"clase":"intencion_ambigua","candidatas":["abrir_app","abrir_sitio"],
 "mensaje":"'abrí spotify' puede ser dos acciones distintas"}
```

**Códigos de retorno de `evactl`**: `0` éxito · `1` rechazo con clase · `2` daemon inalcanzable ·
`3` error de uso. Los códigos son parte del canal no gráfico de FR-033.

---

## 2. `overlay.sock` — daemon ↔ superficie de estado

Socket bidireccional persistente. El daemon es el servidor; la superficie se conecta al arrancar y
se reconecta con backoff si se cae.

### Daemon → superficie

```json
{"tipo":"estado","estado":"escuchando"}
{"tipo":"estado","estado":"procesando","texto":"abrí la terminal en el escritorio 3"}
{"tipo":"estado","estado":"esperando_confirmacion","texto":"cerrá la ventana",
 "accion":{"id":"cerrar_ventana","parametros":{}},"expira_ms":10000}
{"tipo":"estado","estado":"error","clase":"accion_desconocida",
 "mensaje":"no hay ninguna acción para 'hacé un café'"}
{"tipo":"ocultar"}
```

`estado` es uno de los seis de FR-029. La superficie **solo pinta**: no decide nada, no guarda
estado propio más allá del último mensaje, y no interpreta.

El mensaje `ocultar` llega al terminar el turno; la superficie debe dejar de existir en el árbol de
ventanas dentro de los 3 s de NFR-022.

### Superficie → daemon

```json
{"tipo":"listo","version":"1"}
```

**La superficie no envía `confirmar` ni `cancelar`.** Este es el punto del diseño que más
fácilmente se hace mal, así que conviene que el contrato lo diga de forma explícita: la superficie
se declara con `keyboard-interactivity: none` y por lo tanto **no recibe eventos de teclado**. Las
teclas de confirmación son `bind` de Hyprland que invocan `evactl confirm` y `evactl cancel`, que
entran por `control.sock`. El teclado nunca cambia de dueño, y FR-036 y FR-038 se cumplen por
construcción y no por cuidado en el manejo del foco.

---

## 3. Daemon ↔ `pw-record` — audio

Tubería, no socket. El daemon lanza el hijo y lee su stdout.

```
pw-record --rate 16000 --channels 1 --format s16 --target <default> -
```

**Formato**: PCM crudo, s16le, 16 kHz, mono. Sin encabezado. Sin JSON: codificar audio sería gasto
de CPU en la máquina más ajustada del proyecto.

**Ciclo de vida**: se lanza al entrar en `ESCUCHANDO`, se termina con `SIGTERM` al salir. Fuera del
turno el proceso no existe, que es como FR-002 pasa de promesa a hecho verificable con `ps`.

**Errores**: `stderr` va a la bitácora. Un EOF prematuro o un código de salida distinto de cero se
traduce a `fallo_al_ejecutar` con la etapa `captura` (EC-01, EC-12).

---

## 4. Daemon ↔ `eva-stt` — transcripción

**El motor concreto está sin decidir** (research.md D-05). El contrato se define ahora para que la
elección no toque el daemon: cualquiera de los tres candidatos se adapta a esta interfaz.

**Daemon → motor**: PCM crudo por stdin, mismo formato que arriba.

**Motor → daemon**: JSON Lines por stdout.

```json
{"tipo":"parcial","texto":"abrí la termi"}
{"tipo":"final","texto":"abrí la terminal en el escritorio 3","ms":142}
```

Los mensajes `parcial` son **opcionales**: solo los emite un motor con decodificación incremental.
Un motor por lotes emite únicamente `final`. El daemon no depende de los parciales para nada
funcional —solo los reenvía a la superficie como feedback— así que ambos tipos de motor funcionan
contra el mismo adaptador. Esa es la razón de que el contrato los declare opcionales en vez de
exigirlos.

`ms` es el tiempo de procesamiento reportado por el motor, para la bitácora (NFR-009).

**Ciclo de vida**: se lanza al **inicio del turno**, no al fin del habla, de modo que la carga del
modelo transcurra mientras el usuario habla y no consuma presupuesto de NFR-006. Se descarga tras
120 s sin turnos.

---

## 5. Daemon → Hyprland

Cliente del socket de Hyprland; Eva no define este protocolo, lo consume.

**Ruta**: `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket.sock`

| Uso | Comando | Requisito |
|---|---|---|
| Listar ventanas | `j/clients` | FR-014, FR-015 |
| Espacio actual | `j/activeworkspace` | AMB-05 |
| Rango de espacios | `j/monitors` | EC-05 |
| Cambiar de espacio | `dispatch workspace <n>` | FR-019 |
| Mover ventana | `dispatch movetoworkspace <n>` | FR-020 |
| Cerrar ventana | `dispatch killactive` | FR-021 |
| Pantalla completa | `dispatch fullscreen 0` | FR-068 |
| Flotante | `dispatch togglefloating` | FR-069 |

**Correspondencia aplicación ↔ ventana**: el campo `class` de `j/clients` se compara con
`StartupWMClass` de la entrada `.desktop` de la aplicación. Es el vínculo que hace implementable la
política de instancia única de FR-014 y el movimiento de ventana de FR-015. Si una entrada
`.desktop` no declara `StartupWMClass`, se usa el nombre del ejecutable de `Exec` como respaldo, y
si tampoco resuelve, la acción se rechaza con `fallo_al_ejecutar` en vez de abrir una segunda
instancia — un rechazo es preferible a violar FR-014.

**Errores**: el socket responde `ok` o una cadena de error, que se traduce a `fallo_al_ejecutar`
con etapa `ejecucion` (EC-08).

---

## Bitácora

JSONL en `$XDG_STATE_HOME/eva/bitacora/AAAA-MM-DD.jsonl`. **Un objeto por turno** (FR-046), porque
el turno es la unidad de análisis. Local siempre (FR-048).

```json
{
  "turno": "7f3a2b1c-…",
  "inicio": "2026-08-09T14:02:11.412Z",
  "origen": "voz",
  "texto_entrada": "abrí la terminal en el escritorio 3",
  "accion": "abrir_app_en_espacio",
  "parametros": {"app": "kitty", "espacio": 3},
  "estado_final": "ejecutado",
  "clase_error": null,
  "etapa_fallo": null,
  "duraciones_ms": {
    "captura": 2140, "fin_habla": 240, "transcripcion": 151,
    "resolucion": 4, "validacion": 1, "ejecucion": 22, "feedback": 11
  },
  "total_desde_fin_habla_ms": 429,
  "excedio_presupuesto": false
}
```

Turno rechazado:

```json
{
  "turno": "9c1e…", "inicio": "2026-08-09T14:05:03.007Z", "origen": "texto",
  "texto_entrada": "abrí spotify",
  "accion": null, "parametros": null,
  "estado_final": "rechazado",
  "clase_error": "intencion_ambigua",
  "etapa_fallo": "resolucion",
  "candidatas": ["abrir_app", "abrir_sitio"],
  "duraciones_ms": {"resolucion": 6},
  "total_desde_fin_habla_ms": null,
  "excedio_presupuesto": false
}
```

**Invariante** (FR-047, SC-010): si `estado_final ≠ "ejecutado"`, entonces `clase_error` y
`etapa_fallo` son no nulos. Es lo que permite determinar la causa sin reproducir el turno.

**Etapas** — conjunto cerrado, coincide con las filas del presupuesto de latencia de plan.md:
`captura`, `fin_habla`, `transcripcion`, `resolucion`, `validacion`, `confirmacion`, `ejecucion`,
`feedback`.
