# Quickstart — validación de punta a punta

**Feature**: `001-asistente-voz-eva` | **Date**: 2026-08-09 | **Plan**: [plan.md](plan.md)

Escenarios de validación con comandos concretos y resultado esperado. **Todos se corren en la
laptop objetivo**, bajo la *carga de referencia* (navegador con 10 pestañas y editor con un
proyecto cargado, sin compilación en curso), salvo los marcados como independientes de carga.

Los escenarios están ordenados por dependencia. **Q0 y Q1 hay que correrlos antes de
`/speckit-tasks`**: son los que cierran las dos compuertas amarillas del Constitution Check.

---

## Q0 · Confirmar el hardware y el techo de CPU

El conteo de hilos quedó cerrado en la clarificación del 2026-08-09: **4 núcleos físicos, 8 hilos
lógicos, SMT activo**. Este escenario solo confirma el dato y verifica que el techo de CPU esté
efectivamente aplicado.

```bash
lscpu | grep -E 'Model name|^CPU\(s\)|Thread|Core'
free -m
systemctl --user show eva.service -p CPUQuotaPerSecUSec
```

**Esperado**: `AMD Ryzen 5 3450U`, `CPU(s): 8`, `Thread(s) per core: 2`, `Core(s) per socket: 4`.

**Esperado de `free`**: ~8000 MB totales; con navegador y editor abiertos, `available` alrededor de
3000 MB.

**Esperado del techo**: `CPUQuotaPerSecUSec=2s`, que es `CPUQuota=200%` — dos hilos lógicos, es
decir un núcleo físico (NFR-004). Si aparece `infinity`, el techo no está aplicado y NFR-004 no se
está cumpliendo aunque el consumo medido parezca bajo.

---

## Q1 · Medir los candidatos de transcripción — **bloqueante**

Cierra D-05, la única decisión abierta del plan. El protocolo completo está en
[research.md § D-05](research.md). Resumen operativo:

```bash
# 1. Grabar el corpus, una sola vez, reutilizado por los tres candidatos
mkdir -p ~/eva-medicion/corpus
pw-record --rate 16000 --channels 1 --format s16 ~/eva-medicion/corpus/01.wav
```

Grabar 20 enunciados de 2 a 5 segundos tomados del conjunto de frases de referencia, al menos 5 con
un nombre de aplicación o sitio en inglés (FR-064).

```bash
# 2. Por cada candidato: 3 corridas, se descarta la primera
#    Medir RSS con smaps_rollup y CPU con pidstat, en paralelo a la transcripción
/usr/bin/time -v <motor> ... 2>&1 | grep 'Maximum resident'
grep Pss /proc/<pid>/smaps_rollup
pidstat -p <pid> 1
```

**Resultado esperado**: la tabla de research.md D-05 completa, con `t_transcripcion_ms` medido
**desde el fin acústico del habla**, no desde el inicio del archivo. Aplicar después la regla de
decisión de D-05, que elige sin criterio humano.

**Si ningún candidato alcanza el escalón 2**, no seguir a `/speckit-tasks`: aplicar la escalera de
contingencia de D-05, en la que la calidad de transcripción se degrada **antes** que el umbral de
latencia.

---

## Q2 · Presupuesto de memoria en reposo — **cierra el gate del Principio VI**

Independiente de la carga de referencia: mide solo a Eva.

```bash
systemctl --user start eva.service
sleep 600   # 10 minutos sin activaciones (NFR-001, NFR-002)

for p in $(systemctl --user show -p MainPID --value eva.service) \
         $(pgrep -f eva-overlay); do
  echo -n "$p: "; grep '^Pss:' /proc/$p/smaps_rollup
done
```

**Esperado**: suma de `Pss` **< 150 MB** (NFR-002). El desglose previsto es `eva-daemon` 60–80 MB y
`eva-overlay` 8–15 MB. **No debe haber ningún proceso de `pw-record` ni de STT**: en reposo esos dos
no existen, que es como FR-002 se verifica con `ps` en lugar de por revisión de código.

```bash
pgrep -f 'pw-record|eva-stt' && echo "FALLA: hay captura o STT en reposo" || echo "OK"
```

**CPU en reposo** (NFR-001):

```bash
pidstat -p $(systemctl --user show -p MainPID --value eva.service) 60 10
```

Esperado: `%CPU` promedio **< 1%** de un núcleo.

