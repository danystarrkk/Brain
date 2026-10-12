---
aliases:
  - Búsquedas de sistema
  - Comandos de búsqueda
  - Filtrado de archivos
  - find
  - locate
  - fd
  - ripgrep
tags:
  - sys/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Comandos, Rutas y Operadores de Linux]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
  - "[[Estructura de Directorios de Linux]]"
  - "[[Lectura de Permisos]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
fuente: "https://www.gnu.org/software/findutils/manual/html_node/find_html/"
---

# Búsquedas a nivel de sistema

Buscar en Linux tiene **cuatro problemas distintos** que se resuelven con herramientas distintas: buscar por **metadatos** (nombre, dueño, permisos, fecha, tamaño → `find`), buscar por **nombre rápido** en un índice pre-construido (`locate`), buscar por **contenido** (`grep`/`rg`) y resolver **en qué ruta del `PATH`** está un binario (`which`/`type`). Mezclar estas herramientas es el error #1: `grep -r` sobre `/` para encontrar un archivo por nombre es órdenes de magnitud más lento que `find`, y `find` para buscar texto dentro de archivos es simplemente el instrumento equivocado.

> [!info] Cómo busca `find` por dentro
> `find` hace un **recorrido en profundidad** (*depth-first walk*) del árbol de directorios desde cada ruta que le des. En cada entrada consulta el inodo vía `readdir()`/`stat()` y aplica tus tests uno a uno. Implicaciones prácticas que explican casi todos los comportamientos "raros":
> - Para **entrar** a un directorio necesitás permiso `x` (ejecutar/atravesar) en él; sin `x`, `find` no puede listarlo y emite `Permission denied` — por eso se redirige stderr.
> - El coste domina el **metadata I/O**, no el tamaño de los archivos: buscar en `/` es lento por la cantidad de inodos y porque atraviesa filesystems de red y pseudo-filesystems (`/proc`, `/sys`), no por "leer datos".
> - `find` ejecuta los tests en el orden en que los escribís: `find / -type d -prune -o -name '*.log' -print` no hace lo mismo que `find / -name '*.log' -type f -print` en costo.

## `find`: la herramienta de metadatos

### Sintaxis general

```bash
find [RUTAS...] [TESTS...] [ACCIÓN]
```

- **Rutas**: por dónde empezar el recorrido (por defecto, `.`). Varias rutas = varias búsquedas concatenadas.
- **Tests**: condiciones que evalúan cada entrada (nombre, tipo, fecha, permisos…). Varios tests se combinan con **AND lógico implícito**.
- **Acción**: qué hacer con las entradas que pasan los tests. Si no hay ninguna, la acción por defecto es `-print`.

### Ejemplos base (los del día a día)

```bash
find / -name passwd 2>/dev/null
```

Busca desde la raíz todos los archivos llamados exactamente `passwd`. La redirección `2>/dev/null` descarta los mensajes del descriptor 2 (stderr) generados al intentar leer directorios sin permiso — sin esto, la salida se inunda de `find: ‘/proc/1/task/…’: Permission denied`.

> [!tip] Redirección de errores
> `2>/dev/null` suprime stderr; `2>&1` la mezcla con stdout; `2>errores.log` la captura a archivo. En scripts de búsqueda conviene `2>/dev/null` solo cuando los errores no te interesan — en una auditoría, `2> /tmp/errores` te puede decir qué partes del árbol no pudiste leer (posible vía de evasión de detección).

```bash
find / -name passwd 2>/dev/null | xargs ls -l
```

Pasa cada resultado como argumento a `ls -l`. Funciona, pero **rompe con nombres con espacios** — la versión robusta es `find … -ls` (ya hace el `ls -l` sin salir de `find`) o `find … -print0 | xargs -0 ls -l` (ver "Errores comunes").

```bash
find / -group stark -type d 2>/dev/null
```

Todos los directorios (`-type d`) cuyo grupo propietario sea `stark`.

```bash
find / -user root -writable 2>/dev/null
```

Todo lo escrito por el usuario actual *como si fuera root*… cuidado: `-writable` evalúa **si el usuario actual puede escribir**, no si el bit `w` está puesto. Para los bits crudos usá `-perm`.

