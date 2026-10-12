---
aliases: ["Fundamentos Python", "Python Sintaxis", "Python Fundamentos"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Estructuras de Datos]]"
  - "[[Python - Programación Orientada a Objetos]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
  - "[[Python - Módulos, Paquetes y Distribución]]"
  - "[[Python - Referencia Rápida (Cheatsheet)]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python - Concurrencia Threading Multiprocessing Asyncio]]"
  - "[[Node JS]]"
fuente: "https://docs.python.org/es/3/reference/index.html"
---

# Python - Fundamentos y Sintaxis

Referencia profunda del lenguaje: modelo de tipado, semántica de indentación, el objeto por debajo de cada variable, tipos numéricos y de texto, operadores, verdad/falsedad, control de flujo y funciones. Todo lo que sigue fue verificado con Python 3.14 a menos que se indique otra versión. Para colecciones ver [[Python - Estructuras de Datos]]; para clases, [[Python - Programación Orientada a Objetos]].

## Ejecución: shebang, script y módulo

```python
#!/usr/bin/env python3   # portable: busca python3 en el PATH
```

| Forma | Qué hace | Nota |
|---|---|---|
| `python3 script.py` | Ejecuta el archivo; `__name__ == "__main__"` | `sys.path[0]` = directorio del script |
| `python3 -m modulo` | Ejecuta un módulo del sys.path | `sys.path[0]` = directorio de trabajo (verificado: `''`) — permite `python -m http.server` desde cualquier sitio |
| `python3 -c "codigo"` | Ejecuta código inline | útil para probes rápidos |
| `python3 script.py < args` | Redirecciona stdin | `input()` lee del archivo, no de la terminal |

```python
# /tmp/imp_test/main_script.py (verificado)
import sys
print(__name__)            # "__main__" al correr directamente
                           # "main_script" al hacer python -c "import main_script"
```

> [!info] `if __name__ == "__main__":` — por qué existe
> Python ejecuta **cualquier** código de nivel superior al importar. El guard separa el comportamiento de *script* (se ejecuta) del de *módulo* (solo define). Es lo que permite usar un archivo como librería **y** como CLI sin tocarlo (ver [[Python - Módulos, Paquetes y Distribución]]).

## Modelo de datos: todo es un objeto

Python es un lenguaje de **tipado dinámico y fuerte**:

- **Dinámico**: el tipo vive en el objeto, no en la variable. No se declara; nada impide reasignar un nombre a un objeto de otro tipo.
- **Fuerte**: no hay coerción implícita entre tipos incompatibles (`"a" + 1` lanza `TypeError`, no inventa un número).

### Nombres no son variables: son etiquetas sobre objetos

```python
x = 42          # x es un nombre (entrada en una tabla de symtable) que apunta al objeto 42
x = "hola"      # se re-enlaza x a otro objeto; el 42 queda sin referencias y el GC lo recoge
print(id(x))    # id() = dirección CPython del objeto (verificado: 140459189557392)
```

```python
a = []
b = a          # b apunta al MISMO objeto (no hay copia)
b.append(1)
print(a is b, a)   # verificado: True [1] — mutar por un alias muta el original
```

> [!warning] El bug #1 de Python: aliasing accidental
> Pasar listas/dicts a funciones **no las copia**. Si la función muta el argumento, el caller lo ve. Regla: si el callee debe modificar, que devuelva algo nuevo; si solo lee, documentarlo. Detalle completo en [[Python - Estructuras de Datos]] (sección Copias).

### Identidad vs igualdad vs tipo

| Expresión | Pregunta | Ejemplo |
|---|---|---|
| `a is b` | ¿el **mismo objeto**? (compara direcciones) | `x is None` — siempre `is` con None |
| `a == b` | ¿mismo valor/semántica? (llama a `__eq__`) | `[] == []` → `True` aunque sean objetos distintos |
| `type(a)` | ¿clase exacta? | `type(True)` → `<class 'bool'>` (subclase de `int`) |
| `isinstance(a, B)` | ¿instancia de B **o subclase**? | `isinstance(True, int)` → `True` |

