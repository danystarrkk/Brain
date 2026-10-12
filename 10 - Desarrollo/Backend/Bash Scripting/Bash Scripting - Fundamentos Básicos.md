---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Fundamentos Básicos

**Bash** (*Bourne Again SHell*, 1989, Brian Fox para GNU) es el intérprete de comandos y lenguaje de scripting más extendido en Linux y macOS. No es solo un "intérprete de comandos": es un lenguaje de programación completo — variables, arrays, funciones, control de flujo, manejo de errores — con una particularidad que define todo su diseño: **todo vive en un único proceso** (el shell) que ejecuta otros procesos externos, y la "cola" del programa es el flujo de bytes entre descriptors de archivo. Entender ese modelo de proceso y de expansión de tokens es la diferencia entre escribir scripts frágiles y scripts robustos.

> [!info] Para qué sirve Bash y cuándo NO
> Bash es ideal para: orquestar comandos existentes (`curl`, `awk`, `find`, `git`), automatización de sysadmin, glue code, CI. **No** es ideal para: procesamiento masivo de datos, lógica compleja con estructuras de datos, concurrencia real, criptografía. Si tu script pasa de ~200 líneas o necesita estructuras de datos ricas, migrá a [[Python - Fundamentos y Sintaxis]] o a [[Node JS]] — ver "Debugging, Buenas Prácticas y ShellCheck" para el criterio completo de "cuándo dejar Bash".

## Familia de shells: por qué `bash` y no `sh`

| Shell | Año | Nota |
|---|---|---|
| `sh` (Bourne) | 1979 | El original; sin arrays, sin `[[ ]]`, sin `$RANDOM` |
| `bash` | 1989 | Superset de sh; el estándar de facto en Linux |
| `dash` | 1997 | POSIX puro y rápido; **en Debian/Ubuntu `/bin/sh` es un symlink a dash** |
| `zsh` | 1990 | Interactivo moderno (macOS default desde Catalina); scripting compatible |
| `ksh93` | 1993 | POSIX estricto, aritmética potente |

> [!warning] El problema de `#!/bin/sh` en Debian/Ubuntu
> Si escribís un script con `#!/bin/sh` en un sistema Debian-family, **se ejecuta con dash**, no con bash. Construcciones como `[[ ]]`, `local`, arrays, `$RANDOM`, `${var,,}` o `function foo()` fallan con errores crípticos. Regla: si usás cualquier feature de bash, el shebang **debe** decir bash.

## Shebang: variantes y mecánica

El kernel, al hacer `execve()` de un archivo ejecutable, lee los primeros bytes; si empiezan con `#!`, toma el resto de la línea como intérprete y ejecuta: `interprete [arg-opcional] script args...`. El intérprete **debe ser una ruta absoluta o un binario que `env` resuelva** — el kernel no busca en `PATH`.

| Shebang | Mecánica | Cuándo usarlo |
|---|---|---|
| `#!/usr/bin/env bash` | `env` busca `bash` en el `PATH` del usuario | **Default recomendado**: respeta bash instalado vía Homebrew/managers de versión |
| `#!/bin/bash` | Ruta fija | Sistemas embebidos/containers donde `env` puede faltar o el path es garantizado |
| `#!/usr/bin/bash` | Ruta fija (Fedora/SUSE) | Cuando `/bin` es symlink a `/usr/bin` de todos modos |
| `#!/bin/sh` | Cualquier POSIX shell (dash en Debian) | Solo scripts estrictamente POSIX, sin una feature de bash |
| `#!/usr/bin/env -S bash -euo pipefail` | `-S` parte la línea en argumentos (GNU env 8.30+) | Evita el hack de "flags en el shebang" — verificar disponibilidad |
| *(sin shebang)* | El shell que invoca decide; shells interactivos suelen usar a sí mismos | Nunca en scripts serios |

```bash
# La forma "flags en el shebang" (funciona en Linux, NO en macOS/BSD):
#!/bin/bash -euo pipefail
# env -S es portable entre GNU y BSD:
#!/usr/bin/env -S bash -euo pipefail
```