```bash
find / -name 'dex*' 2>/dev/null     # comodín entre comillas: lo interpreta find, no el shell
find / -name \*ex\* 2>/dev/null     # equivalente escapando cada comodín
```

> [!warning] Comodines: escapá o comé
> Los globbos `*`, `?` y `[]` del patrón los expande **el shell** si van sin comillas. `find / -name *.txt` busca con los archivos `.txt` del directorio actual como patrones (y falla si hay más de uno). Regla: **siempre `'patron*'` con comillas simples** o `\*` escapado.

### Tabla completa de tests

| Test | Significado | Ejemplo |
|---|---|---|
| `-name PATRÓN` | Nombre (no ruta) que coincide con glob (`*`, `?`, `[]`). Case-sensitive | `find . -name '*.log'` |
| `-iname PATRÓN` | Igual que `-name` ignorando mayúsculas | `find . -iname 'readme*'` |
| `-path PATRÓN` / `-ipath` | Coincide con la **ruta completa** (con `/`) | `find /etc -path '*/cron.*'` |
| `-regex PATRÓN` | Coincide la ruta completa contra regex (embebida por defecto) | `find . -regex '.*/[0-9]+\.txt'` |
| `-type t` | Tipo: `f` archivo regular, `d` directorio, `l` symlink, `b` bloque, `c` carácter, `p` FIFO, `s` socket | `find /dev -type c` |
| `-size N[bcwkMG]` | Tamaño. `c`=bytes, `k`=KiB, `M`=MiB, `G`=GiB. Con prefijo: `-` menor, `+` mayor | `find . -size +100M` |
| `-user NOMBRE` | Propietario (usuario o UID) | `find / -user root` |
| `-group NOMBRE` | Grupo (grupo o GID) | `find / -group staff` |
| `-uid N` / `-gid N` | Igual por número directo | `find . -uid 1000` |
| `-nouser` / `-nogroup` | Propietario/grupo **no existe** en el sistema (huérfanos) | `find /home -nouser` |
| `-perm MODO` | Permiso **exacto** (octal) | `find . -perm 644` |
| `-perm -MODO` | **Todos** los bits dados están activos (subconjunto) | `find / -perm -4000` (SUID) |
| `-perm /MODO` | **Al menos un** bit activo | `find . -perm /222` (alguien escribe) |
| `-readable` / `-writable` / `-executable` | Evaluados **contra el usuario actual**, no contra los bits | `find /etc -readable` |
| `-mtime -N` / `-mtime +N` | Última **modificación**: hace menos / más de N días | `find . -mtime -7` |
| `-atime -N` / `-atime +N` | Último **acceso** | `find / -atime +30` |
| `-ctime -N` / `-ctime +N` | Cambio de **metadatos** (chmod, chown…) | `find . -ctime -1` |
| `-mmin -N` / `-amin N` | Igual en **minutos** (granularidad fina) | `find /var/log -mmin -15` |
| `-newer REFERENCIA` | Más nuevo que el archivo de referencia (compara timestamps arbitrarios) | `find . -newer backup.tar` |
| `-empty` | Archivos vacíos o directorios sin entradas | `find . -empty` |
| `-inum N` | Archivo con ese **inodo** (para clavar el puntero en hardlinks) | `find / -inum 131075` |
| `-samefile REF` | Mismo inodo que la referencia = todos los **hardlinks** | `find / -samefile ~/.ssh/id_rsa` |
| `-links N` | Número de hardlinks exacto | `find / -links +2` |
| `-xdev` (opción global) | **No cruza** límites de sistema de archivos (no baja a `/proc`, discos montados…) | `find / -xdev -name passwd` |
| `-mount` | Sinónimo de `-xdev` | — |
| `-maxdepth N` | Limita la profundidad del recorrido | `find . -maxdepth 2` |
| `-mindepth N` | Ignora los primeros N niveles | `find . -mindepth 1` |

### Combinar tests: AND, OR, NOT y paréntesis

