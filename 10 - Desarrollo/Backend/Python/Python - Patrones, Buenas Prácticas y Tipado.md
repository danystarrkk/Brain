---
tags:
  - desarrollo
  - backend
  - python
tipo: concepto
estado: terminado
---

# Python - Patrones, Buenas Prácticas y Tipado

Referencia profunda de cómo escribir Python **mantenible**: SOLID aplicado al lenguaje real (no a UML), patrones de diseño en su forma *pythonica*, el sistema de type hints completo hasta 3.14, principios KISS/DRY/YAGNI, cuándo un patrón es un **anti-patrón** en Python, olores de código y tácticas de refactoring. Verificado con Python 3.14.7.

> [!info] Python tiene filosofía propia
> "There should be one-- and preferably only one --obvious way to do it" (PEP 20). Los GoF escritos a la fuerza (clases por clase, fábricas de fábricas) producen código **más** difícil en Python, donde la primera clase, el dict-dispatch, la función de orden superior y los decoradores cubren el 95% de los casos. Este nota enseña **dónde** cada patrón sí paga y su forma idiomática.

## Principios SOLID en Python

### S — Single Responsibility (una clase, una razón para cambiar)

```python
# ❌ God class: persistencia + negocio + formato + SMTP en una
class UsuarioService:
    def guardar(self, u): open("u.json", "w").write(json.dumps(u.__dict__))
    def calcular_puntaje(self, u): ...
    def enviar_email(self, u): smtplib.SMTP(...).sendmail(...)

# ✅ cada unidad con un motivo de cambio distinto
class UsuarioRepositorio:      # cambia cuando cambia la persistencia
    def guardar(self, u: Usuario) -> None: ...
class PuntajeService:          # cambia cuando cambia la regla de negocio
    def calcular(self, u: Usuario) -> int: ...
class EmailSender:             # cambia cuando cambia el canal
    def enviar(self, destino: str, asunto: str) -> None: ...
```

Señal de violación: el constructor de una clase necesita 5+ dependencias, o sus métodos tocan subsystemas distintos. Ver "Arquitectura en capas" abajo para el patrón de composición.

### O — Open/Closed (abierto a extensión, cerrado a modificación)

```python
# La forma pythonica NO es herencia obligada: es "pasar comportamiento"
from functools import singledispatch

@singledispatch
def serializar(obj) -> str:
    raise TypeError(f"No sé serializar {type(obj)}")

@serializar.register
def _(obj: datetime) -> str:
    return obj.isoformat()

@serializar.register
def _(obj: Decimal) -> str:
    return str(obj)

# Nuevo tipo = nuevo registro; el dispatcher central no se toca
```

También: estrategias como parámetros (`def procesar(items, key=funcion)`), protocols (abajo) y registries por decorator. Herencia solo cuando hay una relación "es-un" real.

### L — Liskov (sustitución sin romper contratos)

```python
# ❌ violación clásica: el subtipo debilita el contrato
class Pato:
    def volar(self): return "vuela"
class PatoDeHule(Pato):
    def volar(self): raise NotImplementedError()  # quien usa Pato no esperaba esto

# ✅ la corrección es partir la interfaz (ISP) o usar composición
class Nadador:
    def nadar(self): ...
class Volador:
    def volar(self): ...
class Pato(Nadador, Volador): ...
```

En Python el contrato es **comportamiento + pre/post-condiciones + tipos**; un subtipo que lanza excepciones nuevas donde el base no las lanza, o que acepta menos argumentos, viola LSP. Los type checkers ayudan: `mypy` detecta incompatibilidades de firma en overrides.

### I — Interface Segregation (interfaces mínimas)

```python
from typing import Protocol

# ❌ interfaz gorda: fuerza implementar métodos irrelevantes
class Trabajador(Protocol):
    def trabajar(self): ...
    def cobrar(self): ...
    def fichar(self): ...

# ✅ interfaces por rol, que el código REAL necesita
class EntradaSalida(Protocol):
    def leer_linea(self) -> str | None: ...
    def escribir_linea(self, texto: str) -> None: ...

class Consola:
    def leer_linea(self): return input()
    def escribir_linea(self, texto): print(texto)

def repl(io: EntradaSalida) -> None:   # no le importa si es consola, socket o test double
    while (linea := io.leer_linea()) is not None:
        io.escribir_linea(linea.upper())
```

### D — Dependency Inversion (depender de abstracciones)

