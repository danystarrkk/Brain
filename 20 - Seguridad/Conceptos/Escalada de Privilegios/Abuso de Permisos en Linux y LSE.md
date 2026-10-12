---
aliases:
  - Linux Smart Enumeration
  - LSE
  - Permisos Linux
tags:
  - sec/priv-esc
  - sec/linux
  - sec/enumeracion
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Lectura de Permisos]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Linux Capabilities]]"
  - "[[Abuso de Privilegios Sudoers]]"
  - "[[Detección y Explotación de Tareas Cron]]"
  - "[[Grupos de Linux y Escalada de Privilegios]]"
  - "[[PATH Hijacking]]"
  - "[[Vulnerabilidades del Kernel y Escalada de Privilegios]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
  - "[[Nmap Guía Completa]]"
fuente: "https://github.com/diego-treitos/linux-smart-enumeration"
---

# Abuso de Permisos en Linux y LSE

La escalada de privilegios en Linux no empieza con exploits: empieza con **enumeración**. Este nota cubre las dos caras del mismo problema: el **modelo DAC de permisos** (cómo decide el kernel quién puede leer/escribir/ejecutar) y el flujo de trabajo de reconocimiento — manual y con **LSE** (Linux Smart Enumeration) — para encontrar la decisión mal tomada.

> [!info] Representación de Permisos
> Los permisos se representan comúnmente de dos formas:
> *   **Simbólica**: `rwx` para el propietario, `r-x` para el grupo, `r--` para otros. Por ejemplo, `chmod u=rwx,g=rx,o=r archivo.txt`.
> *   **Octal**: Un número de tres o cuatro dígitos. Cada permiso (r, w, x) tiene un valor numérico: `r=4`, `w=2`, `x=1`. La suma de estos valores para cada categoría forma el dígito octal. Por ejemplo, `rwx` es `4+2+1=7`, `r-x` es `4+0+1=5`, `r--` es `4+0+0=4`. Así, `rwxr-xr--` se representa como `754`. El primer dígito opcional (ej. `4755`) se refiere a permisos especiales como SUID, SGID o Sticky Bit.

El abuso de permisos incorrectamente implementados ocurre cuando los permisos de un archivo crítico son configurados de forma errónea, permitiendo a un usuario no autorizado acceder o modificar el archivo. Esto puede permitir a un atacante leer información confidencial, modificar archivos importantes, ejecutar comandos maliciosos o incluso obtener acceso de superusuario (root) al sistema.

> [!warning] Riesgos de Permisos Incorrectos
> Un archivo con permisos de escritura para "otros" (`o+w`) en una ubicación crítica (como `/etc/passwd` o scripts de inicio) puede ser modificado por cualquier usuario del sistema, lo que podría llevar a la creación de nuevos usuarios, la modificación de contraseñas o la inyección de código malicioso que se ejecute con privilegios elevados.

## Mecánica: cómo evalúa el kernel los permisos

Todo permiso en Linux se reduce a tres identidades evaluadas **en orden**:

| Paso | Identidad | Pregunta | Si coincide, se usan esos permisos |
|---|---|---|---|
| 1 | **Owner** (`u`) | `EUID == uid del archivo`? | rwx del primer triplete |
| 2 | **Group** (`g`) | ¿el EUID está en el GID del archivo (directo o suplementario)? | rwx del segundo triplete |
| 3 | **Otros** (`o`) | cualquier otro | rwx del tercer triquete |

Y una excepción por encima de todo: **root (`EUID=0`) salta la comprobación DAC** (lee/escribe/cualquier archivo), salvo cuando el filesystem es de solo-lectura o hay capabilities/LSM de por medio. Esto explica por qué "ser root" es el objetivo y por qué los bits especiales existen: SUID hace que un binario se ejecute con el **EUID de su dueño** (ver [[Privilegios SUID en Linux]]).

Semántica de cada bit según tipo de objeto — se suele memorizar mal:

| Bit | Archivo normal | Directorio | Script ejecutable |
|---|---|---|---|
| `r` | ver contenido (`cat`) | listar nombres (`ls`) | — |
| `w` | modificar contenido | crear/borrar entradas en él | — |
| `x` | ejecutar | **entrar** (`cd`) | ejecutar por intérprete |

> [!tip] La trampa clásica del directorio
> Un directorio `rwx` para otros permite `ls` pero **no** `cd` (falta `x`); un directorio `r-x` permite entrar pero no crear archivos. Muchos "no puedo entrar a /opt/app" se resuelven con `x`, no con `r`. Y al revés: `o+w` en un directorio = cualquier usuario puede plantar un archivo ahí (vector directo de [[PATH Hijacking]] si el directorio cae en el `PATH` de un proceso privilegiado).