> [!tip] `env` como búsqueda en PATH
> `#!/usr/bin/env bash` significa: "ejecutá el primer `bash` que encuentres en el PATH". Si tenés múltiples versiones (/Homebrew/bash en macOS, versiones en `~/.local`), el shebang con `env` las respeta. La contra: alguien puede manipular el PATH para apuntar a otro bash — en entornos hostiles preferí la ruta fija.

## Ejecución: cuatro formas, cuatro semánticas distintas

```bash
chmod +x script.sh      # habilita bit de ejecución (una vez)
./script.sh arg1        # 1. Proceso nuevo, requiere ./ (o ruta) y permiso +x
bash script.sh arg1     # 2. Proceso nuevo, NO requiere permiso +x ni ./, ignora el shebang
bash -c 'echo hola'     # 3. Ejecuta una cadena inline
source script.sh arg1   # 4. Se ejecuta EN el shell actual (también: . script.sh)
```

| Forma | Proceso nuevo | Requiere `+x` | Usa el shebang | Variables persisten en tu shell | `$0` |
|---|---|---|---|---|---|
| `./script.sh` | Sí | Sí | Sí (via kernel) | No | `./script.sh` |
| `bash script.sh` | Sí | No | **No** (lo ignora, usa bash actual) | No | `script.sh` |
| `source script.sh` / `. script.sh` | **No** | No | N/A (usa el shell actual) | **Sí** | tu shell |
| `bash -c '...'` | Sí | N/A | N/A | No | `bash` |

Verificado en vivo:

```bash
# script.sh contenido: MYVAR=42
bash script.sh;  echo "[$MYVAR]"   # → []  (proceso nuevo: la variable muere con él)
source script.sh; echo "[$MYVAR]"  # → [42] (corrió en TU shell)
```

> [!warning] `source` en `.bashrc` / scripts de entorno
> `source` ejecuta **en el shell actual**: un `exit` adentro mata tu terminal, un `cd` te cambia de directorio, una variable se cuela en tu entorno. Por eso los `source` de librerías (ver [[Bash Scripting - Funciones y Scope]]) empiezan con guardas y usan `local` agresivamente. Además, `source` busca en el `PATH` cuando no hay barra: `source mi_lib.sh` puede cargar un archivo inesperado del PATH — usá siempre `source "${SCRIPT_DIR}/lib/mi_lib.sh"`.

Por qué `bash script.sh` ignora el shebang: el shebang lo lee el **kernel** al hacer exec del archivo. Cuando invocás `bash script.sh`, el que ejecuta eres vos (bash), y bash lee el contenido como script — la línea `#!/usr/bin/env bash` queda tratada como un comentario (por eso no rompe nada).

## Anatomía de un script

```bash
#!/usr/bin/env bash                    # 1. Shebang — siempre línea 1
# Descripción: hace X con Y            # 2. Metadatos (autor, versión, uso)
set -euo pipefail                      # 3. Modo estricto (ver Debugging)
IFS=$'\n\t'                            # 4. Word splitting reforzado (opcional, ver gotcha abajo)

readonly SCRIPT_NAME="$(basename "$0")"                    # 5. Constantes
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

log()   { printf '[INFO] %s\n' "$*"; }                     # 6. Funciones
die()   { printf '[ERROR] %s\n' "$*" >&2; exit 1; }

main() {                                                    # 7. Lógica principal
    log "iniciando $SCRIPT_NAME"
}
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then                # 8. Guard de ejecución
    main "$@"
fi
```

La guardia del paso 8 es lo que hace que el script sea **a la vez ejecutable y sourceable** (para tests o para usar sus funciones desde otro script). Verificado:

```bash
# En un script con la guardia:
#   ejecutado directamente → "${BASH_SOURCE[0]}" == "${0}" → corre main
#   con source            → BASH_SOURCE es el script, $0 es tu shell → main NO corre
```

### Comentarios

```bash
# Comentario simple — explicá el POR QUÉ, no el QUÉ
echo "línea ejecutable"

: <<'COMMENT'
Bloque multi-línea comentado — : es el "true" builtin (no-op) con
un here-doc descartado. Las comillas en el delimitador evitan la
expansión, así que aunque el bloque contenga 'EOF' o $vars no pasa nada.
COMMENT

### separador de sección (greppeable: grep -n '^### ' script.sh)
```

