---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Argumentos y Opciones

La interfaz de un script **es** su parsing de argumentos: es lo primero que toca el usuario y lo que decide si el script se puede invocar desde cron, CI u otros scripts. Bash trae dos mecanismos builtins (`getopts` y el parsing manual con `case`), más el binario externo `getopt` (GNU/BSD) para opciones largas. Este nota cubre los parámetros especiales hasta el nivel de detalle del parsing, las tres estrategias de parsing con sus límites reales, y el esqueleto de un CLI completo y robusto.

## Los parámetros especiales, en detalle

```bash
#!/usr/bin/env bash
# invocado como: ./script.sh -v "arg con espacios" segundo

echo "0:  $0"        # → ./script.sh     cómo se invocó (o bash/path si: bash script.sh)
echo "#:  $#"        # → 2               cantidad de argumentos (sin $0)
echo "1:  $1"        # → -v
echo "@:  $@"        # → -v · arg con espacios · segundo  (como lista)
echo "*:  $*"        # → -v arg con espacios segundo     (como string)
echo "?:  $?"        # rc del último comando
echo "!:  $!"        # PID del último background
echo "-:  $-"        # flags activos (set -e...)
```

### `$@` vs `$*`: la diferencia que importa

| Contexto | `"$@"` | `"$*"` | `$@` / `$*` sin comillas |
|---|---|---|---|
| Con 2 args `"a b"` `"c"` | **2 palabras**: `a b` · `c` | 1 palabra: `a b c` | splitting de nuevo: 3 palabras |
| En `printf '[%s]\n'` | 1 línea por argumento (verificado) | 1 sola línea | basura con espacios |
| En `rm`/`cp` | respeta los límites | los une: nombres rotos | borra/copias cosas de más |

Verificado: `set -- "a b" "c"; printf '[%s]\n' "$@"` → `[a b]` `[c]`. Y `IFS=-; printf '[%s]\n' "$*"` → `[a b-c]` (une con el **primer carácter** de IFS).

> [!info] Regla de oro
> **`"$@"` entre comillas, siempre, en todo call site.** Es la única expansión que preserva argumentos con espacios, glob y caracteres raros. `"$*`" casi nunca es lo que querés; si necesitás un string unido, hacé `printf -v joined '%s ' "$@"` o pasá por `join_by`.

### Slicing de argumentos

```bash
set -- uno dos tres cuatro
echo "${@:2}"       # → dos tres cuatro
echo "${@:2:2}"     # → dos tres        (desde pos 2, longitud 2)
echo "${@: -1}"     # → cuatro          (último, espacio obligatorio)
echo "${#:}"        # → error: no hay slicing de $#; usar $#
```

### `shift`: consumir argumentos

```bash
shift               # descarta $1; $2 pasa a $1; $# baja en 1
shift 2             # descarta los dos primeros
shift 99            # si $# < 99 → error "shift count out of range" (rc>0; con set -e mata)
shift $((OPTIND-1)) # patrón getopts: dejar solo los posicionales
shift "$#"          # deja $@ vacío de forma segura
```

`shift` sobre `$#=0` devuelve rc=1 (con `set -e` hay que blindarlo: `(( $# )) && shift` o `shift || true`).

## `getopts`: opciones cortas builtin

`getopts optstring var [args...]` es el parser builtin POSIX. Recorre los argumentos, setea `var` con la letra de la opción y `OPTARG` con su valor, y avanza `OPTIND`.

```bash
# optstring: letras; una `:` tras la letra = REQUIERE argumento.
#            optstring que EMPIEZA con `:` = modo silencioso (manejo manual de errores).
parse() {
    local OPTIND=1 opt          # ← local OPTIND=1 OBLIGATORIO en funciones (ver gotcha)
    local verbose=0 archivo="" numero=10

    while getopts ":vf:n:h" opt; do
        case $opt in
            v) verbose=1 ;;
            f) archivo="$OPTARG" ;;
            n) numero="$OPTARG" ;;
            h) usage; exit 0 ;;
            :) printf 'error: -%s requiere argumento\n' "$OPTARG" >&2; usage >&2; exit 2 ;;
            \?) printf 'error: opción inválida -%s\n' "$OPTARG" >&2; usage >&2; exit 2 ;;
        esac
    done
    shift $((OPTIND - 1))       # $@ ahora son solo posicionales
    # ... validaciones ...
}
```

