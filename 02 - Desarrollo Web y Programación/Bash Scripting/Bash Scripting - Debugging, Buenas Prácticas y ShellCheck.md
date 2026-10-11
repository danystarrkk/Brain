---
title: "Bash Scripting - Debugging, Buenas Prácticas y ShellCheck"
tags:
  - lang/bash
  - concept/debugging
  - concept/best-practices
  - concept/linting
  - level/intermedio
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Debugging, Buenas Prácticas y ShellCheck

> **Objetivo**: Escribir scripts robustos, mantenibles, y debuggeables usando herramientas y patrones probados.

---

## 1. Modo Estricto (Strict Mode) - Línea 1 Obligatoria

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Explicación:
# -e  : Exit inmediato si cualquier comando falla (exit code != 0)
# -u  : Error si se usa variable no definida (${var:-default} para opcionales)
# -o pipefail : Pipeline falla si CUALQUIER comando falla (no solo el último)
# IFS=$'\n\t' : Word splitting solo en newlines/tabs (no espacios)
```

### 1.1 Excepciones Controladas

```bash
# Comando que puede fallar sin matar script
comando_riesgoso || true
# O capturar exit code
if ! comando_riesgoso; then
    echo "Falló con código $?"
fi

# Desactivar temporalmente
set +e
comando_que_puede_fallar
set -e
```

---

## 2. ShellCheck - Linter Obligatorio

### 2.1 Instalación

```bash
# Linux
sudo apt install shellcheck        # Debian/Ubuntu
sudo dnf install shellcheck        # Fedora
sudo pacman -S shellcheck          # Arch

# macOS
brew install shellcheck

# Docker (sin instalar)
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable script.sh
```

### 2.2 Uso Básico

```bash
# Verificar un script
shellcheck script.sh

# Verificar todos los .sh del proyecto
find . -name '*.sh' -exec shellcheck {} +

# Formato específico
shellcheck -f gcc script.sh       # Formato GCC (para IDEs)
shellcheck -f json script.sh      # JSON para parsing
shellcheck -f checkstyle script.sh

# Severidad
shellcheck --severity=warning script.sh
shellcheck --severity=error script.sh
```

### 2.3 Directivas de Exclusión (Comentarios)

```bash
# Deshabilitar SC2034 (variable no usada) para una línea
# shellcheck disable=SC2034
variable_no_usada="valor"

# Deshabilitar múltiples
# shellcheck disable=SC2034,SC2154

# Deshabilitar para bloque
# shellcheck disable=SC2034
var1="a"
var2="b"
# shellcheck enable=SC2034
```

### 2.4 Códigos Comunes y Fixes

| Código | Problema | Solución |
|--------|----------|----------|
| SC1090 | `source` archivo no encontrado | `source "$(dirname "$0")/lib.sh"` |
| SC1091 | No seguir source | Añadir `# shellcheck source=lib.sh` |
| SC2034 | Variable no usada | Usar `_` prefijo: `_unused=1` o remover |
| SC2086 | Variable sin comillas | `"$var"` no `$var` |
| SC2154 | Variable referenciada pero no asignada | Verificar typos, usar `declare` |
| SC2155 | Declaración y asignación separadas | `local var; var=$(cmd)` o `local var=$(cmd)` con `set -e` |
| SC2016 | Expresión en comillas simples | Usar comillas dobles: `"$var"` |
| SC2002 | `cat file | cmd` inútil | `cmd < file` |
| SC2046 | Word splitting en command sub | `mapfile -t arr < <(cmd)` o `readarray` |
| SC2001 | `sed` para replace simple | `${var//patrón/reemplazo}` |
| SC2039 | `local` en POSIX sh | Usar `#!/usr/bin/env bash` |
| SC2317 | Código inalcanzable tras `exit`/`return` | Revisar lógica |

---

## 3. Técnicas de Debugging

### 3.1 Modo Verbose/Trace

