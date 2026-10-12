---
aliases:
  - Linux Info
  - System Info
  - Network Commands
  - Coreutils
  - Comandos Básicos Linux
tags:
  - sys/linux
  - sys/redes
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Comandos, Rutas y Operadores de Linux]]"
  - "[[Búsquedas a nivel de sistema]]"
  - "[[Lectura de Permisos]]"
  - "[[Estructura de Directorios de Linux]]"
  - "[[Tipos de Interfaces de Red]]"
  - "[[Conceptos Fundamentales de Redes]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
  - "[[Nmap Guía Completa]]"
  - "[[Wireshark y Tshark]]"
fuente: "https://www.gnu.org/software/coreutils/manual/html_node/"
---

# Comandos Básicos de Sistema y Red en Linux

Catálogo denso de los comandos que se usan todos los días para **inspeccionar** el sistema: archivos (coreutils), procesos, identidad, disco y red. Todo lo que sigue funciona en cualquier Linux con GNU userland; al final se tabulan las diferencias de los gestores de paquetes más comunes.

> [!info] Anatomía de un comando GNU
> Casi todos los comandos de este artículo son GNU coreutils y siguen el mismo contrato: `comando [OPCIONES] [ARGUMENTOS]`, opciones con guion simple (`-l`), largas con doble guion (`--human-readable`), combinables (`-lah` = `-l -a -h`), stdin/stdout/stderr como canales estándar, y códigos de salida: **0** = éxito, **≠0** = fallo (verificable con `$?` y utilizable con `&&`/`||` — ver [[Comandos, Rutas y Operadores de Linux]]).

## Catálogo de coreutils

### Archivos y directorios

| Comando | Qué hace | Flags esenciales | Ejemplo |
|---|---|---|---|
| `ls` | Lista directorios | `-a` (ocultos), `-l` (largo), `-h` (tamaños legibles), `-R` (recursivo), `-t` (por mtime), `-S` (por tamaño), `-d` (el dir mismo), `--color=auto` | `ls -lah /var/log` |
| `cp` | Copia | `-r` (recursivo), `-p` (preserva modo/timestamps/dueno), `-a` (archive: -dpr), `-i` (pregunta), `-u` (solo si mas nuevo), `-v` | `cp -a config/ /backup/` |
| `mv` | Mueve/renombra | `-i` (pregunta), `-f` (fuerza), `-n` (no pisa), `-v` | `mv -n descarga/* fotos/` |
| `rm` | Borra | `-r` (recursivo), `-f` (fuerza, sin error si no existe), `-i` (pregunta), `--one-file-system` | `rm -rf build/` |
| `mkdir` | Crea directorios | `-p` (crea padres si faltan, no falla si existe), `-v` | `mkdir -p /tmp/a/b/c` |
| `rmdir` | Borra directorios **vacíos** | `-p` (borra padres vacíos también) | `rmdir -p a/b` |
| `touch` | Crea vacío / actualiza timestamps | `-c` (no crea), `-d FECHA`, `-r REF` | `touch -c existing` |
| `ln` | Enlaces | `-s` (simbólico), `-f` (pisa), `-i` (pregunta), `-v` | `ln -s /opt/app/bin/app /usr/local/bin/app` |
| `file` | Identifica tipo real por contenido (magic bytes) | `-L` (sigue symlinks), `-z` (gzip), `-b` | `file exploit.elf` |
| `stat` | Metadatos del **inodo**: tamaño, bloques, permisos, uid/gid, atime/mtime/ctime/btime, inodo, dispositivo | `-c FORMATO` (p. ej. `%a %U %s %n`), `-f` (del FS) | `stat -c '%a %U %s %n' f` |
| `du` | Uso de disco por árbol | `-s` (resumen), `-h` (legible), `-a` (archivos), `-x` (un solo FS), `--max-depth=N` | `du -xh --max-depth=1 /` |
| `df` | Uso de disco por **montaje** | `-h` (legible), `-T` (tipo de FS), `-i` (inodos), `-x` (solo este FS) | `df -hTi` |
| `readlink` | Resuelve symlinks | `-f` (ruta completa canónica, resuelve todos los componentes), `-e` (igual pero exige que exista), `-m` (no exige existencia) | `readlink -f ./script.sh` |
| `realpath` | Igual que `readlink -f` como binario dedicado | `--relative-to=DIR` | `realpath --relative-to=$PWD /etc` |
| `basename` / `dirname` | Componente final / ruta padre | `-s SUFIJO` (basename: quita sufijo) | `dirname /a/b/c` → `/a/b` |
| `tree` | Árbol visual (no es coreutils; del paquete `tree`) | `-L N`, `-d`, `-a`, `-I PATRÓN` | `tree -L 2 -d /etc` |

