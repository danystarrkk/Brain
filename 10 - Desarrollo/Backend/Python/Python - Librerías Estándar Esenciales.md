---
tags:
  - desarrollo
  - backend
  - python
tipo: concepto
estado: terminado
---

# Python - Librerías Estándar Esenciales

Tour curado del stdlib ("batteries included") por dominio, con la pregunta práctica *"¿cuándo alcanza la stdlib y cuándo no?"*, las funciones clave de cada módulo con ejemplos ejecutados (3.14) y recetas por familia. Para profundidad en I/O ver [[Python - Manejo de Archivos y Serialización]] y para el sistema de módulos, [[Python - Módulos, Paquetes y Distribución]].

> [!info] Filosofía de la stdlib
> Python resuelve en stdlib lo que todo programador necesita a diario: archivos, fechas, regex, JSON, CSV, math, processos, argparse, logging, testing. Regla de decisión: **si está en la stdlib y cubre tu caso, no agregues dependencia** — cada dependencia externa es superficie de ataque, mantenimiento y riesgo de rotura de versiones. La stdlib no cubre: HTTP client moderno (usa `requests`/`httpx`), dataframes (pandas), numerico (numpy), web frameworks.

## Tabla maestra: "¿qué módulo uso?"

| Necesidad | Módulo / snippet |
|---|---|
| Ejecutar comando externo | `subprocess.run(cmd, capture_output=True, text=True, check=True)` |
| Parsear fecha string | `datetime.strptime(s, "%Y-%m-%d")` o `fromisoformat` |
| Diferencia entre fechas | `timedelta` / `-` entre datetimes |
| Zona horaria (3.9+) | `zoneinfo.ZoneInfo("America/Argentina/Buenos_Aires")` |
| Timestamp actual | `time.time()` (epoch) / `datetime.now(tz)` |
| Medir duración precisa | `time.perf_counter()` (no `time.time()`: ese salta con NTP) |
| Sleep en async | `asyncio.sleep()` — jamás `time.sleep()` en event loop |
| Token seguro | `secrets.token_urlsafe(32)` |
| Hash de datos/archivo | `hashlib.sha256(data).hexdigest()` / `hashlib.file_digest(f, "sha256")` (3.11+) |
| Hash de contraseña | `hashlib.pbkdf2_hmac` o `bcrypt` (externo, más simple) |
| UUID aleatorio | `uuid.uuid4()` |
| Leer CSV | `csv.DictReader(f)` |
| Escribir JSON | `json.dump(obj, f, ensure_ascii=False, indent=2)` |
| Archivo de config | `tomllib.load(f)` (3.11+) |
| DB embebida | `sqlite3` |
| Directorios | `Path.rglob("*.py")`, `Path.mkdir(parents=True, exist_ok=True)` |
| CLI argumentos | `argparse.ArgumentParser` |
| Logging | `logging.getLogger(__name__)` + config básica |
| Paralelizar CPU | `ProcessPoolExecutor` (ver Concurrencia) |
| Paralelizar I/O | `ThreadPoolExecutor` / `asyncio` |
| Cache de función | `@functools.cache` (sin límite) o `lru_cache(maxsize)` |
| Cola de prioridad | `heapq.heappush/heappop` |
| Búsqueda binaria | `bisect.bisect_left` sobre lista ordenada |
| Contar/a-grupar | `Counter`, `defaultdict(list)`, `itertools.groupby` (input ordenado) |
| Benchmark rápido | `timeit.timeit("expr", "setup", number=10_000)` |

## Sistema y entorno

| Módulo | Uso | Funciones clave |
|---|---|---|
| `os` | SO | `getenv`, `environ`, `getpid`, `walk`, `replace` (atómico) |
| `sys` | intérprete | `argv`, `path`, `exit`, `version_info`, `stdout`, `modules` |
| `pathlib` | rutas OO | ver [[Python - Manejo de Archivos y Serialización]] |
| `shutil` | archivos alto nivel | `copy2`, `copytree`, `rmtree`, `move`, `make_archive` |
| `tempfile` | temporales | `NamedTemporaryFile`, `TemporaryDirectory`, `mkstemp` |
| `subprocess` | procesos | `run`, `Popen`, `check_output`, `DEVNULL`, `timeout` |
| `signal` | señales | `signal(SIGINT, handler)`, `SIGTERM` |
| `platform` | info SO | `system()`, `python_version()` |