CPython cachea los enteros pequeños (-5…256) y algunas cadenas, así que `256 is 256` puede dar `True` mientras `257 is 257` da `False`: **nunca uses `is` para comparar valores**. Verificado:

```python
>>> 257 is 257, 257 == 257
(False, True)
```

## Indentación: semántica, no estilo

En Python los bloques se delimitan con **espacios al inicio de línea**, no con llaves. El tokenizador agrupa líneas por indentación creciente; la dedentación cierra bloques.

- Regla PEP 8: **4 espacios** por nivel. Nunca tabulaciones (mezclar ambos lanza `TabError`).
- Los dos puntos `:` abren un bloque indentado: `if`, `for`, `while`, `def`, `class`, `with`, `try`, `match`.
- `...` (`Ellipsis`) o `pass` son placeholders válidos donde falta cuerpo.

```python
if True:
       x = 1      # cualquier indentación >0 funciona...
       y = 2      # ...pero debe ser CONSISTENTE dentro del bloque
else:
   z = 3           # TabError/IndentationError si se mezclan tabs y spaces
```

## Convenciones de nombres (PEP 8)

| Tipo | Convención | Ejemplo |
|---|---|---|
| Variables, funciones, módulos | `snake_case` | `mi_variable`, `calcular_total()` |
| Constantes | `UPPER_SNAKE` | `MAX_REINTENTOS = 3` (solo convención; Python no las congela) |
| Clases | `PascalCase` | `GestorUsuarios` |
| Protegido (convención) | `_prefijo` | `_interno` — el intérprete no impide acceder |
| Privado (name mangling) | `__prefijo` | `__privado` → `obj._Clase__privado`, ver [[Python - Programación Orientada a Objetos]] |
| Magic methods | `__nombre__` | `__init__`, `__str__` |

## Tipos numéricos

### `int`: precisión arbitraria

Los enteros de Python tienen **tamaño ilimitado**: crecen hasta agotar la memoria. Verificado en 3.14:

```python
>>> big = 10**30 + 7
>>> big
1000000000000000000000000000007
>>> big.bit_length()
100                       # bits necesarios: O(log n), no hay overflow jamás
>>> str(2**1000)[:12], len(str(2**1000))
('107150860718', 302)     # 2**1000 tiene 302 dígitos decimales
```

Esto tiene costo: operar con enteros gigantes es O(n) en el número de dígitos, y `int` ocupa 28 bytes mínimo (vs 4-8 en C). Para datos tabulares masivos usa `array`, NumPy o `int.bit_count()`/`bit_length()` (3.10+: `(0b1011).bit_count()` → `3`, verificado).

### `float`: IEEE 754 doble precisión — la trampa

```python
>>> 0.1 + 0.2
0.30000000000000004
>>> 0.1 + 0.2 == 0.3
False
```

0.1 no es representable en binario (como 1/3 en decimal): Python imprime la representación decimal más corta que round-tripea, pero **por debajo almacena otro número**. Consecuencias prácticas:

| Situación | Mala idea | Correcto |
|---|---|---|
| Comparar importes | `total == 19.99` | `math.isclose(total, 19.99)` o enteros en centavos |
| Dinero | `float` | `Decimal("19.99")` (`decimal`) o `Fraction` |
| Redondear | `round(x, 2)` (round-half-even binario) | `Decimal(x).quantize(Decimal("0.01"))` |

```python
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')            # exacto: decimal trabaja en base 10
```

### `complex` y literales numéricos

```python
z = 1 + 2j                # complex: z.real=1.0, z.imag=2.0, abs(z)=2.236...
n1, n2 = 1_000_000, 0xFF_FF  # separador _ ; hex/oct/bin literales
0b1010, 0o17, 1e3          # (10, 15, 1000.0) — 1e3 es float, no int
True + True                # 2 → bool ES subclase de int
```

| Tipo | Tamaño | Rango/Notas | Litral |
|---|---|---|---|
| `int` | dinámico (28 B mín.) | ilimitado | `42`, `1_000`, `0xFF` |
| `float` | 8 B | ±1.8e308, 15-16 dígitos significativos | `3.14`, `1e-9` |
| `complex` | 16 B | `z.real`, `z.imag` | `3+4j` |
| `bool` | subclase de `int` | `True==1`, `False==0` | `True`/`False` |