### Estilos de definición de funciones (preview)

```bash
# Estilo POSIX (portable a sh):
mi_funcion() { echo "$1"; }
# Estilo keyword (bash-only, se lee mejor con nombres largos):
function mi_funcion_larga() { echo "$1"; }
```

Ambos son idénticos en bash salvo que `function f()` sin `()` también funciona. Detalle completo en [[Bash Scripting - Funciones y Scope]].

## Códigos de salida (exit codes)

Todo proceso termina con un **código de salida de 8 bits**: `0` a `255`. Convención POSIX: **0 = éxito, cualquier otro = error**. `$?` captura el código del último comando terminado.

```bash
false; echo $?          # → 1
true;  echo $?          # → 0
exit 300                # → el proceso sale con 44 (300 mod 256 — verificado)
```

| Código | Significado habitual |
|---|---|
| `0` | Éxito |
| `1` | Error genérico |
| `2` | Uso incorrecto (GNU convention para "mal invocado") |
| `126` | Archivo encontrado pero no ejecutable (permisos) |
| `127` | Comando no encontrado |
| `128+N` | Muerto por señal N (130 = SIGINT/Ctrl+C, 137 = SIGKILL, 143 = SIGTERM) |
| `130-158` | Idem anterior (128+2 a 128+30) — ver [[Bash Scripting - Procesos y Señales]] |

Los builtins `true` (sale con 0) y `false` (sale con 1) existen precisamente para usar como predicados en `if`/`&&`/`||`. Los códigos semánticos de BSD `sysexits.h` (`EX_USAGE=64`, `EX_DATAERR=65`, `EX_NOINPUT=66`, `EX_UNAVAILABLE=67`, `EX_SOFTWARE=70`, `EX_CANTCREAT=73`, `EX_IOERR=74`, `EX_TEMPFAIL=75`, `EX_CONFIG=78`) se usan en scripts grandes — ver el catálogo en [[Bash Scripting - Argumentos y Opciones]].

## El corazón del asunto: comillas y expansión

Bash parsea cada línea en varios **turnos de expansión**, en orden fijo:

1. **Brace expansion** — `{1..3}` → `1 2 3` (ocurre *antes* de todo, es puramente textual)
2. **Tilde** — `~/x` → `/home/usuario/x`
3. **Parámetro, command substitution, aritmética** — `$var`, `$(cmd)`, `$((x+1))`
4. **Word splitting** — el resultado del paso 3 se parte por `IFS` (espacio/tab/newline por defecto)
5. **Pathname expansion (globbing)** — `*`, `?`, `[abc]` se resuelven contra el filesystem

Las comillas intervienen en los pasos 4 y 5: **comillas dobles desactivan el word splitting y el globbing** (pero NO la expansión de `$`); comillas simples desactivan **todas** las expansiones.

### Comilla simple vs doble vs backslash (verificado)

```bash
name="world"
echo "Hola $name"     # → Hola world      (expansión activa, sin splitting)
echo 'Hola $name'     # → Hola $name      (TODO literal)
echo "Precio: \$5"    # → Precio: $5      (backslash escapea $ dentro de "")
echo 'Precio: \$5'    # → Precio: \$5     (en '' el backslash es literal)
echo "Ruta C:\Users"  # → Ruta C:Users    ("\U" no es escape válido: el \ se come)
echo "Ruta C:\\Users" # → Ruta C:\Users   (\\ → \ dentro de "")
echo -e "tab\there"   # → tab<TAB>here    (echo -e interpretra escapes)
printf 'tab\there\n'  # → tab<TAB>here    (printf siempre interpreta \t \n — portable)
```

Verificación ejecutada del splitting:

```bash
v="a  b"
printf '[%s]' $v    # → [a][b]  ← sin comillas: dos palabras (doble espacio colapsado)
printf '[%s]' "$v"  # → [a  b]  ← una sola palabra
```

### `$'...'` — comillas ANSI-C