### `subprocess`: el reemplazo moderno de `os.system`

```python
import subprocess
r = subprocess.run(["ls", "-la"], capture_output=True, text=True, timeout=30)
r.returncode, r.stdout, r.stderr        # inspeccionar siempre
subprocess.run(cmd, check=True)         # lanza CalledProcessError si rc != 0
```

Reglas: **lista de args, nunca shell=True** (injection — mismo principio de [[Python Ofensivo]]: `shell=True` con string = ejecución de shell arbitrario si entra data externa); `text=True` para str en vez de bytes; `timeout=` obligatorio para comandos de red. Shell solo si necesitás pipes/redirecciones complejas — y entonces con entrada saneada o `shlex.quote`.

## Textos, datos y seguridad ligera

| Módulo | Uso | Funciones clave |
|---|---|---|
| `re` | regex | `search`, `match`, `findall`, `sub`, `compile`, `finditer` |
| `textwrap` | formato | `fill`, `wrap`, `dedent`, `indent`, `shorten` |
| `difflib` | diff | `unified_diff`, `SequenceMatcher`, `get_close_matches` |
| `unicodedata` | Unicode | `normalize("NFKD", s)`, `category` |
| `html` | escape | `escape()`, `unescape()` (XSS en templates manuales) |
| `urllib.parse` | URLs | `urlparse`, `quote`, `urlencode`, `parse_qs`, `urljoin` |
| `base64` | codificación | `b64encode/decode`, `urlsafe_b64encode` |
| `hashlib` | hash | `sha256`, `blake2b`, `pbkdf2_hmac`, `file_digest` (3.11+) |
| `hmac` | MAC | `hmac.compare_digest` (comparación timing-safe) |
| `secrets` | aleatoriedad segura | `token_hex`, `token_urlsafe`, `choice`, `randbelow` |
| `uuid` | identificadores | `uuid4` (aleatorio), `uuid5` (determinista por namespace) |

```python
import re
re.findall(r"\d+", "a1b22c333")        # ['1','22','333']
re.sub(r"\d+", "#", "a1b2")             # 'a#b#'
pat = re.compile(r"^\w+@\w+\.\w+$")     # compilar una vez, usar muchas
pat.search("mail: juan@dom.com")        # match() ancla al inicio; search() en cualquier sitio
```

```python
import secrets, hmac, hashlib
secrets.token_urlsafe(24)                          # tokens de sesión/CSRF
hmac.compare_digest(a, b)                          # comparar MACs sin timing attack
hashlib.pbkdf2_hmac("sha256", pwd.encode(), salt, 600_000)  # KDF de contraseñas
```

> [!warning] `random` vs `secrets`
> `random` es Mersenne Twister: **predecible** si observas suficientes outputs — tokens, sesiones, nonces y sorteos de seguridad con `secrets` (CSPRNG del SO). `random` es correcto para simulación, sampling y tests.

## Estructuras de datos y algoritmos

| Módulo | Uso | Claves |
|---|---|---|
| `collections` | estructuras | `defaultdict`, `Counter`, `deque`, `OrderedDict`, `ChainMap`, `NamedTuple` — ver [[Python - Estructuras de Datos]] |
| `itertools` | iteradores | `chain`, `cycle`, `islice`, `groupby`, `accumulate`, `permutations`, `combinations`, `product`, `zip_longest` |
| `functools` | funciones | `cache`, `lru_cache`, `partial`, `reduce`, `wraps`, `singledispatch`, `cached_property` |
| `operator` | funciones-callables | `itemgetter`, `attrgetter`, `add`, `mul` (más rápidos que lambdas en key) |
| `heapq` | min-heap | `heappush`, `heappop`, `heapify`, `nlargest`, `merge` |
| `bisect` | binaria | `bisect_left/right`, `insort_left/right` (lista ordenada) |
| `array` | array tipado | `array('i', [1,2,3])` — 4 bytes/entero vs 28 de list |
| `weakref` | refs débiles | `WeakValueDictionary` — caches sin retener objetos |
| `copy` | copias | `copy`, `deepcopy` |
| `pprint` | pretty print | `pprint`, `pformat` (tests legibles) |

### Las 6 funciones que valen la pena memorizar

