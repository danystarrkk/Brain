---
aliases:
  - Comandos Linux
  - Rutas Linux
  - Operadores Linux
  - Bash Comandos
  - PATH
  - Redirecciones Bash
tags:
  - sys/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
  - "[[Búsquedas a nivel de sistema]]"
  - "[[Estructura de Directorios de Linux]]"
  - "[[Lectura de Permisos]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
  - "[[PATH Hijacking]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Node JS]]"
fuente: "https://www.gnu.org/software/bash/manual/bash.html"
---

# Comandos, Rutas y Operadores de Linux

Tres mecánicas que gobiernan **todo** lo que se teclea en una terminal: cómo el shell encuentra los binarios (**`$PATH`**), cómo se interpretan las **rutas** de archivo, y qué hacen los **operadores** (`|`, `&&`, `>`, `$( )`…) antes de que ningún comando corra. Entender esto convierte comandos memorizados en comandos deducidos.

> [!info] El orden de vida de una línea de comandos
> Cuando escribís `cat *.log | grep ERR > out.txt 2>&1`, el shell procesa la línea en este orden, **antes** de ejecutar nada:
> 1. **Parseo**: separa tokens, aplica reglas de quoting.
> 2. **Expansiones**, en orden: `{}` → `~` → variable/`$( )` → aritmética `$(( ))` → splitting de palabras → globbing `*?[]`.
> 3. **Redirecciones**: monta los descriptores (`>`, `2>&1`…).
> 4. **Ejecución** de izquierda a derecha por los operadores de pipeline/`&&`/`;`.
> Casi todos los bugs de shell son bugs de expansión (globbing no quotado, splitting sin querer, `$( )` vs comillas) — no del comando final.

## `$PATH`: cómo el shell encuentra tus comandos

### La mecánica

`$PATH` es una lista de directorios separados por `:`. Al teclear `ls`, el shell:

1. Consulta sus **aliases** y **funciones** primero.
2. Si no, busca el binario **en orden** dentro de cada directorio de `$PATH` (el primero que encuentre **ejecutable** gana).
3. Si nada coincide: `command not found`.

```bash
echo $PATH
# Ejemplo típico:
# /home/stark/.local/bin:/usr/local/bin:/usr/bin
```

| Variable | Rol |
|---|---|
| `$PATH` | Dónde buscar ejecutables |
| `$IFS` | Separadores de splitting de palabras (por defecto espacio/tab/newline) |
| `$HOME` | Base de `~` |
| `$PWD` / `$OLDPWD` | Directorio actual / anterior (los usa `cd -`) |
| `$SHELL` | Shell por defecto de la cuenta |
| `$PATH` vacío o con `.` | Peligro: busca en el **directorio actual** (ver [[PATH Hijacking]]) |

### Recetas

```bash
# Añadir un directorio solo para esta sesión (al FINAL: menos precedencia)
export PATH="$PATH:$HOME/.local/bin"

# Añadirlo al principio (máxima precedencia — con cuidado)
export PATH="$HOME/bin:$PATH"

# Hacerlo permanente: ~/.bashrc (usuario) o /etc/profile.d/*.sh (sistema)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Auditar PATHs vacíos o con '.'
echo "$PATH" | tr ':' '\n' | grep -nE '^$|^\.$'   # hallazgos = riesgos
```

> [!warning] `$PATH` orden = orden de precedencia
> El **primero** directorio que contiene un binario con ese nombre **gana**. Si un atacante logra escribir en un directorio que viene antes que `/usr/bin` (por ejemplo vía un servicio que expande el PATH mal, o tu propio `~/bin` si clonaste un repo ahí), puede plantar un `ls` falso. Es el ataque completo en [[PATH Hijacking]]. Por eso los cron jobs y servicios **usan rutas absolutas**.

> [!info] `hash`, `type` y la caché del shell
> Bash cachea la ruta resuelta de cada binario (`hash -r` la limpia). Tras instalar/actualizar un paquete, si `comando` da "not found" o parece viejo: `hash -r`. `type comando` muestra builtins/aliases/funciones además de binarios — `which` solo mira `$PATH` (ver [[Búsquedas a nivel de sistema]]).

### Cómo llega un binario al `$PATH`

