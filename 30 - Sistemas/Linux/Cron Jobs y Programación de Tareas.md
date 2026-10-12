---
tags:
  - sistemas
  - linux
  - cron-tareas
tipo: concepto
estado: terminado
---

# Cron Jobs y Programación de Tareas

Los **cron jobs** son tareas que el demonio `cron` ejecuta en intervalos de tiempo definidos por el usuario o el sistema. Dominar su sintaxis, su entorno de ejecución y sus puntos de instalación es doblemente importante: es administración básica **y** es una de las primeras cosas que se auditan (y abusan) en escalada de privilegios — ver [[Detección y Explotación de Tareas Cron]].

> [!info] Quién ejecuta qué: los tres "cron"
> En un Linux moderno conviven:
> 1. **`cron` del usuario** (`crontab -e`): tablas personales en `/var/spool/cron/crontabs/<usuario>` (Debian) o `/var/spool/cron/<usuario>` (RHEL/Arch), ejecutándose **como ese usuario**.
> 2. **`cron` del sistema**: archivos en `/etc/cron.d/`, `/etc/crontab` y `/etc/cron.{hourly,daily,weekly,monthly}`, ejecutados como el usuario indicado (o root).
> 3. **`anacron`**: garante de que las tareas *diarias/semanales/mensuales* se ejecuten **aunque la máquina estuviera apagada** en su hora — crítico en escritorios/portátiles. En servidores 24/7 con systemd, su rol lo tomaron los **systemd timers**.

## Estructura de una tarea cron

La tabla se compone de **cinco campos de tiempo** seguid del comando:

![[Pasted image 20240608221843.png]]

Los cinco asteriscos (`* * * * *`) significan "cada minuto de cada hora, de cada día del mes, de cada mes, de cada día de la semana" — es decir, **cada minuto**.

> [!info] Granularidad mínima: 1 minuto
> La granularidad estándar de cron es el minuto; no existe sintaxis para sub-minuto. Para ejecuciones más finas: un bucle con `sleep` dentro de un script lanzado cada minuto, o un **systemd timer** con `OnUnitActiveSec=30s`.

Y no todos los campos son asterisco:

![[Pasted image 20240608221902.png]]

## Tabla completa de campos

| Pos | Campo | Rango | Valores especiales |
|---|---|---|---|
| 1 | **Minuto** | 0–59 | — |
| 2 | **Hora** | 0–23 | — |
| 3 | **Día de mes** | 1–31 | — |
| 4 | **Mes** | 1–12 o Ene–Dic | — |
| 5 | **Día de semana** | 0–7 (0 y 7 = **Domingo**) o Dom–Sáb | — |
| 6+ | **Comando** | Cualquier línea de shell | `%` tiene significado especial (ver abajo) |

### Operadores dentro de cada campo de tiempo

| Operador | Significado | Ejemplo |
|---|---|---|
| `*` | Cualquier valor | `* * * * *` = cada minuto |
| `N` | Valor exacto | `0 3 * * *` = a las 3:00 |
| `a-b` | Rango inclusivo | `0 9-17 * * *` = cada hora de 9 a 17 |
| `*/N` | Paso desde el inicio del rango | `*/5 * * * *` = cada 5 min |
| `a-b/N` | Paso dentro de un rango | `0-59/15 * * * *` = 0,15,30,45 |
| `a,b,c` | Lista (también rangos mezclados) | `0 8,12,18 * * *` · `0 2 * * 1-5` (lunes a viernes) |

> [!tip] `*/5` y `0-59/5` son equivalentes
> El operador de paso (`/`) especifica intervalos: `*/N` = "cada N unidades desde 0". `*/2` en hora = cada 2 horas (0,2,4…); `*/10` en minuto = 0,10,20,30,40,50.

![[Pasted image 20240608221925.png]]

La lectura de `*/5 * * * *` es: se ejecutará a las 1:05, 1:10, 1:15… 2:05, 2:10… durante las 24 horas.

![[Pasted image 20240608221952.png]]

