---
aliases: ["Bash Strings y Arrays", "Manipulación de Cadenas Bash", "Arrays Avanzados Bash", "Parameter Expansion"]
tags:
  - dev/backend/bash
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Bash Scripting - Fundamentos Básicos]]"
  - "[[Bash Scripting - Control de Flujo]]"
  - "[[Bash Scripting - Funciones y Scope]]"
  - "[[Bash Scripting - Entrada-Salida y Redirección]]"
  - "[[Bash Scripting - Aritmética y Números Aleatorios]]"
  - "[[Python - Fundamentos y Sintaxis]]"
fuente: "https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion"
---

# Bash Scripting - Manipulación de Cadenas y Arrays Avanzados

Bash no tiene tipo `string` ni tipo `array` como lenguajes de programación modernos: tiene **expansión de parámetros** (una gramática de sustitución sobre variables) y **arrays indexados/associativos** implementados como tablas hash del shell. Aun así, con parameter expansion resuelves el 90% del parsing de rutas, nombres y campos **sin hacer un solo fork** — que es lo que importa, porque cada externo (`sed`, `cut`, `awk`) en un bucle cuesta un orden de magnitud más que la operación builtin.

## Expansión de parámetros: catálogo completo

La forma general es `${variable[operador[patrón]]}`. Todos los operadores funcionan sobre **cualquier variable**, no solo sobre arrays.

### Longitud y substrings

```bash
var="archivo.tar.gz"      # 14 caracteres

echo "${#var}"            # → 14        longitud
echo "${var:0:7}"         # → archivo   offset 0, largo 7
echo "${var:2:5}"         # → chivo     (verificado)
echo "${var:7}"           # → .tar.gz   desde offset hasta el final
echo "${var: -3}"         # → .gz       últimos 3 (el ESPACIO antes de - es obligatorio)
echo "${var: -7:3}"       # → .ta       desde -7, largo 3 (verificado — ojo: ".ta", no "tar")
echo "${var:2:-4}"        # → archivo.ta desde 2, excluyendo los últimos 4 (verificado)
```

> [!warning] El espacio obligatorio en `${var: -3}`
> `${var:-3}` (sin espacio) es **otro operador**: es el de valor por defecto (`-` seguido de `3` como default). `${var: -3}` con espacio es substring negativo. Error clásico que produce resultados silenciosamente wrong.

### Eliminar prefijos y sufijos (patrón glob, NO regex)

```bash
var="archivo.tar.gz"

${var#*.}     # → tar.gz    quita prefijo MÁS CORTO que matchea *. (hasta el primer punto)
${var##*.}    # → gz        quita prefijo MÁS LARGO (hasta el último punto)
${var%.*}     # → archivo.tar  quita sufijo MÁS CORTO (desde el último punto)
${var%%.*}    # → archivo   quita sufijo MÁS LARGO (hasta el primer punto)

path="/home/usuario/documentos/archivo.txt"
echo "${path##*/}"   # → archivo.txt   (basename puro, sin fork)
echo "${path%/*}"    # → /home/usuario/documentos  (dirname puro)
echo "${path##*.}"   # → txt           (extensión)
```

Mnemotecnia: `#` está arriba del teclado = prefijo (izquierda); `%` abajo = sufijo (derecha). **Doble = el más largo, simple = el más corto.** Los patrones son glob (`*`, `?`, `[...]`), no regex.

### Sustitución

```bash
v="archivo.tar.gz"
echo "${v/tar/tgz}"      # → archivo.tgz.gz   primera ocurrencia
echo "${v//./_}"         # → archivo_tar_gz   todas (//)
echo "${v/#archivo/log}" # → log.tar.gz       anclada al INICIO (#)
echo "${v/%.gz/.bz2}"    # → archivo.tar.bz2  anclada al FINAL (%)
echo "${v/l*/LX}"        # → heLX             glob también acá: matchea "llo"
```

