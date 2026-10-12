---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Control de Flujo

El control de flujo en Bash tiene una particularidad que no existe en casi ningún otro lenguaje: **la condición no es un booleano, es el código de salida de un comando**. `if`, `while`, `&&` y `||` no evalúan expresiones: ejecutan procesos (o builtins) y miran si salieron con `0`. De ahí salen todas las peculiaridades: por qué `(( ))` devuelve 0 cuando la cuenta da "verdadero", por qué `errexit` se desactiva dentro de las condiciones, y por qué la comilla donde ponés (o no ponés) cambia qué se compara.

## `[ ]` vs `[[ ]]`: el test builtin y su primo bash

`test` (`[ ]`) es un **builtin POSIX**; `[[ ]]` es una **extensión de bash** que añade globbing, regex, word splitting desactivado y operadores lógicos sin pipes. La diferencia no es cosmética:

| Aspecto | `[ ]` (test) | `[[ ]]` |
|---|---|---|
| Word splitting de variables | **Sí** — `[ $a = $b ]` revienta con espacios | **No** — `[[ $a = $b ]]` seguro sin comillas |
| Globbing en el lado derecho | Literal (salvo glob no comillas → filename) | **Patrón glob** (`== *.txt`) |
| Regex `=~` | No existe | Sí (Bash 3.2+) |
| Lógicos internos | `-a`/`-o` (deprecados) | `&&` `\|\|` `!` |
| Paréntesis | Hay que escapar: `\(` | `( )` normales |
| Numeros no numéricos | `[ "abc" -eq 1 ]` → **error**: `integer expected`, rc=2 | `[[ abc -eq 1 ]]` → `abc` se interpreta como **variable** (unset=0): rc=1, sin error |
| Un error de sintaxis | `[: too many arguments` → rc=2 (falsa, no fatal) | Error de sintaxis real |
| Comando | Es builtin, también existe binario externo `/usr/bin/[` | Solo builtin |

Verificado en vivo:

```bash
a="a b"; b="a b"
[ $a = $b ]        # → "[: too many arguments"  (splitting en 4 palabras)
[[ $a = $b ]]      # → true (sin comillas funciona: no hay splitting)

[ "file.txt" = *.txt ]   # → false (rhs literal; o "no such file" si no existe)
[[ "file.txt" == *.txt ]] # → true  (rhs es patrón glob)

[[ "2" > "10" ]]   # → true (!) — en [[ ]] los operadores string comparan lexicográficamente
```

> [!warning] `>` y `<` en `[[ ]]` son comparaciones de STRINGS
> `[[ "2" > "10" ]]` es **true** porque compara carácter a carácter ("2" > "1"). Para números usá `-gt`/`-lt` o `(( ))`. Y ojo: `>` dentro de `[[ ]]` no necesita escape, pero en `[ ]` sí (`\>`) porque el shell lo interpretaría como redirección.

**Regla práctica**: usá `[[ ]]` siempre en bash (es builtin, no hay costo de fork). `[ ]` solo cuando necesites estricta portabilidad POSIX (dash, `sh` de BSD).

### Operadores de `[[ ]]` — catálogo completo

```bash
# --- Archivos (idénticos en [ ] y [[ ]]) ---
[[ -e ruta ]]    # existe (cualquier tipo)
[[ -f ruta ]]    # archivo regular
[[ -d ruta ]]    # directorio
[[ -L ruta ]]    # symlink
[[ -h ruta ]]    # idem -L
[[ -p ruta ]]    # FIFO
[[ -S ruta ]]    # socket
[[ -b/-c ruta ]] # dispositivo de bloques / caracteres
[[ -r/-w/-x ruta ]]  # legible / escribible / ejecutable (con los permisos VIGENTES)
[[ -s ruta ]]    # existe y tamaño > 0
[[ -u/-g/-k ruta ]]  # setuid / setgid / sticky bit
[[ a -nt b ]]    # a más nuevo que b (newer than)
[[ a -ot b ]]    # a más antiguo que b (older than)
[[ a -ef b ]]    # mismo inode (hardlink o bind mount)

# --- Strings ---
[[ -z "$s" ]]    # longitud 0 (vacío o unset sin set -u)
[[ -n "$s" ]]    # longitud > 0
[[ "$a" == "$b" ]]   # igualdad; rhs puede ser glob
[[ "$a" != "$b" ]]   # desigualdad
[[ "$a" < "$b" ]]    # orden lexicográfico (locale-aware con LC_ALL)
[[ "$a" =~ ^[0-9]+$ ]]  # regex ERE (Bash 3.2+)

# --- Números (enteros; errores si no lo son en contexto aritmético) ---
[[ $a -eq $b ]]  # igual        [[ $a -ne $b ]]  # distinto
[[ $a -lt $b ]]  # menor        [[ $a -le $b ]]  # menor o igual
[[ $a -gt $b ]]  # mayor        [[ $a -ge $b ]]  # mayor o igual

# --- Lógicos ---
[[ c1 && c2 ]]   # AND           [[ c1 || c2 ]]   # OR
[[ ! c ]]        # NOT
[[ ( c1 || c2 ) && c3 ]]  # paréntesis para agrupar
```