```python
from typing import Protocol
from dataclasses import dataclass

class RepositorioUsuarios(Protocol):        # el PUERTO (abstracción)
    def guardar(self, usuario: "Usuario") -> None: ...
    def obtener(self, id: int) -> "Usuario | None": ...

class RepositorioSQL:                        # el ADAPTADOR (detalle)
    def __init__(self, conexion): self.conexion = conexion
    def guardar(self, usuario):
        self.conexion.execute(
            "INSERT INTO usuarios (id, nombre) VALUES (?, ?)",
            (usuario.id, usuario.nombre))
    def obtener(self, id):
        fila = self.conexion.execute("SELECT id, nombre FROM usuarios WHERE id=?", (id,)).fetchone()
        return Usuario(*fila) if fila else None

@dataclass
class CrearUsuario:                          # el CASO DE USO depende del Protocol
    repositorio: RepositorioUsuarios
    def ejecutar(self, nombre: str) -> "Usuario":
        usuario = Usuario(id=uuid4().int % 10**6, nombre=nombre)
        self.repositorio.guardar(usuario)
        return usuario

# Composición en el punto de entrada (main.py):
def main():
    repo = RepositorioSQL(conectar())
    caso = CrearUsuario(repositorio=repo)    # inyección: testear con un fake trivial
    print(caso.ejecutar("Juan"))
```

> [!tip] En Python la "interfaz" es un `Protocol`, no una clase abstracta
> `ABC` obliga a **heredar**; `Protocol` (typing) define estructura **sin herencia**: cualquier clase con los métodos correctos cumple el contrato (duck typing + verificación estática). Es la forma moderna de I y D a la vez. `runtime_checkable` añade `isinstance` con comprobación superficial de métodos.

## Patrones en su forma pythonica

### Creacionales

**Factory** — en Python muchas veces es solo un dict:

```python
from abc import ABC, abstractmethod

class Notificador(ABC):
    @abstractmethod
    def enviar(self, mensaje: str) -> None: ...

class EmailNotificador(Notificador):
    def enviar(self, mensaje): print(f"Email: {mensaje}")

class SMSNotificador(Notificador):
    def enviar(self, mensaje): print(f"SMS: {mensaje}")

class PushNotificador(Notificador):
    def enviar(self, mensaje): print(f"Push: {mensaje}")

_REGISTRO: dict[str, type[Notificador]] = {
    "email": EmailNotificador, "sms": SMSNotificador, "push": PushNotificador,
}

def crear_notificador(tipo: str) -> Notificador:
    try:
        return _REGISTRO[tipo]()
    except KeyError:
        raise ValueError(f"Tipo desconocido: {tipo!r}; opciones: {sorted(_REGISTRO)}") from None
```

El registro por dict escala mejor que una cadena de `if/elif` o una jerarquía de factory-classes; nuevos tipos = una línea.

**Builder** — en Python, casi siempre innecesario gracias a dataclasses con defaults; sí paga con objetos caros o con muchos kwargs opcionales:

```python
from dataclasses import dataclass, field

@dataclass
class Usuario:
    nombre: str
    email: str
    telefono: str | None = None
    direccion: str | None = None
    roles: list[str] = field(default_factory=lambda: ["user"])

    def __post_init__(self):           # validación en construcción
        if "@" not in self.email:
            raise ValueError(f"Email inválido: {self.email}")

class UsuarioBuilder:                  # flujo encadenado con validación al final
    def __init__(self):
        self._campos: dict = {"nombre": "", "email": ""}
    def con_nombre(self, nombre: str) -> "UsuarioBuilder":
        self._campos["nombre"] = nombre; return self
    def con_email(self, email: str) -> "UsuarioBuilder":
        self._campos["email"] = email; return self
    def con_telefono(self, t: str) -> "UsuarioBuilder":
        self._campos["telefono"] = t; return self
    def build(self) -> Usuario:
        if not self._campos["nombre"] or not self._campos["email"]:
            raise ValueError("Nombre y email son requeridos")
        return Usuario(**self._campos)

usuario = UsuarioBuilder().con_nombre("Juan").con_email("juan@e.com").con_telefono("555").build()
```

Cuándo el builder es overkill: si el dataclass con defaults cubre el caso, usarlo (`Usuario(nombre="J", email="j@e.com")` directo). El builder clásico aparece cuando la construcción tiene **etapas** (montar → validar → congelar).

