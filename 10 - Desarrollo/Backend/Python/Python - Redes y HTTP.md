---
tags:
  - desarrollo
  - backend
  - python
tipo: concepto
estado: terminado
---

# Python - Redes y HTTP

Referencia profunda de red en Python: el modelo de sockets (TCP/UDP), semántica HTTP para quien consume APIs, `requests` a fondo (sesiones, autenticación, reintentos, timeouts — **siempre** timeout), clientes REST resistentes (paginación, backoff, circuit breaker), `httpx`/`aiohttp` async, WebSockets, `http.server` y frameworks, gRPC, DNS, TLS y seguridad. Verificado con Python 3.14 y `requests` 2.34.2 / `urllib3` 2.8.0 contra un servidor local.

> [!info] Las capas del stack
> Abajo: **TCP/UDP** (`socket`, del SO) → arriba: **HTTP** (texto sobre TCP: `http.client`, `urllib`, `requests`, `httpx`) → arriba: **patrones REST/JSON** (tus clientes) → arriba: **WebSockets** (HTTP upgrade, bidireccional). Python te da cada capa; elegir la más alta que resuelva tu problema es la mitad del diseño.

## Semántica HTTP para quien escribe clientes

Recordatorio operativo (ver [[Qué es una API]] para el concepto):

| Elemento | Valores clave | Nota para clientes Python |
|---|---|---|
| Método | GET (idempotente, seguro), POST (crear/acción), PUT (reemplazo idempotente), PATCH (parcial), DELETE (idempotente) | GET con body es legal pero rompe caches/intermediarios |
| Status | 2xx ok, 3xx redirect (requests sigue por defecto `allow_redirects`), 4xx tu error (400/401/403/404/409/422/429), 5xx error del server | `resp.ok` es `status < 400` — **no** distingue 4xx de 5xx |
| Headers | `Content-Type`, `Authorization`, `Accept`, `User-Agent`, `Retry-After` (429), `Location` | Personalizados con guion: `X-Request-Id` |
| Estado | HTTP es **sin estado**; la sesión vive en cookies o tokens | Ver "Sesiones" más abajo |

## Elección del cliente HTTP

| Cliente | Sync/Async | Deps | Mejor para |
|---|---|---|---|
| `urllib.request` | Sync | stdlib | Scripts sin dependencias; aprendizaje |
| **`requests`** | Sync | `urllib3` | El estándar de facto: scripts, crawlers, CLIs |
| **`httpx`** | **Sync + Async**, misma API | `httpio`... `httpcore` | Código que puede volverse async; HTTP/2 |
| `aiohttp` | Solo async | propia | Clients/servers async de alto rendimiento |
| `http.client` | Sync | stdlib | Bajo nivel con control total de headers |

```python
# urllib de la stdlib — funciona sin pip, pero la API es cruda:
from urllib.request import urlopen, Request
req = Request("https://api.ejemplo.com/datos", headers={"User-Agent": "MiApp/1.0"})
with urlopen(req, timeout=10) as resp:          # timeout OBLIGATORIO también acá
    datos = resp.read().decode()
```

> [!warning] `urllib` no tiene `timeout` por defecto
> Sin timeout, una conexión muerta **cuelga el proceso para siempre**. Vale para urllib, requests, sockets — toda la pila. Regla: timeout en **toda** llamada de red, sí, también en las internas.

## `requests` a fondo

### Lo básico bien hecho

