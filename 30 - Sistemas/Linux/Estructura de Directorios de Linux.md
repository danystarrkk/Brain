---
aliases:
  - Linux Filesystem Hierarchy Standard
  - FHS
  - Jerarquía de Directorios Linux
  - Estructura Linux
tags:
  - sys/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
  - "[[Comandos, Rutas y Operadores de Linux]]"
  - "[[Lectura de Permisos]]"
  - "[[Búsquedas a nivel de sistema]]"
  - "[[Cron Jobs y Programación de Tareas]]"
  - "[[Tipos de Interfaces de Red]]"
fuente: "https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html"
---

# Estructura de Directorios de Linux

Este documento detalla la jerarquía estándar de directorios en sistemas operativos GNU/Linux, siguiendo los principios del **Filesystem Hierarchy Standard (FHS)**, con las profundidades que el día a día de administración y auditoría exige: qué hay *dentro* de `/proc` y `/sys`, qué es realmente `/run`, cómo se reparten `/usr`, `/opt` y `/usr/local`, y en qué se parecen (y difieren) Debian y RHEL.

> [!info] El Filesystem Hierarchy Standard (FHS)
> El FHS define la estructura principal de directorios en sistemas tipo Unix. Su objetivo es estandarizar la ubicación de archivos y directorios para que usuarios y software puedan predecir dónde encontrar ciertos tipos de archivos. El FHS clasifica cada directorio por tres atributos: **compartible vs no-compartible** (¿puede compartirse vía red?), **estático vs variable** (¿cambia sin administración?) y **privado vs específico de arquitectura**.

> [!tip] La pregunta que ordena todo
> Antes de colocar un archivo, preguntá: *¿es software del sistema o mío?*, *¿es estático o cambia solo?*, *¿debe sobrevivir a un reinicio?*. El FHS responde esas tres preguntas con la ruta: sistema→`/usr`, tuyo→`/usr/local` o `~`, de terceros→`/opt`, variable→`/var`, runtime→`/run` o `/tmp`.

## Mapa maestro: los directorios de primer nivel

| Ruta | Propósito | Estático/Variable | Compartible | Tamaño en disco real |
|---|---|---|---|---|
| `/` | Raíz: todo cuelga de aquí; solo subdirs permitidos | Mixto | No | — |
| `/bin` | Binarios esenciales para **cualquier** usuario (arranque y rescue) | Estático | Sí | Sí (hoy: symlink → `/usr/bin`) |
| `/sbin` | Binarios de **administración** (arranque, recovery, fsck) | Estático | Sí | Sí (symlink → `/usr/sbin`) |
| `/lib` | Bibliotecas compartidas de `/bin` y `/sbin` | Estático | Sí | Sí (symlink → `/usr/lib`) |
| `/lib64` | Bibliotecas 64 bits en algunos sistemas | Estático | Sí | Sí |
| `/usr` | Software y datos del sistema (la "instalación" completa) | Estático | Sí | **Sí, grande** |
| `/usr/local` | Software instalado **por el admin**, no por el gestor de paquetes | Estático | Sí | Sí |
| `/opt` | Paquetes de **terceros** autónomos (Chrome, Spotify…) | Estático | Sí | Sí |
| `/etc` | **Configuración** del sistema y de programas | Estático* | No | Casi nada |
| `/home` | Datos personales de los usuarios (no `root`) | Variable | Sí | **Sí, variable** |
| `/root` | Home del superusuario | Variable | No | Casi nada |
| `/var` | Datos variables: logs, spool, caché, bases | Variable | No | **Sí, crece** |
| `/var/log` | Registros del sistema y servicios | Variable | No | Crece |
| `/run` | Estado de **arranque actual**: PIDs, sockets, locks | **Volatile** (tmpfs) | No | En RAM |
| `/tmp` | Temporales de corta vida; en muchos sistemas tmpfs | Volatile | No | En RAM |
| `/var/tmp` | Temporales que deben **sobrevivir** a reinicios | Variable | No | Sí |
| `/dev` | Dispositivos como archivos (nodes) | Volatile | Sí | En RAM |
| `/proc` | Sistema de archivos **virtual**: procesos y kernel (procfs) | Volatile | No | En RAM |
| `/sys` | Sistema de archivos **virtual**: modelo de dispositivos del kernel (sysfs) | Volatile | No | En RAM |
| `/boot` | Kernel, initramfs y bootloader | Estático | No | Pequeño (~100 MB) |
| `/mnt` | Punto de montaje **manual/temporal** | Variable | No | — |
| `/media` | Puntos de montaje **automáticos** de extraíbles | Variable | No | — |
| `/srv` | Datos de servicios (web, ftp) si el admin lo decide | Variable | Sí | Variable |
| `/opt` (ya arriba) | — | — | — | — |
| `/lost+found` | Huérfanos de `fsck` (por partición ext*) | Variable | No | Casi nada |
| `/snap` | Snapshots de snap (sistemas con snap) | Estático | Sí | Grande |
| `/efi` | Partición EFI montada (arranque UEFI) | Estático | No | ~100 MB |