**Singleton** — anti-pattern en la mayoría de los casos; si de verdad hace falta (config de proceso):

```python
# Forma pythonica mínima: caché a nivel de módulo (el módulo YA es un singleton)
class Config: ...
CONFIG = Config()                      # importable desde cualquier sitio

# Con lru_cache para instancias perezosas con args:
from functools import lru_cache
@lru_cache(maxsize=None)
def get_config(ruta: str) -> "Config":
    return Config(ruta)

# La forma "GoF" (metaclass) — la conocé porque aparece en código heredado:
from threading import Lock
class SingletonMeta(type):
    _instances: dict = {}
    _lock = Lock()
    def __call__(cls, *args, **kwargs):
        with cls._lock:                                  # thread-safe (el original no lo era)
            if cls not in cls._instances:
                cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

El metaclass- Singleton es el que se ve en tutoriales; notá que el `lru_cache`/módulo logra lo mismo en 2 líneas y sin herencia mágica.

### Estructurales

**Adapter** — traducir una interfaz ajena a la que tu dominio espera:

```python
import xml.etree.ElementTree as ET

class ProveedorExterno:                     # API ajena (no la controlás)
    def obtener_datos_xml(self) -> str:
        return "<datos><valor>42</valor></datos>"

class AdaptadorJSON:                        # la envolvemos en lo que el dominio quiere
    def __init__(self, proveedor: ProveedorExterno):
        self._proveedor = proveedor
    def obtener_datos(self) -> dict:
        raiz = ET.fromstring(self._proveedor.obtener_datos_xml())
        return {hijo.tag: hijo.text for hijo in raiz}

print(AdaptadorJSON(ProveedorExterno()).obtener_datos())   # {'valor': '42'}
```

**Decorator (patrón, no el `@`)** — composición de comportamiento envolviendo objetos:

```python
class NotificadorBase(Notificador):
    def enviar(self, mensaje: str) -> None:
        print(f"Enviando: {mensaje}")

class ConLog(Notificador):
    def __init__(self, inner: Notificador): self._inner = inner
    def enviar(self, mensaje: str) -> None:
        print(f"[LOG] {mensaje}")
        self._inner.enviar(mensaje)

class ConReintento(Notificador):
    def __init__(self, inner: Notificador, reintentos: int = 3):
        self._inner, self._reintentos = inner, reintentos
    def enviar(self, mensaje: str) -> None:
        for intento in range(self._reintentos):
            try:
                self._inner.enviar(mensaje)
                return
            except Exception:
                if intento == self._reintentos - 1:
                    raise

# Apilado declarativo: el poder de la composición
notificador = ConReintento(ConLog(NotificadorBase()), reintentos=3)
notificador.enviar("Hola")
```

> [!tip] Decorator de objeto vs decorador de función
> El patrón *Decorator* de GoF se apila en **objetos**; en Python el 80% de ese valor lo da el **decorador de funciones** (`@wraps`, métricas, retry, logging) — ver la sección de decoradores en [[Python - Fundamentos y Sintaxis]]. Para cross-cutting a nivel de función, el decorador gana; para variantes de un objeto configurables en runtime, la composición gana.

### Comportamentales

**Strategy** — en Python, una **función** es la estrategia:

```python
from collections.abc import Callable

def ordenar_por_precio(items): return sorted(items, key=lambda i: i.precio)
def ordenar_por_nombre(items): return sorted(items, key=lambda i: i.nombre.lower())

def publicar(items, key: Callable = ordenar_por_nombre):   # la strategy como parámetro
    return key(items)

# Con enum para estrategias registradas:
from enum import Enum
class Criterio(Enum):
    PRECIO = ordenar_por_precio
    NOMBRE = ordenar_por_nombre
```

La clase-Contexto con `self._strategy` de los libros GoF existe, pero el parámetro de función es más corto, más testeable y compone mejor (ver también `functools.partial`).

**Observer** — señales de la stdlib o callbacks:

```python
# Forma manual (control total)
from typing import Callable
Evento = str

class Sujeto:
    def __init__(self):
        self._observadores: dict[Evento, list[Callable]] = {}
    def suscribir(self, evento: Evento, fn: Callable) -> None:
        self._observadores.setdefault(evento, []).append(fn)
    def emitir(self, evento: Evento, **datos) -> None:
        for fn in self._observadores.get(evento, []):
            fn(**datos)

