---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Debugging, Buenas Prácticas y ShellCheck

Bash no tiene debugger visual ni breakpoints reales: su tooling es **`bash -n`** (parser), **`set -x`** (trace), **`trap`** (hooks de ejecución) y sobre todo **ShellCheck**, un linter estático que detecta la mayoría de los bugs de Bash antes de ejecutarlos. Este nota es el manual completo de ese stack: semántica profunda del strict mode (con sus trampas verificadas), catálogo de hallazgos de ShellCheck con fix para cada uno, guía de estilo, y el criterio para decidir cuándo Bash ya no es la herramienta.

## `bash -n`: chequeo de sintaxis sin ejecutar

```bash
bash -n script.sh        # parsea y sale. rc=0 si la sintaxis es válida
```

Verificado: error real → `syntax error near unexpected token ')'` y rc=2; comilla sin cerrar → `unexpected EOF while looking for matching '"'`. Limitación importante: **solo detecta errores de sintaxis**. `bash -n` no ejecuta nada — un script con `echo $varaible` (typo de nombre) pasa limpio. Su uso correcto: gate barato en pre-commit/CI y verificación tras edits hechos a mano.

> [!warning] `bash -n` no chequea semántica
> `[[ $x = $y ]]` con espacios en los valores, variables tipeadas mal, `set -e` y su interacción: nada de eso lo ve. Para eso está ShellCheck — abajo.

## `set -x`: el trace

```bash
set -x           # desde acá, cada comando se imprime ANTES de ejecutarse (a stderr)
echo "hola"      # → + echo hola
                 # → hola
set +x           # apagar
```

Verificado con `PS4` personalizado:

```bash
PS4='+PID:${BASHPID} L${LINENO}: '
set -x
echo hola
# +PID:92116 L1: echo hola     ← exactamente qué proceso y qué línea
```

| Técnica | Forma | Uso |
|---|---|---|
| Trace completo | `set -x` | Debug de flujo general |
| Sección acotada | `( set -x; comando1; comando2 )` | Subshell: el `set +x` es implícito |
| Formato rico | `PS4='+${BASH_SOURCE}:${LINENO}: '` | Archivo+línea de cada comando |
| Trace con funciones | `set -o functrace` (o `set -T`) | El `set -x` hereda a las funciones llamadas |
| **Trace a archivo** | `exec {fd}>trace.log; export BASH_XTRACEFD=$fd; set -x` | Verificado: el trace va al fd elegido, la salida limpia queda en stdout |
| Trace desde el arranque | `bash -x script.sh` | Sin editar el script |

`BASH_XTRACEFD` es la joya oculta: permite tener el programa escribiendo datos por stdout (pipeable) y el trace espiando por otro fd simultáneamente.

## Modo estricto en profundidad: `-e`, `-u`, `-o pipefail`

```bash
set -euo pipefail
```

### `-e` (errexit): muere si cualquier comando falla

Las **excepciones** (contextos donde errexit NO aplica) — semántica verificada en bash 5.3:

| Contexto | ¿Aplica? | Evidencia verificada |
|---|---|---|
| Comando suel | **Sí** — muere | `set -e; false; echo X` → no imprime |
| Condición de `if`/`elif` | No | `set -e; if false; then :; fi; echo ok` → ok |
| Condición de `while`/`until` | No | el bucle no entra, el script sigue |
| Operando izq. de `&&`/`\|\|` | No | `set -e; false \|\| true; echo ok` → ok |
| Cuerpo del `if`, del bucle, y el **último comando** de `&&`/`\|\|` | **Sí** | |
| Pipeline sin pipefail | Solo el último comando | `true \| false \| true` → rc 0, sobrevive |
| Pipeline **con** pipefail | Cualquier comando | `set -eo pipefail; true \| false \| true` → muere |
| Comando dentro de `$( )` en asignación | Sí (rc del `$( )` manda) | `x=$(false)` mata |
| Función invocada **como condición** | Todo el cuerpo queda **exento** | verificado abajo |

La trampa más profunda — verificada:

