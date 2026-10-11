---
title: "Docker - Gestión de Contenedores"
tags:
  - lang/docker
  - concept/containers
  - concept/lifecycle
  - level/basico
  - type/reference
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Gestión de Contenedores

> **Objetivo**: Dominar el ciclo de vida completo de contenedores: crear, ejecutar, inspeccionar, detener, limpiar.

---

## 1. Ciclo de Vida del Contenedor

```
┌─────────┐     create/run     ┌──────────┐     stop      ┌──────────┐
│ CREATED │ ─────────────────▶ │ RUNNING  │ ───────────▶ │ STOPPED  │
└─────────┘                    └──────────┘               └──────────┘
       ▲                           │   │                         │
       │                           │   │ pause/unpause           │ start
       │                           ▼   ▼                         │
       │                        ┌──────────┐                    │
       └─────────────────────── │ PAUSED  │ ◀──────────────────┘
                                └──────────┘
                                      │
                                      │ rm
                                      ▼
                                ┌──────────┐
                                │ REMOVED  │
                                └──────────┘
```

---

## 2. Crear y Ejecutar (`docker run`)

### 2.1 Sintaxis Esencial

```bash
docker run [OPCIONES] IMAGEN [COMANDO] [ARGUMENTOS]
```

### 2.2 Opciones Más Usadas

| Opción | Corta | Descripción |
|--------|-------|-------------|
| `--detach` | `-d` | Background (daemon) |
| `--interactive` | `-i` | Mantener STDIN abierto |
| `--tty` | `-t` | Asignar pseudo-TTY |
| `--name` | | Nombre legible (único) |
| `--rm` | | Auto-eliminar al parar |
| `--publish` | `-p` | Port mapping `host:container` |
| `--volume` | `-v` | Montar volumen `host:container[:ro]` |
| `--env` | `-e` | Variable entorno `KEY=value` |
| `--env-file` | | Cargar .env file |
| `--network` | | Red personalizada |
| `--restart` | | Política reinicio (`no`, `on-failure`, `always`, `unless-stopped`) |
| `--cpus` | | Límite CPU (ej. `1.5`) |
| `--memory` | `-m` | Límite RAM (ej. `512m`) |
| `--user` | `-u` | Usuario `uid:gid` o `user:group` |
| `--workdir` | `-w` | Directorio trabajo |
| `--entrypoint` | | Sobrescribir ENTRYPOINT |
| `--hostname` | `-h` | Hostname contenedor |
| `--add-host` | | Entrada `/etc/hosts` |
| `--dns` | | Servidor DNS |
| `--tmpfs` | | Montar tmpfs en memoria |
| `--ulimit` | | Límites recursos (nofile, nproc) |
| `--security-opt` | | Opciones seguridad (AppArmor, seccomp) |
| `--cap-add` / `--cap-drop` | | Capabilities Linux |
| `--gpus` | | Acceso GPU (`all` o `device=0`) |

### 2.3 Ejemplos Comunes

```bash
# Básico interactivo
docker run -it --rm ubuntu:22.04 bash

# Detached (background) con nombre
docker run -d --name mi-web nginx:alpine

# Web server con port mapping
docker run -d --name web -p 8080:80 nginx:alpine

# Con variables de entorno
docker run -d --name db \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=miapp \
    postgres:15-alpine

# Con volúmenes (persistencia)
docker run -d --name db \
    -v /host/data:/var/lib/postgresql/data \
    postgres:15-alpine

# Solo lectura + tmpfs
docker run -d --name app \
    --read-only \
    --tmpfs /run --tmpfs /tmp \
    miapp:latest

# Límites recursos
docker run -d --name worker \
    --cpus="1.5" \
    --memory="512m" \
    --memory-swap="1g" \
    miapp:latest

# Usuario no-root
docker run -d --name app \
    --user 1000:1000 \
    miapp:latest

# Red personalizada
docker network create mi-red
docker run -d --name app --network mi-red miapp:latest

# Reinicio automático
docker run -d --name app --restart unless-stopped miapp:latest

# GPU (NVIDIA)
docker run -d --name ml --gpus all tensorflow/tensorflow:latest-gpu
```

---

## 3. Inspeccionar Contenedores

### 3.1 Listar

```bash
# Solo running
docker ps

# Todos (incluidos stopped)
docker ps -a

# Solo IDs (para scripting)
docker ps -aq

# Formato personalizado
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
docker ps --format "json"
```

### 3.2 Logs

```bash
# Logs completos
docker logs nombre_contenedor

# Últimas N líneas + seguir
docker logs -f --tail 100 nombre_contenedor

# Con timestamps
docker logs -t nombre_contenedor

# Desde/hasta fecha
docker logs --since "2024-01-01" --until "2024-01-02" nombre_contenedor
```

### 3.3 Inspección Detallada

```bash
# Info completa (JSON)
docker inspect nombre_contenedor

# Campos específicos
docker inspect --format '{{.State.Pid}}' nombre_contenedor
docker inspect --format '{{.NetworkSettings.IPAddress}}' nombre_contenedor
docker inspect --format '{{.Mounts}}' nombre_contenedor

# Stats en tiempo real
docker stats nombre_contenedor
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### 3.4 Procesos Internos

```bash
# Procesos dentro del contenedor
docker top nombre_contenedor

# Con ps aux
docker exec nombre_contenedor ps aux
```

---

## 4. Ejecutar Comandos en Contenedor Corriendo (`docker exec`)

```bash
# Shell interactivo
docker exec -it nombre_contenedor bash
docker exec -it nombre_contenedor sh