## Cadenas: inmutabilidad y métodos

`str` es **inmutable**: cada "modificación" crea un objeto nuevo. Verificado:

```python
>>> s = "hola"; s[0] = "H"
TypeError: 'str' object does not support item assignment
```

### Catálogo de métodos (N completo, no muestra)

| Método | Comportamiento | Nota clave |
|---|---|---|
| `s.upper()/lower()/swapcase()/title()` | cambio de caso | `title()` falla con `don't` → `Don'T` |
| `s.strip(chars)/lstrip()/rstrip()` | quita extremos | por defecto espacios y `\n\r\t\v\f` |
| `s.removeprefix(p)/removesuffix(p)` | quita prefijo/sufijo si están | 3.9+; verificado: `'aba'.removeprefix('a').removesuffix('a')` → `'b'` |
| `s.split(sep=None)` | parte | `sep=None` colapsa blancos; sin límite |
| `s.rsplit(sep, maxsplit)` | parte desde la derecha | útil con `maxsplit=1` |
| `s.splitlines()` | por líneas | maneja `\n`, `\r\n`, `\x0b`… |
| `sep.join(iterable)` | une | `"-".join(["a","b"])` → `"a-b"`; el separador va primero |
| `s.replace(old, new, count)` | reemplaza | devuelve **nueva** cadena |
| `s.find(sub)/rfind(sub)` | índice o **-1** | no lanza excepción |
| `s.index(sub)` | índice o **ValueError** | use `find` si puede fallar |
| `s.count(sub)` | no-solapadas | `"aaa".count("aa")` → `1` |
| `s.startswith(p)/endswith(p)` | bool | admite tupla: `s.endswith((".png", ".jpg"))` |
| `s.center(w)/ljust(w)/rjust(w)` | alineación | base de los specs `^ < >` |
| `s.isalpha()/isdigit()/isalnum()/isspace()/isupper()` | clasificación | `"".isdigit()` → `False`; `"²".isdigit()` → `True` |
| `s.encode(encoding)` | `bytes` | ver Codificación abajo |
| `f"{s:>10}"` etc. | formato | ver f-strings |

### F-strings en profundidad (3.6+; 3.12 revolucionó la gramática)

```python
name, value = "cpu", 0.91234
d = {"a": {"b": 42}}

f"{name.upper()=}"        # verificado: name.upper()='CPU'  (el = refleja la expresión)
f"{value:.2%}"            # verificado: 91.23%
f"{value = :.3f}"         # verificado: value = 0.912  (espacio antes de :, =, !)
f"{d['a']['b']}"          # comillas simples DENTRO: 3.12+ lo permite
f"{x!r:>{10}}"            # conversión + ancho dinámico: '   texto' (con x='texto')
```

| Parte | Significado | Ejemplo → salida |
|---|---|---|
| `!r` / `!s` / `!a` | `repr()` / `str()` / `ascii()` | `f"{'hola'!r}"` → `'hola'` (con comillas) |
| `=` (3.8+) | refleja la expresión + resultado | `f"{a+b=}"` → `a+b=3` |
| `:{spec}` | formato de formato estándar | `f"{3.14159:.2f}"` → `3.14` |
| `{expr!r:>{w}}` | conversión + spec combinados | padding dinámico |
| Anidamiento | `{...{...}...}` y comillas mixtas | 3.12+: sin escapar, f-strings multilínea |

Specs de formato más usados: `>10` derecha, `<10` izquierda, `^10` centro, `0>5` relleno ceros, `,` separador de miles, `.2f` decimales, `.0%` porcentaje, `+g` signo explícito. Todo el catálogo en [[Python - Referencia Rápida (Cheatsheet)]].

### Codificación y bytes

```python
"ñandú".encode("utf-8")        # b'\xc3\xb1and\xc3\xba' — UTF-8 es el default de Python 3
b'\xc3\xb1'.decode("utf-8")    # 'ñ'
"café".encode("ascii")         # UnicodeEncodeError
```