sujeto = Sujeto()
sujeto.suscribir("usuario.creado", lambda **d: print("log:", d))
sujeto.suscribir("usuario.creado", lambda **d: email_enviar(d["email"]))
sujeto.emitir("usuario.creado", email="a@b.com")   # notifica a todos
```

La stdlib ya trae uno: `contextlib` + `atexit`, y `signal` para señales POSIX; en la práctica los eventos de dominio se resuelven con este pub/sub de 15 líneas, no con una librería.

**Null Object** — elimina `if x is None` repartido:

```python
class UsuarioNulo:
    nombre = "Anónimo"
    email = ""
    def saludar(self) -> str: return "Usuario no identificado"

def obtener_usuario(id: int, bd: dict) -> "Usuario | UsuarioNulo":
    return bd.get(id, UsuarioNulo())      # nunca None: los callsites no chequean

print(obtener_usuario(99, {}).saludar())  # sin AttributeError
```

**Context manager** — el patrón más pythonico de todos (ver `with` en [[Python - Fundamentos y Sintaxis]]):

```python
from contextlib import contextmanager
import time, logging

@contextmanager
def temporizador(nombre: str):
    inicio = time.perf_counter()
    try:
        yield
    finally:
        logging.info("%s tomó %.3fs", nombre, time.perf_counter() - inicio)

with temporizador("reporte"):
    generar_reporte()
```

## Type hints en profundidad

### Semántica de las anotaciones

Las anotaciones son **metadata**: en runtime Python las evalúa y las guarda en `__annotations__` pero **no valida nada** (la validación es de herramientas: mypy, pyright, pyre). Excepción: `dataclass` sí las lee para generar `__init__`. Anotar no ralentiza el código en ejecución; escribir `x: int = "hola"` funciona perfectamente hasta que mypy lo vea.

```python
def suma(a: int, b: int) -> int:
    return a + b

print(suma.__annotations__)   # {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
print(suma("x", "y"))         # runtime no valida: TypeError solo si el + falla
```

Desde 3.14 existe `annotationlib` (PEP 749) y el modo `deferred annotations` (`from __future__ import annotations` sigue siendo la palanca portable para no evaluar anotaciones).

### Opcionales y uniones

```python
# Tres formas de decir "str o None" — prefiero la última (PEP 604, 3.10+):
def f1(x: Optional[str]) -> None: ...     # typing.Optional
def f2(x: Union[str, None]) -> None: ...  # typing.Union
def f3(x: str | None) -> None: ...        # ✅ moderno, legible, y en runtime es real:
print(str | None)   # <class 'types.UnionType'> — uniones son objetos (3.10+)
```

`Optional[X]` **no** significa "kwarg opcional": significa "puede ser X o None". Los kwargs opcionales son defaults: `def g(x: int = 0)`.

### Generics: `TypeVar`, `Generic` y la sintaxis 3.12+

```python
from typing import TypeVar, Generic, Sequence
T = TypeVar("T")                      # forma clásica (todavía válida y extendida)

class Repositorio(Generic[T]):        # forma 3.5+...3.11
    def __init__(self) -> None:
        self._items: list[T] = []
    def agregar(self, item: T) -> None: self._items.append(item)
    def todos(self) -> Sequence[T]: return tuple(self._items)

# Sintaxis PEP 695 (3.12+, verificado en 3.14):
class Repo[T]:                                    # type params en la definición
    def __init__(self) -> None: self.items: list[T] = []
    def add(self, item: T) -> None: self.items.append(item)

def primera_mayuscula[T](xs: Sequence[T]) -> T:   # genéricos en funciones
    return xs[0]

r: Repo[int] = Repo(); r.add(1)                   # y type aliases:
type Punto = tuple[float, float]                  # verificado: funciona en 3.14
p: Punto = (1.5, 2.5)
```

| Sintaxis | Desde | Nota |
|---|---|---|
| `T = TypeVar("T")` + `Generic[T]` | 3.5 | Necesaria para `bound=`, `contravariant=True` |
| `def f[T](...)` / `class C[T]` | **3.12** | Preferida en código nuevo; menos boilerplate |
| `type X = ...` (alias genérico `type X[T] = list[T]`) | **3.12** | Reemplaza a las asignaciones TypeAlias |

`ParamSpec` (3.10+) captura la **firma completa** de una función — indispensable para decoradores que preservan tipos:

```python
from typing import TypeVar, ParamSpec, Callable
P = ParamSpec("P"); R = TypeVar("R")

