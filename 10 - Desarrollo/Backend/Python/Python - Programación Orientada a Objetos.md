---
aliases: ["POO Python", "Clases Python", "Python OOP"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Estructuras de Datos]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Proyectos Prácticos POO]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python - Testing, Calidad y Herramientas]]"
  - "[[Node JS]]"
fuente: "https://docs.python.org/es/3/tutorial/classes.html"
---

# Python - Programación Orientada a Objetos

Referencia profunda del sistema de objetos de Python: qué es realmente una clase (objeto tipo), el protocolo de métodos (descriptors), herencia con MRO C3, `super()` cooperativo, encapsulamiento, `dataclasses`, `enum`, `abc` y cuándo la POO **no** es la respuesta. Verificado con Python 3.14.

> [!info] El modelo mental correcto
> En Python **las clases también son objetos**: `class C: ...` crea un objeto de tipo `type` (metaclass por defecto), con su propio `__dict__`, heredable y pasable como argumento. Los "métodos" son funciones guardadas en la clase que, al accederse desde una instancia, se **enlazan** (se les pasa `self` automáticamente) gracias al **protocolo descriptor**. Todo lo demás —`@property`, `@classmethod`, `super()`— son consecuencias de ese protocolo.

## Clases e instancias: el mapa de memoria

```python
class Persona:                       # esto EJECUTA cuerpo y crea el objeto-tipo Persona
    especie = "humana"               # atributo de CLASE (compartido por todas)
    def __init__(self, nombre, edad):# constructor: se llama al instanciar
        self.nombre = nombre         # atributos de INSTANCIA (por objeto)
        self.edad = edad
    def saludo(self):                # método: función con acceso enlazado
        return f"Hola soy {self.nombre} y tengo {self.edad} años"

daniel = Persona("Daniel", 20)       # llama Persona.__init__(nueva_instancia, "Daniel", 20)
dylan = Persona("Dylan", 9)
print(daniel.saludo(), "|", dylan.saludo())
```

Verificado — la separación clase/instancia es **física**, no conceptual:

```python
class C:
    attr = "class"
c1, c2 = C(), C()
print(c1.attr, C.attr, c2.attr)   # class class class
c1.attr = "instance"              # crea entrada NUEVA en c1.__dict__
print(c1.attr, C.attr, c2.attr)   # instance class class
print("attr" in c1.__dict__)      # True  — el de la instancia vive en __dict__ de la instancia
print("attr" in C.__dict__)       # True  — el de clase sigue intacto
del c1.attr                       # borra la entrada de la instancia...
print(c1.attr)                    # ...y el lookup "cae" al de clase: class
```

| Dónde vive | Contenido | Lookup en `obj.x` |
|---|---|---|
| `obj.__dict__` | atributos de instancia | 1. Data descriptors de la clase |
| `type(obj).__dict__` (mro) | métodos, atributos de clase, properties | 2. `obj.__dict__` |
| `type(obj).__mro__` | cadenas de herencia | 3. atributos/métodos de clase |

> [!warning] Atributo de clase MUTABLE = estado compartido entre instancias
> Verificado: `class Servidor: clientes = []` — `s1.add("a"); s2.add("b")` deja `["a","b"]` **compartido** para todas las instancias. Listas/dicts/sets como default de atributo de clase son casi siempre un bug. Corrección: crear en `__init__` (`self.clientes = []`) o usar `__slots__`/property.

### `self`: solo convención, pero obligatorio

`self` es el nombre por convención del primer parámetro; Python lo llena automáticamente al llamar `obj.metodo(args)` → `Persona.metodo(obj, args)`. Podrías llamarlo `yo` y funcionaría, pero: **no lo hagas**. `self` es explícito a diferencia de `this` en otros lenguajes: puedes referirte a otras instancias (`self.otra.comparar(self)`), que es lo que hace el operador `==` estándar.

```python
p1 = Persona("A", 1); p2 = Persona("B", 2)
Persona.saludo(p1)      # válido: llamada no enlazada — útil en super() y tests
```

## Los tres tipos de método (visto por el protocolo descriptor)