*`/etc` es "estático" en el sentido de que lo administra el administrador — pero obviamente los servicios lo escriben (p. ej. `/etc/mtab` → `/proc/self/mounts`).

> [!tip] Regla de particionado en servidores
> Particiones (o volúmenes LVM) separadas para: `/` (sistema), `/var` (logs que pueden llenar el disco raíz), `/home` (datos), `/tmp` (tmpfs o partición con quota), y `/boot` (a veces `/boot/efi`). Un `/var` desbordado por logs que bloquea el arranque o llena `/` es el incidente más clásico de administración.

## Directorio `/bin`

Directorio **estático y compartible** con los **binarios/esenciales** necesarios para el funcionamiento básico del sistema, usables por **todos** los usuarios. Comandos como `ls`, `cp`, `mv`, `rm`, `cat`, `echo`, `bash` residen aquí.

> [!info] `/bin` hoy: el "usr-merge"
> En sistemas modernos, `/bin` es un **enlace simbólico a `/usr/bin`** (verificado en este host: `bin`, `sbin`, `lib`, `lib64` → `usr/*`). La lógica del FHS distingue `/bin` (esencial para arrancar) de `/usr/bin` (software completo), pero como `/usr` casi siempre está en el mismo volumen que `/`, la fusión simplifica el mantenimiento (una sola actualización alcanza a todo). Debian/Ubuntu lo activaron con la transición *usrmerge*; Fedora/RHEL desde hace años; Arch en releases recientes. Comprobá con `ls -ld /bin`.

## Directorio `/boot`

Directorio **estático, no compartible**, con **todos los archivos necesarios para el arranque**, excepto configuración de arranque de terceros. Contiene:

- **`vmlinuz-X.Y.Z`** — el kernel comprimido.
- **`initramfs-…` / `initrd-…`** — imagen de arranque temporal: contiene módulos y un mini-rootfs para montar el rootfs real (LVM, criptografía, drivers de disco).
- **`grub/`** (o `loader/` en systemd-boot) — configuración y módulos del bootloader.
- **`System.map-…`** (símbolos del kernel) y **`config-…`** (configuración con la que se compiló el kernel).

> [!warning] `/boot` lleno = no arranca
> Tras actualizar kernels, los viejos quedan en `/boot` hasta limpiarlos (`pacman -Sc`, `apt autoremove`, o `dnf remove --oldinstallonly`). Con `/boot` en una partición chica (~200–500 MB), es un fallo recurrente. No borrar a mano el kernel que está corriendo: verificar con `uname -r` antes.

## Directorio `/dev`: los dispositivos como archivos

El kernel expone el hardware como **archivos especiales** en `/dev`. No ocupan espacio en disco: son **nodos** (puntos de acceso) creados por `udev` al detectar hardware.

### Esenciales que conviene conocer