```python
import requests

# GET con parámetros (requests construye la query string y escapa)
resp = requests.get(
    "https://api.ejemplo.com/usuarios",
    params={"page": 2, "activo": "true"},
    headers={"Authorization": "Bearer token123"},
    timeout=10,                    # (connect, read) o un solo número
)
resp.raise_for_status()            # HTTPError si status >= 404
data = resp.json()                 # parsea JSON (json si el body lo es)
print(resp.status_code, resp.headers["Content-Type"], resp.encoding)

# POST JSON — `json=` serializa Y pone Content-Type: application/json
resp = requests.post(
    "https://api.ejemplo.com/usuarios",
    json={"nombre": "Juan", "email": "juan@ejemplo.com"},
    timeout=10,
)

# POST multipart (archivos) y form-urlencoded (data=)
with open("reporte.csv", "rb") as f:
    requests.post("https://api.ejemplo.com/upload",
                  files={"archivo": ("reporte.csv", f, "text/csv")},
                  data={"descripcion": "mensual"}, timeout=30)
```

Verificado contra servidor local: cookies de sesión persisten entre peticiones, `raise_for_status()` lanza `HTTPError` con `.response.status_code == 404`, y el roundtrip `json=` → `resp.json()` devuelve el dict exacto.

### Timeouts: la distinción que salva scripts

```python
try:
    requests.get(url, timeout=(3.05, 30))     # (connect, read): 3s conectar, 30s sin datos
except requests.exceptions.ConnectTimeout:
    ...   # no llegó a establecer TCP
except requests.exceptions.ReadTimeout:
    ...   # VERIFICADO: conectó pero el server no respondió → lanza tras exactamente el timeout
except requests.exceptions.ConnectionError as e:
    ...   # connection refused también cae aquí (verificado: puerto cerrado → refused inmediato,
          #  NO timeout — refused es rápido; el timeout protege contra servers que aceptan y mueren)
```

### Sesiones: conexión persistente y estado

```python
session = requests.Session()
session.headers.update({"User-Agent": "MiApp/1.0", "Accept": "application/json"})
session.auth = ("usuario", "contraseña")          # Basic auth en todas las peticiones
session.proxies = {"http": "http://proxy:8080", "https": "http://proxy:8080"}

r1 = session.get("https://api.ejemplo.com/login")     # Set-Cookie → session.cookies
r2 = session.get("https://api.ejemplo.com/perfil")    # Cookie enviada automáticamente
```

Una `Session` reutiliza el **pool de conexiones TCP** (keep-alive): 100 requests a la misma API pasan de 100 handshakes TLS a ~1. También sirve de "cliente tipado" para tu API: headers base, auth y retries montados una vez.

### Reintentos con backoff (montaje correcto)

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=3,                              # intentos extra (3 = 4 llamadas totales)
    backoff_factor=1,                     # esperas: 1s, 2s, 4s (exponencial)
    status_forcelist=[429, 500, 502, 503, 504],   # reintentar estos status
    allowed_methods=["GET", "PUT", "DELETE"],     # SOLO métodos idempotentes
    respect_retry_after_header=True,      # 429 con Retry-After respeta el header
)
session = requests.Session()
adapter = HTTPAdapter(max_retries=retry, pool_connections=10, pool_maxsize=32)
session.mount("https://", adapter)
session.mount("http://", adapter)
```

Verificado con `urllib3` 2.8.0 (el nombre `allowed_methods` reemplazó al viejo `method_whitelist`, eliminado en urllib3 2.0 — código de la era 1.x puede fallar con `TypeError`). **Nunca** retries en POST sin confirmar idempotencia: duplica efectos.

### Descarga y subida de archivos

```python
# Descarga por chunks: un archivo de 5 GB no cabría en memoria
with requests.get(url, stream=True, timeout=30) as resp:   # stream=True: no lee el body
    resp.raise_for_status()
    with open("grande.zip", "wb") as f:
        for chunk in resp.iter_content(chunk_size=1 << 16):   # 64 KiB
            f.write(chunk)                                    # ver [[Python - Manejo de Archivos y Serialización]]

# Subida progresiva de un file-like
with open("video.mp4", "rb") as f:
    requests.post(upload_url, data=f, headers={"Content-Type": "video/mp4"}, timeout=300)