```bash
echo $'línea1\nlínea2'   # \n real, multi-línea
echo $'con\ttab'         # tabulador
readonly NL=$'\n'        # la forma canónica de guardar un newline en una variable
```

### Los tres contextos de comillas — cuándo QUERÉS que se separe

```bash
# 1. Expansiones SIEMPRE entre comillas salvo que necesites splitting:
cp "$fuente" "$destino"          # filenames con espacios: obligatorio

# 2. "$@" SIEMPRE entre comillas: preserva cada argumento como palabra:
printf '[%s]\n' "$@"             # → [a b]\n[c]  con args "a b" "c" (verificado)

# 3. Sin comillas SOLO cuando el splitting/glob es el objetivo:
#    (leer flags de un string, expandir globs…)
for f in *.txt; do ...; done     # globbing intencional
```

> [!warning] El error #1 de todo script: `"$@"` sin comillas y `$*`
> `$@` y `$*` sin comillas se expanden a las palabras **ya splittingueadas** — un argumento `"a b"` se vuelve dos. `$*` entre comillas une todo con el primer carácter de `IFS`; `$@` entre comillas mantiene la separación. Regla inquebrantable: **`"$@"` siempre entre comillas en todo call site** (`main "$@"`, `func "$@"`). Verificado:
>
> ```bash
> set -- "a b" "c"
> printf '[%s]\n' "$@"   # → [a b] [c]   (2 argumentos, correcto)
> IFS=-; printf '[%s]\n' "$*"  # → [a b-c]  (un string unido con IFS)
> ```

### Globbing: lo que `*` no matchea

```bash
*               # cualquier secuencia (NO archivos ocultos que empiezan con .)
.*              # los ocultos (pero también . y .. — filtrar con find o dotglob)
?.txt           # exactamente un carácter
[aeiou]*.txt    # clase de caracteres
!(patrón)       # negación con extglob (shopt -s extglob)
+(patrón)       # una o más repeticiones
```

Verificado: `echo *` en un dir con `.hidden` no lista `.hidden`. `shopt -s dotglob` lo incluye; `shopt -s nullglob` hace que el glob sin matches se expanda a **nada** (evita el clásico `for f in *.txt` iterando el literal `*.txt`). Detalle completo en [[Bash Scripting - Control de Flujo]].

## Variables

### Asignación y expansión

```bash
nombre="Juan"        # SIN espacios alrededor del = (si los ponés: command not found)
edad=30
echo "$nombre"       # SIEMPRE entre comillas
echo ${nombre}       # llaves solo si viene algo pegado: ${nombre}sufijo
echo "Hola ${nombre}!"  # sin llaves, "!sufijo" se confundiría con history expansion
```

### Atributos con `declare` / `typeset`

```bash
declare -r CONSTANTE="inmutable"          # read-only (asignar de nuevo → error)
declare -i n=42                           # entero: asignaciones se evalúan como aritmética
declare -a arr=(uno dos)                  # array indexado
declare -A mapa=([k]=v)                   # array asociativo (bash 4+)
declare -l min="MAYÚSCULAS"               # guarda en minúsculas
declare -u may="minúsculas"               # guarda en mayúsculas
declare -x EXPORT_VAR="valor"             # exporta al entorno (equivale a export)
declare -p CONSTANTE                      # inspección: muestra la declaración real
```

Verificado: `declare -i n="3*7"; echo $n` → `21`. Atributos completos de arrays en [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]].

### Variables de entorno vs del shell

Una variable **del shell** solo existe en este proceso. `export` la marca para que los **hijos** (`execve`) la hereden:

```bash
MIA="local"                       # no ven los hijos
export COMPARTIDA="global"        # la ven todos los procesos hijos
env | grep ^COMPARTIDA            # inspeccionar entorno exportado
MI_COMANDO=1 comando_especial     # solo para la duración de ese comando (verificado: no persiste)
```

### Variables especiales (las que hay que conocer de memoria)