| Nodo | Tipo | Qué es |
|---|---|---|
| `/dev/null` | carácter | Pozo negro: lee EOF, ignora lo que escribís |
| `/dev/zero` | carácter | Infinitos ceros (inicializar archivos, pruebas) |
| `/dev/random` | carácter | Entropía del kernel; desde kernel 5.6 **no bloquea** (era el "problema" de los boots lentos) |
| `/dev/urandom` | carácter | Entropía criptográficamente segura, no bloqueante (hoy equivalente a `/dev/random`) |
| `/dev/tty` | carácter | La TTY del proceso actual |
| `/dev/stdin`, `/dev/stdout`, `/dev/stderr` | symlink | → `/proc/self/fd/{0,1,2}`: los descriptores del proceso |
| `/dev/console` | carácter | Consola del sistema (serial en servidores headless) |
| `/dev/sda`, `/dev/nvme0n1`, `/dev/vda` | bloque | Discos: SATA/SCSI (`sd`), NVMe, virtio/KVM (`vd`), IDE antiguo (`hd`) |
| `/dev/sda1`, `/dev/nvme0n1p1` | bloque | **Particiones** (número tras el disco; NVMe usa `p`) |
| `/dev/loop0` | bloque | Loop devices: archivos montados como discos (`mount -o loop`) |
| `/dev/mapper/*` | bloque | Volúmenes lógicos LVM / dm-crypt |
| `/dev/cdrom` → `sr0` | bloque | Unidades ópticas (legacy) |
| `/dev/kvm` | carácter | Aceleración de virtualización |
| `/dev/net/tun` | carácter | Interfaces tun/tap para VPN (ver [[Tipos de Interfaces de Red]]) |

> [!info] Dispositivos de bloque vs carácter
> - **Bloque** (`b`): acceden en bloques fijos y permiten seeks — discos, particiones.
> - **Carácter** (`c`): flujo de bytes secuencial — teclados, TTYs, puertos serie, `/dev/null`.
> El tipo se ve en `ls -l /dev` (primer carácter `b`/`c`) y es el mismo `-type b|c` de `find`. Los **permisos** sobre estos nodos controlan quién puede usar el hardware: `disk` en `/dev/sda` = acceso crudo a todo el disco (ver [[Lectura de Permisos]] y el patrón de escalada "disk grupal").

## Directorio `/etc`: la central de configuración

Directorio **estático** con **todos los archivos de configuración del sistema** y de los programas instalados. Es de los más auditados en seguridad: world-writable en `/etc` es escalada casi directa.

### Recorrido por los archivos que todo administrador toca

| Archivo/árbol | Contenido |
|---|---|
| `/etc/passwd` | Cuentas: `nombre:x:UID:GID:comentario:home:shell` (ver [[Comandos, Rutas y Operadores de Linux]]) |
| `/etc/shadow` | Hashes y aging de contraseñas (solo root; modo 640) |
| `/etc/group`, `/etc/gshadow` | Grupos y sus ajustes |
| `/etc/sudoers` + `/etc/sudoers.d/` | Reglas de sudo — editarse **solo** con `visudo` (valida sintaxis) |
| `/etc/fstab` | Qué se monta al arrancar: dispositivo, punto, tipo, opciones (`defaults`, `noexec`, `nosuid`, `nodev`) |
| `/etc/mtab` → `/proc/self/mounts` | Montajes actuales (symlink moderno) |
| `/etc/hosts` | Mapeo estático hostname→IP (precedencia sobre DNS según `/etc/nsswitch.conf`) |
| `/etc/resolv.conf` | Servidores DNS (a menudo gestionado por NetworkManager/systemd-resolved) |
| `/etc/nsswitch.conf` | **Orden** de resolución: files, dns, mdns… (quién manda en `getent hosts`) |
| `/etc/hostname`, `/etc/hosts` | Nombre del equipo |
| `/etc/shells` | Shells válidos para cuentas |
| `/etc/environment`, `/etc/profile`, `/etc/profile.d/*.sh`, `/etc/bash.bashrc` | Entorno global de login/shell |
| `/etc/passwd.d`? no — `/etc/pam.d/` | Configuración de PAM: cómo se autentica cada servicio (login, sshd, sudo) |
| `/etc/ssh/sshd_config` | Configuración del servidor SSH |
| `/etc/systemd/system/`, `/etc/systemd/user/` | Unidades systemd locales (sobrescriben `/usr/lib/systemd/system`) |
| `/etc/crontab`, `/etc/cron.d/`, `/etc/anacrontab` | Tareas programadas del sistema (ver [[Cron Jobs y Programación de Tareas]]) |
| `/etc/netplan/` (Ubuntu) · `/etc/NetworkManager/` · `/etc/systemd/network/` | Configuración de red según el gestor |
| `/etc/os-release` | Identidad de la distro (NAME, ID, VERSION_ID…) — fuente fiable para scripts |
| `/etc/fstab.d/`, `/etc/modules-load.d/` | Fragmentos de configuración por "drop-in" (patrón moderno) |

