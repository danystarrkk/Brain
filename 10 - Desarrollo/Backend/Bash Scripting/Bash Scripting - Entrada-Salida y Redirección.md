---
tags:
  - desarrollo
  - backend
  - bash-scripting
tipo: concepto
estado: terminado
---

# Bash Scripting - Entrada-Salida y Redirección

Toda la E/S en Unix es **file descriptors** (fds): enteros que los procesos heredan y que apuntan a un canal abierto. Redirigir en Bash no es "mandar output a un archivo": es **reemplazar, duplicar o cerrar entradas de la tabla de descriptors del proceso** antes de ejecutar un comando. Con ese modelo mental, construcciones como `2>&1 > archivo` (que hace algo distinto que `> archivo 2>&1`) dejan de ser magia. Este nota cubre el modelo, toda la sintaxis, here-docs, `read`, `printf` vs `echo`, y los dispositivos especiales.

## El modelo de descriptors

| FD | Nombre | Heredado | Descripción |
|---|---|---|---|
| 0 | stdin | sí | De dónde lee el comando (teclado por defecto) |
| 1 | stdout | sí | Dónde escribe resultados |
| 2 | stderr | sí | Dónde escribe diagnósticos (NO bufferizado con stdout) |
| 3-9 | personalizados | a elección | Libres para tus canales |
| 10+ | alta | no heredados | Reservados por algunas herramientas (`flock` usa 200 por convención) |

Al invocar un comando, bash hace `fork` + `execve`; el hijo **hereda la tabla de fds del padre**. Las redirecciones se aplican **antes** del exec, sobre el proceso hijo. `exec` sin comando aplica las redirecciones **al shell actual** — y ahí está la clave de los patrones avanzados.

## Toda la sintaxis de redirección

### Salida

```bash
comando > archivo        # fd1 → archivo (crea/trunca: O_WRONLY|O_CREAT|O_TRUNC)
comando >> archivo       # fd1 → archivo en append
comando 2> archivo       # fd2 → archivo (stderr solo)
comando 2>> archivo      # fd2 append
comando > a.txt 2> e.txt # streams separados (el patrón de logs correcto)
comando > archivo 2>&1   # fd1 → archivo; fd2 → COPIA de fd1 actual → AMBOS al archivo
comando &> archivo       # atajo bash de lo anterior (bash-only)
comando &>> archivo      # atajo append de ambos
comando >| archivo       # fuerza escritura con noclobber activo (verificado: > falla, >| escribe)
comando 1>&2             # stdout → donde está stderr (encontrar errores: imprime por stderr)
comando 2>&1             # stderr → donde está stdout (mezclar en el pipe)
comando |&               # bash 4+: atajo de 2>&1 |
```

### El orden importa — verificado

```bash
out() { echo stdout; echo stderr >&2; }

out > f 2>&1     # f contiene: "stdout\nstderr"   (1→archivo, luego 2→copia de 1)
out 2>&1 > f     # stderr escapa a la terminal; f contiene solo "stdout"
```

Por qué: las redirecciones se aplican **de izquierda a derecha**. En `> f 2>&1`: primero fd1 apunta al archivo, y `2>&1` duplica el fd1 **ya redirigido**. En `2>&1 > f`: fd2 copia al fd1 **original** (terminal), y recién después fd1 va al archivo. Esta es la confusión #1 de la redirección y la razón por la que el orden canónico es siempre *"stdout primero, `2>&1` después"*.

```bash
# Intercambio de streams (swap) — verificado con captura:
out 3>&1 1>&2 2>&3 3>&-
# fd3=stdout original → fd1=stderr actual → fd2=copia de fd3(=stdout original)
# Resultado: el stdout del comando aparece por stderr y viceversa.
# Uso: error=$(comando 2>&1 >/dev/null) captura SOLO stderr en la variable.
```

### Entrada

```bash
comando < archivo        # fd0 ← archivo
comando <<< "string"     # here-string: fd0 ← "string\n" (bash añade \n siempre)
comando << EOF ... EOF   # here-document multilínea
comando 0< archivo       # forma explícita (rara vez necesaria)
read -u 3 var            # leer de un fd personalizado (verificado)
```

