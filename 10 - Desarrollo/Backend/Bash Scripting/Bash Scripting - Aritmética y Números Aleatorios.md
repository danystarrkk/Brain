---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Aritmética y Números Aleatorios

Bash hace **aritmética entera de 64 bits nativa** (desde bash 4.3 en plataformas de 64 bits) y trae un generador pseudoaleatorio builtin (`$RANDOM`). No hay flotantes nativos — para decimales se delega a `awk` o `bc`. Este nota cubre las tres formas de calcular (y por qué dos están deprecadas), la semántica completa de operadores y exit codes, y todo lo relacionado con aleatoriedad: desde escalar `$RANDOM` correctamente hasta seeds reproducibles y `/dev/urandom` para seguridad.

## Tres formas de calcular: `$(( ))`, `(( ))`, `let`, `expr`

| Forma | Tipo | Devuelve | Estado |
|---|---|---|---|
| `$(( expr ))` | **Expansión** | el valor como string (se usa dentro de otro contexto) | Estándar, preferida |
| `(( expr ))` | **Comando** | exit code: 0 si el valor final ≠ 0 | Estándar, preferida para condiciones |
| `let "expr"` | Builtin | exit code igual que `(( ))` | Legacy: hace lo mismo con sintaxis peor |
| `expr 1 + 1` | **Proceso externo** | valor por stdout + rc | **Deprecated**: fork por cada operación, escapes con `\*` |

```bash
a=$(( 5 + 3 ))        # valor en contexto de expansión: a=8
(( a = 5 + 3 ))       # comando: asigna, no imprime nada (rc=0 porque 8≠0)
let "a = 5 + 3"       # idéntico a (( )) pero con comillas obligatorias con espacios
expr 2 \* 3           # → 6 (verificado) pero: 2 forks, \* obligatorio, sin variables

(( a **= 2 ))         # potencia (bash 3.1+): verificado 2**10 = 1024
```

> [!tip] Por qué `expr` está muerto
> `expr` es un binario externo: **cada operación es un fork + exec**. En un bucle de 10k cálculos, `expr` tarda ~100× más que `$(( ))`. Además requiere escapar `*` y `|`, no soporta comparaciones complejas, y sus dos canales de retorno (valor por stdout, condición por rc) generan código verboso. `(( ))` hace todo eso builtin. Si ves `expr` en un script moderno, reemplazalo.

### Semántica de `(( ))` — la inversión que confunde a todos

`(( expr ))` devuelve como **exit code** el resultado de la pregunta "¿el valor final es distinto de cero?":

```bash
(( 5 > 3 )); echo $?   # → 0  (verdadero aritmético → rc 0)
(( 2 > 3 )); echo $?   # → 1
(( x = 0 ));  echo $?   # → 1  ← ASIGNAR CERO devuelvelo "falso"
(( y = 7 )); echo $?   # → 0
```

En C, `true==1` y `false==0`; acá se **invierte**: valor no-cero → exit 0. Las consecuencias prácticas:

- En `if (( ... ))` y `while (( ... ))` todo se comporta "como se espera".
- Como **comando suelto** con `set -e`, `(( contador++ ))` partiendo de 0 **mata el script** (valor resultante 0 → rc 1). Usá `(( contador+=1 ))` — verificado en [[Bash Scripting - Control de Flujo]].
- `(( x = 5 ))` no imprime; el valor hay que extraerlo: `y=$(( x = 5 ))` → `y=5`.

## Operadores completos

### Por categoría

| Categoría | Operadores |
|---|---|
| Aritméticos | `+` `-` `*` `/` `%` `**` (potencia, asoc. derecha) |
| Incremento | `++a` `a++` `--a` `a--` (post devuelve valor viejo) |
| Lógicos | `&&` `\|\|` `!` (devuelven 1 o 0 como valor) |
| Comparación | `<` `<=` `>` `>=` `==` `!=` (devuelven 1 o 0) |
| Bitwise | `&` `\|` `^` `~` `<<` `>>` |
| Asignación | `=` `+=` `-=` `*=` `/=` `%=` `<<=` `>>=` `&=` `\|=` `^=` |
| Ternario | `cond ? a : b` |
| Coma | `,` (secuencia dentro de la expresión) |

### Precedencia (de mayor a menor)

