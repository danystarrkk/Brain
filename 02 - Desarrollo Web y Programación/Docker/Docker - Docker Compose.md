---
title: "Docker - Docker Compose"
tags:
  - lang/docker
  - concept/compose
  - concept/orchestration
  - concept/yaml
  - level/intermedio
  - type/concept
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Docker Compose

> **Objetivo**: Orquestar aplicaciones multi-contenedor con un solo archivo YAML declarativo.

---

## 1. Qué es Docker Compose

**Docker Compose** es una herramienta para definir y ejecutar aplicaciones multi-contenedor. Usa un archivo `docker-compose.yml` (o `.yaml`) para configurar servicios, redes y volúmenes, y un solo comando (`docker compose up`) para levantar todo.

### Versiones

| Versión | Comando | Estado |
|---------|---------|--------|
| **v1 (standalone)** | `docker-compose` | **Deprecated** (Python) |
| **v2 (plugin)** | `docker compose` | **Actual** (Go, integrado en Docker CLI) |

> **Usa `docker compose` (v2)**. Si tienes `docker-compose` v1, migra: `alias docker-compose='docker compose'`

---

## 2. Estructura del Archivo `docker-compose.yml`

```yaml
version: "3.8"  # Opcional en v2+, pero recomendado para compatibilidad

services:          # Definición de contenedores (REQUERIDO)
  servicio1:
    image: nginx:alpine
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./data:/data
    environment:
      - VAR=valor
    networks:
      - mi-red
    depends_on:
      - db

  servicio2:
    ...

networks:          # Redes personalizadas (opcional)
  mi-red:
    driver: bridge

volumes:           # Volúmenes nombrados (opcional)
  datos-db:
    driver: local

configs:           # Configs (Swarm)
secrets:           # Secrets (Swarm)
```

---

## 3. Configuración de Servicios (`services`)

### 3.1 Claves Principales

```yaml
services:
  web:
    # Imagen
    image: nginx:alpine          # Usar imagen existente
    # O build desde Dockerfile
    build:                       # Build context
      context: .                 # Directorio con Dockerfile
      dockerfile: Dockerfile.prod
      args:                      # Build args
        VERSION: "1.0"
      target: production         # Multi-stage target
      cache_from:                # Cache externo
        - miapp:cache

    # Contenedor
    container_name: mi-web       # Nombre fijo (opcional)
    hostname: web-host           # Hostname interno
    user: "1000:1000"            # Usuario no-root
    working_dir: /app            # WORKDIR
    entrypoint: ["/entrypoint.sh"] # Override ENTRYPOINT
    command: ["nginx", "-g", "daemon off;"] # Override CMD

    # Red
    ports:
      - "8080:80"                # host:container
      - "443:443"
      - "127.0.0.1:8080:80"      # Bind a IP específica
    expose:
      - "80"                     # Solo interno (documentación)
    networks:
      - frontend
      - backend
    dns:
      - 1.1.1.1
      - 8.8.8.8
    dns_search:
      - ejemplo.com
    extra_hosts:
      - "host.docker.internal:host-gateway"  # Acceso a host

    # Volúmenes
    volumes:
      - ./src:/app/src:rw        # Bind mount
      - datos:/var/lib/data:rw   # Named volume
      - /host/path:/cont/path:ro # Read-only
      - type: bind               # Sintaxis larga
        source: ./config
        target: /app/config
        read_only: true

    # Variables de entorno
    environment:
      - DEBUG=1                  # KEY=VALUE
      - APP_ENV=production
    env_file:
      - .env                     # Archivo .env
      - .env.prod
    env_file: .env               # O string único

    # Recursos
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M

    # Reinicio
    restart: unless-stopped      # no | always | on-failure | unless-stopped

    # Dependencias (orden de inicio)
    depends_on:
      db:
        condition: service_healthy  # service_started | service_healthy | service_completed_successfully
      redis:
        condition: service_started

    # Healthcheck
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

    # Logging
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    # Seguridad
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /run
      - /tmp
    ulimits:
      nofile:
        soft: 64000
        hard: 64000

    # Configs y Secrets (solo Swarm)
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    secrets:
      - db_password
```

---

## 4. Variables de Entorno y `.env`

### 4.1 Archivo `.env` (Autocargado)

