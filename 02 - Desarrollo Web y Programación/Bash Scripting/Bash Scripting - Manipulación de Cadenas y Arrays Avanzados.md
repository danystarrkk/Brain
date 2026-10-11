---
title: Bash Scripting - Manipulación de Cadenas y Arrays Avanzados
tags:
  - lang/bash
  - concept/string-manipulation
  - concept/arrays
  - concept/parameter-expansion
  - level/avanzado
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Manipulación de Cadenas y Arrays Avanzados

> **Objetivo**: Técnicas avanzadas para procesamiento de texto, parsing, y estructuras de datos en Bash puro.

---

## 1. Expansión de Parámetros (Parameter Expansion) - Completa

### 1.1 Longitud y Substrings

```bash
var="archivo.tar.gz"

${#var}              # 14 (longitud)
${var:offset}        # Desde offset al final
${var:offset:len}    # Substring: posición offset, longitud len
${var: -len}         # Últimos len caracteres (espacio antes de -)
${var:offset:-len}   # Desde offset, excluyendo últimos len
```

### 1.2 Eliminación de Prefijos/Sufijos (Glob Patterns)

```bash
${var#patrón}    # Eliminar prefijo MÁS CORTO que coincida
${var##patrón}   # Eliminar prefijo MÁS LARGO que coincida
${var%patrón}    # Eliminar sufijo MÁS CORTO
${var%%patrón}   # Eliminar sufijo MÁS LARGO

# Ejemplos
path="/home/usuario/documentos/archivo.txt"
${path#*/}       # home/usuario/documentos/archivo.txt
${path##*/}      # archivo.txt (basename)
${path%/*}       # /home/usuario/documentos (dirname)
${path%%/*}      # (vacío, elimina todo hasta primer /)
```

### 1.3 Sustitución

```bash
${var/patrón/reemplazo}      # Primera coincidencia
${var//patrón/reemplazo}     # Todas las coincidencias
${var/#patrón/reemplazo}     # Solo si al INICIO
${var/%patrón/reemplazo}     # Solo si al FINAL

# Con pattern matching extendido (shopt -s extglob)
${var//+([0-9])/NUM}   # Reemplazar secuencias de dígitos por "NUM"
```

### 1.4 Case Modification (Bash 4+)

```bash
${var^}      # Primera letra mayúscula
${var^^}     # Todo mayúsculas
${var,}      # Primera letra minúscula
${var,,}     # Todo minúsculas
${var~}      # Invertir primera letra
${var~~}     # Invertir todas

# Patrones
${var^^[aeiou]}  # Solo vocales a mayúsculas
```

### 1.5 Valores por Defecto y Asignación

```bash
${var:-default}   # Usar default si unset/vacía (NO asigna)
${var:=default}   # Asignar default si unset/vacía
${var:+alternativo} # Usar alternativo SI var tiene valor
${var:?mensaje}   # Error con mensaje si unset/vacía

# Diferencia unset vs vacía:
unset var;  echo ${var:-x}  # x
var=;       echo ${var:-x}  # x
unset var;  echo ${var-x}   # x (solo unset)
var=;       echo ${var-x}   # (vacío, no asigna)
```

### 1.6 Expansión de Índices (Arrays)

```bash
arr=(a b c d e)

${arr[@]}        # Todos elementos (palabras separadas)
${arr[*]}        # Todos elementos (un string, IFS)
${#arr[@]}       # Count
${!arr[@]}       # Índices (0 1 2 3 4)
${arr[@]:1:3}    # Slice: desde índice 1, 3 elementos (b c d)
${arr[@]: -2}    # Últimos 2 (d e)
```

---

## 2. Arrays Avanzados

### 2.1 Arrays Asociativos (Hash Maps) - Bash 4+

```bash
declare -A mapa=(
    [clave1]=valor1
    [clave2]=valor2
    ["clave con espacios"]="valor con espacios"
)

# Operaciones
mapa[nueva]=nuevo_valor
unset 'mapa[clave1]'
echo "${mapa[clave2]}"
echo "${!mapa[@]}"      # Todas las claves
echo "${mapa[@]}"       # Todos los valores
echo "${#mapa[@]}"      # Count

# Iterar
for clave in "${!mapa[@]}"; do
    echo "$clave -> ${mapa[$clave]}"
done

# Verificar existencia
[[ -v mapa[clave] ]] && echo "Existe"
```

### 2.2 Arrays Multidimensionales (Simulados)

```bash
# Opción 1: Índices compuestos
declare -A matriz
matriz[0,0]=1; matriz[0,1]=2
matriz[1,0]=3; matriz[1,1]=4
echo "${matriz[1,0]}"  # 3

# Opción 2: Array de arrays (namerefs, Bash 4.3+)
fila1=(1 2 3)
fila2=(4 5 6)
declare -n ref_fila
for ref_fila in fila1 fila2; do
    echo "${ref_fila[@]}"
done

# Opción 3: Delimitador en string plano
matriz_flat=( "1|2|3" "4|5|6" )
IFS='|' read -r a b c <<< "${matriz_flat[1]}"  # 4 5 6
```

