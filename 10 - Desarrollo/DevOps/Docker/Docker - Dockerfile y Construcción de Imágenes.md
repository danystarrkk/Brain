---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Dockerfile y Construcción de Imágenes

El `Dockerfile` es el **código fuente de la imagen**: un guion de instrucciones que Docker ejecuta capa por capa para producir el artefacto inmutable que después corre en cualquier host (ver [[Docker - Fundamentos y Conceptos]]). Dominarlo no es "saber la lista de instrucciones" — es entender **el modelo de ejecución**: caché por capa, orden que importa, separación build/runtime, y las trampas de la shell. Esta nota es la referencia completa.

> [!info] El modelo mental en una frase
> Cada instrucción que **modifica el filesystem** (`RUN`, `COPY`, `ADD`) produce una capa nueva; el build verifica cada instrucción contra la caché **en orden**; la primera instrucción que cambia invalida **todas las que siguen**. Todo el arte de escribir Dockerfiles eficientes sale de ahí: ordenar de lo que menos cambia a lo que más cambia.

## Anatomía: sintaxis y orden de ejecución

```dockerfile
# Comentario (solo; no hay comentarios multi-línea)
# syntax=docker/dockerfile:1.7     ← directiva del parser (BuildKit), va antes de FROM
INSTRUCCIÓN argumentos              # una por línea; \ para continuar; # para comentario
```

- Docker ejecuta **todas** las instrucciones en orden. No hay `if`, `for` ni variables de shell nativas (las del shell se resuelven dentro de `RUN`).
- Instrucciones **no crean capa** (solo metadatos/config): `CMD`, `ENTRYPOINT`, `ENV`, `ARG`, `EXPOSE`, `VOLUME`, `USER`, `WORKDIR`, `LABEL`, `HEALTHCHECK`, `STOPSIGNAL`, `SHELL`, `ONBUILD`.
- Instrucciones **crean capa** (tocan el filesystem): `RUN`, `COPY`, `ADD`, y `VOLUME` cuenta como capa-marcador.

## Referencia instrucción por instrucción

### `FROM` — la base

```dockerfile
FROM python:3.12-slim                       # por defecto: la plataforma del builder
FROM --platform=linux/amd64 node:22-alpine  # fijar plataforma explícita
FROM scratch                                # base vacía (binarios estáticos)
FROM golang:1.23 AS builder                 # nombrar stage (multi-stage)
```

| Variante | Cuándo usarla |
|----------|---------------|
| `FROM ubuntu:24.04` | Tag exacto: reproducible y parcheado a voluntad |
| `FROM python@sha256:abc…` | Digest: **inmutable de verdad**; el tag ni siquiera se consulta |
| `FROM --platform=$BUILDPLATFORM golang AS builder` | Build multi-arch: compila en la plataforma del builder (rápido, emulación solo si hace falta) |
| `FROM scratch` | Binarios Go/Rust estáticos; cero superficie |

```dockerfile
# ARG antes de FROM es global y solo vale para la línea FROM:
ARG BASE_VERSION=3.12
FROM python:${BASE_VERSION}-slim
```

> [!warning] `FROM latest` es una bomba de reproducibilidad
> Cada build puede recibir una base distinta. Fijá tag (y revisá updates vía Dependabot/Renovate — ver [[Docker - Buenas Prácticas y Seguridad]]).

### `ARG` vs `ENV` — la distinción que más se confunde

| | `ARG` | `ENV` |
|--|-------|-------|
| **Ámbito** | Solo durante el **build** | Build **y runtime** del contenedor |
| **Persiste en la imagen** | ❌ No queda en `docker inspect` | ✅ Sí (`Config.Env`) |
| **Coste de caché** | Cambiar su valor **invalida** capas desde el primer `RUN` que lo use | Cambiar invalida igual |
| **Visible en** | `docker history` si se usa mal | `docker history` (¡cuidado con secretos!) |

