---
title: "Bash Scripting - Patrones Comunes y Plantillas"
tags:
  - lang/bash
  - concept/patterns
  - concept/templates
  - level/intermedio
  - type/pattern
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Patrones Comunes y Plantillas

> **Objetivo**: Catálogo de patrones reutilizables, plantillas base, y soluciones a problemas frecuentes.

---

## 1. Plantilla Base Profesional (Copiar-Pegar)

```bash
#!/usr/bin/env bash
# -*- mode: shell-script -*-
# Descripción: [Una línea describiendo qué hace]
# Autor: [Nombre] <[email]>
# Versión: 1.0.0
# Licencia: [MIT/Apache-2.0/GPL-3.0]

set -euo pipefail
IFS=$'\n\t'

# --- Constantes ---
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly VERSION="1.0.0"

# Colores (auto-detecta terminal)
if [[ -t 1 && -z "${NO_COLOR:-}" ]]; then
    readonly C_RESET=$'\e[0m' C_RED=$'\e[31m' C_GREEN=$'\e[32m'
    readonly C_YELLOW=$'\e[33m' C_BLUE=$'\e[34m' C_CYAN=$'\e[36m'
else
    readonly C_RESET="" C_RED="" C_GREEN="" C_YELLOW="" C_BLUE="" C_CYAN=""
fi

# --- Logging ---
log()   { printf '%b%s%b\n' "$C_BLUE"   "$*" "$C_RESET"; }
ok()    { printf '%b%s%b\n' "$C_GREEN"  "$*" "$C_RESET"; }
warn()  { printf '%b%s%b\n' "$C_YELLOW" "$*" "$C_RESET" >&2; }
err()   { printf '%b%s%b\n' "$C_RED"    "$*" "$C_RESET" >&2; }
die()   { err "$*"; exit 1; }
debug() { [[ "${DEBUG:-}" == "1" ]] && printf '%b[DEBUG]%b %s\n' "$C_CYAN" "$C_RESET" "$*" >&2; }

# --- Utilidades ---
usage() {
    cat <<EOF
Uso: $SCRIPT_NAME [OPCIONES] <argumento_requerido> [args...]

Descripción breve de qué hace el script.

Opciones:
  -h, --help       Mostrar esta ayuda y salir
  -v, --verbose    Modo verboso (más output)
  -q, --quiet      Modo silencioso (solo errores)
  -V, --version    Mostrar versión y salir
  --dry-run        Simular sin ejecutar cambios reales
  --config FILE    Archivo de configuración (default: ~/.config/$SCRIPT_NAME.conf)

Argumentos:
  argumento_requerido   Descripción del argumento principal

Variables de entorno:
  DEBUG=1          Activar debug logging
  NO_COLOR=1       Desactivar colores

Ejemplos:
  $SCRIPT_NAME archivo.txt
  $SCRIPT_NAME -v --config /etc/miapp.conf entrada.dat
  DEBUG=1 $SCRIPT_NAME dato
EOF
}

# --- Configuración por Defecto ---
VERBOSE=false
QUIET=false
DRY_RUN=false
CONFIG_FILE="${HOME}/.config/${SCRIPT_NAME}.conf"

# Cargar config si existe
[[ -f "$CONFIG_FILE" ]] && source "$CONFIG_FILE"

# --- Parseo de Argumentos ---
parse_args() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help) usage; exit 0 ;;
            -v|--verbose) VERBOSE=true; shift ;;
            -q|--quiet) QUIET=true; shift ;;
            -V|--version) echo "$VERSION"; exit 0 ;;
            --dry-run) DRY_RUN=true; shift ;;
            --config) CONFIG_FILE="$2"; shift 2 ;;
            --) shift; break ;;
            -*) die "Opción desconocida: $1" ;;
            *) break ;;
        esac
    done
    
    # Args posicionales restantes en $@
    [[ $# -ge 1 ]] || { usage; die "Falta argumento requerido"; }
}

# --- Limpieza ---
cleanup() {
    local rc=$?
    [[ -n "${TMP_DIR:-}" && -d "$TMP_DIR" ]] && rm -rf "$TMP_DIR"
    exit $rc
}
trap cleanup EXIT

# --- Lógica Principal ---
main() {
    parse_args "$@"
    
    # Validaciones
    command -v jq >/dev/null || die "Requerido: jq"
    
    log "Iniciando $SCRIPT_NAME v$VERSION"
    debug "Args: $*"
    
    # Tu lógica aquí
    procesar_entrada "$@"
    
    ok "Completado exitosamente"
}

# --- Funciones de Negocio ---
procesar_entrada() {
    local input="$1"
    # ...
}

# --- Punto de Entrada ---
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

---

## 2. Patrones de Parsing CLI

### 2.1 Subcomandos (Git-style)

```bash
cmd_init()    { log "Inicializando..."; }
cmd_build()   { log "Construyendo..."; }
cmd_deploy()  { log "Desplegando..."; }
cmd_clean()   { log "Limpiando..."; }