- **Todo** `str` en memoria es Unicode; los `bytes` son datos crudos. Python 3 no mezcla ambos sin `decode`/`encode` explícito.
- El default del intérprete es UTF-8 en casi todas las plataformas modernas; con locales viejos (CP1252, Latin-1) puede fallar — fijá `PYTHONUTF8=1` o pasá `encoding="utf-8"` a `open()`. Detalle en [[Python - Manejo de Archivos y Serialización]].

## Operadores

### Aritméticos: `/` vs `//` y el suelo negativo

```python
>>> 7 / 3, 7 // 3, 7 % 3
(2.3333333333333335, 2, 1)
>>> -7 // 3, 7 % -3          # // es DIVISIÓN ENTERA (floor), no truncamiento
(-3, -2)                     # verificado: cociente ≤ resto real, `//` siempre redondea hacia -∞
>>> import math; math.floor(-2.5)
-3                           # `//` == math.floor(a/b) para enteros
```

| Operador | Significado | Verificado |
|---|---|---|
| `a / b` | división exacta → **siempre** `float` | `7/3` → `2.333...` |
| `a // b` | cociente floor | `-7 // 3` → `-3` |
| `a % b` | resto con **signo del divisor**: `a - (a//b)*b` | `7 % -3` → `-2` |
| `a ** b` | potencia; `2**1000` es exacto | también `pow(a, b, mod)` eficiente modular |
| `a @` | matmul | lo implementan NumPy/listas 3.9+ via `__matmul__` |
| `divmod(a, b)` | `(a//b, a%b)` | evita calcular dos veces |

### Comparaciones encadenadas (verificadas)

```python
>>> x = 5
>>> 1 < x < 10            # equivale a (1 < x) and (x < 10) — x se evalúa UNA vez
True
>>> a, b, c = 1, 1, 2
>>> a == b < c            # (a==b) and (b<c) → True
True
```

Python evalúa `1 < x < 10` como `1 < x and x < 10` con `x` evaluado una sola vez (importante si `x` es una llamada con efectos). Otros ejemplos útiles: `0 <= len(items) <= 10`, `lo is not None is not otro`.

### Lógicos: `and`/`or` devuelven operandos, no bool

```python
>>> "" or "default"
'default'          # or devuelve el primer operandotruthy (o el último)
>>> [1] and [2]
[2]                # and devuelve el primer falsy o el último
>>> x = potential or None   # patrón de default + asignación
```

`not` sí devuelve `bool`. Cortocircuito: el segundo operando solo se evalúa si hace falta.

### Operadores de membresía y identidad

```python
"x" in ["x", "y"]        # True — O(n) en listas, O(1) en sets/dicts
"a" in {"a": 1}          # comprueba CLAVES, no valores
not in                   # negación directa
x is None / x is not None  # único uso correcto de `is`
```

## Truthiness: la tabla completa

Todo objeto Python define truthiness; por defecto es `True` salvo para los falsy:

| Falsy (verificado `bool()` → `False`) | Truthy típico |
|---|---|
| `False`, `None` | `True`, cualquier objeto |
| `0`, `0.0`, `0j` | números ≠ 0 |
| `""`, `b""` | cadenas no vacías |
| `[]`, `()`, `set()`, `dict()`, `frozenset()` | colecciones no vacías |
| objetos con `__bool__()` → `False` o `__len__()` → `0` | el resto |

```python
class P:
    def __bool__(self): return False
bool(P())   # verificado: False — tus clases pueden definir su verdad
```

```python
if not lista:          # pythonico para "vacía" — O(1), mejor que len(lista) == 0
if valor is None:      # None se comprueba con `is`, nunca ==
```

> [!tip] `or` para defaults con cuidado
> `x = entrada or 5` trata `0`, `""` y `[]` como "sin valor". Si el 0 es válido: `x = 5 if entrada is None else entrada`.

## `input()` y E/S básica

```python
nombre = input("Nombre: ")        # lee una línea de stdin; SIEMPRE devuelve str
edad = int(input("Edad: "))       # conversión explícita — ValueError si no es número
from getpass import getpass
pwd = getpass("Contraseña: ")     # sin eco en terminal

print("a", "b", sep="-", end="!\n")   # sep/end como print options
print("error", file=sys.stderr)       # stderr requiere import sys
```