```python
from itertools import groupby, accumulate, chain, zip_longest
from functools import reduce, partial, cache, lru_cache, cached_property

# 1. groupby — agrupa CONSECUTIVOS (¡input ordenado! — verificado)
datos = [("a",1), ("a",3), ("b",2)]
list(groupby(datos, key=lambda t: t[0]))            # 3 grupos si no ordenás: [(a,[..]),(b,..),(a,..)]
[(k, [x[1] for x in g]) for k, g in groupby(sorted(datos), key=lambda t: t[0])]
# verificado: [('a', [1, 3]), ('b', [2])]

# 2. accumulate — acumulados intermedios (running total / max / custom)
list(accumulate([1, 2, 3, 4]))                      # verificado: [1, 3, 6, 10]
list(accumulate([1, 5, 2], max))                    # verificado: [1, 5, 5] — running max
list(accumulate([1,2,3], lambda a, b: a * b))       # acumulado custom

# 3. reduce — colapsa a un solo valor (lee: reduce(func, iterable, inicial))
reduce(lambda a, b: a + b, [1, 2, 3, 4])            # verificado: 10
reduce(lambda a, b: a + b, [], 0)                   # SIEMPRE pasar inicial en riesgo de vacío

# 4. partial — fijar argumentos de una función
from functools import partial
doblar = partial(int, base=2)
doblar("1010")                                      # verificado: 10
# patrones: partial(myfunc, timeout=5), partial(print, end=" ")

# 5. cache / lru_cache — memoización en C
@cache
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)
fib(30)                                             # verificado: 832040, instantáneo
# cache = lru_cache(maxsize=None); lru_cache(128) acota memoria
# ⚠️ solo para funciones puras (mismos args → mismo resultado) y args hasheables

# 6. cached_property (3.8+) — property memoizada por instancia, sin __init__
class Modelo:
    @cached_property
    def pesos(self): return cargar_pesos_grandes()   # calcula 1ª vez, cachea
```

`reduce` casi siempre es más legible como loop o `math.sumprod` (3.12+); mantenlo para composición funcional. `operator.itemgetter(1)` como `key` supera a `lambda r: r[1]` en velocidad de hot loops.

## Fechas, horas y tiempo

| Módulo | Uso | Claves |
|---|---|---|
| `datetime` | fecha/hora | `datetime`, `date`, `time`, `timedelta`, `timezone.utc`, `fromisoformat`, `strftime` |
| `time` | reloj | `time()`, `perf_counter()`, `monotonic()`, `sleep()`, `localtime()` |
| `zoneinfo` (3.9+) | IANA tz | `ZoneInfo("Europe/Madrid")` — sin dependencias en Linux/macOS |
| `calendar` | calendarios | `monthrange`, `isleap`, `weekday` |
| `pytz` (externo) | legacy | solo para código Py2/old — migrar a zoneinfo |

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo
datetime.now(timezone.utc)                                  # aware — SIEMPRE aware
datetime.fromisoformat("2026-06-01T12:00:00")               # parse ISO nativo (3.11+: casi todo ISO)
(datetime(2026,6,1,12, tzinfo=ZoneInfo("America/Argentina/Buenos_Aires")))
# verificado: 2026-06-01 12:00:00-03:00 — tzinfo correcta sin pytz
(datetime.now(timezone.utc) + timedelta(days=7)).date()
```

> [!warning] Naive vs aware
> `datetime.now()` sin tz es **naive**: no sabés qué zona representa → bugs al comparar/restar entre zonas (y `astimezone` asume local). Regla: **todas las fechas aware**, almacená UTC y presentá en tz local. `time.monotonic()` para medir duraciones (inmune a cambios de reloj del SO); `time.time()` solo como epoch storage.

## Matemáticas y números

| Módulo | Uso | Claves |
|---|---|---|
| `math` | matemáticas | `sqrt`, `log`, `sin/cos`, `ceil/floor`, `gcd`, `isclose`, `hypot`, `comb/perm`, `pi`, `e`, `inf`, `nan` |
| `statistics` | estadística | `mean`, `median`, `stdev`, `variance`, `quantiles`, `mode`, `multimode` |
| `decimal` | decimal exacto | `Decimal("0.1")`, `quantize`, `getcontext().prec` |
| `fractions` | racionales | `Fraction(3, 4)`, `limit_denominator` |
| `random` | pseudoaleatorio | `random`, `randint`, `choice`, `sample`, `shuffle`, `seed` |
| `secrets` | CSPRNG | ver arriba |
| `cmath` | complejos | `sqrt`, `phase`, `polar` |

```python
import statistics as st, math
st.mean([1,2,3,4])            # 2.5    st.median([1,3,2]) → 2
st.quantiles([1,2,3,4,5,6,7,8,9], n=4)   # verificado: [2.5, 5.0, 7.5] — terciles
math.isclose(0.1+0.2, 0.3)    # True — la forma correcta de comparar floats
math.comb(5, 2), math.perm(5, 2)         # (10, 20) combinatoria nativa
```

## Concurrencia (resumen — detalle en la nota dedicada)

| Módulo | Modelo | Para qué |
|---|---|---|
| `threading` | hilos + GIL | I/O bloqueante, UI, sync (`Lock`, `Event`, `Condition`) |
| `multiprocessing` | procesos | CPU-bound real (evita GIL); cuidado con pickling |
| `concurrent.futures` | alto nivel | `ThreadPoolExecutor` / `ProcessPoolExecutor` + `as_completed` |
| `asyncio` | event loop | decenas de miles de conexiones I/O, clientes HTTP async |
| `queue` | colas thread-safe | productor-consumidor entre hilos |
| `contextvars` | contexto | locales por task (async) |

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
with ThreadPoolExecutor(10) as ex:
    futuros = {ex.submit(descargar, url): url for url in urls}
    for fut in as_completed(futuros):
        try: resultado = fut.result()
        except Exception as e: manejar(e, futuros[fut])
```