```bash
# .env (en mismo directorio que compose)
COMPOSE_PROJECT_NAME=miapp
VERSION=1.0.0
DB_PASSWORD=secret
```

```yaml
# docker-compose.yml
services:
  app:
    image: miapp:${VERSION}
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
```

### 4.2 Múltiples Entornos

```bash
# Archivos de override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

```yaml
# docker-compose.override.yml (dev local)
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - DEBUG=1
    ports:
      - "8000:8000"
```

---

## 5. Comandos Principales

```bash
# Levantar (build + up)
docker compose up -d

# Build explícito
docker compose build
docker compose build --no-cache
docker compose build --parallel

# Ver logs
docker compose logs -f
docker compose logs -f --tail 100 web

# Ejecutar comando en servicio
docker compose exec web bash
docker compose run --rm web python manage.py migrate

# Escalar (solo servicios replicated)
docker compose up -d --scale worker=3

# Ver estado
docker compose ps
docker compose ps -a

# Detener (mantiene contenedores)
docker compose stop

# Parar y eliminar (contenedores, redes, volúmenes anónimos)
docker compose down

# Eliminar TODO (incluye volúmenes nombrados, imágenes)
docker compose down -v --rmi all --remove-orphans

# Validar configuración
docker compose config
docker compose config --services

# Ver eventos
docker compose events

# Pausar/reanudar
docker compose pause
docker compose unpause

# Reiniciar
docker compose restart
docker compose restart web
```

---

## 6. `depends_on` y Healthchecks

### 6.1 Solo Orden de Inicio (No Espera Ready)

```yaml
services:
  web:
    depends_on:
      - db
      - redis
```

### 6.2 Espera Real con Healthcheck

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    depends_on:
      db:
        condition: service_healthy
```

### 6.3 Script de Espera (Alternativa)

```yaml
services:
  web:
    entrypoint: ["/wait-for-it.sh", "db:5432", "--"]
    command: ["python", "app.py"]
```

---

## 7. Perfiles (Profiles)

```yaml
services:
  frontend:
    profiles: ["frontend", "all"]
    ...

  backend:
    profiles: ["backend", "all"]
    ...

  db:
    profiles: ["backend", "all"]
    ...

  monitoring:
    profiles: ["monitoring"]
    ...
```

```bash
# Levantar solo frontend
docker compose --profile frontend up -d

# Múltiples perfiles
docker compose --profile frontend --profile backend up -d

# Todos
docker compose --profile all up -d
```

---

## 8. Extends y Anclas YAML (Reutilización)

```yaml
# Base común
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  networks:
    - backend

services:
  api:
    <<: *common
    image: miapi:1.0
    depends_on:
      - db

  worker:
    <<: *common
    image: miapi:1.0
    command: ["celery", "worker"]
    depends_on:
      - redis
```

---

## 8. Docker Compose en Producción

### 8.1 Archivo `docker-compose.prod.yml`

```yaml
version: "3.8"

services:
  web:
    image: miapp:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1"
          memory: 1G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    deploy:
      placement:
        constraints:
          - node.role == manager

volumes:
  pgdata:

networks:
  default:
    driver: overlay
    attachable: true
```

### 8.2 Deploy en Swarm

```bash
# Inicializar swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.prod.yml miapp

# Ver servicios
docker stack services miapp

# Escalar
docker service scale miapp_web=5

# Eliminar stack
docker stack rm miapp
```

---

## 9. Debugging Compose

```bash
# Ver configuración resuelta (merge de archivos)
docker compose config

# Ver solo servicios
docker compose config --services

# Ver volúmenes
docker compose config --volumes

# Dry run (sin crear)
docker compose up --dry-run

# Entrar a contenedor fallido
docker compose run --rm --entrypoint sh web

# Ver eventos en tiempo real
docker compose events --json
```

---

## 10. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Gestión de Contenedores](./Docker%20-%20Gesti%C3%B3n%20de%20Contenedores.md)
- [Redes y Volúmenes](./Docker%20-%20Redes%20y%20Vol%C3%BAmenes.md)
- [Buenas Prácticas y Seguridad](./Docker%20-%20Buenas%20Pr%C3%A1cticas%20y%20Seguridad.md)

---

*Etiquetas: #docker #compose #orchestration #yaml #multi-container #services #networks #volumes #healthcheck #profiles #swarm*