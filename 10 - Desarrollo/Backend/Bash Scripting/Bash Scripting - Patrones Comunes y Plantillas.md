---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Patrones Comunes y Plantillas

Colección de los patrones que separan un script de fin de semana de un script que corre en producción: esqueleto completo, temporales seguros, logs, idempotencia, paralelismo con `xargs`, reintentos, locks y las restricciones de correr bajo cron. Cada patrón es copiable y resuelto — los fundamentos de cada pieza están en las notas hermanas.

## Esqueleto profesional (la plantilla maestra)

```bash
#!/usr/bin/env bash
# mi-script — una línea describiendo qué hace
# Uso: mi-script [OPCIONES] <entrada>
set -euo pipefail
IFS=$'\n\t'

readonly SCRIPT_NAME="$(basename "$0")"
readonly VERSION="1.0.0"
LOG_FILE="${LOG_FILE:-/var/log/${SCRIPT_NAME%.sh}.log}"

# --- Utilidades ---
die()  { printf '[ERROR] %s\n' "$*" >&2; exit 1; }
log()  { printf '[%s] %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG_FILE" >&2; }

usage() {
    cat <<EOF
Uso: $SCRIPT_NAME [OPCIONES] <entrada>

Descripción. Códigos: 0 ok · 1 error · 2 uso inválido.

Opciones:
  -h, --help      Ayuda
  -V, --version   Versión
  -n, --dry-run   Simular sin ejecutar
EOF
}

# --- Limpieza (rc preservado) ---
cleanup() {
    local rc=$?
    [[ -n "${TMP_DIR:-}" && -d "$TMP_DIR" ]] && rm -rf "$TMP_DIR"
    exit "$rc"
}
trap cleanup EXIT
trap 'exit 130' INT      # Ctrl+C → sale vía EXIT trap con rc correcto
trap 'exit 143' TERM

# --- Argumentos ---
parse_args() {
    DRY_RUN=0
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)   usage; exit 0 ;;
            -V|--version) echo "$VERSION"; exit 0 ;;
            -n|--dry-run) DRY_RUN=1; shift ;;
            --) shift; break ;;
            -*) usage_error "opción desconocida: $1" ;;
            *) break ;;
        esac
    done
    [[ $# -ge 1 ]] || usage_error "falta <entrada>"
    INPUT="$1"
}
usage_error() { printf '[ERROR] %s\n\n' "$*" >&2; usage >&2; exit 2; }

# --- Lógica ---
main() {
    parse_args "$@"
    [[ -f "$INPUT" ]] || die "no existe: $INPUT"
    command -v jq >/dev/null || die "requiere: jq"
    log "iniciando $SCRIPT_NAME v$VERSION (dry_run=$DRY_RUN)"
    # ... trabajo ...
    log "completado"
}

main "$@"
```

Las decisiones del esqueleto, una por una: shebang con `env` (portabilidad) · strict mode (ver [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]]) · `IFS` reforzado · constantes `readonly` · `die`/`log` a stderr · `usage` accesible por `-h` **antes** de cualquier validación · traps que preservan `rc=$?` · `main "$@"` al final. La guardia `if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then main "$@"; fi` solo si el script también se sourceará (tests) — ver [[Bash Scripting - Funciones y Scope]].

## Temporales seguros: `mktemp` + `trap`

**Nunca** `tmp="/tmp/mi_script_$$"` ni peor `/tmp/mi.tmp` (symlink attack, colisiones, race conditions):

```bash
tmp_file=$(mktemp)                 # → /tmp/tmp.Xa1b2C (nombre aleatorio, permisos 600)
tmp_dir=$(mktemp -d)               # directorio
mktemp -t "miapp.XXXXXX"           # con template (X obligatorias; -t usa TMPDIR)
mktemp /var/tmp/miapp.XXXXXX       # en un fs específico

# Patrón correcto: registrar y limpiar SIEMPRE
TMP_DIR=$(mktemp -d)
trap 'rm -rf "$TMP_DIR"' EXIT      # el trap garantiza la limpieza aunque falle

# Escritura atómica sobre el destino
printf '%s\n' "$contenido" > "$TMP_DIR/out" && mv "$TMP_DIR/out" "$destino"
```

`mktemp` crea con `0600` y nombre no adivinable — la combinación `0600 + aleatorio` elimina la carrera "otro usuario lee/escribe tu temporal". `mv` dentro del mismo filesystem es **atómico**: el destino nunca se ve a medias.

## Idempotencia: operaciones que pueden repetirse

