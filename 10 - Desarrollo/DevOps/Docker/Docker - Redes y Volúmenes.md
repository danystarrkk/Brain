---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Redes y Volúmenes

Dos problemas separados que esta nota resuelve en profundidad: **cómo conectan los contenedores** (su propio network namespace, bridge, DNS embebido, publicación de puertos) y **cómo persisten los datos** más allá de su vida efímera (volumen nombrado, bind mount, tmpfs). Son los dos temas donde un detalle mal entendido (el DNS de 127.0.0.11, el "localhost" de cada contenedor, el copy-on-write) causa el 90 % de los bugs de "no conecta" y "se borraron los datos".

> [!info] El principio de todo
> Cada contenedor tiene su **propio network namespace**: interfaces, rutas, tablas de firewall y puertos propios. Su `127.0.0.1` es **suyo y solo suyo** — nada escuchando en el host ni en otros contenedores es "localhost" desde dentro. Todo lo demás (bridge, DNS, `-p`) son mecanismos para conectar ese namespace de forma controlada.

## Drivers de red: cuál usar y por qué

| Driver | Alcance | Aislamiento | Cuándo usarlo | Notas |
|--------|---------|-------------|---------------|-------|
| **bridge** (default) | Un host | Red privada con NAT saliente | **El caso estándar**: servicios web/API/DB juntos | DNS embebido solo en redes *user-defined* |
| **bridge (default `bridge`)** | Un host | Red pública interna | Legacy | Sin DNS por nombre; no la uses |
| **host** | Comparte la red del host | **Cero** aislamiento de red | Máximo rendimiento (benchmarks, IDS) | Linux-only; sin `-p` (no hay traducción); el servicio escucha en el host |
| **none** | Ninguna | Aislamiento total de red | Labs de seguridad, builds que no deben salir, CI paranoica | Solo `lo` interno |
| **macvlan** | Asigna MAC propia | Aparece como dispositivo físico en la LAN | Integrar contenedores en una red corporativa existente | El switch/router ve hosts "reales"; no funciona bien en WiFi/hotspot |
| **ipvlan** | Asigna IP propia, MAC compartida | L2/L3 | donde macvlan está bloqueado (muchos AP WiFi/proveedores) | Modo L3: sin broadcast |
| **overlay** | Multi-host (Swarm) | Red distribuida con cifrado opcional | Clústeres Swarm; (K8s usa sus propios CNI) | Requiere `docker swarm init`; KV store interno |
| **ipvlan L2 + VLAN** | IP propia + tag 802.1q | Segmentación por VLAN | Redes corporativas con VLANs | `-o parent=eth0.10` (subinterface) |
| **macvlan/ipvlan** | — | — | — | Ambos incompatibles con WiFi/proveedores que filtran MACs |

```bash
docker network ls                          # listar (NAME, DRIVER, SCOPE)
docker network create mi-red               # bridge user-defined
docker network create -d bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1 \
    --opt com.docker.network.bridge.enable_icc=true mi-red
docker network inspect mi-red              # contenedores conectados, IPs, options
docker network connect mi-red contenedor   # conectar existente
docker network disconnect mi-red contenedor
docker network rm mi-red
docker network prune                       # borrar no usadas
```

### Red bridge user-defined vs `bridge` por defecto

La diferencia es **el DNS embebido** (y el ICC controlable):

```bash
docker network create backend
docker run -d --name db --network backend postgres:17-alpine
docker run -d --name api --network backend miapi:1.0

docker exec api getent hosts db
# 172.22.0.2    db            ← resuelto por el DNS embebido 127.0.0.11

docker exec api sh -c 'cat /etc/resolv.conf'
# nameserver 127.0.0.11
# options ndots:0
```

En la red `bridge` por defecto **no hay resolución por nombre** (solo IPs, o legacy `--link` que está deprecado). Regla: siempre redes user-defined; es gratis.

### DNS y service discovery — cómo funciona de verdad

| Nombre | ¿Se resuelve? | Ejemplo |
|--------|----------------|---------|
| Nombre del contenedor | ✅ en su(s) red(es) user-defined | `db` → 172.22.0.2 |
| Alias (`--network-alias`) | ✅ | `database` → mismo contenedor |
| FQDN interno `<nombre>.<red>` | ✅ | `db.backend` (suficiente en la red default) |
| Servicio de Compose | ✅ | `api`, `db` (nombre del servicio) |
| `host.docker.internal` | ✅ (con `--add-host …:host-gateway` o Compose `extra_hosts`) | El host desde el contenedor |
| Cualquier nombre externo | Vía DNS del daemon (forwards a los del host, default 8.8.8.8 si el host no tiene) | `google.com` |