```bash
set -e
f()    { false; echo A; }
g()    { f || true; echo B; }
g
# → imprime A y B. ¿Por qué "A"? Porque f se invocó dentro de `f || true`:
#   el contexto de condición se PROPAGA por toda la cadena de llamadas
#   y desactiva errexit dentro de f también. "A" se imprime aunque false falló.
```

Corolario: **errexit no es un mecanismo de control de errores** (no puedes distinguir "falló" de "falló y decidimos seguir"); es un *fail-fast* para accidentes. El manejo de errores real es explícito: `cmd || manejar`, `if ! cmd; then`, `return` propagado — el patrón completo en [[Bash Scripting - Funciones y Scope]].

Escape controlado:

```bash
comando_riesgoso || true            # rc descartado a propósito
comando_riesgoso || { rc=$?; log "falló con $rc"; }   # capturado
set +e; volatil; set -e             # zona sin errexit (re-apagar SIEMPRE)
```

### `-u` (nounset): error al usar variable no definida

```bash
set -u
echo "$definida"        # → "definida: unbound variable" y muere (antes: cadena vacía silenciosa)
v=""; echo "$v"         # → vacío permitido: set -u solo mata a UNSET
echo "${opcional:-}"    # la forma de declarar opcionales: DEFAULT en la expansión
echo "${arr[@]}"        # con arr vacío: OK en bash ≥ 4.4 (antes: unbound variable;
                        #  el workaround histórico "${arr[@]:-}" ya no hace falta)
```

Patrones de opcionalidad compatibles con `-u`: `${VAR:-default}` · `${VAR:?error}` (mata con mensaje, verificado en Fundamentos) · inicializar todo al arranque del script.

### `-o pipefail`: el pipeline es tan fuerte como su eslabón más débil

```bash
set -o pipefail
echo x | false | echo y     # → y, rc=1 (verificado): el rc es el del PRIMER fallo
```

Sin pipefail, `descomprimir.log | grep -c ERROR` devuelve el rc de `wc`/`grep`: la descompresión corrupta pasa invisible. Con `-e` + pipefail, cualquier eslabón caído aborta. Excepción legítima: pipelines que **usan** rc de último por diseño (`... | grep patrón` donde "no match" es un resultado válido) → `grep ... || [[ $? == 1 ]]` (grep: 0=match, 1=sin match, ≥2=error).

### `IFS=$'\n\t'` — el cuarto caballero (y su costo)

Quita el espacio del splitting por defecto. Protege expansiones accidentales pero puede **romper** código legítimo que splittea por espacio (`read -a`, `for x in $(...)` ya malformado). Es opcional y su uso correcto está discutido en [[Bash Scripting - Fundamentos Básicos]]; alternativa quirúrgica: `IFS= read -r` por lectura.

## Trampas de debugging con `trap`

```bash
# ERR: handler que corre con el contexto del fallo (sufre las mismas excepciones que -e)
on_error() {
    printf '[DEBUG] línea %s: %s (rc=%s)\n' "$1" "$2" "$3" >&2
    printf '  stack: %s\n' "${FUNCNAME[*]}" >&2
}
trap 'on_error ${LINENO} "$BASH_COMMAND" $?' ERR

# DEBUG: corre ANTES de cada comando (costo altísimo; acotar)
trap 'printf ">> %s\n" "$BASH_COMMAND" >&2' DEBUG

# Combinación útil: ERR + xtrace solo del contexto del error
trap 'set -x; : ' ERR      # (combinar con BASH_XTRACEFD para no ensuciar stdout)
```

`$BASH_COMMAND` es el comando a punto de ejecutarse — en el trap ERR contiene el que falló. Con `set -o errtrace` (`set -E`) el trap ERR hereda a funciones y subshells (igual que functrace con -T).

## ShellCheck: el linter obligatorio

**ShellCheck** parsea scripts y detecta los bugs idiomáticos de Bash: splitting no comillado, subshells de pipe, orden de redirecciones, portabilidad rota.

### Instalación y uso