### Permisos especiales (cuarto dígito)

| Valor | Nombre | En archivo | En directorio |
|---|---|---|---|
| `4` | SUID | ejecuta con EUID del dueño | ignorado |
| `2` | SGID | ejecuta con EGID del grupo | archivos nuevos heredan el grupo |
| `1` | Sticky | ignorado | solo el dueño borra sus entradas (`/tmp`) |

```bash
# Auditoría de permisos en un vistazo
stat -c '%A %U:%G %n' /etc/passwd /etc/shadow
namei -l /ruta/al/archivo     # cadena completa de permisos, componente a componente
ls -ld /directorio
```

## Enumeración manual: el flujo en orden de ataque

El orden importa: cada paso reduce el espacio del siguiente.

### 1. Identidad y contexto

```bash
id                                   # uid, gid, grupos suplementarios
whoami; hostname; cat /etc/os-release # qué SO y versión (→ CVEs)
sudo -l                              # permisos sudo del usuario (ver nota Sudoers)
cat /proc/self/status | grep Cap     # capabilities del proceso actual
umask                                # máscara de creación por defecto
```

### 2. Superficie de permisos mal configurados

```bash
# Archivos SUID/SGID (base de explotación directa)
find / -perm -4000 -type f 2>/dev/null          # SUID
find / -perm -2000 -type f 2>/dev/null          # SGID
find / -perm -4000 -o -perm -2000 2>/dev/null   # ambos

# Capabilities en binarios (equivalente moderno de SUID)
getcap -r / 2>/dev/null

# Archivos escribibles por nosotros en rutas sensibles
find /etc /opt /usr /var /home -writable -type f 2>/dev/null
find / -writable -type d 2>/dev/null | grep -vE '^/(proc|sys|dev)'

# Directorios escribibles que no son temporales propios
find / -type d -perm -0002 ! -path '/proc/*' ! -path '/sys/*' 2>/dev/null

# Archivos sin dueño / sin grupo (huella de instalaciones rotas)
find / -nouser -o -nogroup 2>/dev/null
```

### 3. Credenciales y secretos

```bash
grep -rniE 'pass(word)? *[:=]' /home /opt /var/www /etc 2>/dev/null | head -50
history | grep -iE 'pass|ssh|mysql' 2>/dev/null; cat ~/.bash_history 2>/dev/null
ls -la ~/.ssh/ /home/*/.ssh/ 2>/dev/null
find / -name '*.bak' -o -name '*id_rsa*' -o -name '.env' 2>/dev/null
cat /etc/shadow 2>/dev/null || echo "sin lectura de shadow"   # grupo shadow? ver nota Grupos
```

### 4. Servicios, red y tareas programadas

```bash
ss -tulnp                     # puertos escuchando y proceso dueño
ps aux --sort=-%mem | head    # procesos y usuarios que los ejecutan
crontab -l; ls -la /etc/cron.* /var/spool/cron/crontabs/  # ver nota Cron
systemctl list-units --type=service --state=running
```

Ver [[Explotación de Servicios Internos]] para explotar lo que aparezca aquí y [[Nmap Guía Completa]] si hay que mapear la red.

### 5. Kernel y actualizaciones

```bash
uname -a                                  # versión exacta del kernel
cat /proc/version
apt list --upgradable 2>/dev/null | head  # o rpm -qa --last
# buscar exploits: searchsploit linux kernel <versión>
```

El catálogo de vectores de kernel va en [[Vulnerabilidades del Kernel y Escalada de Privilegios]].

## LSE (Linux Smart Enumeration)

> [!info] Qué es LSE
> La herramienta encargada de aplicar este reconocimiento en el sistema es **lse**. Linux Smart Enumeration es una herramienta de enumeración de seguridad para sistemas operativos basados en Linux, diseñada para ayudar a los administradores de sistemas y auditores de seguridad a identificar y evaluar vulnerabilidades y debilidades en la configuración del sistema.
>
> LSE está diseñada para ser fácil de usar y proporcionar una salida clara y legible. La herramienta usa comandos de Linux estándar y se ejecuta en la línea de comandos, por lo que **no requiere software adicional ni compilación** — clave en máquinas de laboratorio sin herramientas. Enumera usuarios, grupos, servicios, puertos abiertos, cron jobs, permisos de archivos, variables de entorno y configuraciones de firewall, entre otros.

### Instalación y ejecución

```bash
git clone https://github.com/diego-treitos/linux-smart-enumeration.git
cd linux-smart-enumeration
chmod +x lse.sh
./lse.sh -l 0          # nivel de verbosidad 0: solo resultados importantes (menos ruidoso)
```

