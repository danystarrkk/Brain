---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Docker Compose

**Docker Compose** define aplicaciones multi-contenedor en un solo archivo declarativo (`compose.yaml`) y las gestiona con un comando: `docker compose up`. Donde el CLI de [[Docker - Gestión de Contenedores]] describe *un* contenedor a la vez, Compose describe **un sistema**: servicios, sus redes, sus volúmenes, sus dependencias y sus healthchecks — todo versionable en el repo. Es la herramienta estándar para desarrollo local, laboratorios (ver [[Docker - Laboratorios y Vulhub]]) y despliegues simples de una máquina.

> [!info] Modelo mental
> Compose traduce el YAML a una o varias llamadas al daemon de Docker y mantiene el estado bajo un *proyecto* (nombre por defecto = directorio; todos los recursos llevan el prefijo `<proyecto>_`). No hay orchestrador: Compose configura y ya — los healthchecks y restart policies los ejecuta Docker, no Compose. Por eso `docker compose down` + `up` es atómico y barato.

## Versiones: v1 vs v2 (y por qué ya no hay `version:`)

| | `docker-compose` (v1) | `docker compose` (v2) |
|--|----------------------|------------------------|
| Lenguaje | Python (externo) | Go (plugin del CLI, `docker-compose-plugin`) |
| Comando | **con guion** | **sin guion** |
| Estado | **EOL desde 2023** | Actual |
| Extras | — | `watch`, `--dry-run`, dependencias `condition:`, `profiles`, buildx integrado |

> [!warning] El `version:` del YAML está obsoleto
> En Compose v2 (spec actual) el atributo `version:` se ignora y genera warning (`the attribute 'version' is obsolete`). Los archivos viejos **siguen funcionando**, pero quitalo. También se abandonó el nombre `docker-compose.yml` obligatorio: el default ahora es `compose.yaml` (acepta `docker-compose.yaml/yml` si existen).

Migración v1 → v2 lo que realmente importa:

| Cambio | v1 | v2 |
|--------|----|----|
| Comando | `docker-compose up` | `docker compose up` |
| `version:` | requerido | obsoleto (quitar) |
| `depends_on` condicional | solo en file format 2.x | `condition: service_healthy` ✅ |
| Límites | `mem_limit`, `cpu_quota` | `deploy.resources.limits` |
| Extend | `extends:` (v1) | anclas YAML (`<<: *x`) |
| Env | `env_file` listas iguales | igual, más interpolación |

## Anatomía de `compose.yaml`

```yaml
name: miapp                     # nombre del proyecto (o COMPOSE_PROJECT_NAME)

services:                       # ← los contenedores (obligatorio)
  web: { … }                    # build o image + todo el runtime
  api: { … }
  db:  { … }

networks:                       # ← redes declaradas (opcional; si no, una default bridge por proyecto)
  edge:   {}
  backend: { internal: false }
  data:    { internal: true }

volumes:                        # ← volúmenes nombrados (opcional)
  pgdata: {}

configs:                        # archivos pequeños montados (Swarm; en compose simple ~ bind)
  app_config: { file: ./app.yaml }

secrets:                        # secretos: SOLO Swarm los monta como archivo; en compose simple son declarativos
  db_password: { file: ./secrets/pg.pass }

x-anchored: &anchored           # x-*: claves extendidas = anclas YAML reutilizables
  restart: unless-stopped
```

### `build` vs `image` (o ambos)

```yaml
services:
  api:
    image: ghcr.io/owner/api:${VERSION:-latest}   # nombre con el que se etiqueta
    build:                                        # cómo construirlo (opcional)
      context: ./api                              # build context (obligatorio si se usa build)
      dockerfile: Dockerfile.prod
      args: { VERSION: "${VERSION:-dev}" }        # → ARG del Dockerfile
      target: runtime                             # stage de multi-stage
      cache_from: ["ghcr.io/owner/api:cache"]
      platforms: ["linux/amd64"]                  # si se empuja multi-arch
```

- `image` + `build` juntos: Compose construye y **etiqueta con ese nombre**. Es el patrón CI correcto.
- Sin `image`, el nombre es `<proyecto>-<servicio>`.
- `docker compose up --build` fuerza rebuild; `docker compose build` solo construye.

