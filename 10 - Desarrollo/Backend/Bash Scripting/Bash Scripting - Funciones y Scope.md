---
aliases: ["Bash Funciones", "Scope en Bash", "Funciones y Variables en Bash"]
tags:
  - dev/backend/bash
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Bash Scripting - Fundamentos Básicos]]"
  - "[[Bash Scripting - Control de Flujo]]"
  - "[[Bash Scripting - Argumentos y Opciones]]"
  - "[[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]]"
  - "[[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]]"
  - "[[Shellshock Vulnerabilidad en Bash]]"
fuente: "https://www.gnu.org/software/bash/manual/bash.html#Shell-Functions"
---

# Bash Scripting - Funciones y Scope

Las funciones de Bash son **objetivos de shell reutilizables**: se definen una vez en el shell actual y se ejecutan *sin crear un proceso nuevo* (a diferencia de un script externo). Esa misma naturaleza es su mayor fuente de problemas: **no tienen scope por defecto** — una variable asignada sin `local` escribe directamente en la tabla del shell que las contiene, contaminando el entorno de quien llama. Este nota cubre el modelo completo: definición, parámetros, retorno, scope, pass-by-reference con namerefs, recursión y bibliotecas.

## Definición: estilos y semántica

```bash
# 1. POSIX (portable a sh/dash): nombre()
mi_funcion() { echo "$1"; }

# 2. Keyword (bash-only): function nombre [()] — el () es opcional
function mi_funcion { echo "$1"; }
function mi_funcion() { echo "$1"; }     # estilo mixto más legible con nombres largos
```

Detalles del modelo:

- La definición se evalúa **al momento de ejecutarse la línea** — la función debe estar definida *antes* de la primera llamada (no hay "hoisting").
- Se pueden redefinir en caliente: la última definición gana.
- El **nombre oculta** al builtin/binario homónimo (`log()` es válida). `command mi_funcion` salta funciones y ejecuta el binario; `builtin echo` fuerza el builtin.
- Se introspeccionan con `declare -f nombre` (muestra el cuerpo) y se borran con `unset -f nombre` (verificado ambas).

```bash
mi_echo() { echo "custom: $*"; }
mi_echo hola        # → custom: hola
type mi_echo        # → mi_echo is a function
command -v mi_echo  # → mi_echo (command -v detecta funciones)
```

## Parámetros: `$1..$n`, `$@`, `$*`, `shift`

Dentro de una función, los parámetros posicionales son **los de la función**; al salir se restauran los del caller automáticamente (las funciones NO consumen los positional params del script — a diferencia de `shift` en el nivel principal).

```bash
saludar() {
    local nombre="${1:-Mundo}"      # default si no se pasa
    local mayuscula="${2:-}"        # segundo opcional
    printf 'Hola, %s\n' "$nombre"
}
saludar "Juan"          # → Hola, Juan
saludar                 # → Hola, Mundo
saludar "Ana" "Sí"      # → Hola, Ana

# Acceso completo igual que en main:
describir() {
    echo "cantidad: $#"
    echo "ultimo: ${@: -1}"
    echo "del 2do en adelante: ${@:2}"
    printf '[%s]\n' "$@"     # ← SIEMPRE "$@" al pasar/iterar
}
describir a "b c" d
```

Verificado: `f(){ echo "$# args"; }; set --; f "$@"` → `0 args` — pasar `"$@"` vacío pasa **cero** argumentos (no uno vacío).

> [!warning] El patrón `"$@"` es inquebrantable
> `func $@` con un arg `"a b"` llega como dos args; `func $*` llega como uno unido. Todas las funciones deben recibir con `"$@"` y las que delegan re-pasan con `"$@"`. Es la causa #1 de bugs de argumentos con espacios.

## Retorno: la convención de dos canales

Bash tiene **un solo mecanismo de retorno numérico**: el exit code (`return`), de 0 a 255. El "retorno" de datos se hace por **stdout**, capturable con `$( )`.

| Canal | Para qué | Cómo captura el caller |
|---|---|---|
| `return N` / último comando | Éxito (0) / fallo (1-255) | `if f; then` · `f \|\| die` · `rc=$?` |
| **stdout** | Los datos (strings, líneas, listas) | `resultado=$(f args)` |
| **stderr** | Logs, mensajes, prompts | visible en terminal; `2>archivo` para capturar |
| Variables globales | Múltiples valores / arrays grandes | patrón "out-param" con nameref (abajo) |