```bash
# "Crear directorio si no existe" — mkdir solo falla si el destino NO es directorio
mkdir -p "$destino_dir"

# "Asegurar línea en archivo" (grep -qxF: exacta, sin regex, quiet)
grep -qxF 'export EDITOR=vim' ~/.bashrc || echo 'export EDITOR=vim' >> ~/.bashrc

# "Estado conocido" — los comandos de management ya son idempotentes:
systemctl is-active --quiet svc || systemctl start svc     # solo arranca si no corre
[[ -L "$link" ]] || ln -s "$real" "$link"                  # symlink solo si falta

# "Procesar solo lo pendiente" — marcador de estado
state_file="$TMP_DIR/ultimo_procesado"
[[ -f "$state_file" ]] && last=$(cat "$state_file") || last=0
for item in $(seq $((last + 1)) "$total"); do
    procesar "$item"; echo "$item" > "$state_file"
done
```

Patrón general: **preguntar el estado antes de actuar** (`mkdir -p`, `|| append`, `is-active ||`) o **marcar el progreso** para retomar. Verificado en `mkdir -p`: segundo `mkdir -p` existente sale con rc 0 sin error.

## Procesamiento de archivos a prueba de todo

```bash
# EL patrón: find -print0 + read -d '' — sobrevive a espacios, comillas, \n y \ en nombres
while IFS= read -r -d '' archivo; do
    printf 'procesando: %s\n' "$archivo"
done < <(find . -type f -name '*.txt' -print0)

# Batch con xargs -0 (paralelizable, sin subshell del while)
find . -type f -name '*.txt' -print0 | xargs -0 -n 10 rm -f --

# Lectura de líneas robusta (verificado contra última línea sin \n)
while IFS= read -r linea || [[ -n "$linea" ]]; do
    [[ "$linea" =~ ^#|^[[:space:]]*$ ]] && continue     # comentarios y vacías
    procesar "$linea"
done < "$archivo"

# Lotes grandes: mapfile en vez de 10k reads (Bash 4+)
mapfile -t -d '' -t archivos < <(find . -print0)
```

> [!warning] El `--` en `rm`/`mv`/`cp`
> `find ... -exec rm -f {} \;` con un archivo llamado `-rf` borra el directorio actual. La forma segura: `xargs -0 rm -f --` — el `--` corta el parsing de flags. Patrón de seguridad complementario en [[Bash Scripting - Fundamentos Básicos]].

## `xargs`: batching y paralelismo

`xargs` convierte stdin en argumentos de otro comando. Verificado todo lo siguiente:

```bash
printf 'a b\nc\n' | xargs -0 -n1 echo "ARG:"   # → ARG: a b / ARG: c (con -0: "a b" es UN argumento)

seq 1 6 | xargs -P3 -I{} bash -c 'echo "job {} on pid $BASHPID"'
# → corre hasta 3 en paralelo, cada job en un PID distinto (verificado)

printf '/no/existe\n' | xargs cat   # → rc=123 (cualquier hijo falló con 1-125 → xargs 123)
: | xargs -r echo "no corre"        # → -r: no ejecuta nada con stdin vacío (GNU)

find . -name '*.log' -print0 | xargs -0 -n 50 -P 4 gzip -9   # lote de 50, 4 en paralelo
```

| Flag | Efecto |
|---|---|
| `-0` | Entrada NUL-delimited (el match obligatorio de `find -print0`) |
| `-n N` | Máximo N argumentos por invocación |
| `-P N` | Hasta N procesos en paralelo |
| `-I{}` | Un argumento por línea, `{}` como placeholder (implica `-L1`) |
| `-r` | No ejecutar si stdin vacío (GNU; sin él, ejecuta el comando **sin args**) |
| `-L N` | N líneas por invocación |
| `--` | Fin de flags para xargs; lo que sigue es el comando |

Códigos de salida de `xargs`: `123` = algún hijo devolvió 1-125 · `124` = el hijo salió con 255 · `125` = xargs fue señalado · `126` = comando no ejecutable · `127` = comando no encontrado.

> [!info] `xargs -P` vs `wait` a mano
> `xargs -P4` es la forma más simple de paralelismo: controla concurrencia, hace batching y propaga rc. El patrón manual con `cmd & pid=$!` + `wait` (ver en esta nota "Worker pool") solo se justifica cuando necesitás el PID de cada job, lógica de reintento por job, o streams de salida entrelazados controlados.

## Retry con backoff exponencial + jitter