> [!warning] Día de mes **Y** día de semana = OR, no AND
> Si en la misma línea tenés día-de-mes **y** día-de-semna definidos (no `*`), cron ejecuta cuando **cualquiera** de los dos coincida. `0 0 1 * 1` es "el día 1 de cada mes **o** cada lunes", no "el lunes 1". Para el segundo caso hay que filtrar dentro del comando.

### Ejemplos de lectura

| Línea | Significado |
|---|---|
| `* * * * *` | Cada minuto |
| `*/5 * * * *` | Cada 5 minutos |
| `0 * * * *` | Cada hora, en punto (min 0) |
| `30 2 * * *` | Cada día a las 2:30 |
| `0 9 * * 1-5` | Lunes a viernes a las 9:00 |
| `0 0 1 1 *` | 1 de enero a medianoche |
| `15 14 1 * *` | El día 1 a las 14:15 |
| `0 22 * * 1-5` | Días laborables a las 22:00 |
| `0 0 * * 0` | Cada domingo a medianoche |
| `20,40 * * * *` | A los minutos 20 y 40 de cada hora |
| `*/10 9-17 1-15 * 1-5` | (recordar OR) — mala idea; filtrar en el comando |

## Gestión de crontabs

| Comando | Qué hace |
|---|---|
| `crontab -e` | Edita **tu** tabla con el editor por defecto (`$EDITOR`) |
| `crontab -l` | Lista tu tabla |
| `crontab -l -u usuario` | Lista la tabla de otro (root) |
| `crontab archivo` | **Instala** un archivo como tu tabla (pisa la existente) |
| `crontab -r` | **Elimina** tu tabla completa (sin confirmación en algunas versiones: `-i` para preguntar) |
| `crontab -i archivo` | Instala preguntando antes |
| `systemctl list-timers --all` | Timers systemd (la alternativa moderna, ver abajo) |

```bash
# Flujo típico
crontab -l > respaldo.cron             # siempre respaldar antes
crontab respaldo.cron                  # restaurar
(crontab -l ; echo '30 3 * * 0 /usr/local/bin/backup.sh') | crontab -   # añadir sin editar
```

### Ubicaciones: usuario vs sistema

| Ubicación | Dueño de la ejecución | Formato | Quién edita |
|---|---|---|---|
| `/var/spool/cron/crontabs/USER` (Debian) · `/var/spool/cron/USER` (RHEL/Arch) | El **usuario** (cada línea se ejecuta como él; sin campo USER) | 5 campos + comando | Solo vía `crontab -e` (o root editando directo) |
| `/etc/crontab` | root (o el indicado) | **6 campos**: el 6º es **USER** antes del comando | Administrador |
| `/etc/cron.d/ARCHIVO` | root (o el indicado) | Igual que `/etc/crontab` (con USER) + puede tener `SHELL=`, `PATH=` | Administrador / paquetes |
| `/etc/cron.hourly/`, `daily/`, `weekly/`, `monthly/` | root | **solo scripts ejecutables**, sin campos de tiempo | Ejecutados por `run-parts` según `/etc/cron.d/` |
| `/var/spool/cron/cronjobs/` o `/etc/cron.d/` (Debian: spool de at) | — | Trabajos `at` (ejecución única) | `at now + 5 minutes` |

> [!warning] Diferencia crítica de formato
> Los crontabs **de usuario** NO llevan el campo de usuario; `/etc/crontab` y `/etc/cron.d/*` **sí** llevan USER antes del comando. Copiar una línea de un formato a otro produce ejecución como root o jobs que directamente fallan con `bad minute` / `(USER) COMMAND (COMMAND NOT FOUND)`.