Con `shopt -s extglob` los patrones se potencian: `${v//+([0-9])/N}` reemplaza cada secuencia de dígitos por `N`.

### Valores por defecto, asignación y errores

| Forma | Si `var` está unset O vacía | Si `var` tiene valor |
|---|---|---|
| `${var:-default}` | devuelve `default` (no asigna) | devuelve `$var` |
| `${var-default}` | devuelve `default` solo si **unset** | vacía → devuelve vacío |
| `${var:=default}` | devuelve `default` **y lo asigna** a var | devuelve `$var` |
| `${var:+alternativo}` | devuelve vacío | devuelve `alternativo` |
| `${var:+alt}` con `-` extra (`${var-}`) | variantes idem para el caso solo-unset | |
| `${var:?mensaje}` | **muere el shell** con `mensaje` en stderr | devuelve `$var` |

Verificado: con `u` unset y `e=""`: `${u:-def}`=`def`, `${e:-def}`=`def`, `${u-def}`=`def`, `${e-def}`=`` (vacío), `${u:+alt}`=``, `${e:+alt}`=``.

```bash
: "${ARCHIVO:?Falta la variable ARCHIVO}"    # guard canónico de arranque
set -u   # sin esto, ${var:?} es la única protección contra variables no seteadas
```

### Case modification (Bash 4+)

```bash
v="hello WORLD"
echo "${v^}"     # → Hello WORLD    primera mayúscula
echo "${v^^}"    # → HELLO WORLD    todo mayúsculas
echo "${v,}"     # → hello WORLD    primera minúscula
echo "${v,,}"    # → hello world    todo minúsculas
echo "${v~}" / "${v~~}"   # invertir primera / todas
echo "${v^^[aeiou]}"      # → hEllO WOrLD  (verificado) SOLO las que matchean el patrón
```

## Word splitting y globbing en profundidad

Los dos mecanismos que convierten `"$var"` en bug o en feature:

1. **Word splitting**: toda expansión **no protegida por comillas** se parte en palabras usando `IFS` (default: espacio, tabulador, newline — en ese orden).
2. **Globbing**: cada palabra resultante se testea contra el filesystem (`*`, `?`, `[...]`); si matchea, se reemplaza por los filenames (orden alfabético, sin dotfiles por defecto).

```bash
v="a  b"
printf '[%s]' $v     # → [a][b]        splitting: 2 palabras
printf '[%s]' "$v"   # → [a  b]        protegido

IFS=:; set -- a:b:c
printf '[%s]' $1     # → [a][b][c]     IFS custom redefine el separador
unset IFS            # ← CUIDADO: IFS vacío DESACTIVA el splitting:
v="a b"; printf '[%s]' $v   # → [a b]  una sola palabra

IFS=$' \t\n'         # restaurar el default cuando terminás
```

Shopt de apoyo:

| Shopt | Efecto |
|---|---|
| `nullglob` | Glob sin matches → expansión vacía (no el literal) |
| `failglob` | Glob sin matches → error, no expande |
| `dotglob` | `*` incluye archivos ocultos |
| `extglob` | Patrones `?(x) *(x) +(x) @(x) !(x)` en glob y `[[ ]]` |
| `noclobber` | `>` no pisa archivos existentes (usa `>|` para forzar) |

Verificado: `bash -c 'shopt -s nullglob; a=(*.nometch); echo ${#a[@]}'` → `0` iteraciones.

## Arrays indexados

```bash
frutas=(manzana banana cereza)      # asignación literal (sin comillas: splitting+glob intencional)
declare -a nums=(1 2 3 4 5)
arr[10]="diez"                      # índices sparse: huecos permitidos
arr+=("uva" "pera")                 # append (bash 3.1+)
```

### Acceso

```bash
echo "${frutas[0]}"        # primer elemento (0-based)
echo "${frutas[-1]}"       # ÚLTIMO (bash 4.3+)
echo "${frutas[@]}"        # todos como palabras SEPARADAS (quoted: array expansion)
echo "${frutas[*]}"        # todos unidos en un string con IFS (quoted)
echo "${#frutas[@]}"       # cantidad de elementos
echo "${#frutas[0]}"       # longitud del elemento 0
echo "${!frutas[@]}"       # los ÍNDICES (0 1 2 ...)
```