```bash
reintentar() {   # reintentar <intentos> <base_s> <max_s> -- <comando...>
    local max=${1:-3} base=${2:-1} max_delay=${3:-60}; shift 3
    [[ "$1" == "--" ]] && shift
    local intento=0 delay
    while (( intento < max )); do
        if "$@"; then return 0; fi
        (( intento += 1 ))
        (( intento >= max )) && break
        delay=$(( base * 2 ** (intento - 1) ))
        (( delay > max_delay )) && delay=$max_delay
        local jitter=$(( delay / 4 ))
        delay=$(( delay + RANDOM % (2 * jitter + 1) - jitter ))   # jitter ±25%
        (( delay < 1 )) && delay=1
        printf '[WARN] intento %d/%d falló; reintentando en %ds\n' "$intento" "$max" "$delay" >&2
        sleep "$delay"
    done
    printf '[ERROR] falló tras %d intentos\n' "$max" >&2
    return 1
}

# Uso:
reintentar 5 2 60 -- curl -fsSL --connect-timeout 5 -o "$TMP_DIR/datos.json" "$URL"
```

El **jitter** (variación aleatoria del delay) no es cosmético: sin él, N clientes reintentando al mismo tiempo sincronizan y revientan el servidor al recovery (*thundering herd*). Para HTTP, `curl --retry 3 --retry-delay 2 --retry-connrefused` cubre el caso simple.

## Logging

```bash
# Nivel configurable + timestamp + stderr (componible)
LOG_LEVEL="${LOG_LEVEL:-info}"
declare -A _lvl=([debug]=0 [info]=1 [warn]=2 [error]=3)

_log() {
    local nivel="$1"; shift
    (( ${_lvl[$nivel]} >= ${_lvl[$LOG_LEVEL]} )) || return 0
    printf '%s [%s] %s\n' "$(date '+%FT%T')" "${nivel^^}" "$*" >&2
}
debug() { _log debug "$@"; }
info()  { _log info  "$@"; }
warn()  { _log warn  "$@"; }
error() { _log error "$@"; }
die()   { error "$@"; exit 1; }

info "arrancando"                    # → 2026-10-11T12:00:00 [INFO] arrancando
DEBUG=1 _log ...                     # o por env: LOG_LEVEL=debug ./script.sh
```

Rotación simple propia (para logs que escribís desde el script) y delegada (para logs de servicios: logrotate del sistema):

```bash
rotar_si_grande() {   # rotar_si_grande <archivo> <max_bytes> <copias>
    local f=$1 max=${2:-10485760} keep=${3:-3} size
    [[ -f "$f" ]] || return 0
    size=$(stat -c%s "$f" 2>/dev/null || stat -f%z "$f")   # GNU || BSD
    (( size > max )) || return 0
    for (( i=keep-1; i>=1; i-- )); do
        [[ -f "$f.$i" ]] && mv "$f.$i" "$f.$((i+1))"
    done
    mv "$f" "$f.1"; : > "$f"
}
```

## Locks: instancias únicas

```bash
# flock (util-linux) — el estándar: bloqueo advisory sobre un fd, se libera SOLO al morir
obtener_lock() {   # retorna 0 si obtuvo el lock, 1 si ya hay instancia
    exec {LOCK_FD}>"${XDG_RUNTIME_DIR:-/tmp}/${SCRIPT_NAME}.lock" || return 1
    flock -n "$LOCK_FD" || return 1
    printf '%s\n' "$$" >&"$LOCK_FD"    # registrar PID para diagnóstico
}

# Uso en main:
if ! obtener_lock; then die "otra instancia ejecutándose"; fi

# Sección crítica entre procesos:
(
    flock -x 200
    contador=$(( $(cat /tmp/contador 2>/dev/null || echo 0) + 1 ))
    echo "$contador" > /tmp/contador
) 200>/tmp/contador.lock
```

Por qué `flock` y no el `mkdir` mítico: `flock` **se libera automáticamente** si el proceso muere (el kernel cierra el fd); los lock files de `mkdir`/`touch` quedan huérfanos tras un crash y bloquean el arranque siguiente. Fallback portable: `mkdir` atómico + trap que haga `rmdir` (ver el semáforo en la versión histórica de abajo).

> [!tip] PID files honestos
> Un PID file solo es válido si verificás que el PID sigue vivo **y** que es el mismo proceso: `kill -0 "$pid" 2>/dev/null` puede dar positivo por un PID reciclado. `flock` elimina toda esta clase de bugs — ver [[Bash Scripting - Procesos y Señales]] para la mecánica.

## Worker pool a mano (cuando `xargs -P` no basta)