> [!warning] `rm -rf` y sus peligros
> `rm -rf` borra **sin confirmación y sin papelera**. Los fallos clásicos: `rm -rf /etc/ ` con espacio (borra `/etc`), variables vacías (`rm -rf "$DIR/"` con `DIR` sin setear → `rm -rf /`), y `--no-preserve-root` contra `/`. Prevención en scripts: `set -u`, comprobar que la variable no está vacía, y nunca interpolar rutas sin comillas.

### Texto por stdin/stdout

| Comando | Qué hace | Flags esenciales | Ejemplo |
|---|---|---|---|
| `cat` | Concatena/muestra archivos | `-n` (numera), `-A` (muestra tabs/EOL), `-s` (colapsa líneas vacías) | `cat -n notas.md` |
| `tac` | `cat` al revés (última línea primero; a nivel de líneas) | `-r -s SEP` (separador) | `tac log \| head` (últimos eventos) |
| `less` | Visor paginado con búsqueda (`/pat`, `n`), seguir (`F`) | `-N` (numera), `+comando`, `+G` (ir al fin) | `less /var/log/syslog` |
| `more` | Paginado primitivo (legado de `less`) | — | `more archivo` |
| `head` | Primeras líneas | `-n N`, `-c N` (bytes) | `head -n 20 access.log` |
| `tail` | Últimas líneas / seguir | `-n N`, `-f` (follow), `-F` (follow por nombre, sobrevive rotación), `-c` | `tail -F /var/log/*.log` |
| `wc` | Cuenta líneas/palabras/bytes | `-l` `-w` `-c` `-m` (caracteres), `-L` (línea más larga) | `wc -l archivo` |
| `sort` | Ordena | `-n` numérico, `-r` inv, `-u` unique, `-h` human, `-V` version, `-t SEP -k N` campo, `-b` ignora blanks, `-f` case-insensitive | `sort -t: -k3 -n /etc/passwd` |
| `uniq` | Colapsa **adyacentes** iguales | `-c` (conteo), `-d` (dupes), `-u` (únicas), `-i`, `-f N`, `-s N` | `sort f \| uniq -c \| sort -rn` |
| `cut` | Columnas/campos | `-d SEP`, `-f N[,M]`, `-c RANGO`, `--complement` | `cut -d' ' -f1,4 syslog` |
| `tr` | Traduce/borra/caracteres | `'de'→'a'`, `-d SET`, `-s SET` (squeeze), `-c` (complemento) | `tr -d '\r' < in > out` |
| `tee` | Duplica a archivo y stdout | `-a` (append), `-i` (ignora stdin) | `make 2>&1 \| tee build.log` |
| `diff` | Diferencias entre archivos | `-u` (unificado), `-r` (recursivo), `-q` (solo "difieren"), `-y` (lado a lado, con `wdiff`) | `diff -u a.conf b.conf` |
| `md5sum` / `sha256sum` | Sumas de verificación | `-c` (verifica contra lista), `--tag` | `sha256sum iso > f.sha256` |
| `base64` | Codifica/decodifica Base64 | `-d` (decode), `-w0` (sin wrap) | `echo -n token \| base64` |
| `xxd` | Hexdump (del vim) | `-ps` (hex plano), `-r` (revierte a binario), `-l N` | `xxd -r -p hex.txt bin` |
| `strings` | Cadenas imprimibles de un binario | `-n N` (mín. longitud), `-e ENC` (codificación) | `strings -n 8 malware.bin` |
| `wc`, `nl`, `rev` | Numerar líneas / invertir caracteres de línea | `nl -ba` | `rev file` |