| Nivel | Operadores | Asoc. |
|---|---|---|
| 1 | `**` | derecha |
| 2 | `*` `/` `%` | izquierda |
| 3 | `+` `-` | izquierda |
| 4 | `<<` `>>` | izquierda |
| 5 | `<` `<=` `>` `>=` | izquierda |
| 6 | `==` `!=` | izquierda |
| 7 | `&` | izquierda |
| 8 | `^` | izquierda |
| 9 | `\|` | izquierda |
| 10 | `&&` | izquierda |
| 11 | `\|\|` | izquierda |
| 12 | `?:` | derecha |
| 13 | `=` `+=` `-=` ... | derecha |
| 14 | `,` | izquierda |

```bash
(( r = 2 + 3 * 4 ))      # → 14 (no 20)
(( r = (2 + 3) * 4 ))    # → 20
(( max = a > b ? a : b ))# ternario
```

### Trampas numéricas verificadas

```bash
echo $(( 7 / 2 ))      # → 3        división ENTERA (trunca)
echo $(( -7 / 2 ))     # → -3       trunca hacia CERO (no hacia -infinito)
echo $(( 7 % 2 ))      # → 1        módulo con signo del DIVIDENDO
echo $(( -7 % 2 ))     # → -1       (verificado)
echo $(( 1/0 ))        # → error bash: division by 0 — el SCRIPT MUERE con set -e
echo $(( 010 + 1 ))    # → 9        ← TRAMPA: 010 es OCTAL (=8). Verificado.
echo $(( 10#010 + 1 )) # → 11       base explícita 10# fuerza decimal
echo $(( 0b1010 ))     # → ERROR "value too great for base" — bash NO acepta 0b en aritmética
echo $(( 2#1010 ))     # → 10       binario con notación base#
```

### Bases: notación `base#número`

```bash
(( hex = 16#FF ))     # → 255      (verificado: 0xFF también vale 255)
(( oct = 8#17 ))      # → 15       (017 octal también = 15, pero solo si no hay 8/9)
(( bin = 2#1010 ))    # → 10
(( b36 = 36#ZZ ))     # → 1295     cualquier base 2-64 (64 usa a-zA-Z0-9@_)

printf '%x %X %o\n' 255 255 255   # → ff FF 377  (formato de salida)
printf '%08d\n' 42                # → 00000042
```

## Matemáticas con decimales: `bc`, `awk`, Python

Bash solo tiene enteros. Tres alternativas, en orden de disponibilidad práctica:

```bash
# 1. awk — casi siempre instalado, rápido, sin shell-injection si se usa -v
awk 'BEGIN { printf "%.4f\n", 1/3 }'                    # → 0.3333
awk -v r=5 'BEGIN { printf "%.2f\n", 3.14159 * r * r }' # → 78.54
awk 'BEGIN { print sqrt(16), log(10), exp(1), sin(1.5708) }'

# 2. bc — precisión arbitraria, estilo calculadora (NO instalado por defecto en algunos sistemas)
echo "3.14 * 2" | bc                 # → 6.28
echo "scale=10; 1/3" | bc           # → .3333333333
echo "s(1.5708)" | bc -l            # seno con librería -l
calc() { echo "scale=${2:-10}; $1" | bc -l; }
calc "sqrt(16)"                     # → 4

# 3. Python — cuando la matemática se pone seria
python3 -c 'import math; print(math.pi * 5**2)'
```

> [!warning] Seguridad de `bc` con variables
> `echo "$entrada_usuario * 2" | bc` es **inyección**: el usuario puede meter `quit` o llamar archivos. Para inputs no confiables usá `awk -v v="$entrada" 'BEGIN { print v * 2 }'` (`-v` escapa) o validación previa con regex.

La frontera de "cuándo dejar Bash" para cálculos serios: [[Python - Fundamentos y Sintaxis]].

## `$RANDOM`: el aleatorio builtin

`$RANDOM` es una **expansión** (no un comando) que expande a un entero **0-32767** (15 bits), generado por un PRNG lineal propio de bash (Bash 4.3+ lo reseede con entropía al arrancar; antes era predecible). Cada evaluación **avanza el estado** del generador.

```bash
echo $RANDOM                        # 0..32767
echo $(( RANDOM % 100 ))            # 0..99
M=10; N=20
echo $(( RANDOM % (N - M + 1) + M ))# 10..20 (verificado: muestra 12 17 18 10 18 10 13 16)

# Los valores CONSECUTIVOS son independientes en la práctica, pero:
echo "$RANDOM $RANDOM"              # dos draws del mismo stream
```

