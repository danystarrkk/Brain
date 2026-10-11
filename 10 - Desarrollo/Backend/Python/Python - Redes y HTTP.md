---
title: "Python - Redes y HTTP"
tags:
  - lang/python
  - concept/networking
  - concept/http
  - concept/sockets
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Redes y HTTP

> **Objetivo**: Cliente/servidor HTTP, sockets, WebSockets, gRPC, y patrones de comunicación.

---

## 1. HTTP Cliente: `requests` (Estándar de Facto)

```bash
pip install requests
```

### 1.1 Básico

```python
import requests

# GET
resp = requests.get("https://api.ejemplo.com/usuarios", timeout=10)
resp.raise_for_status()  # Lanza HTTPError si status >= 400
data = resp.json()

# POST con JSON
resp = requests.post(
    "https://api.ejemplo.com/usuarios",
    json={"nombre": "Juan", "email": "juan@ejemplo.com"},
    headers={"Authorization": "Bearer token123"},
    timeout=10
)

# POST con form data
resp = requests.post(
    "https://api.ejemplo.com/upload",
    data={"campo": "valor"},
    files={"archivo": open("archivo.txt", "rb")},
    timeout=30
)
```

### 1.2 Sesiones (Conexión Persistente)

```python
session = requests.Session()
session.headers.update({"User-Agent": "MiApp/1.0"})
session.auth = ("usuario", "contraseña")

# Cookies se mantienen automáticamente
resp1 = session.get("https://api.ejemplo.com/login")
resp2 = session.get("https://api.ejemplo.com/perfil")

# Adaptadores personalizados
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,
    backoff_factor=1,
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["HEAD", "GET", "OPTIONS"]
)
adapter = HTTPAdapter(max_retries=retry_strategy)
session.mount("https://", adapter)
session.mount("http://", adapter)
```

### 1.3 Streaming y Archivos Grandes

```python
# Descargar archivo grande sin cargar en memoria
with requests.get("https://ejemplo.com/archivo_grande.zip", stream=True) as resp:
    resp.raise_for_status()
    with open("archivo.zip", "wb") as f:
        for chunk in resp.iter_content(chunk_size=8192):
            f.write(chunk)

# Subir archivo grande
with open("archivo_grande.zip", "rb") as f:
    requests.post(
        "https://api.ejemplo.com/upload",
        data=f,
        headers={"Content-Type": "application/octet-stream"}
    )
```

---

## 2. HTTP Cliente Async: `aiohttp` / `httpx`

```bash
pip install aiohttp httpx
```

### 2.1 `aiohttp` (Nativo Async)

```python
import aiohttp
import asyncio

async def fetch(session, url):
    async with session.get(url) as resp:
        resp.raise_for_status()
        return await resp.json()

async def main():
    async with aiohttp.ClientSession() as session:
        # Concurrentemente
        tareas = [fetch(session, f"https://api.ejemplo.com/item/{i}") for i in range(100)]
        resultados = await asyncio.gather(*tareas)
        return resultados

# Semaphore para limitar concurrencia
sem = asyncio.Semaphore(10)

async def fetch_limited(session, url):
    async with sem:
        return await fetch(session, url)

asyncio.run(main())
```

### 2.2 `httpx` (Sync + Async, API Compatible)

```python
import httpx

# Sync
with httpx.Client(timeout=30.0) as client:
    resp = client.get("https://api.ejemplo.com/items")
    resp.raise_for_status()

# Async
async with httpx.AsyncClient(timeout=30.0) as client:
    resp = await client.get("https://api.ejemplo.com/items")
    resp.raise_for_status()
    
    # Streaming
    async with client.stream("GET", "https://ejemplo.com/grande.zip") as resp:
        async for chunk in resp.aiter_bytes(8192):
            await archivo.write(chunk)
```

---

## 3. Servidores HTTP

### 3.1 `http.server` (Built-in, Simple)

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class MiHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps({"mensaje": "Hola"}).encode())
    
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length).decode()
        data = json.loads(body)
        
        self.send_response(201)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps({"recibido": data}).encode())