```bash
# /etc/cron.d/backup — ejemplo con campos de entorno
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
30 3 * * 0 root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

## El entorno de cron: la mitad de los misterios

Cron ejecuta con un entorno **mínimo y predecible**, y ese es el origen de casi todos los "en mi terminal funciona":

| Aspecto | Comportamiento real |
|---|---|
| **`$PATH`** | Fijo y corto: `/usr/bin:/bin` (en algunos `/usr/sbin:/sbin` añadidos). **No** incluye `~/.local/bin`, `/usr/local/bin` en algunas distros, ni nada exportado en tu `.bashrc` |
| **`$HOME`** | Seteado **solo** en los crontabs de usuario (al home del dueño). En `/etc/cron.d` **no** está definido |
| **`$SHELL`** | Por defecto `/bin/sh` (¡en Debian sh = dash!). Declará `SHELL=/bin/bash` si necesitás bash |
| **Variables de tu sesión** | **No** se heredan. Nada de lo que exportaste en `.bashrc`/`.profile` existe aquí |
| **`%` en el comando** | Significado especial: cada `%` se reemplaza por una **nueva línea** antes de ejecutar (como here-doc). El **primer** `%` escapa el comando de stdin. Escapá siempre: `\%` o usá comillas |
| **Salida** | Si no se redirige, cron intenta **enviarla por correo** al dueño del job (mail local). Sin MTA configurado: se pierde o llena `/var/mail` |
| **Redirigir todo** | La receta segura: `>> /var/log/mijob.log 2>&1` |
| **cwd** | El home del usuario (usuario) o `/` (system cron). No asumás el directorio actual: usá rutas absolutas |
| **Acceso a TTY** | No hay TTY: programas que necesitan terminal (`ssh -t`, editores, `su` interactivo) fallan |

```bash
# Mal: % interpretable, PATH ambiguo, sin log
0 * * * * python3 script.py

# Bien: PATH explícito, rutas absolutas, log y entorno explícito
PATH=/usr/local/bin:/usr/bin:/bin
0 * * * * /usr/bin/python3 /opt/miapp/script.py >> /var/log/miapp.log 2>&1
```

> [!warning] El `%` de cron: la trampa clásica
> `0 * * * * echo "100%"` no hace lo que parece: cron convierte cada `%` en salto de línea y pasa el resto como **stdin** al comando. Solución: `echo "100\%"` o `echo '100%'` (con comillas el shell no lo ve, pero cron sí procesa el `%` interior — por eso `\%` es lo robusto). Regla: **en líneas de cron, escapá todos los `%`**.

## Logging: dónde quedan registradas las ejecuciones

| Distro | Registro | Contenido |
|---|---|---|
| Debian/Ubuntu | `/var/log/syslog` (filtro `grep CRON`) | `CRON[pid]: (root) CMD (/ruta/comando)`, instalaciones de tabla, errores |
| RHEL/CentOS/Fedora | `/var/log/cron` | Mismo formato |
| Arch | journal (`journalctl -u cronie` o `grep CRON /var/log/cron.log` si syslog-ng) | — |
| Cualquiera | log propio del job | **Recomendado**: `>> /var/log/mijob.log 2>&1` en la propia línea |

```bash
grep CRON /var/log/syslog | tail -20       # ¿corrió mi job?
journalctl -u cron --since today           # si el cron está como unidad systemd
```

Para depurar: agregar `date` y `env` al principio del job (`/bin/date >> log` como línea extra) deja huella de cuándo corrió y con qué entorno.

## `anacron`: garantizar la ejecución tras apagados

`anacron` no programa *a qué hora* sino *con qué frecuencia mínima*; ejecuta tareas **olvidadas** al arrancar o periódicamente.

| Campo `/etc/anacrontab` | Significado |
|---|---|
| `period` | Frecuencia en días: `1` (diario), `7` (semanal), `30` (mensual) o nombre (`@daily`) |
| `delay` | Espera tras el arranque antes de correr la tanda (p. ej. `5` minutos — da tiempo a que el sistema estabilice) |
| `job-identifier` | Nombre usado en el **stamp file** de `/var/spool/anacron/` para saber si ya corrió en este periodo |
| `comando` | El script a ejecutar (ruta absoluta) |

```cron
# /etc/anacrontab (extracto típico)
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/bin
1   5   cron.daily     run-parts /etc/cron.daily
7   10  cron.weekly    run-parts /etc/cron.weekly
@monthly 15 cron.monthly run-parts /etc/cron.monthly
```

Si la máquina estuvo apagada el domingo 3:00, anacron la ejecutará al encender (tras el `delay`). Por eso en servidores 24/7 el anacron es redundante, y en portátiles es **imprescindible**.

## systemd timers: el reemplazo moderno

Los timers (`.timer` + `.service`) reemplazan a cron con ventajas: dependencias, persistencia (`Persistent=true` ejecuta lo perdido tras apagado), notificaciones, `systemctl list-timers` unificado y logs en journal.

```ini
# /etc/systemd/system/limpieza.service
[Unit]
Description=Limpieza de /tmp
[Service]
Type=oneshot
ExecStart=/usr/local/bin/limpiar.sh