> [!tip] Convención de "drop-ins" y `.d/`
> El patrón moderno de configuración son directorios `/etc/<servicio>.d/` con fragmentos que se cargan en orden alfabético: `/etc/sysctl.d/`, `/etc/tmpfiles.d/`, `/etc/modprobe.d/`, `/etc/sudoers.d/`. Ventaja: los updates del paquete no pisan tu configuración (que vive en tu propio fragmento).

> [!warning] `/etc` editable ≠ `/etc` editable por todos
> La regla de oro: **los archivos de `/etc` son de root y no escritables por otros**. `find /etc -perm -0002` es una de las primeras auditorías de un sistema comprometido o mal configurado — ver [[Abuso de Permisos en Linux y LSE]].

## Directorio `/home`

Directorio **variable y compartible**: aloja los archivos **personales de cada usuario** (no del `root`). Cada usuario tiene `/home/<usuario>` con sus datos, dotfiles ocultos (`.bashrc`, `.config/`, `.local/`, `.ssh/`) y proyectos.

> [!info] Home del root separado
> `/home` no contiene al root; su home es `/root` (ver más abajo). Aislar el home del superusuario evita que la cuota/contaminación de datos de usuarios afecte al root, y permite montar `/home` aparte (incluso cifrado por usuario con eCryptfs/FSCrypt).

## Directorio `/lib` (y `/lib64`)

**Estático y compartible**: contiene las **bibliotecas compartidas** (`*.so`) necesarias para arrancar los ejecutables de `/bin` y `/sbin`. `libc.so.6`, módulos del kernel (`/lib/modules/$(uname -r)/`), firmware (`/lib/firmware/`).

> [!info] Módulos y firmware
> `lsmod` lista módulos cargados; los `.ko` viven en `/lib/modules/<versión>/`. `depmod` genera el índice, `modprobe` carga resolviendo dependencias. El firmware binario de dispositivos (wifi, GPU) vive en `/lib/firmware`. Como `/bin`, `/lib` suele ser symlink a `/usr/lib` (usr-merge).

## Directorio `/mnt`

Punto de montaje para **montajes manuales/temporales** de almacenamiento: discos externos a los que acceder una tarea, particiones de rescate, sistemas de archivos de otro SO.

```bash
# Patrón típico de rescate
sudo mount /dev/sda1 /mnt              # partición rota
ls /mnt                                # inspeccionar sin arrancarla
sudo umount /mnt
```

## Directorio `/media`

Igual función que `/mnt`, pero reservado a los **montajes automáticos de extraíbles** (USB, CD-ROM, tarjetas) gestionados por udisks/udev o el escritorio. Al conectar un USB: `/media/<usuario>/NOMBRE_DISCO`. Diferencia práctica: `/mnt` lo montás **vos** a mano; `/media` lo monta **el sistema**.

## Directorio `/opt`

Contenido **estático y compartible** de **programas de terceros** que no vienen con el sistema ni siguen la estructura FHS: Spotify, Google Chrome, TeamViewer, apps empresariales.

