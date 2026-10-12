---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Procesos y Señales

Bash es un **proceso** que crea otros **procesos**: su trabajo central es hacer `fork` (duplicarse) y `exec` (reemplazar la imagen) para ejecutar comandos, y después esperar o no a que terminen. Las señales son el mecanismo del kernel para notificar eventos a procesos (Ctrl+C, cierre de terminal, `kill`), y `trap` es la forma de que **bash se comporte como adulto** ante ellas: limpiar archivos temporales, guardar estado, terminar hijos. Este nota cubre el modelo de procesos, job control, el catálogo de señales y los patrones de limpieza robustos.

## El modelo de procesos

```bash
systemd (PID 1)                  ← ancestro de todo; adopta huérfanos
└── terminal / shell (PID 1000)  ← sesión, grupo de control
    └── bash script.sh (1001)    ← hijo: hereda fds, cwd, entorno
        ├── sleep 10 (1002)      ← nieto (hijo del script)
        └── grep foo (1003)
```

| Concepto | Definición práctica |
|---|---|
| **PID** | Identificador único del proceso |
| **PPID** | PID del padre (`$PPID` en bash) |
| **fork** | El kernel clona el proceso: copia de la tabla, mismo cwd/fds/entorno |
| **exec** | Reemplaza la imagen del proceso por otro binario (mismo PID) |
| **Zombie** | Terminó pero su padre no hizo `wait()` — queda como entrada `Z` en la tabla |
| **Huérfano** | Su padre murió; lo adopta init/systemd (reparent a PID 1) |
| **Sesión / PGID** | Grupos de señales: SIGHUP se envía a la sesión de control |

Variables y herramientas:

```bash
echo "$$"        # PID del shell actual
echo "$PPID"     # PID del padre
echo "$BASHPID"  # PID REAL del proceso bash (≠ $$ dentro de subshells — hereda el valor)
echo "$!"        # PID del último background job

ps -p $$ -o pid,ppid,stat,comm       # este proceso
ps -ef | grep [m]i_script            # [] evita matchear al propio grep
pgrep -f "mi_script"                 # PIDs por patrón de cmdline
pstree -p $$                         # árbol
ls -l /proc/$$/fd                    # descriptors abiertos (Linux)
cat /proc/$$/status                  # estado detallado (Linux)
```

### Estados de proceso (`STAT` en `ps`)

| Estado | Significado |
|---|---|
| `R` | Running / en la cola de run |
| `S` | Sleeping (esperando evento, la mayoría del tiempo) |
| `D` | Disk sleep (ininterrumpible, bloqueo de E/S) |
| `T` | Stopped (SIGSTOP/SIGTSTP) |
| `t` | Tracing stop (debugger) |
| `Z` | **Zombie** (muerto, sin reap) |
| `<` | Prioridad alta (nice negativo) |
| `N` | Prioridad baja (nice positivo) |

## Background, jobs y `wait`

```bash
comando_largo &
pid=$!                              # capturar el PID INMEDIATAMENTE
echo "lanzado: $pid"

jobs                                # tabla de jobs del shell
jobs -l                             # con PIDs
jobs -p                             # solo PIDs
wait "$pid"                         # esperar ese job: rc = rc del hijo
wait                                # esperar TODOS
wait -n                             # esperar al PRIMERO que termine (bash 4.3+); rc = su rc
wait -n "$pid1" "$pid2"             # cualquiera de ellos (bash 4.4+)
wait -t 5 "$pid"                    # con timeout (bash 5.1+); rc 127 si no existe
```

Verificado: `wait -n` sin hijos → rc `127`; con hijos → rc del primero en terminar. El patrón de **throttling** de paralelismo (lanzar N y esperar a que baje la carga) usa `wait -n` en loop — ver el patrón worker pool en [[Bash Scripting - Patrones Comunes y Plantillas]].

### Foreground / background / suspensión

```bash
comando &           # background desde el arranque
# En terminal interactiva:
#   Ctrl+Z   → suspende el foreground actual (SIGTSTP)
bg %2       # reanudar job #2 en background
fg %1       # traer job #1 a foreground
kill %1     # matar por número de job (también: kill -- -%1 grupo)
```

El **job control** (`jobs`/`fg`/`bg`) necesita *monitor mode*: está activo en shells interactivos, pero **desactivado en scripts** — si lo necesitás adentro de un script, `set -m` al inicio.

### `disown` y `nohup`: sobrevivir al cierre de la terminal