```

### Autenticación

| Tipo | Código |
|---|---|
| Basic | `session.auth = ("user", "pass")` o `HTTPBasicAuth` |
| Bearer token | `headers={"Authorization": f"Bearer {token}"}` |
| Digest | `session.auth = HTTPDigestAuth("user", "pass")` |
| OAuth1 | librería `requests-oauthlib` |
| API key | header o `params={"api_key": TOKEN}` según el proveedor |

### Proxies, TLS y verificación

```python
# Proxy explícito (o variables HTTPS_PROXY del entorno)
requests.get(url, proxies={"https": "http://user:pass@proxy:3128"}, timeout=10)

# TLS: la verificación está ACTIVADA por defecto con certificados de CA del sistema
requests.get("https://api.ejemplo.com", timeout=10)                     # ✅ verifica
requests.get(url, verify="/ruta/ca-corporativa.pem", timeout=10)        # CA propia
# requests.get(url, verify=False)   # ❌ MITM trivial; solo debugging con urllib3 InsecureRequestWarning
```

> [!warning] `verify=False` es un anti-patrón de seguridad
> Desactiva la autenticidad del servidor: cualquier intermediario puede leer y alterar el tráfico. Si el problema era un certificado autofirmado, pasá `verify=` con el PEM; si era una CA vieja, actualizá `certifi`. Verificá también con `socket.get_default_verify_paths()`. Y **nunca** hardcodees tokens: usá variables de entorno (`os.environ`) o `keyring`; en repos, historial de git — ver [[Python Ofensivo]] para cómo se rastrean secretos olvidados.

## Patrones de cliente REST resistente

### Manejo de errores por capas

```python
def llamar_api(session: requests.Session, metodo: str, url: str, **kw) -> requests.Response:
    kw.setdefault("timeout", 10)
    try:
        resp = session.request(metodo, url, **kw)
        resp.raise_for_status()
        return resp
    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 401:
            raise AuthExpirada() from e          # renovar token y reintentar una vez
        if e.response.status_code == 429:
            raise Limitado(retry_after=e.response.headers.get("Retry-After")) from e
        raise ApiError(e.response.status_code, e.response.text[:200]) from e
    except requests.exceptions.Timeout as e:
        raise ApiTimeout() from e
    except requests.exceptions.ConnectionError as e:
        raise ApiSinConexion() from e
```

La taxonomía de excepciones propias (una por causa, raíz `ApiError`) permite que el caller reaccione **por tipo**, no parseando mensajes.

### Paginación (los tres esquemas reales)

```python
# 1) offset/limit
def paginar_offset(session, url, por_pagina=100):
    pagina = 0
    while True:
        r = llamar_api(session, "GET", url, params={"offset": pagina * por_pagina,
                                                    "limit": por_pagina})
        items = r.json()
        if not items:
            return
        yield from items
        pagina += 1

# 2) cursor/next (el diseño moderno: el server decide)
def paginar_cursor(session, url):
    siguiente = url
    while siguiente:
        datos = llamar_api(session, "GET", siguiente).json()
        yield from datos["items"]
        siguiente = datos.get("next")            # null → fin

# 3) Link header (RFC 8288, estilo GitHub)
import re
def paginar_link(session, url):
    r = llamar_api(session, "GET", url)
    yield from r.json()
    while m := re.search(r'<([^>]+)>;\s*rel="next"', r.headers.get("Link", "")):
        r = llamar_api(session, "GET", m.group(1))
        yield from r.json()
```

Generadores (`yield`) → el caller procesa item por item sin cargar millones de registros en RAM.

### Retry con backoff a medida (decorador)

```python
import time, random, requests
from functools import wraps