```bash
# Activar trace (imprime cada comando antes de ejecutar)
set -x
# Desactivar
set +x

# Solo para sección
(
    set -x
    comando1
    comando2
)

# PS4 personalizado (formato trace)
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

### 3.2 Logging Estructurado

```bash
# Niveles: DEBUG < INFO < WARN < ERROR
readonly LOG_LEVEL_DEBUG=0
readonly LOG_LEVEL_INFO=1
readonly LOG_LEVEL_WARN=2
readonly LOG_LEVEL_ERROR=3

# Configurable
LOG_LEVEL="${LOG_LEVEL:-$LOG_LEVEL_INFO}"

_log() {
    local level=$1; shift
    local level_name=$1; shift
    (( level >= LOG_LEVEL )) || return 0
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    printf '[%s] [%s] %s\n' "$timestamp" "$level_name" "$*" >&2
}

log_debug() { _log $LOG_LEVEL_DEBUG DEBUG "$@"; }
log_info()  { _log $LOG_LEVEL_INFO  INFO  "$@"; }
log_warn()  { _log $LOG_LEVEL_WARN  WARN  "$@"; }
log_error() { _log $LOG_LEVEL_ERROR ERROR "$@"; }

# Uso
log_info "Iniciando proceso"
log_debug "Variable x=$x"
log_warn "Reintento 3/5"
log_error "Fallo crítico: $err"
```

### 3.3 Debugging de Variables

```bash
# Inspeccionar variable
declare -p variable    # Muestra declaración con atributos
declare -p array_var   # Array completo

# En funciones
debug_vars() {
    local var
    for var in "$@"; do
        if [[ -v $var ]]; then
            declare -p "$var"
        else
            echo "$var: (unset)"
        fi
    done
}
debug_vars x y arr

# Volcar todo el entorno
env | sort
set -o posix; set  # Todas las variables y funciones
```

### 3.4 Breakpoints Condicionales

```bash
# Breakpoint manual (requiere terminal interactivo)
breakpoint() {
    [[ -t 0 ]] || return  # Solo si stdin es terminal
    local msg="${1:-Breakpoint}"
    echo ">>> $msg (${BASH_SOURCE[1]}:${BASH_LINENO[0]})"
    echo "Presiona Enter para continuar, 'q' para salir"
    read -r input
    [[ $input == "q" ]] && exit 1
}

# Uso
breakpoint "Antes de procesar datos críticos"
```

### 3.5 `trap` para Debugging

```bash
# Trace de todas las llamadas a función
set -o functrace
trap 'echo "CALL: ${FUNCNAME[*]}" >&2' DEBUG

# Trace de asignaciones
trap 'echo "ASSIGN: $BASH_COMMAND" >&2' DEBUG

# Error handler con contexto
error_handler() {
    local line=$1
    local cmd=$2
    local rc=$3
    log_error "Línea $line: '$cmd' falló con exit code $rc"
    log_error "Stack: ${FUNCNAME[*]}"
    exit $rc
}
trap 'error_handler ${LINENO} "$BASH_COMMAND" $?' ERR
```

---

## 4. Buenas Prácticas de Estilo

### 4.1 Naming Conventions

```bash
# Variables globales/constantes: UPPER_SNAKE_CASE
readonly MAX_REINTENTOS=3
readonly API_ENDPOINT="https://api.ejemplo.com"

# Variables locales/funciones: lower_snake_case
procesar_archivo() {
    local archivo_entrada="$1"
    local archivo_salida="$2"
    # ...
}

# Funciones: verbo_sustantivo
validar_entrada()
guardar_configuracion()
leer_archivo()

# Arrays: plural
archivos=()
usuarios=()

# Temporales: prefijo _ o sufijo _tmp
_tmp_file=$(mktemp)
```

### 4.2 Estructura de Script Profesional

```bash
#!/usr/bin/env bash
# -*- mode: shell-script -*-
# Descripción: Breve descripción del script
# Autor: Nombre <email>
# Versión: 1.0.0
# Licencia: MIT

set -euo pipefail
IFS=$'\n\t'

# --- Constantes ---
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly VERSION="1.0.0"