- El resolver 127.0.0.11 resuelve **solo en las redes del contenedor**; otros contenedores de otras redes son invisibles (por diseño).
- `options ndots:0` evita búsquedas DNS locas con nombres cortos.
- Descubrimiento de servicios real (round-robin, health) está **fuera** del alcance de Docker: eso es Compose/ Swarm/K8s. Ver [[Docker - Docker Compose]].

## Publicación de puertos: `-p`, cómo viaja el tráfico

```bash
docker run -d -p 8080:80 web            # cualquier interfaz del host:8080 → contenedor:80 (TCP)
docker run -d -p 127.0.0.1:8080:80 web  # solo loopback del host
docker run -d -p 80 web                 # puerto aleatorio en el host (mirar con docker port)
docker run -d -p 53:53/udp dns          # UDP
docker run -d -p 8000-8010:8000-8010 api  # rango
docker run -d -P web                    # publica TODOS los puertos de EXPOSE en puertos aleatorios
docker port web                         # ver mapeos resueltos
```

Mecanismo (lo que pasa por debajo, en el host):

1. El daemon **abre el puerto** en el host (por defecto en `0.0.0.0`, salvo `bind` explícito).
2. El tráfico se redirige vía **iptables/nftables (DNAT)** al IP del contenedor en su bridge (o vía *userland proxy* como fallback, p.ej. en macOS/Windows donde no hay iptables).
3. El contenedor ve la conexión "normalmente", desde la IP del bridge.

> [!warning] `localhost` no cruza el borde
> `-p 8080:80` publica en el host; un cliente del host usa `localhost:8080` (o la IP de LAN). Pero **dentro del contenedor**, su servicio está en `:80`, no en `localhost:8080` — y su `localhost` no ve al host. Para que un contenedor llegue al host: `host.docker.internal` (ver tabla DNS). Por eso `-p 127.0.0.1:8080:80` es una micro-firewall: nada de la red externa llega.

