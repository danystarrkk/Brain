---
title: "Bash Scripting - Argumentos y Opciones (CLI Parsing)"
tags:
  - lang/bash
  - concept/cli
  - concept/parsing
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Argumentos y Opciones (CLI Parsing)

> **Objetivo**: Crear interfaces de línea de comandos robustas, estándar y auto-documentadas.

---

## 1. Parámetros Posicionales Básicos

```bash
#!/usr/bin/env bash

echo "Script: $0"
echo "Arg 1: $1"
echo "Arg 2: $2"
echo "Total: $#"
echo "Todos (array): $@"
echo "Todos (string): $*"

# Shift manual
while [[ $# -gt 0 ]]; do
    echo "Procesando: $1"
    shift
done
```

---

## 2. `getopts` - Opciones Cortas (Built-in, POSIX)

```bash
# Sintaxis: getopts "optstring" var
# optstring: letras de opciones, : después = requiere argumento
# var: variable donde se guarda la opción actual

uso() {
    cat <<EOF
Uso: $0 [-v] [-f ARCHIVO] [-n NUM] [--] ARGS...
  -v, --verbose    Modo verboso
  -f, --file       Archivo de entrada (requerido)
  -n, --number     Número de iteraciones (default: 10)
  -h, --help       Mostrar esta ayuda
EOF
}

main() {
    local verbose=false
    local archivo=""
    local numero=10
    
    # OPTIND se resetea automáticamente al entrar a función
    local OPTIND opt
    while getopts ":vf:n:h" opt; do
        case $opt in
            v) verbose=true ;;
            f) archivo="$OPTARG" ;;
            n) numero="$OPTARG" ;;
            h) uso; exit 0 ;;
            :) echo "Error: -$OPTARG requiere argumento" >&2; uso; exit 1 ;;
            \?) echo "Error: opción inválida -$OPTARG" >&2; uso; exit 1 ;;
        esac
    done
    
    # Eliminar opciones procesadas, dejar args posicionales
    shift $((OPTIND - 1))
    
    # Validaciones
    [[ -n "$archivo" ]] || { echo "Error: -f requerido" >&2; uso; exit 1; }
    [[ -f "$archivo" ]] || { echo "Error: $archivo no existe" >&2; exit 1; }
    
    echo "Verbose: $verbose, Archivo: $archivo, Num: $numero"
    echo "Args restantes: $@"
}

main "$@"
```

### 2.1 `getopts` Avanzado: Flags Booleanos Múltiples

```bash
# Permitir -abc equivalente a -a -b -c
while getopts ":abvc:" opt; do
    case $opt in
        a) flag_a=true ;;
        b) flag_b=true ;;
        v) verbose=true ;;
        c) valor_c="$OPTARG" ;;
    esac
done
```

---

## 3. Parsing Manual - Opciones Largas (GNU Style)

```bash
# Para opciones largas (--verbose, --file=archivo) sin getopt externo
parse_args() {
    local args=()
    local verbose=false
    local archivo=""
    local numero=10
    
    while [[ $# -gt 0 ]]; do
        case $1 in
            -v|--verbose)
                verbose=true
                shift
                ;;
            -f|--file)
                [[ $# -lt 2 ]] && { echo "Error: --file requiere valor" >&2; return 1; }
                archivo="$2"
                shift 2
                ;;
            --file=*)
                archivo="${1#*=}"
                shift
                ;;
            -n|--number)
                [[ $# -lt 2 ]] && { echo "Error: --number requiere valor" >&2; return 1; }
                numero="$2"
                shift 2
                ;;
            --number=*)
                numero="${1#*=}"
                shift
                ;;
            -h|--help)
                uso; exit 0
                ;;
            --)  # Fin de opciones
                shift
                args+=("$@")
                break
                ;;
            -*)
                echo "Error: opción desconocida: $1" >&2
                uso; exit 1
                ;;
            *)
                args+=("$1")
                shift
                ;;
        esac
    done
    
    # Devolver args posicionales en array global o stdout
    POSITIONAL_ARGS=("${args[@]}")
    # Exportar variables si se usa en main script
    export PARSED_VERBOSE="$verbose"
    export PARSED_ARCHIVO="$archivo"
    export PARSED_NUMERO="$numero"
}
```

---

## 4. `getopt` (Externo, GNU/BSD) - Opciones Largas Reales

```bash
# getopt es comando externo (no built-in), más potente que getopts
# Requiere: util-linux (Linux) o getopt de GNU en macOS (brew install gnu-getopt)

parse_with_getopt() {
    local TEMP
    TEMP=$(getopt -o vf:n:h --long verbose,file:,number:,help -n "$0" -- "$@")
    [[ $? -ne 0 ]] && return 1
    
    eval set -- "$TEMP"
    
    local verbose=false archivo="" numero=10
    
    while true; do
        case $1 in
            -v|--verbose) verbose=true; shift ;;
            -f|--file) archivo="$2"; shift 2 ;;
            -n|--number) numero="$2"; shift 2 ;;
            -h|--help) uso; exit 0 ;;
            --) shift; break ;;
            *) echo "Error interno"; exit 1 ;;
        esac
    done
    
    # Args posicionales en $@
    echo "Verbose: $verbose, Archivo: $archivo, Num: $numero, Args: $@"
}
```

---

## 5. Subcomandos (Git-style CLI)

