---
aliases: ["Estructuras de Datos Python", "Colecciones Python", "Collections Python"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Programación Orientada a Objetos]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
  - "[[Python - Referencia Rápida (Cheatsheet)]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
fuente: "https://docs.python.org/es/3/library/stdtypes.html#types-sequence"
---

# Python - Estructuras de Datos

Referencia exhaustiva de las colecciones nativas (`list`, `tuple`, `set`, `dict`, `frozenset`) y del módulo `collections` (`deque`, `defaultdict`, `Counter`, `OrderedDict`, `ChainMap`, `NamedTuple`). Todo verificado con Python 3.14. Sintaxis base en [[Python - Fundamentos y Sintaxis]].

## Comparativa maestra

| Tipo | Mutable | Ordenado | Indexable | Hashable | Complejidad `in` | Uso principal |
|---|---|---|---|---|---|---|
| `list` | ✅ | ✅ (por índice) | ✅ | ❌ | O(n) | secuencia modificable |
| `tuple` | ❌ | ✅ | ✅ | ✅* | O(n) | registro fijo, clave de dict, retorno múltiple |
| `set` | ✅ | ❌ (itera en orden propio) | ❌ | ❌ | **O(1)** | unicidad, pertenencia, operaciones de conjuntos |
| `frozenset` | ❌ | ❌ | ❌ | ✅ | **O(1)** | clave de dict, elemento de set |
| `dict` | ✅ | ✅ (3.7+: inserción) | por clave | ❌ | **O(1)** | mapeo clave→valor |
| `deque` | ✅ | ✅ | ❌ | ❌ | O(n) | colas/stacks O(1) en extremos |
| `defaultdict` | ✅ | ✅ | por clave | ❌ | O(1) | conteo/agrupación sin `if clave` |
| `Counter` | ✅ | vista ordena por frecuencia | ❌ | ❌ | O(1) | histogramas |
| `OrderedDict` | ✅ | ✅ + LRU ops | por clave | ❌ | O(1) | orden explícito, `move_to_end` |

*Tuple es hashable **solo** si todos sus elementos lo son: verificado `hash((1, [2]))` → `TypeError: unhashable type: 'list'`.

> [!info] El modelo de memoria que explica todo
> Una variable es una **referencia** a un objeto. `[1,2]` crea un objeto lista; asignarla a otra variable copia la **referencia**, no el objeto. `is` compara referencias, `==` compara valores (vía `__eq__`). De aquí salen: el aliasing, la semántica de las copias, los argumentos "por referencia" y los defaults mutables.

## `list`: la secuencia por defecto

### Creación y acceso

```python
l = [1, 2, 3]
l = list("abc")            # ['a', 'b', 'c'] — de cualquier iterable
l = list(range(3))         # [0, 1, 2]
l = [x**2 for x in range(5)]   # comprehension: más rápida que map+lambda
l[0], l[-1]                # primero, último (índice negativo desde el final)
l[1:3], l[::2], l[::-1]    # slice, paso 2, copia invertida (ver Slicing)
l[:100]                    # fuera de rango NO lanza: devuelve hasta el final
```

### Métodos (catálogo completo)

| Método | Efecto | Retorno | Nota |
|---|---|---|---|
| `l.append(x)` | añade al final | `None` | O(1) amortizado |
| `l.extend(iterable)` | concatena elementos | `None` | acepta generadores |
| `l.insert(i, x)` | inserta en i | `None` | O(n) — desplaza a la derecha |
| `l.pop(i)` | elimina y devuelve | elemento | sin índice: el último, O(1) |
| `l.remove(x)` | elimina primera ocurrencia | `None` | `ValueError` si no existe; O(n) |
| `l.clear()` | vacía | `None` | |
| `l.index(x, start, end)` | primer índice | índice | `ValueError` si no existe |
| `l.count(x)` | ocurrencias | int | O(n) |
| `l.sort(key=None, reverse=False)` | ordena **in-place** | `None` | Timsort, estable |
| `l.reverse()` | invierte in-place | `None` | no ordena, solo invierte |
| `l.copy()` | copia superficial | nueva lista | = `list(l)` = `l[:]` = `copy.copy(l)` |

```python
l = [1, 2, 2, 3]
l.count(2)                    # 2
l.index(2)                    # 1 (primer índice; ValueError si falta)
sorted(l, reverse=True)       # [3, 2, 2, 1] — NUEVA lista, no muta
l.sort(key=str)               # ordena por representación — la key define "menor que"
```