### `exec`: redirecciones permanentes del shell

```bash
exec 3> log.txt          # abrir fd3 → log, PARA TODO EL RESTO del script
echo "va al log" >&3     # cada comando que quiera puede escribir ahí
exec 3>&-                # cerrar fd3

# Patrón "todo al log" con recuperación — verificado:
exec 3>&1                        # fd3 guarda el stdout original
exec 1> log.txt                  # stdout del SHELL completo → log
echo "invisible en terminal"
exec 1>&3 3>&-                   # restaurar stdout y cerrar el backup
echo "visible de nuevo"
```

`exec` sin argumentos de comando y sin redirecciones es un error; con redirecciones **no crea proceso** — modifica al shell. Por eso `exec 1> archivo` al principio de un script redirige todo lo que venga después.

> [!warning] `exec` en funciones de librería
> Una `exec` que escape de una función redirige al script entero. Los patrones de logging deben encapsularse y restaurarse con `trap` — ver el patrón completo de logging dual en "Patrones avanzados".

## Here-documents (`<<`)

```bash
# Con expansión de variables y comandos (delimitador sin comillas):
greet() {
    cat << EOF
Hola $USER
Fecha: $(date +%F)
EOF
}
greet

# SIN expansión — delimitador entre comillas (plantillas, SQL, JSON, scripts):
cat << 'EOF'
Literal: $USER, $(date), `comando`, ${todo} queda tal cual.
EOF

# <<- TAB al inicio de cada línea se strippingue (para indentar dentro de bloques):
if true; then
    cat <<- EOF
	línea indentada con tab
	EOF
fi

# Here-doc como stdin de un WHILE (sin subshell):
while read -r l; do echo "got:$l"; done <<EOF
one
two
EOF
# → got:one / got:two  (verificado)
```

Casos de uso reales del delimitador literal: generar SQL/JSON/XML, escribir configs con `$`, generar otros scripts, payloads para `curl -d @-`. Si el contenido puede contener la palabra delimitadora, elegí una improbable: `<<'SQL_EOF_9f2'`.

### Here-doc → variable

```bash
read -r -d '' contenido << 'EOF'    # rc=1 porque no hay \n final que cerrar: esperado
multilínea
EOF
echo "$contenido"
```

## Here-strings (`<<<`)

Atajo de bash/ksh: convierte el string en stdin de un comando, **sin fork** (es builtin):

```bash
tr 'a-z' 'A-Z' <<< "hola"           # → HOLA
awk -F: '{print $1}' <<< "root:x:0" # → root
IFS=: read -r a b c <<< "x:y:z"     # splitting instantáneo
printf '%s\n' <<< "una sola palabra"
```

Limitación: es **una sola línea** (se añade `\n`); para multi-línea usá heredoc o process substitution.

## Process substitution: `<(...)` y `>(...)`

Bash expone la salida de un comando como un **archivo temporal con nombre tipo `/dev/fd/63`**:

```bash
diff <(ls dirA) <(ls dirB)                # comparar sin archivos intermedios
while IFS= read -r l; do ...; done < <(generador)   # while sin subshell (variables persisten)
paste <(seq 1 3) <(seq 10 12)             # inputs múltiples

comando > >(tee stdout.log) 2> >(tee stderr.log >&2)   # tee dual en vivo
```

Cada `<(...)` es un proceso con su pipe: el subshell del generador muere al consumirse. La escritora `>(...)` es asincrónica y **puede perder datos al salir el script** si el consumidor no terminó — en scripts serios preferir `wait` sobre los PIDs o tee simple.

## `tee`: duplicar un stream

```bash
comando | tee archivo.log            # stdout → pantalla + archivo
comando | tee -a archivo.log         # append
comando 2>&1 | tee todo.log          # ambos streams al archivo
comando | tee f1 f2                  # a dos archivos
comando | tee >/dev/null             # "espejo que se descarta": ejecuta el pipeline con side-effect
```