| Instalación | Destino típico | ¿En `$PATH`? |
|---|---|---|
| Gestor de paquetes | `/usr/bin` | Sí, siempre |
| `make install` | `/usr/local/bin` | Sí (tras `/usr/bin`: los tuyos pisan los del sistema) |
| Instalación de usuario | `~/.local/bin` | Lo agregás vos (Convención XDG) |
| Software de terceros | `/opt/<app>/bin` | Hacés symlink a `/usr/local/bin` o agregás al PATH |
| Snap/Flatpak | `/snap/bin`, `/var/lib/flatpak/exports/bin` | Gestionados por el paquete |

## Rutas: absoluta, relativa y resolución de symlinks

| Tipo | Forma | Ejemplo | Se resuelve contra |
|---|---|---|---|
| **Absoluta** | empieza con `/` | `/etc/ssh/sshd_config` | la raíz del filesystem |
| **Relativa** | no empieza con `/` | `docs/notas.md` | `$PWD` (directorio actual) |
| Parent | `..` | `../fotos` | el directorio **padre** del actual |
| Actual | `.` | `./script.sh` | el directorio actual (fuerza ejecución local) |
| Home | `~` o `~/x` | `~/.bashrc` | `$HOME` del usuario (`~otro` = home de otro) |
| Home ajeno | `~user` | `~root/.ssh` | `/root` (vía getpwuid — no literal) |

### La ruta canónica: `pwd -P` vs `pwd -L` y `readlink -f`

Un path puede describir **varias rutas que terminan en el mismo archivo** si hay symlinks en medio: `/var/log` en muchas distros es enlace a `/var/log` real dentro de otra partición, y `/bin` → `/usr/bin` (verificado en este host: `bin`, `sbin`, `lib`, `lib64` son symlinks a `usr/`). La **ruta lógica** (`pwd -L`) es la que te llevó allí; la **ruta física** (`pwd -P`) resuelve todos los symlinks.

```bash
cd /bin && pwd -L          # /bin          ← ruta lógica
cd /bin && pwd -P          # /usr/bin      ← ruta física canónica
readlink -f ./script.sh    # resuelve TODOS los symlinks → ruta absoluta canónica
readlink -m ruta           # igual pero no exige que exista la última componente
realpath --relative-to=$PWD /etc/hosts   # ruta absoluta expresada relativa al cwd
ls -l /bin                 # muestra a dónde apunta cada symlink: /bin -> usr/bin
```

> [!tip] `readlink -f` en scripts
> Es el estándar para obtener la ruta propia del script (para sourced files y configs relativas):
> ```bash
> SCRIPT_DIR="$(cd "$(dirname "$(readlink -f "$0")")" && pwd)"
> ```
> El doble `cd … && pwd` normaliza también el caso de que el script se ejecute desde otro directorio.

### Reglas de resolución del kernel

1. Cada componente se resuelve **de izquierda a derecha**.
2. Un symlink se expande **en el punto exacto** donde aparece (no al final): si `a/b` es symlink a `c`, la ruta `a/b/d` busca `d` **dentro de `c`**.
3. Se permiten symlinks relativos: se resuelven contra el directorio **donde está el symlink**, no contra `$PWD`.
4. `.` y `..` se resuelven *antes* de mirar symlinks en rutas canónicas (por eso `/tmp/../etc` pasa por la raíz, no por el destino de `/tmp`).
5. Sin permiso `x` (search) en un componente, la resolución falla con `Permission denied` aunque el archivo final sea legible.

## Catálogo de operadores del shell

### Control de flujo

| Operador | Significado | Ejemplo real |
|---|---|---|
| `cmd1 ; cmd2` | Ejecuta cmd1 y luego cmd2 **siempre** (aunque cmd1 falle) | `cd /var/log ; ls` |
| `cmd1 && cmd2` | cmd2 solo si cmd1 **exit 0** | `make && make install` |
| `cmd1 \|\| cmd2` | cmd2 solo si cmd1 **falla** (exit ≠ 0) | `command -v fzf \|\| echo "fzf ausente"` |
| `! cmd` | Invierte el código de salida | `! grep -q pat f` = true si NO está |
| `a && b \|\| c` | "b si a, c si a falló" — **no** es "b si a, si no c" (si b falla también corre c) | Trampa clásica; usar `if/then` para lógica real |

