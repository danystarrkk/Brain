---
aliases: ["Concurrencia Python", "Threading Python", "Multiprocessing Python", "Asyncio", "GIL"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
  - "[[Python - Redes y HTTP]]"
  - "[[Python - Testing, Calidad y Herramientas]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python - Programación Orientada a Objetos]]"
fuente: "https://docs.python.org/es/3/library/concurrency.html"
---

# Python - Concurrencia: Threading, Multiprocessing y Asyncio

Referencia definitiva de los tres modelos de concurrencia de Python: **threads** (GIL compartido), **procesos** (paralelismo real) y **asyncio** (event loop cooperativo). Cubre el GIL en profundidad, primitivas de sincronización, cuándo y por qué cada modelo gana, recetas de producción y errores clásicos. Todo lo que sigue fue verificado con **Python 3.14.7** en Linux (20 núcleos) a menos que se indique otra versión.

> [!info] Concurrencia ≠ paralelismo
> **Concurrencia** es estructurar tu programa para atender varias tareas *intercaladas* (puede ser una sola CPU cambiando de tarea). **Paralelismo** es ejecutarlas *a la vez* en hardware distinto. Threads dan concurrencia (y paralelismo solo en I/O); procesos dan paralelismo real; asyncio da concurrencia cooperativa de un solo hilo. La pregunta de diseño no es "¿quiero concurrencia?" sino **"¿dónde espera mi programa?"**.

## Tabla de decisión (la respuesta al 90% de los casos)

| Tipo de tarea | Modelo | Módulo | Por qué funciona |
|---|---|---|---|
| **I/O-bound** (red, disco, BD, APIs) | Threads o Asyncio | `threading`, `concurrent.futures`, `asyncio` | El GIL **se libera durante la espera de I/O**: otro hilo corre mientras uno espera |
| **CPU-bound** (cálculo, compresión, ML, imagen) | **Multiprocessing** | `multiprocessing`, `ProcessPoolExecutor` | Cada proceso tiene su **propio GIL** → paralelismo real entre núcleos |
| **Mixto** (I/O + CPU) | Híbrido | `run_in_executor` con ambos pools | Cada fase con el modelo que le corresponde |
| **Alta concurrencia I/O** (10k+ conexiones) | **Asyncio** | `asyncio`, `aiohttp`, `asyncpg` | Un solo hilo, sin costo de cambio de contexto por tarea |
| **Cron/lotes simples** | Ninguno (serial) | — | Un script secuencial de 20 líneas casi siempre gana a una versión "concurrente" |

> [!tip] Regla de oro
> Si tu programa **espera**, usá threads o asyncio. Si tu programa **calcula**, usá procesos. Si dudás, medí: ver "Benchmark verificado" más abajo.

## El GIL en profundidad

### Qué es y qué serializa exactamente

El **GIL** (*Global Interpreter Lock*, `sys._is_gil_enabled()`) es un mutex dentro de CPython que protege el acceso al **contador de referencias del interpreter state**. Su efecto práctico: **solo un hilo ejecuta bytecode de Python a la vez**, aunque tengas 32 núcleos.

Lo que el GIL **sí** serializa:

- Cada instrucción del bytecode de Python (Aritmética, atributos, llamadas).
- Los incrementos/decrementos de referencia que acompañan a cada objeto.

Lo que el GIL **no** serializa (threads sí corren en paralelo aquí):

- **Esperas de I/O**: `socket.recv`, `time.sleep`, `input()`, lecturas de disco, `requests.get` — el hilo que espera **suelta el GIL**.
- **Operaciones C que lo liberan explícitamente**: extensiones como NumPy (`numpy.dot` en arrays grandes), compresión (`zlib`, `lzma`), `hashlib` con buffers grandes, SQLite.
- Acceso a datos que cada hilo posee de forma privada.

> [!warning] "Threads no sirven para CPU" es una simplificación
> Threads *pueden* ayudar en CPU-bound **si el trabajo pesado vive en C que libera el GIL** (NumPy, zlib). Lo que no pueden es paralelizar puro bytecode Python. El benchmark verificado más abajo muestra la diferencia: 4 hilos de Python puro ≈ mismo tiempo que serial; 4 procesos ≈ proporcional a los núcleos.

### Estado del GIL hoy: 3.13+ y el free-threading

- **CPython 3.13** (2024) añadió el modo **free-threaded** (*no-GIL*, PEP 703): una build opcional (`python3.13t`) donde el GIL queda deshabilitado y los threads de Python corren **realmente en paralelo**. Se activa/desactiva en runtime con `sys._is_gil_enabled()`.
- **CPython 3.14** consolidó el modo: builds `python3.14t` disponibles en los instaladores oficiales; la PEP 776 propone formalizar la semántica. **Verificado en esta máquina**: `sys._is_gil_enabled()` → `True` (build estándar con GIL).
- En free-threading, el costo por hilo sube ligeramente y algunas bibliotecas C todavía asumen el GIL — revisar compatibilidad antes de migrar.
- **Jython/IronPython** no tienen GIL (no son CPython); **PyPy** sí tiene uno. "Python" en esta nota = CPython.

```python
import sys
print(sys._is_gil_enabled())   # True en build estándar (verificado: True)
# python3.14t → False (build free-threaded)
```

### Benchmark verificado (20 núcleos, Python 3.14)

Ejecutado en /tmp con `concurrent.futures`:

| Trabajo | Serial | Threading | Asyncio | Procesos |
|---|---|---|---|---|
| **CPU**: 4 × `sum(i*i for i in range(2_000_000))` | ~0.55 s (impl.) | **0.44 s** (GIL: no gana) | — (no sirve) | **0.19 s** (paralelismo real) |
| **I/O simulada**: 100 × `sleep(0.05)` | 5.0 s (impl.) | 0.26 s (20 workers) | **0.05 s** | ~1.0 s (overhead de proceso) |

Lectura: en CPU, 4 hilos apenas mejoran el serial (pelean por el GIL) y los procesos ganan ~2.3× en esta máquina; en I/O, asyncio colapsa 5 segundos de espera en 50 ms reales — el event loop no paga el costo de 100 hilos del sistema operativo.

---

## Threading: `threading`

### Modelo

Un thread es un hilo del **sistema operativo** ejecutando código Python, compartiendo el mismo heap (objetos, memoria) y el mismo GIL. El scheduler del SO interrumpe hilos cada ~5 ms de Python puro (o antes si el código libera el GIL).

### Crear y coordinar threads

```python
import threading
import time

def tarea(nombre, duracion):
    print(f"{nombre} iniciando")
    time.sleep(duracion)          # libera el GIL: otro hilo puede correr
    print(f"{nombre} finalizado")

threads = [threading.Thread(target=tarea, args=(f"Thread-{i}", 1)) for i in range(3)]
for t in threads:
    t.start()                     # el SO arranca el hilo
for t in threads:
    t.join()                      # el hilo principal espera (también libera el GIL)
print("Todos terminaron")
```

| Método / atributo | Significado |
|---|---|
| `Thread(target, args, kwargs)` | El callable y sus argumentos (¡no `tarea(1)`! — eso lo ejecutaría ya) |
| `t.start()` | Crea el hilo del SO y lo arranca. **Dos `start()` sobre el mismo Thread = `RuntimeError`** |
| `t.join(timeout)` | Espera; con timeout devuelve `None` y podés consultar `t.is_alive()` |
| `t.name`, `t.ident` | Nombre legible e identificador del hilo del SO |
| `t.daemon = True` | Hilo demonio: Python **no espera** a los daemons al terminar el main; se mata al salir |

> [!warning] Daemons y `finally`
> Un daemon muere **sin ejecutar limpieza garantizada**: un daemon con un `finally: archivo.close()` puede perder el flush si el main sale mientras escribe. Los hilos que tocan datos siempre `daemon=False` + `join()`. Daemon solo para tareas de fondo "best effort" (pings, watchdogs).

### `concurrent.futures.ThreadPoolExecutor` (la API recomendada)

Envuelve el ciclo crear/lanzar/join en una interfaz de futuros:

```python
import os
from concurrent.futures import ThreadPoolExecutor, as_completed

def descargar(url):
    resp = requests.get(url, timeout=10)   # ver [[Python - Redes y HTTP]]
    return url, len(resp.content)

urls = ["https://httpbin.org/delay/1", "https://httpbin.org/delay/2"]

# Forma 1: as_completed → resultados en el orden en que terminan
with ThreadPoolExecutor(max_workers=min(32, os.cpu_count() + 4)) as ex:
    futuros = {ex.submit(descargar, u): u for u in urls}
    for fut in as_completed(futuros):
        try:
            print(fut.result())            # re-lanza la excepción del hilo aquí
        except Exception as e:
            print(f"{futuros[fut]} falló: {e}")

# Forma 2: map → resultados en orden de entrada (espera al más lento)
with ThreadPoolExecutor(max_workers=8) as ex:
    resultados = list(ex.map(descargar, urls))
```

- `max_workers` por defecto era `min(32, cpu+4)`; desde 3.8 es un valor fijo de `min(32, ...)`. Con I/O muchos workers ayudan; con GIL no.
- `future.result()` **propaga la excepción del hilo** — no hay que cazar nada dentro del target.
- `executor.map(fn, iterable, chunksize=...)`: chunksize solo aplica a `ProcessPoolExecutor`.

### Primitivas de sincronización

| Primitiva | Para qué | Nota clave |
|---|---|---|
| `Lock` | Mutua exclusión: una sección crítica | `with lock:` — no olvidar el `with` |
| `RLock` | Lock reentrante: el mismo thread puede adquirirlo 2× | Sin él, un método `with lock` que llama a otro `with lock` del mismo lock **se auto-bloquea** |
| `Semaphore(n)` | Máximo `n` concurrentes | Ideal para limitar conexiones/requests |
| `BoundedSemaphore` | Semaphore que no puede pasar de su tope inicial | Captura errores de release extra |
| `Event` | Señal booleana: `set()` / `wait()` / `clear()` | Un productor avisa a N consumidores |
| `Condition` | Esperar a una condición compleja con `wait()` / `notify()` | Siempre esperar en `while`, nunca en `if` (spurious wakeups) |
| `Barrier(n)` | N hilos esperan a estar todos antes de continuar | Fases sincronizadas; `broken=True` si un hilo aborta |
| `Queue` | **La forma correcta de compartir datos** entre hilos | FIFO thread-safe; productor-consumidor sin locks manuales |

#### Race condition trabajado (verificado)

```python
import threading, time
x = [0]

def worker():
    val = x[0]          # leer
    time.sleep(0.0005)  # la ventana peligrosa: otro hilo lee el mismo val
    x[0] = val + 1      # escribir

ts = [threading.Thread(target=worker) for _ in range(5)]
for t in ts: t.start()
for t in ts: t.join()
print(x[0])             # VERIFICADO: 2  (esperado: 5 — 3 incrementos perdidos)
```

El patrón **check-then-act** (`leer → decidir → escribir`) es la race condition #1. La solución: la sección crítica completa bajo `with lock:`.

```python
counter = 0
lock = threading.Lock()

def bump():
    global counter
    for _ in range(50_000):
        with lock:
            counter += 1        # suma y asignación atómicas entre hilos

ts = [threading.Thread(target=bump) for _ in range(4)]
for t in ts: t.start()
for t in ts: t.join()
print(counter)  # VERIFICADO: 200000 exacto
```

> [!info] Por qué `counter += 1` sin lock a veces "funciona"
> `+=` es LOAD_ATTR/STORE_ATTR en varias instrucciones de bytecode y **debería** perder actualizaciones, pero la ventana es tan pequeña que con el GIL y pocos hilos puede sobrevivir (observado: 800000/800000 en una corrida). **No confíes en la ausencia de síntomas**: el bug es intermitente por diseño. La demostración determinista es el check-then-act de arriba.

#### Productor-consumidor con `Queue` (patrón recomendado)

```python
import threading
from queue import Queue, Empty

q: Queue[int] = Queue(maxsize=100)   # limita el buffer: backpressure

def productor():
    for i in range(50):
        q.put(i)
    q.put(None)                       # sentinela de fin (o q.join() + sentinel)

def consumidor():
    while True:
        item = q.get()
        if item is None:
            q.task_done()
            break
        procesar(item)
        q.task_done()

ts = [threading.Thread(target=productor), threading.Thread(target=consumidor)]
for t in ts: t.start()
for t in ts: t.join()
```

`Queue.get(timeout=)` lanza `queue.Empty`; `Queue.put(timeout=)` lanza `queue.Full`. `task_done()`/`join()` permiten esperar al vaciado total.

#### Condition: productor con "datos listos"

```python
cond = threading.Condition()
datos_listos = False

def consumidor():
    with cond:
        while not datos_listos:      # while, no if: puede despertar sin señal
            cond.wait()              # libera el lock y duerme; al volver lo re-adquiere
        print("Datos procesados")

def productor():
    global datos_listos
    with cond:
        datos_listos = True
        cond.notify_all()
```

#### Thread-local storage

```python
local = threading.local()

def procesar(request_id):
    local.request_id = request_id    # cada hilo tiene su propio atributo
```

Útil para conexiones por hilo (un `Connection` por worker sin exponerlo a otros hilos).

### Deadlock: por qué ocurre y cómo evitarlo

Dos hilos adquieren A y B en orden inverso → ambos esperan para siempre:

```python
# ❌ el bug
lock_a, lock_b = threading.Lock(), threading.Lock()
def hilo1():
    with lock_a:
        with lock_b: ...
def hilo2():
    with lock_b:      # orden inverso
        with lock_a: ...
```

Reglas anti-deadlock:

1. **Orden total de adquisición**: todos los hilos toman los locks en el mismo orden global (nombres/ids → ordenar).
2. **Un solo lock** donde sea posible (más simple y casi siempre más rápido que dos).
3. **`with lock(timeout)`**: `lock.acquire(timeout=5)` devuelve `False` en vez de colgarse — mejor fallar ruidoso que colgado.
4. **Nunca bloquear dentro de un lock**: no `sleep`, I/O ni `join()` de otro hilo con el lock puesto.
5. `threading.active_count()` + `faulthandler` para diagnosticar: `python3 -X faulthandler` y `faulthandler.dump_traceback()` cuelga el stack de todos los hilos.

---

## Multiprocessing: `multiprocessing`

### Modelo y el guard de `__main__`

Cada `Process` es un **proceso del SO** con su propio intérprete, su propio GIL y su propio heap. La comunicación entre ellos es por **pickle** (argumentos, retornos, queues).

> [!warning] El guard `if __name__ == "__main__":` ahora es obligatorio EN TODOS los SO
> Clásicamente se decía "obligatorio en Windows". **Verificado en Python 3.14/Linux**: el método de arranque por defecto en Linux **cambió de `fork` a `forkserver`** (`multiprocessing.get_start_method()` → `'forkserver'`), y tanto `spawn` como `forkserver` **re-importan el módulo principal en cada proceso hijo**. Un script con `Process` sin guard falla en Linux con:
> `RuntimeError: An attempt has been made to start a new process before the current process has finished its bootstrapping phase.` (observado exactamente así al correr un benchmark sin guard).
> **Siempre**: todo lanzamiento de procesos dentro de `if __name__ == "__main__":`.

### `spawn` vs `fork` vs `forkserver`

| Método | Cómo crea el hijo | Velocidad | Seguridad de estado | Default |
|---|---|---|---|---|
| `fork` | Copia el proceso con `os.fork()` (COW) | Instantáneo | Hereda todo: locks, threads vivos, estado importado → **puede heredar el lock equivocado** | Linux ≤3.13 |
| `spawn` | Nuevo intérprete; re-importa `__main__` y **picklea** lo necesario | Lento (~ms) | Limpio; exige guard + serializabilidad | Windows, macOS |
| `forkserver` | Un servidor ligero (ya arrancado) hace fork de procesos limpios | Medio | Casi limpio como spawn, más rápido | **Linux 3.14+** |

Cambiar de método: `multiprocessing.set_start_method("spawn")` o por contexto `multiprocessing.get_context("spawn")`. **Regla práctica**: si tu código funciona con `spawn`, funciona con todos; escribí para `spawn` (funciones importables en el top-level, nada de `lambda`/closures como `target`).

### `Pool` vs `ProcessPoolExecutor`

```python
import multiprocessing
from concurrent.futures import ProcessPoolExecutor

def cpu_intensivo(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    # Pool clásico: imap/imap_unordered, apply_async, initializer
    with multiprocessing.Pool(processes=4) as pool:
        print(pool.map(cpu_intensivo, [10**6] * 4))
        for r in pool.imap_unordered(cpu_intensivo, [10**6] * 4):
            print("listo:", r)                    # en orden de terminación

    # ProcessPoolExecutor (API moderna, unificada con threads)
    with ProcessPoolExecutor(max_workers=4) as ex:
        primos = list(ex.map(es_primo, range(100_000, 101_000)))
```

| API | Mejor para | Extras |
|---|---|---|
| `multiprocessing.Pool` | Pipelines de datos masivos | `imap_unordered`, `maxtasksperchild` (recicla procesos que fingen memoria), `initializer` |
| `ProcessPoolExecutor` | Código que ya usa `Future`/`as_completed` | `map`, `submit`, `run_in_executor` desde asyncio |

### Comunicación: Queue, Pipe, shared memory, Manager

```python
import multiprocessing as mp

# 1) Queue: FIFO entre procesos (diferente de queue.Queue de hilos)
def productor(q):
    for i in range(5):
        q.put(i)
    q.put(None)

if __name__ == "__main__":
    q = mp.Queue()
    p = mp.Process(target=productor, args=(q,))
    p.start()
    while (item := q.get()) is not None:
        print("recibido:", item)
    p.join()

# 2) Pipe: dos extremos, bidireccional, más rápido para 2 procesos
def hijo(conn):
    conn.send([42, None, "hola"])   # pickled
    conn.close()

if __name__ == "__main__":
    padre, hija = mp.Pipe()
    p = mp.Process(target=hijo, args=(hija,))
    p.start()
    print(padre.recv())   # [42, None, 'hola']
    p.join()

# 3) Memoria compartida: velocidad bruta, tipada C
def incrementar(contador):
    for _ in range(10_000):
        with contador.get_lock():
            contador.value += 1

if __name__ == "__main__":
    contador = mp.Value("i", 0)                 # 'i' = int C
    arr = mp.Array("d", [0.0] * 100)            # array de doubles
    ps = [mp.Process(target=incrementar, args=(contador,)) for _ in range(4)]
    for p in ps: p.start()
    for p in ps: p.join()
    print(contador.value)                       # 40000

# 4) Manager: objetos "vivos" vía servidor RPC (lento pero rico)
if __name__ == "__main__":
    with mp.Manager() as mgr:
        d = mgr.dict()          # también list(), namespace(), Lock(), Queue()...
        d["clave"] = "valor"
        print(dict(d))
```

| Primitiva | Velocidad | Tipos soportados | Uso típico |
|---|---|---|---|
| `Value`/`Array` | Muy rápida | Escalares C y arrays | Contadores, buffers numéricos |
| `raw_memory`/`from_buffer` | M máxima | Bytes crudos | IPC de alto rendimiento |
| `Queue`/`Pipe` | Rápida (pipe del SO) | **Cualquier picklable** | Mensajes, jobs |
| `Manager.*` | Lenta (RPC por socket) | Objetos ricos | Estado complejo compartido (pocos accesos) |

> [!warning] Memoria compartida y endianness/alineación
> `mp.Value("i", 0)` crea un `ctypes.c_int` compartido — su tamaño depende de la plataforma. Para binarios portables usá `mp.Array(ctypes.c_uint32, n)` con tipos de ancho fijo. Y el lock de `get_lock()` protege **una** operación; las compuestas (leer-comparar-escribir) necesitan el lock alrededor de todo.

### Restricciones de pickling (por qué tu `Process` falla)

Todo lo que cruce el límite del proceso (target, args, Queue, Pipe, retornos de Pool) debe ser **picklable**:

```python
import pickle
def no_picklable():
    pass

# ❌ fallan al picklear:
lambda x: x              # <lambda>
def outer():
    def inner(): ...     # closure local
    return inner
class ConAtributo:
    conn = abrir_socket()  # sockets, locks, generadores, conexiones: no picklables
```

Mensaje de error canónico: `AttributeError: Can't pickle local object 'outer.<locals>.inner'`. Soluciones: funciones de nivel módulo, `functools.partial` para args, o pasar datos crudos y reconstruir objetos dentro del hijo.

---

## Asyncio: `asyncio`

### Modelo: un hilo, un event loop, cooperación

Asyncio ejecuta **corutinas** sobre un **event loop** de un solo hilo. El loop mantiene una cola de callbacks y de **tareas** (corutinas envueltas); cada `await` es un punto de **cesión cooperativa**: la tarea se suspende registrando su interés en un evento (timer, socket, subprocess) y el loop ejecuta la siguiente tarea lista.

| Concepto | Qué es |
|---|---|
| **Corutina** | `async def` + `await`: función que puede pausarse y reanudarse |
| **Event loop** | El scheduler: elige qué tarea corre; único dueño de la I/O asíncrona |
| **Task** | Corutina "en vuelo" (`create_task`); tiene estado, resultado, cancelación |
| **Future** | resultado aún no disponible (las Tasks son Futures) |
| **Awaitable** | Task, Future o corutina (esperar una corutina cruda **sin** crear task es un anti-patrón para concurrencia) |

```python
import asyncio

async def tarea(nombre, duracion):
    print(f"{nombre} iniciando")
    await asyncio.sleep(duracion)   # cede el loop: NO bloquea
    return f"Resultado de {nombre}"

async def main():
    resultados = await asyncio.gather(
        tarea("A", 1), tarea("B", 2), tarea("C", 1)
    )   # → ['Resultado de A', 'Resultado de B', 'Resultado de C'] en ~2s, no 4s
    print(resultados)

asyncio.run(main())   # crea el loop, corre main, cierra todo. ÚNICO entry point
```

`async/await` semantics:

- `await x` solo es legal **dentro** de `async def` (fuera: `SyntaxError`).
- `await` de una **corutina** cede el control en sus puntos internos; `await task` espera a que termine la tarea completa.
- `asyncio.run()` **cierra el loop** al terminar: no se puede llamar dos veces ni usar hilos que crearon loops distintos.

### `gather` vs `wait` vs `TaskGroup` (verificado)

```python
async def t(name="t"): return name

async def demo():
    # gather: retorna la LISTA DE RESULTADOS en orden; por defecto se detiene... no:
    # return_exceptions=False PROPAGA la primera excepción (las demás se cancelan)
    print(await asyncio.gather(t(1), t(2)))          # [1, 2]

    # wait: retorna (done, pending) como SETS de tasks; no da resultados
    done, pending = await asyncio.wait(
        [asyncio.create_task(t("a")), asyncio.create_task(t("b"))],
        return_when=asyncio.FIRST_COMPLETED,
    )                                                # done=2, pending=0 (verificado)

    # ❌ VERIFICADO en 3.14: pasar corutinas crudas a wait()
    try:
        await asyncio.wait([t()])
    except TypeError as e:
        print(e)   # "Passing coroutines is forbidden, use tasks explicitly."

asyncio.run(demo())
```

| Herramienta | Retorna | Excepciones | Cuándo |
|---|---|---|---|
| `gather(*aws)` | Lista ordenada de resultados | `return_exceptions=False` (default): propaga la primera | El 90% de los casos: "haz todo y dame los resultados" |
| `wait(tasks, return_when=)` | `(done, pending)` sets | **No** lanza: revisá `future.result()` | FIRST_COMPLETED (cuelga de timeouts), ALL_COMPLETED |
| `TaskGroup()` (3.11+) | `None` (acceso vía task objects) | `ExceptionGroup` al salir del `async with`; cancela hermanas | Cuando querés fallos agregados tipados |
| `wait_for(aw, timeout)` | Resultado | `TimeoutError` (builtin desde 3.11) | Timeout simple de una operación |
| `shield(aw)` | Resultado | Protege de la cancelación externa (el task original se cancela igual) | Limpieza crítica que no debe morir con el grupo |

```python
# TaskGroup: la forma moderna de manejar errores en conjunto
async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fase1())
        t2 = tg.create_task(fase2())
    # al salir: si ambas ok, seguís; si una falla, la otra se CANCELA y
    # recibís ExceptionGroup con todos los errores
```

### Primitivas asíncronas y Queue

```python
event = asyncio.Event()        # set()/wait()/clear() async
lock = asyncio.Lock()          # async with lock: ...
cond = asyncio.Condition()     # async with cond: await cond.wait()
sem = asyncio.Semaphore(10)    # limitar concurrencia: async with sem: ...
barrier = asyncio.Barrier(3)   # VERIFICADO: existe desde 3.11

queue = asyncio.Queue(maxsize=100)   # backpressure con maxsize

async def productor():
    for i in range(10):
        await queue.put(i)
    await queue.join()               # espera a task_done() de todos

async def consumidor():
    while True:
        item = await queue.get()     # cede el loop mientras espera
        try:
            await procesar(item)
        finally:
            queue.task_done()
```

> [!info] `asyncio.Queue` vs `queue.Queue`
> Son clases distintas: `queue.Queue` bloquea el thread (útil entre hilos); `asyncio.Queue` suspende la tarea (útil dentro del loop). Usar la del módulo equivocado = bloquear el event loop entero.

### Generadores y context managers asíncronos

```python
# Async generator (3.6+): yield dentro de async def
async def stream(url, session):
    async with session.get(url) as resp:
        async for chunk in resp.iter_content(8192):
            yield chunk

async for chunk in stream(u, s): ...

# Context manager asíncrono: __aenter__/__aexit__
class ConexionDB:
    async def __aenter__(self):
        self.conn = await conectar()
        return self.conn
    async def __aexit__(self, exc_type, exc, tb):
        await self.conn.close()

# La forma decorada (contextlib, 3.10+ soporta async generator variantes)
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

### Errores clásicos de asyncio

```python
# ❌ 1. Llamada bloqueante en el loop: congela TODAS las tareas
async def mala():
    time.sleep(5)                     # bloquea el loop 5s
    requests.get(url)                 # bloquea la red 5s
# ✅ fuera de banda:
async def buena():
    await asyncio.sleep(5)                                # I/O async nativa
    loop = asyncio.get_running_loop()
    resp = await loop.run_in_executor(None, requests.get, url)  # en thread
    # o todo el trabajo pesado: await loop.run_in_executor(pool, calcular, datos)

# ❌ 2. Olvidar await: la corutina NUNCA se ejecuta (solo crea el objeto)
asyncio.create_task(tarea())     # ok
tarea()                          # RuntimeWarning: coroutine was never awaited

# ❌ 3. get_event_loop() fuera de loop — VERIFICADO en 3.14:
asyncio.get_event_loop()
# RuntimeError: There is no current event loop in thread 'MainThread'.
# (antes emitía DeprecationWarning; ahora falla). Usá asyncio.run().

# ❌ 4. Llamar asyncio.run() desde código ya async (crea y destruye loops)
# ❌ 5. await en callbacks de GUI/senales que no son async: re-escribir la cadena
```

### Cancelación y timeouts

```python
async def trabajador():
    try:
        await asyncio.sleep(3600)
    except asyncio.CancelledError:    # hereda de BaseException: no lo trague un except Exception
        print("limpiando...")
        raise                         # DEBE re-lanzarse (3.8+: cancelar es una cancelación)
    finally:
        await cerrar_recursos()       # cierre cooperativo

task = asyncio.create_task(trabajador())
await asyncio.sleep(1)
task.cancel()                         # cooperativa: solo surte efecto en un await
await task                            # CancelledError aquí si se canceló
```

Timeouts con contexto (3.11+): `async with asyncio.timeout(5): ...` lanza `TimeoutError` y cancela el bloque completo — reemplaza al `wait_for` en código nuevo.

---

## Patrones híbridos y recetas reales

### 1. CPU-bound: procesos siempre

```python
from concurrent.futures import ProcessPoolExecutor

def comprimir(args):  # args picklable
    entrada, salida = args
    with open(entrada, "rb") as fi, open(salida, "wb") as fo:
        fo.write(zlib.compress(fi.read()))

if __name__ == "__main__":
    trabajos = [(f"in{i}.bin", f"out{i}.z") for i in range(8)]
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as ex:
        for _ in ex.map(comprimir, trabajos):
            pass   # reciclar memoria con Pool(maxtasksperchild=1) si crece
```

### 2. Pipeline I/O → CPU → I/O con asyncio como orquestador

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

async def pipeline(urls):
    loop = asyncio.get_running_loop()
    with ThreadPoolExecutor(10) as tpool, ProcessPoolExecutor() as ppool:
        # 1) I/O: descargas concurrentes en threads
        datos = await asyncio.gather(*[
            loop.run_in_executor(tpool, descargar, u) for u in urls])
        # 2) CPU: paralelismo real en procesos
        procesados = await asyncio.gather(*[
            loop.run_in_executor(ppool, transformar, d) for d in datos])
        # 3) I/O: guardado
        await asyncio.gather(*[
            loop.run_in_executor(tpool, guardar, p) for p in procesados])
```

### 3. Límite de concurrencia con semáforo (receta anti-429)

```python
sem = asyncio.Semaphore(10)

async def fetch_limited(session, url):
    async with sem:
        async with session.get(url, timeout=30) as resp:
            return url, resp.status

# Con aiohttp compartiendo UNA sesión:
async with aiohttp.ClientSession() as session:
    resultados = await asyncio.gather(*[fetch_limited(session, u) for u in urls])
```

> [!warning] El bug del `ClientSession` por request
> El patrón duplicado (y erróneo) es crear `aiohttp.ClientSession()` **dentro** de cada `fetch_limited`: abre un pool de conexiones por request y agota file descriptors. **Una sesión por aplicación**, pasada como argumento.

### 4. "Primer ganador" (cualquiera de N fuentes)

```python
done, pending = await asyncio.wait(
    [asyncio.create_task(get(c)) for c in ["cdn-a", "cdn-b"]],
    return_when=asyncio.FIRST_COMPLETED,
)
best = next(iter(done)).result()
for p in pending:
    p.cancel()
```

### 5. Testing de código concurrente

```python
# pytest + pytest-asyncio (asyncio_mode = "auto" en pyproject.toml)
async def test_concurrency_limit():
    sem = asyncio.Semaphore(2)
    ...
# Ver más en [[Python - Testing, Calidad y Herramientas]]

# Hilos: determinismo con joins, no con sleeps
def test_thread_safety():
    counter = {"v": 0}
    lock = threading.Lock()
    def bump():
        for _ in range(1000):
            with lock:
                counter["v"] += 1
    ts = [threading.Thread(target=bump) for _ in range(10)]
    for t in ts: t.start()
    for t in ts: t.join()          # join, NO sleep(1)
    assert counter["v"] == 10_000
```

---

## Alternativas a la biblioteca estándar

| Librería | Modelo | Nota |
|---|---|---|
| **Trio** | Asyncio-like con semántica estricta de cancelación (nurseries) | Más seguro conceptualmente; ecosistema chico |
| **anyio** | Capa de abstracción sobre asyncio **y** trio | Lo que usan `httpx`, `starlette`, `sniffio` para ser portables |
| **uvloop** | Event loop sobre libuv (Node) | `pip install uvloop; asyncio.set_event_loop_policy(...)` — 2-4× más rápido en microbenchmarks; ya no soporta Windows |
| **gevent** | Greenlets (corutinas a nivel bytecode) con patch de sockets | Código sync que se vuelve concurrente; legado |
| **Dask / Ray** | Paralelismo distribuido (múltiples máquinas) | Cuando un proceso no alcanza |

---

## Debugging concurrente

- **Deadlock en producción**: `python3 -X faulthandler`, o `faulthandler.register(signal.SIGUSR1)` → al mandar `kill -USR1` imprime el stack de **todos** los hilos.
- **Asyncio**: `asyncio.run(main(), debug=True)` o `PYTHONASYNCIODEBUG=1` avisa de corutinas sin await, tareas no recolectadas y callbacks lentos (>100 ms).
- **Threads colgados**: `py-spy dump --pid <pid>` (mencionado en [[Python - Testing, Calidad y Herramientas]]) muestra stacks sin detener el proceso.
- **Reproduce races**: `sys.setswitchinterval(1e-6)` acelera los cambios de hilo y hace visibles los bugs latentes (usado en el ejemplo verificado).

---

## Errores comunes

1. **CPU-bound en threads** → no escala; procesos (o espera a las builds free-threaded).
2. **Sin `if __name__ == "__main__":` en multiprocessing** → `RuntimeError` de bootstrap (ahora también en Linux por `forkserver`).
3. **Estado mutable compartido sin lock** → race conditions intermitentes; preferí `Queue` sobre variables compartidas.
4. **`time.sleep` / `requests` dentro de asyncio** → congela el loop entero; `asyncio.sleep`, libs async o `run_in_executor`.
5. **`asyncio.wait([corutina])`** → `TypeError` (3.11+); envolver con `create_task`.
6. **`asyncio.get_event_loop()` fuera de loop** → `RuntimeError` en 3.12+/3.14; usar `asyncio.run()`.
7. **`join()` sin timeout + `daemon=True` para todo** → un hilo colgado congela el shutdown; joins con timeout y alarmas.
8. **Un `ClientSession`/pool por request** → agota conexiones y FDs; uno por aplicación.
9. **Locks olvidados en `finally`** → con `with lock:` es imposible; no usar `acquire()` manual.
10. **Esperar resultados con `sleep` mágico** → usar `join`, `gather`, `wait`, `queue.join()` — la sincronización existe por algo.

---

## Cheatsheet

| Tema | API clave | Nota de una línea |
|---|---|---|
| Thread básico | `Thread(target, args); t.start(); t.join()` | `args` siempre como tupla |
| Lock | `with lock:` | Sección crítica lo más corta posible |
| Señal | `Event.set()/wait()` | Un booleano entre hilos |
| Límite | `Semaphore(n)` | Máximo n concurrentes |
| Datos entre hilos | `queue.Queue(maxsize)` | Productor-consumidor sin locks |
| Pool de hilos | `ThreadPoolExecutor(n)` + `as_completed` | Para I/O-bound |
| Process | `mp.Process` + guard `__main__` | Para CPU-bound; todo picklable |
| Pool de procesos | `ProcessPoolExecutor` / `mp.Pool` | `map` ordenado / `imap_unordered` |
| Memoria compartida | `mp.Value("i", 0)`, `mp.Array` | Rápida; lock por operación |
| Estado rico | `mp.Manager().dict()` | Lenta (RPC), flexible |
| Corutina | `async def` / `await` | Solo dentro de async |
| Runner | `asyncio.run(main())` | Una vez, al inicio |
| Paralelo async | `asyncio.gather(*aws)` | Resultados en orden |
| Primer ganador | `asyncio.wait(FIRST_COMPLETED)` | Cancelar el resto |
| Timeout | `asyncio.timeout(5)` (3.11+) | Cancela el bloque |
| Límite async | `asyncio.Semaphore(n)` | Anti-429/anti-FD |
| Cola async | `asyncio.Queue(maxsize)` | Backpressure |
| Bloqueante en loop | `loop.run_in_executor(pool, fn, arg)` | Puente a threads/procesos |
| Método de arranque | `mp.get_start_method()` | `forkserver` en Linux 3.14+ |
| GIL check | `sys._is_gil_enabled()` | `False` en builds `3.14t` |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — funciones, context managers y excepciones que sostienen todo lo anterior
- [[Python - Librerías Estándar Esenciales]] — `threading`, `multiprocessing`, `asyncio`, `queue` en la tabela de stdlib
- [[Python - Redes y HTTP]] — `aiohttp`, `httpx` async y sockets como consumidores de estos modelos
- [[Python - Manejo de Archivos y Serialización]] — I/O de disco (bloqueante) y pickling (límite de procesos)
- [[Python - Testing, Calidad y Herramientas]] — `pytest-asyncio`, tests deterministas con joins
- [[Python - Patrones, Buenas Prácticas y Tipado]] — inyección de dependencias para inyectar pools
- [[Python - Proyectos Prácticos POO]] — donde se combinan GUI + red + threads en proyectos reales
- [[Python - Interfaces Gráficas (Tkinter)]] — el patrón hilo + Queue para no congelar la interfaz