`tee` escribe en stdout **también**: para loguear sin duplicar en terminal, `| tee archivo > /dev/null`.

## `read`: la builtin de lectura

```bash
read -r linea            # SIEMPRE -r: sin él, backslash escapea (readline consume \)
IFS= read -r linea       # además sin splitting: linea conserva espacios completos
read -p "Nombre: " n     # prompt a STDERR (no se captura en la variable)
read -s -p "Clave: " c   # sin eco (ojo: queda en memoria del proceso, ps, swap)
read -n 1 -p "S/n: " y   # retorna tras 1 carácter (sin Enter)
read -N 10 buf           # 10 caracteres EXACTOS (ignora delimitadores)
read -t 10 resp          # timeout en segundos (rc>128 si expiró)
read -d '' arr           # delimitador NUL (junto a find -print0)
read -a campos           # split en ARRAY según IFS
read -u 3 x              # leer del fd 3 (verificado)
read -e                  # readline: flechas/historial (solo stdin interactiva)
```

| Flag | Para qué | Gotcha |
|---|---|---|
| `-r` | No interpretar `\` | **Ponélo siempre** — sin él `C:\path` se vuelve `C:path` |
| `-p` | Prompt | Va a stderr; no lo captures con `$( )` |
| `-s` | Sin eco | Para claves; usa `read -r -s -p` junto |
| `-n N` | N caracteres | No espera Enter |
| `-N N` | N caracteres exactos | A diferencia de `-n`, ignora IFS/delimitadores |
| `-t S` | Timeout | rc>128 al expirar; útil con `||` |
| `-d C` | Delimitador | `''` = NUL; default `\n` |
| `-a arr` | Array por splitting | pierde el campo vacío final |
| `-u FD` | Descriptor custom | p.ej. de process substitution |

```bash
# Lectura "a prueba de todo" de stdin:
while IFS= read -r -d '' linea; do procesar "$linea"; done     # terminada por NUL o EOF
# Selección con timeout:
if read -r -t 5 -n 1 -s -p "¿Continuar? [s] " respuesta && [[ $respuesta == s ]]; then
    continuar
fi
```

Los detalles de `read` en bucles (última línea sin `\n`, subshells de pipe) en [[Bash Scripting - Control de Flujo]].

## `printf` vs `echo`: por qué printf gana

`echo` es un builtin **con semántica no-estándar**: `-e`, `-n`, `-E` y el manejo de `\` dependen del shell y del `xpg_echo`. `printf` es el estándar POSIX con formato C.

```bash
# Verificado: idéntico resultado en los dos casos límite
echo -n "a"; echo -n "b"; echo        # → ab  (bash builtin; en dash: -n literal)
printf '%s' "a"; printf '%s\n' "b"    # → ab  (portable siempre)

printf '%s\n' "$var"          # imprimir valor literal — LA forma segura
printf '[%s]\n' "${arr[@]}"   # 1 línea por elemento
printf '%-20s %5d\n' "nombre" 42      # padding izq/der
printf '%05d\n' 42                    # → 00042
printf '%b\n' 'tab\there'             # interpretar escapes como echo -e
printf 'sin newline'                  # printf NO agrega \n: control total
printf '%s\n' ""                      # ← para imprimir línea vacía sin problemas
echo                                 # idéntico, pero por costumbre
```

> [!warning] `echo` con variables que empiezan por `-` o contienen `\`
> `echo "$usuario_input"` con input `-n hola` imprime sin newline; con `\t` decide según el shell. `printf '%s\n' "$entrada"` es inmune — siempre. Regla: **`printf` para datos, `echo` solo para literal hardcodeado**.

## Dispositivos especiales `/dev`

| Dispositivo | Comportamiento |
|---|---|
| `/dev/null` | Descarta todo lo escrito; lectura = EOF inmediato (exit 0) |
| `/dev/zero` | Lectura infinita de bytes `\0` |
| `/dev/random` | Entropía del sistema; puede **bloquearse** (históricamente) |
| `/dev/urandom` | Entropía sin bloqueo — para claves/uuids usá este |
| `/dev/stdin` `/dev/stdout` `/dev/stderr` | Symlinks al fd 0/1/2 (compatibilidad con herramientas que exigen rutas) |
| `/dev/fd/N` | Cualquier descriptor por nombre |
| `/dev/tcp/HOST/PORT` | Bash lo interpreta como **socket TCP** (si compiled con net redirections) |
| `/dev/udp/HOST/PORT` | Idem UDP |

```bash
cmd >/dev/null 2>&1        # descartar ambos
cmd 2>/dev/null            # silenciar solo errores
[[ -s /dev/stdin ]]        # "¿stdin tiene datos?" (comparado con test -s)
head -c 32 /dev/urandom | base64 | tr -d '\n'   # token aleatorio