### Procesos

| Comando | Qué hace | Uso típico |
|---|---|---|
| `ps` | Snapshota de procesos | `ps aux` (BSD: todo, formato clásico) · `ps -ef` (SysV: todo, jerárquico con `--forest`) · `ps -faux` (full + todos los usuarios + sin TTY) · `ps -eo pid,ppid,%cpu,%mem,command --sort=-%cpu` (columnas propias) |
| `top` / `htop` | Tablero vivo (CPU, RAM, swap, top procesos) | `top` (clado: `M` mem, `P` cpu, `k` kill) · `htop` ( interactivo ) |
| `pstree` | Árbol de procesos padre→hijo | `pstree -p` (con PIDs) — herencia tras `fork`/`exec` |
| `kill` / `killall` / `pkill` | Señales a procesos | `kill -TERM PID` (15, cortés) · `kill -9 PID` (KILL, ineludible) · `pkill -f 'python servidor.py'` |
| `jobs`, `fg`, `bg`, `&` | Trabajo con jobs del shell | `comando &` · `jobs -l` · `fg %1` · `bg %2` · `nohup cmd &` (sobrevive al logout) · `disown` (desvincula un job ya lanzado) |
| `free` | Memoria RAM/swap | `free -h` · `free -h -s 2` (refresca cada 2 s) |

> [!info] `kill -9` no es "matar con garra"
> La señal 9 (KILL) la entrega el kernel **sin pasar por el proceso**: no puede ignorarse ni atraparse, pero **tampoco permite limpieza** (no corre `trap`, no cierra fds congelando nada). Un proceso en estado `D` (uninterruptible, esperando I/O de disco/NFS) **no muere ni con -9** hasta que vuelva el I/O. Intentá primero `kill -15` (TERM).

### Identidad y entorno

```bash
whoami                 # usuario efectivo (UID real del proceso)
id                     # uid, gid y TODOS los grupos (GID primario + suplementarios)
sudo -l               # qué puede ejecutar este usuario con sudo (salida del sudoers)
printenv               # todas las variables de entorno
printenv PATH          # una en concreto
echo $PATH             # lo mismo vía expansión del shell
env                   # como printenv, pero también sirve como prefijo: env VAR=x cmd
set                   # variables de shell + locales + funciones (solo bash)
export VAR=valor       # exporta al entorno de los hijos
history | tail -20     # historial de comandos de la sesión
w / who / users        # quién está conectado y qué hace (w: con carga y comando actual)
last -n 10             # historial de logins (de /var/log/wtmp)
lastlog                # último login de cada cuenta (de /var/log/lastlog)
uname -r               # versión del kernel (uname -a: todo)
hostnamectl            # hostname, SO, kernel, arquitectura (systemd)
uptime                 # uptime, carga 1/5/15 min
lsb_release -a         # info de la distro (si está instalado; ver también /etc/os-release)
```

| Comando | Flags | Nota |
|---|---|---|
| `ps -faux` | `-f` full, `-a` todos los usuarios, `-u` por usuario, `-x` incluye sin TTY | Snapshot tipo BSD |
| `ps -eo command` | `-e` todos, `-o column` formato exacto | `command` = línea completa con argumentos |
| `uname -r` / `-a` | `-r` kernel, `-a` todo | Para buscar CVEs del kernel o módulos |
| `watch -n N CMD` | `-n` intervalo, `-d` resalta diffs, `-t` sin cabecera | `watch -n1 'ss -s'`, `watch -d df -h` |