def reintentar(fn: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        for i in range(3):
            try: return fn(*args, **kwargs)
            except OSError:
                if i == 2: raise
    return wrapper
```

Sin `ParamSpec`, decorar con `*args: Any` destruye el tipado de todos los callsites.

### `TypedDict`, `Literal`, `overload`

```python
from typing import TypedDict, NotRequired, Literal, overload

class UsuarioDict(TypedDict):           # forma de un dict para el checker
    nombre: str
    email: str
    edad: NotRequired[int]               # campo opcional EXPLÍCITO (3.11+)
    rol: Literal["admin", "user", "guest"]

def crear_usuario(data: UsuarioDict) -> None: ...
crear_usuario({"nombre": "J", "email": "j@e.com", "rol": "admin"})   # ok
# crear_usuario({"nombre": "J", "rol": "super"})   # mypy: rol inválido + falta email

@overload
def procesar(dato: int) -> str: ...
@overload
def procesar(dato: str) -> int: ...
def procesar(dato: int | str) -> str | int:       # la implementación real
    return str(dato) if isinstance(dato, int) else len(dato)

print(procesar(5), procesar("abc"))   # "5" 3 — el checker resuelve la firma correcta
```

`TypedDict` describe dicts de datos (JSON, configs); `NamedTuple`/`dataclass` describen registros con atributos. Para APIs externas, `TypedDict(total=False)` modela respuestas parciales.

### `Protocol`, `cast`, `assert` y narrowing

```python
from typing import Protocol, cast, Literal

class Serializador(Protocol):
    def serializar(self, obj: object) -> str: ...
    def deserializar(self, data: str) -> object: ...

class JSONSerializador:                    # NO hereda del Protocol — lo cumple por estructura
    def serializar(self, obj): import json; return json.dumps(vars(obj))
    def deserializar(self, data): import json; return json.loads(data)

def guardar(obj: object, s: Serializador) -> str: return s.serializar(obj)
guardar(object(), JSONSerializador())      # ok estructural

# cast: le mentís al checker CON una promesa explícita (no valida nada en runtime)
x = cast(list[int], algo_desconocido)

# assert: narrowing + validación en runtime (lo mejor de ambos en código normal)
def doble(n: int | None) -> int:
    assert n is not None, "n requerido"     # mypy afina n a int después de la línea
    return n * 2
```

| Herramienta | Valida en runtime | Afina tipos | Uso |
|---|---|---|---|
| `isinstance` | ✅ | ✅ (narrowing) | Preferida: chequeo real |
| `assert` | ✅ (se puede quitar con `-O`) | ✅ | Invariantes internos |
| `cast(T, x)` | ❌ | ✅ (a ciegas) | Fronteras con APIs no tipadas |
| `# type: ignore` | ❌ | silencia | Último recurso, siempre con razón |

### Versiones: typing en 3.12–3.14 (verificado en 3.14.7)

- `def f[T]` / `class C[T]` / `type A = ...` (PEP 695) — verificado operativo.
- `typing.override` (3.12): marca overrides para que mypy flaggee firmas incompatibles — verificado:

```python
from typing import override

class A:
    def m(self) -> int: return 1

class B(A):
    @override
    def m(self) -> int: return 2
```
- `typing.Never` (3.11) y `LiteralString` (3.11) — verificado.
- `dataclass(frozen=True, slots=True)`: inmutables y con `__slots__` (menos memoria, atributos fijos); intentar `v.x = 4` lanza `FrozenInstanceError` (verificado). `frozen=True` + `eq=True` hacen falta si se usa como dict key (hashable).
- `ReadOnly` (3.13) para TypedDict; `TypeAliasType` programático (3.12+); `annotationlib` y soporte de anotaciones diferidas (3.14).

### Configurar el checker: mypy estricto vs pyright

```toml
# pyproject.toml — mypy (ver [[Python - Testing, Calidad y Herramientas]] para el pipeline)
[tool.mypy]
python_version = "3.13"
strict = true                      # el atajo: untyped-defs, disallow-any-generics, etc.
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

| Checker | Velocidad | Precisión | Nota |
|---|---|---|---|
| **mypy** | Media | Muy buena; el estándar de facto | Ecosistema de plugins (Django, pydantic) |
| **pyright** | Rápida (análisis en TS) | Muy buena; mejor con PEP 695 | Default en pylance/VSC; `basedpyright` añade stricter |
| pyre (Meta) | Rápida | Buena | Incremental; ecosistema chico |

Recomendación: `strict` en mypy **o** pyright desde el día 1 de un proyecto nuevo; en legado, subir `check_untyped_defs` → `disallow_untyped_defs` por módulo con overrides.

## KISS, DRY, YAGNI en la práctica

| Principio | Señal de violación | Remedio concreto |
|---|---|---|
| **KISS** | Callbacks de callbacks; herencia de 4 niveles | La función de 10 líneas que todos entienden; "comprende el lenguaje" (PEP 20) |
| **DRY** | El mismo cálculo de fechas en 5 archivos | Extraer función/módulo; DRY aplica al **conocimiento**, no a dos `for` parecidos |
| **YAGNI** | `AbstractFactoryFactory`, parámetros "por si acaso" | Implementar la necesidad de HOY; refactorizar cuando aparezca la segunda |

> [!warning] DRY mal entendido es el peor de los tres
> Pre-servir dos usos que "podrían parecerse" acopla módulos que evolucionarán distinto. Regla práctica: **esperá a la duplicación real** (la tercera vez) para extraer; antes, duplicar dos líneas es más barato que la abstracción equivocada.

## Patrones que son anti-patrones en Python

| Patrón GoF | Problema en Python | Alternativa idiomática |
|---|---|---|
| Singleton con metaclass | Estado global oculto, acoplamiento, tests difíciles | Módulo + instancia; `lru_cache`; inyección explícita |
| Factory de fábricas | Burocracia para un dict | `_REGISTRO[klass]()` |
| Visitor | Rompe con duck typing; muchas clases frágiles | `isinstance` múltiple, singledispatch, o métodos |
| Strategy como jerarquía de clases | 3 clases para lo que hace una función | Parámetro callable |
| Java-style getters/setters | `u.get_nombre()` — no es Python | Atributo directo; `@property` solo si hay lógica |
| Herencia profunda por reutilización | Acoplamiento frágil (LSP) | Composición; mixins finos |

## Olores de código (tabla de diagnóstico)

| Olor | Síntoma | Refactor |
|---|---|---|
| God class | >500 líneas, >7 dependencias en `__init__` | Extraer servicios (SRP) |
| Duplicated logic | Mismo patrón copiado 3+ veces | Extraer función pura |
| Boolean flag params | `def f(a, b=False, c=False, d=True)` | Split en funciones o dataclass de opciones |
| Deep nesting | `if/for/try` 4+ niveles | Guard clauses (`if not x: return`), extraer |
| Mutable default | `def f(xs=[])` — bug de estado compartido | `xs: list | None = None` + `xs = xs or []` (ver [[Python - Fundamentos y Sintaxis]]) |
| Bare `except:` | Traga `KeyboardInterrupt` y bugs | `except ValueError as e:` específico |
| `except: pass` silencioso | Errores invisibles | Loguear o propagar |
| Long parameter list | `f(a, b, c, d, e, f)` posicional | Dataclass/kwargs/NamedTuple de config |
| Implicit global mutable | Config escrita en runtime | Inyección de dependencias |
| Premature optimization | Micro-optimizar sin medir | `cProfile`/`scalene` primero |

## Tácticas de refactoring (seguras, incrementalmente)

1. **Caracterizar primero**: tests verdes antes de tocar (ver [[Python - Testing, Calidad y Herramientas]]). Sin red de seguridad, no refactor.
2. **Extract function** (el 80%): seleccionar bloque → función con nombre de intención → parámetros explícitos → pasar a `ruff`/`mypy`.
3. **Replace conditional with polymorphism** solo cuando el `if/elif` crece con tipos **y** se extiende por fuera (registro/singledispatch).
4. **Introduce parameter object**: 4+ params correlacionados → dataclass de config.
5. **Seam-first**: identificar el límite I/O (red, disco, reloj) y extraerlo tras un Protocol — los tests dejan de necesitar mocks del mundo.
6. Medir antes/después con `ruff check` + `mypy` + suite verde; commits pequeños (ver disciplina de trabajo por unidades).

## Arquitectura en capas (clean architecture a la pythonica)

```
mi_proyecto/
├── src/mi_proyecto/
│   ├── domain/           # Entidades (dataclass), reglas, excepciones — CERO imports externos
│   │   ├── entities.py
│   │   └── exceptions.py
│   ├── application/      # Casos de uso; depende de Protocols (ports)
│   │   ├── services.py
│   │   └── ports.py
│   ├── infrastructure/   # Adaptadores: SQL, HTTP, archivos — implementan los Protocols
│   │   ├── repositories.py
│   │   └── database.py
│   └── cli.py o api.py   # Composición: armar el grafo e inyectar
├── tests/
├── pyproject.toml
```

La flecha de dependencia apunta hacia adentro: `domain` no importa nada de `infrastructure`; `application` importa solo `Protocol`s; la composición ocurre en el entry point. Esto es el DIP hecho módulos — y hace que cambiar SQLite por Postgres toque **un archivo**.

### Logging estructurado (cross-cutting sin acoplar)

```python
import json, logging
from datetime import datetime, timezone

class JSONFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module, "func": record.funcName, "line": record.lineno,
        }
        if record.exc_info:
            log["exception"] = self.formatException(record.exc_info)
        return json.dumps(log)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logging.getLogger().handlers = [handler]
