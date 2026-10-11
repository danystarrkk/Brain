---
title: "Python - Estructuras de Datos"
tags:
  - lang/python
  - concept/data-structures
  - concept/collections
  - level/basico
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Python - Estructuras de Datos

> **Objetivo**: Dominio completo de las colecciones nativas: list, tuple, set, dict, y sus variantes.

---

## 1. Comparativa Rápida

| Tipo | Mutable | Ordenado | Indexable | Hashable | Sintaxis | Caso de uso principal |
|------|---------|----------|-----------|----------|----------|----------------------|
| `list` | ✅ | ✅ | ✅ | ❌ | `[1,2,3]` | Secuencia modificable |
| `tuple` | ❌ | ✅ | ✅ | ✅* | `(1,2,3)` | Registro inmutable, clave dict |
| `set` | ✅ | ❌ | ❌ | ❌ | `{1,2,3}` | Unicidad, ops. matemáticas |
| `frozenset` | ❌ | ❌ | ❌ | ✅ | `frozenset([1,2])` | Clave dict, elemento set |
| `dict` | ✅ | ✅ (3.7+) | ✅ (por clave) | ❌ | `{"a":1}` | Mapeo clave→valor |
| `collections.OrderedDict` | ✅ | ✅ | ✅ | ❌ | `OrderedDict()` | Orden de inserción explícito |
| `collections.defaultdict` | ✅ | ✅ | ✅ | ❌ | `defaultdict(list)` | Valores por defecto |
| `collections.Counter` | ✅ | ❌ | ✅ | ❌ | `Counter([1,1,2])` | Conteo de elementos |
| `collections.deque` | ✅ | ✅ | ✅ | ❌ | `deque([1,2,3])` | Cola doble (O(1) extremos) |

* Tuplas son hashables solo si todos sus elementos lo son.

---

## 2. List (Lista)

### 2.1 Creación y Acceso

```python
# Creación
l = [1, 2, 3]
l = list("abc")        # ['a', 'b', 'c']
l = list(range(3))     # [0, 1, 2]
l = [x**2 for x in range(5)]  # [0, 1, 4, 9, 16]

# Acceso
l[0]       # 1 (primero)
l[-1]      # 3 (último)
l[1:3]     # [2, 3] (slice: inicio inclusive, fin exclusive)
l[::2]     # [1, 3] (step)
l[::-1]    # [3, 2, 1] (invertida)
```

### 2.2 Métodos de Modificación

```python
l = [1, 2, 3]

# Añadir
l.append(4)           # [1,2,3,4] - un elemento
l.extend([4,5])       # [1,2,3,4,5] - iterable
l.insert(0, 0)        # [0,1,2,3] - en índice

# Eliminar
l.pop()               # 3 (elimina y devuelve último)
l.pop(0)              # 1 (elimina por índice)
l.remove(2)           # elimina primera ocurrencia de 2 (ValueError si no existe)
del l[0]              # elimina por índice (sin retorno)
l.clear()             # []

# Ordenación
l.sort()              # in-place, retorna None
l.sort(reverse=True)  # descendente
l.sort(key=len)       # por longitud
sorted(l)             # NUEVA lista ordenada (no modifica original)
l.reverse()           # in-place
l[::-1]               # copia invertida
```

### 2.3 Métodos de Consulta

```python
l = [1, 2, 2, 3]

len(l)          # 4
l.count(2)      # 2 (ocurrencias)
l.index(2)      # 1 (primer índice, ValueError si no existe)
2 in l          # True (búsqueda O(n))
l.copy()        # copia superficial
```

### 2.4 Operaciones de Listas

```python
a = [1, 2]
b = [3, 4]

a + b           # [1,2,3,4] (nueva lista)
a * 3           # [1,2,1,2,1,2] (repetición)

# Aplanar lista de listas
plana = [x for sub in [[1,2],[3,4]] for x in sub]  # [1,2,3,4]
# O con itertools
import itertools
list(itertools.chain.from_iterable([[1,2],[3,4]]))
```

---

## 3. Tuple (Tupla)

### 3.1 Características

- **Inmutable**: no se puede modificar tras creación
- **Hashable** (si sus elementos lo son) → usable como clave de dict o elemento de set
- **Ligeramente más rápida y ligera** que lista
- **Desempaquetado** nativo

### 3.2 Uso