Comportamiento verificado:

| Entrada | Resultado verificado |
|---|---|
| `getopts ":f:n:h" ... -f input.txt arg1` | `f=input.txt`, tras `shift $((OPTIND-1))` queda `rest=arg1` |
| `-vfx` (cluster) | procesa `v`, luego `f` **con OPTARG=x** — clustering y toma de arg en la misma tanda |
| `-n` sin valor + `:` inicial | cae en el caso `:` con `OPTARG=n` (no imprime error propio) |
| `-z` desconocido + `:` inicial | cae en `\?` con `OPTARG=z` |
| `-n` sin valor **sin** `:` inicial | getopts imprime su propio mensaje críptico y setea `opt=?` |
| Sin `local OPTIND=1`, segunda llamada a la función | **no parsea nada** (verificado abajo) |

> [!warning] Gotcha verificada: `OPTIND` NO se resetea solo entre llamadas
> La vieja leyenda dice que "OPTIND se resetea al entrar a la función" — es **falso**. Verificado: la segunda llamada sin `local OPTIND=1` arranca con el índice viejo y devuelve `rest=` vacío sin procesar opciones. **Siempre** `local OPTIND=1` (o `OPTIND=1`) al inicio del parser.

Límites de `getopts`: **no soporta opciones largas** (`--verbose`), no acepta `--file=valor`, y con `--` termina el parseo dejando el resto como posicionales (eso sí funciona).

## Parsing manual con `case`: opciones largas reales

El patrón estándar para `--largo`, `--largo=valor` y `-abc` sin depender de nada externo:

```bash
parse_args() {
    VERBOSE=0; DRY_RUN=0; ARCHIVO=""; NUMERO=10
    POSITIONAL=()

    while [[ $# -gt 0 ]]; do
        case $1 in
            -v|--verbose)   VERBOSE=1; shift ;;
            -q|--quiet)     QUIET=1; shift ;;
            -n|--number)    [[ $# -ge 2 ]] || die "-n requiere valor"; NUMERO="$2"; shift 2 ;;
            -n=*|--number=*) NUMERO="${1#*=}"; shift ;;
            -f|--file)      [[ $# -ge 2 ]] || die "--file requiere valor"; ARCHIVO="$2"; shift 2 ;;
            --file=*)       ARCHIVO="${1#*=}"; shift ;;
            -h|--help)      usage; exit 0 ;;
            --version)      echo "$VERSION"; exit 0 ;;
            --)             shift; POSITIONAL+=("$@"); break ;;   # fin de opciones
            -[!-]*)         # flags cortos cluster: -vq → v q
                            local chars="${1#-}"; shift
                            while [[ -n $chars ]]; do
                                case ${chars:0:1} in
                                    v) VERBOSE=1 ;;
                                    q) QUIET=1 ;;
                                    *) die "opción inválida -${chars:0:1}" ;;
                                esac
                                chars="${chars:1}"
                            done ;;
            -*)             die "opción desconocida: $1" ;;
            *)              POSITIONAL+=("$1"); shift ;;          # primer no-flag corta el parseo
        esac
    done
}
```

Puntos sutiles del patrón:

- `-n=*` debe listarse **antes** de `-*` (el `case` prueba en orden).
- `${1#*=}` extrae el valor de `--file=valor` sin forks.
- `--)` marca fin de opciones: todo lo siguiente es positional aunque empiece con `-` (imprescindible para pasar patrones tipo `-name` a otros comandos).
- Los args posicionales se acumulan en un **array** (`POSITIONAL+=("$1")`) — nunca en un string.