# Comando simple
docker exec nombre_contenedor ls -la /app
docker exec nombre_contenedor cat /etc/os-release

# Como usuario específico
docker exec -u root nombre_contenedor apt-get update
docker exec -u 1000 nombre_contenedor whoami

# Con variables de entorno
docker exec -e VAR=valor nombre_contenedor env

# Directorio trabajo
docker exec -w /app nombre_contenedor pwd

# Detached (background)
docker exec -d nombre_contenedor comando_largo
```

---

## 5. Copiar Archivos (`docker cp`)

```bash
# Host → Contenedor
docker cp ./config.yaml nombre_contenedor:/app/config.yaml

# Contenedor → Host
docker cp nombre_contenedor:/var/log/app.log ./logs/

# Directorio completo
docker cp nombre_contenedor:/app/data ./backup/
```

---

## 6. Detener, Iniciar, Reiniciar

```bash
# Detener (graceful: SIGTERM → 10s → SIGKILL)
docker stop nombre_contenedor
docker stop -t 30 nombre_contenedor  # Timeout 30s

# Forzar parada inmediata (SIGKILL)
docker kill nombre_contenedor
docker kill -s SIGINT nombre_contenedor  # Señal personalizada

# Iniciar stopped
docker start nombre_contenedor

# Reiniciar
docker restart nombre_contenedor
docker restart -t 30 nombre_contenedor

# Pausar (cgroups freezer)
docker pause nombre_contenedor
docker unpause nombre_contenedor
```

---

## 7. Eliminar y Limpiar

```bash
# Eliminar contenedor stopped
docker rm nombre_contenedor

# Forzar eliminación (running)
docker rm -f nombre_contenedor

# Eliminar múltiples
docker rm $(docker ps -aq -f status=exited)

# Eliminar TODOS (¡CUIDADO!)
docker rm -f $(docker ps -aq)

# Limpiar contenedores stopped automáticamente
docker container prune
docker container prune -f --filter "until=24h"
```

---

## 7. Commit (Crear Imagen desde Contenedor)

```bash
# Guardar cambios como nueva imagen
docker commit nombre_contenedor miapp:v2

# Con metadatos
docker commit \
    --author "Dev <dev@ejemplo.com>" \
    --message "Añadida config producción" \
    --change 'ENV DEBUG=false' \
    --change 'CMD ["python", "app.py"]' \
    nombre_contenedor miapp:v2
```

> ⚠️ **No recomendado para producción**: Usa Dockerfile para reproducibilidad.

---

## 8. Exportar/Importar

```bash
# Exportar filesystem a tar
docker export nombre_contenedor > backup.tar
docker export nombre_contenedor | gzip > backup.tar.gz

# Importar tar como imagen
cat backup.tar | docker import - miapp:backup
docker import backup.tar.gz miapp:backup
```

---

## 9. Port Mapping Avanzado

```bash
# Host:Container (TCP por defecto)
docker run -p 8080:80 nginx

# Especificar IP host (solo localhost)
docker run -p 127.0.0.1:8080:80 nginx

# Puerto aleatorio en host
docker run -p 80 nginx  # Docker asigna puerto aleatorio
docker port nombre_contenedor  # Ver mapeo

# UDP
docker run -p 53:53/udp dns-server

# Rango de puertos
docker run -p 7000-7010:7000-7010 miapp

# Solo IPv6
docker run -p [::]:8080:80 nginx
```

---

## 10. Políticas de Reinicio (`--restart`)

| Política | Comportamiento |
|----------|----------------|
| `no` | Nunca reiniciar (default) |
| `on-failure[:max-retries]` | Reiniciar si exit code ≠ 0 |
| `always` | Siempre reiniciar (incluye reboot host) |
| `unless-stopped` | Reiniciar salvo si `docker stop` manual |

```bash
# Ejemplos
docker run -d --restart always nginx
docker run -d --restart on-failure:5 miapp
docker run -d --restart unless-stopped miapp

# Cambiar política existente
docker update --restart always nombre_contenedor
```

---

## 11. Resource Constraints (cgroups)

```bash
# CPU
--cpus="1.5"           # 1.5 CPUs
--cpu-shares=1024      # Peso relativo (default 1024)
--cpu-quota=50000      # Microsegundos por período
--cpu-period=100000    # Período (100ms default)
--cpuset-cpus="0,1"    # CPUs específicas

# Memoria
--memory="512m"        # Límite RAM
--memory-swap="1g"     # Límite swap (total - memory = swap)
--memory-swappiness=60 # Agresividad swap (0-100)
--memory-reservation="256m"  # Soft limit
--oom-kill-disable     # Desactivar OOM killer

# I/O
--device-read-bps=/dev/sda:1mb
--device-write-bps=/dev/sda:1mb
--device-read-iops=/dev/sda:1000

# PIDs
--pids-limit=100       # Máx procesos/threads
```

---

## 12. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Dockerfile y Construcción](./Docker%20-%20Dockerfile%20y%20Construcci%C3%B3n%20de%20Im%C3%A1genes.md)
- [Redes y Volúmenes](./Docker%20-%20Redes%20y%20Vol%C3%BAmenes.md)
- [Docker Compose](./Docker%20-%20Docker%20Compose.md)

---

*Etiquetas: #docker #contenedores #run #exec #logs #inspect #lifecycle #resource-limits #restart-policy #cp*