> [!tip] `/opt` = "todo en una carpeta"
> Cada aplicación ocupa su subdirectorio completo (`/opt/google/chrome`, `/opt/spotify`) con binarios, libs y datos propios. Desinstalación trivial (borrar el directorio), cero conflicto con el gestor de paquetes. Para software de terceros que **sí** sigue FHS, el destino es `/usr/local`.

### `/usr` vs `/opt` vs `/usr/local`: quién instala qué

| Directorio | Quién lo escribe | Ejemplo | ¿Lo gestiona apt/pacman/dnf? |
|---|---|---|---|
| `/usr` | **El gestor de paquetes** | `python`, `git`, `nginx` | **Sí — nunca editar/borrar a mano** |
| `/usr/local` | **El administrador** (compilaciones, `make install`) | nginx custom, scripts propios | No (pero debe mantenerse igual de ordenado) |
| `/opt` | **Vendor de terceros** (tarballs/app images/instaladores) | Chrome `.deb` extra-oficial, JBoss | No |

La ruta completa de un binario te dice su origen: `which htop` → `/usr/bin/htop` (paquete), `/usr/local/bin/htop` (tuya), `/opt/foo/bin/foo` (tercero).

## Directorio `/proc`: el kernel como archivos

`/proc` es **procfs**, un sistema de archivos **virtual en memoria** (cero bytes en disco) que expone los **procesos** y los parámetros del kernel como archivos de texto. Es la fuente de datos de `ps`, `top`, `free`, `netstat` y compañía.

### Qué contiene

| Ruta | Contenido |
|---|---|
| `/proc/<PID>/` | Un directorio **por cada proceso**: `cmdline` (argv completo, `\0`-separado), `environ` (su entorno), `cwd`, `exe` (symlink al binario), `root`, `fd/` (sus descriptores de archivo), `maps`, `status`, `stat`, `comm`, `io` |
| `/proc/meminfo` | RAM: total, free, buffers, cached (`free` lo lee) |
| `/proc/cpuinfo` | CPU: modelo, cores, flags (comprobación de `vmx`/`svm` para virtualización) |
| `/proc/loadavg` | Cargas 1/5/15 min + proceso más activo |
| `/proc/uptime` | Segundos encendido |
| `/proc/filesystems` | Filesystems soportados por el kernel |
| `/proc/mounts` | Montajes actuales (idéntico a `/etc/mtab`) |
| `/proc/modules` | Módulos cargados (`lsmod`) |
| `/proc/sys/` | **Parámetros del kernel** en tiempo real (tunable: `/proc/sys/net/ipv4/ip_forward` = 1 habilita IP forwarding). Los cambios son hasta el siguiente reboot; para persistencia: `/etc/sysctl.d/*.conf` |
| `/proc/net/` | Vistas de red: `tcp`, `udp`, `arp`, `dev`, `route`, `wireless` — lo que parsean `ss`/`netstat` |
| `/proc/self/` | Symlink al proceso **que lo consulta** — clave para `/proc/self/fd/1` y scripts que encuentran su propio PID (`/proc/self/status`) |

> [!tip] Trucos forenses con `/proc`
> - Ver el binario que corre un PID sospechoso: `ls -l /proc/PID/exe` (incluso si fue borrado del disco: muestra `(deleted)`).
> - Ver sus argumentos y entorno: `tr '\0' ' ' < /proc/PID/cmdline`, `tr '\0' '\n' < /proc/PID/environ`.
> - Ver qué archivos tiene abiertos: `ls -l /proc/PID/fd` (equivalente a `lsof -p PID`).
> - Leer la memoria del proceso: `/proc/PID/mem` (solo su propio dueño o root).

> [!warning] `/proc` consume entradas, no RAM por archivo
> Cada proceso genera decenas de entradas en `/proc`. `find /proc` recorre miles de archivos efímeros: por eso se poda en búsquedas (`find / \( -path /proc -o -path /sys \) -prune -o …`, ver [[Búsquedas a nivel de sistema]]). Los tamaños reportados (`stat /proc/PID`) son ficticios (0 bytes reales).