def retry(max_attempts=3, base_delay=1.0, max_delay=60.0,
          exceptions=(requests.RequestException,)):
    def decorador(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for intento in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if intento == max_attempts - 1:
                        raise
                    # backoff exponencial + jitter: evita el "thundering herd"
                    delay = min(base_delay * 2 ** intento + random.uniform(0, 1), max_delay)
                    time.sleep(delay)
        return wrapper
    return decorador

@retry(max_attempts=3, exceptions=(requests.Timeout, requests.ConnectionError))
def obtener(url):
    return requests.get(url, timeout=5).json()
```

### Circuit breaker (cuando reintentar hace daño)

```python
import time, requests

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure = 0.0
        self.state = "closed"          # closed → open → half-open

    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.monotonic() - self.last_failure < self.recovery_timeout:
                raise CircuitOpen("servicio en cooldown — no se intenta")
            self.state = "half-open"   # ventana de prueba
        try:
            result = func(*args, **kwargs)
            if self.state == "half-open":
                self.state, self.failures = "closed", 0    # recuperado
            return result
        except Exception:
            self.failures += 1
            self.last_failure = time.monotonic()
            if self.failures >= self.failure_threshold:
                self.state = "open"    # cortar: fallar rápido en vez de colgar la cola
            raise

class CircuitOpen(RuntimeError): ...

breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=30)
datos = breaker.call(requests.get, "https://api.inestable.com", timeout=5)
```

Estado **open** = "no llames ni reintentés": el error se propaga de inmediato y el sistema downstream respira. **half-open** = una ventana de prueba; si pasa, closed.

### Testing de clientes HTTP (sin red)

| Librería | Qué hace |
|---|---|
| `responses` | Intercepta `requests` (y `requests-mock` es la alternativa) |
| `pytest-httpx` | Igual para `httpx` (sync y async) |
| `respx` | Para `aiohttp` |
| `aioresponses` | idem aiohttp |
| Local: `http.server`/`socketserver` | Un server real en un hilo (como el verificado en esta nota) |

```python
import responses, requests

@responses.activate
def test_get_usuario():
    responses.get("https://api.test/usuarios/1",
                  json={"id": 1, "nombre": "Ana"}, status=200)
    r = requests.get("https://api.test/usuarios/1", timeout=5)
    assert r.json()["nombre"] == "Ana"
```

Detalle en [[Python - Testing, Calidad y Herramientas]].

## Cliente async: `aiohttp` y `httpx`

Cuando el cliente vive dentro de un event loop (ver [[Python - Concurrencia Threading Multiprocessing Asyncio]]), `requests` no sirve: bloquea el loop. Dos alternativas:

```python
# aiohttp — nativo async; UNA sesión para toda la app (el bug clásico es una por request)
import asyncio, aiohttp

sem = asyncio.Semaphore(10)          # límite anti-429 / anti-FD

async def fetch(session, url):
    async with sem:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=30)) as resp:
            resp.raise_for_status()
            return url, await resp.json()

async def main(urls):
    timeout = aiohttp.ClientTimeout(total=60)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        return await asyncio.gather(*[fetch(session, u) for u in urls])
```

```python
# httpx — la MISMA API que requests, sync y async; ideal si el código puede migrar
import httpx

with httpx.Client(timeout=30.0) as client:                 # sync: gotos de requests
    r = client.get("https://api.ejemplo.com/items")
    r.raise_for_status()

async with httpx.AsyncClient(timeout=30.0) as client:      # async: mismo nombre de métodos
    r = await client.get("https://api.ejemplo.com/items")
    async with client.stream("GET", "https://ejemplo.com/grande.zip") as resp:
        async for chunk in resp.aiter_bytes(1 << 16):
            ...                                            # escritura asíncrona del chunk
```

| | `requests` | `httpx` | `aiohttp` |
|---|---|---|---|
| Sync | ✅ | ✅ | ❌ |
| Async | ❌ | ✅ (misma API) | ✅ |
| HTTP/2 | ❌ | ✅ | ❌ |
| Migración desde requests | — | Casi 1:1 | Reescritura |

## WebSockets

HTTP es request/response; WebSockets es un **canal bidireccional persistente** (handshake HTTP con `Upgrade: websocket`, después frames binarios/texto). Uso: chats, tickers, notificaciones, juego. Librería estándar de facto: `websockets` (`pip install websockets`; no está en stdlib).

```python
# servidor echo
import asyncio, websockets