Cuando la terminal muere, el kernel envía **SIGHUP a la sesión de control**. Dos defensas:

```bash
comando_largo &
disown              # quita el job de la tabla: el shell ya no lo notifica/manda SIGHUP
disown -a           # todos los jobs
disown -h %1        # "hardening": si llega SIGHUP, el hijo lo recibe pero IGNORA
                    # (no lo quita de la tabla; el comportamiento del hijo debe ignorar HUP)

# nohup (externo): arranca el comando con SIGHUP ignorado y stdout a nohup.out
nohup ./servidor.sh > servidor.log 2>&1 &
```

Diferencia clave: `disown` es del **shell** (sirve para los hijos de ESTE shell); `nohup` modifica el **comando** (funciona aunque otro shell lo lance, p.ej. vía SSH). Modernamente `nohup` + redirección + `&` sigue siendo el estándar para "dejar esto corriendo".

## Señales: el catálogo

Una señal es un **evento asíncrono** que el kernel entrega al proceso. El proceso puede: **ignorarla** (`trap ''`), **instalar handler** (`trap '...'`) o sufrir el **default** (casi siempre: terminar).

| Señal | Nº (Linux x86) | Default | Origen típico | Uso en scripts |
|---|---|---|---|---|
| `SIGHUP` | 1 | Terminar | Terminal cerrada / `kill -HUP` | Recargar config en daemons; "no mueras al cerrar sesión" |
| `SIGINT` | 2 | Terminar | Ctrl+C | Limpieza ante interrupción |
| `SIGQUIT` | 3 | Terminar + core | Ctrl+\ | Dump para debugging |
| `SIGKILL` | 9 | Terminar **forzoso** | `kill -9` | **Inmune a trap**: no se puede bloquear ni interceptar |
| `SIGTERM` | 15 | Terminar | `kill` por defecto, systemd al parar servicios | **Señal de apagado educado**: guardá estado acá |
| `SIGSTOP` | 19 | Pausar | `kill -STOP` | Pausa inmune a trap |
| `SIGCONT` | 18 | Reanudar | `kill -CONT` | Continuar tras STOP |
| `SIGTSTP` | 20 | Pausar | Ctrl+Z | Suspendible |
| `SIGUSR1` | 10 | Terminar | `kill -USR1` | Reabrir logs (logrotate), recargar, dump |
| `SIGUSR2` | 12 | Terminar | `kill -USR2` | Segundo canal de señalización |
| `SIGCHLD` | 17 | Ignorar | Hijo terminó | Detectar hijos sin `wait` (evitar zombies) |

> [!warning] Numeración no universal
> Los números de arriba son de **Linux x86** (`kill -l` los confirma: `TERM=15`, `USR1=10`, `USR2=12`, `STOP=19`, `CONT=18`). En **macOS/BSD** `SIGUSR1=30` y `SIGUSR2=31`. Usá **nombres** (`kill -TERM`), no números, en todo código portable.

### Enviar señales

```bash
kill -TERM "$pid"       # forma por nombre (portable, legible)
kill -15 "$pid"         # por número (frágil entre SO)
kill -s USR1 "$pid"     # sintaxis explícita
kill -0 "$pid"          # "ping": existe y podemos señalarlo (rc=0 si vivo)
kill -TERM -"$pgid"     # NEGATIVO = grupo de procesos completo
pkill -TERM -f "patrón" # por cmdline completa
pkill -u "$USER" -TERM  # por usuario (¡cuidado!)
kill -KILL "$pid"       # último recurso — ver etiquette abajo
```

### Señal vs exit code: la distinción fundamental

- **Exit code** (0-255): el proceso terminó **voluntariamente** con `exit N` o con el rc del último comando. Es comunicación.
- **Señal**: el proceso fue **interrumpido por fuera**. El shell lo traduce a exit code **128+N**: matado por SIGTERM (15) → `wait` devuelve `143`; por SIGKILL (9) → `137`; Ctrl+C → `130`.

Verificado en vivo:

```bash
sleep 30 & pid=$!; kill -TERM "$pid"; wait "$pid" 2>/dev/null
# → rc = 143   (= 128 + 15)

sleep 30 & pid=$!; kill -KILL "$pid"; wait "$pid" 2>/dev/null
# → rc = 137   (= 128 + 9)
```

Por eso los scripts que ejecutan subprocesos interpretan: `rc==0` ok · `rc==130` cancelado por usuario (no es fallo) · `rc==137` matado · `rc==124` timeout (de `timeout`) — patrón de diagnóstico completo en [[Bash Scripting - Patrones Comunes y Plantillas]].