> [!tip] Historial: dónde vive realmente
> `~/.bash_history` se escribe **al cerrar** la sesión (no en vivo) y solo si `HISTFILE` apunta ahí. Los shells interactivos de otros usuarios se ven con `ps` (`-eo args`), no con el historial. `HISTCONTROL=ignoredups` y `HISTSIZE=10000` son ajustes típicos en `~/.bashrc` (ver [[Bash Scripting - Fundamentos Básicos]]).

## Permisos: referencia rápida

La lectura completa (resolución, SUID/SGID, sticky, ACL, umask) vive en [[Lectura de Permisos]]; esto es el mini-resumen operativo:

```bash
ls -l archivo        # -rwxr-xr--  1 root root  4096 ...  → tipo + ugo bits + dueño + grupo
chmod 755 f          # octal: u=rwx g=rx o=rx
chmod u+x,g-w f      # simbólico: ajuste incremental
chmod -R 755 dir/    # recursivo (cuidado con SUID heredados)
chown usuario:grupo f
chown -R www-data:www-data /var/www/
getfacl f / setfacl -m u:otro:r f     # ACLs (ver nota)
umask                # máscara de permisos por defecto al crear (022 típico)
```

| Modo | Significado | Uso típico |
|---|---|---|
| `755` | rwxr-xr-x | directorios y binarios ejecutables |
| `644` | rw-r--r-- | archivos de texto/común |
| `600` | rw------- | secretos del dueño (`~/.ssh/id_rsa`, `.htpasswd`) |
| `700` | rwx------ | directorios privados (`~/.ssh`) |
| `640` | rw-r----- | logs con grupo dedicado (adm/syslog) |
| `+t` / `1777` | sticky bit | `/tmp`: solo el dueño del archivo lo borra |
| `4755` | +SUID | binario con privilegios del dueño (ver [[Privilegios SUID en Linux]]) |

## Redes: lo básico para diagnosticar

### De `ifconfig`/`netstat` a `ip`/`ss` (iproute2)

`ifconfig` (net-tools) y `netstat` están **deprecados** y ausentes en instalaciones mínimas modernas. El reemplazo oficial es iproute2: `ip` (direcciones, rutas, enlaces) y `ss` (sockets).

| Tarea | Clásico (net-tools) | Moderno (iproute2) |
|---|---|---|
| Ver interfaces e IPs | `ifconfig` / `ifconfig eth0` | `ip addr` / `ip -br a` / `ip a show eth0` |
| Activar/desactivar | `ifconfig eth0 up/down` | `ip link set eth0 up/down` |
| Asignar IP | `ifconfig eth0 10.0.0.5/24` | `ip addr add 10.0.0.5/24 dev eth0` |
| Ver rutas | `route -n`, `netstat -rn` | `ip route` / `ip r get 8.8.8.8` |
| Tabla ARP | `arp -an` | `ip neigh` |
| Conexiones | `netstat -natp` | `ss -tunap` |

> [!info] Detrás de `ss` no hay `netstat` reescrito
> `ss` lee directamente las tablas del kernel vía **netlink** (inotify de sockets: `SOCK_DIAG`), mientras `netstat` abría `/proc/net/*` y parseaba texto. Con miles de conexiones, `ss` es ~×10 más rápido. `netstat` sigue disponible en el paquete `net-tools` por compatibilidad.

### Listar puertos abiertos

```bash
# 1. Opción estándar: sockets TCP+UDP en escucha, numérica, con proceso
ss -tulnp
#    -t TCP  -u UDP  -l escucha  -n numérico (sin DNS/servicios)  -p proceso (ver PID/programa)
```