> [!tip] `sort` vs `sorted`
> `l.sort()` es in-place, O(1) de memoria, devuelve `None` (por eso `l = l.sort()` es un bug silencioso). `sorted(l)` devuelve lista nueva y acepta **cualquier** iterable: `sorted(d.items(), key=lambda kv: kv[1])`. Timsort es **estable**: elementos con key igual conservan su orden original — verificado con `[("b",2), ("a",2), ("c",1)]` ordenado por segundo campo → `[("c",1), ("b",2), ("a",2)]` (b antes que a, como entraron).

### Slicing en profundidad

```python
l = [1, 2, 3, 4, 5]
l[1:3]      # [2, 3]    inicio incl., fin excl.
l[-2:]      # [4, 5]    negativos desde el final
l[::2]      # [1, 3, 5] paso
l[:100]     # [1,2,3,4,5] — no IndexError en slices
l[::-1]     # copia invertida (nuevo objeto)
l[3:1]      # [] — el slice nunca "da la vuelta"
del l[1:3]  # borra por slice
l[1:3] = [9, 8, 7]  # ASIGNA por slice: inserta/borra en bloque (cambia longitud)
```

`slice(a, b, step)` como objeto permite reutilizar y componer cortes: `sl = slice(None, None, -1); l[sl]`.

### Operaciones y trampas de repetición

```python
a, b = [1, 2], [3, 4]
a + b       # [1,2,3,4] — NUEVA lista
a * 3       # [1,2,1,2,1,2] — repite REFERENCIAS
row = [[]] * 3
row[0].append(1)
print(row)          # verificado: [[1], [1], [1]] — ¡aliasing! los 3 elementos son el mismo []
```

> [!warning] `[x] * n` con objetos mutables
> Para `int`/`str` (inmutables) es inocuo. Para listas/dicts crea N referencias al mismo objeto. Construcción segura: `[[] for _ in range(n)]`.

### Comprehensions vs generadores

```python
cuadrados = [x**2 for x in range(10**6)]          # materializa: ~40 MB
gen = (x**2 for x in range(10**6))                # generador: O(1) de memoria, lazy
map_cuadrados = map(lambda x: x**2, range(10**6)) # lazy también, estilo funcional
```

| Forma | Lazy | Memoria | Velocidad típica | Uso |
|---|---|---|---|---|
| `[expr for x in it if cond]` | ❌ | O(n) | **la más rápida** | necesitás la lista |
| `(expr for x in it)` | ✅ | O(1) | ligeramente más lenta | pipeline, archivos gigantes |
| `map/filter` | ✅ | O(1) | rápida en C | composición funcional |
| `{x for x in it}` | ❌ | O(n) | — | set resultante |
| `{k: v for ...}` | ❌ | O(n) | — | dict resultante |

Alcance de la variable de comprensión (verificado en 3.14): las comprehensions de **lista/dict/set** tienen su propio scope y **no** contaminan el exterior (`x` global se conserva tras `[x for x in range(3)]`); en **Python 2** sí lo hacían — cuidado leyendo código viejo.

```python
# Comprehension anidada: aplanar
plana = [x for sub in [[1,2],[3,4]] for x in sub]        # [1,2,3,4]
# Con itertools (léelo como chain)
import itertools
list(itertools.chain.from_iterable([[1,2],[3,4]]))
```

## `tuple`: inmutable, hashable, con matices

### Qué garantiza realmente

Una tupla **no puede reasignar ni insertar elementos**, pero si contiene objetos mutables, esos mutan. Verificado:

```python
t = ([1], 2)
t[0] = 9              # TypeError: 'tuple' object does not support item assignment
t[0].append(3)        # ¡funciona!
print(t)              # verificado: ([1, 3], 2) — el contenido cambió
```

Consecuencia para hashing: una tupla con una lista da `TypeError: unhashable` al usarla como clave; una tupla de inmutables es hashable y sirve como clave de dict/set.

### Sintaxis y desempaquetado

```python
t = (1, 2, 3)
t = 1, 2, 3           # paréntesis opcionales (pero no en return/argumentos siempre...)
t = (1,)              # ¡coma obligatoria! (1) es solo el entero 1
t = tuple([1, 2, 3])  # desde iterable; tuple() vacía = ()

a, b, c = t           # desempaquetado exacto (ValueError si no calzan)
a, *resto = t         # a=1, resto=(2, 3) — resto SIEMPRE es lista en asignación
*a, b = t             # a=[1,2], b=3
(a, b), c = (1, 2), 3 # anidado
```

