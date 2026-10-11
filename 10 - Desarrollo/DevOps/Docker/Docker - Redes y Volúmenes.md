---
title: "Docker - Redes y Volúmenes"
tags:
  - lang/docker
  - concept/networking
  - concept/volumes
  - concept/storage
  - level/intermedio
  - type/concept
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Redes y Volúmenes

> **Objetivo**: Conectar contenedores entre sí y con el exterior, y persistir datos más allá del ciclo de vida del contenedor.

---

## 1. Redes en Docker

### 1.1 Drivers de Red

| Driver | Uso | Aislamiento |
|--------|-----|-------------|
| `bridge` | Default, contenedores en mismo host | Red privada interna |
| `host` | Compartir red del host (sin NAT) | Sin aislamiento red |
| `overlay` | Multi-host (Swarm/K8s) | Red distribuida |
| `macvlan` | MAC propia por contenedor | Aparece como dispositivo físico |
| `none` | Sin red | Aislamiento total |
| `ipvlan` | Similar a macvlan, más ligero | L2/L3 |

### 1.2 Red Bridge (Default)

```bash
# Ver redes
docker network ls

# Inspeccionar bridge default
docker network inspect bridge

# Crear red bridge personalizada (recomendado)
docker network create --driver bridge mi-red
docker network create -d bridge --subnet 172.20.0.0/16 --gateway 172.20.0.1 mi-red
```

**Ventajas de red personalizada vs default**:
- **DNS automático**: Contenedores se resuelven por **nombre de servicio**
- **Aislamiento**: Solo contenedores en misma red se ven
- **Control**: Subnet, gateway, MTU configurables

### 1.3 Conectar Contenedores

```bash
# Crear red
docker network create backend

# Conectar al crear
docker run -d --name db --network backend postgres:15-alpine
docker run -d --name api --network backend miapi:latest

# Conectar contenedor existente
docker network connect backend contenedor_existente

# Desconectar
docker network disconnect backend contenedor

# Ver conectividad
docker exec api ping db  # Resuelve por nombre
docker exec api nslookup db
```

### 1.4 Red Host (Host Networking)

```bash
# Sin aislamiento red - usa IP del host directamente
docker run -d --network host --name miapp miapp:latest

# Puertos: localhost:8080 en host = localhost:8080 en contenedor
# Útil para: máximo performance, binding a interfaces específicas
# ⚠️ Menos seguro, no portable (Linux only)
```

### 1.5 Red Overlay (Swarm)

```bash
# Requiere Swarm mode
docker swarm init
docker network create -d overlay --attachable mi-overlay

# Contenedores en diferentes hosts se ven por nombre
docker service create --name web --network mi-overlay nginx
docker service create --name api --network mi-overlay miapi
```

### 1.6 DNS Interno

```bash
# En red bridge personalizada:
# - Nombre contenedor → IP
# - service_name → IP (si docker-compose)
# - container_name → IP

# Buscar servicio
docker exec api getent hosts db
# 172.20.0.2    db

# FQDN: nombre_contenedor.red.docker.internal
```

---

## 2. Volúmenes y Persistencia

### 2.1 Tipos de Montaje

| Tipo | Sintaxis | Persistencia | Caso de Uso |
|------|----------|--------------|-------------|
| **Named Volume** | `-v nombre:/path` | ✅ Gestionado por Docker | DBs, datos app, logs |
| **Bind Mount** | `-v /host/path:/path` | ✅ Directorio host | Config, código dev, certs |
| **Tmpfs** | `--tmpfs /path` | ❌ Solo memoria | Secrets, cache, temp |
| **Anonymous Volume** | `-v /path` | ⚠️ Difícil localizar | Temporal, no nombrado |

### 2.2 Named Volumes (Recomendado para Datos)

```bash
# Crear volumen
docker volume create datos-db
docker volume create --driver local --opt type=none --opt device=/mnt/data --opt o=bind datos-db

# Usar en contenedor
docker run -d --name db -v datos-db:/var/lib/postgresql/data postgres:15

# Listar e inspeccionar
docker volume ls
docker volume inspect datos-db

# Backup
docker run --rm -v datos-db:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .

# Restaurar
docker run --rm -v datos-db:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /data

# Eliminar
docker volume rm datos-db
docker volume prune  # Eliminar no usados
```

