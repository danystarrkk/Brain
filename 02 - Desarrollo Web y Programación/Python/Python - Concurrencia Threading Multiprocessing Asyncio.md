---
title: "Python - Concurrencia: Threading, Multiprocessing, Asyncio"
tags:
  - lang/python
  - concept/concurrency
  - concept/threading
  - concept/multiprocessing
  - concept/asyncio
  - level/avanzado
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Python - Concurrencia: Threading, Multiprocessing, Asyncio

> **Objetivo**: Elegir y usar el modelo de concurrencia correcto para cada caso: I/O-bound vs CPU-bound.

---

## 1. Decisión Rápida: ¿Qué Modelo Usar?

| Tipo de Tarea | Modelo | Módulo | Por qué |
|---------------|--------|--------|---------|
| **I/O-bound** (red, disco, BD, API) | **Threading** / **Asyncio** | `threading`, `concurrent.futures`, `asyncio` | GIL se libera en I/O; threads esperan sin bloquear |
| **CPU-bound** (cálculo, ML, compresión, imagen) | **Multiprocessing** | `multiprocessing`, `concurrent.futures.ProcessPoolExecutor` | Cada proceso tiene su propio GIL → paralelismo real |
| **Mixto** (I/O + CPU) | Híbrido | `ThreadPoolExecutor` + `ProcessPoolExecutor` | Separar por tipo de tarea |
| **Alta concurrencia I/O** (10k+ conexiones) | **Asyncio** | `asyncio`, `aiohttp`, `asyncpg` | Un solo thread, event loop, overhead mínimo |

---

## 2. Threading (I/O-bound)

### 2.1 Básico

```python
import threading
import time

def tarea(nombre, duracion):
    print(f"{nombre} iniciando")
    time.sleep(duracion)
    print(f"{nombre} finalizado")

# Crear y lanzar threads
threads = []
for i in range(3):
    t = threading.Thread(target=tarea, args=(f"Thread-{i}", 1))
    threads.append(t)
    t.start()

# Esperar a que terminen
for t in threads:
    t.join()

print("Todos terminaron")
```

### 2.2 ThreadPoolExecutor (Recomendado)

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

def descargar(url):
    resp = requests.get(url, timeout=10)
    return url, len(resp.content)

urls = [
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/2",
    "https://httpbin.org/delay/1",
]

with ThreadPoolExecutor(max_workers=3) as executor:
    # Enviar tareas
    future_to_url = {executor.submit(requests.get, url, timeout=10): url for url in urls}
    
    # Procesar a medida que completan
    for future in as_completed(future_to_url):
        url = future_to_url[future]
        try:
            resp = future.result()
            print(f"{url}: {len(resp.content)} bytes")
        except Exception as e:
            print(f"{url} falló: {e}")

# O con map (orden preservado)
with ThreadPoolExecutor(max_workers=3) as executor:
    resultados = list(executor.map(lambda u: requests.get(u, timeout=10).status_code, urls))
```

### 2.2 Sincronización

```python
import threading

# Lock básico
lock = threading.Lock()
contador = 0

def incrementar():
    global contador
    with lock:
        contador += 1

# RLock (reentrant - mismo thread puede adquirir múltiple vez)
rlock = threading.RLock()

# Condition (esperar notificación)
cond = threading.Condition()
datos_listos = False

def consumidor():
    with cond:
        while not datos_listos:
            cond.wait()
        print("Datos procesados")

def productor():
    global datos_listos
    with cond:
        datos_listos = True
        cond.notify_all()

# Event (señal simple)
evento = threading.Event()

def esperar():
    evento.wait()
    print("Evento recibido!")

def disparar():
    time.sleep(1)
    evento.set()

# Semaphore (limitar concurrencia)
sem = threading.Semaphore(2)  # máx 2 concurrentes

def tarea_limitada():
    with sem:
        print("Ejecutando...")
        time.sleep(1)

# Barrier (sincronizar N threads)
barrera = threading.Barrier(3)

def fase():
    print("Fase 1")
    barrera.wait()
    print("Fase 2")
```

### 2.3 Thread-Local Storage

```python
import threading

local_data = threading.local()

def procesar(request_id):
    local_data.request_id = request_id
    print(f"Procesando {local_data.request_id}")

# Cada thread tiene su propio local_data.request_id
```

### 2.4 Buenas Prácticas Threading

```python
# ✅ Usar ThreadPoolExecutor en lugar de Thread raw
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(tarea, arg) for arg in args]
    for future in as_completed(futures):
        resultado = future.result()