| Decorador | Cómo se llama | Primer argumento | Para qué |
|---|---|---|---|
| `def m(self)` | `obj.m()` / `Clase.m(obj)` | instancia | comportamiento normal |
| `@classmethod` | `Clase.m(...)` (también vía instancia) | `cls` (la clase) | factories, alternativas a `__init__`, acceso a atributos de clase |
| `@staticmethod` | `Clase.m(...)` | ninguno | utilidad temáticamente ligada que no toca `self` ni `cls` |

Verificado — los tres son **descriptors** con semánticas distintas de binding:

```python
class D:
    def m(self): return "m"
    sm = staticmethod(lambda: "static")
    cm = classmethod(lambda cls: cls.__name__)
>>> type(D.__dict__["m"]), type(D.__dict__["sm"]), type(D.__dict__["cm"])
(<class 'function'>, <class 'staticmethod'>, <class 'classmethod'>)
>>> D().m(), D().sm(), D().cm()
('m', 'static', 'D')          # instance → bound method; static → función cruda; class → recibe cls
```

> [!tip] `classmethod` como *factory* polimórfico
> `cls(...)` respeta la **subclase**: si `Empleado.from_dict` usa `cls(...)`, una subclase `Gerente(Empleado)` que llame `Gerente.from_dict` obtiene una instancia `Gerente` sin reimplementar nada. `__new__` como alternativa cuando el objeto debe existir antes de `__init__` (p.ej. inmutables, singletons, cache).

```python
class Automovil:
    def __init__(self, marca, modelo): self.marca, self.modelo = marca, modelo
    @classmethod
    def deportivos(cls, marca): return cls(marca, "Deportivo")   # factory
    @classmethod
    def sedan(cls, marca): return cls(marca, "Sedan")
    def __str__(self): return f"{self.marca} {self.modelo}"

class Calculadora:                       # staticmethods: cohesión sin acoplamiento
    @staticmethod
    def suma(a, b): return a + b
    @staticmethod
    def dividir(a, b):
        if b == 0: raise ZeroDivisionError("divisor 0")
        return a / b
```

Cuándo `staticmethod` y no función de módulo: cuando la función es **parte de la interfaz pública de la clase** (p.ej. `Calculadora.dividir`) o necesita acceso a nombres privados del módulo. Fuera de eso, una función de módulo es más testeable.

## Properties y descriptores

### `@property`: la forma pythonica de getter/setter

```python
class Circunferencia:
    def __init__(self, radio): self._radio = radio
    @property
    def radio(self): return self._radio
    @property
    def diametro(self): return 2 * self._radio
    @property
    def area(self): return 3.1415 * self._radio ** 2
    @radio.setter
    def radio(self, valor):
        if valor < 0: raise ValueError("radio negativo")
        self._radio = valor

c = Circunferencia(5)
c.radio          # getter calculado sin llamada: c.radio, no c.radio()
c.radio = 10     # pasa por el setter con validación
c.diametro       # 20 — valor DERIVADO, siempre consistente
```

`@property` crea un **data descriptor** (tiene `__set__`): verificado que **gana** sobre `obj.__dict__` — `p.__dict__["x"] = 999` no pisa a una property `x`. Esa es la base de que los atributos "virtuales" no se puedan saltar.

| Cuándo property | Cuándo atributo plano |
|---|---|
| Validación en cada cambio | valor confiable establecido en `__init__` |
| Valor derivado/calculado (`area`, `len`) | simple dato interno (`_x`) |
| Mantener API tras refactor interno | — |
| Lazy con `functools.cached_property` (3.8+) | — |

> [!warning] No crees getters/setters Java por defecto
> Si `get_edad()`/`set_edad()` solo hacen `return self._edad`, borralos: el atributo público basta. Empezá plano y promové a property cuando aparezca una razón (validación, cálculo). Properties no son más lentas que atributos para lecturas frecuentes (son descriptors en C), pero sí cambian la API.

El descriptor completo (para leer el código de otros): un objeto con `__get__` (y opcionalmente `__set__`, `__delete__`) que la clase evalúa en el lookup. `property` es el caso de uso más común; protocolos avanzados viven en librerías como `attrs`/`pydantic`.