```bash
# AND implícito: dos tests seguidos
find . -type f -name '*.sh' -size +1k

# OR explícito con -o
find . -name '*.jpg' -o -name '*.png'

# NOT con !  (escapá la exclamación: el shell la historia)
find . -type f ! -name '*.tmp'

# Paréntesis: agrupar (escaparlos al shell)
find /var/log \( -name '*.gz' -o -name '*.old' \) -mtime +30
```

> [!warning] Precedencia en `find`
> `-a` (AND) **une más fuerte** que `-o` (OR), y `-a` tiene la misma precedencia que los tests: `find . -name '*.txt' -o -name '*.md' -type f` se lee como `('*.txt') OR ('*.md' AND tipo f)`, no como el OR de los tres. Usá `\(...\)` siempre que haya más de un `-o`.

### Acciones: qué hacer con lo encontrado

| Acción | Mecánica |
|---|---|
| `-print` | Imprime la ruta (acción por defecto si no hay otras) |
| `-print0` | Igual pero separado por **NUL** — el formato seguro para `xargs -0` |
| `-printf FORMATO` | Formato tipo `find -printf` (`%p` ruta, `%s` tamaño, `%m` modo octal, `%u` dueño, `%t` fecha…). Más rápido que `-exec printf` |
| `-ls` | Equivale a `ls -dils` en la salida — sin salir de `find` |
| `-delete` | Borra el archivo. **Implica `-depth`**: borra los hijos antes que el padre |
| `-exec CMD {} \;` | Ejecuta CMD con cada resultado como argumento. `\;` = fin de comando **por cada resultado** (un proceso por archivo) |
| `-exec CMD {} +` | Ejecuta CMD con **todos** los resultados agrupados en lotes (como `xargs`). Mucho más eficiente; el `{}` va al final |
| `-execdir CMD {} \;` | Como `-exec`, pero ejecuta **desde el directorio del archivo** (`./nombre`) — inmune a la sustitución de ruta por symlinks maliciosos |
| `-execdir CMD {} +` | Versión agrupada de `-execdir` |
| `-ok CMD {} \;` | Como `-exec` pero **pregunta** antes de cada ejecución (confirmación interactiva) |
| `-fprint ARCHIVO` / `-fls ARCHIVO` | Redirige la salida a archivo |

#### `-exec` vs `-execdir`: la diferencia de seguridad

```bash
find /datos -name '*.bak' -exec rm {} \;
# Ejecuta:  rm /datos/subdir/archivo.bak     ← ruta completa

find /datos -name '*.bak' -execdir rm {} \;
# Ejecuta:  cd /datos/subdir && rm ./archivo.bak
```

Con `-exec`, si un atacante puede reemplazar un directorio intermedio por un **symlink** apuntando a `/etc`, el `rm` con ruta absoluta puede terminar borrando el objetivo real (el clásico ataque de carrera sobre directorios world-writable). Con `-execdir` el comando siempre opera sobre `./archivo` **dentro** del directorio real recorrido. Regla de auditorías: `-execdir` siempre que el directorio pueda ser escrito por otros.

> [!tip] `\;` vs `+` — rendimiento real
> `-exec rm {} \;` lanza **un fork+exec por archivo** (10 000 archivos = 10 000 procesos). `-exec rm {} +` los agrupa en lotes que caben en `ARG_MAX` del sistema: casi siempre un solo proceso. Con `rm`, `chmod`, `chown` y `grep` usá `+` por defecto. `+` no sirve cuando necesitás que el comando se ejecute *una vez por archivo* con contexto individual (p. ej. `sh -c` con variables por archivo) — ahí `\;`.

### Recetas con `-prune`: no bajar a directorios inútiles

`-prune` "poda" el árbol: evita descender en un directorio. Es **la** optimización de rendimiento de `find`.

```bash
# Buscar en / saltándose pseudo-filesystems y montajes (equivalente práctico a -xdev)
find / \( -path /proc -o -path /sys -o -path /dev -o -path /run \) -prune -o -type f -name 'passwd' -print

# Receta idiomática completa: poda a la izquierda, test+acción a la derecha del -o
find /RUTA \( DIRECTORIOS_A_SALTAR \) -prune -o \( TESTS -print \) \;
```