# /etc/systemd/system/limpieza.timer
[Unit]
Description=Limpieza diaria
[Timer]
OnCalendar=*-*-* 03:30:00     # calendario: Año-Mes-Día Hora:Min:Seg
Persistent=true               # si el equipo estaba apagado a las 3:30, corre al encender
[Install]
WantedBy=timers.target
```

| Cron | systemd timer equivalente |
|---|---|
| `* * * * *` (cada min) | `OnUnitActiveSec=1min` |
| `0 * * * *` (cada hora) | `OnCalendar=hourly` o `*-*-* 00..23:00:00` |
| `30 3 * * 0` (domingos 3:30) | `OnCalendar=Sun *-*-* 03:30:00` |
| — (correr tras apagado) | `Persistent=true` |
| — (tras el arranque) | `OnBootSec=5min` |
| — (cada N tras el último) | `OnUnitActiveSec=1h` |

```bash
systemctl list-timers --all        # qué timers hay y cuándo corren (incl. cron.*
systemctl enable --now limpieza.timer
journalctl -u limpieza.service     # logs del job
```

### ¿Cron, anacron o timer?

| Criterio | Cron | Anacron | Timer |
|---|---|---|---|
| Granularidad | Minuto | Día | Cualquiera (incluso segundos) |
| Ejecuta si la máquina estuvo apagada | No (salvo cronie + anacron) | **Sí** | **Sí** (`Persistent`) |
| Logs unificados | Syslog propio | Stamp files | journalctl |
| Dependencias / sandboxing | No | No | Sí (`Requires=`, `ProtectSystem=`) |
| Servidor 24/7 | ✔ | innecesario | ✔ |
| Portátil/escritorio | parcial | **✔** | ✔ con Persistent |

## Recetas

```bash
# 1. Backup diario a las 3:30 con log
crontab -e
#  30 3 * * * /usr/bin/tar czf /var/backups/home-$(date +\%F).tar.gz /home >> /var/log/backup.log 2>&1
#  (el \% porque $(date +\%F) pasa por cron antes que por el shell)

# 2. Refresco de certificados Let's Encrypt semanal
0 4 * * 1 /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx" >> /var/log/letsencrypt.log 2>&1

# 3. Enviar métricas cada 5 minutos con entorno explícito
PATH=/usr/local/bin:/usr/bin
*/5 * * * * /usr/bin/python3 /opt/metrics/enviar.py --token $MI_TOKEN >> /var/log/metrics.log 2>&1
# (MI_TOKEN debe definirse EN el crontab, no hereda tu shell)

# 4. Limpiar /tmp cada domingo
0 6 * * 0 /usr/bin/find /tmp -type f -atime +7 -delete

# 5. Trabajo one-shot con at (ejecución única, sin tabla)
echo '/usr/local/bin/reinicio_limpio.sh' | at now + 30 minutes
atq        # listar    atrm ID  # cancelar