## Catálogo de dunders (métodos especiales)

Los dunders son el **protocolo** por el que los objetos integrados conversan con tu clase. Implementá los mínimos que sostengan el contrato.

| Dunders | Qué activan | Ejemplo de uso |
|---|---|---|
| `__init__`, `__new__`, `__del__` | creación/destrucción | `__new__` para inmutables/caching |
| `__repr__`, `__str__`, `__format__` | `repr()`, `print()`, `f"{x}"` | `__repr__` = debug fiel, `__str__` = legible |
| `__bytes__` | `bytes(x)` | serialización binaria |
| `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__` | `== != < <= > >=` | comparaciones |
| `__hash__` | clave de dict / set | lo anula `__eq__` si no lo definís |
| `__len__`, `__length_hint__` | `len(x)`, truthiness sin `__bool__` | colecciones |
| `__bool__` | `if x:` | precede a `__len__` |
| `__getitem__`, `__setitem__`, `__delitem__` | `x[k]`, `del x[k]` | secuencias/mappings |
| `__contains__` | `k in x` | si no, usa iteración O(n) |
| `__iter__`, `__next__`, `__reversed__` | `for`, `next()`, `reversed()` | iterables/iteradores |
| `__enter__`, `__exit__` | `with x:` | context managers |
| `__call__` | `x(...)` | objetos-función, decoradores con estado |
| `__add__`, `__sub__`, `__mul__`, `__truediv__`, `__floordiv__`, `__mod__`, `__pow__`, `__matmul__` | `+ - * / // % ** @` | aritmética de dominio |
| `__iadd__` etc. | `+=` | si no existe, cae a `__add__` + reasignación |
| `__neg__`, `__abs__`, `__int__`, `__float__`, `__index__` | conversión numérica | `__index__` para slicing |
| `__setattr__`, `__getattr__`, `__getattribute__` | asignación/acceso | proxies, validación (con cuidado: recursión) |
| `__slots__` | elimina `__dict__` | ahorro de memoria |

### Ejemplos canónicos (todos funcionales)

```python
# __str__/__repr__ — el par obligatorio de toda clase de dominio
class Rectangulo:
    def __init__(self, ancho, alto): self.ancho, self.alto = ancho, alto
    @property
    def area(self): return self.ancho * self.alto
    def __repr__(self): return f"Rectangulo({self.ancho}, {self.alto})"   # evaluable
    def __str__(self):  return f"[{self.ancho}x{self.alto} área={self.area}]"  # legible
    def __eq__(self, o): return isinstance(o, Rectangulo) and (self.ancho, self.alto) == (o.ancho, o.alto)

r = Rectangulo(20, 80)
print(r)             # usa __str__:  [20x80 área=1600]
print(f"{r!r}")      # fuerza __repr__: Rectangulo(20, 80)
Rectangulo(20, 80) == Rectangulo(20, 80)   # True vía __eq__

# __add__ — operador de dominio
class Punto:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, o): return Punto(self.x + o.x, self.y + o.y)
    def __repr__(self): return f"P({self.x}, {self.y})"
(Punto(2, 8) + Punto(4, 9))   # P(6, 17)

# __call__ — objeto invocable (decorador con estado, estrategia)
class Saludo:
    def __init__(self, saludo): self.saludo = saludo
    def __call__(self, nombre): return f"{self.saludo} {nombre}!"
Saludo("!Hola")("Luis")       # '!Hola Luis!'

# __len__ + __getitem__ — colección casera
class Caja:
    def __init__(self, *frutas): self.frutas = frutas
    def __len__(self): return len(self.frutas)
    def __getitem__(self, i): return self.frutas[i]
len(Caja("Manzana", "Kiwi"))        # 2 — además habilita truthiness
Caja("a","b","c")[1]                # 'b'

# __iter__/__next__ — iterador (contadores, lectores de stream)
class Contador:
    def __init__(self, limite): self.limite = limite
    def __iter__(self): self.contador = 0; return self
    def __next__(self):
        if self.contador < self.limite:
            self.contador += 1; return self.contador
        raise StopIteration
for i in Contador(3): print(i)      # 1 2 3
```