```bash
# Predicado: return code
existe_usuario() { getent passwd "$1" >/dev/null; }    # rc del getent define el resultado
existe_usuario root && echo "sí"

# Selector de datos: stdout
obtener_fecha() { date '+%Y-%m-%d'; }
hoy=$(obtener_fecha)
```

> [!warning] `$( )` se traga los newlines finales — verificado
> `f(){ printf 'dato\n\n'; }; v=$(f)` → `v` vale `dato` (los `\n` finales desaparecen). Nunca confíes en el número exacto de newlines; si importa, terminá con un sentinel o usá un out-param.

### El error de `local x=$(cmd)` — verificado

```bash
set -e
f(){ local x=$(false); echo "reached: local masks rc"; }
f
# → imprime "reached" — la asignación con local devuelve rc=0 SIEMPRE,
#   enterrando el fallo de false. Bug clásico (ShellCheck SC2155).
```

Forma correcta (dos líneas):

```bash
f(){
    local x
    x=$(comando_que_puede_fallar)   # aquí $? es el rc real, errexit actúa
}
```

### Múltiples valores: tres patrones

```bash
# 1. Out-param con nameref (bash 4.3+, el más limpio)
calcular() {
    local -n _out=$1
    _out=("$(( $2 + $3 ))" "$(( $2 * $3 ))")
}
declare -a res
calcular res 5 3
echo "suma=${res[0]} prod=${res[1]}"

# 2. key=value por stdout + source (frágil si los valores pueden contener código)
calcular_kv() { printf 'suma=%d prod=%d\n' "$(( $1 + $2 ))" "$(( $1 * $2 ))"; }
eval "$(calcular_kv 5 3)"      # SOLO si los valores son controlled; si no, parsear con read
echo "suma=$suma prod=$prod"

# 3. Asignación a variable global con nombre acordado (simple, documentado)
resultado=()
calcular_g() { resultado=("$(( $1 + $2 ))" "$(( $1 * $2 ))"); }
```

## Scope: `local` y sus trampas

Reglas del scope de variables en Bash:

1. **Sin `local`**: toda asignación toca la variable **del shell** (global). Si no existía, se crea; si existía, se pisa. Incluso en contexto de `for`/`if` (que **no** crean scope).
2. **`local`**: crea una variable que solo existe dentro de la función (y sus llamadas hijas, porque es dinámico). Se ejecuta como **comando**: solo válido dentro de funciones.

```bash
variable_global="soy global"

demo() {
    local variable_local="solo aquí"
    local tambien_visible_en_hijas      # declaración sin valor: "local unset"
    variable_global="modificada"        # escribe la global
    declare -g creada_aqui="global explícita"   # crear global desde dentro (bash 4.2+)
}

demo
echo "$variable_local"      # → vacío
echo "$variable_global"     # → modificada
echo "$creada_aqui"         # → global explícita
```

### Gotcha: `local` y las sub-llamadas (scope dinámico)

```bash
padre() { local x="del padre"; hijo; }
hijo()  { echo "hijo ve x=[$x]"; }    # → hijo ve x=del padre (scope DINÁMICO)
padre
echo "fuera=[$x]"                     # → [] (no existía globalmente)
```

Las funciones hijas **ven** las `local` de sus ancestras. Es conveniente para helpers y peligroso para colisiones: nombrá las locales con prefijo de la función o usá nombres descriptivos.

### `local` fuera de función = error, no scope

Verificado: `bash -c 'local x=1'` → `local: can only be used in a function`. No existe "scope de bloque" en Bash — ni `if`, ni bucles, ni `( )` crean scope de variable (sí `( )` crea un **subshell**, que es otra cosa: sus asignaciones se pierden al terminar, pero eso no es `local`).

### Variables internas de inspección

```bash
demo() {
    echo "nombre actual: ${FUNCNAME[0]}"        # nombre
    echo "stack completo: ${FUNCNAME[*]}"       # demo <- main <- ...
    echo "archivo: ${BASH_SOURCE[0]}"           # archivo donde está definida (≠ $0 al sourcear)
    echo "línea suya: $LINENO"                  # línea dentro de la función
    echo "línea del caller: ${BASH_LINENO[0]}"  # desde dónde se la llamó
}
demo
```

