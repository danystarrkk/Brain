---
title: "Docker - Buenas Prácticas y Seguridad"
tags:
  - lang/docker
  - concept/security
  - concept/best-practices
  - concept/hardening
  - level/avanzado
  - type/reference
  - area/devops
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Buenas Prácticas y Seguridad

> **Objetivo**: Construir imágenes seguras, minimalistas, y configurar contenedores con principio de menor privilegio.

---

## 1. Principios Fundamentales

| Principio | Aplicación |
|-----------|------------|
| **Menor privilegio** | Usuario no-root, capabilities mínimas, read-only fs |
| **Superficie mínima** | Imágenes distroless/scratch, solo lo necesario |
| **Inmutabilidad** | No modificar contenedores en producción |
| **Cifrado** | TLS en registry, secrets en runtime |
| **Auditoría** | Escaneo continuo, logging centralizado |
| **Actualizaciones** | Base images actualizadas, rebuild automatizado |

---

## 2. Hardening de Imágenes

### 2.1 Base Images Seguras

```dockerfile
# ❌ EVITAR
FROM ubuntu:latest          # Tag mutable, grande
FROM python:3.11            # Incluye build tools
FROM node:20                # Incluye npm, yarn, etc.

# ✅ RECOMENDADO
FROM python:3.11-slim       # Solo runtime, ~40MB
FROM python:3.11-alpine     # Muy pequeño, musl libc
FROM gcr.io/distroless/python3-debian11  # Sin shell, sin package manager
FROM scratch                # Solo binario estático (Go, Rust)
```

### 2.2 Multi-Stage para Separar Build/Runtime

```dockerfile
# Build stage - con herramientas
FROM golang:1.21-alpine AS builder
RUN apk add --no-cache git make
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/bin .

# Runtime stage - MÍNIMO
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/bin /app/bin
USER nonroot:nonroot
ENTRYPOINT ["/app/bin"]
```

### 2.3 Usuario No-Root Obligatorio

```dockerfile
# Crear usuario y grupo
RUN groupadd -r appgroup -g 1000 \
    && useradd -r -g appgroup -u 1000 -s /sbin/nologin -d /app appuser

# Cambiar ownership de archivos necesarios
COPY --chown=appuser:appgroup . /app
WORKDIR /app

# Cambiar a usuario no-root
USER appuser

# Verificar en runtime
# docker run --rm miapp id
# uid=1000(appuser) gid=1000(appgroup) groups=1000(appgroup)
```

---

## 3. Configuración de Contenedor Segura

### 3.1 Flags de Seguridad en `docker run`

```bash
# Filesystem read-only
docker run --read-only \
    --tmpfs /run --tmpfs /tmp \
    miapp

# Capabilities: quitar todas, añadir solo necesarias
docker run --cap-drop ALL \
    --cap-add NET_BIND_SERVICE \
    miapp

# No new privileges (previene privilege escalation)
docker run --security-opt no-new-privileges:true \
    miapp

# Seccomp profile personalizado
docker run --security-opt seccomp=perfil.json \
    miapp

# AppArmor
docker run --security-opt apparmor=mi-perfil \
    miapp

# Usuario específico
docker run --user 1000:1000 \
    miapp

# Límites recursos
docker run --cpus="1" --memory="512m" --pids-limit=100 \
    miapp
```

### 3.2 Docker Compose Seguridad

```yaml
services:
  app:
    image: miapp:1.0
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /run
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Solo si necesita puerto < 1024
    security_opt:
      - no-new-privileges:true
      - seccomp:default.json
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
    ulimits:
      nofile:
        soft: 64000
        hard: 64000
```

### 3.3 Seccomp Profile Personalizado

```json
// perfil-seccomp.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {"names": ["read", "write", "open", "close", "stat", "fstat", "lstat", "poll", "epoll_wait", "epoll_ctl", "epoll_pwait", "select", "pselect6"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["socket", "connect", "accept", "bind", "listen", "getsockname", "getpeername", "setsockopt", "getsockopt"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["clone", "fork", "vfork", "execve", "exit", "exit_group", "wait4", "waitpid"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["brk", "mmap", "mprotect", "munmap", "mremap", "msync", "mincore", "madvise"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["rt_sigaction", "rt_sigprocmask", "rt_sigreturn", "sigaltstack"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["futex", "nanosleep", "clock_gettime", "gettimeofday"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["getpid", "getppid", "getuid", "geteuid", "getgid", "getegid", "gettid", "sysinfo"], "action": "SCMP_ACT_ALLOW"}
  ]
}
```

```bash
# Usar
docker run --security-opt seccomp=perfil-seccomp.json miapp
```

---

## 4. Gestión de Secretos

### 4.1 NO Hacer Nunca