## `trap`: handlers de señales

```bash
trap 'comandos' SEÑALES...     # sintaxis general
```

### Limpieza garantizada con `EXIT`

`EXIT` es una "pseudo-señal" que dispara **siempre** al terminar el script (por `exit`, por errexit, por fin de archivo):

```bash
cleanup() {
    local rc=$?                       # ← PRIMERA línea: capturar el rc original
    [[ -n "${tmp_dir:-}" && -d "$tmp_dir" ]] && rm -rf "$tmp_dir"
    [[ -n "${lock_fd:-}" ]] && exec {lock_fd}>&-
    log "cleanup (rc=$rc)"
    exit "$rc"                        # preservar el código de salida
}
trap cleanup EXIT
```

> [!info] El patrón `rc=$?` al inicio de EXIT
> Sin esto, el `if`/`[[ ]]` del cleanup **sobreescribe** `$?` y el script sale con el rc del último comando interno (típicamente 0) — los orquestadores (make, CI, cron) verán éxito cuando hubo error. Verificado: con `trap 'rc=$?; echo rc=$rc' EXIT; exit 7` → el trap ve `7`.

### Manejadores de señales

```bash
on_sigint() { printf '\n[!] interrumpido\n' >&2; exit 130; }     # 128+2 por convención
on_sigterm() { log "TERM: cerrando ordenadamente"; guardar_estado; exit 143; }
on_usr1()   { log "USR1: reabrir logs"; exec 1>>"$LOG"; exec 2>>"$LOG"; }   # patrón logrotate

trap on_sigint INT
trap on_sigterm TERM
trap on_usr1 USR1

trap '' HUP              # IGNORAR señal (inmune; heredable por hijos en ciertos casos)
trap - TERM INT          # resetear al default
trap -p                 # volcar todos los traps activos (debugging)
```

Verificado: el trap puede decidir su propio exit code (`trap 'exit 7' EXIT` → rc 7 preservado).

### La trampa del sleep

```bash
# MAL: con sleep en foreground, el trap NO corre hasta que sleep termine
trap 'echo "limpiando"' TERM
sleep 100          # el shell espera: la señal queda pendiente

# BIEN: sleep en background + wait interrumpible
trap 'echo "limpiando"; exit 143' TERM
sleep 100 &
wait $!            # wait ES interrumpible por señales: el trap corre al instante
```

`wait` es la única builtin que cede al señalamiento inmediatamente — por eso el patrón `sleep X & wait $!` aparece en todos los bucles de servicio.

### El handler que no debe hacer mucho

Los handlers corren en contexto delicado (podrían interrumpir a bash en medio de una expansión). Reglas: en el handler, **solo** cerrar archivos, borrar temporales, escribir un mensaje y `exit`. Todo lo demás (`rm -rf` de árboles grandes, llamadas de red) corre afuera del handler o se encola.

## Etiqueta de `kill`: TERM primero, KILL después

`SIGKILL` no puede ser interceptado: mata al proceso **sin** ejecutar traps ni flush de buffers. Un servicio muerto con `-9` puede dejar lock files, datos a medias o conexiones sucias. El protocolo correcto:

```bash
terminar_ordenado() {
    local pid=$1
    kill -TERM "$pid" 2>/dev/null || return 0     # 1. pedir amablemente
    local i
    for i in {1..50}; do                          # 2. esperar hasta 5s
        kill -0 "$pid" 2>/dev/null || return 0    #    (ya murió: ok)
        sleep 0.1
    done
    warn "PID $pid no respondió a TERM; escalando a KILL"
    kill -KILL "$pid" 2>/dev/null || true         # 3. forzar
}
```

Es exactamente lo que hace `systemd` (TimeoutStopSec) y lo que deberían hacer tus scripts antes de "reiniciar" un servicio — patrón PID-file en [[Bash Scripting - Patrones Comunes y Plantillas]].

## Daemons y patrones de servicio

```bash
# El truco "daemonizar" mínimo en bash:
(
    cd /
    exec </dev/null >>/var/log/servicio.log 2>&1     # desvincular de la terminal
    setsid ./servicio.sh &                            # nueva sesión: sobrevive a SIGHUP
)

# O con nohup explícito:
nohup setsid ./servicio.sh </dev/null >>/var/log/servicio.log 2>&1 &
```

