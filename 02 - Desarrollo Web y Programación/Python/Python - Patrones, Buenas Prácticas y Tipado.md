---
title: "Python - Patrones, Buenas Prácticas y Tipado"
tags:
  - lang/python
  - concept/patterns
  - concept/best-practices
  - concept/typing
  - concept/architecture
  - level/avanzado
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Patrones, Buenas Prácticas y Tipado

> **Objetivo**: Código limpio, mantenible, tipado y arquitécticamente sólido.

---

## 1. Principios SOLID en Python

| Principio | Descripción | Ejemplo Python |
|-----------|-------------|----------------|
| **S** Single Responsibility | Una clase, una razón para cambiar | `class UserService:` solo gestiona usuarios |
| **O** Open/Closed | Abierto a extensión, cerrado a modificación | `@singledispatch`, herencia, protocolos |
| **L** Liskov Substitution | Subtipos sustituibles por tipos base | `@abstractmethod`, `Protocol` |
| **I** Interface Segregation | Interfaces específicas > genéricas | `Protocol` con métodos mínimos |
| **D** Dependency Inversion | Depender de abstracciones, no concreciones | Inyección de dependencias, `Protocol` |

---

## 2. Patrones de Diseño en Python

### 2.1 Creacionales

#### Singleton
```python
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Config(metaclass=SingletonMeta):
    pass

# O con lru_cache (más pythonico)
from functools import lru_cache

@lru_cache(maxsize=None)
def get_config():
    return Config()
```

#### Factory
```python
from abc import ABC, abstractmethod

class Notificador(ABC):
    @abstractmethod
    def enviar(self, mensaje: str): ...

class EmailNotificador(Notificador):
    def enviar(self, mensaje): print(f"Email: {mensaje}")

class SMSNotificador(Notificador):
    def enviar(self, mensaje): print(f"SMS: {mensaje}")

class NotificadorFactory:
    @staticmethod
    def crear(tipo: str) -> Notificador:
        return {"email": EmailNotificador, "sms": SMSNotificador}[tipo]()

# Uso
notif = NotificadorFactory.crear("email")
notif.enviar("Hola")
```

#### Builder
```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Usuario:
    nombre: str
    email: str
    telefono: Optional[str] = None
    direccion: Optional[str] = None

class UsuarioBuilder:
    def __init__(self):
        self._usuario = Usuario("", "")
    
    def con_nombre(self, nombre: str):
        self._usuario.nombre = nombre
        return self
    
    def con_email(self, email: str):
        self._usuario.email = email
        return self
    
    def build(self) -> Usuario:
        if not self._usuario.nombre or not self._usuario.email:
            raise ValueError("Nombre y email requeridos")
        return self._usuario

usuario = UsuarioBuilder().con_nombre("Juan").con_email("juan@ejemplo.com").build()
```

### 2.2 Estructurales

#### Adapter
```python
class ProveedorExterno:
    def obtener_datos_xml(self) -> str:
        return "<datos><valor>42</valor></datos>"

class AdaptadorJSON:
    def __init__(self, proveedor: ProveedorExterno):
        self.proveedor = proveedor
    
    def obtener_datos_json(self) -> dict:
        import xml.etree.ElementTree as ET
        root = ET.fromstring(self.proveedor.obtener_datos_xml())
        return {child.tag: child.text for child in root}

# Uso
proveedor = ProveedorExterno()
adaptador = AdaptadorJSON(proveedor)
print(adaptador.obtener_datos_json())  # {'valor': '42'}
```

