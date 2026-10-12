---
aliases: ["Cheatsheet Python", "Python Quick Reference", "Python Referencia"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Estructuras de Datos]]"
  - "[[Python - Programación Orientada a Objetos]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
  - "[[Python - Módulos, Paquetes y Distribución]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Concurrencia Threading Multiprocessing Asyncio]]"
  - "[[Python - Testing, Calidad y Herramientas]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python - Redes y HTTP]]"
  - "[[Python - Interfaces Gráficas (Tkinter)]]"
fuente: "https://docs.python.org/es/3/library/"
---

# Python - Referencia Rápida (Cheatsheet)

Tarjeta de referencia ultra-condensada de todo Python: sintaxis, colecciones, strings+formatos, control de flujo, funciones, POO, stdlib one-liners, idiomas, anti-patrones y línea de versiones. Cada celda apunta a la nota que profundiza. Verificado con Python 3.14.

## Sintaxis esencial

```python
x = 42; x: int = 42                 # tipado dinámico; hints = metadata, no runtime
# comentario    """docstring"""      # docstring = dato; # = solo lector
import os; import os as o; from os import path as p
if __name__ == "__main__": main()    # guard de script
```

| Elemento | Forma | Nota |
|---|---|---|
| Literales | `42 3.14 1+2j "s" 's' """m""" True None` | `1_000`, `0xFF`, `1e3` (float) |
| F-string | `f"{v:.2f} {v!r} {expr()=}"` | `=` refleja (3.8+); specs en Tabla de formatos |
| Concatenar | `"a" "b"` (literal), `"a"+"b"` (runtime) | adjacency solo en literales |
| Swapping | `a, b = b, a` | desempaquetado, sin temporal |
| Slice assign | `l[1:3] = [9]` | cambia longitud |
| Operadores | `+ - * / // % ** @ \| & ^ ~ << >>` | `//` floor, `/` siempre float |
| Comparar | `< <= > >= == != is is not in not in` | encadenables: `0 < x <= 10` |
| Lógicos | `and or not` | devuelven operandos, no bool |
| Identidad | `x is None` | `is` SOLO para None/Singletons |

## Estructuras de datos: construcción y métodos

```python
l = [1,2]; l = list("ab"); l = [x*2 for x in r]
t = (1,2); t = (1,); a, b, *resto = t
s = {1,2}; s = set(l); s = {x for x in r}
d = {"a":1}; d = dict(a=1); d = dict(zip(ks, vs)); d = {k: v for ...}
fs = frozenset([1])
from collections import deque, defaultdict, Counter, OrderedDict, ChainMap, NamedTuple
```

| `list` | `dict` | `set` | `deque` |
|---|---|---|---|
| `append/extend/insert` | `d[k]` (KeyError) | `add/update` | `append/appendleft` |
| `pop(i)/remove/del l[i]` | `get(k, def)` | `discard` (no error) | `pop/popleft` |
| `index/count/copy` | `pop(k,def)/popitem` | `remove` (KeyError) | `rotate(±n)` |
| `sort(key,rev)/reverse` | `keys/values/items` | `pop` (arbitrario) | `maxlen=N` |
| `sorted(l)` (nueva) | `setdefault/update` | ops: `\| & - ^ <= >=` | `extend/extendleft` |
| `l[::-1]` copia | `d1 \| d2` merge (3.9+) | `isdisjoint` | `count/index` |

| `Counter` | `defaultdict` | `OrderedDict` | `ChainMap` |
|---|---|---|---|
| `most_common(n)` | `dd[k].append(x)` | `move_to_end(k)` | capas: 1ª gana |
| `elements()` | factories: list/set/int | LRU manual | `maps`, `new_child` |
| `+ - & \|` aritmética | crea al leer ⚠️ | orden en `==` | escritura al 1º |

Costos clave: `in` → set/dict **O(1)** vs list O(n); `pop(0)` de list O(n) → `deque.popleft()` O(1); copias: `.copy()` shallow, `copy.deepcopy` profunda. Detalle: [[Python - Estructuras de Datos]].