Ver [[Python - Concurrencia Threading Multiprocessing Asyncio]] — GIL, async, y cuándo cada modelo.

## Redes y protocolos

| Módulo | Uso | Claves |
|---|---|---|
| `socket` | TCP/UDP bajo nivel | `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv` |
| `ssl` | TLS | `create_default_context()`, `create_connection` |
| `http.client` | HTTP cliente | `HTTPConnection`, `request`, `getresponse` |
| `urllib.request` | urlopen | `urlopen`, `Request` (básico; preferir `requests`) |
| `http.server` | servidor dev | `python -m http.server 8000` — SOLO desarrollo |
| `ftplib`/`smtplib`/`imaplib` | FTP/email | legacy útil |

El cliente HTTP moderno de la práctica es `requests`/`httpx` (externos); la stdlib da la capa baja. **Ver [[Python - Redes y HTTP]]** para la guía completa de sockets, requests y servidores. Nota de seguridad: nunca armés URLs con f-strings para datos externos — `urllib.parse.quote`/`urlencode` (ver [[Python Ofensivo]] para SSRF/injection de URLs).

## Testing, debugging y calidad

| Módulo | Uso | Claves |
|---|---|---|
| `unittest` | testing xUnit | `TestCase`, `assertEqual`, `assertRaises`, `mock.patch`, `subTest` |
| `doctest` | tests en docstrings | `python -m doctest mod.py -v` |
| `pdb`/`breakpoint()` | debugger | `breakpoint()` (3.7+, honra `PYTHONBREAKPOINT`) |
| `traceback` | errores | `format_exc`, `print_exc`, `format_exception` |
| `logging` | logs | ver receta abajo |
| `warnings` | avisos | `warn`, `filterwarnings`, `simplefilter` |
| `timeit` | micro-benchmark | CLI: `python -m timeit -n 100000 "expr"` |
| `cProfile`/`profile` | profiling | `python -m cProfile -s cumtime script.py` |
| `tracemalloc` | memoria | `start`, `take_snapshot`, `compare_to` — localiza fugas |
| `faulthandler` | crash dumps | `enable()` — traceback ante segfaults/timeout |

```python
import logging
logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(name)s: %(message)s")
log = logging.getLogger(__name__)           # jerarquía por punto — idioma de la stdlib
log.info("arranque puerto=%d", 8080)        # %s lazy: el formateo solo si el nivel pasa
log.warning("lento", extra={"user_id": 123})
```

Reglas de logging: `getLogger(__name__)` en cada módulo, config solo en entry point, nunca `print` para diagnóstico de library, levels = DEBUG/INFO/WARNING/ERROR/CRITICAL, handlers/`basicConfig` en la app y `NullHandler` en librerías. `print` para output **al usuario**, `logging` para observabilidad. Equivalencia con [[Node JS]]: `console.log` ≈ print; `winston` ≈ logging.

