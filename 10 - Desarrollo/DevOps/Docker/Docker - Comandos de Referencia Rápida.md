---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Comandos de Referencia Rápida

Referencia rápida del CLI de Docker y Compose, organizada por dominio, con recetas de formateo (Go templates + jq), one-liners reales y tabla de troubleshooting. La explicación profunda de cada comando vive en sus notas: ciclo de vida en [[Docker - Gestión de Contenedores]], redes/volúmenes en [[Docker - Redes y Volúmenes]], build en [[Docker - Dockerfile y Construcción de Imágenes]].

> [!tip] Convenciones de esta página
> `c` = contenedor, `img` = imagen, `n` = red, `v` = volumen. Los flags cortos son idénticos a los largos (`-p` = `--publish`). Todo es copiable tal cual; los `$(…)` esperan que edites los nombres.

## Sistema y diagnóstico

```bash
docker version                       # cliente + servidor (si el server falla: daemon caído)
docker info                          # storage driver, cgroup, runtime, plugins, imágenes/contenedores
docker system df                     # resumen de uso: images / containers / volumes / build cache
docker system df -v                  # desglose detallado
docker system events --since 30m     # eventos en vivo: die, exec_create, pull…
docker system events --filter event=die --filter container=web
docker context ls                    # contextos (hosts remotos); docker context use default
docker stats --no-stream             # snapshot de CPU/RAM/net I/O de todos
```

## Imágenes

```bash
docker images                        # listar (alias: docker image ls)
docker images -a                     # incluir capas intermedias
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"
docker images -f dangling=true       # huérfanas (sin tag)
docker pull nginx:1.27-alpine
docker pull nginx@sha256:abc123…     # por digest (inmutable)
docker search --filter is-official=true --filter stars=1000 nginx

docker build -t img:1.0 .                       # build básico
docker build -f Dockerfile.prod -t img:1.0 .    # Dockerfile alternativo
docker build -t img:1.0 -t img:latest .         # multi-tag
docker build --no-cache -t img:1.0 .            # ignorar caché
docker build --build-arg V=1.0 -t img:1.0 .     # args
docker build --target builder -t img:dev .      # stage específico (multi-stage)
docker build --progress=plain -t img:1.0 .      # log detallado (debug de caché)
DOCKER_BUILDKIT=1 docker build -t img:1.0 .     # forzar BuildKit (default en Docker 23+)
docker buildx build --platform linux/amd64,linux/arm64 -t img:1.0 --push .   # multi-arch
docker build --secret id=npmrc,src=.npmrc .     # secretos de build (BuildKit)

docker tag img:1.0 registry.internal/owner/img:1.0
docker push img:1.0
docker push registry.internal/owner/img:1.0

docker history img:1.0                # capas/instrucciones y pesos
docker history --no-trunc img:1.0     # completo (¿secretos colados?)
docker image inspect img:1.0 --format '{{.Config.User}} | {{.Os}}/{{.Architecture}}'
docker image inspect img:1.0 --format '{{json .Config.Env}}' | jq

docker rmi img:1.0                    # eliminar imagen
docker rmi $(docker images -q)        # todas (⚠️ destructivo)
docker image prune                    # dangling
docker image prune -a                 # + no referenciadas por contenedores
docker image save img:1.0 | gzip > img.tar.gz    # exportar imagen CON capas
docker load < img.tar.gz
docker image push --all-tags registry.internal/owner/img   # todos los tags
```

## Contenedores — ciclo de vida

```bash
docker run -d --name c img                              # servicio detached
docker run --rm img cmd                                 # efímero (una vez)
docker run -it --rm img sh                              # shell temporal
docker create --name c img                              # sin iniciar
docker start c / stop c / restart c [-t 30]
docker stop -t 30 c                                     # graceful con timeout
docker kill c / docker kill -s SIGINT c
docker pause c / docker unpause c
docker update --restart unless-stopped --memory 1g c    # mutar límites/política en vivo
docker rm c / docker rm -f c
docker rm $(docker ps -aq -f status=exited)             # barrido por estado
docker container prune --filter "until=168h"            # parados, con antigüedad
```

### `docker run` — flags más usados

```bash
docker run -d --name c \
  -p 127.0.0.1:8080:80 \              # publicar (host:contenedor)
  --env K=V --env-file .env \          # variables
  -v vol:/data:ro \                    # volumen read-only
  --network backend \                  # red user-defined
  --restart unless-stopped \           # política
  --read-only --tmpfs /tmp \           # fs duro
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --user 10001:10001 \
  --memory 512m --cpus 1.5 --pids-limit 200 \
  --log-opt max-size=10m --log-opt max-file=3 \
  img:1.0
```

## Contenedores — inspección y debug