## Strings y formatos

```python
s.strip/lstrip/rstrip()  split(sep)  rsplit(sep,1)  splitlines()  sep.join(l)
s.replace(a,b,n)  find→-1  index→ValueError  count  startswith/endswith(tupla)
s.upper/lower/title/casefold  center/ljust/rjust  isdigit/isalpha/isalnum/isupper
s.removeprefix(p)  removesuffix(p)          # 3.9+
s.encode("utf-8") → bytes; b.decode("utf-8")
```

| Spec | Significado | Ejemplo → salida |
|---|---|---|
| `:>10` / `:<10` / `:^10` | alineación | `f"{42:>5}"` → `'   42'` |
| `:0>5` | relleno | `'00042'` |
| `:.2f` | decimales | `3.14` |
| `:,` | miles | `1,000,000` |
| `:.2%` | porcentaje | `91.23%` |
| `:+g` / `:g` | signo / genérico | |
| `!r !s !a` | `repr/str/ascii` | `f"{'h'!r}"` → `'h'` |
| `{expr=}` | refleja expr (3.8+) | `f"{a+b=}"` → `a+b=3` |
| `{x:{w}}` | ancho dinámico | |
| `{v:{v}.{v}f}` | specs desde variables | |

## Control de flujo

```python
if c: ... elif c2: ... else: ...
msg = "si" if c else "no"                     # ternario
for i in range(a, b, step): ...               # for-each sobre iterable
for i, v in enumerate(l, 1): ...              # con índice
for k, v in d.items(): ...                    # dict
for a, b in zip(l1, l2, strict=True): ...     # paralelo + validación (3.10+)
while c: ... 
for/while: ... else: ...                      # else SIN break se ejecuta
break / continue / pass
match cmd:                                    # 3.10+ pattern matching
    case "q" | "quit": ...
    case [primero, *resto]: ...
    case {"id": i}: ...
    case n if n < 0: ...
    case _: ...
```

## Comprensiones y generadores

```python
[x**2 for x in r if x % 2]                    # lista (lazy: (...))
{k: v for k, v in pares}
{x for x in r}                                # set
(x for x in r)                                # generator expression
[t for sub in matrix for t in sub]            # aplanar (orden: externa→interna)
[(x, y) for x in xs for y in ys]              # producto cartesiano
```

| Forma | Lazy | Uso |
|---|---|---|
| `[...]` | ❌ | necesitás la lista: la más rápida |
| `(...)` | ✅ | pipelines, archivos grandes, `sum(len(x) for ...)` |
| `map(f, it)`/`filter(f, it)` | ✅ | cuando `f` ya existe |

## Funciones

```python
def f(a, b=1, /, c=2, *args, d, e=3, **kwargs) -> int:
    """Docstring. Args:... Returns:... Raises:..."""
    return a
f = lambda x: x**2                            # anónima, una expresión
```

| Modo | Sintaxis | Paso | Ejemplo |
|---|---|---|---|
| posicional | `a` | pos o kw | `f(1)`/`f(a=1)` |
| pos-only (3.8+) | `a /` | solo pos | `f(1)` |
| default | `a=1` | opcional | `f()` |
| kw-only | `*, a` | solo kw | `f(a=1)` |
| var-pos | `*args` | tupla | `f(1,2)` → `args=(2,)` |
| var-kw | `**kwargs` | dict | `f(x=9)` → `kwargs={'x':9}` |

```python
@functools.cache                               # memoización (funciones puras)
@functools.wraps(func)                         # preserva __name__/__doc__ en decoradores
def deco(fn):
    def wrapper(*a, **kw): return fn(*a, **kw) # args/kwargs = firma genérica
    return wrapper
@contextmanager
def cm():                                      # context manager en 4 líneas
    setup(); 
    try: yield recurso
    finally: cleanup()
```

⚠️ default mutable = bug: `def f(x=[])` comparte la lista entre llamadas. Scope LEGB; `nonlocal` para closures.