### Redirecciones

| Sintaxis | Descriptor | Significado |
|---|---|---|
| `> archivo` | 1 (stdout) | Escribe y **trunca** (borra el contenido previo) |
| `>> archivo` | 1 | Escribe **añadiendo** al final |
| `> /dev/null` | — | Manda la salida al "pozo negro" del sistema |
| `2> archivo` | 2 (stderr) | Errores a archivo (trunca) |
| `2>> archivo` | 2 | Errores añadiendo |
| `2>/dev/null` | 2 | Silencia errores |
| `&> archivo` | 1+2 | **Todo** a archivo (bash; trunca) |
| `>& archivo` | 1+2 | Sinónimo bash de `&>` |
| `&>> archivo` | 1+2 | Todo añadiendo |
| `2>&1` | — | Redirige stderr **al mismo destino** que stdout tiene ahora |
| `1>&2` | — | Al revés: stdout al canal de errores (útil en wrappers) |
| `<<< "texto"` | 0 (stdin) | *Here-string*: stdin desde una cadena literal |
| `<< EOF … EOF` | 0 | *Here-document*: stdin multilinea (permite variables: `<< EOF` vs `<<'EOF'` literal) |
| `< archivo` | 0 | Lee stdin desde archivo |

> [!warning] El orden importa: `2>&1 > f` ≠ `> f 2>&1`
> Las redirecciones se aplican **de izquierda a derecha**:
> - `> f 2>&1` → stdout va a `f`, y después stderr va *donde va stdout ahora* → todo a `f`. ✔
> - `2>&1 > f` → stderr va a donde stdout estaba (**la terminal**), y después stdout va a `f` → los errores quedan en pantalla. ✘
> Por eso `&> f` (o `> f 2>&1`) es la forma segura. En `$( )` y pipes, `2>&1` va **después** del `>` correspondiente.

### Sustitución de comandos y expansiones

| Sintaxis | Qué hace | Ejemplo |
|---|---|---|
| `$(comando)` | Sustitución por la **salida** del comando (anidable, moderna) | `echo "hoy es $(date +%F)"` |
| `` `comando` `` | Igual, forma antigua: **no** anida ni se anida bien | ``echo `date` `` — evitar en código nuevo |
| `$((expresión))` | Aritmética del shell | `echo $((2**10 + 4))` → 1028 |
| `${var}` | Variable con control de expansión | `${var:-defecto}`, `${var#patrón}`, `${var//a/b}` |
| `${#var}` | Longitud | `echo ${#PATH}` |
| `$?` | Código de salida del último comando | `make; [ $? -eq 0 ] && echo ok` |
| `$!` | PID del último background | `nohup servidor & echo $! > server.pid` |
| `$$` | PID del shell actual | Útil en nombres de archivo temporales |
| `$0`, `$1`… | Nombre del script y argumentos | ver [[Bash Scripting - Fundamentos Básicos]] |
| `{a,b}` | expansión de llaves | `touch respaldo_{enero,febrero}.tar` → 2 archivos |
| `~`, `~user` | Home | `cd ~root` |

> [!tip] `$( )` vs backticks: siempre `$( )`
> Backticks no se anidan (hay que escaparlos con `\``), interpretan `\` raro, y son ilegibles en líneas complejas. `$( )` hace todo bien y se compone: `f=$(basename "$(readlink -f "$1")")`.

## Quoting: comillas simples vs dobles vs nada

| Forma | El shell… | Ejemplo |
|---|---|---|
| **Sin comillas** | Expande variables, `$( )`, globbing **y** splitting por espacios | `echo $VAR *.log` |
| **`'simples'`** | Todo **literal**: nada se expande (ni `$`, ni `\`, ni `` ` ``) | `grep 'precio total: $5' f` |
| **`"dobles"`** | Expande `$var`, `$( )`, `${}`, `~`… pero **no** globbing ni splitting | `echo "archivos: $(ls \| wc -l)"` |
| **`$'...'`** | Literal con escapes C: `\n`, `\t`, `\x41` | `printf $'línea1\nlínea2\n'` |
| **`\"`, `\$`, `\\`** | Escapado puntual dentro de dobles | `echo "total: \$5"` |