## Directorio `/sys`: sysfs, el modelo estructurado del kernel

`/sys` es **sysfs**: como `/proc` pero **estructurado por modelo de dispositivos**, no por procesos. Expone el árbol de hardware y los ajustes de drivers.

| Ruta | Contenido |
|---|---|
| `/sys/class/net/` | **Una entrada por interfaz de red**: `mtu`, `address` (MAC), `flags`, `operstate`, `speed` — lo que lee `ip link` (ver [[Tipos de Interfaces de Red]]) |
| `/sys/class/block/` | Discos y particiones: `size` (en sectores de 512 B), `ro`, `removable` |
| `/sys/class/power_supply/` | Baterías y AC |
| `/sys/class/backlight/` | Brillo de pantalla |
| `/sys/bus/pci/devices/`, `/sys/bus/usb/devices/` | Árbol de buses: IDs vendor:device, drivers enlazados |
| `/sys/devices/system/cpu/cpuN/online` | Online/offline de CPUs (hotplug) |
| `/sys/kernel/` | Parámetros del kernel (también vía `/proc/sys`) |
| `/sys/fs/` | Mountpoints de subsystemos (cgroup, bpf…) |

Escritura en `/sys` = **cambio de configuración del hardware en vivo** (p. ej. `echo 1 > /sys/class/leds/.../brightness`), siempre con permisos adecuados.

## Directorio `/root`

Directorio **variable, no compartible**: es el **home del superusuario**. Contiene sus dotfiles, scripts y claves SSH del root.

> [!warning] `/root` separado de `/home`
> Mantenerlo aparte aísla la configuración y datos del administrador, permite políticas de permisos propias (típicamente `700`) y evita que el montaje o cuota de `/home` afecte al root.

## Directorio `/sbin`

Directorio **estático y compartible** como `/bin`, pero con los binarios de **administración que usa el sistema/root**: `fdisk`, `mkfs`, `fsck`, `mount`, `umount`, `reboot`, `shutdown`, `iptables`, `sysctl`, `ip` (en algunas distros), `sshd`. Usuario normal en el PATH solo lo tiene si su shell lo incluye; la administración los invoca por ruta o con sudo. Hoy symlink a `/usr/sbin` en los sistemas usr-merged.

## Directorio `/srv`

Datos **específicos de servicios** que el administrador decide publicar: `/srv/www` (Apache/Nginx en algunas convenciones), `/srv/ftp`, `/srv/samba`. Es el directorio "por si quieres separar datos de servicios del software", pero su uso es optativo — muchas distros dejan `/var/www` (Debian) o el raíz del servicio.

## Directorio `/tmp`

Donde viven los **archivos temporales de programas** (perfiles Firefox, sockets de apps, temporales de compilación).

> [!warning] Semántica de `/tmp`
> - Es **volátil**: se limpia en cada reboot (o por `systemd-tmpfiles` con la política `tmpfiles.d`, que borra archivos con más de 10 días por defecto).
> - En muchos sistemas se monta como **tmpfs** (RAM): rápido, pero consume memoria y desaparece al apagar.
> - Es **world-writable con sticky bit** (`1777`): cualquiera crea archivos, pero solo su dueño (o root) los borra/renombraba. Este diseño es **vectores de ataque clásicos** (symlink races, archivos pre-posicionados) — ver [[Abuso de Permisos en Linux y LSE]].
> - **Nunca** guardes nada importante en `/tmp`.

### `/tmp` vs `/var/tmp`: dos ideas de "temporal"

| | `/tmp` | `/var/tmp` |
|---|---|---|
| ¿Sobrevive reboot? | **No** | **Sí** |
| Filesystem típico | tmpfs (RAM) | Disco (mismo que `/`) |
| Limpieza | tmpfiles.d al arrancar/periodicamente | A discreción del admin |
| Uso | temporales efímeros (cachés, sockets) | temporales grandes/largos que no deben perderse entre sesiones (p. ej. actualizaciones de paquetes en curso) |