## Manejo de errores

```python
try: riesgo()
except ValueError as e: ...
except (TypeError, KeyError) as e: ...
except Exception: raise                       # re-lanzar
else: ok()                                     # solo sin excepción
finally: cleanup()                             # siempre
raise Err("m") from original                   # encadenar (3+)
except* ValueError as eg: ...                  # 3.11+ ExceptionGroup
class MiError(Exception): pass                # custom: heredar de Exception
```

`EAFP` idiomático (try/except sobre `if hasattr`); jerarquía: `Exception` → ValueError/TypeError/KeyError/IndexError/OSError/FileNotFoundError/RuntimeError. Detalle: [[Python - Fundamentos y Sintaxis]].

## OOP: tarjeta rápida

```python
class C(Base, metaclass=type):
    attr = 1                                   # de clase (¡no mutables!)
    __slots__ = ("a",)                         # sin __dict__, ahorro de memoria
    def __init__(self, a): self.a = a; self._p = 1; self.__priv = 2
    @property
    def p(self): return self._p                # getter (data descriptor)
    @p.setter
    def p(self, v): self._p = v                # setter con validación
    @classmethod
    def factory(cls, *a): return cls(*a)       # cls = clase (polimórfica)
    @staticmethod
    def util(): ...                            # sin self/cls
    def __repr__(self): return f"C({self.a!r})"
    def __eq__(self, o): ...; __hash__ = None  # eq anula hash (definir si inmutable)
    def __enter__/__exit__ ...                 # with
    def __call__(self, *a): ...                # invocable
    def __iter__/__next__ ...                  # iterador
    def __len__/__getitem__/__contains__ ...   # colección
class H(C):                                    # herencia; MRO C3
    def m(self): return super().m() + "!"      # siguiente en __mro__
from dataclasses import dataclass, field
@dataclass(frozen=True, order=True, slots=True)
class P: x: float; y: float = 0.0; tags: tuple = ()
from enum import Enum, Flag, StrEnum
class Color(Enum): RED = 1                     # Color.RED, Color(1), Color["RED"]
from abc import ABC, abstractmethod
class I(ABC):
    @abstractmethod
    def m(self): ...
```

| ¿Cuándo? | Elegir |
|---|---|
| solo datos inmutables | `NamedTuple` / `@dataclass(frozen=True)` |
| datos + invariantes | `@dataclass` + `__post_init__` o `class` |
| contrato duro de interfaz | `ABC`/`@abstractmethod` |
| reutilizar código sin "ser un" | composición, no herencia |

Detalle: [[Python - Programación Orientada a Objetos]].

## Archivos y serialización

```python
from pathlib import Path
p = Path("dir") / "f.txt"
p.read_text(encoding="utf-8"); p.write_text(s, encoding="utf-8")
p.read_bytes(); p.write_bytes(b); p.mkdir(parents=True, exist_ok=True)
list(p.rglob("*.py")); p.exists(); p.stat().st_size; p.replace(dest)   # atómico
with open(p, encoding="utf-8") as f:                                  # SIEMPRE with
    for linea in f: ...                                                # streaming O(1)
json.dumps(o, ensure_ascii=False, indent=2, default=str); json.loads(s)
csv.DictReader(f) / DictWriter(f, fieldnames=[...])   # + newline=""
tomllib.load(open("c.toml","rb"))                     # 3.11+, solo lectura
pickle.load(f)                                        # ⚠️ NUNCA de fuentes ajenas
sqlite3.connect("db") + "SELECT ... WHERE x=?", (v,)  # parámetros ? siempre
```

Detalle: [[Python - Manejo de Archivos y Serialización]].

## Stdlib: top-20 one-liners