### Claves de runtime (equivalencias con el CLI)

```yaml
services:
  api:
    container_name: api-fijo   # ⚠️ ver caveats abajo
    hostname: api
    command: ["npm", "start"]         # override de CMD
    entrypoint: ["/entrypoint.sh"]    # override de ENTRYPOINT
    working_dir: /app
    user: "1000:1000"
    ports:
      - "8080:80"                     # host:contenedor
      - "127.0.0.1:8443:443"          # bind a loopback
      - target: 9090                  # forma larga (publish/protocol)
        published: 9090
        protocol: tcp
        mode: host                    # Swarm: publicación en cada nodo
    expose: ["80"]                    # solo interno/documentación
    environment:                      # ver tabla de la sección siguiente
      DEBUG: "1"
    env_file: [.env]
    networks: [backend]
    volumes:
      - ./src:/app/src:ro             # bind
      - pgdata:/var/lib/postgresql/data   # named volume (declarado abajo)
      - type: tmpfs
        target: /tmp
        tmpfs: { size: 64m }
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:                           # límites (compose v2 los aplica aunque no sea Swarm)
      resources:
        limits:   { cpus: "1.0", memory: 512M }
        reservations: { cpus: "0.25", memory: 256M }
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    read_only: true
    tmpfs: ["/run"]
    ulimits: { nofile: { soft: 65535, hard: 65535 } }
    profiles: ["backend"]             # no arranca salvo con --profile
```

## Manejo de variables de entorno — las tres formas

| Forma | Sintaxis | Semántica | Uso |
|-------|----------|-----------|-----|
| **Map** | `environment: { K: V }` o `K: V` | Valor literal; las no-string (números/bools) Compose las convierte | Pocos valores explícitos |
| **List** | `environment: [K=V]` | Igual | Estilo sh |
| **Hereda del host** | `- K` (sin `=`) | Toma el valor del shell del que ejecutó `compose` | CI: inyectar `VERSION` |
| **env_file** | `env_file: [.env.prod]` | Archivo completo de K=V | Muchas variables / secretos de dev |
| **Interpolación** | `${VAR:-default}`, `${VAR:?err}` | Sustitución **en el YAML** desde shell/`.env` del proyecto | Parametrizar imágenes/puertos |

```yaml
services:
  api:
    env_file: [.env.local]        # primero
    environment:                  # encima mapea/overridea
      APP_ENV: production
      DB_HOST: db                 # ← nombre del SERVICIO, resuelto por el DNS de Compose
```

> [!info] El `.env` del proyecto ≠ env_file
> El archivo `.env` en el mismo directorio que `compose.yaml` **no** se inyecta a los servicios: Compose lo usa solo para **interpolación** del YAML (`${DB_PASSWORD}`). Para que llegue al contenedor hace falta `environment: [DB_PASSWORD=…]` o `env_file:`. Distinción que causa el bug "la variable llega al compose pero no a la app".

## Redes y volúmenes declarativos

```yaml
services:
  api: { networks: [backend, edge], volumes: [pgdata:/data] }

networks:
  edge:    {}                                   # bridge por defecto
  backend: { internal: false }
  data:    { internal: true }                   # sin salida a internet
  legacy:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
          gateway: 172.30.0.1

volumes:
  pgdata: {}
  esdata:
    driver: local
    driver_opts: { type: nfs, o: "addr=10.0.0.5,rw", device: ":/exports/es" }
```

- Cada proyecto crea su red `default`; los servicios se resuelven por **nombre de servicio** (DNS embebido, ver [[Docker - Redes y Volúmenes]]). Dos compose projects **no** se ven entre sí salvo explícitamente.
- `internal: true` es la micro-firewall de Compose: DB/caché sin ruta a internet.
- El modelo completo de red y montajes: [[Docker - Redes y Volúmenes]].

## Orquestación: dependencias, salud, escalado

### `depends_on` con `condition` — esperar *listo*, no *iniciado*

