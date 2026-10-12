---
aliases: ["Gestión de Contenedores Docker", "docker run", "Ciclo de vida de contenedores"]
tags:
  - devops/docker
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Docker - Fundamentos y Conceptos]]"
  - "[[Docker - Dockerfile y Construcción de Imágenes]]"
  - "[[Docker - Redes y Volúmenes]]"
  - "[[Docker - Docker Compose]]"
  - "[[Docker - Buenas Prácticas y Seguridad]]"
  - "[[Docker - Comandos de Referencia Rápida]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
fuente: "https://docs.docker.com/engine/reference/run/"
---

# Docker - Gestión de Contenedores

Una vez que existe la imagen (ver [[Docker - Dockerfile y Construcción de Imágenes]]), la vida del sistema pasa por **contenedores**: crearlos, ejecutarlos, inspeccionarlos, limitarlos y destruirlos. Esta nota es la referencia exhaustiva del CLI de ciclo de vida: cada flag de `docker run` con su semántica real, los estados y sus transiciones, monitorización y limpieza.

> [!info] Modelo mental: contenedor = proceso con metadatos
> Un contenedor es un proceso (o árbol de procesos) que el daemon arrancó con namespaces/cgroups y al que acompaña un JSON de configuración (`docker inspect`). Nada más. Por eso `stop` = enviar señal, `kill` = SIGKILL, `exec` = entrar al mismo namespace con un proceso nuevo, y `top` = `ps` con los PIDs traducidos.

## Ciclo de vida y estados

```
            docker create                docker start
  (no existe) ───────────▶ CREATED ─────────────────▶ RUNNING ◀───┐
                                  docker run = create + start     │ docker start
                                       │        │  ▲   │  ▲      │
                                       │        │  │   │  │unpause│
                                       │        ▼  │   ▼  │      │
                                       │      PAUSED   EXITED ────┘
                                       │        │        ▲   (exit code 0 / ≠0)
                        docker rm ◀────┴────────┴────────┘
                                   (solo si no está RUNNING, o con -f)
                                              ▼
                                           REMOVED
```

| Estado | Significado | Cómo se llega | Cómo sale |
|--------|-------------|---------------|-----------|
| `created` | Creado, nunca iniciado | `docker create` | `rm` |
| `running` | Proceso vivo | `start` / `run` | señal, crash, `stop`, `kill`, `pause` |
| `paused` | Procesos congelados (cgroup freezer; consume RAM, cero CPU) | `pause` | `unpause` |
| `exited(code)` | Proceso terminó; el contenedor y su fs writable **siguen ahí** | fin del PID 1, `stop`, `kill` | `start`, `rm` |
| `dead` | Transición rota (error de cgroup/runtimes raros) | fallo interno | `rm -f` |
| `removing` | En proceso de borrado | `rm` | — |

- **`docker stop`**: SIGTERM al PID 1 → espera (default 10 s, `-t N` para cambiar) → SIGKILL si aún vive. Graceful.
- **`docker kill`**: SIGKILL directo. Sin grace. `kill -s SIGINT` envía otra señal.
- **`docker restart`** = stop + start (misma semántica de timeout).
- **`pause`/`unpause`**: congela vía cgroup freezer — útil para labs (ver [[Docker - Laboratorios y Vulhub]]) o para "detener" sin perder estado en RAM.

## `docker run`: todas las opciones que importan

