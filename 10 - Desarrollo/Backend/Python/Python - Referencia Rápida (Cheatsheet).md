---
title: "Python - Referencia Rápida (Cheatsheet)"
tags:
  - lang/python
  - concept/cheatsheet
  - concept/reference
  - level/basico
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Referencia Rápida (Cheatsheet)

> **Resumen ultra-condensado** para consulta rápida. Ver notas especializadas para profundidad.

---

## 1. Sintaxis Básica

```python
# Comentarios
# Una línea
"""Docstring
multilínea"""

# Variables
x = 42          # int
x = 3.14        # float
x = "hola"      # str
x = True        # bool
x = None        # NoneType

# Tipado
x: int = 42
def f(x: int) -> str: ...

# Operadores
+ - * / // % ** @
<< >> & | ^ ~
< <= > >= == != is is not in not in
and or not
```

---

## 2. Colecciones

```python
# List (mutable, ordenada)
l = [1, 2, 3]
l.append(4)
l.extend([5,6])
l.insert(0, 0)
l.pop()        # 3
l.pop(0)       # 1
l.remove(2)    # elimina primera ocurrencia
l.sort()       # in-place
sorted(l)      # nueva lista
l[::-1]        # invertida
[x*2 for x in l if x>1]

# Tuple (inmutable, hashable)
t = (1, 2, 3)
t = 1, 2, 3
t = (1,)       # tupla 1 elemento
a, b, *resto = t

# Set (únicos, no ordenados)
s = {1, 2, 3}
s.add(4)
s.update([4,5])
s.discard(2)   # no error si no existe
s.remove(2)    # KeyError si no existe
a | b          # unión
a & b          # intersección
a - b          # diferencia
a ^ b          # diff simétrica
frozenset([1,2])  # inmutable, hashable

# Dict (clave→valor)
d = {"a": 1, "b": 2}
d = dict(a=1, b=2)
d["c"] = 3
d.get("x", 0)  # 0 si no existe
d.setdefault("x", [])  # [] si no existe, y asigna
{k: v for k,v in d.items()}
d | d2         # merge (3.9+)
```

---

## 3. Control de Flujo

```python
# If/elif/else
if x > 0:
    pass
elif x < 0:
    pass
else:
    pass

# Ternario
msg = "pos" if x > 0 else "neg"

# Match (3.10+)
match x:
    case 1: pass
    case 2|3: pass
    case _: pass

# For
for i in range(5): pass
for i, v in enumerate(l): pass
for k, v in d.items(): pass
for a, b in zip(l1, l2): pass

# While
while cond: pass
else: pass  # si no hubo break

# Break/Continue
for i in range(10):
    if i == 3: continue
    if i == 7: break
```

---

## 4. Funciones

```python
def f(a: int, b: int = 10, *args, **kwargs) -> str:
    """Docstring.
    Args:
        a: descripción
        b: descripción
    Returns:
        descripción
    Raises:
        ValueError: si...
    """
    return f"{a}+{b}={a+b}"

# Lambda
cuadrado = lambda x: x**2
pares = filter(lambda x: x%2==0, range(10))

# Args flexibles
def f(a, b, *args, kw1=1, **kwargs): pass
f(1, 2, 3, 4, kw1=5, kw2=6)
# a=1, b=2, args=(3,4), kw1=5, kwargs={'kw2':6}

# Type hints
def f(items: list[int]) -> dict[str, int]: ...

# Decoradores
@decorador
def f(): pass

# Context manager
from contextlib import contextmanager
@contextmanager
def cm():
    setup()
    try:
        yield resource
    finally:
        cleanup()
```

---

## 5. Clases y POO

```python
class Clase:
    """Docstring."""
    clase_attr = 0
    
    def __init__(self, x: int):
        self.x = x
        self._protegido = 1
        self.__privado = 2  # name mangling
    
    @property
    def x(self) -> int:
        return self._x
    
    @x.setter
    def x(self, valor: int):
        if valor < 0: raise ValueError
        self._x = valor
    
    def metodo(self) -> str:
        return f"x={self.x}"
    
    @classmethod
    def desde_cero(cls):
        return cls(0)
    
    @staticmethod
    def utilidad():
        pass
    
    def __str__(self): return f"Clase({self.x})"
    def __repr__(self): return f"Clase(x={self.x})"
    def __eq__(self, o): return isinstance(o, Clase) and self.x == o.x

# Herencia
class Hija(Padre):
    def __init__(self, x, y):
        super().__init__(x)
        self.y = y
    
    def metodo(self):
        return super().metodo() + f", y={self.y}"

# Dataclass (3.7+)
from dataclasses import dataclass, field
@dataclass
class Punto:
    x: float
    y: float = 0.0
    etiquetas: list[str] = field(default_factory=list)
    
    @property
    def distancia(self) -> float:
        return (self.x**2 + self.y**2)**0.5

# Protocol (3.8+)
from typing import Protocol
class Serializable(Protocol):
    def to_json(self) -> str: ...
    @classmethod
    def from_json(cls, s: str) -> "Serializable": ...
```