```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 10
  api:
    depends_on:
      db: { condition: service_healthy }         # ← espera a que responda pg_isready
      cache: { condition: service_started }
      migrator: { condition: service_completed_successfully }   # job que termina OK
```

| Condición | Espera a |
|-----------|----------|
| `service_started` (default) | El proceso arrancó (**no** que esté listo — el bug clásico) |
| `service_healthy` | Healthcheck en `healthy` |
| `service_completed_successfully` | Otro servicio **terminó con exit 0** (migraciones, jobs) |

- Compose **espera antes de arrancar** el dependiente; pero si el dependiente se cae *después*, no reinicia al dependiente. Para resiliencia real: `restart: unless-stopped` + la app reintente conexiones.
- Alternativa clásica (evitable hoy): `entrypoint: ["./wait-for-it.sh", "db:5432", "--"]`.

### Escalado y `container_name`

```bash
docker compose up -d --scale api=3     # 3 réplicas de api
docker compose up -d --scale api=3 web=1
```

- ❌ **`container_name` rompe el escalado** (dos réplicas no pueden compartir nombre). Además rompe el prefijo de proyecto: dos clones del repo en el mismo host chocan, y `up` con el mismo nombre **reutiliza** el contenedor viejo ignorando cambios de config.
- Regla: `container_name` solo si algo externo lo exige (monitoreo legacy, licenses); si no, los nombres `<proyecto>-<servicio>-1` son suficientes y escalan.

### `run` vs `up` vs `exec` — comandos one-off

| | Cuándo | Crea contenedor | Red | Puertos | Deps |
|--|--------|-----------------|-----|---------|------|
| `compose up` | servicio normal | sí (o reutiliza) | ✅ | publica | levanta deps |
| `compose run SVC cmd` | tarea puntual (migración, script, `pytest`) | **nuevo, temporal** (con `--rm`) | ✅ (misma red) | ❌ no publica | arranca deps si faltan |
| `compose exec SVC cmd` | sobre el **corriendo** | no | — | — | — |

```bash
docker compose run --rm api python manage.py migrate     # el patrón de migraciones
docker compose run --rm --no-deps api python -m pytest   # sin levantar db/etc (tests con mocks)
docker compose exec api sh                               # shell en el servicio vivo
```

> `compose run` **no publica puertos** (por eso una app de test no choca con la real) y su contenedor no aparece en `compose ps` con los mismos criterios — con `--rm` no deja rastro.

## Profiles: subconjuntos del stack

```yaml
services:
  web:    { image: nginx }
  api:    { image: api, profiles: [backend] }
  db:     { image: postgres, profiles: [backend] }
  adminer: { image: adminer, profiles: [tools] }
  seed:   { image: seeder, profiles: ["!backend"] }   # ⚠️ profile negado: solo si backend NO está activo
```

```bash
docker compose up -d                                   # solo web (los demás tienen profile)
docker compose --profile backend up -d                 # web + api + db
docker compose --profile '*' up -d                     # todos
COMPOSE_PROFILES=backend docker compose up -d          # vía variable de entorno
```

Patrones: entornos (`dev`/`debug`), herramientas (Adminer, mailcatcher) que no deben arrancar por defecto.

## Override files: `compose.yaml` + `compose.override.yaml`

Compose carga automáticamente **`compose.yaml` + `compose.override.yaml`** si existen, fusionándolos. Para otros entornos, `-f` explícito (⚠️ con `-f` se anula la carga automática):

```bash
docker compose up -d                                    # base + override local
docker compose -f compose.yaml -f compose.prod.yml up -d
export COMPOSE_FILE=compose.yaml:compose.prod.yml       # lo mismo sin repetir -f
```

**Reglas de fusión** (lo que salva o arruina un merge):

| Tipo de valor | Regla |
|---------------|-------|
| Escalares (puerto, image) | Sobrescrito por el último archivo |
| Escalares *clave* de Compose | Último archivo gana |
| Listas (`ports`, `networks`) | **Append** (⚠️ `-p 80:80` + `-p 8080:80` = intenta ambas → error de puerto) |
| Mapas (`environment`, `labels`) | Merge por clave; valor del último |
| `depends_on` | Merge por servicio; condición sobrescrita |