if __name__ == "__main__":
    server = HTTPServer(("localhost", 8000), MiHandler)
    print("Servidor en http://localhost:8000")
    server.serve_forever()
```

### 3.2 Frameworks Modernos (Recomendados)

```bash
pip install fastapi uvicorn  # Recomendado
pip install flask            # Clásico
pip install quart            # Flask async
pip install sanic            # Async nativo
pip install starlette        # Base de FastAPI
```

#### FastAPI (Recomendado)

```python
from fastapi import FastAPI, HTTPException, Depends, Query
from pydantic import BaseModel
from typing import Optional
import uvicorn

app = FastAPI(title="Mi API", version="1.0.0")

class Item(BaseModel):
    nombre: str
    precio: float
    en_stock: bool = True

# In-memory DB
items_db = {}

@app.get("/")
def root():
    return {"mensaje": "Hola Mundo"}

@app.get("/items/{item_id}")
def leer_item(item_id: int, q: Optional[str] = None):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item no encontrado")
    return {"item": items_db[item_id], "query": q}

@app.post("/items/", status_code=201)
def crear_item(item: Item):
    item_id = len(items_db) + 1
    items_db[item_id] = item.dict()
    return {"id": item_id, **item.dict()}

@app.put("/items/{item_id}")
def actualizar_item(item_id: int, item: Item):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="No encontrado")
    items_db[item_id] = item.dict()
    return {"id": item_id, **item.dict()}

@app.delete("/items/{item_id}", status_code=204)
def borrar_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="No encontrado")
    del items_db[item_id]

# Dependency Injection
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/usuarios/")
def listar_usuarios(db: Session = Depends(get_db), skip: int = 0, limit: int = 100):
    return db.query(Usuario).offset(skip).limit(limit).all()

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 4. WebSockets

### 4.1 `websockets` (Cliente/Servidor)

```bash
pip install websockets
```

```python
import asyncio
import websockets

# Servidor
async def servidor(websocket):
    async for mensaje in websocket:
        await websocket.send(f"Echo: {mensaje}")

async def main():
    async with websockets.serve(servidor, "localhost", 8765):
        await asyncio.Future()  # Ejecutar para siempre

asyncio.run(main())
```

```python
# Cliente
async def cliente():
    async with websockets.connect("ws://localhost:8765") as ws:
        await ws.send("Hola")
        respuesta = await ws.recv()
        print(respuesta)

asyncio.run(cliente())
```