async def handler(websocket):
    async for mensaje in websocket:          # bucle de recepción
        await websocket.send(f"Echo: {mensaje}")

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        await asyncio.Future()               # sirve para siempre

# cliente
async def cliente():
    async with websockets.connect("ws://localhost:8765") as ws:
        await ws.send("Hola")
        print(await ws.recv())               # "Echo: Hola"

asyncio.run(main())     # ver [[Python - Concurrencia Threading Multiprocessing Asyncio]] para el event loop
```

Con FastAPI, el patrón broadcast (absorbido del curso, versión production-shape):

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []
    async def connect(self, ws: WebSocket):
        await ws.accept(); self.active.append(ws)
    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)
    async def broadcast(self, message: str):
        for ws in list(self.active):          # copia: la lista puede cambiar mientras itera
            await ws.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def endpoint(ws: WebSocket, client_id: int):
    await manager.connect(ws)
    try:
        while True:
            await manager.broadcast(f"Cliente {client_id}: {await ws.receive_text()}")
    except WebSocketDisconnect:
        manager.disconnect(ws)
        await manager.broadcast(f"Cliente {client_id} desconectado")
```

## Sockets crudos (TCP/UDP): cuándo y cómo

Usá sockets **crudos** cuando: el protocolo no es HTTP (SMTP, DNS propio, MQTT, protocolos de juego), necesitás control total del framing, o implementás el servidor que otros llamarán por HTTP. Todo lo demás: subí de capa.

### TCP — modelo `bind → listen → accept → recv/send`

```python
import socket

def servidor_tcp():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # sin esto, re-bind → "Address
        s.bind(("0.0.0.0", 9999))                                #   already in use" (TIME_WAIT)
        s.listen(backlog=5)                                       # cola de conexiones pendientes
        print("TCP en :9999")
        while True:
            conn, addr = s.accept()          # bloquea hasta conexión; conn = socket hijo
            with conn:
                while True:
                    data = conn.recv(1024)   # TCP es FLUJO: recv puede traer 1 byte o 100 KB
                    if not data:             # b"" = el peer cerró (EOF), no "no hay datos"
                        break
                    conn.sendall(data.upper())   # sendall: envía todo (send puede cortar)

# cliente
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(("127.0.0.1", 9999))
    s.sendall(b"hola servidor")
    print(s.recv(1024).decode())     # VERIFICADO contra servidor local: "HOLA SERVIDOR"
```

> [!info] `recv` no es "un mensaje"
> TCP no tiene mensajes: es un flujo de bytes. `recv(1024)` devuelve **hasta** 1024 bytes: podría traer medio mensaje o tres. Los protocolos reales arreglan esto con **delimitadores** (HTTP: `\r\n\r\n` + `Content-Length`; línea: `\n`) o **framing de longitud** (4 bytes big-endian + payload). `sendall` sí envía todo; `send` devuelve cuántos mandó.

Múltiples clientes: thread por conexión (ver `threading` en [[Python - Concurrencia Threading Multiprocessing Asyncio]]) o `socketserver.ThreadingTCPServer`; para 10k+ conexiones: `selectors`/asyncio.

### UDP — datagramas sin conexión

```python
# servidor: no accept(); cada datagrama trae su origen
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    s.bind(("0.0.0.0", 9999))
    while True:
        data, addr = s.recvfrom(1024)   # datagrama completo + dirección remitente
        s.sendto(data.upper(), addr)    # respuesta hacia ese origen

# cliente
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    s.sendto(b"Hola UDP", ("127.0.0.1", 9999))
    data, addr = s.recvfrom(1024)       # puede no llegar NUNCA (sin retransmisión)
```

