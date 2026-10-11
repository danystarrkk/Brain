---
title: "Docker - Dockerfile y Construcción de Imágenes"
tags:
  - lang/docker
  - concept/dockerfile
  - concept/build
  - concept/layers
  - level/intermedio
  - type/concept
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Dockerfile y Construcción de Imágenes

> **Objetivo**: Dominar la escritura de Dockerfiles eficientes, multi-stage builds, y optimización de capas.

---

## 1. Instrucciones Principales

| Instrucción | Función | Crea Capa |
|-------------|---------|-----------|
| `FROM` | Imagen base | Sí (base) |
| `RUN` | Ejecutar comando en build | Sí |
| `COPY` | Copiar archivos host→imagen | Sí |
| `ADD` | Como COPY + URLs + auto-extract tar | Sí |
| `CMD` | Comando por defecto (sobrescribible) | No |
| `ENTRYPOINT` | Ejecutable principal (no sobrescribible fácil) | No |
| `ENV` | Variables de entorno | No |
| `ARG` | Argumentos de build (no en imagen final) | No |
| `EXPOSE` | Documentar puertos | No |
| `WORKDIR` | Directorio de trabajo | No |
| `USER` | Usuario para comandos siguientes | No |
| `VOLUME` | Declarar punto de montaje | No |
| `LABEL` | Metadatos | No |
| `HEALTHCHECK` | Comando de salud | No |
| `ONBUILD` | Trigger para imágenes hijas | No |
| `STOPSIGNAL` | Señal para parar | No |
| `SHELL` | Shell por defecto | No |

---

## 2. Sintaxis y Estructura Básica

```dockerfile
# Comentario
INSTRUCCIÓN argumentos

# Ejemplo completo
FROM python:3.11-slim AS base

# Metadatos
LABEL maintainer="dev@ejemplo.com" \
      version="1.0.0" \
      description="API REST ejemplo"

# Variables de build
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

# Variables de entorno (persisten en imagen)
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    APP_VERSION=${APP_VERSION} \
    BUILD_DATE=${BUILD_DATE}

# Directorio trabajo
WORKDIR /app

# Usuario no-root (seguridad)
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copiar solo requirements primero (caché)
COPY --chown=appuser:appuser requirements.txt .

# Instalar dependencias (caché separado)
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

# Copiar código (cambia frecuentemente → al final)
COPY --chown=appuser:appuser . .

# Cambiar a usuario no-root
USER appuser

# Puerto documentado
EXPOSE 8000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Comando por defecto
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

---

## 3. FROM - Imagen Base

### 3.1 Buenas Prácticas

```dockerfile
# ❌ MALO: latest cambia sin avisar
FROM ubuntu:latest

# ✅ BUENO: Tag específico (inmutable)
FROM ubuntu:22.04

# ✅ MEJOR: Digest SHA256 (absolutamente inmutable)
FROM ubuntu@sha256:abc123...

# ✅ Multi-arch: Docker elige según arquitectura host
FROM python:3.11-slim
```

### 3.2 Imágenes Base Recomendadas

| Uso | Imagen | Tamaño | Notas |
|-----|--------|--------|-------|
| **Python** | `python:3.11-slim` | ~40MB | Debian slim, glibc |
| **Python** | `python:3.11-alpine` | ~15MB | musl libc, puede romper wheels |
| **Node** | `node:20-alpine` | ~20MB | Muy ligero |
| **Go** | `golang:1.21-alpine` | ~15MB | Para build stage |
| **Java** | `eclipse-temurin:21-jre-alpine` | ~50MB | Solo JRE |
| **Base mínima** | `gcr.io/distroless/base` | ~2MB | Sin shell, solo runtime |
| **Scratch** | `scratch` | 0 | Vacío, solo binarios estáticos |

> **Regla**: Usa `slim` para desarrollo, `alpine`/`distroless`/`scratch` para producción.

---

## 4. RUN - Optimización de Capas

### 4.1 Combinar Comandos (Menos Capas)

```dockerfile
# ❌ MALO: 4 capas, caché ineficiente
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*

# ✅ BUENO: 1 capa, limpio, caché eficiente
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*
```

### 4.2 Orden de Instrucciones (Caché)

```dockerfile
# ✅ ORDEN ÓPTIMO: Lo que menos cambia PRIMERO
FROM python:3.11-slim

# 1. Sistema (casi nunca cambia)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 2. Dependencias Python (cambia ocasionalmente)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. Código aplicación (cambia SIEMPRE → AL FINAL)
COPY . .
```

### 4.3 Limpieza en la Misma Capa

```dockerfile
RUN apt-get update && apt-get install -y \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Para pip
RUN pip install --no-cache-dir -r requirements.txt

# Para npm
RUN npm ci --only=production && npm cache clean --force

# Para go
RUN go build -ldflags="-s -w" -o /app/bin .
```

---

## 5. COPY vs ADD

```dockerfile
# ✅ COPY: Simple, predecible, preferido
COPY src/ /app/src/
COPY requirements.txt .
COPY --chown=user:group archivo.txt /dest/