```dockerfile
# ❌ NUNCA en Dockerfile
ENV DB_PASSWORD=secret
ARG API_KEY
RUN echo "secret" > /tmp/key

# ❌ NUNCA en docker-compose.yml
environment:
  - DB_PASSWORD=secret

# ❌ NUNCA en imagen (history lo guarda)
RUN curl -H "Authorization: Bearer token" ...
```

### 4.2 Formas Correctas

#### A. Build Args (Solo Build Time)

```dockerfile
# Dockerfile
ARG NPM_TOKEN
RUN npm ci --registry=https://npm.pkg.github.com

# Build
docker build --build-arg NPM_TOKEN=$NPM_TOKEN .
```

#### B. Variables de Entorno (Runtime)

```yaml
# docker-compose.yml
services:
  app:
    env_file: .env.prod  # No commitear .env.prod
    environment:
      - DB_HOST=db
```

#### C. Docker Secrets (Swarm Mode)

```yaml
# docker-compose.yml (deploy en Swarm)
services:
  app:
    image: miapp:1.0
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    external: true
  api_key:
    file: ./secrets/api_key.txt
```

```bash
# Crear secrets
echo "supersecret" | docker secret create db_password -
docker secret create api_key ./secrets/api_key.txt
```

#### C. HashiCorp Vault / AWS Secrets Manager / Azure Key Vault

```bash
# Sidecar pattern con Vault Agent Injector
# O CSI driver para Kubernetes
```

---

## 5. Escaneo de Vulnerabilidades

### 5.1 Herramientas Recomendadas

| Herramienta | Uso | Integración |
|-------------|-----|-------------|
| **Trivy** | Vuln DB, misconfig, secrets | CI/CD, IDE, pre-commit |
| **Grype** | Vuln DB (Syft) | CI/CD |
| **Snyk** | Vuln + licencia + IaC | CI/CD, IDE |
| **Hadolint** | Dockerfile linting | CI/CD, pre-commit |
| **Dockle** | CIS Benchmark | CI/CD |
| **Checkov** | IaC (Dockerfile, compose, K8s) | CI/CD |

### 5.2 Pipeline CI/CD Ejemplo (GitHub Actions)

```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Hadolint
      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3
        with:
          dockerfile: Dockerfile
      
      # Build imagen
      - name: Build image
        run: docker build -t miapp:${{ github.sha }} .
      
      # Trivy scan
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: miapp:${{ github.sha }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
      
      # Upload SARIF to GitHub Security
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy.sarif
```

### 5.3 Escaneo Local

```bash
# Trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image miapp:latest

# Trivy con salida tabla
trivy image --severity HIGH,CRITICAL --format table miapp:latest

# Grype
docker run --rm anchore/grype miapp:latest

# Hadolint
docker run --rm -i hadolint/hadolint < Dockerfile

# Dockle (CIS Docker Benchmark)
docker run --rm goodwithtech/dockle miapp:latest
```

---

## 5.4 Interpretar Resultados

```bash
# Trivy severities
# CRITICAL: Explotable remotamente, RCE, auth bypass
# HIGH: Privilege escalation, DoS, info disclosure
# MEDIUM: Configuración insegura, versión vulnerable
# LOW: Info disclosure menor
# UNKNOWN: No clasificado

# Filtrar solo críticos/altos
trivy image --severity HIGH,CRITICAL --ignore-unfixed miapp:latest

# Ignorar vulnerabilidades específicas (con justificación)
trivy image --ignorefile .trivyignore miapp:latest
```

```text
# .trivyignore
# CVE-2023-XXXX: No afecta nuestra configuración (justificación)
CVE-2023-XXXX
```

---

## 6. CIS Docker Benchmark

### 6.1 Controles Críticos (Nivel 1)

| Control | Descripción | Verificación |
|---------|-------------|--------------|
| 4.1 | Usuario no-root | `docker inspect --format '{{.Config.User}}' img` |
| 4.2 | Read-only rootfs | `docker inspect --format '{{.HostConfig.ReadonlyRootfs}}'` |
| 4.3 | No privileged | `docker inspect --format '{{.HostConfig.Privileged}}'` |
| 4.4 | Capabilities mínimas | `docker inspect --format '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}'` |
| 4.5 | No new privileges | `docker inspect --format '{{.HostConfig.SecurityOpt}}'` |
| 4.6 | Resource limits | `docker inspect --format '{{.HostConfig.Memory}} {{.HostConfig.CpuQuota}}'` |
| 5.1 | Imagen base actualizada | `docker pull baseimage && docker build --no-cache` |
| 5.2 | No secrets en imagen | `trivy image --scanners secret miapp` |
| 5.3 | Verificar integridad | `docker image inspect --format '{{.RepoDigests}}'` |