```dockerfile
ARG APP_VERSION=0.0.0          # default en build
ARG NPM_TOKEN                  # sin default: obligatorio pasarla o warning
ENV PYTHONUNBUFFERED=1 \       # runtime: logs sin buffer (clave para docker logs)
    PYTHONDONTWRITEBYTECODE=1 \
    APP_VERSION=${APP_VERSION} # ← ARG "promovida" a ENV si la imagen la necesita en runtime
LABEL version="${APP_VERSION}" # se bakea en metadatos
```

```bash
docker build --build-arg APP_VERSION=1.4.2 -t miapp:1.4.2 .
```

> [!warning] Secrets por ARG = secretos en la imagen
> Un `ARG NPM_TOKEN` usado en un `RUN` queda en el historial del build (y la capa puede quedar en cache). Nunca metas credenciales por build-arg: usá `--mount=type=secret` (ver BuildKit más abajo) — detalle de seguridad completo en [[Docker - Buenas Prácticas y Seguridad]].

### `ENV` — variables de runtime

```dockerfile
ENV APP_HOME=/app
ENV APP_HOME=/app MODE=prod    # una línea, dos variables (más eficiente: una capa)
```

Se pueden leer en siguientes `RUN` (`$APP_HOME`) y todo proceso del contenedor las hereda. Recomendación: definir solo las que la app necesita; cada una es superficie de configuración.

### `COPY` — copiar del build context

```dockerfile
COPY src/ /app/src/                       # directorio: el *contenido* de src/ va a /app/src/
COPY requirements.txt .                   # WORKDIR como destino relativo
COPY --chown=app:app --chmod=755 entrypoint.sh /entrypoint.sh
COPY --from=builder /out/bin /app/bin     # ← multi-stage (copiar de otro stage)
COPY --link app/ /app/                    # BuildKit: capa independiente → mejor caché paralela
```

- **El build context** es el directorio que pasaste al `docker build` (el `.` final): solo de ahí se puede copiar. Contexto gigante = subida lenta de tar = build lento → `.dockerignore`.
- `COPY` de un solo archivo preserva los permisos del host; `COPY --chown` y `--chmod` evitan `RUN chown/chmod` extra (una capa menos).

### `ADD` — como COPY, con magia (que casi nunca querés)

| Comportamiento | `COPY` | `ADD` |
|----------------|--------|-------|
| Copia archivos/directorios | ✅ | ✅ |
| Descarga una **URL** | ❌ | ✅ (pero la descarga NO queda cacheada por URL: cualquier cambio remoto no invalida caché y el archivo queda en la capa) |
| Auto-extrae `.tar` local | ❌ | ✅ (descomprime en destino) |
| Predecible | ✅ | ❌ |

**Regla**: usá `COPY` por defecto. `ADD` solo cuando *realmente* querés auto-extraer un tar local. Para URLs: `RUN curl -fsSL …` (con `--mount=type=cache` si hace falta) es explícito y cacheable.

### `RUN` — ejecutar durante el build (la capa que más pesa)

```dockerfile
# ❌ Mal: 4 capas, basura acumulada, peor caché
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*

# ✅ Bien: una capa, sin basura
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl vim \
    && rm -rf /var/lib/apt/lists/*
```

