---
title: "Docker - Comandos de Referencia Rápida"
tags:
  - lang/docker
  - concept/cli
  - concept/cheatsheet
  - level/basico
  - type/reference
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Comandos de Referencia Rápida (Cheatsheet)

> **Referencia rápida** de comandos Docker y Docker Compose más usados. Para detalles ver notas específicas.

---

## 1. Imágenes

```bash
# Listar
docker images
docker images -a                    # Incluir intermedias
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"

# Buscar
docker search nginx
docker search --filter stars=100 nginx

# Descargar
docker pull nginx:alpine
docker pull nginx@sha256:abc123...

# Construir
docker build -t miapp:1.0 .
docker build -t miapp:1.0 -t miapp:latest .
docker build --no-cache -t miapp:1.0 .
docker build --build-arg VERSION=1.0 -t miapp:1.0 .
docker build -f Dockerfile.prod -t miapp:1.0 .
docker build --target builder -t miapp:builder .
docker build --progress=plain -t miapp:1.0 .
DOCKER_BUILDKIT=1 docker build -t miapp:1.0 .

# Multi-platform
docker buildx build --platform linux/amd64,linux/arm64 -t miapp:1.0 --push .

# Etiquetar
docker tag miapp:1.0 miapp:latest
docker tag miapp:1.0 registry.io/usuario/miapp:1.0

# Subir
docker push miapp:1.0
docker push registry.io/usuario/miapp:1.0

# Inspeccionar
docker inspect miapp:1.0
docker inspect --format '{{.Config.User}}' miapp:1.0
docker history miapp:1.0
docker image ls -q

# Eliminar
docker rmi miapp:1.0
docker rmi $(docker images -q)           # Todas
docker rmi $(docker images -f dangling=true -q)  # Dangling
docker image prune -a                    # No usadas

# Exportar/Importar
docker save miapp:1.0 | gzip > miapp.tar.gz
docker load < miapp.tar.gz
docker export container > container.tar
```

---

## 2. Contenedores - Ciclo de Vida

```bash
# Crear (sin iniciar)
docker create --name miapp miapp:1.0

# Ejecutar (create + start)
docker run -d --name miapp miapp:1.0

# Iniciar parado
docker start miapp

# Detener (graceful)
docker stop miapp
docker stop -t 30 miapp

# Matar (SIGKILL)
docker kill miapp
docker kill -s SIGINT miapp

# Reiniciar
docker restart miapp

# Pausar/Reanudar (cgroups freezer)
docker pause miapp
docker unpause miapp

# Eliminar
docker rm miapp
docker rm -f miapp                     # Forzar (running)
docker rm $(docker ps -aq -f status=exited)
docker container prune

# Ejecutar comando en contenedor nuevo (efímero)
docker run --rm miapp:1.0 comando
docker run --rm -it miapp:1.0 bash
```

---

## 3. Contenedores - Inspección y Debug

```bash
# Listar
docker ps                              # Running
docker ps -a                           # Todos
docker ps -aq                          # Solo IDs
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# Logs
docker logs miapp
docker logs -f miapp                   # Follow
docker logs --tail 100 miapp
docker logs -t miapp                   # Timestamps
docker logs --since "1h" miapp

# Inspeccionar
docker inspect miapp
docker inspect --format '{{.NetworkSettings.IPAddress}}' miapp
docker inspect --format '{{.State.Pid}}' miapp
docker inspect --format '{{.Mounts}}' miapp

# Stats (tiempo real)
docker stats miapp
docker stats --no-stream miapp
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Procesos
docker top miapp
docker exec miapp ps aux

# Ejecutar en contenedor corriendo
docker exec -it miapp bash
docker exec miapp ls -la /app
docker exec -u root miapp apt-get update
docker exec -w /app miapp pwd
docker exec -d miapp comando_largo

# Copiar archivos
docker cp ./local.txt miapp:/app/remote.txt
docker cp miapp:/app/remote.txt ./local.txt

# Commit (imagen desde contenedor)
docker commit miapp miapp:v2

# Exportar filesystem
docker export miapp > backup.tar
```

---

## 4. Redes

```bash
# Listar
docker network ls

# Crear
docker network create mi-red
docker network create -d bridge --subnet 172.20.0.0/16 mi-red
docker network create -d overlay --attachable mi-overlay

# Inspeccionar
docker network inspect mi-red

# Conectar/Desconectar
docker network connect mi-red contenedor
docker network disconnect mi-red contenedor

# Eliminar
docker network rm mi-red
docker network prune
```

---

## 5. Volúmenes

```bash
# Listar
docker volume ls

# Crear
docker volume create datos
docker volume create --driver local --opt type=nfs --opt o=addr=1.2.3.4,rw --opt device=:/exports datos

# Inspeccionar
docker volume inspect datos

# Usar
docker run -v datos:/data miapp
docker run -v $(pwd)/src:/app/src miapp
docker run -v /host/path:/container/path:ro miapp

# Backup
docker run --rm -v datos:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .

# Restaurar
docker run --rm -v datos:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /data

# Eliminar
docker volume rm datos
docker volume prune
```

---

## 6. Docker Compose