Regla mnemotécnica: **simples = tú te lo has buscado** (todo literal); **dobles = quiero las variables pero literal el resto**. Casi todo valor que viaja entre comandos debería ir entre comillas dobles: `rm "$ARCHIVO"`, `ssh host "grep '$PAT' /var/log/syslog"`.

```bash
# splitting + globbing sin comillas (bug frecuente)
for f in $(ls *.txt); do echo "$f"; done      # se parte por espacios y re-hace glob
# correcto
for f in *.txt; do echo "$f"; done             # globbing directo del shell
```

## Globbing: los patrones del shell

Aplicado por el shell **antes** de ejecutar el comando (a diferencia de los patrones internos de `find`, `grep`…):

| Patrón | Significado | Ejemplo |
|---|---|---|
| `*` | Cualquier secuencia de caracteres (**no** cruza `/`) | `ls *.log` |
| `?` | Exactamente un carácter | `ls dia?.txt` |
| `[abc]` | Un carácter del conjunto | `ls [aeiou]*` |
| `[a-z]` | Un carácter del rango | `ls backup.[0-9][0-9]` |
| `[!abc]` / `[^abc]` | Un carácter **fuera** del conjunto | `ls img.[!t]*` |
| `**` | **Solo** con `shopt -s globstar` (bash): recursivo | `ls **/*.md` |

> [!warning] No hay coincidencias = el patrón queda **literal**
> Si `*.log` no matchea ningún archivo, bash (por defecto) pasa el string `*.log` tal cual al comando, que luego falla confusamente (`grep: *.log: No such file`). `zsh` falla con "no matches found" — más claro. En scripts: `shopt -s nullglob` (desaparece si no hay match) o verificar con `[ -e patrón ]` antes.

## Pipes: `|` vs `xargs` vs `-exec`

La diferencia fundamental: **`|` conecta flujos de texto (streaming)**; **`xargs` convierte texto en ARGUMENTOS de otro comando**.

```bash
# | — stdout de uno → stdin del otro. El segundo comando NO recibe argumentos.
grep 'ERROR' syslog | wc -l            # wc cuenta LÍNEAS de su stdin
grep 'ERROR' syslog | sort | uniq -c   # pipeline de texto

# xargs — lee stdin, lo parte en tokens y los inyecta como argumentos.
grep -l 'ERROR' *.log | xargs wc -l    # wc recibe NOMBRES de archivo: cuenta cada uno
```

| Herramienta | Mecanismo | Cuándo |
|---|---|---|
| `comando \| otro` | Stream: texto → texto | Filtrado, orden, agregación |
| `… \| xargs CMD` | Texto → **argv** de CMD | Cuando CMD necesita archivos como argumentos (`wc -l`, `rm`, `chmod`) |
| `find … -exec CMD {} +` | Lo mismo, pero sin pasar por texto | Preferible cuando ya tenés `find`: nombres con espacios seguros por diseño |
| `find … -execdir CMD {} \;` | Por archivo, desde su directorio | Cuando necesitás contexto individual (seguridad ante symlinks) |

### Flags de `xargs que evitan accidentes

| Flag | Significado |
|---|---|
| `-0` / `-print0` | Separador NUL: **único** manejo correcto de nombres con espacios/comillas/newlines |
| `-n N` | Máximo N argumentos por ejecución |
| `-I PATRÓN` | Reemplaza el patrón por cada token (un comando por token; incompatible con `-n`) |
| `-P N` | Ejecuta N en paralelo |
| `-r` / `--no-run-if-empty` | No ejecuta si el stdin está vacío (GNU; protección de `rm` accidental) |
| `-t` | Imprime el comando antes de ejecutarlo (auditoría) |
| `-d '\n'` | Delimitador newline (alternativa a `-0` para salidas `-l` limpias) |

```bash
# Seguro con nombres raros
find . -name '*.tmp' -print0 | xargs -0 -r rm
# Una línea por archivo, con confirmación
find . -name '*.bak' -print0 | xargs -0 -I {} rm -i {}
# Paralelo: sha256 de todos los .iso
find . -name '*.iso' -print0 | xargs -0 -P 4 -n 1 sha256sum
```