Sintaxis: `docker run [OPCIONES] IMAGEN [COMANDO] [ARGS…]` — los args tras la imagen **reemplazan `CMD`** (no `ENTRYPOINT`, ver [[Docker - Dockerfile y Construcción de Imágenes#CMD vs ENTRYPOINT]]).

### Interactividad: `-it`, `-i`, `-t` — la matriz

| Flags | STDIN | TTY | Cuándo usarlo | Efecto práctico |
|-------|-------|-----|---------------|-----------------|
| (ninguno) | cerrado | no | Servicios detached, comandos que no leen | `docker run alpine echo hola` |
| `-i` | abierto | no | Piped: `echo cmd \| docker run -i …` | Mantiene stdin, pero sin terminal: no hay prompt |
| `-t` | — | asigna PTY | Casi nunca solo | Colorea salida, prompts interactivos, bash-completion |
| **`-it`** | abierto | asigna | Shell interactivo: `docker run -it … bash` | La experiencia "entrar a una terminal" |
| `-d` + luego `-it` via `exec` | — | — | Apps de larga duración + entrada puntual | Patrón real: `run -d` + `exec -it c bash` |

```bash
docker run -it --rm ubuntu:25.04 bash          # shell efímero
echo "ps aux" | docker run -i --rm ubuntu ps   # stdin pipeado sin TTY
docker run -d --name web nginx:alpine          # servicio en background
docker exec -it web sh                         # ← entrar después (patrón estándar)
```

> [!warning] `-t` sin `-d` y sin comando: el contenedor "vive" mientras tu terminal esté abierta
> `docker run -t alpine` se cierra al soltar la TTY. Para servicios usá `-d`; el TTY es para humanos interactuando.

### Tabla maestra de flags de `docker run`

| Flag | Corta | Qué hace | Detalle / trampa |
|------|-------|----------|------------------|
| `--name` | | Nombre único (si no, aleatorio tipo `focused_aryabhata`) | Único por host; el nombre también es alias DNS en red propia |
| `--detach` | `-d` | Background | El exit code del `run` es el del arranque, no el de la app |
| `--rm` | | Eliminar al salir | Perfecto para comandos-una-vez; **incompatible** con `--restart` |
| `--restart` | | Política de reinicio | Tabla abajo; `no` (default), `on-failure[:N]`, `always`, `unless-stopped` |
| `--publish` | `-p` | Puerto host:contenedor | `8080:80`, `127.0.0.1:8080:80`, `80` (aleatorio), `53:53/udp`, rangos `7000-7010:7000-7010`. Ver [[Docker - Redes y Volúmenes]] |
| `--expose` | | Puerto documentado (sin publicar) | Metadatos |
| `--volume` | `-v` | `nombre:/ruta[:ro]`, `/host:/ruta[:ro]`, o anónimo `/ruta` | Ver [[Docker - Redes y Volúmenes]] |
| `--tmpfs` | | Mount efímero en RAM | `--tmpfs /tmp:rw,size=64m,mode=1777` |
| `--env` / `--env-file` | `-e` | Variables de entorno | `-e K=V`, `-e K` (hereda del shell), `-f .env` |
| `--network` | | Red (bridge default, o user-defined/host/none) | `--network none` = sin red (labs, ver [[Docker - Laboratorios y Vulhub]]) |
| `--network-alias` | | Alias extra en su red | Útil con `docker network connect` |
| `--user` | `-u` | `uid:gid`, `user`, o `uid` | Puede ser numérico sin existir en `/etc/passwd` |
| `--workdir` | `-w` | WORKDIR de runtime | Sobreescribe el del Dockerfile |
| `--entrypoint` | | Sobreescribe ENTRYPOINT | `--entrypoint ""` para volver al CMD solo |
| `--hostname` | `-h` | Hostname (UTS namespace) | Default: primeros 12 del ID |
| `--add-host` | | Entrada en `/etc/hosts` | `--add-host=host.docker.internal:host-gateway` |
| `--dns`, `--dns-search` | | Resolución | Para labs aislados, `--dns 127.0.0.1` corta resolución externa |
| `--read-only` | | Rootfs de solo lectura | + tmpfs; ver [[Docker - Buenas Prácticas y Seguridad]] |
| `--cap-add` / `--cap-drop` | | Capabilities | `--cap-drop ALL --cap-add NET_BIND_SERVICE` |
| `--security-opt` | | `no-new-privileges:true`, `seccomp=…`, `apparmor=…` | Endurecimiento |
| `--privileged` | | Devuelve TODO (caps, devices, AppArmor off) | **Solo labs de [[Docker - Laboratorios y Vulhub]]** — nunca en prod |
| `--pids-limit` | | Máx de procesos | `--pids-limit=100` contra fork bombs |
| `--cpus`, `--cpu-shares`, `--cpuset-cpus` | | CPU | Tabla de límites abajo |
| `--memory`, `-m`, `--memory-swap`, `--memory-reservation` | | RAM | Semántica abajo |
| `--ulimit` | | `nofile=64000:64000`, `nproc`, `core` | Fds abiertos = errores raros de DBs |
| `--device` | | Pasar dispositivo host | `--device=/dev/dri` para GPU sin toolkit |
| `--gpus` | | GPU (toolkit NVIDIA) | `--gpus all` |
| `--label` | | Metadatos | Clave para tooling/CI |
| `--platform` | | Plataforma de la imagen | Multi-arch |
| `--pull` | | `missing`\|`always` antes de correr | En CI: `--pull=always` para no usar imágenes rancias |

### Ejemplos reales combinados

```bash
# Servicio web endurecido
docker run -d --name web \
    -p 127.0.0.1:8080:80 \
    --restart unless-stopped \
    --read-only --tmpfs /var/cache/nginx --tmpfs /run \
    --cap-drop ALL --cap-add NET_BIND_SERVICE \
    --user 101:101 \
    --memory 256m --pids-limit 64 \
    nginx:1.27-alpine

# DB con persistencia y límites
docker run -d --name db \
    -e POSTGRES_PASSWORD_FILE=/run/secrets/pgpass \
    -v pgdata:/var/lib/postgresql/data \
    --memory 1g --memory-swap 1g --cpus 1.5 \
    --ulimit nofile=64000:64000 \
    postgres:17-alpine

# Comando-una-vez (CI): sin rastro
docker run --rm --network none myimage:1.0 python -m pytest -q
```

## Listar e inspeccionar: `ps`, `inspect`, formateo

```bash
docker ps                                   # running
docker ps -a                                # todos
docker ps -aq                               # solo IDs (para scripts)
docker ps -s                                # con tamaño de capa writable
docker ps -a --filter status=exited         # filtros: status, name, label, ancestor, since, until
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
docker ps -a --format '{{.ID}} {{.Names}} {{.State}}'   # sin tabla: pipeable
```

`docker inspect` devuelve el JSON completo (config + estado + mounts + redes). Con Go templates se extrae lo que se necesita:

```bash
docker inspect web --format '{{.State.Status}}'            # running
docker inspect web --format '{{.State.ExitCode}}'          # último exit code
docker inspect web --format '{{.State.Health.Status}}'     # healthcheck
docker inspect web --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
docker inspect web --format '{{json .Config.Env}}'         # JSON "crudo" para jq
docker inspect web --format '{{.NetworkSettings.Ports}}'    # mapeos de puerto resueltos
docker inspect web --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'  # límites aplicados
docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}' | sort
```

`jq` es el complemento natural: `docker inspect web | jq '.[0].Mounts[] | {Source, Destination}'`.

## Logs

```bash
docker logs web                       # todo (driver json-file, ver su max-size)
docker logs -f web                    # follow (tail -f)
docker logs --tail 100 -f web         # últimas 100 + seguir
docker logs -t web                    # con timestamps RFC3339
docker logs --since 1h web            # relativo: 1h, 30m, o absoluto: 2026-10-11T10:00:00
docker logs --until 30m web           # ventana finita (ideal con --since)
```

- Los logs viven en el host (`/var/lib/docker/containers/<id>/<id>-json.log`): **sin `max-size` llenan el disco**. Configuralo en daemon, por contenedor o en Compose ([[Docker - Docker Compose]]).
- Logs de procesos secundarios (que no son el PID 1) también aparecen si el runtime los captura — pero la fuente honesta de logs de apps multi-proceso es stdout del entrypoint con `exec`.
- Log drivers alternativos (`journald`, `syslog`, locales como `loki`, `fluentd`) se configuran por daemon/contenedor; en prod con log collection, es lo correcto.

## `exec` vs `attach` — dos formas de "entrar"

| | `docker exec` | `docker attach` |
|--|---------------|-----------------|
| Qué hace | Lanza un **proceso nuevo** en los namespaces del contenedor | Conecta tu TTY a los streams del proceso **actual** (PID 1) |
| ¿Señales? | Tus keystrokes van al proceso nuevo | Ctrl+C llega al PID 1 (¡puede matar el servicio!) |
| Uso | Debug, tareas, shell | Streams interactivos (consolas de app, `docker run -it` sin `-d`) |
| `--detach` | Sí (`exec -d`) | No |
| ¿Requiere shell en la imagen? | Para `-it … bash`, sí (por eso distroless no tiene) | No |

```bash
docker exec -it web sh                  # shell (usá sh: Alpine no trae bash)
docker exec -u root web apt-get update  # escalada puntual (auditable)
docker exec -e RUST_LOG=debug web app   # env solo para este proceso
docker exec -w /app web pwd             # cwd
docker exec -d web background-task.sh   # detached
docker exec web pg_isready -U postgres  # chequeo sin TTY
```

> [!warning] `docker exec` no es mantenimiento
> Lo que instales con `exec` desaparece con el contenedor y no está en la imagen: si algo debe existir, va al Dockerfile (ver [[Docker - Dockerfile y Construcción de Imágenes]]) — filosofía de inmutabilidad en [[Docker - Fundamentos y Conceptos]].

## Copiar archivos: `docker cp`

```bash
docker cp ./nginx.conf web:/etc/nginx/nginx.conf    # host → contenedor
docker cp web:/var/log/app/app.log ./logs/          # contenedor → host
docker cp web:/app - | gzip > app.tar.gz            # → stdout (tar)
docker cp -T ./config web:/app/config               # -T: preservar tubería
```

Funciona con contenedores **parados** también. Es la vía para rescatar artefactos antes de `rm`, no una estrategia de persistencia (eso son los volúmenes, ver [[Docker - Redes y Volúmenes]]).

## Tabla semántica: stop / kill / restart / pause / start / rm

| Comando | Señal | Espera | Conserva estado | Siguiente disponible | Nota |
|---------|-------|--------|-----------------|----------------------|------|
| `stop [-t N]` | SIGTERM→SIGKILL | 10 s (o N) | ✅ fs writable | `start`, `rm` | Graceful por defecto |
| `kill [-s SIG]` | SIGKILL (o `-s`) | no | ✅ | `start`, `rm` | Último recurso / señal específica |
| `restart [-t N]` | como stop | 10 s | ✅ | — | Para recargar config vieja; en prod: recreate |
| `pause` / `unpause` | — (freezer) | — | ✅ (RAM incluida) | — | Cero CPU, memoria retenida |
| `start` | — | — | ✅ | — | Reutiliza la config del `create` original |
| `rm [-f]` | SIGKILL si `-f` | — | ❌ **destroza** fs writable | — | `-f` necesario si RUNNING |
| `update` | — | — | ✅ | — | Cambia límites de cgroups en vivo |

## Límites de recursos (cgroups) en la práctica

```bash
# CPU
--cpus="1.5"            # máx 1.5 CPUs (equivale a quota 150000/100000)
--cpu-shares=512        # peso relativo ante contención (default 1024; no es límite)
--cpuset-cpus="0,2"     # anclar a CPUs físicas (latencia/NUMA)
--cpu-quota=50000 --cpu-period=100000   # 50 % de un core (método antiguo)

# Memoria (semántica exacta)
--memory="512m"           # hard limit → OOM-kill del contenedor
--memory-reservation="256m"  # soft limit (solo presión)
--memory-swap="1g"        # TOTAL memoria+swap: swap efectiva = 1g − 512m
--memory-swap="-1"        # swap ilimitada
--memory-swappiness=10    # 0 = evita swap (si tu kernel lo soporta)
--oom-kill-disable        # ⚠️ desactiva OOM-kill: el proceso puede tumbar EL HOST

# Protección
--pids-limit=200          # anti fork-bomb
--ulimit nofile=65535:65535   # fds abiertos (PostgreSQL/Redis lo necesitan)
```

> [!warning] `--memory-swap` es el total
> Error clásico: asumir que `--memory 1g --memory-swap 1g` da "1 GB de RAM + 1 GB de swap". Es mentira: la cuenta es *total* (RAM+swap = 1 g ⇒ swap efectiva = 0). Sin `--memory-swap`, el flag de swap no limita nada de RAM. Y `--memory-swap=-1` desactiva el límite.

Verificar que los límites realmente se aplicaron:

```bash
docker inspect db --format 'Mem={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} Pids={{.HostConfig.PidsLimit}}'
cat /sys/fs/cgroup/$(docker inspect db --format '{{.Name}}/memory.max') 2>/dev/null  # cgroup v2 (path real: /sys/fs/cgroup/system.slice/docker-<id>.scope)
```

## Monitorización: `stats`, `top`, `events`

```bash
docker stats                          # tabla vivo: CPU %, RAM, net I/O, block I/O, PIDs
docker stats --no-stream             # snapshot (CI/scripts)
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
docker stats --format "$(cat formato.txt)" db web   # formato compuesto

docker top web                        # procesos con PIDs del host (ps con namespace)
docker top web -eo pid,ppid,user,args # flags pasadas a ps

docker system events --since 30m      # auditoría de eventos (die, start, exec_create…)
docker events --filter event=die --filter container=web
```

- `MemUsage` "350MiB / 1GiB" indica uso/límite: si el límite es 0 no hay límite.
- `CPU%` puede superar 100 % (multicore). Sin límite, un contenedor vecino puede asfixiar al tuyo: ver [[Docker - Laboratorios y Vulhub#Recursos y límites]] para caps en labs.
- Para capacity planning real, exportá `stats` a Prometheus (cAdvisor) en vez de mirar la tabla a mano.

## Filesystem del contenedor: inspección y export

```bash
docker diff web                 # qué cambió en el fs writable (+ = creado, C = modificado, D = borrado)
docker export web > backup.tar  # snapshot del fs COMPLETO (imagen + writable) como tar
cat backup.tar | docker import - miapp:rescate   # ← convertir ese tar en imagen (¡ojo!)
docker cp web:/app ./rescate/   # extraer solo lo que importa
```

- **`export` vs `save`**: `export` = filesystem plano del contenedor (sin historial ni capas); `save` = imagen con todas sus capas y metadata. Para clonar una imagen, `save`/`load`; para rescatar datos, `cp` o `export`.
- `docker diff` es el "git status" del contenedor: perfecto para detectar snowballing (archivos modificados a mano que deberían estar en el Dockerfile, ver [[Docker - Dockerfile y Construcción de Imágenes]]).

## `docker commit` — por qué es un anti-patrón

```bash
docker commit --change 'CMD ["python","app.py"]' --message "parche manual" web miapp:v2
```

Funciona, pero produce imágenes **irreproducibles**: no sabés qué instalaste a mano, no podés auditar, no podés reconstruir, y cada "commit" duplica capas opacas. La excepción legítima: rescate puntual (un contenedor que se te cayó con estado en RAM o fs) — para eso existe, y como paso intermedio hacia un Dockerfile real:

> [!tip] Flujo de rescate correcto
> 1. `docker diff c` para ver qué cambió.
> 2. Escribir esas acciones como instrucciones en un Dockerfile.
> 3. `docker build` → imagen nueva versionada.
> El `commit` solo si hay incendio y necesitás el binario YA (luego, reconstruí).

## Limpiar y podar

```bash
docker rm web                                # parado
docker rm -f web                             # running (SIGKILL + rm)
docker rm $(docker ps -aq -f status=exited)  # todos los exited
docker container prune                       # todos los parados de un plumazo
docker container prune --filter "until=168h" # solo hace >1 semana
docker system df                             # resumen: imágenes/contenedores/volúmenes/build cache
docker system df -v                          # desglose (qué imagen usa qué capa)
```

> [!warning] `docker rm -f $(docker ps -aq)` en un host compartido
> Borra **todos** los contenedores del host, incluidos los de otros equipos. En servidores reales: siempre `--filter` o nombres explícitos. Limpieza segura y completa en [[Docker - Comandos de Referencia Rápida#Sistema y limpieza]].

## Errores comunes (gestión)

1. **`docker run -d` "funciona" pero el contenedor sale al instante**: `docker ps -a` + `docker logs <c>` — casi siempre exit code ≠ 0 en los primeros segundos (config mala, comando mal tipeado).
2. **`the input device is not a TTY`**: usaste `-it` en contexto sin TTY (CI, `$(…)`). Quitá `-t` (o ambos).
3. **`container name is already in use`**: nombres únicos; `docker rm -f nombre` si es un restos de antes.
4. **`Address already in use` al publicar puerto**: otro proceso/contenedor tiene el puerto del host; `docker ps` y `ss -tlnp` (ver [[Comandos Básicos de Sistema y Red en Linux]]).
5. **`OOMKilled` en `docker inspect`**: se superó `--memory`. Subí el límite o arreglá la fuga; mirá `State.OOMKilled` y `docker stats`.
6. **Ctrl+C no mata mi app**: PID 1 no recibe señales (forma shell de CMD) — ver [[Docker - Dockerfile y Construcción de Imágenes#CMD vs ENTRYPOINT]].
7. **`--rm` + `--restart`**: Docker lo rechaza; el efímero no tiene política de reinicio.
8. **`docker exec` falla con `executable file not found`**: el binario no existe en la imagen (especialmente distroless, sin shell).

## Cheatsheet rápido — ciclo de vida

| Quiero… | Comando |
|---------|---------|
| Correr en background con nombre | `docker run -d --name X imagen` |
| Correr y borrar al terminar | `docker run --rm imagen cmd` |
| Shell interactivo temporal | `docker run -it --rm imagen sh` |
| Shell en corriendo | `docker exec -it X sh` |
| Logs en vivo | `docker logs -f --tail 100 X` |
| Parar graceful | `docker stop X` (o `stop -t 30`) |
| Matar ya | `docker kill X` |
| Estado + health | `docker inspect X --format '{{.State.Status}} / {{.State.Health.Status}}'` |
| Qué cambió en su fs | `docker diff X` |
| Limitar recursos | `--cpus --memory --pids-limit` |
| Reinicio automático | `--restart unless-stopped` |
| Borrar | `docker rm X` / `docker rm -f X` |
| Poda general | `docker container prune` |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — qué es este proceso por dentro (namespaces, cgroups)
- [[Docker - Dockerfile y Construcción de Imágenes]] — de dónde salen las imágenes que acá se ejecutan
- [[Docker - Redes y Volúmenes]] — `-p`, `-v` y todo lo que conecta y persiste
- [[Docker - Docker Compose]] — el mismo ciclo de vida, declarativo, para stacks completos
- [[Docker - Buenas Prácticas y Seguridad]] — endurecer cada flag de esta nota
- [[Docker - Comandos de Referencia Rápida]] — versión cheatsheet de todo esto
- [[Docker - Laboratorios y Vulhub]] — ciclo de vida de labs efímeros (`up -d` → práctica → `down -v`)
- [[Comandos Básicos de Sistema y Red en Linux]] — `ps`, `ss`, señales: la base sobre la que Docker opera
