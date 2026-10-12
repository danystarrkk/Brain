---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Buenas Prácticas y Seguridad

Un contenedor **no es una frontera de seguridad**: es un proceso con defensas apilables (namespaces, capabilities, cgroups, seccomp, AppArmor — ver [[Docker - Fundamentos y Conceptos]]). Endurecerlo significa aplicar **defensa en profundidad** y **menor privilegio** en cada capa: imagen mínima, usuario sin privilegios, filesystem de solo lectura, capacidades mínimas, secretos que no vivan en capas, y un pipeline que escanee y firme. Esta nota es la guía completa, y su contraparte ofensiva es [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — lo que allí se explota es exactamente lo que aquí se cierra.

## Principios (y su traducción concreta)

| Principio | Qué significa en Docker | Dónde se aplica |
|-----------|-------------------------|-----------------|
| **Menor privilegio** | Root solo si es obligatorio; caps mínimas; sin `--privileged` | Dockerfile `USER`, `--cap-drop`, `--security-opt` |
| **Superficie mínima** | Imagen = solo runtime; sin shells/package managers si se puede | Multi-stage, distroless, scratch |
| **Inmutabilidad** | Nunca parchar contenedores vivos | Nuevo build, no `docker exec apt install` |
| **Segreación de secretos** | Nada sensible en imágenes ni YAML | secrets mounts, Vault, CI secrets |
| **Reproducibilidad** | Pines de versión/digest, lockfiles | `FROM` digest, `npm ci`, SBOM |
| **Observabilidad** | Logs a stdout + escaneo continuo | logging driver, Trivy en CI, Falco |
| **Respuesta** | Rebuild ante CVE de base | Dependabot/Renovate, rebuild semanal |

## Elección de imagen base — tabla comparativa

| Base | Tamaño | Shell | Package manager | CVE surface | Cuándo |
|------|--------|-------|-----------------|-------------|--------|
| `ubuntu` / `debian` | 70–150 MB | bash | apt | Alta | Dev, imágenes con tooling |
| `*-slim` (python, node…) | 40–90 MB | bash | apt (parcial) | Media | **Default sensato**: compatibilidad glibc |
| `*-alpine` | 5–20 MB | sh | apk | Baja (musl) | Prod con cuidado: musl puede romper wheels/behaviors (DNS search, `getent`) |
| `distroless` (Google) | 2–20 MB | ❌ **ninguno** | ❌ | Mínima | **Prod recomendada**: sin shell no hay `sh` que invocar en un RCE |
| `scratch` | 0 | ❌ | ❌ | Cero (es vacío) | Binarios estáticos Go/Rust |
| Distroless + debugging | + busybox | sh temporal | — | Media | `:debug` para diagnosticar y salir |

```dockerfile
# Patrón hardening completo en 2 etapas
FROM python:3.12-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends gcc libpq-dev && rm -rf /var/lib/apt/lists/*
WORKDIR /wheels
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY --chown=nonroot:nonroot . .
# sin USER: la variante :nonroot ya corre como UID 65532
ENTRYPOINT ["python", "main.py"]
```

> [!info] Distroless sin shell: por qué importa
> El 90 % de la explotación post-RCE usa `sh` para lateralizar (`wget|sh`, reverse shells). Sin shell y sin package manager, un RCE "solo" puede correr **tu binario** — reduce drásticamente lo que [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] puede hacer desde dentro. El costo: no podés `docker exec sh`. Se diagnostica con imágenes `:debug` temporales o `docker cp`/`docker export`.

Multi-stage para excluir herramientas de build: gcc, npm, SDKs de Go, llaves de SSH del builder **no llegan** al stage final si no los copiás — ver recetas en [[Docker - Dockerfile y Construcción de Imágenes]].

## Usuario no-root: la mitigación nº1

```dockerfile
RUN groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --home /app --shell /usr/sbin/nologin app
WORKDIR /app
COPY --chown=app:app . .
USER app
```

- **UIDs numéricos** (`USER 10001:10001`) evitan depender de `/etc/passwd` y de colisiones de nombres.
- El usuario debe poder leer lo que necesita (`--chown` en `COPY`) pero **no escribir** donde no hace falta.
- Verificación: `docker run --rm --entrypoint id miapp` → debe decir `uid=10001`.
- Impacto real: con root interno, cualquier escape de app (RCE, path traversal) ya tiene `CAP_DAC_OVERRIDE` y acceso casi total al contenedor; con UID 10001, la mayoría de ataques muere en permisos.

## Filesystem de solo lectura + tmpfs

```bash
docker run --read-only --tmpfs /tmp:rw,size=64m,mode=1777 \
    --tmpfs /run miapp
```

```yaml
# Compose
services:
  api:
    read_only: true
    tmpfs: ["/tmp", "/run"]
    volumes: ["uploadcache:/app/uploads"]   # solo los paths que DEBEN escribir
```

`--read-only` no impide escribir en volúmenes montados ni en tmpfs: la app se moderniza para no asumir escritura en su propio root. Efecto: una payload que quiera soltar un binario en `/tmp`… no puede escribirlo en disco persistente; y `docker diff` del contenedor queda limpio.

## Capabilities y `no-new-privileges`

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE --security-opt no-new-privileges:true miapp
```

| Mecanismo | Qué corta |
|-----------|-----------|
| `--cap-drop ALL` | Todo el superpoder root del contenedor; añadir solo lo demostrado necesario (`NET_BIND_SERVICE` si bindea <1024; `CHOWN`/`SETUID` si el entrypoint los usa) |
| `no-new-privileges:true` | `setuid`/`setgid` y `execve` con filecaps **no elevan**: mata la escalada via binarios SUID (clásico `chmod u+s /bin/bash`) |
| `--security-opt seccomp=perfil.json` | Filtro de syscalls (default: ~44 bloqueadas, p.ej. `mount`, `kexec_load`, `bpf`, `add_key`) |
| `--security-opt apparmor=docker-default` | Perfil MAC por defecto (Ubuntu/Debian) |
| `--pids-limit` | Fork bombs |
| `--network none` / `internal` | lateralización de red |

Las capabilities por defecto del contenedor y cuáles retira Docker: tabla completa en [[Docker - Fundamentos y Conceptos]]. Para una auditoría de un contenedor vivo: `docker inspect c --format '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}} {{.HostConfig.SecurityOpt}}'`.

Perfil seccomp mínimo de ejemplo (la default de Docker ya es decente; personalizás cuando la app necesita syscalls que la default bloquea):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    { "names": ["read", "write", "open", "close", "stat", "fstat", "lstat", "poll", "epoll_wait",
                "socket", "connect", "accept", "bind", "listen", "clone", "fork", "execve",
                "exit_group", "wait4", "brk", "mmap", "mprotect", "munmap", "rt_sigaction",
                "futex", "nanosleep", "clock_gettime", "gettimeofday", "getpid", "getuid", "geteuid"],
      "action": "SCMP_ACT_ALLOW" }
  ]
}
```

> `defaultAction: ERRNO` (allowlist) es el modo seguro; la default de Docker es también allowlist con reglas de compatibilidad. Bloquear por denylist es un espejismo.

## Secretos: lo que NO hacer y las tres formas correctas

```dockerfile
# ❌ Todo esto deja el secreto en la imagen (docker history / capas):
ENV API_KEY=abc123
ARG NPM_TOKEN
RUN npm config set //npm.pkg.github.com/:_authToken=${NPM_TOKEN}
RUN curl -H "Authorization: Bearer $TOKEN" https://internal/tool && ./install.sh
```

> [!warning] Las capas son para siempre
> Una instrucción con un secreto queda en el historial del build **aunque la borres en una capa posterior** (`RUN echo s > /f && rm /f` sigue siendo inspeccionable con `docker history`/`docker save`). Si un secreto coló: **rotalo** (revocalo) — borrar la capa no basta, y si ya se push-eó, está en el registry para quien tenga acceso.

| Forma | Mecanismo | Ventaja | Uso |
|-------|-----------|---------|-----|
| **Build secret (BuildKit)** | `RUN --mount=type=secret,id=x` + `docker build --secret` | Nunca entra a capa ni cache | Tokens de registry privado, npmrc |
| **Docker secrets (Swarm)** | Archivo montado en `/run/secrets/<nombre>` | No es env var; rotación por orquestador | Credenciales de DB en Swarm/Stacks |
| **env vars de runtime** | `-e` / `env_file` / K8s secrets→env | Simple | Menos ideal: visibles en `inspect`, `/proc`, dumps, logs de crash |
| **Vault / cloud secret managers** | Sidecar/injector/CSI | Rotación, auditoría, dynámicos | Prod real |
| **Archivos externos** | Bind/secret file a los que la app lee | Cero magia | TLS keys, config sensible |

```yaml
# Compose + Swarm: secrets como archivos
services:
  api:
    secrets: [db_password]
secrets:
  db_password: { file: ./secrets/pg.pass }        # compose simple: declarativo (no monta real sin Swarm)
# En Swarm real: echo "s3cret" | docker secret create db_password -
# Montaje: /run/secrets/db_password — la app lee el FILE, no variable
```

Regla de la app: leer credenciales de **archivo** (`DB_PASSWORD_FILE`), así funciona igual con secrets de Swarm, Vault y Kubernetes.

## Secretos en la historia del build — detección

```bash
docker history --no-trunc miapp | grep -iE "token|secret|password|_auth"   # indicio de fuga
docker save miapp | tar -xO manifest.json                                  # inspección offline
trivy image --scanners secret miapp                                        # buscador de secretos en capas
gitleaks detect --source .                                                 # también en el repo (Dockerfile incluido)
```

Y la higiene de CI: secretos solo desde el vault del sistema (`secrets.*` de GitHub Actions), `--password-stdin` en `docker login` (nunca en argv: queda en `ps`), y nunca `set -x` con credenciales en pipelines.

## Escaneo de vulnerabilidades

| Herramienta | Qué hace | Cuándo |
|-------------|----------|--------|
| **Trivy** | CVEs por paquete + secretos + IaC + SBOM | CI, pre-commit, local |
| **Docker Scout** | CVEs con contexto Docker, policy checks, delta entre tags | Integrado en CLI/Hub/ACR |
| **Grype** (Anchor) | CVEs (motor Syft) | Alternativa a Trivy |
| **Hadolint** | Lint de Dockerfile (estilo+seguridad) | Pre-commit y CI |
| **Dockle** | CIS Docker Benchmark a imágenes | CI |
| **Checkov** | IaC: Dockerfile/compose/K8s | CI |
| **cosign / syft** | Firmas y SBOM | Supply chain |

```bash
docker scout quickview miapp:1.0                       # vista rápida en CLI
docker scout cves --only-fixed miapp:1.0               # solo con fix disponible
trivy image --severity HIGH,CRITICAL --ignore-unfixed miapp:1.0
trivy image --scanners vuln,secret --format table miapp:1.0
docker run --rm -i hadolint/hadolint < Dockerfile      # lint
cat .trivyignore                                       # CVEs aceptados con justificación (con fecha y ticket)
```

Cómo interpretar: priorizá **CRITICAL/HIGH con fix** y **secretos detectados** (incidente inmediato); MEDIUM/LOW documentás. Un CVE en la base sin explotabilidad en tu configuración se justifica en `.trivyignore`, no se ignora.

## Higiene de registries y etiquetado

| Práctica | Mal | Bien |
|----------|-----|------|
| Tags | `latest` en prod | `1.4.2`, y `main-<sha>` para dev |
| Provenance | "¿de dónde salió esta imagen?" | Tag = git sha; CI es el único que push-ea |
| Digest pinning | `FROM node:22` | `FROM node@sha256:…` para bases críticas |
| Acceso | Hub público + autodescubierto | GHCR/privado + permisos mínimos + `docker login --password-stdin` |
| Retención | Imágenes eternas | Lifecycle: borrar tags viejos, mantener últimos N |
| Firma | nadie verifica | `cosign verify` en deploy (ver supply chain) |

```bash
# El flujo mínimo honesto de una imagen interna
docker build -t registry.internal/shop/api:1.4.2 -t registry.internal/shop/api:sha-${GITSHA} .
docker push registry.internal/shop/api:1.4.2
docker pull registry.internal/shop/api@sha256:abc...   # deploy pinnnea digest
```

## Supply chain: firmas, attestations y SBOM

```bash
# Cosign (Sigstore): firma y verificación
cosign generate-key-pair                                  # o keyless con OIDC
cosign sign --key cosign.key registry.internal/shop/api:1.4.2
cosign verify --key cosign.pub registry.internal/shop/api:1.4.2   # ← el deploy verifica

# SBOM: inventario de componentes
docker sbom --format spdx-json miapp:1.0 > sbom.spdx.json   # o: syft / trivy sbom
cosign attest --predicate sbom.spdx.json --type spdx registry.internal/shop/api:1.4.2

# Provenance de build (SLSA): qué, quién y cómo lo construyó
docker buildx build --provenance=mode=max -t miapp:1.0 --push .
```

El valor: ante un CVE tipo log4shell, el SBOM te dice **qué imágenes tuyas lo contienen** en minutos. Sin SBOM, es una cacería manual.

## Patrones seguros de CI/CD

```yaml
# .github/workflows/image-security.yml (patrón)
jobs:
  build-scan-sign:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, security-events: write }
    steps:
      - uses: actions/checkout@v4
      - name: Lint
        run: docker run --rm -i hadolint/hadolint < Dockerfile
      - name: Build (sha tag)
        run: docker build -t miapp:sha-${{ github.sha }} .
      - name: Scan
        run: trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 miapp:sha-${{ github.sha }}
      - name: Sign (keyless)
        run: cosign sign --yes miapp:sha-${{ github.sha }}
      - name: Push
        run: echo "${{ secrets.REG_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
             && docker push miapp:sha-${{ github.sha }}
```

Puntos que hacen la diferencia: **el build falla si hay CRITICAL con fix** (`--exit-code 1`); el tag es el sha (reproducible); el push solo lo hace CI (nadie más publica); login con `--password-stdin`.

## Actualización continua de bases

```yaml
# dependabot.yml / renovate: rebuild cuando la base se parchea
updates:
  - package-ecosystem: docker
    directory: "/"
    schedule: { interval: weekly }
```

```yaml
# rebuild semanal (lunes 2 AM): CVEs de la base se cierran aunque tu código no cambie
on: { schedule: [{ cron: "0 2 * * 1" }] }
jobs:
  rebuild: { steps: [ { uses: actions/checkout@v4}, { run: docker build --no-cache -t miapp:latest . && trivy image --exit-code 1 --severity CRITICAL miapp } ] }
```

Por qué: los CVEs viven en los **paquetes de la base**, no en tu código; sin rebuild periódico, tu imagen envejece sin que nadie toque nada.

## Aislamiento de recursos también es seguridad

Un contenedor sin límites es un vector DoS vecino: con `--memory`, `--cpus` y `--pids-limit` el kernel enmuere al abusador y no al host (semántica en [[Docker - Gestión de Contenedores]]). Añade `--ulimit nofile` para DBs y `logging` con `max-size` para que un log flood tampoco llene el disco. Y la red: servicios internos en `internal: true` — ver [[Docker - Redes y Volúmenes]].

## Checklist pre-deploy

```
IMAGEN
[ ] Base con tag fijo/digest; Dependabot/Renovate activo
[ ] Multi-stage: sin compiladores ni dev-deps en runtime
[ ] USER no-root (UID numérico) verificado con `id`
[ ] Hadolint sin WARN/ERROR
[ ] Trivy/Scout: 0 CRITICAL/HIGH con fix; secret-scan limpio
[ ] SBOM generado; imagen firmada (cosign)
[ ] LABELs OCI completos

RUNTIME
[ ] read_only + tmpfs donde la app escribe
[ ] cap_drop: ALL (+ mínimos demostrables)
[ ] security_opt: no-new-privileges:true (+ seccomp/apparmor)
[ ] --user numérico (o USER del Dockerfile)
[ ] límites: memory, cpus, pids-limit, ulimit nofile
[ ] red interna para db/cache; solo web publica puertos
[ ] secrets como archivos (/run/secrets) o Vault — nunca ENV/ARG
[ ] logging json-file con max-size/max-file
[ ] restart: unless-stopped; healthcheck real
[ ] sin mounts del socket de Docker ni --privileged
```

## Errores comunes de seguridad

1. **`--privileged` "temporal" en prod**: devuelve todas las caps, desactiva AppArmor/seccomp y da acceso a todos los devices. Es root del host con pasos extra. Alternativa: caps puntuales + device específico.
2. **Montar `/var/run/docker.sock`** en apps: control total del host ver [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]. Solo en herramientas de build/scan de confianza.
3. **Usuario non-root que rompe la app**: la app intenta escribir donde no puede → crashea. Solución: `--chown` correcto + read_only + tmpfs/volumen para escritura real, no volver a root.
4. **Alpine "por seguridad" que rompe comportamiento**: musl vs glibc (DNS, threads, wheels). Distroless/Alpine son decisiones de testing, no dogmas.
5. **Escaneo una sola vez**: el CVE llega mañana; el pipeline debe escanear **cada build** y las bases deben rebuildearse.
6. **`.env` con prod secrets en el repo**: git history también es para siempre. Vault/CI secrets + `.gitignore`.

## Cheatsheet de hardening

| Capa | Acción | Comando / clave |
|------|--------|-----------------|
| Imagen | mínima | multi-stage → distroless/scratch/alpine |
| Usuario | non-root | `USER 10001:10001` |
| FS | solo lectura | `--read-only` + `--tmpfs` |
| Caps | drop todo | `--cap-drop ALL --cap-add …` |
| Privilegios | sin elevación | `--security-opt no-new-privileges:true` |
| Syscalls | filtro | seccomp default o custom |
| Secretos | archivos | `--mount=type=secret` / `/run/secrets` |
| Red | mínima | networks `internal: true` |
| Límites | cgroups | `--memory --cpus --pids-limit` |
| Escaneo | por build | trivy/scout `--exit-code 1` |
| Integridad | firma | `cosign sign` / `cosign verify` |
| Updates | automatizar | Dependabot docker + rebuild semanal |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — las primitivas (namespaces, caps, cgroups) que estas defensas ajustan
- [[Docker - Dockerfile y Construcción de Imágenes]] — multi-stage, secrets de build, .dockerignore
- [[Docker - Gestión de Contenedores]] — flags de runtime completos (`--read-only`, `--cap-*`, límites)
- [[Docker - Redes y Volúmenes]] — redes internas, `:ro`, tmpfs y bind mounts seguros
- [[Docker - Laboratorios y Vulhub]] — ver la cara ofensiva de estos controles en un lab aislado
- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — los vectores reales que esta nota mitiga
- [[Docker - Comandos de Referencia Rápida]] — auditoría rápida con `inspect`/`history`