# ✅ Limitar workers (evita agotar recursos)
ThreadPoolExecutor(max_workers=min(32, os.cpu_count() + 4))

# ✅ Usar join() con timeout
thread.join(timeout=5.0)
if thread.is_alive():
    print("Thread no terminó a tiempo")

# ❌ EVITAR: mutable shared state sin locks
# ❌ EVITAR: operaciones CPU-intensivas en threads (GIL)
# ❌ EVITAR: crear threads ilimitados en loops
```

---

## 3. Multiprocessing (CPU-bound)

### 3.1 Básico

```python
import multiprocessing
import time

def cpu_intensivo(n):
    # Simula trabajo CPU
    total = sum(i * i for i in range(n))
    return total

if __name__ == "__main__":  # ¡OBLIGATORIO en Windows!
    with multiprocessing.Pool(processes=4) as pool:
        resultados = pool.map(cpu_intensivo, [10**6, 10**6, 10**6, 10**6])
        print(resultados)
```

### 3.2 ProcessPoolExecutor (Recomendado)

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import math

def es_primo(n):
    if n < 2:
        return False
    for i in range(2, int(math.sqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

numeros = range(100000, 101000)

with ProcessPoolExecutor(max_workers=4) as executor:
    # map preserva orden
    primos = list(executor.map(es_primo, numeros))
    
    # as_completed para procesar a medida que terminan
    futures = {executor.submit(es_primo, n): n for n in numeros}
    for future in as_completed(futures):
        n = futures[future]
        if future.result():
            print(f"Primo encontrado: {n}")
```

### 3.3 Comunicación Entre Procesos

```python
import multiprocessing

# Queue (segura para procesos)
def productor(q):
    for i in range(5):
        q.put(i)
    q.put(None)  # señal fin

def consumidor(q):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Consumido: {item}")

if __name__ == "__main__":
    q = multiprocessing.Queue()
    p1 = multiprocessing.Process(target=productor, args=(q,))
    p2 = multiprocessing.Process(target=consumidor, args=(q,))
    p1.start()
    p2.start()
    p1.join()
    p2.join()

# Pipe (bidireccional, dos extremos)
def hijo(conn):
    conn.send([42, None, 'hola'])
    conn.close()

if __name__ == "__main__":
    padre_conn, hijo_conn = multiprocessing.Pipe()
    p = multiprocessing.Process(target=hijo, args=(hijo_conn,))
    p.start()
    print(padre_conn.recv())
    p.join()

# Shared Memory (Value, Array)
def incrementar(contador):
    for _ in range(10000):
        with contador.get_lock():
            contador.value += 1

if __name__ == "__main__":
    contador = multiprocessing.Value('i', 0)
    procesos = [multiprocessing.Process(target=incrementar, args=(contador,)) for _ in range(4)]
    for p in procesos: p.start()
    for p in procesos: p.join()
    print(contador.value)  # 40000
```

### 3.2 Manager (Estado Compartido Complejo)

```python
from multiprocessing import Manager, Process

def trabajador(diccionario_compartido, lista_compartida, lock, id):
    with lock:
        diccionario_compartido[id] = f"proceso_{id}"
        lista_compartida.append(id)

if __name__ == "__main__":
    with Manager() as manager:
        d = manager.dict()
        l = manager.list()
        lock = manager.Lock()
        
        procesos = [Process(target=trabajador, args=(d, l, lock, i)) for i in range(4)]
        for p in procesos: p.start()
        for p in procesos: p.join()
        
        print(dict(d))  # {0: 'proceso_0', 1: 'proceso_1', ...}
        print(list(l))  # [0, 1, 2, 3]
```

---

## 3. Asyncio (I/O-bound, Alta Concurrencia)

### 3.1 Conceptos Fundamentales

```python
import asyncio

async def tarea(nombre, duracion):
    print(f"{nombre} iniciando")
    await asyncio.sleep(duracion)  # NO bloquea event loop
    print(f"{nombre} finalizado")
    return f"Resultado de {nombre}"

async def main():
    # Ejecutar concurrentemente
    resultados = await asyncio.gather(
        tarea("A", 1),
        tarea("B", 2),
        tarea("C", 1)
    )
    print(resultados)

asyncio.run(main())
```

### 3.2 Patrones Comunes

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as resp:
        return url, await resp.text()

