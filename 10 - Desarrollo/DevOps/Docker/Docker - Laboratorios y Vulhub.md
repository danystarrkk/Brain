---
tags:
  - desarrollo
  - devops
  - docker
tipo: concepto
estado: terminado
---

# Docker - Laboratorios y Vulhub

Los contenedores son la mejor herramienta que tuvo un practicante de seguridad: **entornos vulnerables que se levantan en segundos, se destruyen sin miedo y se reinician idénticos mil veces**. Vulhub empaquetó cientos de CVEs famosas como stacks de Docker Compose; DVWA, Juice Shop y WebGoat son clásicos de práctica web. Esta nota cubre cómo usarlos **sin exponer nada ni quemar tu propio host**: aislamiento de red, límites de recursos, patrones de reset, y cómo portar o crear tus propios laboratorios. La contraparte teórica de todo lo que practicás aquí: [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]].

> [!info] Por qué Docker cambió la práctica de seguridad
> Antes: una VM de 4 GB por laboratorio, snapshots frágiles, "se me rompió el entorno". Hoy: `docker compose up -d` levanta un Apache 2.4.49 con CVE-2021-41773 en 20 s; `down -v` lo devuelve al estado cero. La disciplina que falta no es técnica — es **aislamiento**: un entorno vulnerable publicado en `0.0.0.0` en una red corporativa es un caballo de Troya. Las secciones de red y recursos de esta nota existen por eso.

## Reglas de higiene de un lab (las cinco de oro)

1. **Nunca `--network host`** y nunca publicar en `0.0.0.0` si no es estrictamente necesario: `-p 127.0.0.1:8080:80` (ver [[Docker - Redes y Volúmenes]]).
2. **Red interna por defecto**: `internal: true` o `--network none` cuando el lab no necesita internet (la mayoría no lo necesita si la imagen ya está descargada).
3. **Nunca `--privileged` ni montar el socket de Docker** en el contenedor vulnerable: un lab que necesita eso para "funcionar" es un lab mal diseñado — y esos son exactamente los vectores que practicás en [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]].
4. **Límites de recursos siempre**: `--memory --cpus --pids-limit`. Un DoS accidental no debe tumbar tu workstation.
5. **Reset idéntico**: `docker compose down -v` destruye datos; el estado inicial vive en imágenes/git, nunca en el contenedor (filosofía de inmutabilidad en [[Docker - Fundamentos y Conceptos]]).

```yaml
# plantilla lab segura (usable con cualquier stack)
services:
  vulnerable:
    build: ./victim
    networks: [lab]
    cap_drop: ["ALL"]
    cap_add: ["NET_BIND_SERVICE"]        # solo si bindea <1024
    security_opt: ["no-new-privileges:true"]
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 1G }
    # sin `ports:` — el atacante entra por la red del proyecto
  attacker:
    build: ./attacker
    networks: [lab]
    cap_add: ["NET_ADMIN", "NET_RAW"]    # necesario para sniffing/ARP dentro del lab
    profiles: ["attack"]                 # solo levanta con --profile attack
networks:
  lab:
    internal: true                       # sin salida a internet
```

## Vulhub: el catálogo CVE-dockerizado