- `apt-get update` **sin** `install` en la misma capa cachea un índice viejo: el día que el cache invalida, instalás contra un índice obsoleto → "Unable to locate package". Siempre juntos.
- `&&` en vez de `;`: que un paso falle detenga el build (BuildKit ya corta en el primer error de cada `RUN`, pero `;` escondía errores en el builder clásico).
- Comandos por ecosistema:

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt            # Python
RUN npm ci && npm prune --production                          # Node (npm ci = lockfile exacto)
RUN go build -trimpath -ldflags="-s -w" -o /app/bin .        # Go (sin symbols, sin paths)
RUN apk add --no-cache curl                                   # Alpine (sin cache de apk)
```

- **Heredoc (BuildKit)** para scripts multi-línea sin `RUN echo … >` sucio:

```dockerfile
RUN <<EOF
set -eux
apt-get update
apt-get install -y --no-install-recommends curl
rm -rf /var/lib/apt/lists/*
EOF
```

### `CMD` vs `ENTRYPOINT` — la tabla definitiva

| | `CMD` | `ENTRYPOINT` |
|--|-------|--------------|
| Propósito | Comando **por defecto** | **Ejecutable principal** del contenedor |
| ¿Se sobreescribe con `docker run imagen args`? | ✅ Sí (se reemplaza entero) | ❌ No: los args **se le añaden** |
| ¿Se sobreescribe con `--entrypoint`? | — | ✅ Sí |
| Forma recomendada | `CMD ["nginx", "-g", "daemon off;"]` | `ENTRYPOINT ["/entrypoint.sh"]` |

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]   # execución directa, sin shell
CMD ["start", "--verbose"]      # args por defecto
```

```bash
docker run miapp                     # → /entrypoint.sh start --verbose
docker run miapp worker              # → /entrypoint.sh worker        (args reemplazan CMD)
docker run --entrypoint sh miapp     # → sh                            (romper todo, para debug)
```

> [!info] Por qué la forma exec (`["a","b"]`) y no shell (`"a b"`)
> En forma shell, Docker lanza `/bin/sh -c "a b"`: el proceso **PID 1** del contenedor es `sh`, que no reenvía señales. `docker stop` envía SIGTERM, `sh` lo ignora, y tras 10 s Docker mata todo con SIGKILL → **las apps nunca ven el graceful shutdown** (conexiones cortadas, datos a medias). En forma exec, tu app es el PID 1 y recibe SIGTERM directamente. **Siempre exec form**, salvo que necesites explícitamente shell (`CMD` con variables de entorno expandidas, por ejemplo).

Script de entrada canónico (con `exec` para no romper señales):

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```
```sh
#!/bin/sh
set -e
# setup: migraciones, render de configs desde ENV, chown de mounts...
exec "$@"        # ← reemplaza el shell por el comando real como PID 1
```

### `WORKDIR` — directorio de trabajo (y la trampa del `cd`)

```dockerfile
WORKDIR /app              # crea el dir si no existe (a diferencia de cd)
RUN echo "esto corre en /app" > status.txt
COPY . .                  # destino relativo = /app
WORKDIR /app/subdir       # acumulativo
```

- ❌ `RUN cd /app && …`: el `cd` vive solo en esa capa; la siguiente instrucción vuelve al `/`. `WORKDIR` **persiste**.
- Un `WORKDIR` relativo sin definir cae en `/` y copia archivos al root: recetas de desorden y colisiones.
- `docker run -w` puede sobreescribirlo por invocación (ver [[Docker - Gestión de Contenedores]]).

### `USER` — quién ejecuta

```dockerfile
USER 1000:1000        # UID:GID numérico (evita depender de /etc/passwd)
USER appuser          # nombre definido previamente
```

- Afecta a `RUN` siguientes **y** a `CMD`/`ENTRYPOINT` en runtime. Si necesitás un `RUN` como root, ponelo **antes** del `USER`.
- Sin `USER`, todo corre como root dentro del contenedor: la mitigación nº1 de [[Docker - Buenas Prácticas y Seguridad]]. El usuario debe existir en la imagen (o ser numérico).

### `EXPOSE` — documentación, no publicación

```dockerfile
EXPOSE 80            # documenta el puerto (y lo abre para links entre stages)
EXPOSE 8080/tcp 53/udp
```

- **No** publica nada en el host: eso es `-p`/`ports:` en runtime (ver [[Docker - Redes y Volúmenes]]). `EXPOSE` deja metadatos (`docker inspect`), permite `docker run --expose` heredado y el linking entre stages. `EXPOSE 80` + sin `-p` = el mundo no lo ve.

### `VOLUME` — marcar puntos de datos

```dockerfile
VOLUME /data
```

Declara que `/data` debe vivir en un volumen. **Trampa**: si el contenedor escribe ahí y luego agregás una capa que modifica ese path en un `RUN` posterior, esos cambios **no persisten en la imagen** (el volumen los oculta). Úsalo para documentar intención; la definición seria de volúmenes es en `docker run -v` / Compose (ver [[Docker - Redes y Volúmenes]]).

### `HEALTHCHECK` — saber si la app está *viva de verdad*

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD curl -fsS http://localhost:8000/health || exit 1
# HEALTHCHECK NONE   ← desactivar el heredado de la imagen base
```

| Parámetro | Significado | Default |
|-----------|-------------|---------|
| `--interval` | Cadencia de chequeo | 30s |
| `--timeout` | Máx por chequeo (falla si se pasa) | 30s |
| `--start-period` | Gracia arranque (fallos no cuentan) | 0s |
| `--retries` | Fallos seguidos → `unhealthy` | 3 |
| `CMD-SHELL` vs `CMD` | forma shell (`curl … || exit 1`) vs exec | — |

Estado visible: `docker inspect --format '{{.State.Health.Status}}' c` → `starting|healthy|unhealthy`. En Compose es lo que hace funcionar `depends_on: condition: service_healthy` (ver [[Docker - Docker Compose]]). El chequeo debe ser **barato** y de verdad representar la salud (un `/health` que responda 200 aunque la DB esté caída miente).

### `LABEL` — metadatos

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/owner/repo" \
      org.opencontainers.image.version="1.4.2" \
      org.opencontainers.image.licenses="MIT" \
      maintainer="dev@ejemplo.com"
```

No afectan al runtime; sí al ecosistema (registros, SBOM, herramientas). Preferí los estándares **OCI** (`org.opencontainers.image.*`) sobre `maintainer` (obsoleto).

### `STOPSIGNAL` — señal de parada

```dockerfile
STOPSIGNAL SIGQUIT   # p.ej. nginx graceful con SIGQUIT
```

Por defecto SIGTERM. Si tu app no sabe manejar SIGTERM (mucho software Java viejo), declará la señal que sí procesa — o mejor, arreglá el entrypoint. Ver [[Docker - Gestión de Contenedores]] para la semántica de stop/kill.

### `SHELL` — cambiar el shell de la forma shell

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]   # bash con pipefail para | sensibles
```

Solo afecta a las instrucciones escritas en forma **shell** (`RUN echo x | …`). En Alpine no hay bash: `SHELL ["/bin/ash","-eo","pipefail","-c"]`.

### `ONBUILD` — triggers para imágenes hijas

```dockerfile
ONBUILD COPY . /app/src
ONBUILD RUN pip install -r /app/src/requirements.txt
```

Ejecuta estas instrucciones **en el build de la imagen que use esta como base** (patrón clásico: imágenes base de frameworks — la vieja `node` usaba `ONBUILD` para apps). Riesgo: el hijo no ve qué va a pasar hasta que falla; se documenta con `docker history`.

## Caché de build: mecánica profunda

1. El builder (BuildKit) evalúa instrucciones **en orden**. Para cada una calcula un hash de: instrucción + inputs (contenido de archivos copiados, valores de ARG usados) + imagen padre.
2. Si el hash coincide con un build anterior → **cache hit**: se reutiliza la capa, sin ejecutar nada.
3. La **primera** instrucción que cambia invalida **toda la cadena posterior** — por eso ordenar importa:

```dockerfile
# ✅ Orden óptimo: de menos a más cambiante
FROM node:22-slim
WORKDIR /app
COPY package.json package-lock.json ./   # 1) cambia poco
RUN npm ci                               # 2) cacheada mientras el lockfile no cambie
COPY . .                                 # 3) cambia en cada commit → al final
RUN npm run build                        # 4) hereda el cache de npm ci
```

| Técnica | Efecto |
|---------|--------|
| `.dockerignore` | Contexto chico → hash de `COPY` no cambia por `.git/`, docs, etc. |
| Copiar manifest antes que el código | `npm ci`/`pip install` cacheados por commit |
| `--mount=type=cache` | Caché **entre builds** (npm/pip/go) sin pegarla a la capa |
| `--mount=type=secret` | Credenciales sin capa ni caché contaminada |
| `--link` en `COPY` | Capa desacoplada del padre → reutilizable en paralelo |
| `--no-cache`, `--no-cache-filter` | Invalidar a propósito (builds de release/patching) |

> [!tip] Caché rota "fantasma"
> Cambiar un `ENV` o `ARG` usado antes invalida todo lo posterior aunque el código no cambió. Síntoma: builds lentos inexplicables. Diagnóstico: `docker build --progress=plain` mira desde qué instrucción ya no dice `CACHED`.

### `.dockerignore` — la diferencia entre 300 MB y 3 GB de contexto

```gitignore
# .dockerignore (como .gitignore pero para el build context)
.git
.github
.vscode
.idea
**/node_modules
**/__pycache__
**/*.pyc
**/.pytest_cache
**/.env
**/.env.*
!**/.env.example
*.md
tests/
dist/
build/
coverage/
*.log
docker-compose*.yml
Dockerfile*
```

Sin este archivo, cada `COPY . .` arrastra `.git` (decenas de MB), `node_modules` de tu host (binarios de otra plataforma que rompen el build) y **archivos `.env` con secretos que quedan pegados en la imagen** (ver seguridad en [[Docker - Buenas Prácticas y Seguridad]]).

## Multi-stage builds: la técnica que definió la Dockerfile moderna

Separar **build** (compiladores, SDKs, cachés) de **runtime** (solo lo necesario) en un solo Dockerfile: el stage final **no hereda nada** de los intermedios salvo lo que copies con `COPY --from`.

```dockerfile
# ===== Stage 1: BUILD =====
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