```yaml
# compose.override.yaml (dev local, no commitear — está en .gitignore)
services:
  api:
    build: { context: . }
    environment: { DEBUG: "1" }
    volumes: ["./src:/app/src"]          # hot-reload con ver [[ReactJS - Vite]] / [[Node JS]]
    ports: ["3000:3000"]
```

Para *sacar* algo que el base lista (p.ej. quitar puertos en prod), hay que reimplementar la clave — el merge de listas no resta. Es una de las razones por las que conviene mínimo-en-base, aditivo-en-override.

## Compose watch (sync en desarrollo)

```yaml
services:
  api:
    build: .
    develop:
      watch:
        - action: sync                    # copia cambios al contenedor
          path: ./src
          target: /app/src
        - action: rebuild                 # rebuild completo ante cambios
          path: ./package.json
        - action: sync+restart            # sync y reinicia el proceso
          path: ./src
          target: /app/src
```

```bash
docker compose up --watch      # en vez de -d: queda en foreground con el watch activo
```

Para apps con hot-reload nativo (Vite/Webpack, `nodemon`), mejor `sync`; para binarios compilados, `rebuild`. Ver [[ReactJS - Vite]] y [[Node JS]] para los flujos de dev.

## Ejemplo real completo: web + api + db + redis

```yaml
name: shop

services:
  web:
    image: nginx:1.27-alpine
    depends_on:
      api: { condition: service_healthy }
    ports: ["80:80"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks: [edge]
    restart: unless-stopped
    logging: { driver: json-file, options: { max-size: "10m", max-file: "3" } }

  api:
    build: { context: ./api }
    image: shop-api:${VERSION:-dev}
    environment:
      DATABASE_URL: postgres://shop:${DB_PASSWORD}@db:5432/shop
      REDIS_URL: redis://cache:6379/0
    env_file: [.env]
    depends_on:
      db:        { condition: service_healthy }
      cache:     { condition: service_started }
      migrate:   { condition: service_completed_successfully }
    networks: [edge, backend]
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      resources: { limits: { cpus: "1.0", memory: 512M } }
    restart: unless-stopped
    security_opt: ["no-new-privileges:true"]
    cap_drop: ["ALL"]
    read_only: true
    tmpfs: ["/tmp"]

  migrate:
    build: { context: ./api }
    image: shop-api:${VERSION:-dev}
    command: ["python", "manage.py", "migrate", "--noinput"]
    environment: { DATABASE_URL: postgres://shop:${DB_PASSWORD}@db:5432/shop }
    env_file: [.env]
    depends_on: { db: { condition: service_healthy } }
    networks: [backend]
    restart: "no"                         # ← job: termina y ya

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD requerido}
      POSTGRES_DB: shop
    volumes: [pgdata:/var/lib/postgresql/data]
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped
    deploy: { resources: { limits: { cpus: "2.0", memory: 1G } } }

  cache:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
    volumes: [redisdata:/data]
    networks: [backend]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 5
    restart: unless-stopped

networks:
  edge: {}
  backend: { internal: true }        # la DB y el cache no necesitan internet

volumes:
  pgdata: {}
  redisdata: {}
```

Leer este stack como diagrama: `web` solo expone el puerto; `api` es el único con doble red; `db`/`cache` quedan aislados en `backend`; el `migrate` es un job (restart `no`) que Compose espera con `service_completed_successfully`. La contraparte de seguridad endurecida de este archivo: [[Docker - Buenas Prácticas y Seguridad]].

## Comandos esenciales

```bash
docker compose up -d                     # levantar (reutiliza si existe)
docker compose up -d --build --force-recreate
docker compose ps                        # servicios activos (ps -a: todos)
docker compose logs -f --tail 200 api
docker compose pull / build / push
docker compose restart api               # rápido (no recrea)
docker compose up -d --force-recreate db # recrear con config nueva
docker compose down                      # para y borra contenedores + red del proyecto
docker compose down -v                   # + volúmenes (reset total de datos)
docker compose down --remove-orphans     # + contenedores de servicios borrados del YAML
docker compose down --rmi all            # + imágenes del proyecto
docker compose config                    # YAML resuelto tras merge/interpolación (¡debuggear!)
docker compose config --services / --volumes / --profiles
docker compose up --dry-run              # simulacro
docker compose events --json             # stream de eventos
docker compose top                       # procesos de los servicios
```