### Escalado: el sesgo del módulo

`RANDOM % n` tiene **sesgo** cuando 32768 no es múltiplo de `n` (los primeros `32768 % n` valores tienen doble probabilidad). Para usos normales (juegos, sampling, jitter) no importa; para distribuciones uniformes serias:

```bash
# Uniforme sin sesgo: descartar la cola del rango que no divide exacto
rand_uniform() {   # 0..$1-1 uniforme
    local n=$1 limit
    limit=$(( 32768 - 32768 % n ))          # múltiplo de n más cercano por debajo
    local r=$RANDOM
    while (( r >= limit )); do r=$RANDOM; done   # rechazo de la zona sesgada
    echo $(( r % n ))
}
rand_uniform 10     # 0..9 uniforme de verdad
```

### `$SRANDOM` (Bash 5.1+)

```bash
echo $SRANDOM                    # 0..4294967295 (32 bits, mejor entropía) — verificado: 681674364
echo $(( SRANDOM % 1000 ))       # rango grande sin la repetición del ciclo de 15 bits
```

`SRANDOM` se reseedeó con entropía del kernel en cada invocación en bash 5.1+, pero **sigue siendo un PRNG no criptográfico**. Para tokens/claves: `/dev/urandom` o `openssl rand` — ver abajo.

### Seed: reproducibilidad

El estado del PRNG se puede fijar — verificado que es **determinista**:

```bash
RANDOM=42; r1=$RANDOM
RANDOM=42; r2=$RANDOM
# r1 == r2  (verificado: 17772 == 17772)
```

Patrón de testing: seed fija en los tests ⇒ outputs deterministas. En producción, **nunca** fijes el seed (con seed conocido, el "aleatorio" es predecible por atacantes).

## Aleatoriedad del sistema y herramientas

```bash
# /dev/urandom — entropía del kernel, sin bloqueo (la recomendada)
head -c 16 /dev/urandom | od -An -tx1 | tr -d ' \n'   # 32 hex chars
LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 12; echo   # token alfanumérico
#   LC_ALL=C evita que tr falle con el locale al filtrar bytes altos

# Token URL-safe
head -c 9 /dev/urandom | base64 | tr '+/' '-_' | tr -d '='   # 12 chars

# UUID v4 (Linux)
cat /proc/sys/kernel/random/uuid      # verificado en kernel moderno

# shuf (GNU coreutils) — permutación/muestreo sin implementar nada
shuf -i 1-100 -n 5        # 5 enteros únicos de 1..100      (verificado formato)
shuf -n 3 archivo.txt     # 3 líneas aleatorias del archivo
shuf -e a b c d           # barajar argumentos
seq 1 100 | shuf -n 10    # sample de un stream

# sort -R — orden aleatorio por hash de línea (para dedupe aleatorio)
printf 'a\nb\nc\n' | sort -R
```

> [!warning] `$RANDOM` no es criptografía
> El PRNG de bash es lineal y su estado puede reconstruirse con ~16 draws observados. **Jamás** uses `$RANDOM` para contraseñas, tokens de sesión, salts o nonces. Para eso: `head -c 32 /dev/urandom | base64`, `openssl rand -hex 16` o `python3 -c 'import secrets; print(secrets.token_hex(16))'`.

## Recetas con módulo y aleatoriedad

```bash
# Selección aleatoria de elemento de array (con SRANDOM si existe)
random_element() {
    local -n _arr=$1
    local n=${#_arr[@]} r
    (( n )) || return 1
    r=${SRANDOM:-$RANDOM}
    printf '%s' "${_arr[$(( r % n ))]}"
}

# Fisher-Yates para barajar un array en sitio
shuffle_array() {
    local -n _arr=$1
    local i j tmp r
    for (( i=${#_arr[@]}-1; i>0; i-- )); do
        j=$(( ( ${SRANDOM:-$RANDOM} ) % (i+1) ))
        tmp=${_arr[i]}; _arr[i]=${_arr[j]}; _arr[j]=$tmp
    done
}

# Reservoir sample: k items uniformes de un stream de tamaño desconocido
reservoir() {   # reservoir <k>  — lee stdin, escribe muestra por stdout
    local k=$1 i=0 j item
    local -a out=()
    while IFS= read -r item; do
        if (( i < k )); then out+=("$item")
        else
            j=$(( ( ${SRANDOM:-$RANDOM} ) % (i+1) ))
            (( j < k )) && out[j]="$item"
        fi
        (( i += 1 ))
    done
    printf '%s\n' "${out[@]}"
}
seq 1 1000000 | reservoir 5

# Backoff exponencial con jitter (patrón de red — en Patrones Comunes)
delay=$(( base * 2 ** intento ))
jitter=$(( delay / 4 ))
delay=$(( delay + RANDOM % (2 * jitter + 1) - jitter ))
```