| TCP | UDP |
|---|---|
| Conexión, confiabilidad, orden | Sin conexión, sin garantías |
| `connect`/`accept`, stream | `sendto`/`recvfrom`, datagramas |
| HTTP, SMTP, SSH, bases de datos | DNS, DHCP, gaming, VoIP, video |
| Coste: handshake, retransmisión | Coste: perdidas y reordenación a tu cargo |

### TLS sobre un socket

```python
import ssl
ctx = ssl.create_default_context()            # CA del sistema + verificación activa
with socket.create_connection(("example.com", 443), timeout=10) as sock:
    with ctx.wrap_socket(sock, server_hostname="example.com") as ssock:
        ssock.sendall(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
        print(ssock.recv(4096)[:80])
```

`ssl.create_default_context()` configura el TLS moderno (TLS 1.2+) y **verifica el hostname**. La API vieja `ssl.wrap_socket` del módulo fue **eliminada en Python 3.12** (corrección aplicada también al chat de [[Python - Interfaces Gráficas (Tkinter)]]).

## `http.server` y frameworks

### Servidor stdlib (para tools, no para producción)

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class MiHandler(BaseHTTPRequestHandler):
    def _responder(self, status: int, datos: dict) -> None:
        cuerpo = json.dumps(datos).encode()
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(cuerpo)))   # sin esto, clientes cuelgan
        self.end_headers()
        self.wfile.write(cuerpo)

    def do_GET(self):
        self._responder(200, {"mensaje": "Hola"})

    def do_POST(self):
        largo = int(self.headers.get("Content-Length", 0))
        datos = json.loads(self.rfile.read(largo) or b"{}")
        self._responder(201, {"recibido": datos})

if __name__ == "__main__":
    HTTPServer(("localhost", 8000), MiHandler).serve_forever()
```

Sirve para: herramientas internas, healthchecks, prototipos. **No**: maneja un request a la vez, sin autenticación, sin validación — para algo real, un framework.

### Frameworks (elegir según estilo)

```bash
pip install fastapi uvicorn    # moderno: async, tipado, OpenAPI gratis, validación pydantic
pip install flask              # clásico: minimal, síncrono, maduro
pip install quart flask[async] # Flask con async
```

```python
# FastAPI: el server mínimo con validación declarativa
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
import uvicorn

app = FastAPI(title="Mi API", version="1.0.0")

class Item(BaseModel):
    nombre: str = Field(min_length=1)
    precio: float = Field(gt=0)
    en_stock: bool = True

items_db: dict[int, Item] = {}

@app.get("/items/{item_id}", status_code=200)
def leer_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item no encontrado")
    return items_db[item_id]

@app.post("/items/", status_code=201)
def crear_item(item: Item):
    item_id = len(items_db) + 1
    items_db[item_id] = item
    return {"id": item_id, **item.model_dump()}   # .model_dump() (pydantic v2)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## gRPC (RPC de alto rendimiento)

Protocolo binario sobre HTTP/2 con `.proto` como contrato — ver el ejemplo completo del curso: `service.proto` → `python -m grpc_tools.protoc --python_out=. --grpc_python_out=. service.proto` → servicer + stub. Elegí gRPC cuando: múltiples lenguajes, llamadas frecuentes pequeñas (menor overhead que JSON), o streaming bidireccional nativo. Elegí REST/JSON cuando: consumidores web, caché HTTP importa, o simplicidad. `pip install grpcio grpcio-tools protobuf`.

## DNS e IP con la stdlib

```python
import socket, ipaddress

socket.gethostbyname("example.com")                 # IPv4 simple
socket.getaddrinfo("example.com", 443)              # lista completa: (family, type, proto, canon, sockaddr)
                                                     # — IPv4 e IPv6; elige la que funcione
socket.gethostname(), socket.getfqdn()              # nombre local

ip = ipaddress.ip_address("192.168.1.50")
red = ipaddress.ip_network("192.168.1.0/24")
print(ip in red, red.num_addresses)                 # True 256 — validación sin regex
```