Verificado el quoting — la diferencia entre `"${arr[@]}"` y `"${arr[*]}"`:

```bash
arr=(a b c d); IFS=,
echo "${arr[*]}"    # → a,b,c,d   (un string, unido con el primer char de IFS)
echo "${arr[@]}" | tr ' ' ','   # → a,b,c,d pero como PALABRAS separadas
# En contextos de comando es donde importa de verdad:
printf '[%s]\n' "${arr[@]}"     # → [a] [b] [c] [d]   4 argumentos
```

> [!warning] `"${arr[@]}"` es LA expansión segura para pasar arrays a comandos
> Sin comillas, cada elemento se splittinguea de nuevo y se globbea: `arr=("un archivo.txt" "*.log"); rm ${arr[@]}` borra cosas que nunca pediste. **`"${arr[@]}"` siempre**, en `for`, en `printf`, en `rm`.

### Modificación y reindexado

```bash
arr[3]="durazno"        # asigna en índice 3 (crea hueco si 2 no existe)
unset 'arr[1]'          # borra el elemento (comillas simples: evita la expansión del índice)
arr+=("extra")          # append al final
arr=("${arr[@]}")       # "compactar": reconstruir reindexando desde 0
```

> [!tip] Slices de arrays
> ```bash
> arr=(a b c d e)
> echo "${arr[@]:1:3}"    # → b c d    desde índice 1, 3 elementos
> echo "${arr[@]: -2}"    # → d e      últimos 2 (mismo espacio obligatorio)
> ```

## Arrays asociativos (Bash 4+)

Hash maps reales: claves de texto con lookup O(1).

```bash
declare -A colores=(
    [rojo]="#FF0000"
    [verde]="#00FF00"
    ["clave con espacios"]="#0000FF"    # claves con espacio: comillas dentro del literal
)

colores[azul]="#0000FF"       # agregar
echo "${colores[rojo]}"       # lookup
echo "${!colores[@]}"         # todas las claves
echo "${colores[@]}"          # todos los valores
echo "${#colores[@]}"         # cantidad
[[ -v colores[rojo] ]] && echo "existe"    # verificación (bash 4.2+)
unset 'colores[rojo]'         # borrar

for clave in "${!colores[@]}"; do           # iteración canónica
    printf '%s -> %s\n' "$clave" "${colores[$clave]}"
done
```

Gotchas verificados:

- `declare -A` **sin inicializar** y luego usar `mapa[clave]` en contexto de expansión → "unbound variable" con `-u`. Declará con `=()` o inicializá.
- `+=` en asociativos es **concatenación de strings**, no aritmética: `m[a]=1; m[a]+=5` → `15` (verificado). Para sumar: `m[a]=$(( m[a] + 5 ))`.
- `[[ -v mapa[clave] ]]` con claves que tienen caracteres raros puede confundir al parser: usá `[[ -n "${mapa[clave]+x} ]]` como alternativa blindada.

### Matrices simuladas

```bash
# Opción 1: clave compuesta
declare -A matriz
matriz[0,0]=1; matriz[0,1]=2; matriz[1,0]=3
echo "${matriz[1,0]}"     # → 3

# Opción 2: índices separados por subíndices de dos variables
declare -A m2
for ((i=0; i<3; i++)); do for ((j=0; j<3; j++)); do m2["${i}_${j}"]=$((i*j)); done; done

# Opción 3: array de arrays vía namerefs (Bash 4.3+)
fila1=(1 2 3); fila2=(4 5 6)
declare -n ref
for ref in fila1 fila2; do echo "${ref[@]}"; done   # → 1 2 3 / 4 5 6
```

## `mapfile` / `readarray`: lectura masiva sin forks