# ===== Stage 2: RUNTIME =====
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

Resultado: imagen final de **~10 MB** (vs ~800 MB del builder), sin shell, sin package manager, sin compiladores. Ejemplo equivalente en Python (compilar wheels nativas en builder, instalar en runtime):

```dockerfile
FROM python:3.12-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /wheels
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY . .
USER 1000:1000
CMD ["python", "main.py"]
```

Trucos avanzados:
- **Copiar entre stages con rutas dinámicas**: `COPY --from=builder /src/version.txt` también sirve para extraer artefactos.
- **Un stage como base de otro stage intermedio** (`FROM builder AS test`) para tests dentro del build.
- **`--target`** para construir hasta donde quieras (build de debug, stage de tests en CI):

```bash
docker build --target builder -t miapp:dev .    # imagen con herramientas, para debug
docker run -it --rm --entrypoint sh miapp:dev  # shell con gcc disponible
```

## BuildKit: el builder moderno (default desde Docker 23)

BuildKit trae paralelismo, caché mejorada, mounts efímeros y output selectivo. Con `docker buildx` se gestionan builders (incl. multi-arch con QEMU).

```dockerfile
# syntax=docker/dockerfile:1.7     ← pinnea la versión de features del Dockerfile
```

| Mount | Qué hace | Ejemplo |
|-------|----------|---------|
| `--mount=type=cache,target=…` | Dir persistente **entre builds** (no va a la capa) | pip, npm, go modules, apt |
| `--mount=type=secret,id=…` | Archivo montado solo durante el `RUN`, jamás en la imagen | tokens, `.npmrc`, llaves |
| `--mount=type=bind,source=…` | Lee del contexto sin copiarlo a la capa | ficheros de build grandes |
| `--mount=type=tmpfs` | RAM efímera durante el RUN | builds que necesitan /tmp grande |

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
RUN --mount=type=cache,target=/var/cache/apt --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
# Secrets: el archivo nunca se copia, nunca queda en capa ni en history
docker build --secret id=npmrc,src=.npmrc -t miapp:1.0 .
# SSH forwarding para clonar repos privados
docker build --ssh default -t miapp:1.0 .   # con RUN --mount=type=ssh git clone git@…
```

Comandos útiles de buildx:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t miapp:1.0 --push .   # multi-arch (requiere --push o --output)
docker buildx build --progress=plain --no-cache -t miapp:1.0 .                 # log línea a línea (debug de cache)
docker buildx build -t miapp:1.0 --output type=docker,dest=- . | gzip > img.tar # exportar sin registrar
```