```bash
#!/usr/bin/env bash
# Estilo: script subcomando [opciones] [args]

cmd_iniciar() {
    echo "Iniciando servicio..."
    # Opciones específicas del subcomando
    local force=false
    while [[ $# -gt 0 ]]; do
        case $1 in
            -f|--force) force=true; shift ;;
            *) break ;;
        esac
    done
    [[ $force == true ]] && echo "Forzado"
}

cmd_detener() {
    echo "Deteniendo..."
}

cmd_estado() {
    echo "Estado: corriendo"
}

cmd_ayuda() {
    cat <<EOF
Uso: $0 <comando> [opciones]

Comandos:
  iniciar   Iniciar servicio
  detener   Detener servicio
  estado    Ver estado
  ayuda     Mostrar esta ayuda
EOF
}

# Dispatcher principal
main() {
    local subcmd="${1:-ayuda}"
    shift || true
    
    case $subcmd in
        iniciar|start)   cmd_iniciar "$@" ;;
        detener|stop)    cmd_detener "$@" ;;
        estado|status)   cmd_estado "$@" ;;
        ayuda|help|-h|--help) cmd_ayuda ;;
        *) echo "Comando desconocido: $subcmd"; cmd_ayuda; exit 1 ;;
    esac
}

main "$@"
```

---

## 6. Validación y Tipado de Argumentos

```bash
validar_entero() {
    local val="$1" nombre="$2"
    [[ "$val" =~ ^-?[0-9]+$ ]] || { echo "Error: $nombre debe ser entero, got '$val'" >&2; return 1; }
}

validar_archivo_existe() {
    local path="$1" nombre="$2"
    [[ -f "$path" ]] || { echo "Error: $nombre '$path' no existe o no es archivo" >&2; return 1; }
}

validar_directorio() {
    local path="$1" nombre="$2"
    [[ -d "$path" ]] || { echo "Error: $nombre '$path' no es directorio" >&2; return 1; }
}

validar_opcion() {
    local val="$1" nombre="$2" ; shift 2
    local validos=("$@")
    for v in "${validos[@]}"; do
        [[ "$val" == "$v" ]] && return 0
    done
    echo "Error: $nombre debe ser uno de: ${validos[*]}, got '$val'" >&2
    return 1
}

# Uso
validar_entero "$numero" "iteraciones" || exit 1
validar_archivo_existe "$archivo" "archivo entrada" || exit 1
validar_opcion "$modo" "modo" "rapido" "normal" "completo" || exit 1
```

---

## 7. Completado (Tab Completion) - Básico

```bash
# Generar script de completion para bash
# Guardar como /etc/bash_completion.d/mi_script o sourcear en .bashrc

_mi_script_completions() {
    local cur="${COMP_WORDS[COMP_CWORD]}"
    local prev="${COMP_WORDS[COMP_CWORD-1]}"
    local opts="-v --verbose -f --file -n --number -h --help"
    local subcmds="iniciar detener estado ayuda"
    
    if [[ $COMP_CWORD -eq 1 ]]; then
        COMPREPLY=( $(compgen -W "$subcmds" -- "$cur") )
    else
        case "${COMP_WORDS[1]}" in
            iniciar)
                COMPREPLY=( $(compgen -W "-f --force" -- "$cur") )
                ;;
            *)
                COMPREPLY=( $(compgen -W "$opts" -- "$cur") )
                ;;
        esac
    fi
}

complete -F _mi_script_completions mi_script
```

---

## 8. Patrones de Diseño CLI

### 8.1 Configuración por Archivo + CLI (Precedencia)

```bash
# 1. Defaults
config_verbose=false
config_puerto=8080

# 2. Archivo de config (source seguro)
[[ -f "$HOME/.miscript.conf" ]] && source "$HOME/.miscript.conf"

# 3. Variables de entorno (override config)
config_verbose="${MI_SCRIPT_VERBOSE:-$config_verbose}"
config_puerto="${MI_SCRIPT_PUERTO:-$config_puerto}"

# 4. CLI (máxima precedencia)
parse_args "$@"
config_verbose="${PARSED_VERBOSE:-$config_verbose}"
config_puerto="${PARSED_NUMERO:-$config_puerto}"  # si -n mapea a puerto
```

### 8.2 Modo Dry-Run / Simulación

```bash
dry_run=false

# En funciones que modifican estado
ejecutar_cambio() {
    if [[ $dry_run == true ]]; then
        log_info "[DRY-RUN] Ejecutaría: $*"
        return 0
    fi
    # Ejecución real
    "$@"
}

# En main
case $1 in
    --dry-run) dry_run=true; shift ;;
esac
```

---

## 9. Librerías Recomendadas (Para CLIs Complejos)

| Librería | Descripción | Instalación |
|----------|-------------|-------------|
| **argparse.bash** | Port de argparse de Python | `curl -sL ...` |
| **bashly** | Generador de CLI desde YAML | `gem install bashly` |
| **docopts** | Parsing desde docstring (docopt) | `pip install docopts` |
| **shflags** | Google-style flags | Incluir .sh |

---

## 10. Checklist CLI Robusto

- [ ] `set -euo pipefail` al inicio
- [ ] `usage()` / `--help` clara y completa
- [ ] Validación temprana de args requeridos
- [ ] Mensajes de error a `stderr` (`>&2`)
- [ ] Exit codes significativos (0=ok, 1=error general, 2=uso inválido, 3=permiso, etc.)
- [ ] Soporte `--` para separar opciones de args
- [ ] Manejo de args con espacios/comillas (`"$@"`)
- [ ] Completion opcional para UX
- [ ] Tests de CLI (bats, shunit2)

---

## 11. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Plantilla base con parsing simple
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md) - Estructura modular
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - Case para parsing
- [Entrada-Salida y Redirección](./Bash%20Scripting%20-%20Entrada-Salida%20y%20Redirecci%C3%B3n.md) - Stdin/stdout/stderr

---

*Etiquetas: #bash #cli #getopts #getopt #argumentos #opciones #subcomandos #validacion #completion*