main() {
    local subcmd="${1:-help}"
    shift || true
    
    case $subcmd in
        init)    cmd_init "$@" ;;
        build)   cmd_build "$@" ;;
        deploy)  cmd_deploy "$@" ;;
        clean)   cmd_clean "$@" ;;
        help|--help|-h) usage ;;
        *) die "Subcomando desconocido: $subcmd. Use 'help'." ;;
    esac
}
```

### 2.2 Config File + ENV + CLI (Precedencia)

```bash
# 1. Defaults
PORT=8080
HOST="localhost"
DEBUG=false

# 2. Config file
[[ -f "$CONFIG_FILE" ]] && source "$CONFIG_FILE"

# 3. Environment variables
PORT="${MYAPP_PORT:-$PORT}"
HOST="${MYAPP_HOST:-$HOST}"
DEBUG="${MYAPP_DEBUG:-$DEBUG}"

# 4. CLI (máxima precedencia)
while [[ $# -gt 0 ]]; do
    case $1 in
        --port) PORT="$2"; shift 2 ;;
        --host) HOST="$2"; shift 2 ;;
        --debug) DEBUG=true; shift ;;
    esac
done
```

---

## 3. Patrones de Procesamiento de Archivos

### 3.1 Leer Archivo Línea a Línea (Robusto)

```bash
procesar_lineas() {
    local archivo="$1"
    local linea
    
    while IFS= read -r linea; do
        [[ -z "$linea" ]] && continue  # Saltar vacías
        [[ "$linea" =~ ^# ]] && continue  # Saltar comentarios
        
        procesar_linea "$linea"
    done < "$archivo"
}
```

### 3.2 Leer CSV Simple

```bash
leer_csv() {
    local archivo="$1"
    local IFS=","
    local -a headers
    local -a valores
    
    # Leer headers
    IFS= read -r -a headers < "$archivo"
    
    # Procesar filas
    while IFS= read -r -a valores; do
        (( ${#valores[@]} != ${#headers[@]} )) && continue
        
        declare -A fila
        for i in "${!headers[@]}"; do
            fila["${headers[i]}"]="${valores[i]}"
        done
        
        procesar_fila fila
    done < <(tail -n +2 "$archivo")
}
```

### 3.3 Procesar Archivos en Paralelo

```bash
procesar_archivos_paralelo() {
    local max_jobs=4
    local -a pids=()
    
    for archivo in "$@"; do
        # Control de concurrencia
        while (( ${#pids[@]} >= max_jobs )); do
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[i]}" 2>/dev/null; then
                    wait "${pids[i]}" 2>/dev/null
                    unset 'pids[i]'
                fi
            done
            pids=("${pids[@]}")
            sleep 0.1
        done
        
        procesar_archivo "$archivo" &
        pids+=($!)
    done
    
    # Esperar todos
    for pid in "${pids[@]}"; do wait "$pid"; done
}
```

---

## 4. Patrones de Red y API

### 4.1 HTTP Client Robusto

```bash
http_get() {
    local url="$1"
    local output="${2:-}"
    local max_retries=3
    local retry=0
    
    while (( retry < max_retries )); do
        if [[ -n "$output" ]]; then
            curl -fsSL \
                --connect-timeout 5 \
                --max-time 30 \
                --retry 0 \
                -o "$output" \
                "$url" && return 0
        else
            curl -fsSL \
                --connect-timeout 5 \
                --max-time 30 \
                --retry 0 \
                "$url" && return 0
        fi
        ((retry++))
        sleep $(( 2 ** retry ))
    done
    return 1
}

http_post_json() {
    local url="$1"
    local json="$2"
    curl -fsSL -X POST \
        -H "Content-Type: application/json" \
        -d "$json" \
        "$url"
}
```

### 4.2 API con Autenticación

```bash
api_call() {
    local method="$1"
    local endpoint="$2"
    local data="${3:-}"
    
    local auth_header="Authorization: Bearer ${API_TOKEN:?Falta API_TOKEN}"
    local headers=(-H "$auth_header" -H "Accept: application/json")
    
    [[ -n "$data" ]] && headers+=(-H "Content-Type: application/json")
    
    curl -fsSL -X "$method" \
        "${headers[@]}" \
        ${data:+-d "$data"} \
        "${API_BASE_URL}${endpoint}"
}
```

---

## 5. Patrones de Sistema

### 5.1 Lock Exclusivo (Single Instance)

```bash
# Al inicio de main()
obtener_lock() {
    local lock_file="/var/lock/${SCRIPT_NAME}.lock"
    exec {lock_fd}>"$lock_file" || die "No se pudo abrir lock"
    flock -n "$lock_fd" || die "Otra instancia ejecutándose (PID: $(cat "$lock_file" 2>/dev/null))"
    echo $$ > "$lock_file"
}
obtener_lock
```

### 5.2 PID File para Daemons

```bash
escribir_pid() {
    local pid_file="/var/run/${SCRIPT_NAME}.pid"
    echo $$ > "$pid_file"
}

leer_pid() {
    local pid_file="/var/run/${SCRIPT_NAME}.pid"
    [[ -f "$pid_file" ]] && cat "$pid_file"
}

matar_proceso_anterior() {
    local old_pid=$(leer_pid)
    if [[ -n "$old_pid" ]] && kill -0 "$old_pid" 2>/dev/null; then
        log "Matando proceso anterior: $old_pid"
        kill -TERM "$old_pid"
        sleep 2
        kill -0 "$old_pid" 2>/dev/null && kill -KILL "$old_pid"
    fi
}
```

### 5.3 Rotación de Logs Simple

```bash
rotar_log() {
    local log_file="$1"
    local max_size="${2:-10M}"
    local max_files="${3:-5}"
    
    [[ -f "$log_file" ]] || return 0
    
    local size=$(stat -c%s "$log_file" 2>/dev/null || stat -f%z "$log_file")
    local max_bytes
    case $max_size in
        *K) max_bytes=$(( ${max_size%K} * 1024 )) ;;
        *M) max_bytes=$(( ${max_size%M} * 1024 * 1024 )) ;;
        *G) max_bytes=$(( ${max_size%G} * 1024 * 1024 * 1024 )) ;;
        *) max_bytes=$max_size ;;
    esac
    
    if (( size > max_bytes )); then
        for ((i=max_files-1; i>=1; i--)); do
            [[ -f "${log_file}.$i" ]] && mv "${log_file}.$i" "${log_file}.$((i+1))"
        done
        mv "$log_file" "${log_file}.1"
        touch "$log_file"
    fi
}
```

---

## 6. Patrones de Datos

### 6.1 Cache Simple con TTL

```bash
declare -A CACHE
declare -A CACHE_TTL

cache_get() {
    local key="$1"
    local ttl="${2:-300}"  # 5 min default
    local now=$(date +%s)
    
    if [[ -v CACHE[$key] ]] && (( now - CACHE_TTL[$key] < ttl )); then
        echo "${CACHE[$key]}"
        return 0
    fi
    return 1
}

cache_set() {
    local key="$1" value="$2"
    CACHE[$key]="$value"
    CACHE_TTL[$key]=$(date +%s)
}

# Uso
if ! resultado=$(cache_get "clave_api"); then
    resultado=$(curl -s "https://api.ejemplo.com/dato")
    cache_set "clave_api" "$resultado"
fi
```

### 6.2 Contador Atómico (Archivo)

```bash
incrementar_contador() {
    local file="$1"
    local lock_file="${file}.lock"
    
    exec 9>"$lock_file"
    flock -x 9
    
    local count=0
    [[ -f "$file" ]] && count=$(cat "$file")
    count=$((count + 1))
    echo "$count" > "$file"
    
    flock -u 9
    exec 9>&-
    echo "$count"
}
```

---

## 7. Patrones de UI/UX en Terminal

### 7.1 Spinner / Progress Indicator

```bash
spinner() {
    local pid=$1
    local delay=0.1
    local spinstr='|/-\'
    local msg="${2:-Procesando...}"
    
    tput civis  # Ocultar cursor
    while kill -0 "$pid" 2>/dev/null; do
        local temp=${spinstr#?}
        printf " [%c] %s\r" "$spinstr" "$msg"
        spinstr=$temp${spinstr%"$temp"}
        sleep $delay
    done
    printf " [✓] %s\n" "$msg"
    tput cnorm  # Restaurar cursor
}

# Uso
proceso_largo &
spinner $! "Descargando datos..."
```

### 7.2 Barra de Progreso

```bash
progress_bar() {
    local current=$1 total=$2 width=50
    local percent=$(( current * 100 / total ))
    local filled=$(( current * width / total ))
    local empty=$(( width - filled ))
    
    printf "\r["
    printf "%${filled}s" | tr ' ' '█'
    printf "%${empty}s" | tr ' ' '░'
    printf "] %3d%%" "$percent"
    
    (( current == total )) && echo
}

# Uso
for i in {1..100}; do
    progress_bar $i 100
    sleep 0.05
done
```

### 7.3 Confirmación Interactiva

```bash
confirmar() {
    local prompt="${1:-¿Continuar?}"
    local default="${2:-N}"
    local yn
    
    [[ $default == [Yy]* ]] && prompt+=" [S/n] " || prompt+=" [s/N] "
    
    while true; do
        read -rp "$prompt" yn
        yn=${yn:-$default}
        case $yn in
            [SsYy]*) return 0 ;;
            [Nn]*) return 1 ;;
            *) echo "Responda sí o no." ;;
        esac
    done
}

# Uso
confirmar "¿Borrar archivos temporales?" && limpiar_temp
```

### 7.4 Selector de Menú

```bash
menu_seleccionar() {
    local prompt="$1"
    shift
    local opciones=("$@")
    local seleccion
    
    echo "$prompt"
    select seleccion in "${opciones[@]}" "Cancelar"; do
        [[ -n "$seleccion" ]] || continue
        [[ "$seleccion" == "Cancelar" ]] && return 1
        echo "$seleccion"
        return 0
    done
}

# Uso
opcion=$(menu_seleccionar "Elige entorno:" "desarrollo" "staging" "producción")
```

---

## 8. Patrones de Manejo de Errores

### 8.1 Reintento con Backoff

```bash
reintentar() {
    local max_intentos=${1:-3}
    local delay_base=${2:-1}
    local max_delay=${3:-30}
    shift 3
    
    local intento=0
    while (( intento < max_intentos )); do
        if "$@"; then
            return 0
        fi
        ((intento++))
        if (( intento < max_intentos )); then
            local delay=$(( delay_base * 2 ** (intento - 1) ))
            (( delay > max_delay )) && delay=$max_delay
            # Jitter ±25%
            local jitter=$(( delay / 4 ))
            delay=$(( delay + RANDOM % (2 * jitter + 1) - jitter ))
            (( delay < 1 )) && delay=1
            warn "Intento $intento falló. Reintentando en ${delay}s..."
            sleep "$delay"
        fi
    done
    err "Falló tras $max_intentos intentos"
    return 1
}

# Uso
reintentar 3 2 30 curl -fsSL -o archivo.zip "https://ejemplo.com/archivo.zip"
```

### 8.2 Circuit Breaker Simple

```bash
declare -A CB_STATE
declare -A CB_FAILURES
declare -A CB_LAST_FAIL

circuit_breaker() {
    local name="$1"
    shift
    local threshold=5
    local timeout=60
    
    local now=$(date +%s)
    local failures=${CB_FAILURES[$name]:-0}
    local last_fail=${CB_LAST_FAIL[$name]:-0}
    local state=${CB_STATE[$name]:-closed}
    
    # Reset si pasó timeout
    if (( now - last_fail > timeout )); then
        CB_FAILURES[$name]=0
        CB_STATE[$name]=closed
        state=closed
    fi
    
    if [[ $state == "open" ]]; then
        err "Circuit breaker OPEN para $name"
        return 1
    fi
    
    if "$@"; then
        CB_FAILURES[$name]=0
        CB_STATE[$name]=closed
        return 0
    else
        CB_FAILURES[$name]=$((failures + 1))
        CB_LAST_FAIL[$name]=$now
        if (( CB_FAILURES[$name] >= threshold )); then
            CB_STATE[$name]=open
            warn "Circuit breaker OPENED para $name tras $threshold fallos"
        fi
        return 1
    fi
}
```

---

## 9. Snippets Útiles (One-Liners)

```bash
# Timestamp ISO 8601
date -u +'%Y-%m-%dT%H:%M:%SZ'

# UUID v4
cat /proc/sys/kernel/random/uuid  # Linux
# python3 -c "import uuid; print(uuid.uuid4())"  # Portable

# Hash de archivo
sha256sum archivo | cut -d' ' -f1

# Tamaño legible
du -sh directorio
ls -lh archivo

# IPs locales
ip -br addr show | awk '$2=="UP"{print $3}' | cut -d/ -f1

# Puertos en uso
ss -tulpn | grep LISTEN

# Procesos por usuario
ps -u "$USER" -o pid,cmd

# Matar procesos por patrón
pkill -f "patrón_comando"

# Espacio en disco
df -h / | awk 'NR==2{print $5 " usado de " $2}'

# Memoria
free -h | awk 'NR==2{print $3 "/" $2}'

# CPU load
uptime | awk -F'load average:' '{print $2}'

# JSON pretty print
jq . archivo.json
cat archivo.json | jq .

# YAML a JSON
python3 -c "import yaml, json, sys; print(json.dumps(yaml.safe_load(sys.stdin)))"

# Base64 encode/decode
echo "texto" | base64
echo "dGV4dG8=" | base64 -d

# URL encode/decode
python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "texto con espacios"
python3 -c "import urllib.parse, sys; print(urllib.parse.unquote(sys.argv[1]))" "texto%20con%20espacios"
```

---

## 10. Estructura de Proyecto Recomendada

```
mi-proyecto/
├── bin/
│   └── mi-script          # Script principal (ejecutable)
├── lib/
│   ├── common.sh          # Funciones compartidas
│   ├── logging.sh         # Logging functions
│   ├── http.sh            # HTTP client
│   └── config.sh          # Config parsing
├── config/
│   └── default.conf       # Configuración por defecto
├── test/
│   ├── test_mi_script.bats
│   └── fixtures/
├── docs/
│   └── README.md
├── .shellcheckrc          # Configuración ShellCheck
├── .gitignore
├── Makefile               # Comandos comunes
└── README.md
```

### Makefile Estándar

```makefile
SHELL := /usr/bin/env bash
SCRIPTS := $(wildcard bin/*.sh)

.PHONY: all lint test install clean

all: lint test

lint:
	shellcheck $(SCRIPTS) lib/*.sh

test:
	bats test/

install:
	install -m 755 bin/mi-script /usr/local/bin/mi-script

clean:
	rm -rf /tmp/mi-script-*
```

---

## 11. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Plantilla base
- [Argumentos y Opciones](./Bash%20Scripting%20-%20Argumentos%20y%20Opciones.md) - Parsing CLI
- [Entrada-Salida y Redirección](./Bash%20Scripting%20-%20Entrada-Salida%20y%20Redirecci%C3%B3n.md) - Patrones de archivos
- [Procesos y Señales](./Bash%20Scripting%20-%20Procesos%20y%20Se%C3%B1ales.md) - Locks, daemons
- [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md)

---

*Etiquetas: #bash #patrones #plantillas #templates #cli #http #daemon #lock #cache #spinner #progress #retry #circuit-breaker #makefile*