async def main():
    urls = [
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/delay/1",
    ]
    
    async with aiohttp.ClientSession() as session:
        # Concurrentemente con límite
        semaphore = asyncio.Semaphore(10)
        
        async def fetch_limited(url):
            async with semaphore:
                async with aiohttp.ClientSession() as session:
                    async with session.get(url) as resp:
                        return url, resp.status
        
        tareas = [fetch_limited(url) for url in urls]
        resultados = await asyncio.gather(*tareas)
        
        for url, status in resultados:
            print(f"{url}: {status}")

asyncio.run(main())
```

### 3.2 Patrones Avanzados

```python
import asyncio

# Timeout
async def con_timeout():
    try:
        await asyncio.wait_for(operacion_lenta(), timeout=5.0)
    except asyncio.TimeoutError:
        print("Timeout!")

# Semaphore (limitar concurrencia)
sem = asyncio.Semaphore(5)

async def tarea_limitada():
    async with sem:
        await hacer_algo()

# Queue async
queue = asyncio.Queue()

async def productor():
    for i in range(10):
        await queue.put(i)
    await queue.put(None)  # señal fin

async def consumidor():
    while True:
        item = await queue.get()
        if item is None:
            break
        await procesar(item)
        queue.task_done()

# Event, Lock, Condition, Semaphore, Barrier
event = asyncio.Event()
lock = asyncio.Lock()
condition = asyncio.Condition()
semaphore = asyncio.Semaphore(3)
barrier = asyncio.Barrier(3)

# Task groups (3.11+)
async def main():
    async with asyncio.TaskGroup() as tg:
        tg.create_task(tarea1())
        tg.create_task(tarea2())
    # Si una falla, se cancelan las otras

# Wait for first completion
done, pending = await asyncio.wait(
    [tarea1(), tarea2(), tarea3()],
    return_when=asyncio.FIRST_COMPLETED
)

# Shield (proteger de cancelación)
await asyncio.shield(operacion_critica())
```

### 3.3 Async Context Managers

```python
class ConexionDB:
    async def __aenter__(self):
        self.conn = await asyncpg.connect()
        return self.conn
    
    async def __aexit__(self, exc_type, exc, tb):
        await self.conn.close()

async def main():
    async with ConexionDB() as conn:
        await conn.execute("SELECT 1")

# contextlib.asynccontextmanager
from contextlib import asynccontextmanager

@asynccontextmanager
async def transaccion(conn):
    tr = conn.transaction()
    await tr.start()
    try:
        yield tr
        await tr.commit()
    except Exception:
        await tr.rollback()
        raise
```

### 3.4 Buenas Prácticas Asyncio

```python
# ✅ Usar asyncio.run() una sola vez al inicio
asyncio.run(main())

# ✅ No mezclar código bloqueante en event loop
# ❌ time.sleep(1)  # BLOQUEA todo el loop
# ✅ await asyncio.sleep(1)

# ✅ Ejecutar código bloqueante en executor
loop = asyncio.get_event_loop()
resultado = await loop.run_in_executor(None, funcion_bloqueante, arg)

# ✅ Limitar concurrencia con Semaphore
sem = asyncio.Semaphore(100)
async def fetch(url):
    async with sem:
        return await session.get(url)

# ✅ Cancelación cooperativa
async def tarea_larga():
    for i in range(100):
        if asyncio.current_task().cancelled():
            break
        await asyncio.sleep(0.1)

# ✅ TaskGroups (3.11+) para manejo de errores
async with asyncio.TaskGroup() as tg:
    tg.create_task(tarea1())
    tg.create_task(tarea2())
# Si una falla, las otras se cancelan automáticamente
```

---

## 4. Comparativa y Benchmarks

```python
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def io_task():
    time.sleep(0.1)  # Simula I/O

def cpu_task(n):
    return sum(i*i for i in range(n))

def benchmark_io():
    # Threading
    start = time.perf_counter()
    with ThreadPoolExecutor(10) as ex:
        list(ex.map(lambda _: io_task(), range(100)))
    print(f"Threading: {time.perf_counter() - start:.2f}s")
    
    # Asyncio
    async def async_io():
        await asyncio.sleep(0.1)
    
    async def run_async():
        await asyncio.gather(*[async_io() for _ in range(100)])
    
    start = time.perf_counter()
    asyncio.run(run_async())
    print(f"Asyncio: {time.perf_counter() - start:.2f}s")

def benchmark_cpu():
    # Threading (malo para CPU)
    start = time.perf_counter()
    with ThreadPoolExecutor(4) as ex:
        list(ex.map(cpu_task, [10**6]*4))
    print(f"Threading CPU: {time.perf_counter() - start:.2f}s")
    
    # Multiprocessing
    start = time.perf_counter()
    with ProcessPoolExecutor(4) as ex:
        list(ex.map(cpu_task, [10**6]*4))
    print(f"Multiprocessing: {time.perf_counter() - start:.2f}s")
