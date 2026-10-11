---
title: "Bash Scripting - Aritmética y Números Aleatorios"
tags:
  - lang/bash
  - concept/arithmetic
  - concept/random
  - concept/math
  - level/basico
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Aritmética y Números Aleatorios

> **Objetivo**: Realizar cálculos numéricos, generar aleatoriedad, y manejar matemáticas en Bash.

---

## 1. Aritmética Entera Nativa

### 1.1 `(( ))` - Comando Aritmético (Recomendado)

```bash
# Asignación y operaciones
(( a = 5 + 3 ))       # a=8
(( a += 2 ))          # a=10 (incremento)
(( a -= 3 ))          # a=7
(( a *= 4 ))          # a=28
(( a /= 3 ))          # a=9 (división entera)
(( a %= 5 ))          # a=4 (módulo)
(( a **= 2 ))         # a=16 (potencia, Bash 3.1+)

# Incremento/Decremento
(( a++ ))             # Post-incremento (usa valor, luego incrementa)
(( ++a ))             # Pre-incremento (incrementa, luego usa valor)
(( a-- ))             # Post-decremento
(( --a ))             # Pre-decremento

# Comparación (retorna exit code 0=true, 1=false)
(( a < b ))           # Menor
(( a > b ))           # Mayor
(( a <= b ))          # Menor o igual
(( a >= b ))          # Mayor o igual
(( a == b ))          # Igual
(( a != b ))          # Distinto

# Uso en condicionales
if (( contador > 10 )); then
    echo "Muchos"
fi

# Bucle estilo C
for (( i=0; i<10; i++ )); do
    echo $i
done
```

### 1.2 `let` - Builtin Alternativo

```bash
let a=5+3
let "a = 5 + 3"       # Con espacios (comillas obligatorias)
let a+=2 a*=3         # Múltiples operaciones
```

### 1.3 `$(( ))` - Expansión Aritmética (En Strings)

```bash
# En asignación
a=$(( 5 + 3 ))

# En echo/command substitution
echo "Resultado: $(( 2 * 3 ))"  # 6

# En arrays
indices=( $((i*2)) $((i*2+1)) )

# En parameter expansion
echo "${array[$((i+1))]}"
```

---

## 2. Operadores y Precedencia

| Precedencia | Operadores | Asociatividad |
|-------------|------------|---------------|
| 1 (mayor) | `++`, `--` (pre/post) | Derecha |
| 2 | `+`, `-` (unario), `!`, `~` | Derecha |
| 3 | `**` | Derecha |
| 4 | `*`, `/`, `%` | Izquierda |
| 5 | `+`, `-` | Izquierda |
| 6 | `<<`, `>>` | Izquierda |
| 7 | `<`, `<=`, `>`, `>=` | Izquierda |
| 8 | `==`, `!=` | Izquierda |
| 9 | `&` (bitwise AND) | Izquierda |
| 10 | `^` (bitwise XOR) | Izquierda |
| 11 | `|` (bitwise OR) | Izquierda |
| 12 | `&&` (lógico AND) | Izquierda |
| 13 | `\|\|` (lógico OR) | Izquierda |
| 14 | `? :` (ternario) | Derecha |
| 15 (menor) | `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `<<=`, `>>=`, `&=`, `^=`, `|=` | Derecha |

```bash
# Ejemplos de precedencia
(( a = 2 + 3 * 4 ))     # a = 14 (no 20)
(( a = (2 + 3) * 4 ))   # a = 20

# Ternario
(( max = a > b ? a : b ))