```python
# Profile rápido de un script
# python -m cProfile -s cumulative script.py | head -25
# timeit desde shell
# python -m timeit -n 100000 -s "l=list(range(1000))" "l[500]"
```

## Introspección, tipos y metaprogramación

| Módulo | Uso | Claves |
|---|---|---|
| `inspect` | introspección | `signature`, `getsource`, `getmembers`, `isclass` |
| `types` | objetos-tipo | `SimpleNamespace`, `ModuleType`, `MappingProxyType` |
| `importlib` | import dinámico | `import_module`, `reload`, `resources`, `metadata` |
| `dataclasses` | data classes | `dataclass`, `field`, `asdict`, `replace` |
| `typing` | hints | `Protocol`, `TypeVar`, `Literal`, `TypedDict`, `Annotated`, `Never` (3.11) |
| `enum` | enums | `Enum`, `IntEnum`, `Flag`, `StrEnum` (3.11), `auto` |
| `abc` | abstractos | `ABC`, `abstractmethod` |
| `contextlib` | context managers | `contextmanager`, `ExitStack`, `suppress`, `redirect_stdout` |
| `functools` | decoradores | `cached_property`, `singledispatch`, `total_ordering` |
| `argparse` | CLI | ver receta |

```python
from dataclasses import asdict, replace
@dataclass(frozen=True)
class Cfg: host: str; puerto: int = 8080
c = Cfg("localhost")
asdict(c)                                  # {'host': 'localhost', 'puerto': 8080} — a JSON directo
replace(c, puerto=9090)                    # copia con cambio (¡la vía immutable!)

from typing import Protocol                # interfaces estructurales sin herencia
class Cerrable(Protocol):
    def close(self) -> None: ...
```

### `argparse`: la CLI de la stdlib

```python
import argparse
p = argparse.ArgumentParser(description="Ejemplo")
p.add_argument("archivo")                       # posicional obligatorio
p.add_argument("-v", "--verbose", action="store_true")
p.add_argument("-n", "--repeticiones", type=int, default=3)
p.add_argument("--salida", type=Path, default=Path("out.txt"))
args = p.parse_args()                           # --help automático, errores con uso
```

Valida tipos, genera ayuda y errores de uso sin código. Para CLIs complejas el estándar externo es `click` (composición, testing) — la mencionó la nota original como alternativa; `typer` lo envuelve con type hints. argparse es correcto para el 90% de los scripts.

## Compresión y archivos

| Módulo | Formato | Claves |
|---|---|---|
| `zlib` | raw zlib | `compress`, `decompress`, `crc32` |
| `gzip` | .gz | `open()`, `GzipFile`, `compress` |
| `bz2` | .bz2 | `open`, `compress` |
| `lzma` | .xz | `open`, `LZMAFile` |
| `zipfile` | .zip | `ZipFile`, `extractall` ⚠️ sanitizar rutas (zip-slip), `writestr` |
| `tarfile` | .tar/.gz | `open`, `getmembers` ⚠️ `data_filter="tar"` (3.12+: filtrado anti path-traversal por defecto) |
| `shutil` | atajo | `make_archive`, `unpack_archive` |

```python
import gzip
with gzip.open("datos.json.gz", "rt", encoding="utf-8") as f:   # 'rt' = texto
    data = json.load(f)
```

## Recetas por familia

```python
# ── Texto/regex ──────────────────────────────────────
import re, unicodedata, textwrap
def slug(txt):
    txt = unicodedata.normalize("NFKD", txt).encode("ascii", "ignore").decode()
    return re.sub(r"[^a-z0-9]+", "-", txt.lower()).strip("-")

# ── Fechas ───────────────────────────────────────────
def proximo_lunes(dt):                      # datetime → siguiente lunes
    from datetime import timedelta
    return dt + timedelta(days=(7 - dt.weekday()) % 7 or 7)

# ── Estructuras ──────────────────────────────────────
from collections import Counter, defaultdict
top3 = Counter(palabras).most_common(3)
por_tipo = defaultdict(list)
for it in items: por_tipo[it.tipo].append(it)

# ── Procesos ─────────────────────────────────────────
def run_seguro(cmd, timeout=30):
    import subprocess
    return subprocess.run(cmd, capture_output=True, text=True, timeout=timeout, check=True)

# ── Aleatoriedad ─────────────────────────────────────
import random, secrets
muestra = random.sample(poblacion, k=10)     # sin reemplazo, unbiased
token = secrets.token_urlsafe(32)

# ── Heap: top-K de un stream gigante ────────────────
import heapq
top_k = []
for item in stream:
    heapq.heappush(top_k, (item.score, item))
    if len(top_k) > 50: heapq.heappop(top_k)   # O(n log k) con k=50, memoria O(k)

# ── DB local ─────────────────────────────────────────
import sqlite3
with sqlite3.connect("app.db", isolation_level=None) as conn:   # autocommit
    conn.execute("CREATE TABLE IF NOT EXISTS eventos (ts TEXT, tipo TEXT, payload TEXT)")

# ── HTTP mínimo (dev) ────────────────────────────────
# python -m http.server 8000 --bind 127.0.0.1   → ver [[Python - Redes y HTTP]]
```