> [!warning] Por qué `find … | xargs` es frágil
> `find /ruta | xargs rm` parte por **espacios y saltos de línea**: `mi archivo.txt` se vuelve dos argumentos (`mi` y `archivo.txt`) y rm borra/fracasa mal. Además puede cruzar límites de comando (ARG_MAX) y reordenar. Regla: o `-print0 | xargs -0`, o mejor `-exec CMD {} +` directamente en `find` (ver [[Búsquedas a nivel de sistema]]).

## Comandos esenciales (catálogo operativo)

| Comando | Descripción |
|---|---|
| `whoami` | Usuario efectivo actual |
| `id` | UID, GID y grupos del usuario |
| `sudo -l` | Qué puede ejecutar con sudo (base del sudoers) |
| `sudo <comando>` | Ejecuta **un** comando como root |
| `sudo su` / `sudo -i` | Shell completo como root (`-i` simula login: carga el entorno) |
| `which` / `type` / `command -v` | ¿Qué binario ejecuta el shell? (`type` ve builtins; ver sección PATH) |
| `echo "texto"` / `echo $VAR` | Imprime cadena o variable (ver `printf` para formatos serios) |
| `grep "Patrón"` | Filtra líneas. Comunes: `-n` línea, `-v` invertido, `-r` recursivo, `-E` ERE, `-c` conteo, `-l` archivos, `-i` case-insensitive |
| `cd` | Cambia de directorio: `..`, `~`, `-` (anterior), ruta |
| `ls` | Lista. `-a` ocultos, `-l` largo, `-lh` legible, `-lah` todo |
| `pwd` | Directorio actual (`-P` físico) |
| `whereis` / `whatis` / `apropos` | Binario+fuente+man / 1 línea de man / búsqueda en man |
| `find <ruta> <tests>` | Búsqueda por metadatos (ver nota completa) |
| `touch <nombre>` | Crea vacío / refresca timestamps |
| `nvim`, `nano`, `vi` | Editores de terminal |
| `mkdir <nombre>` | Crea directorio (`-p` para padres) |
| `rmdir <nombre>` | Borra directorio **vacío** |
| `rm <f>` | Borra. `-r` recursivo, `-f` fuerza (¡peligroso!) |
| `mv <origen> <destino>` | Mueve/renombra |
| `cp <origen> <destino>` | Copia (`-r` recursivo, `-a` archive) |
| `disown` | Desvincula un job ya lanzado de la terminal (sigue tras cerrar) |
| `useradd` / `groupadd` | Crea usuario/grupo (`useradd -m -s /bin/bash u`) |
| `chown` / `chgrp` | Cambia dueño / grupo (`chown usuario:grupo f`, `-R` recursivo) |
| `xargs` | Convierte stdin en argumentos (ver sección dedicada) |
| `file` | Tipo real por magic bytes |
| `sort` / `uniq` | Ordena / colapsa adyacentes (`-c` conteo) |
| `strings` | Cadenas imprimibles de binarios |
| `base64` / `xxd` | Codificación Base64 (`-d` decode) / hexdump (`-ps`, `-r`) |
| `tee` | Duplica stdout a archivo y sigue el pipe (`-a` append) |
| `sponge` | Lee **todo** antes de escribir: redirección in-place segura |
| `diff` | Diferencias (`-u` unificado) |
| `alias` / `unalias` | Atajos de shell (solo interactivos por defecto) |

### `sponge` (moreutils): editar un archivo con su propia salida

`grep patrón f > f` **trunca `f` antes** de que grep lea nada: quedás con archivo vacío. `sponge` espera a tener toda la entrada y luego escribe:

```bash
grep -v "patrón" archivo.txt | sponge archivo.txt
sort -u lista.txt | sponge lista.txt
# Alternativas sin moreutils: el patrón atómico clásico
grep -v patrón archivo.txt > tmp && mv tmp archivo.txt    # mv es atómico en el mismo FS
```

## Archivos clave de identidad en `/etc`