```python
subprocess.run(["ls","-la"], capture_output=True, text=True, check=True)
datetime.now(timezone.utc); datetime.fromisoformat(s); timedelta(days=7)
ZoneInfo("America/Argentina/Buenos_Aires")           # 3.9+
secrets.token_urlsafe(32); uuid.uuid4()
hashlib.sha256(b).hexdigest(); hashlib.file_digest(f,"sha256")   # 3.11+ file
Counter(palabras).most_common(3)
defaultdict(list)[k].append(v)
sorted(l, key=operator.itemgetter(1)); sorted(d, key=d.get)
list(chain.from_iterable(listas))                    # aplanar
list(accumulate([1,2,3]))                            # [1,3,6] acumulados
list(groupby(sorted(datos), key=k))                  # ⚠️ input ordenado
functools.reduce(add, l); functools.partial(f, kw=1)
heapq.nlargest(10, stream)                           # top-10 de stream
bisect.insort(ordenada, x)                           # mantener orden
@functools.lru_cache(maxsize=None)                    # memoización
logging.getLogger(__name__).info("x=%s", x)           # %s lazy, no f-string
argparse.ArgumentParser().parse_args()               # CLI con --help gratis
time.perf_counter()                                  # medir duraciones (no time.time)
re.sub(pat, rep, s); re.findall(r"\d+", s)
```

Concurrencia rápida:

```python
with ThreadPoolExecutor(10) as ex: list(ex.map(f, urls))      # I/O-bound
with ProcessPoolExecutor(4) as ex: list(ex.map(f, datos))     # CPU-bound
asyncio.run(main()); await asyncio.gather(*tareas)            # alta concurrencia I/O
```

Detalle: [[Python - Librerías Estándar Esenciales]], [[Python - Concurrencia Threading Multiprocessing Asyncio]].

## Idiomas pythonicos

| Idioma | Equivalente ingenuo | Por qué |
|---|---|---|
| `if not lista:` | `if len(lista) == 0:` | truthiness, O(1) |
| `for i, v in enumerate(l, 1):` | `for i in range(len(l)): v = l[i]` | legible, seguro |
| `for a, b in zip(l1, l2, strict=True):` | bucle con índices dobles | paralelo validado |
| `sorted(l, key=str.lower)` | comparador custom | la key es la comparación |
| `try: d[k] except KeyError:` (EAFP) | `if k in d: d[k]` | más rápido y pythonico |
| `a, b = b, a` | temp | sin variable temporal |
| `"".join(map(str, l))` | `s += x` en bucle | O(n) vs O(n²) |
| `raise ValueError(...) from e` | `raise` solo | trazabilidad |
| `with open(p) as f:` | `f = open(p); ...; f.close()` | cierre garantizado |
| `x = a or default` | `if a is None: x = default` | ⚠️ si `0`/`""` son válidos, usar `is None` explícito |
| `functools.cache` en funciones puras | dict manual | memoización correcta |
| `return None` explícito | `return` | ok `return` solo; `None` implícito igual |

## Anti-patrones

| Anti-patrón | Problema | Corrección |
|---|---|---|
| `def f(x=[])` / `@dataclass` campo `[]` | default mutable compartido | `None` + crear / `field(default_factory=list)` |
| `except: pass` | silencia todo (incluye `KeyboardInterrupt`) | excepción concreta + manejar o propagar |
| `for x in l: l.remove(x)` | salta elementos | filtro en comprensión / lista nueva |
| `if isinstance(x, A) elif isinstance(x, B)` | polimorfismo roto | métodos/`singledispatch` |
| `is` con valores / `==` con None | bugs por cache/negación | `is None`; `==` para valores |
| `type(x) == Clase` | rompe herencia | `isinstance(x, Clase)` |
| `import *` | namespace pollution | imports explícitos |
| `sys.path.insert` en código | frágil, inseguro | venv + instalación (`[[Python - Módulos, Paquetes y Distribución]]`) |
| `print` como log | sin niveles/rotación | `logging` |
| `shell=True` + f-string | inyección de shell | lista de args |
| `time.time()` para benchmark | reloj no monotónico | `perf_counter()` |
| `pickle.load` de datos externos | ejecución arbitraria | JSON/TOML-safe |
| `l.pop(0)` en bucle | O(n²) | `deque.popleft()` |
| `yaml.load` (sin SafeLoader) | ejecución arbitraria | `safe_load` |
| `== []` para None-check | confunde vacío con ausencia | `is None` |
| mutar dict/list mientras iterás | `RuntimeError`/fantasmas | iterar copia |