### 2.3 Bind Mounts (Desarrollo/Config)

```bash
# Directorio host → contenedor
docker run -d -v $(pwd)/src:/app/src miapp:dev

# Archivo config
docker run -d -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro nginx

# Múltiples
docker run -d \
    -v $(pwd)/src:/app/src \
    -v $(pwd)/config:/app/config:ro \
    -v $(pwd)/logs:/app/logs \
    miapp:dev

# Con permisos (UID/GID)
docker run -d -v $(pwd)/data:/data:rw,uid=1000,gid=1000 miapp
```

### 2.4 Tmpfs (Solo Memoria)

```bash
# Directorio en RAM (rápido, seguro, efímero)
docker run -d --tmpfs /run --tmpfs /tmp:rw,size=100m,mode=1777 miapp

# Para secrets temporales
docker run -d --tmpfs /secrets:ro,size=10m miapp
```

### 2.5 Drivers de Volumen Avanzados

```bash
# NFS
docker volume create --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.100,rw \
    --opt device=:/exports/data \
    nfs-volume

# CIFS/SMB
docker volume create --driver local \
    --opt type=cifs \
    --opt o=username=user,password=pass,vers=3.0 \
    --opt device=//server/share \
    cifs-volume
```

---

## 3. Patrones Comunes

### 3.1 Base de Datos con Persistencia

```bash
# PostgreSQL
docker run -d --name postgres \
    --network backend \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=miapp \
    -v pgdata:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres:15-alpine

# MySQL
docker run -d --name mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=miapp \
    -v mysqldata:/var/lib/mysql \
    mysql:8

# Redis
docker run -d --name redis \
    -v redisdata:/data \
    redis:7-alpine redis-server --appendonly yes
```

### 3.2 Compartir Datos entre Contenedores

```bash
# Volumen compartido
docker volume create shared-logs

docker run -d --name app1 -v shared-logs:/var/log/app miapp
docker run -d --name app2 -v shared-logs:/var/log/app miapp
docker run -d --name log-processor -v shared-logs:/logs:ro log-processor
```

### 3.3 Código en Desarrollo (Hot Reload)

```bash
# Bind mount + volumen para node_modules (evita conflictos OS)
docker run -d --name frontend \
    -v $(pwd):/app \
    -v /app/node_modules \
    -p 3000:3000 \
    node:20-alpine sh -c "npm install && npm run dev"
```

---

## 4. Networking Avanzado

### 4.1 Exponer Puertos Sin Publicar (Internal)

```bash
# Solo accesible desde red Docker
docker run -d --name internal --expose 8080 miapp

# Desde otro contenedor en misma red
docker exec otro curl internal:8080
```

### 4.2 Macvlan (IP Propia en LAN)

```bash
# Crear red macvlan
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 \
    macvlan-net

# Contenedor con IP en LAN física
docker run -d --name server --network macvlan-net --ip 192.168.1.100 miapp
```

### 4.3 Proxy Reverso (Nginx/Traefik)

```bash
# Nginx proxy simple
docker run -d --name proxy \
    --network frontend \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    -p 80:80 -p 443:443 \
    nginx:alpine
```

---

## 5. Limpieza y Mantenimiento

```bash
# Volúmenes huérfanos (no usados por ningún contenedor)
docker volume ls -f dangling=true
docker volume prune

# Redes no usadas
docker network prune

# Todo en uno (contenedores parados, redes, volúmenes, imágenes dangling)
docker system prune -a --volumes

# Ver uso de disco
docker system df
docker system df -v  # Detallado
```

---

## 6. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Gestión de Contenedores](./Docker%20-%20Gesti%C3%B3n%20de%20Contenedores.md)
- [Docker Compose](./Docker%20-%20Docker%20Compose.md)
- [Buenas Prácticas y Seguridad](./Docker%20-%20Buenas%20Pr%C3%A1cticas%20y%20Seguridad.md)

---

*Etiquetas: #docker #networking #volumes #bind-mount #named-volume #tmpfs #bridge #overlay #macvlan #persistencia #dns*