# /dev/tcp para smoke-tests sin curl/nc:
if exec 3<>/dev/tcp/localhost/8080 2>/dev/null; then
    echo "puerto abierto"; exec 3>&-   # importante cerrar
fi
```

> [!warning] `/dev/tcp` no es un archivo
> Es magia del builtin de redirección de bash. No existe en `ls`, no funciona si bash fue compilado sin soporte, y solo TCP/UDP. Para producción usá `curl`, `nc` o `ss` — ver [[Comandos Básicos de Sistema y Red en Linux]].

## FIFOs (named pipes) y coprocesos

```bash
# FIFO: archivo que conecta procesos sin relación padre-hijo
mkfifo /tmp/canal
producer > /tmp/canal &          # el escritor bloquea hasta que haya lector
while IFS= read -r msg < /tmp/canal; do echo "recibido: $msg"; done
rm /tmp/canal

# coproc (bash 4+): coproceso con canal bidireccional
coproc CALC { awk '{print $1 * 2}'; }
echo 21 >&"${CALC[1]}"           # escribir a su stdin
read -u "${CALC[0]}" respuesta   # leer su stdout
echo "doble de 21 = $respuesta"
wait "$CALC_PID"
```

`coproc` expone `${NAME[0]}` (lectura) y `${NAME[1]}` (escritura); es la forma builtin de mantener un proceso vivo hablando (parsers, servidores de test). Para ICP serio entre procesos independientes: FIFOs, sockets Unix (`socat`), o BD — ver también [[Bash Scripting - Procesos y Señales]].

## Buffering: el fantasma de los pipes interactivos

Los procesos en un pipe bufferizan stdout cuando **no** es un TTY (fully buffered ≈ 4-64KB). Síntoma: `comando_largo | less` no muestra nada hasta el final, o un log en vivo "no avanza". Solución: `stdbuf` (coreutils), que fuerza el modo con `LD_PRELOAD`:

```bash
stdbuf -oL -eL comando | tee log.txt    # line-buffered ambos
stdbuf -o0 comando | menos              # sin buffer (lento, para debugging)
# Para binarios Go/static que ignoran stdbuf: unbuffer (expect) simula un TTY:
unbuffer comando | tee log.txt
```

Bash mismo escribe stdout casi sin buffer en scripts, así que el problema casi siempre es del comando externo.

## Patrones avanzados

### Logging dual (todo a log + stderr original)

```bash
setup_logging() {
    exec 3>&1 4>&2                                   # backups
    exec 1> >(tee -a "$LOG_FILE")                    # stdout → log (+ backup vía tee... no: ver nota)
    exec 2> >(tee -a "$LOG_FILE" >&2)                # stderr → log + stderr original
}
```

> [!info] Tradeoff de `tee` con process substitution
> `>(tee ...)` corre en proceso **asincrónico**: al salir el script puede perder las últimas líneas si tee no drenó. Para logs exactos usá `exec >>"$LOG_FILE"` (append simple) y mandá a terminal solo con `tee` puntual, o `wait` a los PIDs de tee antes de salir.

### Swap de streams para capturar solo errores

```bash
errores=$(comando 2>&1 >/dev/null)     # stdout va al terminal, stderr queda en la variable
```

### Escritura atómica (evitar archivos a medias)

```bash
escribir_atomico() {
    local destino="$1"; shift
    local tmp="${destino}.tmp.$$"
    printf '%s\n' "$@" > "$tmp" && mv "$tmp" "$destino"   # mv es atómico en el mismo fs
}
```

## Errores comunes

1. **`2>&1 > archivo` en vez de `> archivo 2>&1`** — el stderr se pierde en la terminal (verificado). Mnemotecnia: *primero el archivo, después `2>&1`*.
2. **`echo` para imprimir valores de usuario** — inputs con `-n`/`-e`/`\` cambian el output. `printf '%s\n'`.
3. **`read` sin `-r`** — backslashes corrompidos. Siempre `IFS= read -r`.
4. **`read -p` capturado con `$( )`** — el prompt va a stderr y no se captura; el resultado es vacío.
5. **Confundir `tee` con redirect** — `comando | tee archivo > /dev/null` es el log sin eco; `comando | tee archivo` duplica.
6. **Here-doc sin comillas con contenido que contiene `$`** — expansión no deseada. Delimitador `'EOF'` para literales.
7. **Asignar `read` en la misma línea de condición sin `||`** — `while read -r l` pierde la última línea sin newline (verificado en Control de Flujo).
8. **Olvidar cerrar `exec 3<>/dev/tcp/...`** — descriptors colgados mantienen sockets abiertos.

## Rendimiento

- `cat archivo | cmd` = 2 procesos extra. `cmd < archivo` es equivalente y más rápido (SC2002).
- `$(cat f)` para todo un archivo = fork + splitting. Usá `mapfile -t arr < f` (builtin, cero forks) — ver [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]].
- Redirecciones builtins (`>`, `<<<`, `<<`) no crean procesos; los process substitutions `<(...)` sí (uno por uso).
- `printf` es builtin; `echo | tr` para transformar un string es un fork innecesario → parameter expansion.

## Cheatsheet

| Quiero | Forma |
|---|---|
| stdout a archivo | `> archivo` · append `>> archivo` |
| stderr a archivo | `2> archivo` |
| Ambos al archivo | `> archivo 2>&1` · `&> archivo` |
| Silenciar todo | `>/dev/null 2>&1` · `&>/dev/null` |
| Silenciar errores | `2>/dev/null` |
| stderr por stdout (pipe) | `2>&1 \|` · `\|&` |
| Forzar con noclobber | `>\| archivo` |
| Entrada desde archivo | `< archivo` |
| String como stdin | `<<< "str"` |
| Multilínea literal | `<< 'EOF'` |
| Redirigir todo el script | `exec >>log 2>&1` |
| FD custom | `exec 3> f` · `echo x >&3` · `read -u 3` |
| Guardar/restore stdout | `exec 3>&1; ...; exec 1>&3 3>&-` |
| Salida de cmd como archivo | `diff <(a) <(b)` |
| Duplicar en vivo | `cmd \| tee f` |
| Leer línea segura | `while IFS= read -r l; do ...; done < f` |
| NUL-delimited | `read -d ''` con `find -print0` |
| Print exacto | `printf '%s\n' "$var"` |

## Relacionadas

- [[Bash Scripting - Fundamentos Básicos]] — quoting y expansión (los pasos que interactúan con el splitting de aquí)
- [[Bash Scripting - Control de Flujo]] — `while read`, process substitution en bucles
- [[Bash Scripting - Manipulación de Cadenas y Arrays Avanzados]] — `mapfile`, here-strings para parsing
- [[Bash Scripting - Procesos y Señales]] — quién es el dueño de los fds (fork/herencia) y FIFOs en detalle
- [[Bash Scripting - Patrones Comunes y Plantillas]] — logging con trap + exec en plantillas reales
- [[Bash Scripting - Debugging, Buenas Prácticas y ShellCheck]] — SC2002 (`cat |`) y redirecciones en debugging
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `curl`, `nc` para cuando `/dev/tcp` no basta