```

### Resultados Típicos

| Tarea | Threading | Asyncio | Multiprocessing |
|-------|-----------|---------|-----------------|
| 100x sleep(0.1) I/O | ~1.0s | ~0.1s | ~1.0s (overhead) |
| 4x sum(1M²) CPU | ~4.0s (GIL) | ~4.0s (GIL) | ~1.0s |

---

## 5. Patrones Híbridos

### 5.1 ThreadPool + ProcessPool

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import asyncio

def cpu_intensivo(datos):
    # Procesamiento pesado
    return procesar(datos)

async def pipeline_async(datos_lista):
    loop = asyncio.get_event_loop()
    
    # 1. I/O: descargar datos (ThreadPool)
    with ThreadPoolExecutor(10) as thread_pool:
        futuros_descarga = [loop.run_in_executor(thread_pool, descargar, url) for url in urls]
        datos_descargados = await asyncio.gather(*futuros_descarga)
    
    # 2. CPU: procesar datos (ProcessPool)
    with ProcessPoolExecutor(4) as process_pool:
        futuros_proceso = [loop.run_in_executor(process_pool, cpu_intensivo, d) for d in datos_descargados]
        resultados = await asyncio.gather(*futuros_proceso)
    
    # 3. I/O: guardar resultados (ThreadPool)
    with ThreadPoolExecutor(5) as thread_pool:
        await asyncio.gather(*[loop.run_in_executor(thread_pool, guardar, r) for r in resultados])
```

### 5.2 Pipeline Streaming

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def etapa_fuente(queue, urls):
    for url in urls:
        await queue.put(await descargar_async(url))
    await queue.put(None)  # fin

async def etapa_proceso(queue_in, queue_out, executor):
    while True:
        item = await queue_in.get()
        if item is None:
            await queue_out.put(None)
            break
        # Ejecutar CPU en executor
        resultado = await asyncio.get_event_loop().run_in_executor(executor, procesar_cpu, item)
        await queue_out.put(resultado)

async def etapa_sink(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        await guardar_async(item)

async def main():
    q1 = asyncio.Queue(maxsize=10)
    q2 = asyncio.Queue(maxsize=10)
    
    with ProcessPoolExecutor(4) as executor:
        await asyncio.gather(
            etapa_fuente(q1, urls),
            etapa_proceso(q1, q2, executor),
            etapa_sink(q2)
        )
```

---

## 6. Testing Concurrente

```python
import pytest
import asyncio
from unittest.mock import AsyncMock, patch

# Test asyncio
@pytest.mark.asyncio
async def test_fetch():
    async with aiohttp.ClientSession() as session:
        with patch('aiohttp.ClientSession.get', new_callable=AsyncMock) as mock_get:
            mock_get.return_value.__aenter__.return_value.text = AsyncMock(return_value="OK")
            result = await fetch(session, "http://test.com")
            assert result == "OK"

# Test threading
def test_thread_safety():
    import threading
    counter = {'value': 0}
    lock = threading.Lock()
    
    def increment():
        for _ in range(1000):
            with lock:
                counter['value'] += 1
    
    threads = [threading.Thread(target=increment) for _ in range(10)]
    for t in threads: t.start()
    for t in threads: t.join()
    
    assert counter['value'] == 10000

# Test multiprocessing
def test_multiprocessing():
    with ProcessPoolExecutor(2) as ex:
        results = list(ex.map(lambda x: x*2, range(5)))
    assert results == [0, 2, 4, 6, 8]
```

---

## 6. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Funciones, context managers
- [Librerías Estándar](./Python%20-%20Librer%C3%ADas%20Est%C3%A1ndar%20Esenciales.md) - Referencia módulos threading/multiprocessing/asyncio
- [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) - I/O asíncrono
- [Redes y HTTP](./Python%20-%20Redes%20y%20HTTP.md) - aiohttp, sockets async
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - Tests concurrentes

---

*Etiquetas: #python #concurrencia #threading #multiprocessing #asyncio #async-await #threadpoolexecutor #processpoolexecutor #concurrent-futures #asyncio-gather #asyncio-semaphore #asyncio-queue #asyncio-lock #gil #io-bound #cpu-bound #concurrencia #paralelismo #aiohttp #aiofiles #asyncpg #uvloop*