#### Decorator (Patrón, no `@decorator`)
```python
from abc import ABC, abstractmethod

class Notificador(ABC):
    @abstractmethod
    def enviar(self, mensaje: str): ...

class NotificadorBase(Notificador):
    def enviar(self, mensaje: str):
        print(f"Enviando: {mensaje}")

class NotificadorConLog(Notificador):
    def __init__(self, notificador: Notificador):
        self._notificador = notificador
    
    def enviar(self, mensaje: str):
        print(f"[LOG] Enviando: {mensaje}")
        self._notificador.enviar(mensaje)

class NotificadorConReintento(Notificador):
    def __init__(self, notificador: Notificador, reintentos: int = 3):
        self._notificador = notificador
        self._reintentos = reintentos
    
    def enviar(self, mensaje: str):
        for i in range(self._reintentos):
            try:
                self._notificador.enviar(mensaje)
                break
            except Exception as e:
                if i == self._reintentos - 1:
                    raise

# Uso
notif = NotificadorConReintento(
    NotificadorConLog(NotificadorBase()),
    reintentos=3
)
notif.enviar("Hola")
```

### 2.3 Comportamentales

#### Strategy
```python
from abc import ABC, abstractmethod

class OrdenadorStrategy(ABC):
    @abstractmethod
    def ordenar(self, datos: list) -> list: ...

class QuickSort(OrdenadorStrategy):
    def ordenar(self, datos):
        return sorted(datos)

class MergeSort(OrdenadorStrategy):
    def ordenar(self, datos):
        return sorted(datos)  # Python usa Timsort

class ContextoOrdenador:
    def __init__(self, strategy: OrdenadorStrategy):
        self._strategy = strategy
    
    def ejecutar(self, datos):
        return self._strategy.ordenar(datos)

# Uso
ctx = ContextoOrdenador(QuickSort())
ctx.ejecutar([3, 1, 2])
ctx._strategy = MergeSort()
ctx.ejecutar([3, 1, 2])
```

#### Observer
```python
from abc import ABC, abstractmethod
from typing import List

class Observador(ABC):
    @abstractmethod
    def actualizar(self, evento: str): ...

class Sujeto:
    def __init__(self):
        self._observadores: List[Observador] = []
        self._estado = ""
    
    def agregar(self, obs: Observador):
        self._observadores.append(obs)
    
    def notificar(self, evento: str):
        for obs in self._observadores:
            obs.actualizar(evento)
    
    @property
    def estado(self):
        return self._estado
    
    @estado.setter
    def estado(self, valor: str):
        self._estado = valor
        self.notificar(f"Estado cambiado a: {valor}")

class LoggerObserver(Observador):
    def actualizar(self, evento: str):
        print(f"[LOG] {evento}")

# Uso
sujeto = Sujeto()
sujeto.agregar(LoggerObserver())
sujeto.estado = "activo"
```

### 2.4 Patrones Pythonicos (Idiomáticos)

#### Context Manager
```python
from contextlib import contextmanager

@contextmanager
def transaccion(db):
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise

# Uso
with transaccion(db) as conn:
    conn.execute("INSERT ...")
```

#### Null Object
```python
class Usuario:
    def __init__(self, nombre: str):
        self.nombre = nombre
    def saludar(self): return f"Hola, {self.nombre}"

class UsuarioNulo(Usuario):
    def __init__(self):
        super().__init__("Anónimo")
    def saludar(self): return "Usuario no identificado"

def obtener_usuario(id: int) -> Usuario:
    if id in bd:
        return bd[id]
    return UsuarioNulo()  # Nunca None

usuario = obtener_usuario(999)
usuario.saludar()  # Seguro, no AttributeError
```

---

## 3. Buenas Prácticas Generales

### 3.1 Código Limpio

| Práctica | Ejemplo |
|----------|---------|
| **Nombres descriptivos** | `calcular_total_factura()` vs `calc()` |
| **Funciones pequeñas** | < 20 líneas, una responsabilidad |
| **Evitar magic numbers** | `MAX_REINTENTOS = 3` vs `range(3)` |
| **Type hints siempre** | `def f(x: int) -> str:` |
| **Docstrings en APIs públicas** | Google/NumPy style |
| **Evitar mutable defaults** | `def f(lista=None): lista = lista or []` |
| **Preferir `pathlib`** | `Path("archivo.txt").read_text()` |
| **Usar `enum`** | `class Color(Enum): ROJO = 1` |
| **Dataclasses** | `@dataclass class Punto: x: float; y: float` |
| **Evitar `*args`/`**kwargs`** | Solo cuando sea necesario |

