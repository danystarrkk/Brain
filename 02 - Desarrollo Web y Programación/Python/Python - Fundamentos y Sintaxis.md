---
title: "Python - Fundamentos y Sintaxis"
tags:
  - lang/python
  - concept/fundamentals
  - concept/syntax
  - level/basico
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Python - Fundamentos y Sintaxis

> **Objetivo**: Base sólida del lenguaje: sintaxis, tipos, control de flujo, funciones y manejo de errores.

---

## 1. Shebang y Ejecución

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

- `#!/usr/bin/env python3` → Portable (busca python3 en PATH)
- `python3 script.py` → Ejecutar
- `python3 -m modulo` → Ejecutar módulo

---

## 2. Variables y Tipos Básicos

### 2.1 Tipado Dinámico

```python
# No se declara tipo; Python lo infiere en runtime
x = 42          # int
x = 3.14        # float
x = "hola"      # str
x = True        # bool
x = None        # NoneType
x = [1, 2, 3]   # list
x = (1, 2)      # tuple
x = {1, 2}      # set
x = {"a": 1}    # dict
```

### 2.2 Convenciones de Nombrado (PEP 8)

| Tipo | Convención | Ejemplo |
|------|------------|---------|
| Variables/funciones | `snake_case` | `mi_variable`, `calcular_total()` |
| Constantes | `UPPER_SNAKE` | `MAX_REINTENTOS = 3` |
| Clases | `PascalCase` | `MiClase`, `GestorUsuarios` |
| Módulos | `snake_case` | `gestor_archivos.py` |
| Privados (convención) | `_prefijo` | `_interno`, `__privado` (name mangling) |

---

## 3. Tipos de Datos Fundamentales

### 3.1 Números

```python
entero = 42
flotante = 3.14
complejo = 1 + 2j

# Operadores
7 + 3      # 10
7 - 3      # 4
7 * 3      # 21
7 / 3      # 2.333... (float)
7 // 3     # 2 (división entera)
7 % 3      # 1 (módulo)
7 ** 3     # 343 (potencia)

# Funciones built-in
abs(-5)      # 5
round(3.14159, 2)  # 3.14
pow(2, 10)   # 1024
divmod(10, 3) # (3, 1) -> (cociente, resto)
```

### 3.2 Strings (Inmutables)

```python
s = "hola"
s = 'hola'
s = """multilínea"""
s = f"Hola {nombre}"  # f-string (Python 3.6+)

# Métodos comunes
"hola".upper()        # "HOLA"
"HOLA".lower()        # "hola"
"  hola  ".strip()    # "hola"
"hola mundo".split()  # ["hola", "mundo"]
"a,b,c".split(",")    # ["a", "b", "c"]
"-".join(["a","b"])   # "a-b"
"hola".replace("h","H")  # "Hola"
"hola".startswith("h")   # True
"hola".endswith("a")     # True
"hola".find("l")         # 2 (índice o -1)
"hola".index("l")        # 2 (ValueError si no existe)
"hola".count("l")        # 1
```

### 3.3 Booleanos y None

```python
True, False
None  # Ausencia de valor

# Truthy / Falsy
# Falsy: False, None, 0, 0.0, "", [], (), {}, set()
# Truthy: Todo lo demás

if valor:       # True si truthy
if not valor:   # True si falsy
if valor is None:  # Identidad (no ==)
```

---

## 4. Estructuras de Datos Nativas

| Tipo | Mutable | Ordenado | Sintaxis | Uso típico |
|------|---------|----------|----------|------------|
| `list` | ✅ | ✅ | `[1, 2, 3]` | Secuencia modificable |
| `tuple` | ❌ | ✅ | `(1, 2, 3)` | Registro inmutable, clave dict |
| `set` | ✅ | ❌ | `{1, 2, 3}` | Unicidad, operaciones matemáticas |
| `dict` | ✅ | ✅ (3.7+) | `{"a": 1}` | Mapeo clave→valor |
| `frozenset` | ❌ | ❌ | `frozenset([1,2])` | Clave dict, elemento set |

```python
# List
l = [1, 2, 3]
l.append(4)       # [1,2,3,4]
l.extend([5,6])   # [1,2,3,4,5,6]
l.insert(0, 0)    # [0,1,2,3,4,5,6]
l.pop()           # 6 (elimina y devuelve último)
l.pop(0)          # 0 (elimina por índice)
l.remove(2)       # elimina primera ocurrencia
l.sort()          # ordena in-place
sorted(l)         # nueva lista ordenada
l.reverse()       # invierte in-place
l[::-1]           # copia invertida

# Dict
d = {"a": 1, "b": 2}
d["c"] = 3
d.get("x", 0)     # 0 (default si no existe)
d.setdefault("x", [])  # [] si no existe
d.keys()          # dict_keys(['a','b','c','x'])
d.values()        # dict_values([1,2,3,[]])
d.items()         # dict_items([('a',1),...])
{k: v*2 for k,v in d.items()}  # dict comprehension

# Set
s = {1, 2, 3}
s.add(4)
s.update([5,6])
s.discard(2)      # no error si no existe
s.remove(2)       # KeyError si no existe
s1 | s2           # unión
s1 & s2           # intersección
s1 - s2           # diferencia
s1 ^ s2           # diferencia simétrica
```

---

## 5. Control de Flujo

### 5.1 Condicionales

```python
if x > 0:
    print("positivo")
elif x < 0:
    print("negativo")
else:
    print("cero")

# Ternario
msg = "positivo" if x > 0 else "no positivo"

# Match-case (Python 3.10+)
match comando:
    case "iniciar":
        start()
    case "parar" | "detener":
        stop()
    case _:
        print("desconocido")
```