### Regex con `=~` y `BASH_REMATCH`

```bash
if [[ "$email" =~ ^([a-z0-9._%+-]+)@([a-z0-9.-]+\.[a-z]{2,})$ ]]; then
    echo "usuario=${BASH_REMATCH[1]} dominio=${BASH_REMATCH[2]}"
fi
```

> [!tip] El patrón de `=~` NO va entre comillas
> `[[ $x =~ "^abc" ]]` (comillas) busca la **cadena literal** `^abc`, no una regex. Para reglas reutilizables guardá el patrón **sin comillas** en una variable y usá esa variable:
>
> ```bash
> re='^[0-9]{4}-[0-9]{2}-[0-9]{2}$'   # SIN comillas al definir
> [[ "$fecha" =~ $re ]] && echo "fecha válida"   # la VARIABLE va sin comillas
> ```

Verificado: `[[ "abc" =~ ^a.c ]]` → true, `BASH_REMATCH[0]=abc`.

### Predicados de archivos como funciones — funciones como condiciones

Cualquier función sirve de condición **por su exit code**:

```bash
es_mayor() { (( $1 > $2 )); }        # (( )) da 0 si la cuenta es ≠ 0
es_menor() { (( $1 < $2 )); }

if es_mayor 5 3; then echo "5 > 3"; fi

validar() { [[ -f "$1" && -r "$1" ]]; }   # último comando define el rc
validar /etc/hostname && echo "válido"    # usable directo con &&
```

Esto es el patrón base de las librerías de funciones: **`return 0` = sí, `return 1` = no** — ver [[Bash Scripting - Funciones y Scope]].

## `if` / `elif` / `else` / `fi`

```bash
if [[ -f "$archivo" ]]; then
    procesar "$archivo"
elif [[ -d "$archivo" ]]; then
    echo "es un directorio" >&2; exit 1
else
    echo "no existe" >&2; exit 1
fi

# Forma de una línea (estilo correcto: then en la misma línea)
if [[ -n "$x" ]]; then echo sí; fi

# Combinadores habituales (sin if)
[[ -f "$f" ]] && echo "existe"        # cortocircuito: si el test falla, no ejecuta
[[ -f "$f" ]] || { echo "falta $f" >&2; exit 1; }
command -v jq >/dev/null || die "requiere jq"
```

> [!warning] `&&` como condición no es `if`
> `[[ cond ]] && acción` **falla con rc=1** si la condición es falsa — mata el script con `set -e` si no es la última línea de un pipeline. En scripts estrictos usá `if` completo o agregá `|| true`.

## `(( ))`: aritmética como condición

`(( expr ))` evalúa una expresión aritmética de C y devuelve **exit code 0 si el valor final es ≠ 0**, y 1 si es 0. Esta inversión (verdadero aritmético → rc 0) es la fuente de los malentendidos:

```bash
(( 5 > 3 )); echo $?   # → 0  (verdadero)
(( 2 > 3 )); echo $?   # → 1  (falso)
(( x = 0 ));  echo $?   # → 1  ← ASIGNAR CERO ES "FALSO": con set -e mata el script

if (( contador > 10 )); then echo "demasiados"; fi
if (( ${#arr[@]} == 0 )); then echo "vacío"; fi

# Variables SIN $ (contexto aritmético): (( a + 1 )), no (( $a + 1 ))
# Aunque $a también funciona — excepto con nombres que son operadores o con arrays:
# (( arr[i] )) sí, (( $arr[i] )) es un error silencioso en algunos contextos
```