logger = logging.getLogger(__name__)
logger.info("Usuario creado", extra={"user_id": 123, "accion": "registro"})
```

## Pipeline de calidad (resumen; detalle en la nota de Testing)

```bash
ruff check --fix . && ruff format .      # lint + formato (ver [[Python - Testing, Calidad y Herramientas]])
mypy --strict .                          # tipos
pytest --cov=mi_proyecto --cov-fail-under=80
bandit -r .                              # seguridad estática
```

## Errores comunes

1. **Aplicar GoF al pie de la letra** → jerarquías artificiales; Python premia funciones y composición.
2. **`Optional` confundido con kwarg opcional** → son cosas distintas (ver arriba).
3. **`cast` como`assert`** → `cast` no valida nada en runtime; no lo uses para "arreglar" errores de tipos, usá `isinstance`.
4. **Ignorar `__post_init__`/validación** → dataclasses sin validación aceptan basura; validar al construir.
5. **Tipar solo funciones sueltas, nunca las fronteras** → el valor está en APIs públicas y límites de datos (TypedDict/pydantic).
6. **DRY prematuro** → abstracciones que nadie pidió; la tercera copia es la señal.
7. **`# type: ignore` sin explicación** → convención: `# type: ignore[assignment]  # pydantic v1 API`.
8. **Subclases por reutilización de código** → composición; la herencia es para "es-un" con contrato compatible.