### 3.2 Manejo de Errores

```python
# Excepciones personalizadas
class ErrorDominio(Exception): pass
class UsuarioNoEncontrado(ErrorDominio): pass
class SaldoInsuficiente(ErrorDominio): pass

# Encadenar excepciones
try:
    riesgo()
except ValueError as e:
    raise ErrorDominio("Fallo") from e

# Context manager para limpieza
from contextlib import contextmanager

@contextmanager
def transaccion(db):
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise

# Uso
with transaccion(db) as conn:
    conn.execute(...)
```

### 3.3 Logging Estructurado

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        if record.exc_info:
            log["exception"] = self.formatException(record.exc_info)
        return json.dumps(log)

logging.basicConfig(level=logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logging.getLogger().handlers = [handler]

logger = logging.getLogger(__name__)
logger.info("Usuario creado", extra={"user_id": 123, "accion": "registro"})
```

---

## 4. Tipado Avanzado

### 4.1 Generics y TypeVars

```python
from typing import TypeVar, Generic, Sequence

T = TypeVar('T')

class Repositorio(Generic[T]):
    def __init__(self):
        self._items: list[T] = []
    
    def agregar(self, item: T) -> None:
        self._items.append(item)
    
    def obtener_todos(self) -> Sequence[T]:
        return self._items

repo_int = Repositorio[int]()
repo_int.agregar(1)
# repo_int.agregar("hola")  # Error mypy
```

### 4.2 Protocolos (Structural Subtyping)

```python
from typing import Protocol

class Serializador(Protocol):
    def serializar(self, obj: object) -> str: ...
    def deserializar(self, data: str) -> object: ...

class JSONSerializador:
    def serializar(self, obj):
        import json
        return json.dumps(obj.__dict__)
    
    def deserializar(self, data):
        import json
        return json.loads(data)

def guardar(obj: object, serializador: Serializador) -> str:
    return serializador.serializar(obj)

guardar(Usuario("Juan"), JSONSerializador())  # OK, cumple protocolo
```

### 4.3 TypedDict y Literal

```python
from typing import TypedDict, Literal, NotRequired

class UsuarioDict(TypedDict):
    nombre: str
    email: str
    edad: NotRequired[int]  # opcional (3.11+)
    rol: Literal["admin", "user", "guest"]

def crear_usuario(data: UsuarioDict) -> None:
    pass

crear_usuario({"nombre": "Juan", "email": "juan@ejemplo.com", "rol": "admin"})
# crear_usuario({"nombre": "Juan", "rol": "super"})  # Error mypy
```

### 4.4 Overload

```python
from typing import overload, Union

@overload
def procesar(dato: int) -> str: ...
@overload
def procesar(dato: str) -> int: ...
@overload
def procesar(dato: list) -> tuple: ...

def procesar(dato: Union[int, str, list]) -> Union[str, int, tuple]:
    if isinstance(dato, int):
        return str(dato)
    elif isinstance(dato, str):
        return len(dato)
    return tuple(dato)
```

### 4.5 Typed Exceptions

```python
from typing import Never

def validar_edad(edad: int) -> None:
    if edad < 0:
        raise ValueError("Edad inválida")
    if edad > 150:
        raise ValueError("Edad imposible")

def procesar(edad: int) -> str:
    validar_edad(edad)
    return f"Edad: {edad}"

# Type narrowing
def ejemplo(edad: int | None) -> str:
    if edad is None:
        return "Sin edad"
    # Aquí edad es int
    return f"Edad: {edad}"
```

---

## 5. Arquitectura Limpia

### 5.1 Estructura de Proyecto

```
src/
├── mi_paquete/
│   ├── __init__.py
│   ├── domain/           # Entidades, reglas de negocio
│   │   ├── entities.py
│   │   ├── value_objects.py
│   │   └── exceptions.py
│   ├── application/      # Casos de uso
│   │   ├── services.py
│   │   ├── dtos.py
│   │   └── ports.py      # Interfaces (Protocol)
│   ├── infrastructure/   # Implementaciones externas
│   │   ├── repositories.py
│   │   ├── external_apis.py
│   │   └── database.py
│   └── presentation/     # API, CLI, GUI
│       ├── api/
│       │   ├── routes.py
│       │   └── schemas.py
│       └── cli/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── pyproject.toml
└── README.md
```

### 5.2 Dependency Injection

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Protocol

# Puerto (Interfaz)
class RepositorioUsuarios(Protocol):
    def guardar(self, usuario: "Usuario") -> None: ...
    def obtener(self, id: int) -> "Usuario | None": ...

# Adaptador (Implementación)
class RepositorioSQL:
    def __init__(self, conexion):
        self.conexion = conexion
    
    def guardar(self, usuario: "Usuario"):
        self.conexion.execute("INSERT ...", (usuario.id, usuario.nombre))
    
    def obtener(self, id: int):
        row = self.conexion.execute("SELECT * FROM usuarios WHERE id=?", (id,))
        if row:
            return Usuario(row[0], row[1])

# Caso de uso (Application Service)
@dataclass
class CrearUsuario:
    repositorio: RepositorioUsuarios
    
    def ejecutar(self, nombre: str) -> "Usuario":
        usuario = Usuario(nombre=nombre)
        self.repositorio.guardar(usuario)
        return usuario

# Composición (main.py)
def main():
    db = ConexionDB()
    repo = RepositorioSQL(db)
    caso_uso = CrearUsuario(repo)
    usuario = repo.ejecutar("Juan")
```

---

## 6. Anti-Patrones a Evitar

| Anti-Patrón | Problema | Solución |
|-------------|----------|----------|
| **God Class** | Clase que hace todo | Separar responsabilidades (SRP) |
| **Spaghetti Code** | Código enredado | Funciones pequeñas, módulos |
| **Magic Numbers** | Constantes hardcodeadas | `MAX_REINTENTOS = 3` |
| **Mutable Default Args** | `def f(lista=[])` | `def f(lista=None): lista = lista or []` |
| **Catch-all Exception** | `except:` | `except SpecificError:` |
| **God Object** | Objeto que sabe todo | Dependency Injection |
| **Premature Optimization** | Optimizar antes de medir | `cProfile` primero |
| **Copy-Paste** | Código duplicado | Extraer función/clase |
| **Large Interfaces** | Interfaz con muchos métodos | Interface Segregation (Protocol) |
| **Tight Coupling** | Clases dependen de concretas | Dependency Inversion (Protocol) |

---

## 7. Herramientas de Calidad Automatizadas

```bash
# pre-commit hooks
pre-commit install
pre-commit run --all-files

# CI Pipeline
ruff check .
ruff format --check .
mypy .
pytest --cov=mi_paquete --cov-fail-under=80
bandit -r .
safety check
```

---

## 8. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Base sintáctica
- [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) - Tipos para type hints
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - Pipeline completo
- [Módulos y Paquetes](./Python%20-%20M%C3%B3dulos%2C%20Paquetes%20y%20Distribuci%C3%B3n.md) - Arquitectura de paquetes
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - Patrones concurrentes
- [Arquitectura Limpia](./Python%20-%20Patrones%2C%20Buenas%20Pr%C3%A1cticas%20y%20Tipado.md#5-arquitectura-limpia) - Dependency Injection

---

*Etiquetas: #python #patrones #solid #design-patterns #clean-code #typing #mypy #protocols #generics #dataclasses #dependency-injection #clean-architecture #anti-patterns #best-practices #refactoring #code-quality #type-hints #overload #typeddict #literal #protocol #singleton #factory #builder #adapter #decorator #strategy #observer #null-object #context-manager*