`mapfile` (alias `readarray`) lee **todo un stream en un array en una sola builtin call** — para archivos grandes es 10-50× más rápido que `while read`.

```bash
mapfile -t lineas < archivo.txt     # -t: quita el newline final
mapfile -t -n 10 lineas < archivo.txt    # solo primeras 10
mapfile -t -s 1  lineas < archivo.txt    # salta las 2 primeras (skip)
mapfile -d '' -t arr < <(find . -print0) # delimitador NUL: filenames a prueba de todo
mapfile -O 3 -t arr < archivo.txt        # empieza a escribir en el índice 3
mapfile -t -C procesar_linea -c 100 < big.txt   # callback cada 100 líneas (bash 4.4+)
```

Verificado: `mapfile -td ',' arr <<< "a,b,c,"` → 4 elementos (el campo vacío final **se preserva**). `readarray` es sinónimo exacto.

> [!warning] `mapfile` no parsea la última línea sin `\n`
> Igual que `read`: con `printf 'a\nb'` (sin newline final) la última línea queda fuera. Para archivos de origen desconocido agregá un newline: `mapfile -t arr < <(cat archivo; echo)`.

## Here-strings y manipulación sin externos

```bash
tr 'a-z' 'A-Z' <<< "hola"        # stdin = "hola\n" (aquí SIEMPRE se agrega \n)
IFS=: read -r a b c <<< "x:y:z"  # splitting instantáneo en 3 campos
read -r -d '' multi << 'EOF'     # heredoc → variable (rc=1 sin \n final: esperado)
contenido
EOF
```

### Split de strings

```bash
IFS=',' read -ra campos <<< "a,b,c"        # → (a b c)
IFS=',' read -ra campos <<< "a,b,,d"       # → 4 campos, vacío preservado EN el medio (verificado)
# El campo vacío FINAL se pierde con read (el delimitador corta):
IFS=',' read -ra f <<< "a,b,"; echo ${#f[@]}    # → 2 (no 3)

# Múltiples delimitadores: cada char de IFS separa
IFS=$',;' read -ra campos <<< "a,b;c,d"    # → (a b c d)

# Para splits multi-línea, read no sirve (lee UNA línea siempre):
v=$'l1\nl2'
mapfile -t lineas <<< "$v"                 # → 2 elementos correcto
```

### Trim sin externos

```bash
trim() {   # elimina espacios/tab/newlines de ambos extremos
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"    # leading: al primer no-blanco
    var="${var%"${var##*[![:space:]]}"}"    # trailing: al último no-blanco
    printf '%s' "$var"
}
echo "[$(trim "  hola mundo  ")" ]"         # → [hola mundo ] (verificado, sin \n extra)

# Con extglob, la versión legible:
shopt -s extglob
trim2() { local v="$*"; v="${v##*([[:space:]])}"; v="${v%%*([[:space:]])}"; printf '%s' "$v"; }
```

### Búsqueda sin externos

```bash
s="hello world"
[[ "$s" == *wor* ]] && echo "contiene"                  # subcadena: glob en ambos lados

str_index() {   # posición de la subcadena, -1 si no está
    local haystack="$1" needle="$2" temp="${haystack%%$needle*}"
    [[ "$temp" == "$haystack" ]] && printf '%s' -1 || printf '%s' "${#temp}"
}
str_index 'hello world' 'world'    # → 6   (verificado)
str_index 'hello' 'zz'             # → -1

count_occurrences() {   # contar no-solapadas
    local str="$1" substr="$2" count=0
    while [[ "$str" == *"$substr"* ]]; do str="${str#*$substr}"; ((count+=1)); done
    printf '%s' "$count"
}
count_occurrences banana a          # → 3   (verificado)
```

### Join y padding

```bash
join_by() { local IFS="$1"; shift; printf '%s' "$*"; }
join_by , a b c                     # → a,b,c   (verificado)

printf '%05d|%-8s|%8s\n' 42 hola hola   # → 00042|hola    |    hola
printf '%*s' 5 ""                     # 5 espacios (ancho por argumento)
```