```bash
docker ps                                    # running
docker ps -a                                 # todos
docker ps -aq -f status=exited -f name=web    # filtros acumulativos
docker ps -s                                 # con tamaño writable
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

docker logs -f --tail 100 --since 1h c       # seguir últimas 100 de la hora
docker logs -t --until 10m c                 # con timestamps, ventana finita

docker exec -it c sh                         # shell en el corriendo
docker exec -u root c id                     # como otro usuario
docker exec -w /app c pwd
docker exec -e DEBUG=1 c app
docker exec -d c background-job.sh
docker attach c                              # streams del PID 1 (Ctrl+C lo mata)

docker top c                                 # procesos (ps traducido)
docker diff c                                # qué cambió en su fs writable
docker cp ./file c:/app/file                 # host → contenedor
docker cp c:/var/log/app.log ./              # contenedor → host
docker export c > c.tar                      # fs plano (sin capas)
docker commit --change 'CMD ["x"]' c img:v2  # anti-patrón (salvo rescate)

docker inspect c --format '{{.State.Status}} / exit {{.State.ExitCode}}'
docker inspect c --format '{{.State.Health.Status}}'
docker inspect c --format '{{range .Mounts}}{{.Source}} => {{.Destination}}{{"\n"}}{{end}}'
docker inspect c --format '{{json .NetworkSettings.Ports}}' | jq
docker port c                                # mapeos resueltos
```

## Redes

```bash
docker network ls
docker network ls -f driver=bridge
docker network create n
docker network create -d bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1 n
docker network create --internal aislada              # sin salida a internet
docker network create -d overlay --attachable o       # Swarm
docker network inspect n
docker network inspect n --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
docker network connect n c                            # conectar existente
docker network disconnect n c
docker network rm n
docker network prune
```

## Volúmenes

```bash
docker volume ls
docker volume ls -f dangling=true
docker volume create v
docker volume create --driver local --opt type=none --opt o=bind --opt device=/mnt/data v
docker volume inspect v
docker volume rm v
docker volume prune

# Backup / restaurar (receta canónica)
docker run --rm -v v:/data -v $(pwd):/backup alpine tar czf /backup/v.tar.gz -C /data .
docker run --rm -v v:/data -v $(pwd):/backup alpine tar xzf /backup/v.tar.gz -C /data
```

## Sistema: limpieza y espacio

```bash
docker system df
docker system prune                       # contenedores parados + redes + imágenes dangling + build cache
docker system prune -a                    # + imágenes sin contenedor que las use
docker system prune -a --volumes          # + volúmenes no usados (⚠️ datos)
docker system prune -a --volumes --filter "until=24h"
docker builder prune                      # solo caché de build
docker builder prune -a --keep-storage 5GB
docker container prune / image prune / network prune / volume prune   # granular
```

> [!warning] Orden de destructividad
> `prune` < `prune -a` < `prune -a --volumes`. El último borra volúmenes con datos de DBs reales. En hosts compartidos: siempre `--filter` o nombres explícitos.

## Docker Compose (resumen)

```bash
docker compose up -d [--build] [--wait] [--force-recreate] [--scale svc=3]
docker compose down [-v] [--rmi all] [--remove-orphans]
docker compose ps [-a] / logs -f --tail 100 svc / top / events
docker compose exec svc sh
docker compose run --rm [--no-deps] svc cmd
docker compose build [--no-cache] [svc]
docker compose pull / push / restart / pause / unpause
docker compose config [--services|--volumes|--profiles]     # YAML resuelto (debug)
docker compose up --dry-run
docker compose --profile X up -d
docker compose -f compose.yaml -f compose.prod.yml up -d
docker compose watch
```

Detalle completo: [[Docker - Docker Compose]].

## Formateo: Go templates y jq

```bash
# Campos comunes en ps/images
{{.ID}} {{.Names}} {{.Image}} {{.Command}} {{.CreatedAt}} {{.RunningFor}} {{.Status}} {{.Ports}} {{.State}} {{.Labels}}

# Campos de inspect (JSON del daemon)
{{.State.Status}} {{.State.ExitCode}} {{.State.Pid}} {{.Config.User}} {{.Config.Env}}
{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}} {{.HostConfig.Privileged}} {{.HostConfig.ReadonlyRootfs}}
{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}} {{.HostConfig.NetworkMode}}
{{.Mounts}} {{.NetworkSettings.Ports}} {{.RestartCount}} {{json .}}      # json = crudo

# Trucos de template
{{range .Config.Env}}{{.}}{{"\n"}}{{end}}          # iterar lista
{{index .Args 0}}                                   # primer elemento
docker ps -a --format '{{.Names}}\t{{.Status}}' | sort

# jq para el JSON completo (más potente que templates)
docker inspect c | jq '.[0].Mounts[].Source'
docker inspect c | jq -r '.[0].State.Health.Status'
docker stats --no-stream --format '{{json .}}' | jq -r '[.Name,.CPUPerc,.MemUsage] | @tsv'
docker system df -v --format '{{json .}}' | jq
```

## One-liners útiles