# 6. Auditoría rápida de todos los jobs del sistema
for f in /etc/crontab /etc/cron.d/*; do echo "== $f"; grep -v '^#' "$f"; done
ls /var/spool/cron/crontabs/ 2>/dev/null        # (Debian) tablas por usuario
crontab -l -u root; ls /etc/cron.*              # lo habitual en pentest
```

## Endurecimiento y buenas prácticas

- **Rutas absolutas** a los binarios y a los argumentos: cron tiene PATH corto y es vector de PATH poisoning (ver [[PATH Hijacking]]).
- **Privilegio mínimo**: el job corre como root **solo si** está en `/etc/crontab`/`cron.d` como root o en el crontab de root. Crear un usuario dedicado para tareas no críticas.
- **Logs propios** con `>> archivo 2>&1`; no depender del correo local.
- **Permisos de los scripts**: dueño root y no escritables por otros (`755`), o un root puede ser secuestrado vía cron (ver [[Abuso de Permisos en Linux y LSE]]).
- **Entorno declarado** (`PATH=`, `SHELL=`, variables necesarias) dentro del propio crontab.
- **Escapar `%`** siempre.
- **Respaldar antes de editar**: `crontab -l > respaldo.cron`.
- En auditorías: mirar también los **contenidos de los scripts** referenciados, no solo la tabla — un job de root ejecutando `/opt/script.sh` world-writable es escalada directa.

## Errores comunes

1. **`-w` al editar de root sin `visudo`-like check** — `crontab -e` de root valida; editar `/etc/cron.d/*` a mano puede dejar la tabla rota (cron la ignora entera con error en syslog: revisar logs tras editar).
2. **`COMMAND NOT FOUND` en syslog** — PATH del entorno de cron no contiene el binario; ruta absoluta.
3. **El job "no hace nada"** — sin log y sin MTA la salida desaparece; añadir redirección a archivo.
4. **Fecha DST** — en cambios de hora, un job programado en la hora que "no existe" puede no correr o correr dos veces; timers con `OnCalendar` lo manejan mejor.
5. **`5 * * * *` interpretado como "cada 5 minutos"** — no: es "minuto 5 de cada hora". `*/5` es el paso.
6. **Campo de semana malinterpretado** — `0` y `7` son domingo; `7` solo existe por compatibilidad.
7. **Cron de usuario en `/etc/cron.d`** (o viceversa) — formato distinto (campo USER); verificar `bad minute` en syslog.

## Cheatsheet

```bash
# Sintaxis:  min hora dia-mes mes dia-semana comando
* * * * *        cada minuto        */5 * * * *   cada 5 min
0 9 * * 1-5      lunes a viernes 9h  0 0 1 * *     día 1 a medianoche
# Usuario:  crontab -e / -l / -r / archivo / (crontab -l; echo 'NUEVA') | crontab -
# Sistema: /etc/crontab · /etc/cron.d/* (incluyen campo USER) · /etc/cron.{hourly,daily,weekly,monthly}
# Anacron:  /etc/anacrontab → period delay id comando  (ejecuta lo perdido tras apagadas)
# Timers:   OnCalendar=Sun *-*-* 03:30:00 · Persistent=true · systemctl list-timers --all
# Logs:     grep CRON /var/log/syslog · journalctl -u unidad
# Reglas:   rutas absolutas · PATH= explícito · escapar % · redirigir >> log 2>&1 · privilegio mínimo
```

## Relacionadas

- [[Detección y Explotación de Tareas Cron]] — el lado ofensivo: encontrar y abusar jobs
- [[Abuso de Permisos en Linux y LSE]] — scripts world-writable invocados por root
- [[Bash Scripting - Fundamentos Básicos]] — escribir los scripts que cron ejecuta
- [[Comandos, Rutas y Operadores de Linux]] — redirecciones y `$PATH` usados aquí
- [[Estructura de Directorios de Linux]] — `/var/spool`, `/var/log`, `/etc`

## Referencias

- `man 5 crontab`, `man 5 anacrontab`, `man 8 anacron`, `man 5 systemd.timer`
- Cronie (cron con anacron integrado): <https://wiki.archlinux.org/title/Cron>
- systemd timers: <https://wiki.archlinux.org/title/Systemd/Timers>
- Debian Administrator's Handbook, tareas programadas: <https://debian-handbook.info/browse/stable/sect-tasks-scheduled.html>