## Directorio `/usr`: el sistema operativo "de verdad"

Directorio **estático y compartible** con la **gran mayoría del software instalado**; su contenido es accesible para todos los usuarios y se considera **solo lectura** para el administrador (lo escribe el gestor de paquetes). Originalmente "Unix System Resources" (no "user"). Subdirectorios clave:

- `/usr/bin` — binarios de usuario (la mayoría de los ejecutables).
- `/usr/sbin` — binarios de administración no esenciales al arranque.
- `/usr/lib` — bibliotecas, y también `systemd/system/` con las unidades de los paquetes.
- `/usr/include` — cabeceras C para compilar (`gcc`, módulos del kernel).
- `/usr/share` — datos compartibles entre arquitecturas: man pages (`man/man1/…`), doc, iconos, i18n (`locale/`), dict, terminfo.
- `/usr/src` — fuentes (kernel headers, código fuente de paquetes).
- `/usr/local` — jerarquía aparte para software **del admin** (bin, sbin, lib, etc): se actualiza sin tocar el gestor de paquetes.

## Directorio `/var`: los datos que cambian

Directorio **variable**: logs, spool, cachés, colas de impresión, bases de datos, www. El FHS lo trata como el área de **crecimiento y diagnóstico** del sistema.

- `/var/log` — logs: `syslog`/`messages`, `auth.log`/`secure` (autenticaciones), `kern.log`, `dpkg.log`/`pacman.log`, logs de servicios. **Rotación**: `logrotate` (`/etc/logrotate.conf`, `/etc/logrotate.d/`) recorta comprimiendo; sin rotación, un log crece hasta llenar el disco.
- `/var/spool` — datos en cola: `cron/crontabs/` (tablas de usuarios, ver [[Cron Jobs y Programación de Tareas]]), `anacron/`, `mail/`, `cups/` (trabajos de impresión).
- `/var/cache` — cachés re-generables (`/var/cache/apt`, `pacman/pkg`): se pueden limpiar sin romper nada.
- `/var/lib` — **estado persistente** de servicios: `/var/lib/apt/lists`, `/var/lib/postgresql`, `/var/lib/docker`. Borrarlo es perder datos, no caché.
- `/var/tmp` — temporales persistentes entre reinicios (ver tabla anterior).
- `/var/www` — raíz web por convención Debian/Ubuntu.
- `/var/run` → symlink a `/run` (transición histórica).

> [!tip] `/var` en su propia partición
> Un `/var` separado evita que logs o spool desbordados dejen el sistema raíz sin espacio (imposible escribir en `/`, fallo de arranque o de servicios críticos). En contenedores, `/var/log` suele mandarse a stdout/journal en su lugar.

## `/run`: el estado del arranque actual

`/run` (antes `/var/run`) es un **tmpfs montado en cada arranque** con el **estado de la sesión actual**: PIDs de daemons (`/run/sshd.pid`, `/run/docker.pid`), sockets (`/run/systemd/`, `/run/docker.sock`), locks (`/run/lock/`), credenciales efímeras. Su regla de oro: **todo lo que hay en `/run` es válido solo hasta el próximo reboot** — no debe persistirse ahí nada que el sistema no pueda regenerar. Ventaja de tmpfs: los sockets y PID files no tocan disco y desaparecen solos tras un crash.

## `tmpfs`: qué es cuando "no hay disco"

Varios directorios (`/run`, `/tmp` en muchos sistemas, `/dev/shm`, `/sys/fs/cgroup`…) son **tmpfs**: un filesystem **en RAM** que el kernel expone como directorio. Características: lecturas/escrituras tan rápidas como la RAM, tamaño máximo limitado (por defecto ~50% de RAM, visible en `df -h /tmp`), volátil. `/dev/shm` es el tmpfs compartido entre procesos (memoria compartida POSIX: `shm_open`) — por eso `df -h /dev/shm` siempre aparece.