En CTFs la copia directa con `curl -O https://raw.githubusercontent.com/diego-treitos/linux-smart-enumeration/master/lse.sh` (o transferencia binaria, ver [[Elevación de Privilegios mediante Explotación de Binarios]] para técnicas de subida).

### Flags

| Flag | Efecto |
|---|---|
| `-l <0-4>` | Nivel de verbosidad: `0` solo hallazgos importantes (bueno para triage), valores mayores añaden detalle y ruido. **Empezar siempre por `-l 0`** |
| `-c` | Incluir las pruebas **lentas** (barridos profundos de archivos) que LSE salta por defecto para no tardar minutos |
| `-e <test>` | Ejecutar **solo** la prueba indicada (p. ej. `-e suid`) |
| `-x <test>` | **Excluir** la prueba indicada |
| `-t <segundos>` | Timeout por prueba — evita bloqueos en scans enormes |
| `-o <archivo>` | Guardar informe en archivo para revisarlo sin scroll |
| `-h` | Ayuda con el listado completo de pruebas |

> [!tip] Flujo recomendado con LSE
> 1. `./lse.sh -l 0` → mirar solo lo marcado con `!!` (hallazgos claros).
> 2. Si hay dudas, `./lse.sh -l 1` o `-l 2` para contexto.
> 3. Con `-e` repetir solo el grupo sospechoso (p. ej. `-e fw` si sospechas del firewall, `-e suid` tras revisar binarios).
> 4. Los `**`/`!!` de LSE son **pistas, no confirmaciones**: cada hit debe verificarse manualmente con los comandos de la sección anterior.

### Grupos de pruebas de LSE (qué cubre)

| Grupo | Cubre | Qué buscar en la salida |
|---|---|---|
| `sysinfo` | SO, kernel, hostname, uptime | versión de kernel exploitable |
| `env` | variables de entorno, PATH, tokens | credenciales en env, PATH escribible |
| `user` | id, grupos, sudo -l, home | grupos de riesgo (`docker`, `sudo`, `disk`) |
| `perms` | SUID/SGID, capabilities, archivos escribibles | binarios inusuales, `/etc` escribible |
| `proc` | procesos, jobs de systemd, servicios | procesos de otros usuarios con binarios escribibles |
| `net` | interfaces, rutas, escucha, iptables | servicios localhost sin auth |
| `fs` | mounts, NFS (`no_root_squash`), docker | mounts compartidos, socket docker |
| `cron` | crontabs, timers systemd | tareas con scripts escribibles |
| `db`/`creds` | MySQL/Postgres/Redis accesibles, history | contraseñas por defecto/reusadas |
| `software` | contenedores, paquetes instalados | `docker`, `lxd`, `gcc`, `python` disponibles |

### Triage: qué hacer con la salida (metodología)

| Prioridad | Señal en LSE/manual | Acción inmediata |
|---|---|---|
| **P1** | `sudo -l` con binario abusable | GTFOBins → shell root (nota Sudoers) |
| **P1** | grupo `docker`/`lxd`/`disk`/`adm` en `id` | nota [[Grupos de Linux y Escalada de Privilegios]] |
| **P1** | SUID/SGID fuera del estándar (`find` + lista base) | buscar CVE / GTFOBins del binario |
| **P2** | archivo crítico escribible (`/etc/passwd`, script de cron, `.so` en PATH) | editar → root al ejecutarse |
| **P2** | capabilities (`getcap -r /`) | explotación directa, nota [[Linux Capabilities]] |
| **P3** | kernel vulnerable conocido | exploit de searchsploit, riesgo de crash |
| **P3** | servicio local con creds reusadas | nota [[Explotación de Servicios Internos]] |
| **P4** | solo hallazgos informativos (info leak, configs legibles) | anotar para el informe; seguir enumerando |

> [!warning] Errores de triage
> - **Volver a ejecutar LSE en nivel alto y no leer nada**: el ruido ahoga los `!!`. Siempre `-l 0` primero.
> - **Confiar en el `find /` sin `/proc` y `/sys` filtrados**: se cuelga o tarda minutos por los filesystems virtuales.
> - **Confundir "SUID estándar" con vulnerable**: `passwd`, `su`, `sudo`, `mount` son SUID por diseño; el hallazgo interesante es el SUID **raro** (o un SUID común con versión parcheable).
> - **False positive de escribibilidad**: comprobar con `test -w archivo && echo SI` — los bits de ACL/`immutable` (`lsattr`) pueden hacer que `find` diga writable pero el escritorio falle.