```bash
sudo apt install shellcheck        # Debian/Ubuntu
sudo dnf install shellcheck        # Fedora
brew install shellcheck            # macOS
shellcheck script.sh               # humano legible
shellcheck -f gcc script.sh        # formato GCC (IDEs/editors)
shellcheck -f json1 script.sh      # JSON para tooling
shellcheck -S warning script.sh    # severidad mínima: error|warning|info|style
find . -name '*.sh' -exec shellcheck {} +
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck script.sh   # sin instalar

# Directivas de exclusión (siempre justificadas con comentario):
# shellcheck disable=SC2086    # word splitting intencional en "$1" de un -I de xargs
cmd $intencional

# .shellcheckrc en el proyecto:
# disable=SC1091    # source no followeable en CI
```

### Catálogo de hallazgos típicos (problema → causa → fix)

| Código | Síntoma | Fix |
|---|---|---|
| **SC2086** | `Dividir $var sin comillas` — splitting+glob accidentales | `"$var"` — o `# shellcheck disable` si el splitting es intencional |
| **SC2046** | `Word splitting de $( ) sin comillas` — `for f in $(ls)` | `"$()"` entre comillas, o `while read`, o arrays (`mapfile`) |
| **SC2006** | Backticks `` `cmd` `` — anidamiento ilegible, no escapa bien | `$(cmd)` |
| **SC2002** | `cat archivo \| cmd` — proceso extra inútil | `cmd < archivo` |
| **SC2001** | `sed` para replace simple sobre variable | `${var//patrón/reemplazo}` (builtin) |
| **SC2164** | `cd` sin chequear si funcionó — sigues en el dir viejo | `cd "$d" \|\| exit` o `set -e` + no ignorar |
| **SC2155** | `local x=$(cmd)` — entierra el exit code | `local x; x=$(cmd)` |
| **SC2034** | Variable asignada pero no usada | borrar, o prefijo `_`, o export si es para hijos |
| **SC2154** | Variable referenciada pero nunca asignada (typo) | corregir el nombre; declarar si viene de env: `: "${VAR:=}"` |
| **SC2181** | `if [[ $? -eq 0 ]]` — chequeo indirecto | `if cmd; then` directo |
| **SC2064** | `trap "rm $tmp" EXIT` — expande AHORA, no al salir | `trap 'rm -rf "$TMP_DIR"' EXIT` (comillas simples) |
| **SC2068** | `for x in $array` sin comillas | `"${array[@]}"` |
| **SC2145** | `"args: $@"` mezclado con texto — pierde límites | `"args: $*"` o printf |
| **SC2086 variantes** | `eval set -- $(getopt ...)` sin comillas | `eval set -- "$(getopt ...)"` |
| **SC1090/1091** | source no resoluble en tiempo de lint | `# shellcheck source=lib/x.sh` + `-P` |
| **SC2039** | `local` en POSIX sh | cambiar shebang a bash |
| **SC2317** | Código inalcanzable tras exit/return | revisar lógica o borrar |
| **SC2206** | `arr=( $var )` — splitting no controlado | `read -ra` o `mapfile` |
| **SC2207** | `arr=( $(cmd) )` — splits la salida | `mapfile -t arr < <(cmd)` |
| **SC2199** | `[[ $arr = x ]]` — arrays en contexto escalar | `"${arr[@]}"` o índice explícito |

La regla de oro: **cada `disable=` lleva comentario de por qué**. Un disable sin justificación es deuda.

## shfmt: formateador

```bash
sudo apt install shfmt        # (o go install mvdan.cc/sh/v3/cmd/shfmt@latest)
shfmt -w -i 2 -ci script.sh   # escribe formateado: indent 2, switch cases indentados
shfmt -d script.sh            # solo diff (CI)
shfmt --language-dialect bash # forzar bash (no detecta por shebang si no hay)
```

Convención común: indentación de 2 espacios, `then`/`do` en línea propia o `; then` — el formateador estabiliza el debate.

## Guía de estilo