Solo 2 métodos: `t.count(x)` e `t.index(x)` — mismas semánticas que los de `list`. Concatenación `(1,2)+(3,4)` crea tupla nueva, O(n).

> [!tip] ¿Cuándo tuple en vez de list?
> 1) **Clave de dict/set** (debe ser hashable). 2) **Registro fijo** donde el orden y la longitud son contrato (coordenadas, filas de DB, retornos múltiples): la inmutabilidad documenta "esto no cambia" y lo hace seguro para compartir. 3) Como clave, **más rápida** de hashear en caché. Si vas a mutar, es lista — no "protejas" convirtiendo a tuple si el contenido es mutable igual.

### `namedtuple` y `NamedTuple`: tuplas con nombre

```python
from typing import NamedTuple
from collections import namedtuple

class Punto(NamedTuple):
    x: float
    y: float
    etiqueta: str = "P"

p = Punto(1.0, 2.0)
p.x, p[0], p.etiqueta         # atributo + indexación + default
Punto._fields                  # ('x', 'y', 'etiqueta')
Punto._make((3, 4))            # construcción desde iterable
p._asdict()                    # dict ordenado (serializable a JSON)
p._replace(x=9)                # devuelve copia modificada (la vía "mutable")
```

`collections.namedtuple("Punto", "x y")` es la forma dinámica (sin class body). `typing.NamedTuple` añade anotaciones y herencia de campos. Peso: una namedtuple pesa ~lo mismo que la tupla y menos que un dataclass sin `slots` — ver [[Python - Programación Orientada a Objetos]] para cuándo preferir `@dataclass`.

## `set` y `frozenset`: matemática de conjuntos

Los sets almacenan **claves hasheables únicas** en una tabla hash: pertenencia O(1) pero **sin orden garantizado** (la iteración sigue el orden interno de la tabla, que puede cambiar entre ejecuciones — no confíes en él para salida estable).

```python
s = {1, 2, 3}
s = set([1, 2, 2, 3])           # {1, 2, 3} — deduplica
s = set("hola")                 # {'h','o','l','a'}
s = {x**2 for x in range(5)}    # {0, 1, 4, 9, 16}
# {[1,2]}  → TypeError: unhashable type: 'list'  (los elementos deben ser hashables)
```

| Operación | Set | Método equivalente | Complejidad |
|---|---|---|---|
| Unión | `a \| b` | `a.union(b)` | O(len(a)+len(b)) |
| Intersección | `a & b` | `a.intersection(b)` | O(min) |
| Diferencia | `a - b` | `a.difference(b)` | O(len(a)) |
| Diferencia simétrica | `a ^ b` | `a.symmetric_difference(b)` | O(len(a)+len(b)) |
| Subconjunto | `a <= b`, `a < b` | `a.issubset(b)` | O(len(a)) |
| Superconjunto | `a >= b`, `a > b` | `a.issuperset(b)` | O(len(a)) |
| Disjuntos | `a.isdisjoint(b)` | — | O(min) |

Métodos mutables: `add(x)` (no lanza si existe), `update(iterable)` (acepta varios), `discard(x)` (**no** lanza si no existe), `remove(x)` (KeyError si no existe), `pop()` (elimina y devuelve **un elemento arbitrario** — sorpresa si lo usas en bucle), `clear()`, `difference_update`, `symmetric_difference_update`.

```python
a, b = {1, 2, 3}, {3, 4, 5}
a | b   # {1,2,3,4,5}    a & b  # {3}    a - b  # {1,2}    a ^ b  # {1,2,4,5}
{frozenset([1,2]), frozenset([3])}   # set de sets: solo con frozenset (verificado)
```

### `frozenset`

Versión inmutable y **hashable**: sirve como clave de dict o elemento de set, y como vista constante de conjuntos (p.ej. permisos de un usuario). Sin métodos mutadores — toda operación devuelve frozenset nuevo.

### Trampas de rendimiento

```python
l = list(range(10000)); s = set(range(10000))
# timeit verificado (100k / 1k iteraciones respectivas):
# 9999 in l  ≈ 48.6 µs/op   (O(n), recorre la lista)
# 9999 in s  ≈  0.019 µs/op (O(1) hash)
# → el set es ~2500× más rápido para pertenencia
```