---

## Q3 · Turno completo por texto, sin audio — **el escenario más importante**

Prueba US1 entera y demuestra el Principio II. **Se corre con el micrófono físicamente
desconectado** para que la prueba sea honesta.

```bash
evactl text "abrí la terminal"
```

**Esperado**: `{"ok":true,...}`, código de retorno `0`, y una ventana de terminal nueva enfocada.

```bash
evactl text "andá al escritorio 3"          # → cambia de espacio (US1-2)
evactl text "buscá recetas de milanesas"    # → búsqueda web (US1-3)
evactl text "hacé un café"                  # → rechazo accion_desconocida (US1-4)
evactl text "abrí"                          # → rechazo parametro_invalido (US1-5)
evactl text "abrí Reddit"                   # → rechazo sitio_no_declarado (US1-10, EC-13)
```

**Instancia única** (US1-8, FR-014) — con la terminal ya abierta en el espacio 1:

```bash
evactl text "abrí la terminal"
hyprctl -j clients | jq '[.[] | select(.class=="kitty")] | length'
```

Esperado: `1`. Nunca `2`. La ventana existente queda enfocada.

**Movimiento entre espacios** (US1-9, FR-015, EC-14) — con la terminal abierta en el espacio 1:

```bash
evactl text "abrí la terminal en el escritorio 3"
hyprctl -j clients | jq '[.[] | select(.class=="kitty") | .workspace.id]'
```

Esperado: `[3]`. Un solo elemento, y en el espacio 3.

**Ambigüedad** (US1-7, FR-056) — declarando "spotify" como aplicación **y** como sitio:

```bash
evactl text "abrí spotify"
```

Esperado: `ok:false`, `clase: intencion_ambigua`, y el campo `candidatas` con **las dos** acciones
nombradas. **No debe ejecutar ninguna de las dos.** Es el escenario que prueba que el matcher
recolecta todos los candidatos en vez de cortar en el primero.

---

## Q4 · Conjunto de frases de referencia — cierra NFR-014 a NFR-016

```bash
./gradlew :eva-aplicacion:test --tests '*ConjuntoReferenciaTest'
```

**Esperado**, sobre las 300 entradas de `tests/referencia/frases.jsonl`:

| Métrica | Umbral | Requisito |
|---|---|---|
| Acción y parámetros correctos | ≥ 90% | NFR-014, SC-002 |
| Acción distinta de la pedida | < 1% | NFR-015, SC-003 |
| Entradas de rechazo, rechazadas con su clase | 100% | NFR-016, SC-004 |

El test también verifica la **estructura** del conjunto —300 entradas, 20 por acción, 80 de
rechazo, 10 por clase (FR-063 a FR-065)— y falla si se degrada. Corre por la vía de texto plano,
sin audio (FR-067, SC-012).

---

## Q5 · Turno por voz con audio pregrabado

Prueba el pipeline completo sin depender de que alguien hable en el momento.

```bash
EVA_PCM_SOURCE=file:///home/tiago/eva-medicion/corpus/01.wav evactl activate
```

**Esperado**: mismo resultado que el `evactl text` equivalente (US2-5, FR-040). El adaptador de
archivo entra por el puerto `PcmSource`, así que el orquestador recorre el mismo código.

**Topes del estado escuchando** (FR-061, FR-062, NFR-020):

```bash
evactl activate            # y no decir nada
```

Esperado: a los **10 s** vuelve a reposo, sin transcribir ni ejecutar, con la señal sonora de turno
vacío (EC-02, US2-2).

```bash
# con audio continuo, por ejemplo música de fondo cerca del micrófono
evactl activate
```

Esperado: la captura se cierra por tope duro a los **20 s** (EC-16, US2-6) y lo capturado se procesa
como transcripción parcial.

---

## Q6 · Confirmación de acción destructiva

```bash
evactl text "cerrá la ventana"
```

**Esperado**: `ok:true`, `estado: esperando_confirmacion`. **La ventana sigue abierta.** La
superficie muestra la acción concreta con sus parámetros (US3-1, FR-026).

```bash
evactl confirm     # → la ventana se cierra (US3-2)
evactl cancel      # → no pasa nada, vuelve a reposo (US3-3)
# no hacer nada    → a los 10 s vuelve a reposo sin cerrar (US3-4, FR-055)
```