`setsid` crea **nueva sesión** (el proceso deja de ser líder de la sesión de control del terminal — no recibe SIGHUP). Los gestores modernos (systemd) hacen todo esto y más; para servicios de producción usá una unit file, no un daemon hecho a mano — ver [[Cron Jobs y Programación de Tareas]] para la contraparte programada.

### Zombies y `wait` selectivo

```bash
# Un script que lanza hijos en background y NO espera deja zombies al terminar.
# Soluciones:
wait                       # al final: drena todos los hijos
wait -n                    # o ir drenando conforme terminan
# Para hijos largos que NO querés esperar: disown o doble-fork con setsid.
```

## Observabilidad: `/proc`, `ps`, `strace`

```bash
cat /proc/$pid/status      # Estado, PPID, threads, memoria (Linux)
ls -l /proc/$pid/fd/       # qué archivos/pipes tiene abiertos
cat /proc/$pid/cmdline | tr '\0' ' '    # cmdline real (con args)
cat /proc/$pid/environ | tr '\0' '\n'   # entorno (¡solo el dueño o root!)

ps aux --sort=-%mem | head       # top por memoria
ps -u "$USER" -o pid,etime,cmd   # procesos propios con tiempo transcurrido
top -p "$pid"                    # monitoreo en vivo

strace -p "$pid" -f -e trace=network,process   # syscalls en vivo (requiere permisos)
lsof -p "$pid"                    # archivos abiertos (alternativa a /proc/pid/fd)
```

Detalle de comandos de sistema en [[Comandos Básicos de Sistema y Red en Linux]].

## Errores comunes

1. **`kill %1` / `kill $!` sin verificar** — el job ya terminó: rc 127 o señal a un PID **reciclado** (otro proceso). Guardá el PID y chequeá con `kill -0` antes.
2. **Usar `-9` como primera opción** — sin trap ni flush. Siempre TERM → esperar → KILL.
3. **Leer `$!` después de otro comando** — `$!` es solo del último background. `cmd & pid=$!` inmediato.
4. **Trap que no preserva `$?`** — el script sale con rc del cleanup, no del error. `local rc=$?` primera línea.
5. **`sleep 300` como "espera interrumpible"** — el trap espera a que sleep termine. `sleep 300 & wait $!`.
6. **Zombies acumulados** — hijos en background sin `wait` al final. `wait` de cierre.
7. **Números de señal hardcodeados** — `USR2` es 12 en Linux y 31 en macOS. Nombres siempre.
8. **Esperar job control en scripts** — `jobs`/`fg`/`bg` necesitan `set -m` fuera de interactivos.
9. **Olvidar que KILL tampoco salva a los hijos** — matar al padre no mata a los hijos (se re-parentean); para el árbol completo: `kill -- -"$pgid"` o `pkill -P`.

## Cheatsheet

| Acción | Forma |
|---|---|
| Background | `cmd &` · capturar `pid=$!` |
| Esperar hijo | `wait "$pid"` · primero que termine: `wait -n` |
| Tabla de jobs | `jobs -l` · foreground `fg %1` · background `bg %1` |
| Existe y vivo | `kill -0 "$pid"` |
| Matar educado | `kill -TERM "$pid"` |
| Forzar | `kill -KILL "$pid"` (último recurso) |
| Ignorar señal | `trap '' HUP` |
| Reset señal | `trap - TERM` |
| Limpieza final | `trap cleanup EXIT` + `rc=$?` primera línea |
| Handler | `trap 'on_term' TERM` (salir con 128+N) |
| Espera interrumpible | `sleep X & wait $!` |
| Desvincular | `disown` · `nohup cmd &` · `setsid` |
| Grupo completo | `kill -TERM -"$pgid"` |
| Matar por patrón | `pkill -f "patrón"` |
| Traducir rc de señal | `rc - 128 = número de señal` (143=TERM, 137=KILL, 130=INT) |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — `$$`, `$!`, y la traducción 128+N
- [[Bash Scripting - Control de Flujo]] — bucles de servicio interrumpibles
- [[Bash Scripting - Entrada-Salida y Redirección]] — herencia de fds y FIFOs como IPC
- [[Bash Scripting - Patrones Comunes y Plantillas]] — worker pool, PID files, retry
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — trap ERR/DEBUG y strict mode
- [[Comandos Básicos de Sistema y Red en Linux]] — ps, top, lsof, ss en contexto
- [[Cron Jobs y Programación de Tareas]] — procesos sin terminal adjunta