```python
# Creación
t = (1, 2, 3)
t = 1, 2, 3           # paréntesis opcionales
t = (1,)              # tupla de 1 elemento (coma obligatoria)
t = tuple([1, 2, 3])  # desde iterable

# Acceso (igual que lista)
t[0]      # 1
t[-1]     # 3
t[1:3]    # (2, 3)

# Desempaquetado
a, b, c = t
a, *resto = t      # a=1, resto=(2,3)
*a, b = t          # a=(1,2), b=3

# Métodos (solo 2)
t.count(2)    # ocurrencias
t.index(2)    # primer índice

# Concatenación/repetición (crean NUEVA tupla)
(1, 2) + (3, 4)  # (1, 2, 3, 4)
(1, 2) * 3       # (1, 2, 1, 2, 1, 2)
```

### 3.3 NamedTuple (Tipado + Acceso por Atributo)

```python
from typing import NamedTuple

class Punto(NamedTuple):
    x: float
    y: float
    etiqueta: str = "P"

p = Punto(1.0, 2.0)
print(p.x, p.y)       # 1.0 2.0
print(p[0], p[1])     # 1.0 2.0 (indexable)
print(p.etiqueta)     # "P"
```

---

## 4. Set / Frozenset

### 4.1 Set (Mutable)

```python
s = {1, 2, 3}
s = set([1, 2, 2, 3])   # {1, 2, 3} (elimina duplicados)
s = set("hola")         # {'h', 'o', 'l', 'a'}
s = {x**2 for x in range(5)}  # {0, 1, 4, 9, 16}

# Elementos deben ser hashables (inmutables)
# ❌ {[1,2], [3,4]}  # TypeError: unhashable type: 'list'
# ✅ {(1,2), (3,4)}  # OK

# Modificación
s.add(4)                # añade uno
s.update([4, 5, 6])     # añade iterable
s.discard(2)            # elimina si existe (no error)
s.remove(2)             # elimina (KeyError si no existe)
s.pop()                 # elimina y devuelve arbitrario
s.clear()

# Operaciones matemáticas
a = {1, 2, 3}
b = {3, 4, 5}

a | b      # {1,2,3,4,5}  unión
a & b      # {3}          intersección
a - b      # {1,2}        diferencia (en a no en b)
a ^ b      # {1,2,4,5}    diferencia simétrica

a <= b     # subset (subconjunto)
a < b      # proper subset
a >= b     # superset
a > b      # proper superset
a.isdisjoint(b)  # True si no comparten elementos
```

### 4.2 Frozenset (Inmutable + Hashable)

```python
fs = frozenset([1, 2, 3])
# fs.add(4)  # AttributeError

# Usos: clave de dict, elemento de set
d = {frozenset([1,2]): "par"}
s = {frozenset([1,2]), frozenset([3,4])}
```

---

## 5. Dict (Diccionario)

### 5.1 Creación y Acceso

```python
# Creación
d = {"a": 1, "b": 2}
d = dict(a=1, b=2)
d = dict([("a", 1), ("b", 2)])
d = {x: x**2 for x in range(5)}  # {0:0, 1:1, 2:4, 3:9, 4:16}
d = dict.fromkeys(["a", "b"], 0)  # {"a":0, "b":0}

# Claves deben ser hashables (inmutables)
# ❌ {[1,2]: "x"}  # TypeError

# Acceso
d["a"]           # 1 (KeyError si no existe)
d.get("x")       # None
d.get("x", 0)    # 0 (default)
d.setdefault("x", [])  # [] si no existe, y lo asigna

# Claves, valores, items (vistas dinámicas)
d.keys()      # dict_keys(['a','b'])
d.values()    # dict_values([1,2])
d.items()     # dict_items([('a',1),('b',2)])
list(d.keys())  # materializar
```

### 5.2 Modificación

```python
d = {"a": 1}
d["b"] = 2           # añade/actualiza
d.update({"c": 3, "d": 4})  # múltiplo
d.update(e=5, f=6)          # keyword args

del d["a"]           # elimina (KeyError si no existe)
d.pop("b")           # elimina y devuelve valor
d.pop("x", None)     # default si no existe
d.popitem()          # elimina y devuelve (k,v) último (3.7+)
d.clear()
```

### 5.3 Operaciones Avanzadas

```python
# Fusión (Python 3.9+)
d1 = {"a": 1}
d2 = {"b": 2}
d3 = d1 | d2          # {"a":1, "b":2} (nuevo)
d1 |= d2              # d1 actualizado in-place

# Comprobaciones
"a" in d              # True (O(1))
d.get("x") is None    # check existencia

# Iteración
for k in d:           # itera claves
for k, v in d.items():  # clave + valor
for v in d.values():    # solo valores

# Diccionarios anidados
users = {"juan": {"edad": 30, "ciudad": "Madrid"}}
users["juan"]["edad"]  # 30
users.setdefault("ana", {}).setdefault("edad", 25)
```

---