```bash
MAX_JOBS=4; pids=()
lanzar() {   # lanzar <funcion> <args...> — respeta el cupo
    while (( ${#pids[@]} >= MAX_JOBS )); do
        local i
        for i in "${!pids[@]}"; do
            if ! kill -0 "${pids[i]}" 2>/dev/null; then
                wait "${pids[i]}" || failed+=("$i")    # reaping + captura de rc
                unset 'pids[i]'
            fi
        done
        pids=("${pids[@]}")   # reindexar
        sleep 0.1
    done
    "$@" & pids+=("$!")
}

for item in "${items[@]}"; do lanzar procesar_item "$item"; done
for pid in "${pids[@]}"; do wait "$pid"; done
```

Variante moderna y más limpia: `wait -n` (bash 4.3+) — esperar al primero que termine en vez del polling con `kill -0`:

```bash
active=0
for item in "${items[@]}"; do
    (( active >= MAX_JOBS )) && { wait -n; (( active -= 1 )); }
    "$item_job" "$item" &; (( active += 1 ))
done
wait
```

## Patrones de UI en terminal

```bash
# Confirmación con default
confirmar() {
    local r
    read -r -p "${1:-¿Continuar?} [s/N] " r
    [[ "$r" == [sS] ]]
}

# Spinner mientras corre un job
spinner() {   # spinner <pid> <mensaje>
    local pid=$1 msg="${2:-Procesando}" c='|/-\\' i=0
    tput civis
    while kill -0 "$pid" 2>/dev/null; do
        printf '\r [%c] %s' "${c:i++%4:1}" "$msg"; sleep 0.1
    done
    tput cnorm; printf '\r [✓] %s\n' "$msg"
}
curl -sO "$URL" & spinner $! "Descargando"

# Barra de progreso
progress() {   # progress <actual> <total>
    local pct=$(( $1 * 100 / $2 )) filled=$(( $1 * 40 / $2 ))
    printf '\r[%-40s] %3d%%' "$(printf '%*s' "$filled" '' | tr ' ' '#')" "$pct"
    (( $1 == $2 )) && echo
}
```

## Precedencia de configuración y dry-run

```bash
# defaults → archivo → env → CLI (detalle en Argumentos y Opciones)
PORT="${PORT:-8080}"
[[ -f "${XDG_CONFIG_HOME:-$HOME/.config}/app.conf" ]] && source "${XDG_CONFIG_HOME:-$HOME/.config}/app.conf"
PORT="${APP_PORT:-$PORT}"
# ... luego parse_args que pisa todo

# Dry-run: centralizar toda ejecución destructiva en una función
ejecutar() {   # ejecutar <comando...>
    if (( DRY_RUN )); then printf '[DRY] %s\n' "$*" >&2; return 0; fi
    "$@"
}
ejecutar rm -f "$archivo_viejo"
ejecutar mv "$tmp" "$destino"
```

Toda acción destructiva **pasa por `ejecutar`** — el dry-run deja de ser un afterthought y se prueba el flujo completo sin riesgo.

## HTTP robusto

```bash
http_get() {   # http_get <url> [destino]
    curl -fsSL --connect-timeout 5 --max-time 60 \
         ${2:+-o "$2"} "$1"
}
# -f: falla en HTTP ≥ 400 (sin él, un 500 devuelve la página de error con rc=0)
# -sS: silencioso pero muestra errores reales · -L: seguir redirects

api_call() {
    curl -fsSL --connect-timeout 5 \
         -H "Authorization: Bearer ${API_TOKEN:?Falta API_TOKEN}" \
         -H 'Accept: application/json' \
         -H 'Content-Type: application/json' \
         -d "$json" "$URL"
}
```

El patrón de smoke-test de puerto sin dependencias (bash builtin): `exec 3<>/dev/tcp/host/port` — ver [[Bash Scripting - Entrada-Salida y Redirección]].

## Cron-friendly: scripts sin terminal

Los scripts bajo [[Cron Jobs y Programación de Tareas]] corren con un entorno mínimo y sin terminal. Checklist:

```bash
#!/usr/bin/env bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"   # 1. PATH fijo
export LC_ALL=C                                   # 2. locale estable (sort/grep/date)
set -euo pipefail                                 # 3. strict mode
cd /                                              # 4. cwd conocido (cron arranca en $HOME)

LOG=/var/log/mi-script.log
{
    echo "=== $(date -u +'%FT%TZ') inicio ==="
    /usr/local/bin/paso1.sh                        # 5. rutas ABSOLUTAS (el PATH de cron es mínimo)
    /usr/bin/flock /var/lock/mi-script.lock /usr/local/bin/paso2.sh
} >> "$LOG" 2>&1                                  # 6. TODO redirigido: cron mailtea stdout/stderr basura
```