## Regex: patrones de bolsillo

```python
import re
re.findall(r"\d+", s); re.search(m, s); re.sub(p, r, s); re.compile(p)  # compilar = más rápido
r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"     # email
r"^\d{1,3}(\.\d{1,3}){3}$"                               # IPv4
r"\d{4}-\d{2}-\d{2}"                                     # fecha ISO
r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"   # UUID
r"https?://[\w.-]+(?:/[\w./%-]*)?"                        # URL simple
# flags: re.IGNORECASE | re.MULTILINE | re.DOTALL; crudos r"" siempre
```

## Testing rápido (pytest)

```python
@pytest.fixture
def usuario(): return Usuario("Juan")
@pytest.mark.parametrize("a,b,esp", [(1,2,3), (0,0,0)])
def test_suma(a, b, esp): assert a + b == esp
with patch("modulo.api") as api: api.get.return_value = {"ok": True}
@pytest.mark.asyncio
async def test_async(): assert await f() == 1
```

Detalle: [[Python - Testing, Calidad y Herramientas]].

## Línea de versiones (3.9 → 3.13+)

| Versión | Novedades imprescindibles |
|---|---|
| 3.9 | dict `\|` merge, `str.removeprefix/suffix`, `zoneinfo`, genéricos builtin (`list[int]`), `functools.cache` |
| 3.10 | `match/case`, `X \| Y` en tipos, `zip(strict=)`, `TypeAlias`, `int.bit_count` |
| 3.11 | `tomllib`, `except*`/`ExceptionGroup`, `datetime.fromisoformat` completo, `StrEnum`, `Self`, `TaskGroup`, `LiteralString` |
| 3.12 | `type` alias, f-strings libres (comillas/lines), `itertools.batched`, `@override`, `Path.walk`, `@dataclass(slots=True)` maduro |
| 3.13 | `copy.replace`, `ast` unparse en error messages, nuevo JIT experimental (`PYTHON_JIT=1`), preview free-threading (nogil), `mimalloc` por defecto |
| 3.14 | **PEP 649** anotaciones evaluadas perezosamente, `compile -O` mejoras, t-strings (template strings, preview) |

## CLI de Python (one-liners de shell)

```bash
python -m json.tool f.json                 # pretty-print JSON
python -m http.server 8000 --bind 127.0.0.1
python -m timeit -n 100000 -s "l=list(range(1000))" "l[500]"
python -m cProfile -s cumulative script.py
python -m venv .venv && source .venv/bin/activate
python -m pip install -e ".[dev]"
python -m pdb script.py                    # / breakpoint() en código
python -m doctest mod.py -v
python -m compileall src/                  # pre-compile checks
ruff check . --fix && ruff format . && mypy .
```

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — modelo de datos, truthiness, control de flujo profundo
- [[Python - Estructuras de Datos]] — colecciones y collections completas
- [[Python - Programación Orientada a Objetos]] — clases, dunders, MRO, dataclasses
- [[Python - Manejo de Archivos y Serialización]] — open(), pathlib, JSON/pickle/CSV/TOML
- [[Python - Módulos, Paquetes y Distribución]] — imports, venv, pyproject, PyPI
- [[Python - Librerías Estándar Esenciales]] — stdlib por dominio con recetas
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — hilos, procesos, async
- [[Python - Patrones, Buenas Prácticas y Tipado]] — type hints, patrones de diseño
- [[Python - Testing, Calidad y Herramientas]] — pytest, ruff, mypy, CI
- [[Python - Redes y HTTP]] — sockets, requests, servidores
- [[Python - Interfaces Gráficas (Tkinter)]] — GUI con la stdlib