`input()` lanza `EOFError` con stdin cerrado (redirección vacía) — envolverlo en `try` para CLIs robustas. Para argumentos de línea de comandos reales se usa `argparse` (ver [[Python - Librerías Estándar Esenciales]]).

## Control de flujo

### `if / elif / else`

```python
if x > 0:
    msg = "positivo"
elif x < 0:
    msg = "negativo"
else:
    msg = "cero"

msg = "positivo" if x > 0 else "no positivo"   # ternario: cond if condicion else cond
```

### `for`: itera sobre **iterables**, no sobre índices

El `for` de Python es un `for-each` sobre el protocolo de iteración (`__iter__`/`__next__`); no hay `for (i=0; i<n; i++)`.

```python
for i in range(5):             # 0..4 (range es lazy, no crea lista)
for i in range(2, 10, 2):      # 2,4,6,8  — range(a,b,step)
for i in range(10, 0, -2):     # paso negativo: 10,8,...,2
for i, v in enumerate(lista):  # índice + valor; enumerate(lista, 1) empieza en 1
for k, v in dic.items():       # clave + valor
for a, b in zip(l1, l2):       # paralelo; zip(..., strict=True) (3.10+) valida longitudes
for linea in archivo:          # el archivo es un iterador perezoso
```

```python
>>> list(zip([1,2],[3], strict=True))
ValueError: zip() argument 2 is shorter than argument 1    # verificado — atrapa bugs de longitudes
>>> from itertools import zip_longest
>>> list(zip_longest([1,2],[3], fillvalue=0))
[(1, 3), (2, 0)]                                            # rellena en vez de cortar
```

### `else` de bucle: el bloque que casi nadie conoce

El `else` de `for`/`while` se ejecuta **solo si el bucle terminó sin `break`**. Verificado:

```python
for n in range(3):
    pass
else:
    print("ejecutado (sin break)")      # se imprime

for n in range(3):
    break
else:
    print("NO debo aparecer")           # no se imprime
```

Patrón canónico — búsqueda sin flag:

```python
for patron in candidatos:
    if validar(patron):
        break
else:
    raise RuntimeError("ningún patrón válido")   # el else = "no se encontró nada"
```

### `while`

```python
while condicion:
    ...
else:
    ...   # mismo semantics: se ejecuta si la condición se volvió falsa (sin break)
```

`while True: ... break` es idiomático para bucles infinitos con salida interior. Para esperar con tiempo usa `time.sleep` (ver [[Python - Concurrencia Threading Multiprocessing Asyncio]] para variantes async).

### `break`, `continue`, `pass`

| Sentencia | Efecto | En `try/finally` |
|---|---|---|
| `break` | sale del bucle más interno | ejecuta `finally` antes de salir |
| `continue` | salta a la siguiente iteración | ejecuta `finally` antes de continuar (3.8+: sin warning) |
| `pass` | no-op; placeholder | — |

### `match / case` (3.10+) — desestructuración real

No es un switch: es **pattern matching** con desestructuración, guards y literales. Verificado en 3.14:

```python
def clasificar(c):
    match c:
        case "a" | "b":        return "letra"          # alternación de literales
        case [1, *resto]:      return f"lista {resto}"  # secuencia + rest capture
        case {"k": v}:         return f"dict {v}"       # mapping
        case _:                return "otro"            # comodín (obligatorio cubrir o usar guard)
>>> clasificar("a"), clasificar([1,2,3]), clasificar({"k": 5}), clasificar(9)
('letra', 'lista [2, 3]', 'dict 5', 'otro')
```

| Caso de patrón | Ejemplo | Captura |
|---|---|---|
| Literal | `case 0:` | — |
| Alternación | `case "q" \| "quit":` | — |
| Secuencia | `case [x, y]:` | longitudes exactas |
| Rest | `case [primero, *resto]:` | resto es lista |
| Mapping | `case {"id": i, **demas}:` | `**demas` es dict |
| Clase | `case Point(x=0, y=n):` | `Point(x=..., y=...)` o posicional |
| Guard | `case n if n < 0:` | condición adicional |
| Comodín | `case _:` | captura cualquier cosa |