## 6. Módulo `collections` (Estructuras Especializadas)

```python
from collections import defaultdict, Counter, deque, OrderedDict, namedtuple, ChainMap

# defaultdict: valor por defecto automático
dd = defaultdict(list)
dd["a"].append(1)   # crea [] automáticamente
dd["a"].append(2)   # [1, 2]

dd_int = defaultdict(int)
dd_int["x"] += 1    # 0 + 1 = 1

# Counter: conteo de elementos
c = Counter("abracadabra")  # {'a':5, 'b':2, 'r':2, 'c':1, 'd':1}
c.most_common(2)    # [('a', 5), ('b', 2)]
c + Counter("abc")  # suma contadores
c - Counter("aa")   # resta (elimina negativos/ceros)

# deque: cola doble (O(1) en ambos extremos)
dq = deque([1, 2, 3])
dq.append(4)        # derecha
dq.appendleft(0)    # izquierda
dq.pop()            # 4
dq.popleft()        # 1
dq.rotate(1)        # rota derecha
dq.rotate(-1)       # rota izquierda
dq.maxlen = 3       # límite automático (elimina viejos)

# OrderedDict: orden de inserción explícito (Python 3.7+ dict ya lo hace)
od = OrderedDict([("a", 1), ("b", 2)])
od.move_to_end("a")  # mueve al final
od.move_to_end("a", last=False)  # mueve al inicio

# ChainMap: búsqueda en múltiples dicts
defaults = {"color": "rojo", "size": 10}
user = {"color": "azul"}
config = ChainMap(user, defaults)
config["color"]  # "azul" (user tiene prioridad)
config["size"]   # 10 (defaults)
```

---

## 7. `dataclasses` (Python 3.7+) - Clases de Datos

```python
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class Usuario:
    nombre: str
    edad: int = 18
    email: Optional[str] = None
    tags: List[str] = field(default_factory=list)
    # field() permite: default, default_factory, init, repr, compare, hash

u = Usuario("Juan")
u.tags.append("admin")
# Auto-genera: __init__, __repr__, __eq__, __hash__ (si frozen=True)

@dataclass(frozen=True)  # inmutable + hashable
class Config:
    debug: bool
    puerto: int = 8080
```

---

## 8. `typing` - Tipos para Colecciones (Python 3.9+)

```python
# Sintaxis moderna (built-in generics)
lista: list[int] = [1, 2, 3]
tupla: tuple[str, int, float] = ("a", 1, 1.0)
conjunto: set[str] = {"a", "b"}
diccionario: dict[str, int] = {"a": 1}
opcional: int | None = None
union: int | str = "hola"

# Para versiones < 3.9 (typing)
from typing import List, Tuple, Set, Dict, Optional, Union
lista: List[int] = [1, 2, 3]
```

---

## 9. Rendimiento y Complejidad

| Operación | List | Tuple | Set | Dict | Deque |
|-----------|------|-------|-----|------|-------|
| Acceso por índice | O(1) | O(1) | ❌ | O(1)* | O(1) |
| Búsqueda (`in`) | O(n) | O(n) | **O(1)** | **O(1)** | O(n) |
| Inserción final | O(1)** | ❌ | O(1) | O(1) | **O(1)** |
| Inserción inicio | O(n) | ❌ | ❌ | ❌ | **O(1)** |
| Eliminación final | O(1) | ❌ | O(1) | O(1) | **O(1)** |
| Eliminación inicio | O(n) | ❌ | ❌ | ❌ | **O(1)** |
| Memoria | Media | Baja | Alta | Alta | Media |

* Dict: acceso por clave. ** Amortizado.

---

## 10. Copias: Shallow vs Deep

```python
import copy

original = [[1, 2], [3, 4]]

shallow = original.copy()           # o list(original), original[:]
shallow = copy.copy(original)       # mismo efecto
shallow[0].append(99)               # ¡AFECTA AL ORIGINAL!
# original: [[1, 2, 99], [3, 4]]

deep = copy.deepcopy(original)      # copia recursiva
deep[0].append(88)                  # original intacto
```

---

## 11. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Sintaxis base, control de flujo
- [Programación Orientada a Objetos](./Python%20-%20Programaci%C3%B3n%20Orientada%20a%20Objetos.md) - Clases, herencia
- [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) - Serialización JSON, pickle, CSV
- [Librerías Estándar](./Python%20-%20Librer%C3%ADas%20Est%C3%A1ndar%20Esenciales.md) - itertools, functools, pathlib

---

*Etiquetas: #python #list #tuple #set #dict #frozenset #collections #dataclasses #deque #counter #defaultdict #namedtuple #chainmap #complejidad #shallow-deep-copy*