### 4.2 FastAPI + WebSockets

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Cliente {client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Cliente {client_id} desconectado")
```

---

## 5. Sockets Raw (TCP/UDP)

### 5.1 TCP Servidor/Cliente

```python
# Servidor TCP
import socket

def servidor_tcp():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(("0.0.0.0", 9999))
        s.listen()
        print("TCP Server en puerto 9999")
        
        while True:
            conn, addr = s.accept()
            with conn:
                print(f"Conectado: {addr}")
                while True:
                    data = conn.recv(1024)
                    if not data:
                        break
                    conn.sendall(data.upper())  # Echo en mayúsculas

# Cliente TCP
def cliente_tcp():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect(("localhost", 9999))
        s.sendall(b"Hola servidor")
        data = s.recv(1024)
        print(f"Recibido: {data.decode()}")

# Threading para múltiples clientes
import threading

def manejar_cliente(conn, addr):
    with conn:
        while True:
            data = conn.recv(1024)
            if not data:
                break
            conn.sendall(data)

def servidor_threading():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(("0.0.0.0", 9999))
        s.listen()
        while True:
            conn, addr = s.accept()
            threading.Thread(target=manejar_cliente, args=(conn, addr), daemon=True).start()
```

### 5.2 UDP

```python
# Servidor UDP
def servidor_udp():
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind(("0.0.0.0", 9999))
        print("UDP Server en puerto 9999")
        while True:
            data, addr = s.recvfrom(1024)
            print(f"De {addr}: {data.decode()}")
            s.sendto(data.upper(), addr)

# Cliente UDP
def cliente_udp():
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.sendto(b"Hola UDP", ("localhost", 9999))
        data, addr = s.recvfrom(1024)
        print(f"Respuesta: {data.decode()}")
```

---

## 6. gRPC (RPC de Alto Rendimiento)

```bash
pip install grpcio grpcio-tools protobuf
```

### 6.1 Definir Proto (`service.proto`)

```protobuf
syntax = "proto3";

package mi_servicio;

service Calculadora {
    rpc Sumar (SumaRequest) returns (SumaResponse);
    rpc SumarStream (stream SumaRequest) returns (SumaResponse);
}

message SumaRequest {
    int32 a = 1;
    int32 b = 2;
}

message SumaResponse {
    int32 resultado = 1;
}
```

### 6.2 Generar Código

```bash
python -m grpc_tools.protoc --proto_path=. --python_out=. --grpc_python_out=. service.proto
```

### 6.3 Servidor

```python
import grpc
from concurrent import futures
import service_pb2
import service_pb2_grpc

class CalculadoraServicer(service_pb2_grpc.CalculadoraServicer):
    def Sumar(self, request, context):
        return service_pb2.SumaResponse(resultado=request.a + request.b)
    
    def SumarStream(self, request_iterator, context):
        total = 0
        for req in request_iterator:
            total += req.a + req.b
        return service_pb2.SumaResponse(resultado=total)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    service_pb2_grpc.add_CalculadoraServicer_to_server(CalculadoraServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

### 6.3 Cliente

```python
import grpc
import service_pb2
import service_pb2_grpc

def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = service_pb2_grpc.CalculadoraStub(channel)
        
        # Unary
        response = stub.Sumar(service_pb2.SumaRequest(a=10, b=20))
        print(f"10 + 20 = {response.resultado}")
        
        # Streaming
        requests = [service_pb2.SumaRequest(a=i, b=i*2) for i in range(5)]
        response = stub.SumarStream(iter(requests))
        print(f"Stream sum: {response.resultado}")

if __name__ == "__main__":
    run()
```

---

## 7. DNS y Utilidades de Red

```python
import socket
import ipaddress

# Resolución DNS
socket.gethostbyname("google.com")  # IPv4
socket.getaddrinfo("google.com", 443)  # Lista de (family, type, proto, canonname, sockaddr)

# IP validation
ip = ipaddress.ip_address("192.168.1.1")
red = ipaddress.ip_network("192.168.1.0/24")
ip in red  # True/False

# Hostname local
socket.gethostname()
socket.getfqdn()
```

---

## 7. Patrones de Resiliencia

### 7.1 Circuit Breaker

```python
import asyncio
from functools import wraps

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker open")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise

# Uso
breaker = CircuitBreaker(failure_threshold=3, timeout=30)

def llamada_protegida():
    return breaker.call(requests.get, "https://api.inestable.com")
```

### 7.2 Retry con Backoff

```python
import time
import random
from functools import wraps

def retry(max_attempts=3, base_delay=1, max_delay=60, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
                    time.sleep(delay)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@retry(max_attempts=3, base_delay=1, exceptions=(requests.RequestException,))
def api_call():
    return requests.get("https://api.ejemplo.com/datos", timeout=5)
```

---

## 8. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Funciones, excepciones
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - Threading, asyncio
- [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) - Streaming, upload/download
- [Interfaces Gráficas](./Python%20-%20Interfaces%20Gr%C3%A1ficas%20(Tkinter).md) - Tkinter networking
- [Librerías Estándar](./Python%20-%20Librer%C3%ADas%20Est%C3%A1ndar%20Esenciales.md) - socket, ssl, urllib
- [Interfaces Gráficas](./Python%20-%20Interfaces%20Gr%C3%A1ficas%20(Tkinter).md) - WebSockets en GUI

---

*Etiquetas: #python #redes #http #requests #aiohttp #httpx #fastapi #flask #websockets #grpc #sockets #tcp #udp #dns #circuit-breaker #retry #resiliencia #streaming #async-http #microservicios #api-rest #api-graphql*