| Variable | Significado | Ejemplo |
|---|---|---|
| `$0` | Nombre del script como fue invocado | `./script.sh` |
| `$1..$9`, `${10}` | Argumentos posicionales | `script.sh a b` → `$1=a` |
| `$#` | Cantidad de argumentos | `2` |
| `"$@"` | Argumentos como **array** (uno por palabra, con comillas) | uso canónico |
| `$*` | Argumentos como **string** unido por IFS | rara vez deseado |
| `$?` | Exit code del último comando | `if ! cmd; then ...` |
| `$$` | PID del shell | `log_file="/tmp/x.$$"` |
| `$BASHPID` | PID real del subshell (≠ `$$` dentro de `( )` — verificado) | debugging |
| `$!` | PID del último proceso background | `wait $!` |
| `$_` | Último argumento del comando anterior | auto en shells |
| `BASH_SOURCE` | Archivo donde está el código (no `$0` al sourcear) | guard de main |
| `LINENO` | Línea actual | logging |
| `FUNCNAME` | Stack de funciones (array) | traces |
| `RANDOM` | Pseudoaleatorio 0-32767 | [[Bash Scripting - Aritmética y Números Aleatorios]] |
| `SECONDS` | Segundos desde arranque del shell (resettable) | timers |
| `IFS` | Separador de word splitting (default: space/tab/newline) | parsing |

## PATH: cómo resuelve bash los comandos

Cuando escribís un comando, bash busca en este orden: **1) builtins** (`cd`, `echo`, `printf`, `test`) → **2) funciones** del shell → **3) el `PATH`**, de izquierda a derecha, ejecutando el primer match. `PATH` es una lista de directorios separados por `:`.

```bash
echo "$PATH"
command -v ls          # → /usr/bin/ls  (builtin POSIX: resuelve builtin/función/binario)
type cd                # → cd is a shell builtin (muestra el tipo real)
which ls               # → externo, puede no existir y no distingue builtins/functions
```

> [!warning] Seguridad del PATH
> 1. **Nunca** incluyas `.` (directorio actual) en el PATH: cualquier archivo con el nombre de un comando que "bajes" de internet se ejecuta primero.
> 2. El orden importa: si `/usr/local/bin` va antes de `/usr/bin`, un binario ahí **pisa** al del sistema. Los atacantes que logran escribir en un directorio del PATH ganan ejecución de código.
> 3. En scripts que se ejecutan como root o desde cron, exportá un PATH fijo: `export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"` — ver patrones de cron en [[Cron Jobs y Programación de Tareas]].

Los scripts que invocan a otros scripts deben referenciarlos por ruta derivada del script, no del PATH:

```bash
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
"${SCRIPT_DIR}/lib/helper.sh"    # ejecuta el hermano sin depender del PATH
```

La estructura de directorios del sistema (dónde vive `/usr/bin`, `/etc`, `/var`) está cubierta en [[Estructura de Directorios de Linux]].

## Modo estricto: `set -euo pipefail` (introducción)

```bash
set -euo pipefail
```

| Flag | Efecto | Sin él |
|---|---|---|
| `-e` (errexit) | El script **muere** si un comando falla (rc ≠ 0) | Los errores se ignoran y el script sigue escribiendo sobre un estado corrupto |
| `-u` (nounset) | Error "unbound variable" al usar variable no definida | `$var` no seteada expande a cadena vacía silenciosamente |
| `-o pipefail` | El pipeline falla si **cualquier** de sus comandos falla | Solo se mira el último: `descomprimir | grep patrón | wc -l` "funciona" aunque descomprimir falle |

Verificado el riesgo de no usarlo:

```bash
bash -c 'set -eo pipefail; true | false | true; echo never'
# → el script muere: pipefail hace que el rc del pipeline sea 1
```

Verificado el gotcha más famoso del `-e` (los detalles y las 9 excepciones están en [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]]):

```bash
bash -c 'set -e; ((count++)); echo not-reached'
# → muere: con count=0, ((count++)) evalúa a 0 y devuelve rc=1 → errexit lo mata
```

> [!tip] `IFS=$'\n\t'` — con pinzas
> Poner `IFS=$'\n\t'` (quitar el espacio del splitting por defecto) protege contra word splitting accidentales, pero **rompe** patrones que dependen del espacio: `for f in $(ls)` ya no parte por espacios... pero ese código estaba mal igual. La alternativa más segura y quirúrgica es `IFS= read -r linea` por lectura — ver [[Bash Scripting - Entrada-Salida y Redirección]].