> [!tip] `match` no valida tipos automáticamente
> `case [x, y]:` acepta **cualquier** secuencia (listas, tuplas, str…). Para tipos, usá `case Point(x, y):` con clases con `class_pattern` o guards explícitos (`case p if isinstance(p, Point):`).

## Funciones: los 6 modos de parámetros

```python
def f(a, b=1, /, c=2, *args, d, e=3, **kwargs):
    #   ↑pos-only  ↑kw con default      ↑kw-only  ↑extra kw
    pass
```

| Modo | Sintaxis | Se puede pasar como… | Ejemplo de llamada |
|---|---|---|---|
| Posicional (default) | `a` | posición o keyword | `f(1)` o `f(a=1)` |
| Posicional-only (3.8+) | `a /` | solo posición | `f(1)`; `f(a=1)` → `TypeError` |
| Keyword-only (3+) | `*, a` o `a=1` después de `*` | solo keyword | `f(a=1)` |
| Por defecto | `a=valor` | opcional | — |
| Var-positional | `*args` | tupla de extras | `f(1,2,3)` → `args=(2,3)` |
| Var-keyword | `**kwargs` | dict de extras | `f(a=1, z=9)` → `kwargs={'z':9}` |

```python
def ejemplo(a, b, *args, **kwargs):
    print(a, b, args, kwargs)
ejemplo(1, 2, 3, 4, x=10, y=20)
# verificado patrón: a=1 b=2 args=(3, 4) kwargs={'x': 10, 'y': 20}
```

Verificado:

```python
def f(a, b=10, *args, kw1=1, **kwargs): pass
f(1, 2, 3, 4, kw1=5, kw2=6)
# a=1, b=2, args=(3,4), kw1=5, kwargs={'kw2':6}
```

> [!warning] Default mutable
> `def f(lista=[])` comparte **una misma lista** entre todas las llamadas — bug clásico y silencioso. Usá `None` + creación interna, o `dataclasses.field(default_factory=list)`. Verificado con `dict.fromkeys(["a","b"], [])`: mutar `d["a"]` también muta `d["b"]`.

Alcance LEGB: los parámetros y variables locales son locales; para asignar una global hay que declarar `global x` (casi siempre señal de diseño a repensar). Closures capturan variables **por referencia**:

```python
def contador():
    n = 0
    def siguiente():
        nonlocal n        # asigna al n del closure, no crea local
        n += 1
        return n
    return siguiente
```

### Anotaciones de tipo (type hints)

```python
def procesar(items: list[str], config: dict[str, int], filtro: str | None = None) -> int | str: ...
x: int = 42
type Pares = list[tuple[int, int]]      # alias de tipo, 3.12+ (verificado)
```

Python **no valida** los hints en runtime (son metadata); los validan mypy/pyright y librerías como pydantic (ver [[Python - Patrones, Buenas Prácticas y Tipado]]). Sintaxis moderna: genéricos builtin (`list[int]`) desde 3.9, `X | Y` desde 3.10.

### `map`/`filter`/`lambda`

```python
list(map(lambda x: x**2, range(5)))         # [0,1,4,9,16]
list(filter(lambda x: x % 2 == 0, range(10)))
sorted(claves, key=lambda k: d[k])          # key define la comparación, no la mutación
```

`lambda` es una función anónima de **una expresión**. Para dos líneas, usá `def`. Regla práctica: si el `lambda` necesita lectura atenta, un `def` con nombre documenta mejor.

### Docstrings y documentación

```python
def func(a: int) -> str:
    """Resumen breve (una línea).

    Args:
        a: Descripción del parámetro.
    Returns:
        Descripción del retorno.
    Raises:
        ValueError: Si a < 0.
    """
```

Docstring = cadena literal primera dentro de módulo/clase/función; accesible como `__doc__`, base de `help()` y `pydoc`. Diferencia clave: **comentario** (`#`) es para el lector del código y Python lo ignora; **docstring** es dato en runtime. `python -m doctest archivo.py` ejecuta los ejemplos del docstring como tests (ver [[Python - Testing, Calidad y Herramientas]]).

## Manejo de errores: semántica completa

### Jerarquía (los que importan)