Regla: cualquier `if x in lista` dentro de un bucle = señal para convertir la lista a set una vez (`s = set(l)`).

## `dict`: el tipo más importante del lenguaje

### Modelo interno

CPython 3.7+ implementa dict como **hash table con orden de inserción garantizado** (sección de entradas compactada + tabla de índices). Implicaciones: `d["k"]` es O(1) amortizado, la iteración sigue el orden de inserción **verificado**, y `del` + reinsertar mueve la clave al final:

```python
d = {}; d["b"] = 1; d["a"] = 2; d["c"] = 3
print(list(d))              # verificado: ['b', 'a', 'c'] — orden de inserción
del d["b"]; d["b"] = 4
print(list(d))              # verificado: ['a', 'c', 'b'] — ¡"b" ahora es la última!
```

### Creación

```python
d = {"a": 1, "b": 2}
d = dict(a=1, b=2)                      # claves como identifiers
d = dict([("a", 1), ("b", 2)])          # de pares (clave, valor)
d = dict(zip(claves, valores))          # de dos listas paralelas
d = {x: x**2 for x in range(5)}         # dict comprehension
d = dict.fromkeys("abc", 0)             # {'a':0,'b':0,'c':0}
```

> [!warning] `dict.fromkeys` comparte el valor por defecto
> Verificado: `bad = dict.fromkeys(["a","b"], []); bad["a"].append(1)` → `{'a': [1], 'b': [1]}` — las dos claves apuntan a la **misma** lista. Usá comprehension: `{k: [] for k in "ab"}`.

### Acceso y actualización (catálogo)

| Operación | Sintaxis | Si no existe | Nota |
|---|---|---|---|
| Acceso directo | `d["k"]` | **KeyError** | fallo ruidoso — bien para invariantes |
| Seguro | `d.get("k", default)` | `default` | `None` si no pasás default |
| Entrar/si no crear | `d.setdefault("k", [])` | crea y devuelve | evalúa el default SIEMPRE (coste) |
| Fusionar | `d1 \| d2` (3.9+) | — | nuevo dict; `d1 \|= d2` in-place |
| Actualizar | `d.update({"a":1}, b=2)` | — | mezcla dict + kwargs |
| Poblar desde cero | `d \| {"k": v}` | — | `{}\|d` es copia barata |
| Borrar | `del d["k"]`, `d.pop("k", default)`, `d.popitem()` | KeyError / default / `()` vacío | `popitem` devuelve el **último** (k,v) en 3.7+ — verificado LIFO |

```python
a_ = {"x": 1, "y": 2}; b_ = {"y": 9, "z": 3}
print(a_ | b_)              # verificado: {'x': 1, 'y': 9, 'z': 3} — la de la DERECHA gana
```

### Vistas dinámicas

`d.keys()`, `d.values()`, `d.items()` son **vistas vivas**: reflejan cambios del dict y se pueden combinar como conjuntos (`d1.keys() & d2.keys()`). No las materialices salvo que necesites indexar: `list(d)` ya itera claves.

```python
for k in d:             # claves (equivale a iter(d))
for v in d.values():    # valores
for k, v in d.items():  # pares — el patrón de iterating preferido
```

### Patrones avanzados

```python
# Diccionario anidado sin KeyError con setdefault
users.setdefault("ana", {}).setdefault("edad", 25)

# Agrupar sin collections (ingenuo)
for k, v in pares:
    grupos.setdefault(k, []).append(v)     # crea [] la primera vez

# Contar
frec = {}
for p in palabras: frec[p] = frec.get(p, 0) + 1

# Invertir
inv = {v: k for k, v in d.items()}          # claves únicas o pierde datos

# Dict como switch/registry (patrón clave)
ACCIONES = {"start": arrancar, "stop": detener}
ACCIONES.get(comando, lambda: print("desconocido"))()
```

## `collections`: las especializadas

### `defaultdict(factory)` — agrupar/conteo sin ruido

```python
from collections import defaultdict
dd = defaultdict(list)
dd["a"].append(1); dd["a"].append(2)     # crea [] automáticamente — verificado [1, 2]
dd_int = defaultdict(int)
dd_int["x"] += 1                          # 0 + 1 = 1
dd_set = defaultdict(set)
```