# Colores (solo si terminal)
if [[ -t 1 && -z "${NO_COLOR:-}" ]]; then
    readonly RED=$'\e[31m' GREEN=$'\e[32m' YELLOW=$'\e[33m' BLUE=$'\e[34m' RESET=$'\e[0m'
else
    readonly RED="" GREEN="" YELLOW="" BLUE="" RESET=""
fi

# --- Funciones de Utilidad ---
log()   { printf '%b%s%b\n' "$BLUE"   "$*" "$RESET"; }
ok()    { printf '%b%s%b\n' "$GREEN"  "$*" "$RESET"; }
warn()  { printf '%b%s%b\n' "$YELLOW" "$*" "$RESET" >&2; }
err()   { printf '%b%s%b\n' "$RED"    "$*" "$RESET" >&2; }
die()   { err "$*"; exit 1; }

usage() {
    cat <<EOF
Uso: $SCRIPT_NAME [OPCIONES] ARGS...

Descripción breve.

Opciones:
  -h, --help     Mostrar esta ayuda
  -v, --verbose  Modo verboso
  -V, --version  Mostrar versión

Ejemplos:
  $SCRIPT_NAME -v archivo.txt
EOF
}

# --- Lógica Principal ---
main() {
    local verbose=false
    
    # Parseo args (ver nota dedicada)
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help) usage; exit 0 ;;
            -v|--verbose) verbose=true; shift ;;
            -V|--version) echo "$VERSION"; exit 0 ;;
            --) shift; break ;;
            -*) die "Opción desconocida: $1" ;;
            *) break ;;
        esac
    done
    
    [[ $# -ge 1 ]] || { usage; die "Falta argumento requerido"; }
    
    log "Iniciando $SCRIPT_NAME v$VERSION"
    # ... lógica ...
    ok "Completado exitosamente"
}

# --- Punto de Entrada ---
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

### 4.3 Comentarios y Documentación

```bash
# Comentario de una línea: explica POR QUÉ, no QUÉ

### Sección principal (para grep/awk parsing)

# Función con docstring estilo
# Nombre: procesar_archivo
# Descripción: Procesa archivo de entrada y genera salida
# Argumentos:
#   $1 - archivo_entrada (requerido)
#   $2 - archivo_salida (opcional, default: stdout)
# Retorna: 0 en éxito, 1 en error
# Ejemplo: procesar_archivo entrada.txt salida.txt
procesar_archivo() {
    local archivo_entrada="${1:?Falta archivo entrada}"
    local archivo_salida="${2:-/dev/stdout}"
    # ...
}
```

---

## 5. Patrones de Robustez

### 5.1 Validación Temprana

```bash
main() {
    # Validar dependencias
    for cmd in jq curl awk; do
        command -v "$cmd" >/dev/null || die "Requerido: $cmd"
    done
    
    # Validar archivos
    [[ -r "$archivo_entrada" ]] || die "No legible: $archivo_entrada"
    [[ -w "$(dirname "$archivo_salida")" ]] || die "No escribible: $archivo_salida"
    
    # Validar permisos
    [[ -x "$(command -v docker)" ]] || warn "Docker no disponible, modo simulado"
}
```

### 5.2 Limpieza Garantizada (`trap EXIT`)

```bash
cleanup() {
    local rc=$?
    [[ -n "${tmp_dir:-}" && -d "$tmp_dir" ]] && rm -rf "$tmp_dir"
    [[ -n "${lock_fd:-}" ]] && exec {lock_fd}>&-
    exit $rc
}
trap cleanup EXIT

# Crear recursos
tmp_dir=$(mktemp -d -t "miscript.XXXXXX")
exec {lock_fd}>/tmp/miscript.lock
flock -x "$lock_fd" || die "Otra instancia corriendo"
```

### 5.3 Timeouts en Operaciones de Red

```bash
# curl con timeouts estrictos
curl -fsSL \
    --connect-timeout 5 \
    --max-time 30 \
    --retry 3 \
    --retry-delay 2 \
    --retry-connrefused \
    "https://api.ejemplo.com/datos" \
    -o "$tmp_dir/datos.json" \
    || die "Descarga falló tras reintentos"
```

### 5.4 Manejo de Señales

```bash
shutdown_graceful() {
    log_info "Señal de apagado recibida, limpiando..."
    # Cancelar hijos
    pkill -P $$
    wait
    exit 0
}
trap shutdown_graceful TERM INT
```

---

## 6. Testing

### 6.1 BATS (Bash Automated Testing System)

```bash
# Instalación
# sudo apt install bats / brew install bats-core

# test/mi_script.bats
#!/usr/bin/env bats

load 'libs/bats-support/load'
load 'libs/bats-assert/load'

setup() {
    # Preparar entorno de test
    export TEST_TMPDIR="$(mktemp -d)"
}

teardown() {
    rm -rf "$TEST_TMPDIR"
}

@test "procesar_archivo convierte mayúsculas" {
    echo "hola" > "$TEST_TMPDIR/entrada.txt"
    run bash -c "source ../mi_script.sh; procesar_archivo '$TEST_TMPDIR/entrada.txt' '$TEST_TMPDIR/salida.txt'"
    assert_success
    assert_output ""
    assert_equal "$(cat "$TEST_TMPDIR/salida.txt")" "HOLA"
}

@test "validar_entrada falla con archivo inexistente" {
    run bash -c "source ../mi_script.sh; validar_entrada 'noexiste.txt'"
    assert_failure
    assert_output --partial "No legible"
}
```

### 6.2 Ejecutar Tests

```bash
# Todos los tests
bats test/

# Con output TAP
bats --tap test/

# Filtrar
bats -f "procesar" test/
```

---

## 7. CI/CD Integration

### 7.1 GitHub Actions / GitLab CI

```yaml
# .github/workflows/shellcheck.yml
name: ShellCheck
on: [push, pull_request]
jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        run: |
          find . -name '*.sh' -exec shellcheck {} +
```

### 7.2 Pre-commit Hook

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.9.0
    hooks:
      - id: shellcheck
        args: ['--severity=warning']
```

---

## 8. Performance Tips

| Problema | Lento | Rápido |
|----------|-------|--------|
| Loop sobre archivo | `while read; done < file` | `mapfile -t arr < file` |
| String concat en loop | `str+="$x"` | `arr+=("$x"); str="${arr[*]}"` |
| Subshell en pipe | `cmd | while read; done` | `while read; done < <(cmd)` |
| `cat file | grep` | `cat file \| grep` | `grep pattern file` |
| `sed` para replace | `sed 's/a/b/' <<< "$var"` | `${var//a/b}` |
| `expr` / `let` | `expr 1 + 1` | `(( 1 + 1 ))` |
| `date` en loop | `date +%s` repetido | `start=$(date +%s); ... $(( $(date +%s) - start ))` |

---

## 9. Checklist Pre-Commit

```
[ ] Shebang correcto: #!/usr/bin/env bash
[ ] set -euo pipefail + IFS=$'\n\t'
[ ] ShellCheck pasa sin warnings (o justificados)
[ ] Variables globales: readonly + UPPER_SNAKE
[ ] Variables locales: local + lower_snake
[ ] Comillas dobles en TODAS las expansiones: "$var"
[ ] Validación de args y dependencias al inicio
[ ] trap cleanup EXIT para recursos temporales
[ ] Logging estructurado (no echo crudo)
[ ] Manejo de errores en pipes (pipefail)
[ ] Tests BATS para funciones críticas
[ ] Documentación: usage(), docstrings en funciones
[ ] Sin código muerto (SC2317)
[ ] Sin variables sin usar (SC2034)
```

---

## 10. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Plantilla base con strict mode
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md)
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md)
- [Argumentos y Opciones](./Bash%20Scripting%20-%20Argumentos%20y%20Opciones.md)
- [Procesos y Señales](./Bash%20Scripting%20-%20Procesos%20y%20Se%C3%B1ales.md)

---

*Etiquetas: #bash #debugging #shellcheck #best-practices #testing #bats #logging #strict-mode #robustez #ci-cd*