```bash
# 2. Equivalente clásico (net-tools); -t es SOLO TCP (no muestra UDP)
netstat -nat
#    -n numérico  -a todos los estados (escucha + conectados)  -t solo TCP
#    -l escucha  -p PID/programa (root)  -u UDP  -e extendido
```

> [!warning] `netstat -nat` no muestra UDP
> Aunque se cite como "TCP y UDP", `-t` filtra a TCP. Para UDP agregá `-u` (`netstat -natu`) o usá `ss -tulnp` que ya lo cubre todo.

```bash
# 3. Fuente cruda: /proc/net/tcp en hexadecimal (lo que netstat/ss parsean)
cat /proc/net/tcp
#    st: 0A = LISTEN, 01 = ESTABLISHED; addr/puerto en hex big-endian por campo

# 4. Proceso detrás de un puerto
lsof -i :80               # cualquier protocolo en el puerto 80
lsof -iTCP:80 -sTCP:LISTEN   # solo listeners TCP
ss -tlnp | grep :80       # alternativa sin lsof
fuser 80/tcp              # solo los PIDs (del paquete psmiss)
```

| Herramienta | Mecanismo | Ventaja |
|---|---|---|
| `ss` | netlink (sock_diag) | Rápido, moderno, sin binario extra |
| `netstat` | parsea `/proc/net/*` | Compatibilidad con guiones viejos |
| `lsof` | lista **todos** los fds abiertos de todos los procesos | Ve archivos, pipes, NFS, tty — no solo red |
| `/proc/net/tcp` | texto crudo del kernel | Funciona sin herramientas (busybox/rescate) |

### Conectividad básica

```bash
ping -c 4 8.8.8.8          # ICMP echo; -c limita paquetes (sin -c: infinito)
ping -c 2 ::1              # IPv6
ip route get 1.1.1.1       # qué ruta/interface usaría el kernel para llegar
traceroute 8.8.8.8         # salto a salto (o tracepath, sin root)
curl -sSI https://example.com   # HEAD: cabeceras y estado
curl -s http://ip/info     # APIs HTTP desde CLI
wget -qO- URL              # descarga simple
```

```bash
# netcat: la navaja suiza
nc -lnvp 4444              # listener: -l escucha, -n numérico, -v verboso, -p puerto
nc host 80                 # cliente: conecta y habla por la terminal (banner grabbing)
nc -zv host 1-1024         # sondeo de puertos sin datos (-z scan-only)
# Ojo: existen dos "netcat" (openbsd-netcat y nmap-ncat) con flags distintos; -e no existe en OpenBSD.
```

Para reconocimiento completo de hosts y puertos, ver [[Nmap Guía Completa]]; para inspección de paquetes, [[Wireshark y Tshark]].

## Gestores de paquetes

| Tarea | Debian/Ubuntu (apt) | Arch (pacman) | RHEL/Fedora (dnf/yum) | openSUSE (zypper) |
|---|---|---|---|---|
| Índice | `apt update` | `pacman -Sy` | `dnf check-update` | `zypper refresh` |
| Instalar | `apt install pkg` | `pacman -S pkg` | `dnf install pkg` | `zypper install pkg` |
| Actualizar todo | `apt upgrade` | `pacman -Syu` | `dnf upgrade` | `zypper update` |
| Quitar | `apt remove pkg` | `pacman -R pkg` | `dnf remove pkg` | `zypper remove pkg` |
| Buscar | `apt search patrón` | `pacman -Ss patrón` | `dnf search patrón` | `zypper search patrón` |
| ¿Qué paquete da X? | `dpkg -S /usr/bin/f` | `pacman -Qo /usr/bin/f` | `dnf provides '*/f'` | `zypper what-provides f` |
| Información | `apt show pkg` | `pacman -Si pkg` | `dnf info pkg` | `zypper info pkg` |
| Listar instalados | `dpkg -l` | `pacman -Q` | `dnf list installed` | `zypper se -i` |
| Archivos del paquete | `dpkg -L pkg` | `pacman -Ql pkg` | `dnf repoquery -l pkg` | `zypper what-provides` |
| Autolimpiar | `apt autoremove` | `pacman -Sc` | `dnf autoremove` | `zypper clean` |
| Binario | `dpkg` (deb) | `pacman` (pkg.tar.zst) | `dnf/rpm` (rpm) | `zypper/rpm` |