## Secuencias y timers

```bash
seq 1 10          # 1..10            seq 1 2 10 → impares    seq -w 01 10 → zero-pad
echo {1..10}      # brace expansion (bash; sin fork)
echo {1..10..2}   # con paso (bash 4+)
echo {a..z}       # letras           echo {1..3}{a..c} → 1a 1b 1c 2a ...

# Medición de tiempo
SECONDS=0; sleep 2; echo "$SECONDS"                    # builtin, segundos enteros, reseteable
t0=$(date +%s%N); ...; echo $(( ( $(date +%s%N) - t0 ) / 1000000 ))  # milisegundos
time comando                       # keyword: real/user/sys
EPOCHREALTIME                      # bash 5+: segundos.microsegundos (sin fork)
awk -v t="$EPOCHREALTIME" 'BEGIN { printf "%.3f\n", t }'
```

## Errores comunes

1. **`(( contador++ ))` con `set -e` partiendo de 0** — mata el script (rc=1 del valor 0). `(( contador+=1 ))`.
2. **`010` interpretado como octal** — `$((010))` es 8, no 10. Decimal explícito: `10#010`.
3. **`RANDOM % n` para seguridad** — predecible y sesgado. `/dev/urandom` para secretos.
4. **División por cero no chequeada** — `$((1/0))` aborta el script con error del parser. Validar el divisor.
5. **`echo $(( 10#$var ))` con var vacía o con signo** — `10#` no admite `-`: validar con regex antes (`^-?[0-9]+$`) y manejar el signo aparte.
6. **`expr` en código nuevo** — fork innecesario, sintaxis de escapes. `(( ))`.
7. **`let` sin comillas con espacios** — `let a = 5` es error de sintaxis; `(( a = 5 ))` no tiene ese problema.
8. **Sumas en `bc` sin `scale`** — `bc` trunca enteros por defecto fuera de la división; `scale` antes de la expresión.
9. **Fisher-Yates con `j=$(( RANDOM % i ))` en vez de `% (i+1)`** — sesgo fuerte; el `% (i+1)` es el correcto (y el `% n` con rechazo si se necesita uniformidad real).

## Cheatsheet

| Operación | Forma | Nota |
|---|---|---|
| Expansión | `$(( a + b ))` | valor como string |
| Condición/comando | `(( a > b ))` | rc 0 si ≠ 0 |
| Asignar y usar | `c=$(( a = b + 1 ))` | `(( ))` solo no imprime |
| Potencia | `$(( 2 ** 10 ))` | bash 3.1+ |
| Módulo | `$(( x % n ))` | recetas de rango |
| Binario/hex | `2#1010` · `0xFF` · `16#FF` | `0b1010` NO existe |
| Decimal explícito | `10#010` | evita el octal |
| Aleatorio 0-32767 | `$RANDOM` | no criptográfico |
| Aleatorio 32 bits | `$SRANDOM` | bash 5.1+ |
| Rango M..N | `$(( M + RANDOM % (N - M + 1) ))` | sesgo menor: rechazo |
| Seed reproducible | `RANDOM=42` | solo para tests |
| Token seguro | `head -c 32 /dev/urandom \| base64` | criptográfico |
| Sample de archivo | `shuf -n 3 f` · `shuf -i 1-100 -n 5` | GNU |
| Flotantes | `awk 'BEGIN{...}'` · `bc` | awk preferido |
| Timer | `SECONDS` · `EPOCHREALTIME` · `time` | |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — `declare -i` y variables especiales (`RANDOM`, `SECONDS`)
- [[Bash Scripting - Control de Flujo]] — `(( ))` como condición y for C-style
- [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]] — índices computados y `printf` de formato
- [[Bash Scripting - Patrones Comunes y Plantillas]] — backoff con jitter y timers en scripts reales
- [[Bash Scripting - Procesos y Señales]] — traducción de exit codes 128+N usada aquí
- [[Python - Fundamentos y Sintaxis]] — cuando los números se ponen serios