## `getopt` externo: opciones largas con reordenamiento

El binario GNU `getopt` (util-linux) parsea cortas+largas, **reordena** los posicionales al final y normaliza `--file=v` a `--file v`. Su salida es un string que hay que re-parsear con `eval set --`:

```bash
parse_getopt() {
    local tmp
    tmp=$(getopt -o vf:n:h --long verbose,file:,number:,help -n "$0" -- "$@") || return 2
    eval set -- "$tmp"                       # reemplaza $@ con el parseo normalizado
    while true; do
        case $1 in
            -v|--verbose) VERBOSE=1; shift ;;
            -f|--file)    ARCHIVO="$2"; shift 2 ;;
            -n|--number)  NUMERO="$2"; shift 2 ;;
            -h|--help)    usage; exit 0 ;;
            --)           shift; break ;;
        esac
    done
    POSITIONAL=("$@")
}
```

Verificado en este sistema (util-linux 2.42.4):

```text
getopt -o vf:n:h --long verbose,file:,number:,help -- -v --file=arch.txt arg1
→  -v --file 'arch.txt' -- 'arg1'        (rc=0, normalizado)

getopt ... -- --bogus
→  test: unrecognized option '--bogus'   (rc=1 — el || return lo captura)
```

> [!warning] Portabilidad de `getopt`
> - **Linux (util-linux)**: GNU getopt, soporta long options y reordenamiento.
> - **macOS/BSD**: getopt del sistema es el **viejo** (solo cortas, sin reordenar). Instalar `brew install gnu-getopt` y usar su path completo.
> - `eval set -- "$tmp"` es seguro **solo porque tmp lo generó getopt**; nunca evalúes salida de usuario/red.

## Comparación de estrategias

| Criterio | `getopts` builtin | Parsing manual `case` | `getopt` externo |
|---|---|---|---|
| Opciones cortas `-f` | nativas | a mano | nativas |
| Clustering `-vq` | automático | a mano | automático |
| Opciones largas `--file` | **no** | **sí** | **sí** |
| `--file=valor` | no | sí (`-n=*`) | normaliza a `--file valor` |
| Reordenar args | parcial (`--`) | el que escribas | automático |
| Error handling | `:` y `\?` | a mano | rc≠0 + mensaje |
| Dependencias | ninguna | ninguna | binario externo + path correcto en macOS |
| Seguridad del eval | sin eval | sin eval | requiere `eval set --` |

**Recomendación**: para CLIs pequeños, `getopts` (cortas) o `case` manual (si querés largas). Para CLIs grandes con users esperando GNU behavior, `getopt` con el path verificado.

## Subcomandos (estilo git)

```bash
cmd_iniciar() {
    local force=0
    while [[ $# -gt 0 ]]; do case $1 in
        -f|--force) force=1; shift ;;
        *) break ;;
    esac; done
    printf 'iniciando (force=%d, args=%s)\n' "$force" "$*"
}

main() {
    local subcmd="${1:-ayuda}"
    (( $# )) && shift || true          # el shift con $# = 0 da rc 1: blindar
    case $subcmd in
        iniciar|start)   cmd_iniciar "$@" ;;     # cada subcomando recibe SU parsing
        detener|stop)    cmd_detener "$@" ;;
        estado|status)   cmd_estado "$@" ;;
        ayuda|help|-h|--help) usage ;;
        *) printf 'comando desconocido: %s\n' "$subcmd" >&2; usage >&2; exit 2 ;;
    esac
}
```

Patrón recomendado: **el dispatcher delega `"$@"` y cada subcomando parsea sus propias flags** — evita un `case` gigante y permite flags exclusivas por subcomando.

## Validación de argumentos