## Cheatsheet

| Tema | API | Una línea |
|---|---|---|
| Contrato estructural | `Protocol` + métodos `...` | Duck typing verificable |
| Genéricos nuevos | `class C[T]:` / `def f[T](...)` (3.12+) | Sin TypeVar boilerplate |
| Alias | `type X = list[int]` (3.12+) | Legible y diferido |
| Opcional | `x: str | None` | Preferido a Optional |
| Dict tipado | `TypedDict` + `NotRequired` | Para JSON/configs |
| Variantes | `Literal["a", "b"]` | Strings fijos |
| Firmas múltiples | `@overload` + impl real | Para APIs polimórficas |
| Decoradores tipados | `ParamSpec` + `TypeVar` | Preserva firma |
| Narrowing runtime | `isinstance` / `assert` | Mejor que `cast` |
| Mentira controlada | `cast(T, x)` | Solo en fronteras no tipadas |
| Override seguro | `@override` (3.12+) | Flaggea firmas rotas |
| Inmutable | `@dataclass(frozen=True, slots=True)` | `FrozenInstanceError` al mutar |
| Fábrica | dict `_REGISTRO[k]()` | Sin clases extra |
| Estrategia | parámetro `Callable` | Sin jerarquías |
| Contexto | `@contextmanager` | El patrón más pythonico |
| DI | Protocol + constructor | Testeable sin frameworks |
| Caché/módulo | `CONFIG = Config()` | El módulo es el singleton |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — decoradores, `with`, mutable defaults, anotaciones básicas
- [[Python - Programación Orientada a Objetos]] — clases, dataclasses, `__dunder__`, herencia vs composición
- [[Python - Estructuras de Datos]] — qué colección elige cada patrón (dict, deque, Counter...)
- [[Python - Proyectos Prácticos POO]] — los principios aplicados a proyectos completos
- [[Python - Testing, Calidad y Herramientas]] — mypy/ruff/pytest: la red de seguridad del refactoring
- [[Python - Módulos, Paquetes y Distribución]] — estructura de paquetes y publicación
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — inyectar pools y coordinar sin acoplar