### 5.2 Bucles

```python
# for (iterable)
for i in range(5):          # 0,1,2,3,4
for i in range(2, 10, 2):   # 2,4,6,8
for i, v in enumerate(lista):  # índice + valor
for k, v in dic.items():    # clave + valor
for a, b in zip(lista1, lista2):  # paralelo

# while
while condicion:
    pass

# Control
break    # sale del bucle
continue # salta a siguiente iteración
else:    # ejecuta si NO hubo break
```

### 5.3 Comprensiones

```python
# List
[x**2 for x in range(10)]
[x**2 for x in range(10) if x % 2 == 0]

# Dict
{x: x**2 for x in range(5)}

# Set
{x**2 for x in range(5)}

# Generator (lazy)
(x**2 for x in range(10))
```

---

## 6. Funciones

### 6.1 Definición Básica

```python
def saludar(nombre: str, saludo: str = "Hola") -> str:
    """Docstring: describe qué hace, args y retorno."""
    return f"{saludo}, {nombre}!"

# Llamada
saludar("Juan")
saludar("Juan", "Buenos días")
saludar(saludo="Hi", nombre="Juan")  # keyword args
```

### 6.2 Parámetros Flexibles

```python
def ejemplo(a, b, *args, **kwargs):
    print(a, b)        # posicionales obligatorios
    print(args)        # tupla de posicionales extra
    print(kwargs)      # dict de keyword args extra

ejemplo(1, 2, 3, 4, x=10, y=20)
# a=1, b=2, args=(3,4), kwargs={'x':10, 'y':20}
```

### 6.3 Anotaciones de Tipo (Type Hints)

```python
from typing import List, Dict, Optional, Union

def procesar(items: List[str], config: Dict[str, int], 
             filtro: Optional[str] = None) -> Union[int, str]:
    ...

# Python 3.9+: list, dict, tuple, set genéricos
def f(items: list[str]) -> dict[str, int]: ...
```

### 6.4 Lambdas y Funciones de Orden Superior

```python
cuadrado = lambda x: x**2
pares = filter(lambda x: x%2==0, range(10))
cuadrados = map(lambda x: x**2, range(5))
reduce(lambda a,b: a+b, [1,2,3,4])  # 10 (functools.reduce)
```

### 6.5 Docstrings y `help()`

```python
def func(a: int) -> str:
    """Resumen breve.
    
    Args:
        a: Descripción del parámetro.
    Returns:
        Descripción del retorno.
    Raises:
        ValueError: Si a < 0.
    """
    if a < 0:
        raise ValueError("a debe ser >= 0")
    return str(a)
```

---

## 7. Manejo de Errores (Excepciones)

### 7.1 Jerarquía Básica

```
BaseException
 ├── KeyboardInterrupt
 ├── SystemExit
 └── Exception
      ├── ValueError
      ├── TypeError
      ├── KeyError / IndexError
      ├── FileNotFoundError
      ├── IOError / OSError
      ├── RuntimeError
      └── ...
```

### 7.2 Bloques try/except/else/finally

```python
try:
    resultado = 10 / x
except ZeroDivisionError:
    print("No se puede dividir por cero")
except (TypeError, ValueError) as e:
    print(f"Error de tipo/valor: {e}")
except Exception as e:
    print(f"Error inesperado: {e}")
else:
    print("Sin errores")      # solo si no hubo excepción
finally:
    print("Limpieza")         # siempre se ejecuta
```

### 7.3 Levantar Excepciones

```python
raise ValueError("x debe ser positivo")
raise CustomError("mensaje") from original_error  # encadenar
```

### 7.4 Excepciones Personalizadas

```python
class MiError(Exception):
    """Excepción base del módulo."""
    pass

class ErrorValidacion(MiError):
    def __init__(self, campo: str, mensaje: str):
        self.campo = campo
        super().__init__(f"{campo}: {mensaje}")
```

---

## 8. Entrada/Salida Básica

```python
# Consola
nombre = input("Nombre: ")
print(f"Hola {nombre}")
print("Error", file=sys.stderr)  # stderr

# Contraseña (oculta)
from getpass import getpass
pwd = getpass("Contraseña: ")

# Formateo (f-strings recomendadas)
print(f"Valor: {valor:.2f}")      # 2 decimales
print(f"{'texto':<10}")           # alinear izq 10 chars
print(f"{numero:>10}")            # alinear der
print(f"{numero:0>5}")            # padding zeros
print(f"{porcentaje:.1%}")        # porcentaje
```

---

## 9. Patrones Comunes y Buenas Prácticas

| Práctica | Por qué |
|----------|---------|
| `if __name__ == "__main__":` | Permite importar sin ejecutar |
| `with open(...) as f:` | Cierre automático (context manager) |
| `try/except` específico | Evita ocultar bugs |
| `raise ... from err` | Preserva traceback original |
| Type hints en APIs públicas | Documentación viva, IDE support |
| `if not lista:` en vez de `len(lista)==0` | Pythonic, O(1) |
| `enumerate()` en vez de `range(len())` | Más legible |
| `zip()` para iterar paralelo | Limpio y eficiente |

---

## 10. Referencias Cruzadas

- [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) - List, tuple, set, dict en profundidad
- [Programación Orientada a Objetos](./Python%20-%20Programaci%C3%B3n%20Orientada%20a%20Objetos.md) - Clases, herencia, polimorfismo
- [Manejo de Archivos](./Python%20-%20Manejo%20de%20Archivos%20y%20Serializaci%C3%B3n.md) - I/O, serialización
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - Threading, asyncio
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - pytest, linting, type checking

---

*Etiquetas: #python #fundamentos #sintaxis #tipos #control-flujo #funciones #excepciones #type-hints*