| Elemento | Regla | Ejemplo |
|---|---|---|
| Variables globales | `UPPER_SNAKE_CASE` | `MAX_RETRIES=3` |
| Constantes | `readonly` o `declare -r` | `readonly VERSION="1.0"` |
| Locales y funciones | `lower_snake_case` | `procesar_archivo()` |
| Funciones | `verbo_sustantivo` | `validar_entrada`, `leer_config` |
| Arrays | plural | `archivos=()`, `usuarios=()` |
| Prefijos | `_` para internals de librería | `_lib_trim` |
| Internos de nameref | `_alias` | `local -n _out=$1` |
| Comillas | `"$var"` en TODA expansión | salvo splitting/glob intencional |
| `printf` sobre `echo` | datos → printf | `printf '%s\n' "$var"` |
| Comentario | el POR QUÉ, no el QUÉ | |
| Tamaño | función < ~50 líneas, script < ~300 | si no, [[Python - Fundamentos y Sintaxis]] |

## Patrones de robustez (resumen operativo)

```bash
# Validación temprana de dependencias y entorno
for cmd in jq curl; do command -v "$cmd" >/dev/null || die "requiere: $cmd"; done
[[ -w /var/lib/miapp ]] || die "sin permiso de escritura en /var/lib/miapp"

# Limpieza garantizada (verificado: rc preservado)
cleanup() { local rc=$?; [[ -d ${TMP_DIR:-} ]] && rm -rf "$TMP_DIR"; exit "$rc"; }
trap cleanup EXIT

# Timeouts en red: acotar SIEMPRE operaciones remotas
curl -fsSL --connect-timeout 5 --max-time 30 "$URL" -o "$destino" || die "descarga falló"

# Cierre graceful ante señales (ver 128+N en Procesos y Señales)
trap 'exit 130' INT; trap 'exit 143' TERM
```

## Testing con bats

```bash
# test/mi_script.bats
#!/usr/bin/env bats

setup()    { export TEST_TMPDIR="$(mktemp -d)"; }
teardown() { rm -rf "$TEST_TMPDIR"; }

@test "procesar convierte a mayúsculas" {
    echo "hola" > "$TEST_TMPDIR/in.txt"
    run bash -c "source ../mi_script.sh; procesar '$TEST_TMPDIR/in.txt' '$TEST_TMPDIR/out.txt'"
    assert_success
    assert_equal "$(cat "$TEST_TMPDIR/out.txt")" "HOLA"
}

@test "validar falla con archivo inexistente" {
    run bash -c "source ../mi_script.sh; validar_entrada 'noexiste.txt'"
    assert_failure
    assert_output --partial "no legible"
}
```

```bash
bats test/                 # correr todo
bats --tap test/           # output TAP (CI)
bats -f "mayúsculas" test/ # filtrar
```

El patrón `source + run` funciona porque el esqueleto incluye la guard `BASH_SOURCE` — ver [[Bash Scripting - Funciones y Scope]].

## CI y hooks

```yaml
# .github/workflows/shellcheck.yml
name: shellcheck
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: find . -name '*.sh' -exec shellcheck -S warning {} +
      - run: shfmt -d .
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.10.0
    hooks: [{ id: shellcheck, args: ['--severity=warning'] }]
```

## Rendimiento: cuándo Bash es el cuello de botella

| Operación | Bash | Alternativa |
|---|---|---|
| Bucle de 10k líneas | lento (forks/builtins por línea) | `awk`, `sed` |
| `cat f \| grep p` | 2 procesos | `grep p f` |
| `$(basename "$p")` en bucle | 1 fork/iteración | `${p##*/}` (verificado ~100×) |
| `sed` sobre variable | 1 fork | `${v//a/b}` |
| `expr` / `let` externo | fork | `$(( ))` |
| Concat `s+="$x"` en bucle | O(n²) | array + join o printf directo |
| `for x in $(cat f)` | splitting + forks | `mapfile -t arr < f` |
| `while read` de 1M líneas | ~segundos | `awk` una pasada: ~milisegundos |

Regla: **Bash orquesta, las herramientas procesan**.

## Cuándo dejar de usar Bash

Señales de que superaste el umbral:

- El script pasa de **~200-300 líneas** o tiene structs/árboles/mapas de datos reales.
- Parsing serio: **JSON anidado, CSV con comillas, YAML** → [[Python - Fundamentos y Sintaxis]] (o `jq` como puente).
- **Concurrencia real**: workers con estado, colas, timeouts granulares → [[Node JS]], Python, Go.
- **Lógica numérica/científica**: decimales, matrices, estadística → Python.
- **Interfaces complejas**: TUIs, prompts ricos → Python/Node.
- **Criografía o seguridad**: nunca `$RANDOM` ni bash.

Bash sigue siendo la mejor opción para orquestar, glue, CI y automatización de sysadmin — la frontera está en la complejidad de datos y control, no en la longitud.

## Checklist pre-commit

```
[ ] bash -n pasa (sintaxis)
[ ] shellcheck -S warning limpio (o disables justificados)
[ ] set -euo pipefail + IFS revisado
[ ] "$@" y "${arr[@]}" comillados en TODOS los call sites
[ ] trap cleanup EXIT con local rc=$?
[ ] cd con || exit; redirecciones en orden (>f 2>&1)
[ ] read con IFS= y -r
[ ] usage() documenta opciones Y exit codes
[ ] constantes readonly; locales con local
[ ] printf para datos, no echo
[ ] sin expr/let legacy; sin backticks
[ ] timeouts en toda operación de red
[ ] bats para funciones críticas
```

## Errores comunes

1. **Tratar `-e` como try/catch** — no lo es: dentro de if/`||`/`&&` desapacienta. Manejo explícito.
2. **`trap "cleanup $tmp" EXIT`** (SC2064) — expande en el momento del trap, no al salir. Comillas simples.
3. **Confundir `-u` con "opcional"** — la opción es `${v:-def}`, no bajar el `-u`.
4. **Debug con `echo` pegado** — un echo de más cambia `$?` y rompe el flujo. `set -x` o log a stderr.
5. **ShellCheck "arreglado" con disable masivo** — desactivar SC2086 globalmente es peor que el bug. Exclusiones puntuales y justificadas.
6. **Testear solo el camino feliz** — bats con `assert_failure` para los caminos de error.
7. **Olvidar el pipefail en pipelines "inocentes"** — `convert | upload` sube basura si convert falla.

## Cheatsheet

| Herramienta | Comando | Detecta/hace |
|---|---|---|
| Parser | `bash -n s.sh` | Errores de sintaxis (rc=2) |
| Trace | `bash -x s.sh` · `set -x` | Cada comando antes de ejecutar |
| PS4 | `PS4='+${LINENO}: '` | Formato del trace |
| Trace a archivo | `BASH_XTRACEFD=$fd` | Separa trace de datos |
| ERR trap | `trap '...' ERR` | Contexto del fallo |
| Linter | `shellcheck -S warning s.sh` | Bugs idiomáticos |
| Formatear | `shfmt -w -i 2 s.sh` | Estilo uniforme |
| Test | `bats test/` | Tests behaviorales |
| Strict | `set -euo pipefail` | Fail-fast |
| Excepción -e | `cmd \|\| true` · `if ! cmd` | Control explícito |
| Excepción -u | `${v:-def}` · `${v:?err}` | Opcionales declarados |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — quoting, shebang y la plantilla mínima que este nota endurece
- [[Bash Scripting - Control de Flujo]] — la semántica de condiciones que hace las excepciones de `-e`
- [[Bash Scripting - Funciones y Scope]] — SC2155, stack traces con FUNCNAME, librerías testables
- [[Bash Scripting - Entrada-Salida y Redirección]] — orden de redirecciones (SC2xxx relacionadas)
- [[Bash Scripting - Procesos y Señales]] — traps EXIT/ERR/TERM y 128+N
- [[Bash Scripting - Patrones Comunes y Plantillas]] — esqueleto, cleanup y locks probados
- [[Bash Scripting - Aritmética y Números Aleatorios]] — `(( ))` vs `expr` en la tabla de performance
- [[Shellshock Vulnerabilidad en Bash]] — el caso de estudio de por qué el linter importa
- [[Python - Fundamentos y Sintaxis]] — el punto de migración
- [[Node JS]] — alternativa para lógica de servicios y concurrencia