`BASH_SOURCE` + `BASH_LINENO` juntos dan el "stack trace" manual — el patrón de logging de errores de [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]].

## Pass-by-reference: namerefs (`declare -n`)

Bash 4.3+ añadió **namerefs**: variables que son *alias* de otra variable por nombre. Es el mecanismo para que una función devuelva arrays/assoc arrays sin globals feos.

```bash
swap() {
    local -n _a=$1 _b=$2
    local tmp=$_a; _a=$_b; _b=$tmp
}
x=1; y=2; swap x y; echo "x=$x y=$y"      # → x=2 y=1  (verificado)

getval() { local -n _r=$1; echo "${_r[$2]}"; }
declare -A persona=([nombre]="Ana")
echo "${persona[nombre]}"                  # funciona con asociativos (verificado)
```

Reglas y peligros:

- El **nombre** se pasa como string, la función hace `local -n _alias="$1"`. El prefijo `_` evita colisiones.
- **Auto-referencia prohibida**: `declare -n self=self` → error `nameref variable self references not allowed` (verificado). Si la variable de destino se llama igual que el nameref, bash entra en conflicto — por eso el alias con `_`.
- Los namerefs propagan la referencia: `local -n a=$1; local -n b=a` funciona, pero legibilidad decreciente.
- En `set -u`, referenciar un nameref a variable unset falla: inicializá antes (`out=()`).

## Recursión

```bash
factorial() {
    local n=$1
    if (( n <= 1 )); then echo 1
    else echo $(( n * $(factorial $((n-1))) ))     # verificado: fact 5 = 120
    fi
}

# Versión iterativa siempre es preferible en Bash:
factorial_iter() {
    local n=$1 acc=1
    while (( n > 1 )); do acc=$(( acc * n )); ((n-=1)); done
    echo "$acc"
}
```

> [!warning] Sin `local`, la recursión destruye sus propias variables
> El `n` de cada nivel pisa al del anterior: el caso base nunca se alcanza (o se alcanza con valores basura). `local` en **todas** las variables de funciones recursivas es obligatorio.

También recordá que cada nivel de `$( )` en la recursión es un **subshell/proceso**: `factorial 200` puede agotar procesos. Para recursión seria, iterá.

## Funciones como predicados y callbacks

```bash
es_par()  { (( $1 % 2 == 0 )); }        # el (( )) define el rc
no_vacio() { [[ -n "$1" ]]; }

for n in 1 2 3 4; do es_par "$n" && echo "$n es par"; done

# Dispatcher con función-predicado + tabla asociativa
declare -A HANDLERS=(
    [start]=iniciar   [stop]=detener   [reload]=recargar
)
handler="${HANDLERS[$comando]:-}"
[[ -n $handler ]] && "$handler"          # invocación indirecta por nombre
```

El patrón map/filter con callbacks está en [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]].

## Bibliotecas: el patrón `source` con guardas

```bash
# lib/mi_lib.sh
#!/usr/bin/env bash
# Guarda de carga múltiple: evita redefinir y duplicar side-effects
[[ -n "${_MI_LIB_LOADED:-}" ]] && return 0
_MI_LIB_LOADED=1

_mi_lib_die() { printf '[mi_lib] %s\n' "$*" >&2; exit 1; }   # prefijo propio: sin colisiones

trim() {
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"
    var="${var%"${var##*[![:space:]]}"}"
    printf '%s' "$var"
}
to_lower() { printf '%s' "${1,,}"; }
to_upper() { printf '%s' "${1^^}"; }
```

```bash
# En el script consumidor:
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=lib/mi_lib.sh
source "${SCRIPT_DIR}/lib/mi_lib.sh"

echo "|$(trim "  sucio  ")|"      # → |sucio|
```

Checklist de una librería correcta:

| Elemento | Por qué |
|---|---|
| Guard `_LIB_LOADED` + `return 0` | Carga idempotente |
| Prefijo en helpers internos (`_mi_lib_*`) | Evita pisar funciones del consumidor |
| `local` en **todas** las variables | Cero contaminación del caller |
| Sin side-effects al cargar (solo definiciones) | `source` no debe hacer nada más |
| Comentario `# shellcheck source=...` | ShellCheck puede seguir el source y lintear la lib |
| Nombres sin colisión o con namespace | `trim` global vs `strings_trim` — decidí y documentá |