`EXPOSE` (Dockerfile) no publica nada: solo metadatos/documentación que usa `-P`. Detalle en [[Docker - Dockerfile y Construcción de Imágenes#EXPOSE]].

## Recetas de red

### Patrón: tres redes (edge / backend / internal)

```yaml
# equivalente Compose — ver [[Docker - Docker Compose]]
networks:
  edge:    {}                                  # pública
  backend: { internal: false }                 # api ↔ db
  data:    { internal: true }                  # sin salida a internet (db, cache)
```

```bash
# Equivalente CLI
docker network create edge
docker network create backend
docker network create --internal data          # ← sin NAT saliente

docker run -d --name db  --network data   postgres:17-alpine
docker run -d --name api --network data --network backend miapi:1.0
docker run -d --name web --network edge -p 8080:80 nginx:alpine
```

Regla de oro: **cada contenedor solo en las redes que necesita**. La DB en red `internal` no puede ser escaneada desde internet aunque publiques mal algo.

### Aislamiento total para labs

```bash
docker network create --internal --driver bridge lab
docker run --network lab --network-alias victim vulnerable/app   # ni siquiera sale
```

Para laboratorios de seguridad esta configuración es obligatoria — ver [[Docker - Laboratorios y Vulhub]].

### Debugging de conectividad

```bash
docker exec api getent hosts db              # ¿resuelve?
docker exec api sh -c 'nc -zv db 5432'       # ¿alcanza el puerto?
docker exec api sh -c 'curl -sv http://db:5432 --max-time 3'
docker network inspect backend --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
nslookup db 127.0.0.11                       # preguntar DIRECTO al resolver embebido
iptables -t nat -L DOCKER -n                 # ver reglas DNAT (como root, en el host)
```

La mitad de los problemas: el cliente habla `localhost` cuando debería hablar el nombre; la otra mitad: el servicio escucha en `127.0.0.1` en vez de `0.0.0.0` **dentro** del contenedor.

## Volúmenes: persistencia más allá del contenedor

Recordá de [[Docker - Fundamentos y Conceptos]]: la capa writable del contenedor muere con él. Los volúmenes son objetos de primera clase gestionados por Docker (viven en `/var/lib/docker/volumes/`).

### Comparativa de los tres mecanismos de montaje

| | **Named volume** | **Bind mount** | **Tmpfs** | **Anonymous volume** |
|--|------------------|----------------|-----------|----------------------|
| Sintaxis | `-v datos:/var/lib/pg` | `-v /host/ruta:/app` | `--tmpfs /tmp` | `-v /var/lib/app` |
| Quién lo crea | Docker (`volume create` o implícito) | Debe existir en el host | Kernel (RAM) | Docker (ID aleatorio) |
| Ubicación | `/var/lib/docker/volumes/<n>/` | Ruta arbitraria del host | memoria | igual que named |
| Persistencia | ✅ hasta `volume rm` / prune | ✅ según el host | ❌ muere con el contenedor | ⚠️ queda "huérfano" al borrar el contenedor |
| Moverlo de host | `docker cp`/tar | es el host ya | — | difícil de localizar |
| Control de versiones | no | ✅ (puede ser el repo) | — | no |
| Rendimiento | nativo overlay2 | **passthrough total** | el más rápido (RAM) | nativo |
| Ideal para | **datos de DB, estado de app** | código en dev, configs, certs | secretos, cache, swap | nunca a propósito |
| SELinux (RHEL) | `:z`/`:Z` | `:z`/`:Z` | — | — |

> [!warning] Anonymous volumes son deudas
> `-v /app/node_modules` sin nombre crea un volumen opaco que **sobrevive** a `docker rm` (hasta `docker volume prune`) y no podés saber a qué contenedor pertenecía. Si necesitás uno, usá `--mount type=volume,src=nombre,dst=/ruta`.

### Named volumes

```bash
docker volume create pgdata
docker volume create --driver local \
    --opt type=none --opt o=bind --opt device=/mnt/nvme/pg pgdata  # ← volumen apuntando a un disco concreto
docker volume inspect pgdata          # driver, opts, mountpoint
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:17-alpine
docker volume ls -f dangling=true     # huérfanos (ya sin contenedores)
docker volume rm pgdata               # solo si no está en uso (o -f)
docker volume prune                   # todos los huérfanos
```

### Bind mounts — y sus trampas de permisos

```bash
docker run -v $(pwd)/src:/app/src miapp:dev               # hot-reload típico
docker run -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro nginx   # config read-only
docker run --mount type=bind,src=/data,dst=/data,readonly miapp      # forma explícita (falla si /data no existe)
```

- **Permisos**: el contenedor escribe con el UID real del proceso. Si la app corre como UID 1000 y el host tiene `root:root 700` en el directorio → `permission denied`. Fix: `chown`/`chmod` en el host, o alinear UIDs, o `--user` acorde. En bind mounts **no existe** `uid=/gid=` (eso solo funciona en volúmenes de tipo `volume` con opciones de driver).
- **`:ro`** en el montaje hace el path read-only *para el contenedor* — el host igual puede escribir.
- **Propagación** (`:rprivate` default, `:rshared`, `:rslave`): quién ve los mounts que el contenedor monta *después* de arrancar. Solo importa en hosts con mounts dinámicos (K8s, DinD): default `rprivate` (nada cruza) es lo correcto y lo seguro.

### Tmpfs — en RAM, invisible

```bash
docker run --tmpfs /run --tmpfs /tmp:rw,size=64m,mode=1777 miapp
```

No persiste, no aparece en `docker diff`, no escribe en disco. Perfecto para secretos volátiles, `/tmp` de bases de datos y caches. Rendimiento superior y cero riesgo de residuos en disco.

### Compartir datos entre contenedores

```bash
docker volume create shared-logs
docker run -d -v shared-logs:/var/log/app app1
docker run -d -v shared-logs:/var/log/app app2
docker run -d -v shared-logs:/logs:ro logproc   # lector
```

El patrón más simple para log-shareo: todos apuntan al mismo volumen nombrado. Y para datos que solo necesitan **compartirse efímeramente entre procesos del mismo servicio**, tmpfs (en RAM, sin residuos). Lo que NO sirve: esperar que dos contenedores compartan su capa writable — es imposible por diseño (cada uno tiene la suya, ver [[Docker - Fundamentos y Conceptos]]).

### `--volumes-from` (legacy)

`docker run --volumes-from db miapp` reutiliza los mounts de otro contenedor. Es **legacy**: prefieren volúmenes nombrados explícitos (auto-documentados, inspeccionables) o `depends_on` + declaraciones en Compose.

## Respaldo y migración de datos

```bash
# Backup de un named volume (la receta canónica)
docker run --rm -v pgdata:/data -v $(pwd):/backup alpine \
    tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .

# Restaurar (en volumen vacío)
docker run --rm -v pgdata:/data -v $(pwd):/backup alpine \
    tar xzf /backup/pgdata-2026-10-11.tar.gz -C /data
```

- Para DBs con integridad, el tar en frío es arriesgado: preferí el dump nativo (`pg_dump`, `mysqldump`) como parte del respaldo.
- `docker export` exporta el fs del **contenedor**, no el volumen: para migrar volúmenes entre hosts, tar como arriba, o `docker volume create` + copia.
- En labs, el ciclo `down -v` (destruir todo incluidos volúmenes) + `up` es el "reset" — ver [[Docker - Laboratorios y Vulhub]].

## Storage drivers: el suelo donde pisan los volúmenes

- El driver por defecto en hosts modernos es **overlay2** (o el *snapshotter* `overlayfs` de containerd; `docker info` → `Storage Driver`). Mecánica y copy-on-write en [[Docker - Fundamentos y Conceptos]].
- Los named volumes se crean como `local` driver sobre el mismo filesystem que usa Docker: **si `/var/lib/docker` está en disco pequeño, tus DBs están en disco pequeño**. En producción: disco dedicado/monitoreado para `/var/lib/docker`.
- `docker system df -v` muestra qué imagen consume qué y qué volúmenes existen; `docker volume inspect` da el mountpoint real.
- Cambiar de storage driver (btrfs/zfs) es operación de migración mayor: requiere vaciar el host.

## Errores comunes (redes y volúmenes)

1. **`getaddrinfo: Name or service not known` entre contenedores**: están en redes distintas o en la `bridge` default (sin DNS). `docker network connect backend c`, o crearlos en la misma red user-defined.
2. **`Connection refused` a pesar de resolverse**: el servicio escucha en `127.0.0.1` y no en `0.0.0.0` dentro del contenedor.
3. **`bind: address already in use` en `-p`**: puerto del host ocupado (otro contenedor o servicio del host); `ss -tlnp | grep 8080` (ver [[Comandos Básicos de Sistema y Red en Linux]]).
4. **`permission denied` escribiendo en bind mount**: mismatch de UIDs (sección bind mounts).
5. **Datos "desaparecieron" tras recrear el contenedor**: se escribieron en el fs del contenedor, no en un volumen. `docker diff` para el autopsia.
6. **`volume is in use` al borrar**: otro contenedor lo referencia; o `docker volume rm -f`.
7. **Contenedor "no tiene internet"**: red `--internal`, `--network none`, o DNS mal configurado (`--dns`).
8. **`--dns 127.0.0.1` usado fuera de un lab**: corta toda resolución externa; el DNS embebido no está en el host.
9. **Host no resuelve el nombre del contenedor**: el DNS de Docker es **solo entre contenedores**; desde el host usá la IP (`docker inspect`) o publicá el puerto.

## Cheatsheet — redes y volúmenes

| Tarea | Comando |
|-------|---------|
| Crear red con subnet | `docker network create -d bridge --subnet 172.22.0.0/16 mi-red` |
| Red sin internet | `docker network create --internal aislada` |
| Ver miembros de una red | `docker network inspect X --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'` |
| Publicar en loopback | `-p 127.0.0.1:8080:80` |
| Solo contenedores (sin host) | `--expose 80` + misma red (sin `-p`) |
| Crear volumen | `docker volume create datos` |
| Montar read-only | `-v datos:/d:ro` |
| En RAM | `--tmpfs /tmp:size=64m` |
| Backup volumen | `docker run --rm -v datos:/d -v $(pwd):/b alpine tar czf /b/b.tar.gz -C /d .` |
| Huérfanos | `docker volume ls -f dangling=true` / `docker volume prune` |
| Uso de disco | `docker system df -v` |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — el namespace de red y el union filesystem que sustentan todo
- [[Docker - Gestión de Contenedores]] — `-p`, `-v`, `--network` en contexto del ciclo de vida completo
- [[Docker - Docker Compose]] — redes y volúmenes declarativos, modelo de red por proyecto
- [[Docker - Buenas Prácticas y Seguridad]] — redes internas, `:ro`, tmpfs y least privilege
- [[Docker - Laboratorios y Vulhub]] — aislamiento de labs: `--internal`, `none`, caps
- [[Docker - Comandos de Referencia Rápida]] — comandos de red/volumen en cheatsheet
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `iptables`, `ip` para depurar el host