```
BaseException
 ├── KeyboardInterrupt   (Ctrl+C)
 ├── SystemExit           (sys.exit)
 ├── GeneratorExit
 └── Exception
      ├── ArithmeticError (ZeroDivisionError, OverflowError)
      ├── LookupError     (KeyError, IndexError)
      ├── OSError         (FileNotFoundError es subclase)
      ├── TypeError, ValueError, AttributeError
      ├── RuntimeError    (NotImplementedError aquí)
      └── Warning         (DeprecationWarning… — hereda de Exception, pero se emite, no se lanza)
```

### `try / except / else / finally`

```python
try:
    resultado = 10 / x
except ZeroDivisionError:
    print("división por cero")
except (TypeError, ValueError) as e:
    print(f"error de tipo/valor: {e}")
except Exception as e:                 # último recurso: NO pongas except vacío
    raise
else:
    print("solo si NO hubo excepción")  # lugar correcto del código que puede fallar
finally:
    print("siempre se ejecuta")          # liberar recursos (archivos, locks, conexiones)
```

| Bloque | Cuándo corre | Uso idiomático |
|---|---|---|
| `except` | al capturar | manejar y continuar |
| `else` | solo sin excepción | separar el happy path del manejo de errores |
| `finally` | siempre | limpieza incondicional |

`else` tras `try` evita el anti-patrón de bandera `hubo_error`, y hace el alcance de las variables claro: lo creado en `try` puede no existir en `except`.

### Lanzar y encadenar

```python
raise ValueError("x debe ser positivo")
try:
    int("abc")
except ValueError as e:
    raise MiError("config inválida") from e   # preserva __cause__ y el traceback
raise                                          # re-lanza la excepción capturada
```

Excepciones personalizadas: subclasear `Exception` (nunca `BaseException`), mantenerlas ligeras y nombrarlas `ErrorXxx`:

```python
class MiError(Exception): pass
class ErrorValidacion(MiError):
    def __init__(self, campo: str, mensaje: str):
        self.campo = campo
        super().__init__(f"{campo}: {mensaje}")
```

### `except*` (3.11+) — grupos de excepciones

```python
try:
    raise ExceptionGroup("varios", [ValueError("a"), TypeError("b")])
except* ValueError as eg:
    print("capturado ValueError:", eg.exceptions)   # verificado: (ValueError('a'),)
```

`except*` captura subconjuntos de un `ExceptionGroup` — base del soporte de errores parciales en `asyncio.TaskGroup` y en ejecución paralela de múltiples validaciones.

## Cheatsheet — Fundamentos

| Tema | Regla / sintaxis | Trampa típica |
|---|---|---|
| Tipado | dinámico y fuerte, tipo vive en el objeto | reasignar un nombre a otro tipo es legal pero rara vez intencional |
| Identidad | `is` para None; `==` para valores | `is` con enteros cacheados/strings |
| Enteros | precisión arbitraria | 28 bytes mínimo; O(n) con números enormes |
| Floats | IEEE 754 | `0.1+0.2 != 0.3`; dinero → `Decimal` |
| Cadenas | inmutables; todo devuelve nueva | `s.strip()` sin asignar = no-op |
| `//` y `%` | floor; resto con signo del divisor | `-7//3 == -3`, no `-2` |
| Comparaciones | encadenadas: `1 < x < 10` | evalúa `x` una sola vez |
| Truthiness | falsy: `0 "" [] {} None False` | `or` default trate `0` como vacío |
| `for` | itera iterables; `else` sin `break` | `for-else` se ignora al leer código |
| `match` | pattern matching 3.10+ | no valida tipos sin clase/guard |
| Args | `a, /, b=1, *args, c, **kwargs` | default mutable = bug |
| `try` | `except/else/finally` | `except: pass` silencia bugs |
| Errores | `raise ... from e` | nunca `except Exception: pass` |

## Errores comunes