### Map / filter / reduce con namerefs

```bash
map_array()  { local -n _a=$1 _o=$3; local f=$2; _o=(); for e in "${_a[@]}"; do _o+=("$("$f" "$e")"); done; }
filter_array(){ local -n _a=$1 _o=$3; local p=$2; _o=(); for e in "${_a[@]}"; do "$p" "$e" && _o+=("$e"); done; }
reduce_array(){ local -n _a=$1; local f=$2 acc=$3; for e in "${_a[@]}"; do acc=$("$f" "$acc" "$e"); done; printf '%s' "$acc"; }

nums=(1 2 3 4 5)
doble() { echo $(( $1 * 2 )); }
es_par() { (( $1 % 2 == 0 )); }
sumar()  { echo $(( $1 + $2 )); }

declare -a doblados pares
map_array nums doble doblados        # → 2 4 6 8 10
filter_array nums es_par pares       # → 2 4
reduce_array nums sumar 0            # → 15
```

La mecánica de `local -n` (pass-by-reference) está en [[Bash Scripting - Funciones y Scope]].

## Estructuras de datos

```bash
# Cola FIFO (amortizado con head-shift; suficiente para cientos de items)
queue=()
queue_push() { queue+=("$1"); }
queue_pop()  { local v="${queue[0]}"; queue=("${queue[@]:1}"); printf '%s' "$v"; }

# Pila LIFO (O(1))
stack=()
stack_push() { stack+=("$1"); }
stack_pop()  { local i=$(( ${#stack[@]} - 1 )) v="${stack[i]}"; unset 'stack[i]'; printf '%s' "$v"; }

# Registro tipo struct: asociativo por campo + índice compuesto por fila
declare -A u=([nombre]="Juan Pérez" [email]="juan@ejemplo.com" [activo]=1)
declare -A tabla
tabla[1001,nombre]="Ana"; tabla[1001,email]="ana@ejemplo.com"
```

## Parsing: cuándo Bash alcanza y cuándo no

| Tarea | Bash puro | Herramienta externa | Veredicto |
|---|---|---|---|
| Ruta→basename/dirname/ext | `${p##*/}` `${p%/*}` | `basename`, `dirname` | **Bash** (0 forks) |
| Split de un campo | `IFS=, read -ra` | `cut -d, -f3` | Bash para pocos campos |
| Reemplazo simple | `${v//a/b}` | `sed 's/a/b/g'` | Bash en variables; sed en streams |
| Búsqueda/regex sobre string | `[[ =~ ]]` | `grep -oP` | Bash para 1-2 capturas |
| **CSV real** (comillas, comas dentro) | no fiable | `mlr`, `csvkit`, Python | **Nunca Bash** |
| **JSON** | regex frágil | **`jq`** | **Siempre jq** |
| YAML/TOML | imposible | python, yq | externo |
| >10k líneas | muy lento | `awk`, `sed`, `perl` | externo |

```bash
# JSON con jq (lo correcto):
jq -r '.usuario.nombre' datos.json
jq -r '.items[] | select(.activo) | .id' datos.json

# JSON mínimo SOLO si es plano y de confianza (NO recomendado):
json_get() {
    local re="\"$2\"[[:space:]]*:[[:space:]]*\"([^\"]*)\""
    [[ $1 =~ $re ]] && printf '%s' "${BASH_REMATCH[1]}"
}
json_get '{"nombre":"Juan"}' nombre   # → Juan
```

La filosofía de delegar datos a herramientas especializadas es transversal — ver la tabla de performance en [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] y el complemento en [[Python - Fundamentos y Sintaxis]] para el punto de migración.

## Rendimiento