## Comandos de build (CLI completa)

```bash
docker build -t miapp:1.0 .                                  # build desde contexto "."
docker build -f Dockerfile.prod -t miapp:1.0 .              # Dockerfile alternativo
docker build -t miapp:1.0 -t miapp:latest .                 # múltiples tags
docker build --build-arg VERSION=1.0 -t miapp:1.0 .         # con args
docker build --no-cache -t miapp:1.0 .                      # ignorar caché
docker build --no-cache-filter runtime -t miapp:1.0 .       # invalidar solo un stage
docker build https://github.com/owner/repo.git#main         # build desde git (contexto remoto)
docker build 'https://github.com/owner/repo.git#main: subdir' # subdirectorio como contexto
docker history miapp:1.0                                    # qué instrucciones/capas pesan
docker history --no-trunc miapp:1.0                         # sin truncar (comandos completos)
docker image inspect miapp:1.0 --format '{{json .Config}}'  # config bakeada
docker builder prune -a                                     # limpiar caché de builds
```

## Optimización de tamaño — tabla de técnicas

| Técnica | Ahorro típico | Nota |
|---------|---------------|------|
| Multi-stage + distroless/scratch | 80–95 % | La nº1: el SDK no viaja a producción |
| `.dockerignore` | Build rápido; menos capa basura | Especialmente `node_modules`/`.git` |
| Una capa `RUN` con `&&` + limpieza | 10–40 % | `rm -rf /var/lib/apt/lists/*` **en la misma capa** (otra capa no devuelve espacio) |
| `--no-cache-dir` / `npm ci && npm cache clean` | Variable | Pip/npm dejan 200–500 MB de cache |
| `apk add --no-cache` | ~15 % Alpine | Equivale a update+install+rm en un paso |
| `COPY --chown/--chmod` | 1 capa | Evita `RUN chown/chmod` |
| Go: `-ldflags="-s -w"`, `CGO_ENABLED=0` | 30–50 % | Sin symbols ni libc |
| Node: `npm ci --omit=dev` o pruned | 40 % | DevDependencies fuera |
| Squash/`--squash` | Colapsa capas | Antipatrón general: pierdes reutilización de capas |