### 2.3 Operaciones de Array Funcionales

```bash
# Map (transformar cada elemento)
map_array() {
    local -n arr=$1
    local func=$2
    local -n out=$3
    out=()
    for elem in "${arr[@]}"; do
        out+=("$("$func" "$elem")")
    done
}

# Filter (filtrar con predicado)
filter_array() {
    local -n arr=$1
    local pred=$2
    local -n out=$3
    out=()
    for elem in "${arr[@]}"; do
        if "$pred" "$elem"; then
            out+=("$elem")
        fi
    done
}

# Reduce (reducir a valor único)
reduce_array() {
    local -n arr=$1
    local func=$2
    local init=$3
    local acc=$init
    for elem in "${arr[@]}"; do
        acc=$("$func" "$acc" "$elem")
    done
    echo "$acc"
}

# Uso
nums=(1 2 3 4 5)
doble() { echo $(( $1 * 2 )); }
es_par() { (( $1 % 2 == 0 )); }
sumar() { echo $(( $1 + $2 )); }

declare -a doblados pares
map_array nums doble doblados      # 2 4 6 8 10
filter_array nums es_par pares     # 2 4
total=$(reduce_array nums sumar 0) # 15
```

### 2.4 `mapfile` / `readarray` - Lectura Eficiente

```bash
# Leer archivo a array (Bash 4+)
mapfile -t lineas < archivo.txt
# -t: strip newline
# -n N: leer N líneas
# -s N: saltar N líneas
# -C callback -c N: callback cada N líneas

# Desde comando
mapfile -t salida < <(comando)

# Callback (Bash 4.4+)
procesar() { echo "LINEA: $1"; }
mapfile -t -C procesar -c 1 < archivo.txt
```

---

## 3. Manipulación de Cadenas (Strings)

### 3.1 Concatenación Eficiente

```bash
# Evitar += en bucles (O(n²))
# Malo:
resultado=""
for i in {1..1000}; do resultado+="$i "; done

# Bueno: array + join
partes=()
for i in {1..1000}; do partes+=("$i"); done
resultado="${partes[*]}"  # O con IFS controlado
```

### 3.2 Dividir Strings (Split)

```bash
# IFS + read (estándar)
IFS=',' read -ra campos <<< "a,b,c,d"
# campos=(a b c d)

# Múltiples delimitadores
IFS=$',;' read -ra campos <<< "a,b;c,d"

# En array directamente (Bash 4.4+)
readarray -td ',' campos <<< "a,b,c,d,"  # -d delim, incluye vacío final

# Parameter expansion (solo primer/último)
${var%%,*}   # Antes de primera coma
${var##*,}   # Después de última coma
```

### 3.3 Unir Arrays (Join)

```bash
# Con IFS
IFS=',' joined="${array[*]}"
IFS=$' \t\n'  # Restaurar

# Función join robusta
join_by() {
    local IFS="$1"; shift
    echo "$*"
}
join_by ',' "${array[@]}"  # a,b,c
```

### 3.4 Búsqueda y Reemplazo Avanzado

```bash
# Verificar subcadena
[[ "$string" == *"sub"* ]] && echo "Contiene 'sub'"

# Posición de subcadena (Bash no nativo, usar expr o awk)
# Función pura Bash:
str_index() {
    local haystack="$1" needle="$2"
    local temp="${haystack%%$needle*}"
    [[ "$temp" == "$haystack" ]] && echo -1 || echo ${#temp}
}

# Contar ocurrencias
count_occurrences() {
    local str="$1" substr="$2"
    local count=0
    while [[ "$str" == *"$substr"* ]]; do
        str="${str#*$substr}"
        ((count++))
    done
    echo $count
}
```

### 3.5 Padding y Alineación

```bash
# Padding izquierdo (zeros)
printf "%05d\n" 42      # 00042
printf "%0*d\n" 5 42    # 00042 (ancho variable)

# Padding derecho
printf "%-10s\n" "hola" # "hola      "

# Centrar (aprox)
center() {
    local str="$1" width="$2"
    local pad=$(( (width - ${#str}) / 2 ))
    printf "%*s%s%*s\n" $pad "" "$str" $pad ""
}
```

### 3.6 Trimming (Whitespace)

```bash
# Trim leading/trailing (Bash parameter expansion)
trim() {
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"  # Leading
    var="${var%"${var##*[![:space:]]}"}"  # Trailing
    echo -n "$var"
}

# Solo leading
ltrim() { local v="$*"; echo -n "${v#"${v%%[![:space:]]*}"}"; }
# Solo trailing
rtrim() { local v="$*"; echo -n "${v%"${v##*[![:space:]]}"}"; }

# Con extglob (más legible)
shopt -s extglob
trim_ext() { local v="$*"; v="${v##*([[:space:]])}"; v="${v%%*([[:space:]])}"; echo -n "$v"; }
```

