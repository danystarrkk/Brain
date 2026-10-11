---
title: "Bash Scripting - Funciones y Scope"
tags:
  - lang/bash
  - concept/functions
  - concept/scope
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Bash Scripting - Funciones y Scope

> **Objetivo**: Escribir código modular, reutilizable y con variables bien aisladas.

---

## 1. Definición Básica

```bash
# Forma 1: keyword function (preferida, más legible)
function nombre_funcion() {
    # cuerpo
}

# Forma 2: POSIX (portable a sh)
nombre_funcion() {
    # cuerpo
}

# Llamada
nombre_funcion arg1 arg2
```

---

## 2. Parámetros y Argumentos

### 2.1 Parámetros Posicionales

```bash
saludar() {
    local nombre="${1:-Mundo}"  # Default si no se pasa
    local saludo="${2:-Hola}"
    echo "$saludo, $nombre!"
}

saludar "Juan" "Buenos días"  # Buenos días, Juan!
saludar                       # Hola, Mundo!
```

### 2.2 Acceso a Argumentos

```bash
procesar() {
    echo "Total args: $#"           # Número de argumentos
    echo "Todos: $@"                # Array seguro: "arg1" "arg2" ...
    echo "Como string: $*"           # Un string: "arg1 arg2 ..."
    echo "Primero: $1"               # Primer argumento
    echo "Último: ${@: -1}"         # Último argumento (Bash 3.0+)
    echo "Desde 2do: ${@:2}"        # Todos menos el primero
    echo "2do y 3ro: ${@:2:2}"      # Slice: posición 2, longitud 2
}
```

### 2.3 `shift` - Consumir Argumentos

```bash
# Patrón común: procesar opciones y dejar argumentos restantes
procesar_opciones() {
    local verbose=false
    local output=""
    
    while [[ $# -gt 0 ]]; do
        case $1 in
            -v|--verbose) verbose=true; shift ;;
            -o|--output) output="$2"; shift 2 ;;
            --) shift; break ;;
            -*) echo "Opción desconocida: $1"; return 1 ;;
            *) break ;;  # Argumentos posicionales restantes
        esac
    done
    
    # $@ ahora contiene solo argumentos posicionales
    echo "Verbose: $verbose, Output: $output, Args: $@"
}
```

---

## 3. Scope de Variables: `local` vs Global

### 3.1 Reglas de Oro

```bash
variable_global="soy global"

funcion_scope() {
    local variable_local="soy local"   # Solo existe aquí
    variable_global="modificada"       # Modifica la global
    declare -g nueva_global="creada aquí"  # Crea global explícitamente
}

funcion_scope
echo "$variable_local"   # Vacío (no existe)
echo "$variable_global"  # "modificada"
echo "$nueva_global"     # "creada aquí"
```

### 3.2 `local` en Bucles y Condicionales

```bash
# local SOLO funciona dentro de funciones
# En bucles/condicionales NO crea scope

for i in {1..3}; do
    local x=$i  # ERROR: local solo en funciones
done

# Correcto: usar subshell para aislar
(
    local x=10
    echo "Subshell: $x"
)
echo "Padre: $x"  # Vacío o valor previo
```

### 3.3 Variables Especiales en Funciones

```bash
demo() {
    echo "FUNCNAME: ${FUNCNAME[0]}"      # Nombre función actual
    echo "FUNCNAME stack: ${FUNCNAME[*]}" # Stack de llamadas
    echo "BASH_SOURCE: ${BASH_SOURCE[0]}" # Archivo donde se define
    echo "LINENO: $LINENO"                # Línea actual en script
    echo "BASH_LINENO: ${BASH_LINENO[0]}" # Línea de llamada
}

demo
```

---

## 4. Valores de Retorno

### 4.1 Exit Code (0-255)

```bash
validar_archivo() {
    local archivo="$1"
    [[ -f "$archivo" && -r "$archivo" ]] || return 1
    return 0
}

if validar_archivo "data.txt"; then
    echo "OK"
else
    echo "Error: $?"  # $? captura exit code
fi
```

### 4.2 Retornar Datos (Stdout)

```bash
# PATRÓN RECOMENDADO: stdout para datos, stderr para logs/errores
obtener_fecha() {
    date '+%Y-%m-%d'
}

fecha=$(obtener_fecha)  # Captura stdout
echo "Hoy es $fecha"
```

### 4.3 Retornar Múltiples Valores