> [!tip] `__iter__` devuelve **iterador**; si tu objeto es una colección, delegá: `def __iter__(self): return iter(self._items)` — así soportás múltiples bucles simultáneos (el patrón "iterador que se a sí mismo" de Contador rompe eso).

### `__eq__` sin `__hash__` anula el hashing (verificado)

```python
class Q:
    def __init__(self, v): self.v = v
    def __eq__(self, o): return isinstance(o, Q) and self.v == o.v
{Q(1)}
# TypeError: cannot use 'Q' as a set element (unhashable type: 'Q')

class R:
    __slots__ = ("v",)
    def __init__(self, v): self.v = v
    def __eq__(self, o): return isinstance(o, R) and self.v == o.v
    def __hash__(self): return hash(self.v)     # ¡hash explícito lo restaura!
{R(1), R(1), R(2)}   # verificado: set de 2 elementos
```

Regla: definir `__eq__` pone `__hash__ = None` (mutabilidad implícita). Si la igualdad es por valor **y** el objeto es inmutable, definí `__hash__` junto a `__eq__` (usa los mismos campos). Dataclasses lo resuelven según `eq`/`frozen`.

### `__slots__`: memoria y rendimiento

```python
class SinSlots: pass
class ConSlots:
    __slots__ = ("a", "b")
# verificado con sys.getsizeof: instancia sin slots ~112 B (incl. __dict__)
# vs con slots ~40 B — 65% menos; y el lookup de atributos es un poco más rápido
```

`__slots__` elimina `__dict__` por instancia: menos memoria, atributos desconocidos lanzan `AttributeError` (fail-fast), lookup directo. Costos: no podés añadir atributos no listados (herencia complicada: cada clase necesita sus slots) y `weakref` necesita incluir `"__weakref__"`. Justificado con millones de instancias (nodos de grafo, parsers, eventos).

## Herencia y polimorfismo

```python
class Animal:                              # superclase
    def __init__(self, nombre): self.nombre = nombre
    def hablar(self):
        raise NotImplementedError("las subclases deben implementar hablar()")

class Gato(Animal):
    def hablar(self): return "!Miau!"
class Perro(Animal):
    def hablar(self): return "!Guau!"

def hacer_hablar(objeto):                  # polimorfismo: solo necesita el protocolo
    print(f"{objeto.nombre} dice {objeto.hablar()}")

hacer_hablar(Gato("Firulais")); hacer_hablar(Perro("Lulú"))
```

Notas de diseño:

- **`raise NotImplementedError`** en métodos "abstractos" manualmente: funciona y documenta, pero no **impide** instanciar la base. Para contratos duros, `abc.ABC` (más abajo).
- **Herencia vs composición**: heredá solo cuando la subclase *es un* tipo de la base **y** el contrato LSP se mantiene (todo que funcione con la base debe funcionar con la subclase). Si solo reutilizas código, delegá: `self.pagador = StripePagador()`.
- **Duck typing**: Python prefiere "¿tiene el método?" sobre "¿es de la clase?": `hacer_hablar` aceptaría cualquier objeto con `.nombre` y `.hablar()`. `Protocol` (typing) documenta eso sin herencia — ver [[Python - Patrones, Buenas Prácticas y Tipado]].

### `super()` y la MRO (C3 linearization)

`super()` **no** significa "la clase padre": significa *"siguiente clase en la MRO de la clase de `self`"*. Verificado en 3.14:

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
D.__mro__
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

C3 garantiza: 1) los padres se visitan en el orden declarado, 2) cada clase aparece **antes** que sus bases, 3) la linearización es **consistente** — si no puede ordenarse, Python lanza error en la definición. Verificado:

```python
class X(A): pass
class Y(X): pass
class Z(X): pass
class W(Y, Z): pass      # OK: (W, Y, Z, X, A, object)
# Pero una jerarquía "diamante invertido" inconsistente:
class F(D, E): pass      # TypeError: Cannot create a consistent method resolution order (MRO)
```

Super cooperativo verificado — el patrón de la herencia múltiple de mixins:

```python
class A2:
    def m(self): return "A2"
class B2(A2):
    def m(self): return "B2->" + super().m()
class C3(A2):
    def m(self): return "C3->" + super().m()
class D3(B2, C3):
    def m(self): return "D3->" + super().m()
>>> D3().m()
'D3->B2->C3->A2'        # super() recorre TODA la MRO, no solo el padre textual
```

| Aspecto | `super()` | `Clase.metodo(self)` |
|---|---|---|
| Alcance | siguiente en la MRO (cualquiera) | solo esa clase exacta |
| Herencia múltiple | ✅ cooperativo, correcto | ❌ salta clases |
| Sintaxis | `super()` (3+: sin args) | explícita |
| `__init__` | ✅ recomendado | solo si necesitás forzar |

```python
class Empleado(Persona):
    def __init__(self, nombre, edad, salario):
        super().__init__(nombre, edad)       # inicializa la parte "Persona"
        self.salario = salario
    def saludo(self):
        return f"{super().saludo()} y cobro {self.salario}$"
```

> [!warning] Errores clásicos de herencia
> 1. **No llamar a `super().__init__()`**: los atributos de la base quedan sin crear → `AttributeError` tardío. 2. **Herencia sin `super()` y con múltiples padres**: los otros `__init__` nunca corren. 3. **Usar `isinstance` en cadena de if/elif**: señal de que polimorfismo o registry resolvería mejor.

## Encapsulamiento: las tres convenciones

| Nivel | Sintaxis | Garantía real | Uso |
|---|---|---|---|
| Público | `x` | ninguna | interfaz estable de la clase |
| Protegido | `_x` | **ninguna** (solo convención) | "interno, no toques" |
| Privado | `__x` | **name mangling**: se guarda como `_Clase__x` | evita colisión de nombres en subclases |
| Restricción dura | `__slots__` + property | atributos no listados → `AttributeError` | cuando hay que impedir de verdad |

Verificado el mangling:

```python
class Ejemplo:
    def __init__(self):
        self._protegido = "ok"
        self.__privado = "secreto"        # se almacena como _Ejemplo__privado
Ejemplo().__privado                        # AttributeError
Ejemplo()._Ejemplo__privado                # "secreto" — siempre accesible si insistís
```

> [!warning] Python no tiene privacidad de datos
> El mangling existe para que **las subclases** no pisen atributos por accidente, no para esconder. Diseñá APIs donde lo interno sea `_x` y documentado como tal; usá `__x` solo contra herencia conflictiva. Para secretos reales: entornos o gestores de secretos, nunca clases.

Patrón de encapsulación correcto — invariantes vía métodos (CuentaBancaria):

```python
class CuentaBancaria:
    def __init__(self, num_cuenta, titular, saldo_inicial=0):
        self.num_cuenta, self.titular = num_cuenta, titular
        if saldo_inicial < 0: raise ValueError("saldo inicial negativo")
        self.__saldo = saldo_inicial                     # invariante: saldo >= 0
    def depositar(self, cantidad):
        if cantidad <= 0: raise ValueError("cantidad no positiva")
        self.__saldo += cantidad
    def retirar(self, cantidad):
        if not 0 < cantidad <= self.__saldo: raise ValueError("fondos insuficientes")
        self.__saldo -= cantidad
    @property
    def saldo(self): return self.__saldo                 # lectura sin setter directo
```

Toda modificación pasa por métodos que mantienen el invariante — el verdadero encapsulamiento (a diferencia de los ejercicios con `print` del tutorial original, versión limpia arriba absorbe `depositar`/`retirar`/`mostrar_dinero`).

## `@dataclass`: data objects sin ruido

```python
from dataclasses import dataclass, field, asdict, replace

@dataclass(order=True, frozen=False, slots=True)
class Point:
    x: float
    y: float = 0.0
    tags: tuple = ()
    def distancia(self): return (self.x**2 + self.y**2) ** 0.5
```

Genera `__init__`, `__repr__`, `__eq__` (y con `order=True`, comparaciones; con `frozen=True`, `__hash__`). Verificado:

| Config | Verificado |
|---|---|
| `@dataclass(order=True, frozen=True)` | `p1 == p2` → True; `p1 < Pt(2,0)` → True; `hash(p1)` funciona → set acepta |
| `frozen=True` + `field(default_factory=list)` | la **clase se crea bien** (a diferencia del default mutable directo que lanza `TypeError`); `f.nums = []` → `AttributeError` (no reasigna) pero `f.nums.append(1)` funciona — frozen no congela el **contenido** mutable |
| Herencia | `@dataclass class Der(Base)`: campos base primero en `__init__`; `__dataclass_fields__` → `dict_keys(['nombre', 'edad'])` |
| Defaults mutables | `lista: list = []` → `TypeError: mutable default ... use field(default_factory=...)` — protección incorporada |

Cuándo `dataclass` vs `NamedTuple` vs `class`:

| Necesidad | Elegir |
|---|---|
| Solo datos, inmutables, hashables, ligeros | `NamedTuple` |
| Datos mutables con comportamiento mínimo | `@dataclass` |
| Invariantes fuertes + métodos | `@dataclass(frozen=True)` + validación en `__post_init__` o `class` con `__init__` |
| Objetos con estado rico y polimorfismo | `class` normal |

`__post_init__` para validación post-construcción; `field(init=False)` para atributos calculados; `slots=True` (3.10+) para ahorro real de memoria.

## `enum`: tipos con dominio limitado

```python
from enum import Enum, auto, unique, IntEnum, Flag

class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3
Color.RED              # <Color.RED: 1>
Color.RED.name, Color.RED.value     # 'RED', 1
Color(2)               # busca por valor → Color.GREEN
Color["GREEN"]         # busca por nombre
list(Color)            # [Color.RED, Color.GREEN, Color.BLUE] — iterables
class Permiso(Flag):
    LECTURA = 4; ESCRITURA = 2; EJECUCION = 1
Permiso.LECTURA | Permiso.ESCRITURA   # combinación: Permiso.LECTURA|ESCRITURA
class Estado(IntEnum): ACTIVO = 1     # compara igual a 1 (útil para flags de C APIs)
```

Reglas: identidad por miembro (`Color.RED is Color.RED`), comparación entre miembros de distintos enums da `False` (o `TypeError` según operación), `auto()` asigna valores automáticamente, `@unique` prohíbe duplicados. Para str-enum (3.11+: `StrEnum`) cuando necesitás serializar limpio a JSON.

## `abc`: contratos duros de interfaz

```python
from abc import ABC, abstractmethod

class Dispositivo(ABC):
    def __init__(self, modelo): self.modelo = modelo
    @abstractmethod
    def scanner_vuln(self): ...            # sin cuerpo: obligatorio implementar

class Ordenador(Dispositivo):
    def scanner_vuln(self): return "actualización necesaria"
class Router(Dispositivo):
    def scanner_vuln(self): return "puertos críticos"

Dispositivo("x")
# TypeError: Can't instantiate abstract class Dispositivo without an implementation for 'scanner_vuln'
```

A diferencia del `raise NotImplementedError` manual, ABC **impide instanciar** la base (`TypeError` en la definición) y deja `__subclasshook__` para registro estructural (`isinstance` sin herencia). Para herencia múltiple de interfaces: mixins sin estado (`class JsonMixin: def to_json(self): ...`).

## Propiedades avanzadas: decoradores de verdad

El decorador es una función de orden superior: toma una función, devuelve otra (closure), y el `@deco` es azúcar para `f = deco(f)`.

```python
import time
from functools import wraps

def cronometro(func):                     # decorador genérico (sin args)
    @wraps(func)                          # preserva __name__/__doc__ (¡imprescindible!)
    def envoltura(*args, **kwargs):       # args/kwargs: acepta CUALQUIER firma
        inicio = time.perf_counter()
        resultado = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter() - inicio:.3f}s")
        return resultado                  # NO olvidar devolver el valor
    return envoltura
```

`functools.wraps` copia `__wrapped__`, `__name__`, `__doc__` — sin él, `help()` y pytest muestran "envoltura" y los decorators no se componen limpio. Decoradores **con argumentos** necesitan una capa más (`def repetir(n): def deco(f): ...  return deco`). Más patrones en [[Python - Patrones, Buenas Prácticas y Tipado]] y ejemplos completos en [[Python - Proyectos Prácticos POO]].