# ⚠️ ADD: Solo si necesitas sus features extra
# - URLs remotas: ADD https://ejemplo.com/archivo.tar.gz /tmp/
# - Auto-extract .tar/.tar.gz: ADD archivo.tar.gz /dest/
# - De lo contrario, usa COPY
```

---

## 6. CMD vs ENTRYPOINT

| Escenario | Uso |
|-----------|-----|
| Comando por defecto, sobrescribible | `CMD ["ejecutable", "arg1"]` |
| Ejecutable fijo, args variables | `ENTRYPOINT ["ejecutable"]` + `CMD ["arg1"]` |
| Wrapper/script de entrada | `ENTRYPOINT ["/entrypoint.sh"]` |

```dockerfile
# Patrón recomendado: Entrypoint + CMD por defecto
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["start"]

# entrypoint.sh
#!/bin/sh
set -e
# Setup: migraciones, configs, etc.
exec "$@"
```

> **Nota**: `docker run imagen arg` sobrescribe `CMD`, no `ENTRYPOINT`. Usa `--entrypoint` para cambiar entrypoint.

---

## 7. Multi-Stage Builds (Build Avanzado)

### 7.1 Patrón Clásico: Build → Runtime

```dockerfile
# ===== STAGE 1: BUILDER =====
FROM golang:1.21-alpine AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/bin .

# ===== STAGE 2: RUNTIME (MÍNIMA) =====
FROM gcr.io/distroless/static-debian12 AS runtime

COPY --from=builder /app/bin /app/bin
USER nonroot:nonroot

ENTRYPOINT ["/app/bin"]
```

### 7.2 Python Multi-Stage

```dockerfile
# Builder: compila dependencias nativas
FROM python:3.11-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc python3-dev libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /wheels
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# Runtime: solo runtime, sin compilador
FROM python:3.11-slim AS runtime

RUN groupadd -r app && useradd -r -g app app
WORKDIR /app

COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/* \
    && rm -rf /wheels

COPY --chown=app:app . .
USER app

CMD ["python", "main.py"]
```

### 7.3 Target Stages (Build Parcial)

```bash
# Solo construir hasta stage 'builder'
docker build --target builder -t miapp:builder .

# Usar stage específico como base para debug
docker build --target builder -t miapp:debug .
docker run -it --rm miapp:debug sh
```

---

## 8. BuildKit (Builder Moderno)

```bash
# Habilitar BuildKit (Docker 18.09+)
export DOCKER_BUILDKIT=1

# O en daemon.json permanentemente
# { "features": { "buildkit": true } }
```

### 8.1 Sintaxis Avanzada (Heredoc, Cache Mounts)

```dockerfile
# syntax = docker/dockerfile:1.7

FROM python:3.11-slim

# Cache de pip entre builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# Cache de apt
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl

# Secrets (no en imagen final)
RUN --mount=type=secret,id=npmrc \
    npm ci
```

```bash
# Build con secrets
docker build --secret id=npmrc,src=.npmrc .
```

---

## 9. .dockerignore (Esencial)

```gitignore
# .dockerignore - Igual que .gitignore pero para build context
.git
.github
.gitignore
.dockerignore
README.md
Dockerfile*
docker-compose*.yml
*.md
tests/
*.pyc
__pycache__/
.pytest_cache/
.coverage
.vscode/
.idea/
*.log
.env
.env.*
!/.env.example
node_modules/
dist/
build/
*.tar.gz
```

> **Impacto**: Reduce context size → builds más rápidos, menos memoria, sin secretos filtrados.

---

## 10. Comandos de Build

```bash
# Build básico
docker build -t miapp:1.0 .

# Build con tag latest también
docker build -t miapp:1.0 -t miapp:latest .

# Build con BuildKit y progress plano
DOCKER_BUILDKIT=1 docker build --progress=plain -t miapp:1.0 .

# Build multi-platform
docker buildx build --platform linux/amd64,linux/arm64 -t miapp:1.0 --push .

# Build con args
docker build --build-arg VERSION=1.0.0 -t miapp:${VERSION} .

# Build sin caché (forzar rebuild)
docker build --no-cache -t miapp:1.0 .

# Build desde URL (GitHub)
docker build https://github.com/usuario/repo.git#main -t miapp .

# Ver capas e historia
docker history miapp:1.0

# Inspeccionar imagen
docker inspect miapp:1.0
```

---

## 11. Testing de Imágenes

```bash
# Test rápido: comando ayuda
docker run --rm miapp:1.0 --help

# Test entrada interactiva
docker run -it --rm --entrypoint sh miapp:1.0

# Test usuario no-root
docker run --rm miapp:1.0 id

# Test healthcheck
docker run -d --name test miapp:1.0
sleep 10
docker inspect test --format='{{.State.Health.Status}}'
docker rm -f test

# Análisis seguridad
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image miapp:1.0
```

---

## 12. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Gestión de Contenedores](./Docker%20-%20Gesti%C3%B3n%20de%20Contenedores.md)
- [Multi-stage y Optimización](./Docker%20-%20Multi-Stage%20y%20Optimizaci%C3%B3n.md)
- [Buenas Prácticas y Seguridad](./Docker%20-%20Buenas%20Pr%C3%A1cticas%20y%20Seguridad.md)

---

*Etiquetas: #docker #dockerfile #build #multistage #buildkit #cache #layers #optimizacion #security #distroless*