> [!warning] Limpieza en la MISMA capa
> `RUN apt-get install …` (capa A) + `RUN rm -rf /var/lib/apt/lists/*` (capa B) **no libera nada**: el filesystem de la capa A ya contiene los archivos. Los `rm` deben ir en el mismo `RUN` que la instalación.

## Builds deterministas y buenas prácticas de release

- **Pinneá todo**: `FROM` por tag exacto (o digest), `pip install pkg==1.2.3` vía lockfile (`requirements.txt` con hashes o `poetry.lock`/`uv.lock`), `npm ci` con `package-lock.json`.
- **No dependas de la red cuando puedas evitarlo**: cache mounts + lockfiles; mirrors oficiales.
- **Versioná la syntax**: `# syntax=docker/dockerfile:1.7` para que todos los builders (CI, tu máquina, otra persona) usen las mismas features.
- **Etiquetá con la versión real**: `--build-arg VERSION` + `LABEL` OCI; el tag de imagen debe salir del CI, no del developer (ver [[Docker - Docker Compose]] y pipelines en [[Docker - Buenas Prácticas y Seguridad]]).
- **CI**: `docker buildx build --push` con tag = `git sha`, scan post-build (Trivy/Docker Scout), y cache remoto (`--cache-to/--cache-from type=gha`) para builds rápidos en CI.

## Catálogo de errores comunes