Además: `snap` y `flatpak` (empaquetado universal, aislado), e instalaciones de usuario con lenguajes (`pip`, `npm`, `cargo` — ver [[Node JS]]).

## Errores comunes

1. **`command not found`** — no está en el `$PATH` o no está instalado. `which`/`type` para confirmar; `pacman -Fy`/`apt-file search` para encontrar el paquete que lo trae (ver [[Comandos, Rutas y Operadores de Linux]]).
2. **`Permission denied` en `ss -p`/`netstat -p`** — el PID del socket solo lo ve el dueño o root.
3. **`Cannot find device "eth0"`** — el nombre moderno es predecible (`enp3s0`, `wlo1`); mirá `ip -br link` (ver [[Tipos de Interfaces de Red]]).
4. **`No space left on device` con `df` OK** — pueden agotarse **inodos** (`df -i`) o el espacio de `/tmp` (tmpfs).
5. **`kill: No such process`** — el PID ya murió (o es de otro namespace de containers).
6. **`bash: ./script.sh: Permission denied`** — falta bit `x` en el archivo o `x` en algún directorio padre (`chmod +x`, ver [[Lectura de Permisos]]).
7. **`tail -f` no ve logs rotados** — usá `-F` (follow by name) en lugar de `-f` (follow by fd).

## Cheatsheet

```bash
# Archivos          # Texto              # Procesos           # Red
ls -lah             cat / tac / less     ps aux               ip -br a
cp -a / mv -n       head / tail -F       top / htop           ip route
rm --one-file-system wc -l               pstree -p            ss -tulnp
mkdir -p            sort | uniq -c       kill -15 / -9        lsof -i :80
stat -c '%a %U %s'  cut -d: -f1          nohup cmd &          ping -c 4 host
du -xh --max-depth=1                     free -h              curl -sSI URL
df -hTi             tr -d '\r' | tee -a                        nc -zv host 1-1024
readlink -f PATH    diff -u a b
# Identidad: whoami · id · sudo -l · printenv · uname -r · w · last -n 10
# Paquetes:  apt update/install/search  ·  pacman -Syu/-Ss  ·  dnf install/search
```

## Relacionadas

- [[Comandos, Rutas y Operadores de Linux]] — PATH, redirecciones, pipes, quoting, `xargs`
- [[Búsquedas a nivel de sistema]] — `find`, `locate`, `fd`, `rg`, `grep` a fondo
- [[Lectura de Permisos]] — el modelo completo de rwx, SUID, ACL, umask
- [[Estructura de Directorios de Linux]] — qué vive en cada ruta (`/etc`, `/var/log`, `/proc`)
- [[Tipos de Interfaces de Red]] — `ip link/addr/route` en profundidad
- [[Conceptos Fundamentales de Redes]] — OSI, TCP/UDP, puertos comunes
- [[Nmap Guía Completa]] — reconocimiento de puertos allá de `ss`
- [[Wireshark y Tshark]] — captura y análisis de paquetes
- [[Bash Scripting - Fundamentos Básicos]] — automatizar todo lo anterior

## Referencias

- `man 1 ls`, `man 1 ps`, `man 8 ss`, `man 8 ip`, `man 1 curl`, `man 1 nc`
- GNU Coreutils manual: <https://www.gnu.org/software/coreutils/manual/>
- iproute2: <https://man7.org/linux/man-pages/man8/ip.8.html>
- ss vs netstat: <https://unix.stackexchange.com/questions/117467/how-to-list-all-ports-listening-on-my-linux-box>