Errores clásicos de cron que este patrón resuelve: comando "no encontrado" (PATH de cron = `/usr/bin:/bin`) · `~` no expandida (cron no pasa por el shell de login) · variables de entorno vacías · mails de cron con warnings que nadie lee (redirigir todo) · ejecuciones solapadas (flock).

## One-liners de utilidad

```bash
date -u +'%Y-%m-%dT%H:%M:%SZ'                  # timestamp ISO-8601 UTC
cat /proc/sys/kernel/random/uuid               # UUID v4
sha256sum archivo | cut -d' ' -f1              # hash
ip -br addr show | awk '$2=="UP"{print $3}'    # IPs activas
ss -tulpn | grep LISTEN                        # puertos en escucha
df -h / | awk 'NR==2{print $5" usado de "$2}'
du -sh -- * | sort -h | tail                   # directores más pesados
jq -r '.items[] | select(.activo) | .id' f.json
find . -type f -empty                          # archivos vacíos
find . -type f -mtime +30 -delete              # (>30 días) — probar primero sin -delete
grep -rn 'patrón' --include='*.sh' .           # búsqueda recursiva filtrada
```

Detalle de cada uno en [[Comandos Básicos de Sistema y Red en Linux]].

## Estructura de proyecto y Makefile

```
mi-proyecto/
├── bin/mi-script           # ejecutable principal
├── lib/{common,logging}.sh # bibliotecas sourceadas (con guarda _LOADED)
├── test/                   # bats + fixtures
├── .shellcheckrc           # disables globales documentados
└── Makefile
```

```makefile
SHELL := /usr/bin/env bash
.PHONY: all lint test
all: lint test
lint:  ; shellcheck bin/* lib/*.sh
test:  ; bats test/
```

## Errores comunes

1. **`tmp="/tmp/x.$$"`** — predecible y con race. `mktemp` siempre.
2. **Cleanup sin `rc=$?`** — el script reporta el rc del `rm` en vez del error real.
3. **`find | while read`** con `read` sin `-d ''` — filenames con espacios partidos en dos. `-print0` + `read -d ''`.
4. **`xargs` sin `-r` en GNU** — con stdin vacío ejecuta el comando una vez igual (borra/encribe de más).
5. **`xargs` sin `-0`** — vuelve a partir por espacios todo lo que `find -print0` juntó.
6. **Cron sin PATH/redirecciones** — "command not found" y mails basura. Sección cron de arriba.
7. **Locks con `mkdir` sin cleanup** — lock huérfano tras crash bloquea el próximo arranque. `flock`.
8. **Paralelismo sin cupo** — `for ...; do cmd & done` en 50k archivos = fork bomb. `xargs -P` o worker pool.
9. **Retry sin jitter** — synchronized stampede al recovery del servicio.

## Cheatsheet

| Patrón | Forma |
|---|---|
| Temporal seguro | `d=$(mktemp -d); trap 'rm -rf "$d"' EXIT` |
| Atomic write | `> "$tmp" && mv "$tmp" "$destino"` |
| Idempotente | `mkdir -p` · `grep -qxF x f \|\| echo x >> f` |
| Archivos raros | `find . -print0 \| xargs -0 -n10 cmd --` |
| Línea robusta | `while IFS= read -r l \|\| [[ -n $l ]]; do ...; done < f` |
| Paralelo simple | `seq 1 N \| xargs -P4 -I{} cmd {}` |
| Paralelo fino | worker pool con `wait -n` |
| Retry | `reintentar 5 2 60 -- cmd` (backoff ×2 + jitter) |
| Lock único | `exec {fd}>lock; flock -n "$fd" \|\| die` |
| Dry-run | wrapper `ejecutar()` en toda acción destructiva |
| Logging | nivel + timestamp + stderr vía `_log` |
| Cron | PATH fijo, rutas absolutas, `>> log 2>&1`, flock |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — la plantilla mínima y el quoting que sostiene todo
- [[Bash Scripting - Argumentos y Opciones]] — `parse_args`/`usage` del esqueleto en detalle
- [[Bash Scripting - Entrada-Salida y Redirección]] — redirecciones de log, here-docs en templates
- [[Bash Scripting - Procesos y Señales]] — traps EXIT/TERM, `wait -n`, flock vs PID files
- [[Bash Scripting - Funciones y Scope]] — librerías sourceadas con guardas
- [[Bash Scripting - Control de Flujo]] — `case` dispatcher y bucles de archivos
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — linting y testing de estos patrones
- [[Bash Scripting - Aritmética y Números Aleatorios]] — jitter, backoff, contadores
- [[Cron Jobs y Programación de Tareas]] — el entorno hostil donde corren estos scripts
- [[Comandos Básicos de Sistema y Red en Linux]] — los comandos que componen los one-liners