| Error / síntoma | Causa | Fix |
|-----------------|-------|-----|
| `COPY failed: file not found in build context` | Ruta relativa al contexto, no al Dockerfile; o path en `.dockerignore` | Revisá rutas y `.dockerignore` (`**/patrón`) |
| `failed to compute cache key: … not found` | `COPY requirements.txt` sin el archivo en el contexto | `.dockerignore` excluyó algo o path mal |
| `Unable to locate package` | `RUN apt-get update` cacheado viejo, o capa separada | `update && install` en un mismo `RUN` |
| `Cannot connect to the Docker daemon… build` | CLI apunta a otro context/host | `docker context ls` / `docker context use default` |
| `exec format error` al correr | Imagen de otra plataforma (arm64 vs amd64) | `FROM --platform=linux/amd64 …` o build multi-arch |
| `no matching manifest for linux/arm64` | Imagen single-arch | `--platform linux/amd64` (con emulación) o build propio |
| Container no responde a Ctrl+C / `docker stop` tarda 10 s | CMD en forma shell → PID 1 es `sh` sin señales | Forma exec `["cmd","arg"]` + `exec "$@"` |
| `npm ERR! lockfile` o builds distintos entre máquinas | `npm install` en vez de `npm ci` | Lockfile commiteado + `npm ci` |
| Build lento con `COPY . .` enorme | Contexto con `.git`/`node_modules`/docs | `.dockerignore` |
| Cambió una línea y rebuildó todo | Instrucción volatile arriba (fecha, ARG usado, `ENV` de version) | Ordenar de menos a más cambiante; sacar `date` del Dockerfile |
| `pip install` falla con `gcc: not found` | Wheels con extensión nativa en slim | Builder stage con gcc o usar imágenes con wheels manylinux |

## Testing de imágenes (antes de publicar)

```bash
docker run --rm miapp:1.0 --help                 # arranca y responde
docker run --rm miapp:1.0 id                     # ¿corre como root? (debe decir uid!=0)
docker run --rm --entrypoint sh miapp:1.0 -c 'ls /'  # inspeccionar sin entrypoint
docker run -d --name t miapp:1.0 && sleep 10 \
  && docker inspect t --format '{{.State.Health.Status}}' && docker rm -f t
docker history miapp:1.0                         # ¿algún secreto colado en comandos?
docker scout quickview miapp:1.0                 # vulnerabilidades (o trivy, ver seguridad)
```

## Cheatsheet del Dockerfile

| Instrucción | Crea capa | Sobreescribible en `run` | Nota clave |
|-------------|-----------|--------------------------|------------|
| `FROM` | base | — | Pinneá tag o digest; `--platform` para cross-arch |
| `ARG` | no | build-arg | Solo build; no persiste en imagen |
| `ENV` | no | — | Persiste en runtime; ojo con secretos en history |
| `LABEL` | no | — | Metadatos OCI |
| `COPY` | **sí** | — | Preferida; `--chown/--chmod/--link/--from` |
| `ADD` | **sí** | — | Solo tar autoextract o URL conscientemente |
| `RUN` | **sí** | — | `update && install && rm` juntos; cache mounts |
| `CMD` | no | ✅ args la reemplazan | Forma exec |
| `ENTRYPOINT` | no | args se añaden | Exec form + `exec "$@"` |
| `WORKDIR` | no | `-w` | Crea dirs; persiste (no es `cd`) |
| `USER` | no | `-u` | Non-root antes de CMD (ver seguridad) |
| `EXPOSE` | no | `-p` publica | Solo documentación de metadatos |
| `VOLUME` | marcador | `-v` | Ojo: oculta capas posteriores en ese path |
| `HEALTHCHECK` | no | `--health-*` | `NONE` para desactivar |
| `STOPSIGNAL` | no | — | Señal de graceful stop |
| `SHELL` | no | — | Solo forma shell |
| `ONBUILD` | no | — | Triggers para imágenes hijas |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — namespaces, capas y el modelo mental detrás del build
- [[Docker - Gestión de Contenedores]] — ejecutar, inspeccionar y limitar lo que esta nota construye
- [[Docker - Redes y Volúmenes]] — qué exponer (`EXPOSE` vs `-p`) y dónde persistir datos
- [[Docker - Docker Compose]] — `build:` contextualizado dentro de stacks multi-servicio
- [[Docker - Buenas Prácticas y Seguridad]] — hadolint, secrets, distroless y escaneo de la imagen resultante
- [[Docker - Comandos de Referencia Rápida]] — CLI de build resumida
- [[Python - Fundamentos y Sintaxis]] — el lenguaje que típicamente empaquetás
- [[Node JS]] — ecosistema npm/ci que esta nota usa como ejemplo