Funciona porque `-prune` devuelve true y así satisface el AND implícito de la rama izquierda del `-o`, mientras la rama derecha imprime los que sí interesan. Sin los paréntesis y el `-print` explícito, `-prune` termina "imprimiendo" los directorios podados.

### Rendimiento: cuándo `find /` es una mala idea

- **`find /` recorre todo**: montajes de red (NFS lentísimos), `/proc` (cientos de miles de entradas efímeras), `/sys`, discos externos. Usá `-xdev`, `-maxdepth` o `-prune` para acotar.
- **El costo es de inodos**, no de bytes: buscar `*.log` en `/` toca millones de `stat()`. Para búsquedas repetidas de *nombres*, la base de `locate` es 100× más rápida (indexa en background con `updatedb`).
- **`grep -r` sobre binarios** pierde tiempo; `grep -rI` (`--binary-files=without-match`) o `rg` (ignora binarios por defecto) evitan el bulto.
- Para recorridos paralelos, `find` no paraleliza; `fd` y `rg` sí (usan todos los cores).

## `locate` / `mlocate`: búsqueda por índice

`locate` **no busca en el disco**: consulta una **base de datos** (`/var/lib/mlocate/mlocate.db` en distros Debian, `plocate.db` en Arch/plocate) que `updatedb` genera en background (típicamente vía cron diario o timer `plocate-updatedb.service`).

| Comando | Flags y significado |
|---|---|
| `locate passwd` | Subcadenas de la ruta que contienen `passwd` (¡no es glob por defecto, es coincidencia de subcadena!) |
| `locate -i '*.log'` | `-i`: ignora mayúsculas. Con comillas, los glob los aplica **locate** |
| `locate -c passwd` | `-c`: solo cuenta coincidencias, no las lista |
| `locate -e archivo` | `-e`: solo devuelve archivos que **existen ahora** (descarta borrados desde la última indexación) |
| `locate -S` / `locate --statistics` | Estadísticas de la base de datos |
| `locate -b '*.sh'` | `-b`: matchea solo el **componente de nombre** del final de la ruta |
| `updatedb` | Regenera la base (normalmente requiere root; el usuario solo indexa su home con `updatedb -U ~`) |

> [!warning] La base de `locate` está desactualizada
> Cualquier archivo creado/renombrado desde la última corrida de `updatedb` **no aparece**. Para "¿dónde está ahora mismo X", `find`/`fd`. Para "¿alguna vez vi un archivo con este patrón", `locate`.

## Resolver binarios: `which`, `type`, `whereis`, `whatis`, `apropos`

| Comando | Qué responde | Mecánica | Ejemplo |
|---|---|---|---|
| `which CMD` | **¿Qué ejecutará el shell** por esa línea de comandos? | Escanea `$PATH` en orden y devuelve la primera coincidencia ejecutable. No ve builtins ni aliases | `which python3` → `/usr/bin/python3` |
| `type CMD` | Como `which` pero completo: distingue *builtins*, *aliases*, *functions*, *hash* y binarios | Es builtin del shell (bash/zsh); no depende de `$PATH` | `type cd` → `cd is a shell builtin` |
| `command -v CMD` | Versión POSIX-portable de `type` (script-safe) | — | `command -v git` |
| `hash -r` | **Borra la caché** de rutas del shell | Bash cachea la ruta de cada binario ejecutado; tras instalar/eliminar paquetes, la caché puede apuntar a un fantasma | `hash -r; which foo` |
| `whereis CMD` | Binario, **fuente y manual** del comando | Busca en rutas compiladas (`/usr/bin`, `/usr/src`, `/usr/share/man`) | `whereis gcc` → binario, fuentes y man |
| `whatis CMD` | **Una línea** de descripción del manual | Lee la base de la sección 1 de `man` (`whatisdb`/`mandb`) | `whatis find` → `find (1) - search for files in a directory hierarchy` |
| `apropos PALABRA` | **Todos** los man pages** cuyo nombre/descripción contienen la palabra | Equivale a `man -k`. Busca en todas las secciones, no solo la 1 | `apropos firewall`, `apropos 'time.*zone'` |