---

## 6. Manejo de Errores

```python
try:
    riesgo()
except ValueError as e:
    print(e)
except (TypeError, KeyError) as e:
    print(e)
except Exception as e:
    print(e)
else:
    print("sin error")
finally:
    print("limpieza")

# Levantar
raise ValueError("mensaje")
raise ValueError("mensaje") from original_error

# Excepciones personalizadas
class MiError(Exception): pass
class ErrorValidacion(MiError):
    def __init__(self, campo: str):
        self.campo = campo
        super().__init__(f"Campo inválido: {campo}")

# Context manager
with open("archivo.txt") as f:
    pass  # auto-close

# Context manager personalizado
from contextlib import contextmanager
@contextmanager
def transaccion(db):
    try:
        yield db
        db.commit()
    except:
        db.rollback()
        raise
```

---

## 7. Archivos y Serialización

```python
# Pathlib (recomendado)
from pathlib import Path
p = Path("archivo.txt")
p.read_text(encoding="utf-8")
p.write_text("hola", encoding="utf-8")
p.read_bytes()
p.write_bytes(b"datos")
p.mkdir(parents=True, exist_ok=True)
for f in p.rglob("*.py"): pass

# JSON
import json
json.dumps(obj, ensure_ascii=False, indent=2)
json.loads(s)
json.dump(obj, f, indent=2)
json.load(f)

# Pickle (solo Python, ¡NO seguro!)
import pickle
pickle.dumps(obj)
pickle.loads(b)
pickle.dump(obj, f)
pickle.load(f)

# CSV
import csv
with open("f.csv", newline="") as f:
    for row in csv.DictReader(f): pass
with open("f.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=["a","b"])
    w.writeheader()
    w.writerow({"a":1,"b":2})

# YAML (pip install pyyaml)
import yaml
yaml.safe_load(f)
yaml.dump(obj, f, allow_unicode=True)

# TOML (3.11+ tomllib / tomli)
import tomllib
tomllib.load(f)
```

---

## 8. Concurrencia

```python
# Threading (I/O-bound)
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(10) as ex:
    list(ex.map(func, items))
    list(ex.map(lambda u: requests.get(u), urls))
    for fut in as_completed(futuros): fut.result()

# Multiprocessing (CPU-bound)
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor(4) as ex:
    list(ex.map(cpu_intensivo, datos))

# Asyncio (I/O-bound, alta concurrencia)
import asyncio
async def f(): await asyncio.sleep(1)
asyncio.run(f())
await asyncio.gather(f1(), f2(), f3())
await asyncio.wait_for(f(), timeout=5)
sem = asyncio.Semaphore(10)
async with sem: await tarea()

# aiohttp
async with aiohttp.ClientSession() as session:
    async with session.get(url) as resp:
        data = await resp.json()
```

---

## 9. Type Hints

```python
from typing import *

# Básicos
x: int = 1
x: str | None = None  # 3.10+
x: int | str = 1      # 3.10+

# Colecciones
l: list[int] = [1,2]
t: tuple[str, int] = ("a", 1)
s: set[str] = {"a"}
d: dict[str, int] = {"a": 1}

# Generics
T = TypeVar('T')
def f(x: T) -> T: ...

# Callable
f: Callable[[int, str], bool] = ...

# Union / Optional
x: int | None = None
x: Union[int, None] = None  # <3.10

# Literal / TypedDict
from typing import Literal, TypedDict
x: Literal["a", "b"] = "a"

class User(TypedDict):
    name: str
    age: NotRequired[int]  # 3.11+

# Overload
from typing import overload
@overload
def f(x: int) -> str: ...
@overload
def f(x: str) -> int: ...
def f(x): ...

# Protocol (3.8+)
from typing import Protocol
class Serializer(Protocol):
    def serialize(self) -> str: ...
```

---

## 9. Módulos Útiles