```bash
validar_entero() { [[ "$1" =~ ^-?[0-9]+$ ]] || { printf 'error: %s debe ser entero, got: %s\n' "$2" "$1" >&2; return 1; }; }
validar_rango()  { (( $1 >= $3 && $1 <= $4 )) || { printf 'error: %s fuera de rango [%s..%s]\n' "$2" "$3" "$4" >&2; return 1; }; }
validar_archivo(){ [[ -f "$1" && -r "$1" ]] || { printf 'error: %s no legible: %s\n' "$2" "$1" >&2; return 1; }; }
validar_dir()    { [[ -d "$1" ]] || { printf 'error: %s no es directorio: %s\n' "$2" "$1" >&2; return 1; }; }
validar_en()     { local v; for v in "${@:3}"; do [[ "$1" == "$v" ]] && return 0; done; printf 'error: %s inválido: %s (opciones: %s)\n' "$3" "$1" "${*:3}" >&2; return 1; }

# Uso en parse_args, con exit codes semánticos:
validar_entero "$NUMERO" "iteraciones" || exit 2
validar_rango "$NUMERO" "iteraciones" 1 100 || exit 2
validar_archivo "$ARCHIVO" "entrada" || exit 66      # EX_NOINPUT de sysexits
```

Validaciones de **dependencias** (al arrancar, no en caliente):

```bash
for cmd in jq curl; do command -v "$cmd" >/dev/null || die "requiere: $cmd"; done
[[ -n "${API_TOKEN:-}" ]] || die "variable requerida: API_TOKEN"
```

## Convención de exit codes para CLIs

| Código | Uso |
|---|---|
| `0` | Éxito |
| `1` | Error genérico (fallo de negocio: la descarga falló, el archivo está corrupto) |
| `2` | **Uso incorrecto** (opciones inválidas, faltan args) — convención GNU |
| `3` | Recurso no disponible / permisos (convención propia, documentarla) |
| `126` / `127` | No ejecutable / comando no encontrado (los asigna el shell) |
| `130` | Interrumpido por Ctrl+C (128+2) |
| `64-78` | sysexits (`EX_USAGE=64`, `EX_DATAERR=65`, `EX_NOINPUT=66`, `EX_UNAVAILABLE=67`, `EX_IOERR=74`, `EX_TEMPFAIL=75`) — para scripts que otros scripts consumen |

La regla de oro para **componibilidad**: stdout para los **datos**, stderr para los **mensajes**. `mi_script fichero | jq .` debe funcionar aunque el script esté gritando warnings por stderr.

## Esqueleto completo de CLI (plantilla real)

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly VERSION="1.0.0"
SCRIPT_NAME="$(basename "$0")"

usage() {
    cat <<EOF
Uso: $SCRIPT_NAME [OPCIONES] <entrada>

Descripción: qué hace (una línea).

Opciones:
  -h, --help        Ayuda
  -V, --version     Versión
  -v, --verbose     Logging debug
  -o, --output FILE Salida (default: stdout)
  -n, --retries N   Reintentos (default: 3)

Códigos de salida: 0 ok · 1 error · 2 uso inválido
EOF
}

die()  { printf '[ERROR] %s\n' "$*" >&2; exit 1; }
usage_error() { printf '[ERROR] %s\n\n' "$*" >&2; usage >&2; exit 2; }