**Cuidado**: leer una clave faltante **crea** la entrada con la factory — `dd["no_existe"]` en un `if clave in dd` previo igual puede meter la clave si te despistas, y `defaultdict` rompe el contrato de `dict.get` en serializers que iteran. Alternativa sin efectos secundarios: `d.setdefault(k, []).append(v)` o `{k: f(v) for ...}`.

### `Counter(iterable)` — histogramas y operaciones aritméticas

```python
from collections import Counter
c = Counter("abracadabra")     # verificado: Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
c.most_common(2)               # [('a', 5), ('b', 2)]
c["a"], c.get("z", 0)          # 5, 0 — c["z"] no crea entrada (Counter especial)
c + Counter("abc")             # suma contadores: {'a':6, 'b':3, 'c':2, ...}
c - Counter("aa")              # resta: elimina claves ≤ 0
c.subtract("aaaa")             # resta INCLUSIVA: puede dejar contadores NEGATIVOS
list(c.elements())             # repite según cuenta: ['a','a','a','a','a','b','b',...]
Counter(palabras).most_common(3)   # top-3 directo
Counter(map(len, lineas)).most_common(1)  # línea más frecuente
```

> [!tip] `Counter` para dos listas
> - `c1 & c2` = mínimo por clave (intersección); `c1 | c2` = máximo (unión).
> - `+c1` elimina negativos/ceros; `(c1 - c2)` descarta no positivos mientras que `subtract` conserva negativos (verificado).

### `deque(maxlen=...)` — cola O(1) en ambos extremos

```python
from collections import deque
dq = deque([1, 2, 3])
dq.append(4); dq.appendleft(0)   # extremos O(1); insert/pop central O(n)
dq.pop(); dq.popleft()
dq.rotate(1)                      # derecha; rotate(-1) izquierda
dq = deque(maxlen=3)              # verificado: extend([1..5]) → deque([3,4,5])
```

| Uso | Patrón |
|---|---|
| Queue FIFO | `q.append(x); q.popleft()` |
| Stack LIFO | `s.append(x); s.pop()` |
| Historial N-últimos | `deque(maxlen=N)` — la memoria se corta sola |
| BFS (búsqueda) | `cola.popleft()` + `cola.append(hijo)` |
| Rotación/round-robin | `dq.rotate(-1)` da el siguiente turno |

Para colas **entre hilos** usá `queue.Queue` (thread-safe, con bloqueo) — ver [[Python - Concurrencia Threading Multiprocessing Asyncio]].

### `OrderedDict` — cuándo sigue haciendo falta

El dict normal ya preserva orden (3.7+, garantía de lenguaje desde 3.8). `OrderedDict` queda justificado por:

1. `move_to_end(key, last=True/False)` — reordenar (implementar LRU cache a mano).
2. `reversed(od)` — iteración inversa eficiente (dict normal: O(n) desde 3.8 también, pero él nació para eso).
3. Comparaciones que **inlcuyen el orden**: `OrderedDict([("a",1),("b",2)]) == OrderedDict([("b",2),("a",1)])` → `False` (dict normal: `True`).

### `ChainMap` — capas de configuración

```python
from collections import ChainMap
cfg = ChainMap({"color": "azul"}, {"color": "rojo", "size": 10})
cfg["color"], cfg["size"]   # verificado: 'azul', 10 — busca en orden, primera gana
len(cfg)                    # verificado: 2 — len cuenta SOLO el primer dict
cfg["nuevo"] = 1            # verificado: la escritura va al PRIMER dict
```

Patrón canónico CLI: `ChainMap(cli_args, env_vars, defaults)`. `maps` es la lista de dicts subyacente; `cfg.new_child()` añade capa.

## `dataclasses` — structs de datos en 3 líneas

```python
from dataclasses import dataclass, field, asdict, replace

@dataclass
class Usuario:
    nombre: str
    edad: int = 18
    tags: list[str] = field(default_factory=list)   # mutable => factory obligatoria

u = Usuario("Juan")          # __init__ generado
u                            # __repr__ generado: Usuario(nombre='Juan', edad=18, tags=[])
Usuario("a", 1) == Usuario("a", 1)   # __eq__ generado por campo
```

Flags clave (profundidad completa en [[Python - Programación Orientada a Objetos]]):