[Vulhub](https://github.com/vulhub/vulhub) es un repositorio de **cientos** de entornos vulnerables, cada uno en su carpeta con `docker-compose.yml` + `Dockerfile` + README con el exploit. Estructura (fragmento representativo):

```
vulhub/
├── apache/CVE-2021-41773/         # path traversal + RCE (2.4.49)
├── docker/unauthorized-access/    # daemon TCP expuesto sin auth
├── docker/CVE-2019-5736/          # escape vía runc (imagen del lab)
├── elasticsearch/CVE-2014-3120/   # RCE vía scripting
├── jenkins/CVE-2018-1999002/
├── log4j/CVE-2021-44228/          # Log4Shell
├── mysql/CVE-2016-6662/
├── nginx/CVE-2017-7529/           # integer overflow → info leak
├── redis/CVE-2022-0543/           # escape del sandbox Lua
├── spring/CVE-2022-22965/         # Spring4Shell
├── struts2/CVE-2017-5638/         # Jakarta Multipart RCE
├── tomcat/CVE-2020-1938/          # Ghostcat
└── wordpress/…, ghostscript/…, wordpress/…
```

### Descarga selectiva (sin clonar 2 GB)

```bash
# Opción A: sparse-checkout (git moderno, recomendada)
git clone --depth 1 --filter=blob:none --sparse https://github.com/vulhub/vulhub.git
cd vulhub && git sparse-checkout set apache/CVE-2021-41773

# Opción B: export de una sola carpeta vía svn
svn export https://github.com/vulhub/vulhub/trunk/apache/CVE-2021-41773 lab-apache

# Explorar sin descargar
svn list https://github.com/vulhub/vulhub/trunk    # o navegar la web de GitHub
```

### Flujo estándar con cualquier lab de Vulhub

```bash
cd apache/CVE-2021-41773
cat docker-compose.yml && less README.md      # 1) SIEMPRE leer antes de up
docker compose up -d                           # 2) levantar
docker compose ps                              # 3) estado/puertos reales
curl -s http://127.0.0.1:8080 | head          # 4) verificar
# … práctica …
docker compose down -v                         # 5) reset completo (incluye volúmenes)
```

> [!warning] Los compose de Vulhub están pensados para VMs descartables
> Algunos publican en `0.0.0.0` y no traen límites. Antes del `up`: editar el `ports:` a `127.0.0.1:…` o añadir `deploy.resources.limits`, o envolverlo en tu compose overlay (ver sección de aislamiento). Nunca corras un lab de este tipo en un host con datos o exposición corporativa.

## Catálogo de imágenes vulnerables populares

| Lab | Imagen (Docker Hub) | Puerto | Tema | Nivel |
|-----|---------------------|--------|------|-------|
| DVWA | `vulnerables/web-dvwa` | 80 | SQLi, XSS, CSRF, upload | Principiante |
| bWAPP | `raesene/bwapp` | 80 | 100+ vulnerabilidades | Principiante |
| OWASP Juice Shop | `bkimminich/juice-shop` | 3000 | OWASP Top 10 moderna (Node) | Intermedio |
| WebGoat | `webgoat/webgoat` | 8080 | Aprendizaje guiado OWASP | Intermedio |
| Mutillidae | `citizenstig/nowasp` | 80 | OWASP Top 10 (PHP) | Principiante |
| WordPress legacy | `wordpress:4.9.8` (tag viejo) | 80 | Plugins/CMS vulnerables | Variable |
| Metasploitable2 | `tleemcjr/metasploitable2` | 21,22,80,445,3306… | Linux vulnerable completo | Explotación clásica |
| Vulnerhub Vulhub stacks | `vulhub/<servicio>:<versión>` | según CVE | CVE específica | Intermedio–Avanzado |

```bash
# Los clásicos en un comando (⚠️ con bind a loopback)
docker run -d --name dvwa -p 127.0.0.1:8080:80 vulnerables/web-dvwa
docker run -d --name juice -p 127.0.0.1:3000:3000 bkimminich/juice-shop
docker run -d --name webgoat -p 127.0.0.1:8080:8080 webgoat/webgoat
```

- Los nombres exactos de imágenes **cambian con el tiempo**: verificá siempre el repo oficial del proyecto (DVWA, Juice Shop, WebGoat tienen docs dockerizadas) antes de `pull`.
- Cada `pull` es código de terceros: en un host serio, escanéalas (Trivy) — ver [[Docker - Buenas Prácticas y Seguridad]] — y desconfiá de forks no oficiales.
- Para CTFs multi-equipo, la plataforma CTFd se dockeriza (`ctfd/ctfd` + MySQL), y existen repos de challenges dockerizados listos; montalos con la misma plantilla de aislamiento.

## Portar una app vulnerable a contenedor

Cualquier app que corra en un SO puede dockerizarse; para labs conviene **fijar la versión vulnerable explícitamente**:

```dockerfile
# Lab: Flask con RCE intencional (inyección de comandos en /ping)
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir flask==3.0.3
# En labs el non-root es opcional pero recomendable: practicarás escalada de todas formas
EXPOSE 5000
CMD ["python", "app.py"]
```
```python
# app.py — vulnerable A PROPÓSITO
from flask import Flask, request
import subprocess
app = Flask(__name__)

@app.route("/ping")
def ping():
    host = request.args.get("host", "127.0.0.1")     # sin sanitizar → command injection
    return f"<pre>{subprocess.check_output(f'ping -c 1 {host}', shell=True, text=True)}</pre>"

@app.route("/")
def index():
    return '<form action="/ping">host <input name="host"><input type="submit"></form>'

app.run(host="0.0.0.0", port=5000)
```

Puntos al portar apps reales antiguas:
- **Fijá versiones**: `pip install flask==0.12`, `npm install express@3` — la vulnerabilidad vive en la versión.
- **Evita dependencias de servicios externos caídos**: la mayoría de labs antiguos necesitan archivos que ya no se descargan; vendoréalos en el build context.
- **La imagen vieja del lab puede requerir repos muertos** (`apt-get update` falla en Debian EOL): mirror en `archive.debian.org` o build desde snapshot.
- PHP 5/7, Tomcat 7, Struts 2: usá las imágenes oficiales de la época (`tomcat:7`, `php:5.6-apache`) — son el lab perfecto sin escribir Dockerfile.

## Snapshot y reset: patrones

| Patrón | Comando | Cuándo |
|--------|---------|--------|
| Reset total | `docker compose down -v --rmi local --remove-orphans` | Entre sesiones / devolver a cero |
| Reset de datos, conservar imágenes | `down -v` | Rápido entre intentos |
| Snapshot del estado *actual* | `docker commit c lab/snapshot:punto1` | Guardar avance a mitad de CTF (lab efímero; no confundir con el anti-patrón de prod, ver [[Docker - Gestión de Contenedores]]) |
| Restaurar snapshot | `docker run -d --name c2 lab/snapshot:punto1` | Continuar desde ahí |
| Backup de evidencias | `docker cp c:/root/loot ./evidencias/` | Antes de cualquier `down` |
| Reinicio limpio | `docker compose restart` | Sin perder datos |

> [!tip] El estado de un lab vive en la definición
> Todo lo que no esté en el repo del lab (compose, Dockerfile, script de explotación, notas) **se pierde** en el próximo `down -v`. Tratá el lab como código: lo importante son tus writeups y scripts, no el contenedor. La práctica de documentar hallazgos es la que se lleva el CTF — ver los writeups de concepto en la sección de Seguridad del vault (20 - Seguridad).

## Laboratorio completo: víctima + atacante aislados

```yaml
# docker-compose.yml del lab
services:
  victim:
    build: ./victim                      # la app vulnerable
    networks: { lab: { aliases: [victim] } }
    deploy: { resources: { limits: { cpus: "0.5", memory: 512M } } }
  attacker:
    image: kalilinux/kali-rolling        # ver receta de tools abajo
    networks: [lab]
    cap_add: ["NET_ADMIN", "NET_RAW"]
    stdin_open: true
    tty: true
    profiles: ["attack"]
networks:
  lab: { internal: true }
```

```dockerfile
# attacker/Dockerfile: kali con lo mínimo del kit
FROM kalilinux/kali-rolling
RUN apt-get update && apt-get install -y --no-install-recommends \
        nmap netcat-openbsd curl python3 python3-pip sqlmap gobuster \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /lab
CMD ["sleep", "infinity"]
```

```bash
docker compose up -d                              # solo víctima
docker compose --profile attack up -d             # con atacante
docker compose exec attacker bash
# dentro del atacante:
nmap -sV victim
curl 'http://victim:5000/ping?host=127.0.0.1%3Bid'
```

El victim es inalcanzable desde fuera de la red `lab` (sin `ports:`, red interna) — práctica íntegra de red: [[Docker - Redes y Volúmenes]].

## Recursos y límites

Sin límites, un lab te puede tumbar la máquina (un cryptominer, un fork bomb, un `stress-ng` de una prueba mal dimensionada):

```bash
# CLI
docker run --memory 1g --cpus 1 --pids-limit 256 --name victim vulnerable/app

# Compose (como en la plantilla)
deploy: { resources: { limits: { cpus: "1.0", memory: 1G } } }
```

Y para que el lab tampoco envenene tu host: logging con `max-size`, `--restart: "no"` (labs no deben sobrevivir a reboots), y `docker compose down` como parte del ritual — semántica exacta de todo esto en [[Docker - Gestión de Contenedores]].

## Errores comunes de labs

1. **`Cannot connect to 127.0.0.1:8080` tras `up -d`**: la app tarda en arrancar (DB interna, migrations). `docker compose logs -f` y esperar; o healthchecks (ver [[Docker - Docker Compose]]).
2. **Lab "no vulnerable" tras unos días**: muchas apps antiguas generan datos aleatorios al primer arranque (DVWA pide reset en cada sesión; hay que recrear la DB con su botón o credenciales por defecto `admin/password`).
3. **`apt-get update` falla en imágenes EOL**: repos movidos a archive; no es la vulnerabilidad que buscabas — parchá el Dockerfile con el mirror.
4. **Puerto ocupado**: otro lab quedó corriendo; `docker ps` + `docker compose down` en el viejo.
5. **Exposición accidental**: `docker ps` con `0.0.0.0:8080->…` en un host con IP pública = tu lab es de todos. Bind loopback siempre.
6. **`down` sin `-v` "no resetea"**: volúmenes con datos de la sesión anterior sobreviven; `-v` cuando querés cero real.
7. **Kali sin herramientas / image gigante**: la rolling de Kali es enorme; instalá solo el kit (`nmap`, `nc`, `sqlmap`, `gobuster`) como arriba, o usá `kalilinux/kali-rolling-lite` + tools.

## Cheatsheet de labs

| Tarea | Comando |
|-------|---------|
| Levantar lab de Vulhub | `cd <lab> && docker compose up -d` |
| Ver logs de la víctima | `docker compose logs -f victim` |
| Reset total | `docker compose down -v` |
| Red aislada | `internal: true` / `--network none` |
| Sin puerto publicado | solo `networks:` (sin `ports:`) |
| Límites | `deploy.resources.limits` / `--memory --cpus` |
| Snapshot | `docker commit c lab/snap:punto1` |
| Evidencias | `docker cp c:/ruta ./evidencias/` |
| Atacante interno | `docker compose --profile attack up -d` + `exec attacker bash` |
| Escanear la imagen del lab | `trivy image vulnerables/web-dvwa` |

## Relacionadas

- [[Docker - Fundamentos y Conceptos]] — el aislamiento que hace posible (y frágil) estos labs
- [[Docker - Redes y Volúmenes]] — `internal: true`, bind loopback, DNS entre victim/attacker
- [[Docker - Docker Compose]] — el formato nativo de los stacks de Vulhub
- [[Docker - Gestión de Contenedores]] — ciclo de vida efímero, límites, snapshot/commit
- [[Docker - Buenas Prácticas y Seguridad]] — endurecer; y qué mirar cuando atacas
- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — el objetivo final de muchos labs: escapar del contenedor
- [[Comandos Básicos de Sistema y Red en Linux]] — `nmap`/`ss`/redes del host para el trabajo de campo