# Bitwise
(( flags = 1 | 2 | 4 ))     # 7 (bits 0,1,2)
(( has_flag = flags & 2 ))  # 2 (tiene flag 2)
(( toggle = flags ^ 1 ))    # Invierte bit 0
```

---

## 3. Bases Numéricas

```bash
# Notación: base#número
(( dec = 10#10 ))      # Decimal: 10
(( oct = 8#12 ))       # Octal: 10
(( hex = 16#A ))       # Hex: 10
(( bin = 2#1010 ))     # Binario: 10

# Cualquier base 2-64
(( base36 = 36#ZZ ))   # 1295

# Conversión a string en base
printf "%d\n" 255      # 255 (decimal)
printf "%x\n" 255      # ff (hex minúscula)
printf "%X\n" 255      # FF (hex mayúscula)
printf "%o\n" 255      # 377 (octal)
printf "%b\n" 255      # Binario (no estándar, usar función)

# Función a binario
to_bin() {
    local n=$1
    (( n == 0 )) && echo 0 && return
    local bits=""
    while (( n > 0 )); do
        bits="$(( n & 1 ))$bits"
        (( n >>= 1 ))
    done
    echo "$bits"
}
```

---

## 4. Matemáticas Avanzadas (Floating Point)

> **Bash solo hace enteros. Para decimales: `bc`, `awk`, `dc`, o Python.**

### 4.1 `bc` - Calculadora de Precisión Arbitraria

```bash
# Básico
echo "3.14 * 2" | bc           # 6.28
echo "scale=10; 1/3" | bc      # .3333333333

# Variables
radio=5
area=$(echo "scale=4; 3.14159 * $radio * $radio" | bc)
echo "Área: $area"

# Funciones matemáticas (librería -l)
echo "s(3.14159/2)" | bc -l    # 1 (seno)
echo "c(0)" | bc -l            # 1 (coseno)
echo "a(1)*4" | bc -l          # 3.14159... (pi = 4*arctan(1))
echo "e(1)" | bc -l            # 2.71828... (e)
echo "l(10)" | bc -l           # 2.30258... (ln)
echo "10^2" | bc -l            # 100 (potencia)

# Función wrapper
calc() {
    echo "scale=10; $*" | bc -l
}
calc "sqrt(16)"     # 4
calc "2^10"         # 1024
```

### 4.2 `awk` - Matemáticas Rápidas

```bash
# Más rápido que bc para operaciones simples
awk "BEGIN { print 3.14 * 2 }"           # 6.28
awk "BEGIN { print sqrt(16) }"           # 4
awk "BEGIN { print log(10) }"            # 2.30259 (ln)
awk "BEGIN { print exp(1) }"             # 2.71828 (e)
awk "BEGIN { print sin(3.14159/2) }"     # 1

# Con variables
radio=5
awk -v r="$radio" 'BEGIN { print 3.14159 * r * r }'
```

### 4.3 Python One-Liners (Si Disponible)

```bash
python3 -c "import math; print(math.pi * 5**2)"
python3 -c "import random; print(random.randint(1,100))"
python3 -c "import secrets; print(secrets.token_hex(16))"
```

---

## 5. Números Aleatorios

### 5.1 `$RANDOM` - Nativo Bash (15 bits, 0-32767)

```bash
# Entero 0-32767
echo $RANDOM

# Rango 0-N
echo $(( RANDOM % (N+1) ))

# Rango M-N (inclusive)
M=10; N=20
echo $(( RANDOM % (N - M + 1) + M ))

# Múltiples
for i in {1..5}; do echo $RANDOM; done
```

### 5.2 `$SRANDOM` - Bash 5.1+ (32 bits, mejor entropía)

```bash
# Requiere Bash 5.1+
echo $SRANDOM        # 0-4294967295
echo $(( SRANDOM % 100 ))
```

### 5.3 `/dev/urandom` / `/dev/random` - Entropía del Sistema

```bash
# Bytes aleatorios
head -c 4 /dev/urandom | od -An -tu4  # uint32

# Entero rango M-N
rand_range() {
    local min=$1 max=$2
    local range=$((max - min + 1))
    local bytes=4
    local max_val=$((256**bytes))
    local limit=$((max_val - (max_val % range)))
    local rand
    while :; do
        rand=$(head -c $bytes /dev/urandom | od -An -tu4 | tr -d ' ')
        (( rand < limit )) && break
    done
    echo $(( min + rand % range ))
}

# String aleatorio (base64)
rand_string() {
    local len=${1:-16}
    head -c $(( len * 3 / 4 + 1 )) /dev/urandom | base64 | tr -d '=+/' | head -c "$len"
}

# UUID v4 (aleatorio)
uuid_gen() {
    local uuid=$(cat /proc/sys/kernel/random/uuid 2>/dev/null)
    [[ -n "$uuid" ]] && echo "$uuid" || python3 -c "import uuid; print(uuid.uuid4())"
}
```

### 5.4 `shuf` - Permutación Aleatoria (GNU coreutils)

```bash
# Números 1-10 aleatorios
shuf -i 1-10 -n 5

# Líneas de archivo aleatorias
shuf -n 3 archivo.txt

# Mezclar array
array=(a b c d e)
shuf -e "${array[@]}"

# Una línea aleatoria de stdin
echo -e "a\nb\nc" | shuf -n 1
```

### 5.5 `sort -R` - Orden Aleatorio

```bash
# Mezclar líneas
printf "a\nb\nc\n" | sort -R
```

---

## 6. Aplicaciones Prácticas

### 6.1 Generar Contraseñas

```bash
gen_pass() {
    local len=${1:-16}
    local chars='A-Za-z0-9!@#$%^&*()_+-='
    local pass=""
    for ((i=0; i<len; i++)); do
        pass+=${chars:$((RANDOM % ${#chars})):1}
    done
    echo "$pass"
}

# Versión más segura (usando /dev/urandom)
gen_pass_secure() {
    local len=${1:-16}
    LC_ALL=C tr -dc 'A-Za-z0-9!@#$%^&*()_+-=' < /dev/urandom | head -c "$len"
    echo
}
```

### 6.2 Seleccionar Elemento Aleatorio de Array

```bash
random_element() {
    local -n arr=$1
    echo "${arr[$((RANDOM % ${#arr[@]}))]}"
}

frutas=(manzana banana cereza durazno)
random_element frutas
```

### 6.3 Barajar Array (Fisher-Yates)

```bash
shuffle_array() {
    local -n arr=$1
    local i j tmp
    for ((i=${#arr[@]}-1; i>0; i--)); do
        j=$((RANDOM % (i+1)))
        tmp=${arr[i]}; arr[i]=${arr[j]}; arr[j]=$tmp
    done
}
```

### 6.4 Muestreo Reservoir (Stream Grande)

```bash
# k elementos aleatorios de stream infinito/grande
reservoir_sample() {
    local k=$1
    local -n out=$2
    local i=0 item
    out=()
    while read -r item; do
        if (( i < k )); then
            out+=("$item")
        else
            local j=$((RANDOM % (i+1)))
            (( j < k )) && out[j]="$item"
        fi
        ((i++))
    done
}

# Uso: seq 1 1000000 | reservoir_sample 10 muestra
```

### 6.5 Backoff Exponencial con Jitter

```bash
# Reintento con backoff exponencial + jitter aleatorio
retry_with_backoff() {
    local max_attempts=${1:-5}
    local base_delay=${2:-1}
    local max_delay=${3:-60}
    local attempt=0
    
    while (( attempt < max_attempts )); do
        if "$@"; then
            return 0
        fi
        ((attempt++))
        if (( attempt < max_attempts )); then
            local delay=$(( base_delay * 2**(attempt-1) ))
            (( delay > max_delay )) && delay=$max_delay
            # Jitter: ±25%
            local jitter=$(( delay / 4 ))
            delay=$(( delay + (RANDOM % (2*jitter+1)) - jitter ))
            (( delay < 1 )) && delay=1
            sleep "$delay"
        fi
    done
    return 1
}

# Uso
retry_with_backoff 3 2 30 curl -sf http://api.example.com/health
```

---

## 7. Secuencias y Series

```bash
# Secuencias numéricas
seq 1 10              # 1..10
seq 1 2 10            # 1 3 5 7 9
seq -w 01 10          # 01 02 ... 10 (zero-pad)
seq -s, 1 5           # 1,2,3,4,5 (separador)

# Brace expansion (Bash)
echo {1..10}
echo {01..10}
echo {10..1}          # Descendente
echo {1..10..2}       # Paso 2 (Bash 4+)
echo {a..z}           # Letras

# Combinaciones
echo {a..c}{1..3}     # a1 a2 a3 b1 b2 b3 c1 c2 c3
```

---

## 8. Benchmarking y Timing

```bash
# Tiempo en segundos con precisión nanosegundos
timer_start() { timer_start_ns=$(date +%s%N); }
timer_elapsed() {
    local end_ns=$(date +%s%N)
    echo $(( (end_ns - timer_start_ns) / 1000000 ))  # ms
}

timer_start
sleep 0.5
echo "Transcurrido: $(timer_elapsed) ms"

# Con $SECONDS (segundos enteros, resetable)
SECONDS=0
sleep 2
echo "Segundos: $SECONDS"
```

---

## 9. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - `$RANDOM` básico
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - Bucles con contadores
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md) - Funciones matemáticas
- [Manipulación de Cadenas y Arrays](./Bash%20Scripting%20-%20Manipulaci%C3%B3n%20de%20Cadenas%20y%20Arrays%20Avanzados.md)

---

*Etiquetas: #bash #aritmética #matemáticas #random #bc #awk #aleatorio #uuid #backoff #benchmark*