```bash
# Levantar
docker compose up -d
docker compose up -d --build
docker compose up -d --scale worker=3

# Build
docker compose build
docker compose build --no-cache
docker compose build --parallel

# Logs
docker compose logs -f
docker compose logs -f --tail 100 web

# Ejecutar comando
docker compose exec web bash
docker compose run --rm web python migrate.py

# Estado
docker compose ps
docker compose ps -a

# Detener/Eliminar
docker compose stop
docker compose down
docker compose down -v --rmi all --remove-orphans

# Validar config
docker compose config
docker compose config --services

# Perfiles
docker compose --profile frontend up -d
docker compose --profile all up -d

# Archivos múltiples
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 7. Sistema y Limpieza

```bash
# Info sistema
docker version
docker info
docker system df
docker system df -v

# Eventos
docker system events
docker events --since "1h" --until "now"

# Limpieza total
docker system prune                    # Contenedores parados, redes, imágenes dangling, build cache
docker system prune -a                 # + imágenes no usadas
docker system prune -a --volumes       # + volúmenes no usados
docker system prune -a --volumes --filter "until=24h"

# Build cache
docker builder prune
docker builder prune -a
```

---

## 8. Registry y Autenticación

```bash
# Login
docker login
docker login registry.io
echo $PASSWORD | docker login -u $USER --password-stdin

# Logout
docker logout
docker logout registry.io
```

---

## 9. Formato de Salida (Go Templates)

```bash
# Comunes
{{.ID}}              # ID corto
{{.Image}}           # Imagen
{{.Command}}         # Comando
{{.CreatedAt}}       # Fecha creación
{{.RunningFor}}      # Tiempo corriendo
{{.Status}}          # Estado
{{.Ports}}           # Puertos
{{.Names}}           # Nombres
{{.Labels}}          # Labels
{{.NetworkSettings.IPAddress}}  # IP
{{.Mounts}}          # Montajes
{{.State.Pid}}       # PID host
{{.Config.User}}     # Usuario
{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}
{{.RepoDigests}}     # Digests

# Ejemplos
docker ps --format "{{.Names}}: {{.Status}}"
docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}"
docker inspect --format '{{.Config.Env}}' miapp
```

---

## 10. Atajos Útiles (Alias Bash/Zsh)

```bash
# Añadir a ~/.bashrc o ~/.zshrc
alias d='docker'
alias dc='docker compose'
alias dps='docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"'
alias dpsa='docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"'
alias di='docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"'
alias dlog='docker logs -f --tail 100'
alias dexec='docker exec -it'
alias drm='docker rm -f'
alias drmi='docker rmi'
alias dprune='docker system prune -a --volumes'
alias dstats='docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"'
alias dbuild='docker build -t'
alias drun='docker run -d --name'
alias drunit='docker run -it --rm'

# Funciones
dip() { docker inspect --format '{{.NetworkSettings.IPAddress}}' "$1"; }
dlogs() { docker logs -f --tail 100 "$1"; }
dexec() { docker exec -it "$1" "${2:-bash}"; }
dstoprm() { docker stop "$1" && docker rm "$1"; }
```

---

## 11. Puertos Comunes

| Servicio | Puerto | Protocolo |
|----------|--------|-----------|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| SSH | 22 | TCP |
| PostgreSQL | 5432 | TCP |
| MySQL | 3306 | TCP |
| MongoDB | 27017 | TCP |
| Redis | 6379 | TCP |
| Elasticsearch | 9200/9300 | TCP |
| Kafka | 9092 | TCP |
| RabbitMQ | 5672/15672 | TCP |
| Prometheus | 9090 | TCP |
| Grafana | 3000 | TCP |
| DNS | 53 | UDP/TCP |
| NTP | 123 | UDP |

---

## 12. Variables de Entorno Útiles

```bash
# BuildKit
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1

# Registry
export DOCKER_REGISTRY=registry.io
export DOCKER_USER=usuario
export DOCKER_PASSWORD=******

# Compose
export COMPOSE_PROJECT_NAME=miapp
export COMPOSE_FILE=docker-compose.yml:docker-compose.prod.yml

# CLI
export DOCKER_CLI_HINTS=false
export DOCKER_SCAN_SUGGEST=false
```

---

## 13. Troubleshooting Rápido

| Problema | Diagnóstico | Solución |
|----------|-------------|----------|
| Puerto ocupado | `docker port contenedor` | Cambiar host port |
| Sin espacio | `docker system df` | `docker system prune -a --volumes` |
| Contenedor no inicia | `docker logs contenedor` | Ver error en logs |
| Permisos volúmenes | `ls -la /host/path` | `chown -R 1000:1000 /host/path` |
| DNS falla | `docker exec cat /etc/resolv.conf` | `--dns 8.8.8.8` |
| Red no conecta | `docker network inspect` | Misma red, nombres correctos |
| Imagen grande | `docker history imagen` | Multi-stage, .dockerignore, slim |
| Lento build | `docker build --progress=plain` | Cache, .dockerignore, BuildKit |

---

## 14. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Gestión de Contenedores](./Docker%20-%20Gesti%C3%B3n%20de%20Contenedores.md)
- [Redes y Volúmenes](./Docker%20-%20Redes%20y%20Vol%C3%BAmenes.md)
- [Docker Compose](./Docker%20-%20Docker%20Compose.md)
- [Buenas Prácticas y Seguridad](./Docker%20-%20Buenas%20Pr%C3%A1cticas%20y%20Seguridad.md)

---

*Etiquetas: #docker #cheatsheet #cli #comandos #referencia-rapida #docker-compose #volumenes #redes #limpieza #troubleshooting*