```bash
# Opción 1: Array global (nombre acordado)
resultado=()
calcular() {
    resultado=( "$1" "$2" "$(( $1 + $2 ))" )
}
calcular 5 3
echo "Suma: ${resultado[2]}"

# Opción 2: Output estructurado (JSON, CSV, key=value)
calcular_kv() {
    local a=$1 b=$2
    printf 'suma=%d producto=%d\n' "$((a+b))" "$((a*b))"
}

eval "$(calcular_kv 5 3)"
echo "Suma: $suma, Producto: $producto"

# Opción 3: Referencia por nombre (Bash 4.3+ nameref)
calcular_ref() {
    local -n ref=$1  # ref es alias de la variable pasada
    ref=$(( $2 + $3 ))
}
declare -a mi_resultado
calcular_ref mi_resultado 5 3
echo "${mi_resultado}"  # 8
```

---

## 5. Funciones Avanzadas

### 5.1 Funciones Recursivas

```bash
factorial() {
    local n=$1
    if (( n <= 1 )); then
        echo 1
    else
        echo $(( n * $(factorial $((n-1))) ))
    fi
}

# Con tail recursion optimization manual
factorial_iter() {
    local n=$1 acc=1
    while (( n > 1 )); do
        acc=$(( acc * n ))
        ((n--))
    done
    echo $acc
}
```

### 5.2 Funciones como Argumentos (Callbacks)

```bash
# Filtrar array con función predicado
filtrar() {
    local -n arr=$1
    local pred_func=$2
    local -n out=$3
    
    out=()
    for elem in "${arr[@]}"; do
        if "$pred_func" "$elem"; then
            out+=("$elem")
        fi
    done
}

es_par() { (( $1 % 2 == 0 )); }

numeros=(1 2 3 4 5 6 7 8 9 10)
declare -a pares
filtrar numeros es_par pares
echo "${pares[@]}"  # 2 4 6 8 10
```

### 5.3 Decoradores / Wrappers

```bash
# Medir tiempo de ejecución
timeit() {
    local inicio=$(date +%s.%N)
    "$@"
    local fin=$(date +%s.%N)
    printf "Tiempo: %.3fs\n" "$(echo "$fin - $inicio" | bc -l)"
}

timeit sleep 1
timeit encontrar_archivos /home/usuario

# Logging automático
log_call() {
    log_debug "→ ${FUNCNAME[1]}($*)"
    "$@"
    local rc=$?
    log_debug "← ${FUNCNAME[1]} returned $rc"
    return $rc
}
```

---

## 6. Patrones de Diseño Comunes

### 6.1 Función `main` + Punto de Entrada

```bash
main() {
    local args=("$@")
    parse_args "${args[@]}"
    validar_entorno
    ejecutar_logica
}

# Solo ejecutar main si no se está sourceando
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

### 6.2 Biblioteca de Funciones (Sourceable)

```bash
# lib/strings.sh
# -*- mode: sh -*-

# Evitar carga múltiple
[[ -n "${_LIB_STRINGS_LOADED:-}" ]] && return 0
readonly _LIB_STRINGS_LOADED=1

trim() {
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"  # Leading
    var="${var%"${var##*[![:space:]]}"}"  # Trailing
    echo -n "$var"
}

to_lower() { echo "${1,,}"; }
to_upper() { echo "${1^^}"; }
```

```bash
# Uso en otro script
source "${SCRIPT_DIR}/lib/strings.sh"
nombre="  Juan  "
echo "|$(trim "$nombre")|"  # |Juan|
```

---

## 7. Debugging de Funciones

```bash
# Trace de llamadas
set -o functrace
trap 'echo "CALL: ${FUNCNAME[*]}" >&2' DEBUG

# O con PS4 personalizado
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x  # Activar trace
```

---

## 8. Buenas Prácticas

| Práctica | Beneficio |
|----------|-----------|
| `local` por defecto | Evita fugas de variables, bugs sutiles |
| Nombres verbosos: `calcular_total_factura` | Auto-documentado |
| Una función = una responsabilidad | Reutilizable, testeable |
| `return` explícito al final | Claridad de flow control |
| Documentar con comentarios `#` | Mantenibilidad |
| `declare -f nombre` para inspeccionar | Debugging rápido |

---

## 9. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Plantilla base con funciones log_*
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - Bucles y condicionales
- [Argumentos y Opciones](./Bash%20Scripting%20-%20Argumentos%20y%20Opciones.md) - Parsing CLI con funciones
- [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md)

---

*Etiquetas: #bash #funciones #scope #local #global #return #callback #recursion #librerias*