**Colisión de palabra de confirmación** (US3-7, FR-058) — declarar en `config.toml` una
`palabra_aceptar` que también active una acción:

```bash
evactl reload
```

Esperado: `ok:false`, con la colisión informada. La configuración se rechaza.

---

## Q7 · Superficie de estado sin robo de foco

Prueba las cinco condiciones del Principio X. **La parte crítica es la 1**, porque es la que suele
fallar y la que rompe la usabilidad entera.

```bash
# Con el foco en un editor, anotar la ventana enfocada
hyprctl -j activewindow | jq -r '.address'
evactl activate
hyprctl -j activewindow | jq -r '.address'
```

**Esperado**: la misma dirección antes y después (FR-036, US4-1). Escribir en el editor durante la
activación: los caracteres llegan al editor.

```bash
# Al terminar el turno, esperar y comprobar que la superficie se fue
sleep 4
hyprctl -j clients | jq '[.[] | select(.class|test("eva"))] | length'
```

Esperado: `0` a los 3 s de terminado el turno (NFR-022, US4-3, US4-5).

**Con la superficie apagada** (FR-039, SC-013) — poner `superficie.habilitada = false`:

```bash
evactl reload
evactl text "abrí la terminal"
```

Esperado: la acción se ejecuta igual y el resultado llega por notificación y por código de retorno
(US4-6, FR-033).

---

## Q8 · Robustez — US7

```bash
# Micrófono ocupado por otra aplicación
pw-record --target <mic> /dev/null &
evactl activate
```

Esperado: error de dispositivo ocupado, **el PID del daemon no cambia**, y el siguiente
`evactl text` funciona (US7-1, EC-01).

```bash
# La superficie muere
pkill -f eva-overlay
evactl text "abrí la terminal"
```

Esperado: la acción se ejecuta igual, degradando a notificación (Principio V, FR-039). El daemon
reintenta levantar la superficie con backoff.

```bash
# Activación concurrente
evactl activate & sleep 0.2; evactl activate
```

Esperado: la segunda se ignora e informa; el turno en curso no se altera (US7-4, EC-07, FR-052).

---

## Q9 · Operación sin red — SC-007

```bash
sudo ip link set <iface> down
evactl text "abrí la terminal"
evactl text "andá al escritorio 2"
ss -tulpn | grep -iE 'eva|java' && echo "FALLA: hay socket de red" || echo "OK: sin sockets de red"
```

**Esperado**: las acciones se ejecutan, y `ss` no muestra ningún socket de red atribuible a Eva
(NFR-011, NFR-012, SC-007). Una búsqueda web con la red caída fallará en el navegador, **no en
Eva**: esa es la distinción que NFR-012 hace explícita, y el turno de Eva debe llegar hasta la
entrega al navegador sin errores propios.

---

## Q10 · Presupuesto de latencia — cierra el gate del Principio VII

```bash
./gradlew :eva-aplicacion:test --tests '*PresupuestoLatenciaTest'
```

Corre el conjunto de referencia por la vía de audio pregrabado y calcula los percentiles desde los
timestamps por etapa de la bitácora.

**Esperado**, sobre turnos sin confirmación:

| Métrica | Umbral | Requisito |
|---|---|---|
| `total_desde_fin_habla_ms` p50 | < 2000 | NFR-006 |
| `total_desde_fin_habla_ms` p95 | < 3500 | NFR-007 |
| Suma de presupuestos por etapa declarados | ≤ 2000 / 3500 | NFR-009 |

Inspección manual de una entrada:

```bash
tail -1 ~/.local/state/eva/bitacora/$(date +%F).jsonl | jq '.duraciones_ms, .total_desde_fin_habla_ms'
```

---

## Q11 · Máquina de estados completa — SC-014

```bash
./gradlew :eva-aplicacion:test --tests '*MaquinaEstadosTest'
```

**Esperado**: las 30 combinaciones de estado × entrada de
[data-model.md § Máquina de estados](data-model.md) producen la transición declarada. Corre por la
vía de texto, sin audio.

---

## Q12 · Gate de arquitectura — Principio IX

```bash
./gradlew :eva-dominio:test --tests '*ArquitecturaTest'
```

**Esperado**: `eva-dominio` no importa `eva.infraestructura`, ni `java.net`, ni `java.nio.file`, ni
`javax.sound`. El build falla si alguna regla se viola: el Principio IX se verifica, no se confía.