## Rendimiento del stdlib

- Los builtins en C (`sum`, `min/max(key=)`, `sorted`, `list.extend`, `str.join`) ganan a loops Python equivalentes — `"".join(map(str, datos))` sobre `+` acumulado.
- `timeit` para decidir dudas reales: compará con el **mismo** setup y número de iteraciones; desconfía de micro-diferencias <10%.
- Perfilá antes de optimizar: `cProfile -s cumulative` te muestra que el 90% del tiempo suele estar en 2-3 funciones (o en I/O). Para memoria: `tracemalloc` con snapshots.
- Colecciones correctas primero: la diferencia set-vs-list (2500× verificado en [[Python - Estructuras de Datos]]) opaca cualquier otro micro-ahorro.

## Errores comunes

1. **`time.time()` para medir duraciones** — usa `perf_counter`/`monotonic`; el reloj de sistema salta con correcciones NTP.
2. **`datetime.now()` naive** — crea bug de zonas; siempre `timezone.utc` o `ZoneInfo`.
3. **`random` para seguridad** — predecible; `secrets` para tokens.
4. **`groupby` sin ordenar** — grupos duplicados (verificado).
5. **`subprocess` con `shell=True` + f-string** — injection de shell; lista de args.
6. **`logging` con f-strings** — formatea aunque el nivel lo descarte; usa `%s` lazy.
7. **`heapq` con objetos sin `__lt__`** — `TypeError`; emparejá `(prioridad, objeto)` o implementá comparación.
8. **`csv` sin `newline=""`** — `\r\r\n` en Windows (ver Manejo de Archivos).
9. **`pip install` de lo que ya trae la stdlib** — verificá con esta nota antes: duplicados generan conflictos de import.
10. **`yaml.load` / `pickle.load` de datos ajenos** — ejecución arbitraria; JSON/TOML-safe.

## Compatibilidad y versiones (notas clave)

| Versión | Aportes relevantes del stdlib |
|---|---|
| 3.8 | `walrus :=`, `math.prod`, `importlib.metadata`, `breakpoint` |
| 3.9 | `zoneinfo` (IANA nativo), `str.removeprefix/suffix`, dict `|` |
| 3.10 | `match`, `zip(strict=)`, `TypeAlias`, `graphlib.TopologicalSorter` |
| 3.11 | `tomllib`, `datetime.fromisoformat` completo, `ExceptionGroup`/`except*`, `StrEnum`, `Self`, `TaskGroup` |
| 3.12 | `type` alias, `itertools.batched` (3.12), `Path.walk`, f-strings libres, `@override` |
| 3.13+ | `copy.replace`, `ast.unparse` mejorado, módulo `mimalloc`, JIT experimental, free-threading preview |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — semántica base para todos los ejemplos
- [[Python - Estructuras de Datos]] — collections en profundidad
- [[Python - Manejo de Archivos y Serialización]] — json/csv/toml/sqlite/pathlib detallados
- [[Python - Módulos, Paquetes y Distribución]] — cómo se distribuye/consume stdlib y externos
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — los modelos de concurrencia completos
- [[Python - Redes y HTTP]] — sockets, requests, servidores
- [[Python - Testing, Calidad y Herramientas]] — pytest, ruff, mypy, CI
- [[Python - Referencia Rápida (Cheatsheet)]] — la tarjeta de una página
- [[Comandos Básicos de Sistema y Red en Linux]] — el lado shell equivalente