Operadores: todos los de C — `+ - * / % **` (asoc. derecha), `++ --` pre/post, ternario `?:`, comparadores, bitwise `& | ^ ~ << >>`, lógicos `&& || !`. Comparación completa y precedencia en [[Bash Scripting - Aritmética y Números Aleatorios]].

## `case`: pattern matching múltiple

```bash
case "$ext" in
    txt|md)          echo "texto" ;;        # alternativa con |
    jpg|png|gif)     echo "imagen" ;;
    *.tar.gz|*.tgz)  echo "tarball" ;;      # glob patterns
    [0-9]*)          echo "empieza con dígito" ;;
    "")              echo "string vacío" ;; # $1 no pasado
    -*)              echo "flag" ;;         # empieza con guion
    *)               echo "default" ;;      # catch-all: SIEMPRE poné uno
esac
```

### Fall-through: `;&` y `;;&`

| Terminador | Significado | Verificado |
|---|---|---|
| `;;` | Sale del case (normal) | — |
| `;&` | **Fall-through**: ejecuta también el bloque siguiente sin testear su patrón | `case abc in a*) echo A ;;&…` → imprime A **y** el bloque de `b*)` |
| `;;&` | **Continúa escaneando** los patrones siguientes (re-testea) | `case xyz in a*)…;;& x*) echo X` → imprime X aunque a* no matcheó |

```bash
# ;;& para "múltiples patrones con lógica compartida + extra":
case "$opt" in
    -v|--verbose) log_level=debug ;;&     # sigue testeando las reglas de abajo
    -q|--quiet)   log_level=error ;;&
    -*)           flags_ok=true ;;        # cualquier flag setea esto
esac
```

### `select`: menú interactivo

```bash
PS3="Elegí opción: "
select opcion in "Iniciar" "Detener" "Salir"; do
    case $REPLY in      # $REPLY tiene el número elegido
        1) iniciar; break ;;
        2) detener; break ;;
        3) break ;;
        *) echo "Inválido: $REPLY" >&2 ;;
    esac
done
```

`select` imprime el menú en stderr, repite en loop ante entradas inválidas, y sale del loop con `break` o con EOF (Ctrl+D).

## Bucles

### `for` — cuatro formas

```bash
# 1. Por lista (la más común)
for fruta in manzana pera uva; do echo "$fruta"; done
for f in *.txt; do [[ -f "$f" ]] && procesar "$f"; done   # guard contra literal

# 2. Por rango (brace expansion, se expande ANTES del loop — sin variables)
for i in {1..10}; do ...; done
for i in {0..10..2}; do ...; done      # paso 2 (bash 4+)
for c in {a..f}; do ...; done          # letras

# 3. Estilo C (expresiones aritméticas, con variables reales)
for (( i=0, j=10; i<j; i++, j-- )); do echo "$i/$j"; done

# 4. Por array (SIEMPRE con comillas)
for elem in "${arr[@]}"; do ...; done
for i in "${!arr[@]}"; do echo "[$i]=${arr[i]}"; done   # índices
```

Verificado el output del patrón break/continue (más abajo).

### `while` — la condición manda

```bash
# Contador
n=0
while (( n < 5 )); do echo $((n++)); done

# Bucle infinito (el patrón de daemons y workers)
while true; do ...; done
while :; do ...; done          # : es el "true" builtin, idéntico

# Lectura segura de archivo — ESTE es el patrón canónico
while IFS= read -r linea; do
    procesar "$linea"
done < "$archivo"
```

> [!warning] La gotcha #1 del `while read`: la última línea sin newline
> `read` devuelve rc=1 (falso) al llegar a EOF **o** al cortar por falta de newline final — y la última línea parcial **se pierde**. Verificado:
>
> ```bash
> printf 'a\nb' > f          # sin \n final
> c=0; while read -r l; do ((c++)); done < f   # → c=1  (perdió "b")
> c=0; while read -r l || [[ -n $l ]]; do ((c++)); done < f   # → c=2 (patrón correcto)
> ```
>
> El patrón `while IFS= read -r l || [[ -n $l ]]` cubre ambos casos. Con `mapfile -t` no hay el problema del splitting pero sí se pierde la línea sin newline final — ver [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]].