### 6.2 Automatización con Dockle

```bash
# Dockle implementa CIS Docker Benchmark
docker run --rm goodwithtech/dockle miapp:latest

# Output con códigos CIS
# WARN  - CIS-DI-0001: Create a user for the container
# FATAL - CIS-DI-0005: Enable user namespace
# INFO  - CIS-DI-0008: Confirm safety of setuid/setgid files
```

---

## 7. Runtime Security (Falco, Sysdig)

### 7.1 Falco (Detección Anomalías Runtime)

```bash
# Instalar en host
curl -s https://falco.org/repo/falcosecurity-packages.asc | sudo apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt update && sudo apt install -y falco

# Reglas personalizadas
# /etc/falco/falco_rules.local.yaml
```

### 7.2 Reglas Comunes

```yaml
# Detectar shell en contenedor
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: container and proc.name in (bash, sh, zsh, ksh)
  output: "Shell spawn in container (user=%user.name container=%container.name)"
  priority: WARNING

# Detectar escritura en /etc
- rule: Write to /etc in container
  desc: Writing to /etc directory in container
  condition: container and open_write and fd.directory=/etc
  output: "Write to /etc in container (file=%fd.name)"
  priority: ERROR
```

---

## 8. Supply Chain Security

### 8.1 Firmar Imágenes (Cosign/Sigstore)

```bash
# Instalar cosign
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign

# Generar par de claves
cosign generate-key-pair

# Firmar imagen
cosign sign --key cosign.key miapp:1.0

# Verificar firma
cosign verify --key cosign.pub miapp:1.0
```

### 8.2 SBOM (Software Bill of Materials)

```bash
# Generar SBOM con Syft
docker run --rm anchore/syft miapp:latest -o spdx-json > sbom.spdx.json

# O con Trivy
trivy image --format spdx-json --output sbom.spdx.json miapp:latest
```

### 8.3 Verificar Provenance (SLSA)

```bash
# Verificar attestations
cosign verify-attestation --type slsaprovenance miapp:1.0
```

---

## 9. Actualizaciones y Mantenimiento

### 9.1 Dependabot / Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"
```

### 9.2 Rebuild Automatizado Semanal

```yaml
# .github/workflows/weekly-rebuild.yml
on:
  schedule:
    - cron: '0 2 * * 1'  # Lunes 2 AM

jobs:
  rebuild:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build with no cache
        run: docker build --no-cache -t miapp:latest .
      - name: Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: miapp:latest
          severity: CRITICAL,HIGH
      - name: Push if clean
        if: success()
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USER }} --password-stdin
          docker push miapp:latest
```

---

## 10. Checklist de Seguridad Pre-Deploy

```
IMAGEN:
[ ] Base image con tag fijo (no latest)
[ ] Multi-stage build (build ≠ runtime)
[ ] Usuario no-root (UID > 0)
[ ] Read-only rootfs + tmpfs para /tmp, /run
[ ] Capabilities: DROP ALL, ADD solo necesarias
[ ] No new privileges
[ ] Seccomp profile (default o custom)
[ ] Sin secrets en capas (trivy --scanners secret)
[ ] Vulnerabilidades CRITICAL/HIGH = 0 (trivy/grype)
[ ] Hadolint pasa sin warnings
[ ] Dockle pasa CIS Benchmark
[ ] Imagen firmada (cosign)
[ ] SBOM generado (syft)

RUNTIME:
[ ] --user no-root
[ ] --read-only + tmpfs
[ ] --cap-drop ALL --cap-add MINIMAS
[ ] --security-opt no-new-privileges:true
[ ] --security-opt seccomp=perfil.json
[ ] Resource limits (CPU, RAM, PIDs)
[ ] --pids-limit < 100
[ ] Red aislada (internal: true si no necesita internet)
[ ] Volúmenes read-only donde posible
[ ] Logging configurado (max-size, max-file)
[ ] Healthcheck configurado
[ ] Restart policy: unless-stopped
[ ] Secrets via Docker Secrets / Vault (nunca env vars)
[ ] Monitoring/Falco activo
```

---

## 11. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Dockerfile y Construcción](./Docker%20-%20Dockerfile%20y%20Construcci%C3%B3n%20de%20Im%C3%A1genes.md)
- [Gestión de Contenedores](./Docker%20-%20Gesti%C3%B3n%20de%20Contenedores.md)
- [Laboratorios y Vulhub](./Docker%20-%20Laboratorios%20y%20Vulhub.md)

---

*Etiquetas: #docker #security #hardening #cis-benchmark #trivy #grype #hadolint #secrets #cosign #sbom #falco #seccomp #capabilities #read-only #supply-chain*