> [!tip] `docker compose config` es tu mejor amigo de debug
> Ante cualquier "¿por qué ve esto Compose?": `config` muestra el resultado **final** de merge + interpolación + defaults. 80 % de los misterios de YAML mueren ahí.

## Compose en CI/CD

```yaml
# patrón GitHub Actions mínimo
- run: docker compose -f compose.yaml -f compose.ci.yml up -d --build --wait
- run: docker compose exec api pytest -q
- run: docker compose down -v
```

- `--wait` espera a que **todos** los servicios con healthcheck estén `healthy` (perfecto para CI).
- `COMPOSE_FILE`/`-f` para el overlay de CI (sin puertos, con servicios fake).
- Caché: BuildKit ya cachea; en Actions, `type=gha` cache (ver [[Docker - Dockerfile y Construcción de Imágenes]]).
- Versioná siempre el YAML en el repo: el archivo **es** el entorno.

## Errores comunes

1. **`dependency failed to start: container … is unhealthy`**: el healthcheck falla (comando inexistente en la imagen, puerto mal, `start_period` corto). `docker compose logs db` + `docker compose ps` muestra `(unhealthy)`.
2. **`version` obsolete**: warning; quitá el `version:`.
3. **Variable vacía en runtime** ("llega al compose pero no a la app"): confundiste interpolación `${VAR}` con `environment`/`env_file` — sección de variables.
4. **`port is already allocated` tras un override**: las listas de `ports` se **appendan** al fusionar; revisá `docker compose config`.
5. **`container name X is already in use` / cambios ignorados**: `container_name` fijo reutilizado; quitá la clave o `--force-recreate`.
6. **`no such service` con perfiles**: el servicio está tras un profile; `--profile` o `COMPOSE_PROFILES`.
7. **Dos proyectos chocando**: mismo nombre de proyecto en el mismo host → nombres/puertos iguales. `name:` explícito o directorios distintos.
8. **`down` no borra los datos**: por diseño; `-v` escribe destrucción de volúmenes — en prod, nunca automático.
9. **Builds viejos "fantasma"**: `up` no rebuilda sin cambios de declaracion; `--build` o `build --no-cache`.
10. **`docker compose` vs `docker-compose`**: el binario con guion es el v1 EOL; instalá el plugin.

## Cheatsheet Compose

| Tarea | Comando |
|-------|---------|
| Levantar stack | `docker compose up -d` |
| Levantar y esperar salud | `docker compose up -d --wait` |
| Ver config resuelta | `docker compose config` |
| Logs de un servicio | `docker compose logs -f svc` |
| Shell en servicio | `docker compose exec svc sh` |
| Tarea one-off | `docker compose run --rm svc cmd` |
| Escalar | `docker compose up -d --scale worker=3` |
| Profile | `docker compose --profile X up -d` |
| Overrides prod | `-f compose.yaml -f compose.prod.yml` |
| Reset total datos | `docker compose down -v --remove-orphans` |
| Solo construir | `docker compose build svc` |
| Ver eventos | `docker compose events` |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — qué ejecuta Compose por debajo (contenedores, capas, red embebida)
- [[Docker - Gestión de Contenedores]] — el CLI del que Compose es una capa declarativa
- [[Docker - Redes y Volúmenes]] — semántica de `networks:`/`volumes:` y DNS entre servicios
- [[Docker - Dockerfile y Construcción de Imágenes]] — lo que `build:` invoca, cache y multi-stage
- [[Docker - Buenas Prácticas y Seguridad]] — endurecer cada servicio (limits, cap_drop, secrets)
- [[Docker - Laboratorios y Vulhub]] — Compose como formato estándar de stacks de práctica
- [[Docker - Comandos de Referencia Rápida]] — cheatsheet CLI + Compose en una página
- [[ReactJS - Vite]] / [[Node JS]] — hot-reload y watch en desarrollo con Compose