| Flag | Efecto | Nota verificada |
|---|---|---|
| `frozen=True` | inmutable + `__hash__` | solo impide **reasignar**: `f.nums = []` → `AttributeError`, pero `f.nums.append(1)` funciona |
| `order=True` | `<`, `>`, `<=`, `>=` por campo | orden lexicográfico de campos |
| `slots=True` (3.10+) | sin `__dict__` | ahorra memoria y acelera |
| `kw_only=True` (3.10+) | todos por keyword | herencia sin conflicto de defaults |
| `field(init=False)` | no va al `__init__` | para valores calculados |
| `field(compare=False)` | fuera del `__eq__`/orden | p.ej. cache fields |
| `@dataclass(slots=True)` + mutable default | — | `field(default_factory=list)` sigue obligatorio |

Herencia: los campos de la base van primero en `__init__`; sobreescribir un campo en subclase lo reordena detrás. `asdict()`, `astuple()`, `replace()`, `fields()` completan la utilidad.

## Copias: shallow vs deep (la semántica completa)

```python
import copy
original = [[1, 2], [3, 4]]

shallow = original.copy()      # lista nueva, PERO sus 2 elementos son los MISMOS objetos
shallow[0].append(99)
print(original)                # verificado: [[1, 2, 99], [3, 4]] — ¡mutó el original!

deep = copy.deepcopy(original)  # copia recursiva (con memo para ciclos)
deep[0].append(88)
print(original, deep)           # verificado: original intacto, deep aislado
```

| Método | Copia | Cuándo |
|---|---|---|
| `l.copy()`, `list(l)`, `l[:]`, `copy.copy(l)` | shallow | contenedores "planos" (elementos inmutables) |
| `copy.deepcopy(l)` | profunda | estructuras anidadas que mutarás |
| `l2 = [*l1]`, `{**d1}` | shallow (unpack) | literal conciso |
| `d.copy()`, `d1 \| d2` | shallow | dict |
| `dict(d)` | copia barata | ≈ `{}\|d` |

Regla práctica: si todos los elementos son `int/str/tuple de inmutables/frozenset`, shallow es suficiente; si hay listas/dicts/objetos anidados dentro, necesitás deep **o** diseño inmutable (tuplas, NamedTuple, dataclass frozen).

## Cuándo usar qué: tabla de decisión

| Situación | Elegir | Por qué |
|---|---|---|
| Lista de trabajo que muta | `list` | O(1) append, sort in-place |
| Registro fijo / coordenadas / retorno múltiple | `tuple` | inmutable, hashable, liviano |
| Clave de dict | `tuple` (inmutables) o `frozenset` | hasheable |
| "¿Está este elemento?" en bucle | `set` | O(1) vs O(n) — verificado 2500× |
| Deduplicar | `set` / `dict.fromkeys` | orden-preserving desde 3.7 con dict |
| Contar ocurrencias | `Counter` | + most_common, aritmética |
| Agrupar por clave | `defaultdict(list)` | sin `if` ni `KeyError` |
| Cola FIFO / stack / historial | `deque(maxlen=n)` | O(1) extremos, memoria acotada |
| Mapeo clave→valor | `dict` | O(1), orden de inserción |
| Config en capas | `ChainMap` | precedencia sin merge destructivo |
| Fila de DB con nombre | `NamedTuple` o `dataclass` | acceso legible + serializable |
| Objeto con comportamiento | `class`/`@dataclass` | ver POO |

## Complejidad: tabla de operaciones comunes

| Operación | `list` | `tuple` | `set` | `dict` | `deque` |
|---|---|---|---|---|---|
| Acceso `l[i]` / `d[k]` | O(1) | O(1) | ❌ | **O(1)** | O(n) / ❌ |
| `in` (pertenenia) | O(n) | O(n) | **O(1)** | **O(1)** | O(n) |
| Append final | **O(1) amort.** | ❌ | O(1) | O(1) | O(1) |
| Insert/inicio | O(n) | ❌ | ❌ | ❌ | **O(1)** |
| Pop final | O(1) | ❌ | O(1) | O(1) | O(1) |
| Pop inicio | O(n) | ❌ | ❌ | ❌ | **O(1)** |
| Borrar medio `del l[i]` | O(n) | ❌ | O(1) | O(1) | O(n) |
| Iterar completa | O(n) | O(n) | O(n) | O(n) | O(n) |
| Búsqueda `index`/`get` | O(n) | O(n) | O(1) add/discard | O(1) | O(n) |