```bash
# IDs de contenedores parados → borrar
docker ps -aq -f status=exited | xargs -r docker rm

# IP de un contenedor
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c

# ¿En qué red está?
docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' c

# Último exit code de todos los caídos
docker ps -a --format '{{.Names}} {{.ExitCode}}' -f status=exited

# Logs filtrados con grep (timestamps incluidos)
docker logs -t c 2>&1 | grep -i error | tail -50

# Tamaño real de una imagen (no el de la tabla)
docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}' | sort -k2 -h | tail

# Shell como root temporal (auditable)
docker exec -u 0 -it c sh -c 'apt-get update && apt-get install -y procps && ps aux'

# Reiniciar solo lo que cambió en compose
docker compose up -d --force-recreate --no-deps api

# Exportar imágenes de un host a otro (air-gap)
docker save img1 img2 | gzip > bundle.tar.gz && docker load < bundle.tar.gz

# Ver a qué contenedor pertenece un volumen
docker volume inspect v --format '{{range .Mounts}}{{.Name}}{{end}}' ; docker ps -a --filter volume=v

# Traza rápida de red: desde el contenedor al host
docker exec c sh -c 'getent hosts db; nc -zv db 5432'   # ver [[Docker - Redes y Volúmenes]]
```

## Aliases y variables de entorno

```bash
# ~/.bashrc o ~/.zshrc
alias dps='docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"'
alias dpsa='docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"'
alias dc='docker compose'
alias dex='docker exec -it'
alias dlog='docker logs -f --tail 100'
alias dstats='docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"'
alias dprune='docker system prune -a --volumes'   # ⚠️ revisar antes de ejecutar

dip()  { docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$1"; }
dexs() { docker exec -it "$1" sh -c 'command -v bash >/dev/null && exec bash || exec sh'; }
```

```bash
export DOCKER_BUILDKIT=1               # BuildKit (default moderno)
export COMPOSE_PROJECT_NAME=miapp      # nombre de proyecto fijo
export COMPOSE_FILE=compose.yaml:compose.prod.yml
export DOCKER_CLI_HINTS=false          # silencia tips del CLI
```

## Puertos frecuentes

| Servicio | Puerto | | Servicio | Puerto |
|----------|--------|-|----------|--------|
| HTTP | 80 | | PostgreSQL | 5432 |
| HTTPS | 443 | | MySQL | 3306 |
| SSH | 22 | | MongoDB | 27017 |
| DNS | 53 udp/tcp | | Redis | 6379 |
| Node/Express dev | 3000/5173 | | Elasticsearch | 9200/9300 |
| Flask dev | 5000 | | Kafka | 9092 |
| Gunicorn/uWSGI | 8000 | | RabbitMQ | 5672/15672 |
| Prometheus | 9090 | | Grafana | 3000 |

## Troubleshooting rápido

| Síntoma | Diagnóstico | Solución |
|---------|-------------|----------|
| `Cannot connect to Docker daemon` | `systemctl status docker`, `docker context ls` | Arrancar daemon / `context use default` |
| `permission denied … docker.sock` | `id \| grep docker` | `usermod -aG docker` + relogin |
| Contenedor sale al instante | `docker ps -a` + `docker logs c` | Ver exit code; typo en CMD |
| `port is already allocated` | `docker ps`, `ss -tlnp \| grep :8080` | Cambiar lado host del `-p` |
| `no such host` entre contenedores | `docker network inspect n` | Misma red user-defined; nombre = servicio |
| DNS no resuelve externo | `docker exec c cat /etc/resolv.conf` | `--dns 1.1.1.0` o revisar red `--internal` |
| `permission denied` en bind mount | `ls -la /ruta/host` | Alinear UID (`--user`, `chown`) |
| `OOMKilled` | `docker inspect c \| jq '.[0].State'` | Subir `--memory` o arreglar fuga |
| CPU/RAM disparados | `docker stats`, `docker top c` | `--cpus/--memory` (ver [[Docker - Gestión de Contenedores]]) |
| Disco lleno | `docker system df -v` | `prune` en orden de destructividad |
| Build lento / rebuild total | `docker build --progress=plain` | .dockerignore, orden de capas, cache mounts |
| Imagen enorme | `docker history img` | multi-stage + slim/distroless |
| `exec format error` | `docker image inspect` (arquitectura) | `--platform` correcto o build nativo |
| Ctrl+C no mata la app | PID 1 = shell | CMD en forma exec (ver [[Docker - Dockerfile y Construcción de Imágenes]]) |
| Compose "ignora" cambios | `docker compose config` | `--force-recreate`; revisar merge/override |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — qué hace cada comando por dentro
- [[Docker - Gestión de Contenedores]] — referencia profunda de run/exec/logs/inspect/límites
- [[Docker - Dockerfile y Construcción de Imágenes]] — build, caché, BuildKit
- [[Docker - Redes y Volúmenes]] — DNS embebido, port publishing, tipos de montaje
- [[Docker - Docker Compose]] — orquestación declarativa completa
- [[Docker - Buenas Prácticas y Seguridad]] — auditoría con inspect/history y endurecimiento
- [[Docker - Laboratorios y Vulhub]] — flujos efímeros con estas herramientas
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `ps`, `iptables` para el host
- [[Comandos Git]] — el mismo espíritu de cheatsheet para el VCS