main() {
    local verbose=0 output="" retries=3 input=""

    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)   usage; exit 0 ;;
            -V|--version) echo "$VERSION"; exit 0 ;;
            -v|--verbose) verbose=1; shift ;;
            -o|--output) [[ $# -ge 2 ]] || usage_error "-o requiere archivo"; output="$2"; shift 2 ;;
            -n|--retries) [[ $# -ge 2 ]] || usage_error "-n requiere número"; retries="$2"; shift 2 ;;
            --) shift; break ;;
            -*) usage_error "opción desconocida: $1" ;;
            *) break ;;
        esac
    done

    input="${1:-}"
    [[ -n "$input" ]]          || usage_error "falta <entrada>"
    [[ -f "$input" ]]          || usage_error "no existe: $input"
    [[ "$retries" =~ ^[0-9]+$ ]] || usage_error "--retries debe ser entero: $retries"
    shift || true
    (( $# == 0 ))              || usage_error "argumentos de más: $*"

    # lógica...
    printf 'procesando %s (retries=%d)\n' "$input" "$retries" >&2
}

main "$@"
```

## Tab completion (bash-completion)

```bash
_miscript_completions() {
    local cur="${COMP_WORDS[COMP_CWORD]}" prev="${COMP_WORDS[COMP_CWORD-1]}"
    case $prev in
        -o|--file)  COMPREPLY=( $(compgen -f -- "$cur") ); return ;;   # archivos
        -n)         COMPREPLY=( $(compgen -W "1 3 5 10" -- "$cur") ); return ;;
    esac
    if [[ $cur == -* ]]; then
        COMPREPLY=( $(compgen -W "-h --help -v --verbose -o --output -n --retries" -- "$cur") )
    else
        COMPREPLY=( $(compgen -f -- "$cur") )
    fi
}
complete -F _miscript_completions miscript
```

## Precedencia de configuración (defaults → config → env → CLI)

```bash
# 1. defaults del script
PORT=8080; VERBOSE=0
# 2. archivo de config (source = código ejecutable: SOLO de fuentes confiables)
[[ -f "${XDG_CONFIG_HOME:-$HOME/.config}/miscript.conf" ]] && source "${XDG_CONFIG_HOME:-$HOME/.config}/miscript.conf"
# 3. variables de entorno
PORT="${MISCRIPT_PORT:-$PORT}"
# 4. CLI (máxima precedencia, parse_args al final)
parse_args "$@"
```

## Errores comunes

1. **Olvidar `local OPTIND=1`** en parsers con `getopts` reutilizados — la segunda llamada no parsea (verificado).
2. **`shift` sin argumentos con `set -e` cuando `$#=0`** — rc=1 mata el script. Blindar.
3. **Leer `$2` sin verificar `$#`** — con `set -u` muere; sin `-u` procesa string vacío silenciosamente. Siempre `[[ $# -ge 2 ]] || usage_error`.
4. **Mezclar datos y mensajes en stdout** — rompe la composición con pipes. Mensajes a stderr.
5. **`eval set --` con salida no generada por getopt** — inyección de comandos. Solo el binario.
6. **No documentar los exit codes en `usage()`** — los consumidores scripteados no pueden reaccionar.
7. **Acumular posicionales en string** (`args="$args $1"`) — pierde espacios. Array + `"${arr[@]}"`.
8. **`-n=*` listado después de `-*`** en el case — nunca matchea; el orden importa.

## Cheatsheet

| Elemento | Forma | Nota |
|---|---|---|
| Cantidad | `$#` | sin $0 |
| Argumentos | `"$@"` | siempre comillas |
| Último / desde N | `${@: -1}` / `${@:2}` | espacio antes de `-` |
| Consumir | `shift` / `shift 2` | rc=1 si $# < n |
| getopts | `while getopts ":vf:h" o; do ... done` | `OPTARG`, `OPTIND` |
| Falta arg | `:)` con optstring inicial `:` | |
| Opción inválida | `\?)` con `:` inicial | |
| Posicionales tras getopts | `shift $((OPTIND-1))` | |
| Opción larga | `-n=*\|--n=*) v="${1#*=}"` | case antes de `-*` |
| Fin de opciones | `--)` | todo lo resto es data |
| getopt GNU | `eval set -- "$(getopt ... -- "$@")"` | Linux; macOS: brew gnu-getopt |
| Error de uso | salir con `2` | mensaje a stderr |
| Ayuda | `usage()` + `-h` siempre | antes de cualquier validación |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — `$@`/quoting y la plantilla mínima
- [[Bash Scripting - Funciones y Scope]] — parsers como funciones y `local OPTIND`
- [[Bash Scripting - Control de Flujo]] — `case` como base del dispatcher
- [[Bash Scripting - Entrada-Salida y Redirección]] — la regla stdout-datos/stderr-mensajes
- [[Bash Scripting - Patrones Comunes y Plantillas]] — esqueleto completo con logging y traps
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — testing de CLIs con bats