## Terminal: colores y formato

```bash
# Códigos ANSI: \e[...m  (o \033[...m)
reset=$'\e[0m'; bold=$'\e[1m'; red=$'\e[31m'; green=$'\e[32m'
yellow=$'\e[33m'; blue=$'\e[34m'; cyan=$'\e[36m'
bg_red=$'\e[41m'; b_green=$'\e[92m'    # bright: 90-97 fg, 100-107 bg

printf '%sERROR%s: falló todo\n' "$red" "$reset"      # printf %b o $'' para interpretar \e
printf '%bERROR%b\n' "\e[31m" "\e[0m"                # equivalente con %b
```

**`tput` es la forma portable** (lee el terminfo del sistema, funciona en terminales exóticas):

```bash
tput setaf 1      # color fg rojo (números en vez de códigos crudos)
tput sgr0         # reset (más robusto que \e[0m en algunos terminales)
tput civis        # ocultar cursor;  tput cnorm # restaurar
tput cup 10 20    # mover cursor a fila 10, col 20
cols=$(tput cols); lines=$(tput lines)
```

Y la detección obligatoria antes de emitir color:

```bash
if [[ -t 1 && -z "${NO_COLOR:-}" ]]; then   # -t 1 = stdout es un TTY
    readonly RED=$'\e[31m' RESET=$'\e[0m'
else
    readonly RED="" RESET=""                 # pipe/log: sin secuencias
fi
```

La convención `NO_COLOR` está adoptada de facto por GNU coreutils, curl, grep y muchos más — respetala en tus scripts.

## Entrada del usuario (preview)

```bash
read -p "Nombre: " nombre                 # prompt a stderr, dato a la variable
read -s -p "Clave: " clave; echo          # sin eco (la clave queda en memoria)
read -n 1 -p "¿Sí? [s/n] " yn; echo      # un solo carácter
read -t 10 -p "Rápido: " resp || echo "timeout"   # con límite de tiempo
IFS= read -r linea                        # línea COMPLETA sin splitting ni backslash-mangling
```

`read` es un mundo (flags `-d`, `-N`, `-a`, `-u`, `-e`): catálogo completo y por qué **`read` siempre con `-r`** en [[Bash Scripting - Entrada-Salida y Redirección]].

## Plantilla mínima recomendada

Para scripts serios usá la plantilla completa de [[Bash Scripting - Patrones Comunes y Plantillas]] (trap de cleanup, usage, colores condicionales, main con guard). La versión mínima que cubre el 80%:

```bash
#!/usr/bin/env bash
set -euo pipefail

die() { printf '[ERROR] %s\n' "$*" >&2; exit 1; }

main() {
    local arg="${1:-}"
    [[ -n "$arg" ]] || die "uso: $0 <argumento>"
    printf 'hola %s\n' "$arg"
}

main "$@"
```

## Errores comunes

1. **`#!/bin/sh` con features de bash** — en Debian corre dash y explota en `[[ ]]`/arrays. Shebang con bash explícito.
2. **`variable = "valor"` con espacios** — bash lo interpreta como comando `variable` con args `=` y el valor. Sin espacios.
3. **`rm $archivo` sin comillas** — con `archivo="mis archivos.txt"` borra dos cosas distintas (o nada). `"$archivo"` siempre.
4. **`source script.sh` esperando que no afecte al shell** — afecta siempre: variables, `cd`, `exit`. Es la semántica, no un bug.
5. **`./script.sh: command not found`** — falta `+x` (`chmod +x`) o falta el `./` (el PATH no incluye `.` — y con razón).
6. **`$?` leído después de un `echo` intermedio** — `$?` es del **último** comando; guardalo de inmediato: `cmd; rc=$?`.
7. **Ignorar el word splitting de `$( )` sin comillas** — `for x in $(ls)` parte por espacios y quiebra filenames. `find ... -print0 | while IFS= read -r -d ''`.
8. **`echo` para datos** — `echo` con flags (`-e`, `-n`) y escapes es portabilidad frágil; `printf '%s\n' "$var"` es la forma segura.