> [!tip] `apropos` sin resultados = base desactualizada
> Las bases `whatis`/`apropos` se regeneran con `mandb` (Debian/Ubuntu) o `update-db` (Arch). Si instalaste un paquete y `apropos` no lo ve, corré `mandb` (o esperá al timer `mandb.timer`).

> [!warning] `which` vs `type`: el caso de seguridad
> En un PATH poisoning (ver [[PATH Hijacking]]) `which python3` puede devolver `/usr/local/bin/python3` porque ese directorio viene **primero** en el `$PATH` del usuario. `type` y `command -v` muestran lo mismo que ejecutará el shell *realmente* — que es lo que importa ante un binario plantado.

## Alternativas modernas: `fd` y `ripgrep`

### `fd` — find con saneamiento por defecto

[`fd`](https://github.com/sharkdp/fd) (binario en Rust; empaquetado como `fd` o `findfd` para no chocar con BSD's `fd`) mantiene la semántica de `find` pero con **defaults que coinciden con lo que la gente espera**:

| `find` | `fd` equivalente | Diferencia de comportamiento |
|---|---|---|
| `find . -name '*.log'` | `fd '\.log$'` | `fd` toma **regex**, no glob (también acepta `-e log` por extensión) |
| `find . -iname '*.log'` | `fd -i '\.log$'` | `fd` ya es case-insensitive con `-i` |
| incluye `.git/` y `target/` | **los ignora** (respeta `.gitignore`, `.ignore`, `.fdignore`) | Con `--no-ignore` se comporta como `find` |
| incluye ocultos | **excluye ocultos** por defecto | `--hidden` los incluye |
| recorre todo en un core | paralelo (todos los cores) | 3–5× más rápido en escritorios |
| `-exec rm {} \;` | `-x rm` / `fd … \| xargs rm` | `-x CMD` = ejecuta por resultado; `-X CMD` = lotea |

```bash
fd -e conf                    # todos los archivos .conf (no ignorados)
fd -t d -e git                # directorios que terminan en .git
fd --hidden --no-ignore secret  # incluye ocultos y lo que ignore gitignore
fd -x chmod 644 {} +          # lotes, como -exec ... +
```

### `ripgrep` (`rg`) — grep con defaults correctos

[`rg`](https://github.com/BurntSushi/ripgrep) busca **contenido** como `grep -r` pero: ignora binarios y `.gitignore` por defecto, es multilinea-capable, paralelo y con soporte real de regex.

| Flag | Significado |
|---|---|
| `rg PATRÓN [RUTA]` | Búsqueda recursiva por defecto (sin `-r`) |
| `-i` / `-S` | Case-insensitive / case-sensitive forzado |
| `-w` | Palabra completa |
| `-F` | Patrón literal (fixed string), sin regex |
| `-t tipo` / `-T tipo` | Filtra por tipo de archivo (`-t py`, `-t json`) — usa los `type` de ripgrep |
| `-g 'GLOB'` | Globs include/exclude: `-g '*.log'`, `-g '!vendor/**'` |
| `--hidden` / `--no-ignore` | Incluir ocultos / ignorar `.gitignore` |
| `-L` | Solo listar **archivos** con coincidencias (como `grep -l`) |
| `-c` | Conteo por archivo |
| `-n` / `-N` | Numerar líneas (por defecto sí con tty) |
| `-v` | Invertir coincidencia |
| `-A/-B/-C N` | Contexto Después/Antes/Ambos |
| `-r` / `-e PAT` | `-r` = reemplazo; `-e` = patrón adicional (varios `-e`) |
| `-z` | Separa por NUL (archivos con nombres raros, salida a `xargs -0`) |
| `--multiline` (`-U`) | El patrón puede cruzar líneas; `(?s)` hace que `.` matchee `\n` |

```bash
rg -i 'password|passwd' /etc            # contenido, case-insensitive
rg -t python 'def main'                 # solo en .py
rg -l 'BEGIN RSA' ~/.ssh                # qué archivos contienen la clave
rg --pcre2 '(?s)<script.*?</script>' page.html   # dotall con PCRE2
```

> [!info] `(?s)` y compañía pertenecen a PCRE, no a ERE
> Flags inline como `(?s)` (*dotall*), `(?i)` o `(?m)` son sintaxis **PCRE**. `grep -E` (ERE) y `grep` (BRE) no los entienden. Para usarlos: `grep -P '(?s)…'` (si grep se compiló con PCRE), `rg` (por defecto) o `perl -0777 -ne`.

## Toolkit de filtrado de texto (complementa `find`)

Cuando la búsqueda es por **contenido y columnas** de la salida, la cadena clásica es `grep` + `awk`/`cut`/`sed`/`tr`/`sort`/`uniq`. `rg` reemplaza a `grep` en lectura, pero `awk`/`sed`/`cut` siguen siendo el músculo de transformación.

### `grep`

| Parámetro | Descripción |
| :--- | :--- |
| `-r` | Búsqueda recursiva en directorios |
| `\w` | Carácter alfanumérico (en BRE/ERE de GNU: letra, dígito y `_`) |
| `-v` | Invierte la coincidencia: muestra las líneas que **NO** coinciden |
| `-E` | Expresiones regulares extendidas (ERE): `+`, `?`, `\|`, `()` con significado |
| `-P` | Expresiones regulares PCRE (donde esté compilado): lookarounds, `(?s)`, `\d`… |
| `-F` | Patrón literal, sin regex (fast: fija strings fijas) |
| `-n` | Número de línea de cada coincidencia |
| `-i` | Ignora mayúsculas/minúsculas |
| `-c` | Solo cuenta las líneas coincidentes |
| `-l` | Solo lista los **nombres de archivo** con coincidencias |
| `-w` | Coincidencia de palabra completa |
| `-o` | Imprime **solo** la parte que coincidió |
| `-z` | Trata todo el texto como una sola línea separada por NUL (grepping multilinea) |
| `-e PAT` | Patrón adicional (varios `-e` = OR) |
| `-q` | Quiet: sale con 0/1 según haya coincidencia (para `if grep -q …`) |
| `-A/-B/-C N` | Líneas de contexto Después/Antes/Ambos |

```bash
grep -rn 'TODO' src/                    # recursivo con nº de línea
grep -vE '^\s*(#|$)' config.conf       # líneas de contenido real (quita comentarios y vacías)
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' access.log   # extrae solo las IPs
grep -rlZ 'needle' . | xargs -0 rm     # nombres seguros vía NUL
```

> [!tip] BRE vs ERE vs PCRE
> - **BRE** (default): `grep 'colou*r'` — `*` sí, pero `+`, `?`, `|` y `()` son literales (escapados con `\` cobran sentido).
> - **ERE** (`-E`): `grep -E 'colou?r|color'` — operadores sin escapar.
> - **PCRE** (`-P`): lookarounds `\d \w \s`, backreferences no capturantes, flags inline `(?i)(?s)`.
> `egrep` está deprecado; usá `grep -E`.

### `awk`

| Parámetro | Descripción |
| :--- | :--- |
| `'{print $N}'` | Imprime el campo N (1-based; `$0` = línea completa) |
| `FS="X"` | Field Separator de entrada (`-F` en CLI: `awk -F: '{print $1}'`) |
| `OFS="X"` | Output Field Separator (al reconstruir con `print`) |
| `'NF{print $NF}'` | `NF` = número de campos: imprime la línea si tiene campos, y su último campo |
| `NR==N` | Number Record: número de línea global |
| `FNR==N` | Número de línea **por archivo** (útil con varios archivos) |
| `'/regex/{...}'` | Bloque condicional: actúa solo en líneas que matchean |
| `'END{print NR}'` | Bloque final: corre una vez al cerrar la entrada |

```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd        # usuarios reales (UID ≥ 1000)
df -h | awk '$5+0 > 80 {print $6, $5}'             # mountpoints > 80% usado
ps aux | awk 'NR==1 || $3 > 50 {print}'            # cabecera + procesos >50% CPU
```

### `sed`

| Parámetro | Descripción |
| :--- | :--- |
| `'s/pat/nuevo/'` | Sustituye la **primera** ocurrencia por línea |
| `'s/pat/nuevo/g'` | **Todas** las ocurrencias por línea (`g` = global) |
| `'Ns/pat/nuevo/'` | Solo en la línea N |
| `-i` / `-i.bak` | Edita in-place (con `.bak` deja respaldo) |
| `'d'` | Borra la línea (`sed '/^#/d'` quita comentarios) |
| `'/pat/!d'` | Borra todo lo que **no** matchea |
| `-E` | ERE, como en grep | 

```bash
sed 's/\t/    /g' archivo.txt            # tabs → 4 espacios
sed -E 's/([0-9]{1,3}\.){3}[0-9]{1,3}/IP/g' log   # enmascara IPs
sed -i.bak 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

### `cut`, `tr`, `sort`, `uniq`, `tee`, `head`/`tail`

| Herramienta | Flags clave | Uso canónico |
|---|---|---|
| `cut` | `-d SEP` (delimitador), `-f N` (campos), `-c N` (columnas de caracteres), `--complement` | `cut -d: -f1 /etc/passwd` |
| `tr` | `'de' 'a'` (mapeo 1:1), `-d SET` (borra), `-s SET` (comprime), `\n` → `\n` | `tr 'a-z' 'A-Z'`, `tr -d '\r'` (CRLF→LF), `tr -s ' '` |
| `sort` | `-n` numérico, `-r` invertido, `-u` unique, `-t SEP -k N` por campo, `-h` human-readable, `-V` version sort | `du -s * \| sort -rh \| head` |
| `uniq` | `-c` conteo, `-d` solo duplicadas, `-u` solo únicas, `-f N` salta campos | **Requiere `sort` previo**: `sort f \| uniq -c \| sort -rn` |
| `tee` | `-a` (append) | `comando \| tee salida.log` duplica a archivo y sigue el pipe |
| `head`/`tail` | `-n N` líneas, `+N` desde la línea N, `-f` seguir (tail), `-F` seguir por nombre (recrea el fd si rotan) | `tail -f -n 100 /var/log/syslog` |

> [!tip] `uniq` no detecta duplicados no contiguos
> `uniq` compara **líneas adyacentes**: sin `sort` previo solo elimina repetidas pegadas. El pipeline canónico de conteo es `sort archivo | uniq -c | sort -rn`.

## Errores comunes

1. **Glob expandido por el shell**: `find . -name *.log` falla con `find: ‘a.log’: No such file` o busca solo el primer archivo. Solución: `'*.log'` entre comillas simples.
2. **`xargs` con nombres con espacios**: `find . -name '*.txt' | xargs rm` rompe `mi archivo.txt`. Solución: `find . -name '*.txt' -print0 | xargs -0 rm` o directamente `find . -name '*.txt' -delete`.
3. **`find /` eterno o con basura**: recorre `/proc`, `/sys` y NFS. Solución: `-xdev`, `-maxdepth`, o `-prune` explícito.
4. **`locate` no encuentra lo recién creado**: la base está desactualizada → `find`/`fd`, o regenerar con `updatedb`.
5. **Confusión con `-perm`**: `-perm 644` pide el **modo exacto**; `-perm -644` pide que tengan *todos* esos bits (un `666` también pasa); `-perm /644` pide *algún* bit. `-perm -u+w` (notación simbólica) es más legible que buscar el octal mágico.
6. **`find -type l` no sigue symlinks**: los tests aplican al symlink mismo; `-xtype` testea el tipo del **destino**. Para búsquedas rotas: `find . -xtype l` (symlinks colgantes).
7. **`which` no ve builtins/aliases**: `which cd` puede fallar o dar un binario irrelevante; usá `type cd`.
8. **`grep -r` sobre binarios**: saca `Binary file matches` o bulto; `grep -rI` o `rg`.
9. **Borrar con `-delete` sin revisar**: `-delete` implícitamente usa `-depth` y no pide confirmación — pruebe primero sin `-delete` (la misma línea sin la acción) y con `-ok` cuando haya duda.

## Recetas del día a día

```bash
# 1. Auditar binarios SUID/SGID (puerta clásica de escalada — ver [[Privilegios SUID en Linux]])
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null

# 2. Archivos world-writable (superficie de abuso — ver [[Abuso de Permisos en Linux y LSE]])
find / -xdev -type d -perm -0002 -not -path '/proc/*' -ls 2>/dev/null   # dirs escribibles
find / -xdev -type f -perm -0002 -not -path '/proc/*' -ls 2>/dev/null   # archivos

# 3. Capabilities en binarios (alternativa al SUID — ver [[Linux Capabilities]])
getcap -r / 2>/dev/null

# 4. Archivos modificados en las últimas 24 h (¿qué tocó un intruso?)
find /etc /var/www -xdev -type f -mtime -1 -ls 2>/dev/null

# 5. Los 10 archivos más grandes de un árbol
find /var -xdev -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head

# 6. Archivos huérfanos (dueño no existe: señal de cuentas borradas)
find / -xdev -nouser -o -nogroup 2>/dev/null

# 7. Hardlinks de un archivo sensible
find / -xdev -samefile /etc/shadow 2>/dev/null

# 8. Reemplazar texto en todos los .conf de un sitio (con respaldo)
find /etc/nginx -name '*.conf' -exec sed -i.bak 's/listen 80/listen 8080/' {} +

# 9. Vaciar logs antiguos sin borrar el inodo (conserva los fds de los servicios)
find /var/log -name '*.gz' -mtime +30 -delete

# 10. Búsqueda rápida de nombre mientras escribís
fd -e py main
locate -i 'nmap.*guide'
```

## Cheatsheet

```bash
find RUTAS TESTS ACCIÓN

# Tests rápidos
-name '*.pat'  -iname '*.pat'     # nombre (glob, entre comillas)
-type f|d|l                        # tipo
-size +100M|-10k                   # tamaño
-user root  -group root            # dueño / grupo
-perm -4000  -perm /222            # SUID / alguien escribe
-mtime -7  -mmin -15  -newer ref   # fechas
-empty  -nouser  -links +2         # misceláneo
-maxdepth N  -xdev                 # alcance del recorrido

# Acciones
-print / -print0 / -ls / -printf '%p %s\n'
-exec CMD {} \;   # por archivo      -execdir CMD {} \;   # por archivo, desde su dir
-exec CMD {} +    # en lotes (rápido)  -ok CMD {} \;       # con confirmación
-delete           # borra (con -depth implícito)

# Atajos modernos
fd 'regex' [-e ext] [-t tipo] [-x CMD]        # find moderno (paralelo, respeta gitignore)
rg 'patrón' [-t tipo] [-g glob] [-F] [-L]     # grep -r moderno
which|type|command -v  CMD   # ¿qué binario ejecuta el shell?
whereis CMD;  whatis CMD;  apropos PALABRA     # binario+fuente+man / 1-línea / búsqueda en man
locate -i 'pat';  updatedb                     # índice pre-construido
```

## Relacionadas

- [[Comandos, Rutas y Operadores de Linux]] — operadores de shell, redirecciones y `xargs` en profundidad
- [[Comandos Básicos de Sistema y Red en Linux]] — catálogo de coreutils y red básica
- [[Estructura de Directorios de Linux]] — qué hay en cada ruta por la que buscas
- [[Lectura de Permisos]] — qué significan los bits que `-perm` filtra
- [[Privilegios SUID en Linux]] — el hallazgo #1 de `find / -perm -4000`
- [[Abuso de Permisos en Linux y LSE]] — world-writable, cron y sudo como vectores
- [[Linux Capabilities]] — `getcap -r` como alternativa a SUID
- [[PATH Hijacking]] — por qué `which` puede mentirte
- [[Bash Scripting - Fundamentos Básicos]] — automatizar estas búsquedas en scripts

## Referencias

- `man 1 find`, `man 1 locate`, `man 1 grep`, `man 1 awk`, `man 1 sed`, `man 1 xargs`
- GNU findutils manual: <https://www.gnu.org/software/findutils/manual/html_node/find_html/>
- fd (sharkdp): <https://github.com/sharkdp/fd>
- ripgrep (BurntSushi): <https://github.com/BurntSushi/ripgrep> y el libro *The Art of Command Line* (sección búsqueda): <https://github.com/jlevy/the-art-of-command-line>