> [!warning] `export -f`: funciones "exportadas" y Shellshock
> `export -f mi_funcion` serializa la función al entorno y la heredan los hijos bash. **Ese es exactamente el vector de [[Shellshock Vulnerabilidad en Bash]]**: bash ≤4.3 ejecutaba código oculto al importar funciones del entorno. En bash moderno está parcheado, pero la práctica sigue siendo mala idea: las funciones no viajan bien por `execve`. Pasá scripts, no funciones.

## Funciones y errexit: la interacción peligrosa

```bash
# 1. La función entera es "una condición" cuando se usa en if/&&/||:
set -e
f() { false; echo "esto NO corre (errexit mata la función)"; }
if f; then echo "f tuvo rc 0"; else echo "f falló"; fi
# → dentro de if, el errexit DENTRO de f sigue activo: false mata f, rc=1

# 2. pero el caller sobrevive porque la invocación está en contexto de condición.
# 3. Lo contrario: un error DENTRO de una función usada como condición
#    no mata el script — puede pasar desapercibido si no mirás $?:
if f; then ...; fi     # si f falla por un bug interno, solo ves el else
```

Regla práctica: las funciones deben **propagar errores explícitamente** (`|| return`, `cmd || die`) y no depender del errexit tácito — así se comportan igual con `-e` activo o no.

```bash
procesar() {
    local entrada="$1"
    [[ -f "$entrada" ]] || { printf 'no existe: %s\n' "$entrada" >&2; return 1; }
    leer "$entrada"  || return            # propagación explícita
    transformar      || return
    guardar
}
```

## Estilo y buenas prácticas

| Práctica | Razón |
|---|---|
| `local` en toda variable de función | Sin fugas ni colisiones (verificado el scope dinámico) |
| Nombrar `verbo_sustantivo`: `validar_archivo`, `leer_config` | Auto-documentado; `ls \| grep función` legible |
| Una responsabilidad por función | Testeable con bats; reutilizable |
| `local x; x=$(cmd)` en vez de `local x=$(cmd)` | No enterrar rc (SC2155) |
| Documentar contrato: `$1`, retorno, side-effects | Encima de la definición |
| `return 0` explícito al final de predicados | Claridad aunque el último comando ya lo dé |
| Prefijo `_` en aliases de namerefs y helpers internos | Anti-colisión |
| No definir funciones con nombres de builtins (`echo`, `test`) | Confusión garantizada con `builtin`/`command` |

```bash
# Encabezado documental recomendado:
# leer_config <ruta>
#   Carga key=value en CONFIG (asociativa global).
#   Retorna: 0 si el archivo existe y parsea, 1 si no.
#   Efectos: define/muta CONFIG; emite warn por línea inválida.
leer_config() { ... }
```

## Cheatsheet

| Concepto | Forma |
|---|---|
| Definición | `nombre() { ...; }` · `function nombre { ...; }` |
| Argumentos | `$1..$9` · `${10}` · `$#` · `"$@"` |
| Retorno de código | `return N` (0-255) · `rc=$?` |
| Retorno de datos | `salida=$(f)` (stdout) |
| Múltiples valores | out-param con `local -n` |
| Variable local | `local x=1` · `local x; x=$(cmd)` |
| Global explícita | `declare -g x=1` |
| Alias por referencia | `local -n _alias="$1"` |
| Introspección | `declare -f nombre` · `FUNCNAME` · `BASH_SOURCE` · `LINENO` |
| Borrar definición | `unset -f nombre` |
| Invocación indirecta | `"$handler"` con nombre en variable |
| Saltar funciones | `command ls` · `builtin echo` |
| Guarda de librería | `[[ -n "${_LIB_LOADED:-}" ]] && return 0` |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — shebang, quoting y la plantilla mínima
- [[Bash Scripting - Control de Flujo]] — funciones como predicados de `if`/`&&`/`||`
- [[Bash Scripting - Argumentos y Opciones]] — `shift` y parsing con funciones
- [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]] — map/filter/reduce sobre namerefs
- [[Bash Scripting - Patrones Comunes y Plantillas]] — estructura main + guard `BASH_SOURCE`
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — functrace, SC2155 y testing con bats
- [[Bash Scripting - Procesos y Señales]] — por qué una función NO es un proceso (y cuándo lo es)
- [[Shellshock Vulnerabilidad en Bash]] — por qué `export -f` fue tan peligroso