### `until` — el espejo de `while`

```bash
until [[ -f "$lock_file" ]]; do sleep 1; done        # espera activa a que aparezca un archivo
until wait -n 2>/dev/null; do sleep 0.2; done        # (idea) espera a que termine un hijo
```

Ejecuta **mientras la condición sea falsa** — equivalente a `while ! cond`. Útil cuando la condición negativa se lee mejor ("hasta que el servicio responda").

### El subshell del pipe: variables que desaparecen

```bash
n=0; cat lines.txt | while read -r l; do ((n++)); done; echo "$n"
# → 0  (verificado: while corre en un subshell — su n muere con él)

n=0; while read -r l; do ((n++)); done < lines.txt          # → 2
n=0; while read -r l; do ((n++)); done < <(cat lines.txt)   # → 2 (process substitution)
```

Cada lado de un pipe es un proceso; el lado derecho de `a | while` corre en subshell y **no persiste variables, ni directorios, ni traps**. Soluciones: redirección desde archivo, process substitution `< <(cmd)`, o `mapfile`. Proceso completo del modelo detrás en [[Bash Scripting - Entrada-Salida y Redirección]].

### Iterar archivos raros: `find -print0`

```bash
while IFS= read -r -d '' archivo; do      # -d '' = separador NUL, acepta espacios/newlines
    procesar "$archivo"
done < <(find . -type f -name '*.txt' -print0)
```

`find ... -print0` + `read -d ''` es la única combinación que sobrevive a filenames con espacios, comillas, saltos de línea y backslashes. Equivalente con array: `mapfile -d '' -t arr < <(find ... -print0)`.

## Control de bucles: `break` y `continue`

```bash
for i in {1..10}; do
    if (( i == 3 )); then continue; fi    # salta solo la iteración 3
    if (( i == 8 )); then break; fi       # corta el bucle
    echo -n "$i "
done
# → 1 2 4 5 6 7   (verificado)
```

Con nivel para salir de bucles anidados (el nivel 1 es el bucle más interno):

```bash
for i in {1..3}; do
    for j in {1..3}; do
        (( j == 2 )) && break 2    # sale de AMBOS bucles
        echo "i=$i j=$j"
    done
done
```

`break`/`continue` funcionan también dentro de `case` anidado en un bucle:

```bash
while true; do
    read -r -p "comando: " cmd
    case $cmd in
        salir)  break ;;
        saltar) continue ;;
        *)      ejecutar "$cmd" ;;
    esac
done
```

## Errores comunes con las condiciones

1. **Esperar que `(( ))` devuelva 1/0 como valor**: devuelve exit codes. `(( x = 5 ))` imprime nada; hay que usar `$(( ))` para obtener el valor: `y=$(( x = 5 ))` → `y=5`.
2. **`[[ $x ]]` con `x="-n"` o `"("`** — bash puede confundir el valor con un operador. Forma segura: `[[ -n "$x" ]]` explícito siempre.
3. **`if [ "$a" == "$b" ]`** — `==` en `[ ]` no es POSIX (funciona en bash, pero no en shells estrictos). Para `[ ]` usá `=`.
4. **Comparar números con `==` en `[[ ]]`**: `[[ "07" == "7" ]]` es false, y `[[ 010 -eq 8 ]]` es false porque `-eq` en bash es decimal (a diferencia de `$((010))` que es octal). Usar consistencia en bases — ver [[Bash Scripting - Aritmética y Números Aleatorios]].
5. **Glob sin matches**: `for f in *.txt` sin `nullglob` itera el string literal `*.txt`. Guard con `[[ -f $f ]]` o `shopt -s nullglob`.
6. **`read` sin `-r`**: los backslashes de los datos se comen (`\n` → `n`). Siempre `IFS= read -r`.

## Errores comunes con `set -e` (errexit) y el control de flujo

Esto es **la** interacción sutil de Bash — detallado en [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]], pero el mapa completo:

| Contexto | ¿errexit aplica? | Verificado |
|---|---|---|
| Comando suelto (`false;`) | **Sí** — el script muere | `set -e; false; echo X` → no imprime nada |
| Condición de `if`/`elif` | **No** — la condición completa se prueba | `set -e; if false; then :; fi; echo ok` → imprime ok |
| Condición de `while`/`until` | **No** | verificado: el bucle simplemente no entra |
| Cuerpo del `if`/`while` | Sí | |
| Comando a la izquierda de `\|\|` o `&&` | No (cortocircuito) | `set -e; false \|\| true; echo ok` → ok |
| **Último** comando de un pipeline sin pipefail | Sí (solo mira el último) | `true \| false \| true` → rc 0 |
| Con `pipefail`, cualquier comando del pipeline | Sí | `set -eo pipefail; true \| false \| true` → muere |
| Dentro de `$( )` en una asignación | El rc del `$( )` manda | `x=$(false)` mata el script |
| `cmd || true` | **No** — escape controlado | patrón canónico |

> [!warning] `((contador++))` desde 0 mata el script con `set -e`
> Verificado: `set -e; ((count++)); echo never` → nunca imprime. Con `count=0`, el valor del `(( ))` es 0 → rc=1 → errexit. Soluciones: `((count+=1))`, `count=$((count+1))` (siempre rc=0 salvo overflow), o `((count++)) || true`. Es el bug #1 de scripts con `-e` y contadores.

> [!tip] Regla mental de errexit
> Bash no tiene evaluación perezosa de condiciones como otros lenguajes: tiene **excepciones de contexto**. Memorizá: *las condiciones de if/while/until y los operandos de && / || son "zonas libres de errexit"* — cualquier comando ahí puede fallar y el script sigue. Todo lo demás mata.

## Patrones avanzados

### Bucle daemon con espera interrumpible

```bash
while true; do
    procesar_cola
    sleep 5 &      # sleep en background…
    wait $!        # …para poder recibir señales durante la espera
                   # (sin &, sleep bloquea el shell y el trap recién actúa al terminar)
done
```

### Índice y valor a la vez

```bash
i=0
for elem in "${arr[@]}"; do
    printf '%d: %s\n' "$i" "$elem"
    ((i+=1))       # no ((i++)) con set -e
done
```

### Subcomandos con `case` (dispatcher)

```bash
case "${1:-}" in
    start|iniciar)   iniciar_servicio ;;
    stop|detener)    detener_servicio ;;
    status|estado)   mostrar_estado ;;
    *) printf 'uso: %s {start|stop|status}\n' "$0" >&2; exit 2 ;;
esac
```

La forma completa con validación de argumentos en [[Bash Scripting - Argumentos y Opciones]].

## Cheatsheet

| Construcción | Forma | Nota clave |
|---|---|---|
| Test moderno | `[[ ... ]]` | Sin splitting, con glob/regex |
| Test POSIX | `[ ... ]` | Solo para portabilidad |
| Regex | `[[ $x =~ $re ]]` | Patrón en var SIN comillas |
| Aritmética/bool | `(( expr ))` | rc 0 si valor ≠ 0 |
| If | `if c; then ... elif ... else ... fi` | `then` en misma línea |
| Case | `case $v in pat) ... ;; esac` | `*&` para fall-through: `;&` `;;&` |
| For lista | `for x in "${arr[@]}"` | comillas obligatorias |
| For C | `for ((i=0; i<n; i++))` | acepta variables |
| While lectura | `while IFS= read -r l; do ...; done < f` | nunca sin `-r` |
| Lectura a prueba de EOF | `while IFS= read -r l \|\| [[ -n $l ]]` | conserva última línea |
| Loop infinito | `while true; do ...; done` | `:` como alternativa |
| Salir N niveles | `break 2` / `continue 2` | 1 = interno |
| Menú | `select op in ...; do ... done` | `$REPLY` = número |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — quoting y exit codes que sostienen estas condiciones
- [[Bash Scripting - Funciones y Scope]] — predicados como funciones (`return 0/1`)
- [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]] — `mapfile`, arrays, `${!arr[@]}` para iterar
- [[Bash Scripting - Entrada-Salida y Redirección]] — por qué el pipe crea subshells y cómo leer de process substitution
- [[Bash Scripting - Procesos y Señales]] — bucles interrumpibles con `wait` y `trap`
- [[Bash Scripting - Aritmética y Números Aleatorios]] — todo el universo de `(( ))`
- [[Bash Scripting - Argumentos y Opciones]] — `case` como dispatcher de CLI
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — errexit/pipefail en profundidad