## Errores comunes

1. **Sin `timeout`** → procesos colgados para siempre; siempre `(connect, read)` o número.
2. **`resp.json()` sin `raise_for_status()`** → parseás el HTML de la página de error 500 y el bug se va lejos.
3. **`ConnectionError` interpretado como timeout** → refused es inmediato (verificado); distinguí los tipos de `requests.exceptions`.
4. **Retries en POST** → duplica efectos; solo idempotentes (GET/PUT/DELETE).
5. **`Session` por request** → anula el keep-alive y agota FDs; una sesión por aplicación (igual que `aiohttp.ClientSession`).
6. **Asumir que `recv` trae un mensaje completo** → framing con delimitador o longitud.
7. **`verify=False` en producción** → MITM; `verify=` con tu PEM.
8. **Secretos hardcodeados** → entorno/keyring; el git history es eterno.
9. **Leer un archivo grande entero con `resp.content`** → `stream=True` + `iter_content`.
10. **Parsear JSON con `eval` o regex** → `json.loads`/`resp.json()`; ver [[Formato Json]].
11. **`time.sleep` como retry sin jitter** → todos los clientes reintantan al mismo segundo (thundering herd).
12. **Confundir UDP con TCP fiable** → sin retransmisión ni orden; o implementás confiabilidad o elegís TCP.

## Cheatsheet

| Tarea | Código | Nota |
|---|---|---|
| GET con params | `requests.get(url, params={...}, timeout=10)` | Escaping automático |
| JSON out/in | `resp.json()` / `json=` en request | Content-Type automático |
| Lanzar en 4xx/5xx | `resp.raise_for_status()` | Antes de parsear |
| Sesión | `s = requests.Session()` | Keep-alive + cookies + headers base |
| Retries | `HTTPAdapter(max_retries=Retry(...))` | Solo idempotentes |
| Descarga grande | `stream=True` + `iter_content(64<<10)` | Memoria constante |
| Auth bearer | `headers={"Authorization": f"Bearer {t}"}` | — |
| Proxy | `proxies={"https": "http://proxy:3128"}` | — |
| TLS propio | `verify="ca.pem"` | `verify=False` jamás en prod |
| Async | `httpx.AsyncClient()` / `aiohttp` | Ver Concurrencia |
| WebSocket | `websockets.connect(...)` / `serve(...)` | Bidireccional |
| TCP crudo | `socket(); bind(); listen(); accept()` | Framing a tu cargo |
| UDP crudo | `sendto/recvfrom` | Sin garantías |
| TLS crudo | `ssl.create_default_context().wrap_socket(...)` | Verifica hostname |
| Server stdlib | `HTTPServer` + `BaseHTTPRequestHandler` | Tools, no prod |
| API real | FastAPI + uvicorn | Tipado + OpenAPI |
| Test sin red | `responses` / `pytest-httpx` | Interceptores |
| IP/cidr | `ipaddress.ip_network(".../24")` | Validación nativa |

## Relacionadas

- [[Qué es una API]] — el concepto de API web sobre el que se apoya todo HTTP
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — `aiohttp`, asyncio y threads como motores del cliente
- [[Python - Manejo de Archivos y Serialización]] — guardar descargas; JSON como formato (ver [[Formato Json]])
- [[Python - Interfaces Gráficas (Tkinter)]] — chat TCP+TLS sobre Tkinter (Proyecto 3)
- [[Python - Librerías Estándar Esenciales]] — `socket`, `ssl`, `urllib`, `http.server` en el mapa de la stdlib
- [[Python - Testing, Calidad y Herramientas]] — `responses`/`pytest-httpx` para clientes sin red
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `curl`, `ping`, `traceroute` para diagnosticar lo que tu código ve
- [[Python Ofensivo]] — contexto defensivo: cómo se interceptan/analizan tráfico y secretos