## Cuándo NO usar clases

| Señal | Mejor alternativa |
|---|---|
| Estructura de solo datos sin métodos | `dataclass`/`NamedTuple`/dict |
| Una sola función, cero estado | función de módulo |
| Estrategia con un caso | lambda/función |
| "Clase con un solo método" | función + `functools.partial` |
| Jerarquía de 4+ niveles por reutilizar código | composición/delegación |
| Config/dispatch | dict registry (ver Fundamentos, receta 6) |

La POO cuesta: verbosidad, acoplamiento de jerarquías, dificultad de test. Python es multiparadigma: funciones + inmutables + POO donde el dominio lo merezca (p.ej. un parser AST, modelos de dominio, plugin systems).

## Errores comunes

1. **Atributo de clase mutable** — estado compartido sorprendente (verificado arriba).
2. **Olvidar `self`** en la definición del método → `TypeError: takes 1 positional argument but 2 were given` al llamar.
3. **`self.x = x` en `__init__` sin validar** — invariantes después tardan en fallar; validá temprano.
4. **`__eq__` sin `__hash__`** → objetos no usables en sets/dicts (verificado).
5. **Mutable como default de dataclass** → `TypeError` (protección incluida); en funciones normales el bug es **silencioso**.
6. **Comparar con `type(x) == Y`** en vez de `isinstance` — rompe herencia; usa `isinstance` salvo que quieras exactitud total.
7. **`super()` con argumentos en Python 3** — `super(Cosa, self)` es estilo Py2; el `super()` vacío es igual de rápido desde 3.x.
8. **MRO imposible** → `TypeError: Cannot create a consistent MRO` en la **definición** de la clase; reordena bases o elimina la redundancia.
9. **`@property` sobre un setter que no existe** → `AttributeError: can't set attribute` al asignar; definir ambos o nada.
10. **Mezclar `__getattr__` y `__getattribute__`** sin entender: `__getattribute__` intercepta TODOS los accesos (fácil recursion infinita → debe delegar a `super()`); `__getattr__` solo cuando no se encontró.

## Cheatsheet — POO

| Elemento | Sintaxis | Nota |
|---|---|---|
| Clase | `class C(Base, metaclass=M):` | crea objeto `type` |
| Constructor | `def __init__(self, ...)` | validación temprana |
| Atributo de clase | `x = 1` en cuerpo | compartido — NO mutables |
| Property | `@property` + `@x.setter` | data descriptor, gana a `__dict__` |
| Class method | `@classmethod` / `cls` | factory polimórfica |
| Static method | `@staticmethod` | sin self/cls |
| Herencia | `class H(P)` | MRO C3; `__mro__` para inspeccionar |
| super | `super().metodo()` | siguiente en MRO, cooperativo |
| Privado | `__x` → `_C__x` | name mangling |
| Slots | `__slots__ = ("a",)` | sin `__dict__`, -memoria |
| Dataclass | `@dataclass(frozen=True, order=True, slots=True)` | datos en 3 líneas |
| Enum | `class E(Enum): A = 1` | `E["A"]`, `E(1)`, `Flag` para bits |
| ABC | `class I(ABC): @abstractmethod` | impide instanciar |
| Igualdad | `__eq__` + `__hash__` | hashable requiere ambos |
| Colección | `__len__`, `__getitem__`, `__iter__` | protocolo completo |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — semántica de objetos y names
- [[Python - Estructuras de Datos]] — NamedTuple, dataclass vs collections
- [[Python - Librerías Estándar Esenciales]] — `enum`, `abc`, `functools.cached_property`, `inspect`
- [[Python - Proyectos Prácticos POO]] — ejercicios integradores con estas ideas
- [[Python - Patrones, Buenas Prácticas y Tipado]] — Protocol, composition patterns, tipado
- [[Python - Testing, Calidad y Herramientas]] — testear clases, mocks, fixtures
- [[Node JS]] — comparación con el sistema de prototipos de JavaScript