1. **`NameError` vs `UnboundLocalError`**: `UnboundLocalError` es un `NameError` de variable local — aparece cuando asignás a un nombre en una función y lo leés antes: `print(x); x = 1` → `cannot access local variable 'x'`. Síntoma clásico en bucles que "olvidan" su acumulador.
2. **`IndentationError` / `TabError`**: mezclar tabs y spaces. Fijo con `python -X dev` + editor a 4 espacios.
3. **`TypeError: unsupported operand`**: `"a" + 1` — fuerte tipado; convertir con `str()`/`int()` explícito.
4. **`ValueError` en `int(input())`**: entrada no numérica. Envolver en `try` o usar `argparse` con `type=int` (valida y reporta).
5. **`AttributeError` en `None`**: `None.method()` — el sintoma de un `or` mal usado o un `.get()` sin default.
6. **Comparar floats con `==`**: usar `math.isclose(a, b, rel_tol=1e-9)`.
7. **`is` con valores**: `if x == "":` o `if not x:`; `is` solo para `None`/singleton intencional.
8. **Modificando lo que crees copiar**: `b = a` no copia; ver [[Python - Estructuras de Datos]] → Copias.
9. **`break`/`continue` fuera de bucle** → `SyntaxError`; **`return` dentro de `finally`** silencia la excepción pendiente — evitar.
10. **f-string con `=` y `:` mal espaciados**: `f"{x = :.2f}"` necesita el espacio antes de `:` (o no lo necesita: `f"{x=:06.2f}"` también vale — el `=` se detecta solo; la ambigüedad famosa es `f"{x=:.2f}"` que puede parsear mal en <3.12).

## Rendimiento y semántica de ejecución

- Python es interpretado por bytecode (`.pyc` en `__pycache__/`); no compiles mentalmente a C: un `for` puro en Python es ~50-100x más lento que en C. Optimizá eligiendo estructuras correctas (set/dict O(1)) antes que micro-optimizar.
- `range` es lazy; `list(range(n))` solo si necesitás indexar.
- Builtin calls (`sum`, `len`, `sorted`) están implementados en C: `sum(map(int, datos))` le gana a un loop manual en Python.
- La GIL, hilos y procesos: ver [[Python - Concurrencia Threading Multiprocessing Asyncio]].
- Python 3.13 trae **JIT experimental** y el nuevo **módulo `mimalloc`**; 3.14 añade **`experimental-jit`** por defecto opcional y free-threading (nogil) madurando — no cambia semántica, solo rendimiento.

## Recetas prácticas

```python
# 1. Contar y agrupar en una pasada (sin imports raros)
from collections import Counter
frec = Counter(palabra.lower().strip(".,;:") for linea in archivo for palabra in linea.split())

# 2. Validar rango con comparación encadenada
if not (0 <= indice < len(datos)):
    raise IndexError(f"índice {indice} fuera de rango")

# 3. Default seguro con `or` cuando el vacío también es default
pagina = args.get("pagina") or 1

# 4. Reintentos idiomáticos (else = tuvo éxito)
for intento in range(3):
    try:
        resultado = operacion_frágil()
    except Transitorio as e:
        esperar(2 ** intento)
    else:
        break
else:
    raise RuntimeError("sin éxito tras 3 intentos")

# 5. Desestructurar tuplas (swap sin variable temporal)
a, b = b, a

# 6. Tabla de dispatch sin if/elif gigantes
ACCIONES = {"start": arrancar, "stop": detener}
match comando:
    case [accion, *args] if accion in ACCIONES:
        ACCIONES[accion](*args)
    case _:
        print("desconocido")
```

## Relacionadas

- [[Python - Estructuras de Datos]] — list/tuple/set/dict y collections en profundidad
- [[Python - Programación Orientada a Objetos]] — clases, dunders, descriptors
- [[Python - Manejo de Archivos y Serialización]] — `open()`, pathlib, JSON, pickle
- [[Python - Módulos, Paquetes y Distribución]] — imports, `sys.path`, `pyproject.toml`
- [[Python - Librerías Estándar Esenciales]] — el tour completo del stdlib
- [[Python - Referencia Rápida (Cheatsheet)]] — la tarjeta ultra-condensada
- [[Python - Patrones, Buenas Prácticas y Tipado]] — EAFP, type hints, arquitectura
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — qué pasa cuando el `for` ya no alcanza
- [[Python - Testing, Calidad y Herramientas]] — pytest, ruff, mypy
- [[Python Ofensivo]] — los fundamentos aplicados a seguridad ofensiva