## Detección (lado defensor)

- **auditd**: reglas sobre directorios sensibles → `auditctl -w /etc -p wa -k etc_write`, `-w /usr/local/bin -p wa -k bin_write`.
- **AIDE/Tripwire/FIM**: baseline de hashes; cualquier SUID nuevo o cambio de permisos dispara alerta.
- **Logs**: `grep 'session opened' /var/log/auth.log` para sesiones inusuales; entradas de `find`/`getcap` masivos sobre `/` suelen indicar reconocimiento (correlacionar con `auditd` si está activo).
- **File Integrity + EDR**: creación de binarios SUID (`chmod u+s`) es una señal de alto valor.

## Impacto

Depende de la clasificación P1–P4: en el mejor de los casos, un permiso mal configurado entrega **root directo** (SUID, sudo, `disk`, archivo escribible ejecutado por root); en el mínimo, información que habilita el siguiente nivel (versiones de kernel, servicios internos, credenciales legibles).

## Sección defensiva: endurecimiento

```bash
# Permisos base
chmod 644 /etc/passwd; chmod 640 /etc/shadow; chgrp shadow /etc/shadow
chmod 755 /usr/local/bin /usr/local/sbin
find / -xdev -perm -4000 -type f -ls    # baseline de SUID y retirar lo innecesario
find / -xdev -perm -0002 -type f ! -path '/proc/*' -ls   # world-writable global

# Retirar SUID innecesario
chmod u-s /usr/bin/nc /usr/bin/tee /path/al/binario/heredado

# Bastionar directorios
chown root:root /etc /usr/bin /usr/sbin
chmod o-w /etc /opt /var/backups

# Excluir mounts temporales
mount -o remount,nosuid,nodev,noexec /tmp /var/tmp /dev/shm

# Auditoría continua
apt install -y auditd
auditctl -w /etc/passwd -p wa -k passwd_write
auditctl -w /etc/sudoers -p wa -k sudoers_write
aide --init && mv /var/lib/aide/aliases.db.new /var/lib/aide/aliases.db
```

## Errores comunes

1. **Ejecutar LSE como `sh lse.sh`** en vez de `./lse.sh` — pierde flags y comportamiento de tests.
2. **Dar permisos de ejecución a LSE pero no verificación de red**: en labs sin salida, `git clone` falla; llevar el script en la cadena de transferencia.
3. **Buscar SUID solo en `/usr/bin`** — los binarios SUID suelen estar en `/usr/local/bin`, `/opt`, `/snap`.
4. **Olvidar los grupos suplementarios** en `id`: los hallazgos de grupos (docker, disk, adm) no aparecen en `whoami`.
5. **Asumir que `o+r` en `/etc/shadow` es root**: hay que ser root o tener el grupo `shadow` (ver nota Grupos).

## Cheatsheet

```bash
id; whoami; hostname; cat /etc/os-release; uname -a; sudo -l
find / -perm -4000 -type f 2>/dev/null          # SUID
find / -perm -2000 -type f 2>/dev/null          # SGID
getcap -r / 2>/dev/null                         # capabilities
find /etc /opt /usr -writable -type f 2>/dev/null
ss -tulnp; crontab -l; ls -la /etc/cron.*
grep -rniE 'pass(word)? *[:=]' /home /opt 2>/dev/null

./lse.sh -l 0        # triage rápido
./lse.sh -c          # + pruebas lentas
./lse.sh -e suid      # solo un grupo de pruebas
```

## Referencias

- LSE — Linux Smart Enumeration: https://github.com/diego-treitos/linux-smart-enumeration
- HackTricks — Linux local enumeration: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- GTFOBins: https://gtfobins.github.io/
- PayloadsAllTheThings — Linux privilege escalation: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md
- Notas internas: [[Lectura de Permisos]], [[Privilegios SUID en Linux]], [[Linux Capabilities]], [[Abuso de Privilegios Sudoers]], [[Detección y Explotación de Tareas Cron]]

## Relacionadas

- [[Lectura de Permisos]] — referencia de `ls -l`, `stat` y notación octal
- [[Privilegios SUID en Linux]] — el hallazgo P1 más común de esta enumeración
- [[Linux Capabilities]] — `getcap` como SUID moderno
- [[Abuso de Privilegios Sudoers]] — qué hacer con cada salida de `sudo -l`
- [[Detección y Explotación de Tareas Cron]] — grupo `cron` de la enumeración
- [[Grupos de Linux y Escalada de Privilegios]] — grupos de riesgo vistos en `id`
- [[Comandos Básicos de Sistema y Red en Linux]] — base de comandos usados aquí