Verificado con `timeit` en 3.14 (n=10 000): pertenencia en lista ≈ 48.6 µs vs set ≈ 0.019 µs. `in` de dict comprueba **claves** (usa `d.values()` si necesitas valores, eso sí es O(n)).

## Iteración: patrones idiomáticos

```python
for i, v in enumerate(lista, 1):        # con índice starting en 1
for k, v in d.items():                  # claves+valores
for a, b in zip(l1, l2, strict=True):   # paralelo con validación (3.10+)
for item in sorted(d.values(), key=...):# ordenar por valor
for clave, grupo in groupby(datos, key=lambda x: x[0]):  # requiere input ORDENADO
it = iter(lista); next(it, default)     # next seguro con default
```

```python
# Bucle infinito sobre un ciclo (cycle) con salida
from itertools import cycle
for intento in cycle(range(3)):
    ...
    if condicion_de_salida: break

# Reverse sin copiar grande: reversed() es O(1) en seq y dicts (3.8+)
for v in reversed(lista): pass
```

> [!warning] `groupby` exige input ordenado por la key (verificado)
> `list(groupby([("a",1),("b",2),("a",3)], key=…))` produce **tres** grupos (a, b, a) porque solo agrupa consecutivos. Primero: `sorted(datos, key=clave)`.

Errores de iteración típicos: mutar la colección mientras iteras (`RuntimeError: dictionary changed size during iteration` — itera sobre `list(d)` o `.items()` materializado), y `for x in d.keys(): del d[x]` (usa `for k in [k for k in d if cond]: del d[k]`).

## Errores comunes

1. **`IndexError`** en `l[len(l)]` — el último índice es `len(l)-1`; `l[len(l):]` en cambio no falla.
2. **`KeyError`** por confundir acceso con `get`: `d[k]` lanza; `d.get(k, default)` no.
3. **`TypeError: unhashable type`** — `set([1, [2]])`, `{[1]: 2}`, tuple con lista como clave. Solución: convertir a tuple inmutable o usar listas de pares.
4. **`AttributeError` con `deque`/`set`**: no soportan indexación ni slicing (`dq[0]` falla) — usa `itertools.islice` o convierte.
5. **Modificar durante iteración** — `RuntimeError` o resultados fantasma. Copiá primero o construí nueva colección.
6. **Comparar `Counter`/`set` con `==` a `list`**: `Counter("aa") == {"a": 2}` → `True` (Counter es dict), pero `Counter("aa") == ["a","a"]` → `False`.
7. **`list.pop(0)` en bucle** — O(n) por pop = O(n²) total; usa `deque.popleft()` o índices.
8. **Copiar shallow y mutar anidado** — el bug silencioso de la sección de copias.
9. **Orden de set en salida determinista** — itera `sorted(s)` si el orden importa (logs, tests).
10. **`d.values()` como `in`** — pertenencia en valores es O(n); invertí o usa set auxiliar.

## Rendimiento: micro-benchmark de referencia

```python
import timeit
# verificado en 3.14:
timeit.timeit("l[5000]", "l=list(range(10000))", number=100_000)   # 0.0010 s → index O(1)
timeit.timeit("9999 in l", "l=list(range(10000))", number=1_000)   # 0.0487 s → in O(n)
timeit.timeit("9999 in s", "s=set(range(10000))", number=100_000)  # 0.0019 s → in O(1)
```

- Comprensiones > `map+lambda` para transformación simple; `map` gana cuando el callable ya existe en C.
- `+` de listas en bucle es O(n²): usá `extend` o acumulá en lista y `join`/`sum(..., [])` con cuidado (preferí listas de batches + `itertools.chain.from_iterable`).
- `sorted(key=dict.get)` o precalcular keys cuando la key es cara (Schwartzian transform: `sorted(items, key=lambda x: (key(x), x))` ya lo hace Python internamente con la key decorada — solo necesitás pasarla).

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — semántica de nombres, objetos y truthiness
- [[Python - Programación Orientada a Objetos]] — protocolos `__len__`, `__getitem__`, `__iter__` que hacen "colecciones" a tus clases
- [[Python - Librerías Estándar Esenciales]] — `itertools`, `functools`, `heapq`, `bisect`
- [[Python - Manejo de Archivos y Serialización]] — qué tipos serializa JSON/pickle
- [[Python - Referencia Rápida (Cheatsheet)]] — tarjeta de métodos de colecciones
- [[Python - Patrones, Buenas Prácticas y Tipado]] — anotaciones de tipos para colecciones