| Ruta | Contenido | Formato de cada línea |
|---|---|---|
| `/etc/passwd` | Cuentas de usuario (sin secretos) | `nombre:x:UID:GID:comentario:home:shell` — la `x` = la contraseña está en `/etc/shadow` |
| `/etc/group` | Grupos | `nombre:x:GID:miembros` — `miembros` son los usuarios cuyo **grupo primario no** es este |
| `/etc/shadow` | Contraseñas hash + aging (solo root) | `nombre:hash:últ.cambio:mín:máx:aviso:inactivo:expira:…` |
| `/etc/gshadow` | hashes/ajustes de grupos (solo root) | — |
| `/etc/sudoers` | Quién puede hacer qué con sudo (editarse solo con `visudo`, que valida sintaxis) | `usuario ALL=(ALL:ALL) ALL`, `%sudo ALL=(ALL:ALL) ALL` |
| `/etc/shells` | Shells válidos (para `chsh` y para evitar shells raros en cuentas) | rutas a `/bin/bash` etc. |

## Errores comunes

1. **`Permission denied` al ejecutar `./algo`** — falta bit `x` en el archivo (o en un directorio padre): `chmod +x`, ver [[Lectura de Permisos]].
2. **`command not found` pero el binario existe** — no está en `$PATH` o ganó otro nombre; `hash -r`, `type`, o agregar el directorio al PATH.
3. **Variables perdidas entre líneas de un script** — `VAR=valor` solo asigna al proceso actual; `export VAR=valor` lo propaga a los hijos. Y `cmd1 | cmd2` **no comparte variables** (subshell).
4. **`2>` y `1>` invertidos**: los errores se van a la pantalla y el log queda limpio — recordar el orden (`> f 2>&1` o `&>`).
5. **Glob sin coincidencias pasa literal** (`grep: *.log: No such file`) — `shopt -s nullglob` o comprobar existencia.
6. **Nombres con espacios rotos en pipelines** — `-print0 | xargs -0` o `-exec … {} +`.
7. **`cd` falla y el resto de la receta corre en el directorio equivocado** — encadenar `cd ruta && resto`, no con `;`.
8. **`rm -rf $DIR/` con `DIR` vacío** — `set -u` en scripts y comprobaciones previas; error clásico de scripting ver [[Bash Scripting - Fundamentos Básicos]].
9. **`sudo` no ve los `alias`** — los aliases no se expanden sin `shopt -s expand_aliases` y fuera del contexto interactivo; usar funciones o rutas completas en scripts.

## Cheatsheet

```bash
# PATH
echo $PATH | tr ':' '\n'        export PATH="$PATH:$nuevo"        type cmd; hash -r
# Rutas
readlink -f ruta    pwd -P      realpath --relative-to=$PWD /etc
# Operadores
&& || ; !                 # flujo
> >> 2> 2>> &> 2>&1       # redirección (orden importa: >f 2>&1)
<<< 'texto'  << EOF       # stdin inline
$(comando)  $((2+2))  ${var}   $?"$1" $$ $!   # sustituciones
# Quoting
'literal total'  "expande $var"  $'con \n escapes'
# Globbing
*  ?  [abc]  [!x]  **(globstar)
# Streams → argumentos
cmd | cmd               # texto→texto
find . -print0 | xargs -0 -r CMD   # texto→argv seguro
find . -exec CMD {} +             # argv sin texto intermedio
```

## Relacionadas

- [[Comandos Básicos de Sistema y Red en Linux]] — catálogo de coreutils, procesos y red
- [[Búsquedas a nivel de sistema]] — `find`/`fd`/`locate`/`rg` en profundidad
- [[Estructura de Directorios de Linux]] — qué hay en las rutas que navegas
- [[Lectura de Permisos]] — por qué un comando falla con `Permission denied`
- [[Bash Scripting - Fundamentos Básicos]] — todo esto como base de scripts
- [[PATH Hijacking]] — el ataque que abusa del orden del `$PATH`
- [[Privilegios SUID en Linux]] — por qué los binarios con `x` especial son distintos
- [[Node JS]] — `npx`, `node_modules/.bin` y `$PATH` en el ecosistema JS

## Referencias

- `man 1 bash` — secciones *EXPANSIONS*, *REDIRECT*, *QUOTING*
- GNU Bash manual: <https://www.gnu.org/software/bash/manual/bash.html>
- POSIX shell command language (IEEE Std 1003.1): <https://pubs.opengroup.org/onlinepubs/9699919799/>
- FHS (rutas): <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html>