- **Concatenar strings en bucle es O(n²)**: `s+="$x"` recopia el string cada vez. Para milenares de items, acumulá en un array y uní al final: `arr+=("$x"); s="${arr[*]}"` (o `printf` directo).
- `+=` sobre un array gigante también recopia el backing store; `mapfile` en bloque es preferible a 10k `read`.
- Cada `$( )` es un fork: dentro de bucles de 10k iteraciones, preferir parameter expansion a `$(basename "$p")` → `${p##*/}` es ~100× más rápido.
- Para transformaciones de stream, `awk` es casi siempre más rápido que el equivalente bash puro — Bash manda en el **control**, no en los **datos**.

## Errores comunes

1. **`${var: -3}` vs `${var:-3}`** — substring vs default. El espacio es semántica, no estilo.
2. **`rm ${arr[@]}` sin comillas** — splitting + globbing sobre los nombres. `"${arr[@]}"` siempre.
3. **`echo ${arr[@]}` para inspeccionar** — pierde límites de elementos con espacios; `printf '[%s]\n' "${arr[@]}"`.
4. **`arr[-1]` en array vacío** — `bad array subscript` (verificado). Guardar con `(( ${#arr[@]} )) || return`.
5. **Confundir glob con regex** — `${v//[0-9]/X}` es glob: `[0-9]` funciona, pero `+` o `\d` no existen. Para regex: `[[ =~ ]]` + sed/perl.
6. **`read -ra` pierde el campo vacío final** — agregá un delimitador ficticio: `IFS=, read -ra f <<< "$line,"` y quitá el último.
7. **`mapfile` sin `-t`** — cada elemento conserva el `\n`, ensuciando los datos.
8. **Asociativos sin `declare -A`** — `mapa[clave]=v` sobre un array indexado crea el índice *literal* `clave` y opera como string: resultados absurdos.

## Cheatsheet

| Operación | Forma | Resultado |
|---|---|---|
| Longitud | `${#v}` | n |
| Substring | `${v:offset:len}` · `${v: -len}` | desde offset / finales |
| Prefijo corto/largo | `${v#pat}` / `${v##pat}` | quita de izquierda |
| Sufijo corto/largo | `${v%pat}` / `${v%%pat}` | quita de derecha |
| Replace 1a/todas | `${v/a/b}` / `${v//a/b}` | |
| Anclados | `${v/#pat/r}` / `${v/%pat/r}` | inicio / fin |
| Default/assign | `${v:-d}` / `${v:=d}` | `:` = también si vacía |
| Mayús/minús | `${v^^}` `${v,,}` `${v^^[pat]}` | bash 4+ |
| Elementos/count | `"${arr[@]}"` / `${#arr[@]}` | comillas siempre |
| Índices | `${!arr[@]}` | |
| Slice | `"${arr[@]:1:3}"` | desde 1, 3 elems |
| Último | `"${arr[-1]}"` | bash 4.3+ |
| Append | `arr+=(x y)` | |
| Claves/values | `"${!map[@]}"` / `"${map[@]}"` | asociativos |
| Existe | `[[ -v map[k] ]]` | bash 4.2+ |
| Archivo→array | `mapfile -t arr < f` | `-t -n -s -d -C` |
| Split | `IFS=, read -ra f <<< "$s"` | |
| Join | `IFS=,; s="${arr[*]}"` | |
| Trim | `${v#"${v%%[![:space:]]*}"}` | ver sección |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — quoting y expansiones base (los pasos 3-5 del parsing)
- [[Bash Scripting - Control de Flujo]] — iterar arrays y leer archivos en bucles
- [[Bash Scripting - Funciones y Scope]] — namerefs para pass-by-reference de arrays
- [[Bash Scripting - Entrada-Salida y Redirección]] — `read`, here-strings y process substitution desde el modelo de FDs
- [[Bash Scripting - Aritmética y Números Aleatorios]] — índices computados y `(( ))`
- [[Bash Scripting - Patrones Comunes y Plantillas]] — parsing de CSV/config reutilizable
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — cuando migrar a Python y la tabla de performance
- [[Python - Fundamentos y Sintaxis]] — el siguiente escalón para datos estructurados