---

## 4. Parsing Estructurado

### 4.1 CSV Simple (sin comillas)

```bash
parse_csv() {
    local line="$1" IFS=","
    read -ra fields <<< "$line"
    # Manejar campos vacíos: read -ra no preserva vacíos al final
    # Solución: añadir delimitador ficticio
    read -ra fields <<< "$line,"
    fields=("${fields[@]%,}")  # Quitar último ficticio
}
```

### 4.2 Key=Value Pairs

```bash
# Parsear "key1=val1 key2=val2"
parse_kv() {
    local str="$1"
    local -n out=$2
    out=()
    local kv
    for kv in $str; do  # Word splitting intencional
        [[ $kv == *=* ]] || continue
        out["${kv%%=*}"]="${kv#*=}"
    done
}

declare -A config
parse_kv "host=localhost port=8080 debug=true" config
```

### 4.3 JSON Simple (sin jq)

```bash
# Extraer valor simple (clave en nivel superior)
json_get() {
    local json="$1" key="$2"
    # Muy básico, solo para JSON simple sin anidación
    local regex="\"$key\"[[:space:]]*:[[:space:]]*\"([^\"]*)\""
    [[ $json =~ $regex ]] && echo "${BASH_REMATCH[1]}"
}

json='{"nombre":"Juan","edad":30,"activo":true}'
json_get "$json" "nombre"  # Juan
```

---

## 5. Expresiones Regulares en Bash

```bash
# Operador =~ en [[ ]]
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Email válido"
fi

# Capturar grupos (BASH_REMATCH)
if [[ "archivo_123.txt" =~ ^(.*)_([0-9]+)\.(.*)$ ]]; then
    echo "Base: ${BASH_REMATCH[1]}"      # archivo
    echo "Num: ${BASH_REMATCH[2]}"       # 123
    echo "Ext: ${BASH_REMATCH[3]}"       # txt
fi

# Reemplazo con regex (no nativo, usar sed/perl o función)
# Función usando parameter expansion iterativo
regex_replace() {
    local str="$1" pattern="$2" repl="$3"
    while [[ "$str" =~ $pattern ]]; do
        str="${str/${BASH_REMATCH[0]}/$repl}"
    done
    echo "$str"
}
```

---

## 6. Herramientas Externas vs Bash Puro

| Tarea | Bash Puro | Herramienta Externa | Cuándo Usar Cada |
|-------|-----------|---------------------|------------------|
| Split simple | `read -ra` | `cut`, `awk` | Bash: simple, pocos datos |
| Join | `"${arr[*]}"` | `paste` | Bash: arrays en memoria |
| Regex extract | `[[ =~ ]]` | `grep -oP`, `sed` | Bash: 1-2 matches; ext: complejos |
| JSON | Muy limitado | `jq` | **Siempre jq para JSON real** |
| CSV robusto | Difícil | `csvkit`, `awk` | **csvkit/mlr para CSV real** |
| Matemáticas complejas | `(( ))`, `bc` | `bc`, `awk`, `python` | Bash: enteros simples |
| Procesamiento masivo | Lento | `awk`, `sed`, `perl` | >1000 líneas: herramienta externa |

---

## 7. Patrones de Diseño de Datos

### 7.1 Registro (Struct-like)

```bash
# Usar array asociativo por registro
declare -A usuario=(
    [id]=1001
    [nombre]="Juan Pérez"
    [email]="juan@ejemplo.com"
    [activo]=true
    [roles]="admin,usuario"
)

# Array de registros
declare -a usuarios
usuarios+=("$(declare -p usuario)")  # Serializar (eval para restaurar)

# Mejor: índices compuestos
declare -A usuarios
usuarios[1001,nombre]="Juan"
usuarios[1001,email]="juan@ejemplo.com"
```

### 7.2 Cola (Queue) / Pila (Stack)

```bash
# Queue (FIFO) - array + índices
queue=()
queue_push() { queue+=("$1"); }
queue_pop() { local val="${queue[0]}"; queue=("${queue[@]:1}"); echo "$val"; }
queue_peek() { echo "${queue[0]}"; }

# Stack (LIFO)
stack=()
stack_push() { stack+=("$1"); }
stack_pop() { local idx=$((${#stack[@]}-1)); local val="${stack[idx]}"; unset 'stack[idx]'; echo "$val"; }
```

---

## 8. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Arrays básicos, parameter expansion intro
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - Iteración sobre arrays
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md) - Namerefs para arrays
- [Aritmética y Números Aleatorios](./Bash%20Scripting%20-%20Aritm%C3%A9tica%20y%20N%C3%BAmeros%20Aleatorios.md)

---

*Etiquetas: #bash #strings #arrays #parameter-expansion #asociativos #mapfile #regex #split #join #map-filter-reduce*