```python
# pathlib
from pathlib import Path
Path("archivo.txt").read_text()
Path("dir").mkdir(parents=True, exist_ok=True)
for f in Path(".").rglob("*.py"): pass

# datetime
from datetime import datetime, timedelta, timezone
datetime.now()
datetime.now(timezone.utc)
datetime.fromisoformat("2024-01-15T10:30:00")
datetime(2024,1,15) + timedelta(days=7)

# collections
from collections import defaultdict, Counter, deque
defaultdict(list)["a"].append(1)
Counter("abracadabra")
deque([1,2,3]).append(4)

# itertools
from itertools import *
chain([1,2], [3,4])
cycle([1,2])
islice(iterable, 10)
permutations([1,2,3])
combinations([1,2,3], 2)
groupby(sorted(l), key=lambda x: x%2)

# functools
from functools import *
@lru_cache(maxsize=128)
def f(x): return x*2
partial(f, a=1)
reduce(lambda a,b: a+b, [1,2,3])
wraps(func)(wrapper)

# re
import re
re.search(r"\d+", "a123b")
re.findall(r"\d+", "a1b2c3")
re.sub(r"\d+", "X", "a1b2")
re.compile(r"\d+")

# subprocess
import subprocess
subprocess.run(["cmd", "arg"], capture_output=True, text=True, timeout=30)
subprocess.run(cmd, check=True, timeout=10)

# logging
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
logger.info("mensaje", extra={"user_id": 123})
```

---

## 10. Testing Rápido

```python
import pytest
from unittest.mock import Mock, patch, AsyncMock

# Fixture
@pytest.fixture
def usuario():
    return Usuario("Juan")

# Parametrizado
@pytest.mark.parametrize("a,b,esp", [(1,2,3), (0,0,0)])
def test_suma(a, b, esp): assert a+b == esp

# Mock
def test_mock(usuario):
    with patch("modulo.api") as mock_api:
        mock_api.get.return_value = {"ok": True}
        assert usuario.obtener() == {"ok": True}

# Async
@pytest.mark.asyncio
async def test_async():
    async def f(): return 1
    assert await f() == 1
```

---

## 11. One-Liners Útiles

```bash
# JSON pretty
python -m json.tool archivo.json

# HTTP server
python -m http.server 8000

# Profile
python -m cProfile -o prof.stats script.py
snakeviz prof.stats

# Timeit
python -m timeit -n 1000 "sum(range(100))"

# Virtualenv
python -m venv venv
source venv/bin/activate

# Pip
pip install -e .
pip freeze > requirements.txt
pip install -r requirements.txt

# Lint/Format
ruff check . --fix
ruff format .
mypy .
black .
isort .
```

---

## 12. Patrones Regex Comunes

```python
import re

# Email
r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"

# URL
r"https?://(?:[-\w.]|(?:%[\da-fA-F]{2}))+"

# IP
r"\b(?:\d{1,3}\.){3}\d{1,3}\b"

# Fecha ISO
r"\d{4}-\d{2}-\d{2}"

# UUID
r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"

# Email simple
r"^[\w\.-]+@[\w\.-]+\.\w+$"
```

---

## 13. Referencias Cruzadas

| Tema | Nota Detallada |
|------|----------------|
| Sintaxis completa | [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) |
| Colecciones | [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) |
| POO | [Programación Orientada a Objetos](./Python%20-%20Programaci%C3%B3n%20Orientada%20a%20Objetos.md) |
| Archivos/Serialización | [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) |
| Concurrencia | [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) |
| Redes/HTTP | [Redes y HTTP](./Python%20-%20Redes%20y%20HTTP.md) |
| Testing | [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) |
| Patrones | [Patrones y Buenas Prácticas](./Python%20-%20Patrones%2C%20Buenas%20Pr%C3%A1cticas%20y%20Tipado.md) |
| Módulos | [Módulos y Paquetes](./Python%20-%20M%C3%B3dulos%2C%20Paquetes%20y%20Distribuci%C3%B3n.md) |
| Librerías Estándar | [Librerías Esenciales](./Python%20-%20Librer%C3%ADas%20Est%C3%A1ndar%20Esenciales.md) |
| GUI Tkinter | [Interfaces Gráficas](./Python%20-%20Interfaces%20Gr%C3%A1ficas%20(Tkinter).md) |

---

*Etiquetas: #python #cheatsheet #referencia-rapida #sintaxis #colecciones #control-flujo #funciones #clases #excepciones #archivos #concurrencia #type-hints #testing #one-liners #regex #pathlib #datetime #collections #itertools #functools #subprocess #logging #pytest #mock #asyncio #aiohttp*