## Diferencias Debian/Ubuntu vs RHEL/Fedora vs Arch

| Aspecto | Debian/Ubuntu | RHEL/CentOS/Fedora | Arch |
|---|---|---|---|
| Log de autenticación | `/var/log/auth.log` | `/var/log/secure` | journal |
| Log general | `/var/log/syslog` | `/var/log/messages` | journal |
| Servicios | `systemd` (+ histórico SysV en `/etc/init.d`) | `systemd` | `systemd` |
| Gestor de paquetes | `apt`/`dpkg` | `dnf`/`rpm` | `pacman` |
| Config de red | NetworkManager + `/etc/netplan/` (Ubuntu) | NetworkManager o `/etc/sysd/network/` | NetworkManager o systemd-networkd |
| Grupos de administración | `sudo` (su `adm` para logs) | `wheel` | `wheel` |
| Auditoría de logs | `adm` lee `auth.log` | `wheel` lee `secure` | journal (política) |
| Firewall | `ufw` (sobre nftables/iptables) | `firewalld` | `nftables`/iptables manual |
| `/usr` fusionado | Sí (desde 21.04 en Ubuntu; Debian bookworm+ por defecto) | Sí (Fedora desde hace años) | Sí (releases recientes) |
| `ifconfig`/`netstat` | Paquete `net-tools` opcional | Desinstalado por defecto | No instalado |

> [!info] Lo portable entre distros
> La jerarquía FHS (`/etc`, `/var`, `/usr`, `/tmp`…) es común a todas; las diferencias están en **herramientas y rutas de logs**. Para scripts portables: detectar la distro con `. /etc/os-release && echo $ID` y ajustar, o preferir `journalctl` (disponible en cualquier systemd).

## Cheatsheet

```bash
ls -ld /bin /lib            # ¿usr-merged? (symlink → usr/*)
cat /etc/os-release         # identidad de la distro
ls /proc/1/cmdline | tr '\0' ' '    # cmdline del init (systemd)
grep MemTotal /proc/meminfo;  cat /proc/cpuinfo | grep vmx   # RAM / virtualización
ls /sys/class/net           # interfaces de red visibles para el kernel
find /etc -perm -0002       # configuración world-writable (¡auditoría!)
df -hT; df -i               # discos por montaje (con tipo FS) / inodos
mount | grep tmpfs          # qué es tmpfs (¿/tmp en RAM?)
ls -l /proc/PID/exe /proc/PID/fd/   # binario y archivos abiertos de un proceso
readlink -f /bin/ls         # resolución canónica (symlinks incluidos)

# Mapa mental
/etc config · /var datos variables · /usr software del sistema (paquetes)
/usr/local software tuyo · /opt terceros · /run y /tmp volátiles (tmpfs)
/proc y /sys el kernel hecho archivos · /dev hardware como nodos
```

## Relacionadas

- [[Comandos Básicos de Sistema y Red en Linux]] — `ls`, `du`, `df`, `stat` para explorar
- [[Comandos, Rutas y Operadores de Linux]] — rutas absolutas/relativas y symlinks
- [[Lectura de Permisos]] — los bits sobre nodos en `/dev` y directorios críticos
- [[Búsquedas a nivel de sistema]] — `find` con poda de `/proc` y `/sys`
- [[Cron Jobs y Programación de Tareas]] — `/etc/cron*` y `/var/spool/cron`
- [[Tipos de Interfaces de Red]] — `/sys/class/net` y la mecánica de interfaces
- [[Abuso de Permisos en Linux y LSE]] — `/tmp`, `/etc` y directorios escribibles como vectores

## Referencias

- `man 7 file-hierarchy` (systemd), `man 5 proc`, `man 5 sysfs`, `man 5 fstab`, `man 5 tmpfs`
- FHS 3.0: <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html>
- Debian Administrator's Handbook (estructura del sistema): <https://debian-handbook.info/browse/stable/>
- ArchWiki — Filesystem Hierarchy: <https://wiki.archlinux.org/title/Filesystem_Hierarchy>