## Seguridad

- **Quoting = primer muro de inyección**: `grep $usuario archivo` con `usuario='-r / -e rm'` ejecuta lo que el atacante quiera (argument injection). Siempre `grep -- "$usuario"` o `"$var"` entre comillas y `--` antes de valores que podrían empezar con `-`.
- **`eval` y `source` de contenido no confiable = ejecución de código**. Nunca hagas `eval "$respuesta_de_red"`.
- **`[[ $x == $patron ]]`**: si el patrón viene de usuario, puede matchear más de lo esperado (es glob). Para comparación exacta usá `[ "$x" = "$patron" ]` o escapá el patrón.
- Los scripts que corren tareas alineadas con eventos del sistema deben endurecerse: ver [[Shellshock Vulnerabilidad en Bash]] (inyección vía variables de entorno en funciones exportadas) y los patrones de [[Cron Jobs y Programación de Tareas]].
- Validar todo input de usuario antes de usarlo en rutas de filesystem: `"../../etc/passwd"` es traversal.

## Rendimiento

- Bash es lento (~10-50k iteraciones simples/seg): los bucles intensivos en Bash puro son 100-1000× más lentos que `awk`/`sed`. Regla: **control en Bash, datos en herramientas** — ver tabla completa en [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]].
- Cada `$( )` o subshell `( )` es un fork: en bucles de miles de iteraciones, evitalos.
- `source` de librerías grandes al inicio de cada invocación tiene costo; para CLIs frecuentes, considerá binarios compilados.

## Cheatsheet

| Concepto | Forma | Nota |
|---|---|---|
| Shebang portable | `#!/usr/bin/env bash` | Busca bash en PATH |
| Ejecutar | `./s.sh` · `bash s.sh` · `source s.sh` | source = en tu shell |
| Modo estricto | `set -euo pipefail` | Ver Debugging para excepciones |
| Variable | `x="v"` · `echo "$x"` | Sin espacios, con comillas |
| Constante | `readonly X="v"` o `declare -r` | Error al reasignar |
| Entero auto-expr | `declare -i n=1+1` | `n=2` |
| Arg array seguro | `"$@"` | Siempre entre comillas |
| Cantidad args | `$#` | |
| Exit code | `$?` · `exit N` | 0=ok, 128+N=murió por señal N |
| Comando encontrado | `command -v cmd` | POSIX, detecta builtins |
| Literal $ | `'...'` · `\$` en `"..."` | |
| Multi-línea literal | `$'a\nb'` | ANSI-C quoting |
| Glob sin matches | `shopt -s nullglob` | Evita iterar el literal |
| Color si TTY | `[[ -t 1 ]]` | + convención NO_COLOR |

## Relacionadas

- [[Bash Scripting - Control de Flujo]] — `if`/`[[ ]]`/bucles: la lógica que va encima de estos fundamentos
- [[Bash Scripting - Funciones y Scope]] — modularizar scripts con `local` y exit codes
- [[Bash Scripting - Argumentos y Opciones]] — `$@`/`getopts` en profundidad
- [[Bash Scripting - Entrada-Salida y Redirección]] — el modelo de descriptors detrás de `>`/`|`/`$( )`
- [[Bash Scripting - Procesos y Señales]] — qué pasa realmente cuando ejecutás un script
- [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]] — parameter expansion completo
- [[Bash Scripting - Aritmética y Números Aleatorios]] — `(( ))`, `$RANDOM`, `bc`
- [[Bash Scripting - Patrones Comunes y Plantillas]] — plantilla profesional para copiar
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — strict mode en profundidad y linting
- [[Comandos Básicos de Sistema y Red en Linux]] — los comandos que estos scripts orquestan
- [[Estructura de Directorios de Linux]] — FSBD y rutas donde viven los binarios
- [[Cron Jobs y Programación de Tareas]] — cómo corren tus scripts sin supervisión
- [[Shellshock Vulnerabilidad en Bash]] — por qué el quoting y el entorno importan tanto
