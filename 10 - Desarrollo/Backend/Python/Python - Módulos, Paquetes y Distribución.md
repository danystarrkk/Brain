---
aliases: ["Módulos Python", "Paquetes Python", "Distribución Python", "pyproject Python"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Testing, Calidad y Herramientas]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python Library Hijacking]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
fuente: "https://docs.python.org/es/3/tutorial/modules.html"
---

# Python - Módulos, Paquetes y Distribución

Referencia profunda del sistema de módulos: mecánica real de `import` (sys.path, `sys.modules`, `__name__`), paquetes y namespace packages, importaciones relativas y ciclos, entornos virtuales, `pip`, el estándar moderno `pyproject.toml` (PEP 621), entry points y el panorama de gestión de dependencias en 2026. Verificado con Python 3.14.

## Módulos: qué es importar de verdad

Un **módulo** es un archivo `.py` (o extensión C `.so`) cuyo código se ejecuta **una vez** al importar y cuyo namespace queda cacheado. `import math_ops` no "copia" nada: busca, ejecuta el archivo en un módulo nuevo y lo registra.

### Las 6 formas de importar

```python
import math_ops                 # módulo completo: math_ops.sumar(...)
import math_ops as mo           # alias (obligatorio en nombres con guiones/estilo)
from math_ops import sumar      # nombres sueltos en el namespace local
from math_ops import sumar as s # alias de nombre
from math_ops import *          # ⚠️ TODOS los públicos — contamina namespace
import pkg.sub.mod              # importa la cadena de paquetes
```

| Forma | Riesgo | Estilo |
|---|---|---|
| `import x` / `import x as y` | ninguno | **default recomendado** — explícito de dónde viene cada cosa |
| `from x import y` | sombreamiento de nombres | ok para 1-2 símbolos frecuentes |
| `from x import *` | namespace pollution, ciclos invisibles, mypy/ruff lo marcan | solo en `__init__.py` con `__all__` definido |

### Mecánica interna del `import` (lo que ocurre)

1. Python busca el módulo en `sys.meta_path` (importadores hook: extensiones, builtins, PathFinder).
2. `PathFinder` recorre `sys.path` (lista de directorios): encuentra `nombre.py` o `nombre/__init__.py`.
3. Ejecuta el archivo en un **nuevo módulo** (`types.ModuleType`), con su `__name__`, `__file__`, `__package__`.
4. Lo registra en **`sys.modules`** — cache definitiva: imports posteriores devuelven ese objeto.
5. Delimita el nombre en el namespace del importador.

Verificado:

```python
# main_script.py
import sys
print(__name__)                 # "__main__" al correr directamente
                                # "main_script" al python -c "import main_script"
import mypkg
print(mypkg.VALOR)              # 42 — __init__.py se ejecutó al importar el paquete
print(mypkg.__name__)           # "mypkg" — los paquetes tienen su __name__ propio
import mypkg.mod_a as m
print("mypkg.mod_a" in sys.modules)   # True — cache en sys.modules
print(sys.path[0])              # '' (cwd) con -m; dir del script con python script.py
```

### `sys.path`: el orden de búsqueda

| Fuente | Prioridad | Nota |
|---|---|---|
| Directorio del script (o `''`=cwd con `-m`) | 1 | por eso `python mi_script.py` encuentra `utils.py` de su carpeta |
| `PYTHONPATH` | 2 | variable de entorno con rutas separadas por `:` |
| Instalación de Python (stdlib, site-packages) | 3 | `sys.prefix` |
| `.pth` / easy-install | 4 | herencia de setuptools |

```python
import sys
sys.path.insert(0, "/ruta/a/mis/modulos")   # funciona, pero es parche frágil
# export PYTHONPATH="/ruta:$PYTHONPATH"     # mejor para desarrollo suelto
# python -m paquete.script desde la raíz    # la forma correcta para paquetes
```

> [!warning] Seguridad: el cwd en `sys.path[0]`
> Al correr `python script.py`, el directorio del script está **primero** en el path: un `os.py` o `json.py` malintencionado ahí **sombrea la stdlib** (hijacking de import). Es la técnica detrás de [[Python Library Hijacking]] en escalada de privilegios — y la razón de no tener directorios escribibles por otros usuarios en el PATH ni hacer `python ./script` desde cwd sospechoso. El PEP 446 + `-P` (3.11+: no poner cwd en sys.path) mitigan.

### `__name__ == "__main__"` y ejecución como paquete

```python
# paquete/modulo.py
def principal(): ...
if __name__ == "__main__":
    principal()                    # solo si: python paquete/modulo.py
# con entry point (ver abajo): python -m paquete.modulo  →  __name__ == "__main__"
```

Patrón recomendado para CLIs de paquete: lógica en `principal()` + guard `__main__` + **console script** en pyproject (el usuario ejecuta `mi-cli` sin saber de Python).

### Importaciones relativas y sus reglas

```python
# dentro de paquete/sub/modulo_b.py:
from . import modulo_a            # hermano (mismo nivel) — paquete padre = __package__
from .modulo_a import suma        # nombre del hermano
from .. import algo               # UN nivel arriba
from ..otro import x              # paquete hermano
```

| Relativo | Resuelve a | Error típico |
|---|---|---|
| `.` | paquete actual (`__package__`) | — |
| `..` | padre | más `..` que niveles → `ImportError` |
| absoluto (`import x`) | desde sys.path | `ModuleNotFoundError` |

Verificado: importar un archivo con `from . import x` **fuera de paquete** lanza:

```
ImportError: attempted relative import with no known parent package
```

Reglas de oro: 1) **scripts sueltos → imports absolutos**; 2) dentro de un paquete instalable → absolutos (`from mi_paquete.modulo import f`) o relativos, pero **consistente en todo el paquete**; 3) los imports relativos rompen `python paquete/modulo.py` directo (por eso la CLI usa `-m` o entry points).

### Ciclos de importación

```python
# a.py: from b import B   |   b.py: from a import A   → ImportError o valores a medias
```

Síntomas: `ImportError: cannot import name ... due to circular import` o `AttributeError` con nombres "que existen". Causa: `a` empezó a ejecutarse, llama a `b`, `b` importa `a` **a medio construir** (está en `sys.modules` pero incompleto). Soluciones en orden de bondad:

1. **Mover el código compartido a un tercer módulo** (`base.py`) — rompe el ciclo de verdad.
2. Import **local** (dentro de la función) — aceptable si el import es lento/costoso y el ciclo es de runtime.
3. `import a` + uso tardío `a.f()` — el binding completo llega después.
4. TYPE_CHECKING para hints sin costo: `if TYPE_CHECKING: from b import B`.

## Paquetes

```
mi_paquete/
├── __init__.py          # marca paquete regular (y puede exponer API)
├── modulo_a.py
├── modulo_b.py
└── sub_paquete/
    ├── __init__.py
    └── modulo_c.py
```

### `__init__.py`: rol real

- En Python 3.3+ es **opcional** (existen los namespace packages), pero para paquetes normales **se recomienda siempre**: define el contrato público.
- Se ejecuta al `import mi_paquete` — lugar de inicialización barata (no cargues submódulos pesados ahí: cuesta tiempo de import en cada uso).

```python
# mi_paquete/__init__.py
__all__ = ["sumar", "PI"]                # controla from . import * (y documenta API)
from .modulo_a import sumar, resto       # "aplanar" la API: mi_paquete.sumar(...)
PI = 3.14159
__version__ = "1.2.3"
def __getattr__(name):                   # 3.7+: lazy submodule imports
    if name in {"pesado"}:
        from . import pesado; return pesado
    raise AttributeError(name)
```

### Namespace packages (3.3+)

Directorios **sin** `__init__.py` cuyo namespace se **parte** en varias rutas de `sys.path`:

```
dir1/mi_paquete/mod_a.py     y     dir2/mi_paquete/mod_b.py
# ambos importables: mi_paquete.mod_a / mi_paquete.mod_b — un "paquete" partido
```

Útiles para plugins instalados independientemente y para grandes organizaciones (`google.cloud.*`); peligrosos para principiantes (un directorio olvidado en el path "aparece" mágicamente). Regla: paquetes de tu proyecto = `__init__.py` explícito.

### Estructura de proyecto recomendada (layout `src/`)

```
mi_proyecto/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── mi_paquete/        # src/ impide importar el paquete SIN instalar
│       ├── __init__.py
│       └── modulo.py
├── tests/
│   └── test_modulo.py
└── docs/
```

> [!tip] ¿Por qué `src/`?
> Con el layout plano, `import mi_paquete` desde la raíz del repo funciona **sin instalar** — y tus tests pasan contra el código sin empaquetar, ocultando bugs de `pyproject.toml` (olvidos en `packages`, datos no incluidos). Con `src/`, el único camino es `pip install -e .`: los tests prueban **exactamente** lo que se instalará.

## Entornos virtuales: aislamiento obligatorio

```bash
python3 -m venv .venv                  # crea entorno (llama a pip por defecto)
source .venv/bin/activate              # activa (Windows: .venv\Scripts\activate)
deactivate                             # desactiva
python -m venv --upgrade-deps .venv    # pip/setuptools actualizados
uv venv                                # alternativa moderna, ~100× más rápida
```

Por qué: la stdlib vive en el intérprete, pero **toda librería externa debe estar aislada** por proyecto — versiones conflictivas entre proyectos, permisos de sistema (PEP 668: Debian/marcas marcan el Python del sistema como "externamente gestionado" y `pip install` global lanza error), y reproducibilidad. Un venv es solo: un link al intérprete + `site-packages` propio. **No** se versiona (`.venv/` en `.gitignore`); lo que se versiona es la *declaración* de dependencias.

## `pip`: instalación y dependencias

```bash
pip install requests                  # PyPI
pip install "requests>=2.28,<3"       # con restricciones
pip install -e .                      # editable (desarrollo)
pip install -e ".[dev]"               # extras del pyproject
pip install ./dist/mi_pkg-1.0-py3-none-any.whl
pip install git+https://github.com/u/repo.git@v1.0
pip install -r requirements.txt       # desde archivo
pip freeze > requirements.txt         # snapshot del entorno ACTUAL
pip download requests -d wheels/      # para vendorizar / air-gap
pip index versions requests           # versiones disponibles
```

### requirements.txt vs pyproject: roles distintos

| Archivo | Para quién | Quién lo escribe | Contenido |
|---|---|---|---|
| `pyproject.toml `[project]` | consumidores de tu **librería** | tú (autor) | rangos: `requests>=2.28,<3` |
| `requirements.txt` (aplicación) | tu **deploy** | humano o `pip-compile` | pins exactos: `requests==2.32.3` |
| lock (uv/poetry) | deploy reproducible | herramienta | pins + hashes |

`pip freeze` incluye TODAS las dependencias transitivas y el editable local — en aplicaciones reales se prefiere `pip-compile` (pip-tools): el `requirements.in` declara directas con rangos, y la herramienta resuelve y **fija** en `requirements.txt` con hashes. Librerías publicadas **no** fijan: dejan rangos amplios y resuelven los consumidores.

### Panorama de gestión de dependencias (estado 2026)

| Herramienta | Resolver | Lock | pyproject nativo | Velocidad | Nota |
|---|---|---|---|---|---|
| **pip + venv** | backtracking (23+) | con pip-tools | lee `[project]` | lenta | estándar mínimo, siempre funciona |
| **uv** | resolutor propio | ✅ `uv.lock` | ✅ | **~100× pip** | Rust; venv+pip+pip-tools+hatch en uno; estándar emergente |
| **poetry** | propio | ✅ `poetry.lock` | ✅ (adoptó PEP 621) | media | maduro, ecosistema fuerte |
| **pdm** | propio (PEP 582) | ✅ | ✅ | media | PEP-first, buen soporte de standards |
| **hatch / hatchling** | — (build backend) | con uv/pip-tools | ✅ | rápida | backend de build popular con uv |

> [!tip] Recomendación práctica 2026
> Librerías: `pyproject.toml` + cualquier backend (hatchling o setuptools) + publicar wheel. Aplicaciones: **uv** (`uv sync`, `uv run`) si podés adoptar lo nuevo; si no, pip + venv + pip-tools. Poetry sigue siendo válido si ya lo usás. Evitá invertir en pipenv (estancado).

## `pyproject.toml`: el estándar (PEP 621 + PEP 660)

```toml
[build-system]
requires = ["hatchling"]                  # o setuptools>=61
build-backend = "hatchling.build"

[project]
name = "mi-paquete"                       # en PyPI: guiones
version = "1.0.0"                         # o dinámico (abajo)
description = "Descripción corta"
readme = "README.md"
license = {text = "MIT"}                  # o license = "MIT" con PEP 639 (2024+)
authors = [{name = "Tu Nombre", email = "tu@email.com"}]
keywords = ["ejemplo"]
classifiers = [
  "Development Status :: 4 - Beta",
  "Programming Language :: Python :: 3",
]
requires-python = ">=3.10"
dependencies = ["requests>=2.28", "pydantic>=2.0"]

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.5", "mypy>=1.10"]

[project.scripts]
mi-cli = "mi_paquete.cli:main"            # console script (entry point)

[project.urls]
Homepage = "https://github.com/usuario/mi-paquete"
Issues = "https://github.com/usuario/mi-paquete/issues"

[tool.hatch.version]
path = "src/mi_paquete/__init__.py"       # versión dinámica desde __version__

[tool.ruff]
line-length = 88
target-version = "py310"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Entry points / console scripts

`[project.scripts]` crea un ejecutable en `bin/` del entorno que llama a la función indicada — el usuario ejecuta `mi-cli` sin `python -m`. Los **entry points de grupo** (`[project.entry-points."mi_plugin"]`) son el sistema de plugins nativo: `importlib.metadata.entry_points(group="mi_plugin")` descubre implementaciones de terceros (mismo mecanismo que los `plugin11y` de pytest/flake8). Ver [[Python - Librerías Estándar Esenciales]] → `importlib.metadata`.

## Construcción y publicación

```bash
pip install build twine          # (o: uv tool install build twine / pipx)
python -m build                  # genera dist/*.tar.gz (sdist) + *.whl
twine check dist/*               # valida metadatos
twine upload --repository testpypi dist/*   # prueba
pip install --index-url https://test.pypi.org/simple/ mi-paquete
twine upload dist/*              # PyPI (token: --username __token__ --password pypi-...)
```

- **Wheel** (`.whl`): ZIP con código ya empaquetado — instalación instantánea sin ejecutar setup. **sdist** (`.tar.gz`): fuente, necesario para que pip pueda reconstruir en otras plataformas.
- CI/CD: GitHub Actions con `pypa/gh-action-pypi-publish` sobre `release: published`; tokens con scope de proyecto vía Trusted Publishing (OIDC, sin token estático) es lo moderno.
- Versionado SemVer (`MAJOR.MINOR.PATCH`): MAJOR = breaking, MINOR = features compatibles, PATCH = fixes. Los tags de git alimentan la versión dinámica.

## `__all__`, star-imports y el contrato público

```python
# modulo.py
__all__ = ["sumar", "resto"]      # solo esto sale con from modulo import *
def sumar(a, b): ...
def resto(a, b): ...
def _interno(): ...               # sin __all__, lo público = sin _
```

- `from x import *` **no** respeta el `_` sin `__all__`, ejecuta código de inicialización, hace imposible rastrear de dónde viene cada nombre y rompe con ciclos. Los linters (`ruff` F403) lo marcan.
- `__all__` también afecta a la documentación (sphinx) y al re-export de `__init__.py`: define qué es API estable de tu paquete.
- Para re-exports explícitos y legibles: `from .modulo_a import sumar as sumar` (alias idéntico marca "re-export intencional" para mypy/pyflakes).

## Errores comunes

1. **`ModuleNotFoundError: No module named 'X'`** — el paquete no está en el entorno activo (olvidaste `activate` o `pip install`), o el nombre del archivo no coincide (guión vs underscore: el paquete `mi-paquete` se importa como `mi_paquete`).
2. **`ImportError: attempted relative import with no known parent package`** — corriste un módulo interno como script; usa `python -m paquete.script`.
3. **Ciclo de import** — ver sección dedicada; síntoma: `cannot import name X` que "debería" existir.
4. **Funciona local, falla instalado** — importabas desde el cwd (layout plano). Migrate a `src/` y probá con `pip install -e .`.
5. **Versión fantasma por namespace package accidental** — un directorio en el path con el mismo nombre "absorbe" el import.
6. **`pip install` en Python del sistema (Debian/macOS)** — PEP 668 `error: externally-managed-environment`; crea venv.
7. **Dependencias que faltan en deploy** — librería importada dentro de `try` u opcional; declarala en el extra correcto.
8. **Actualizar TODO con `pip install -U`** — rompe compat; cambia versiones con tests corriendo y lock file.
9. **Nombre PyPI duplicado/case-sensitive** — verifica disponibilidad antes de nombrar; publicá con token limitado.
10. **`__init__.py` con imports pesados** — penaliza cada `import tu_paquete`; usa lazy `__getattr__` (PEP 562).

## Recetas prácticas

```python
# 1. Versión accesible en runtime (estándar de paquetes)
# __init__.py:
__version__ = "1.2.3"
# consumidor:
from importlib.metadata import version
version("mi-paquete")           # la versión INSTALADA, no la del código fuente

# 2. Descubrimiento de plugins (entry points, 3.10+ simplificado)
from importlib.metadata import entry_points
for ep in entry_points(group="mi_plugin"):
    plugin = ep.load()           # importa y devuelve el objeto

# 3. Ejecutar un script como módulo desde otro código
import runpy
runpy.run_module("paquete.cli", run_name="__main__")

# 4. Recursos de datos del paquete (3.9+: files API)
from importlib.resources import files
config = files("mi_paquete.data").joinpath("default.json").read_text(encoding="utf-8")
```

## Cheatsheet — Módulos y paquetes

| Tema | Regla |
|---|---|
| Import default | `import x` / `import x as y` |
| Star import | evitar; solo en `__init__` con `__all__` |
| CLI de paquete | `python -m paquete` + guard `__main__` + console script |
| Relativos | solo dentro de paquetes; consistencia en todo el paquete |
| sys.path frágil | no editar en producción; `-P`/venv/layout src |
| Ciclos | tercer módulo / import local / TYPE_CHECKING |
| venv | siempre; uno por proyecto; `.venv` ignorado |
| Declaración | librería: rangos en pyproject; app: pins en lock/requirements |
| Build | `python -m build` → sdist + wheel; `twine check` |
| Versionado | SemVer + tags git + `__version__` dinámico |
| Plugins | `[project.entry-points."grupo"]` + `importlib.metadata` |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — `if __name__ == "__main__"`, ejecución de scripts
- [[Python - Librerías Estándar Esenciales]] — `importlib`, `pkgutil`, `sys`, `venv`
- [[Python - Testing, Calidad y Herramientas]] — layout de tests, CI con pip/uv
- [[Python - Manejo de Archivos y Serialización]] — TOML (`pyproject`), `importlib.resources`
- [[Python - Patrones, Buenas Prácticas y Tipado]] — arquitectura de paquetes y capas
- [[Python Library Hijacking]] — abuso ofensivo del orden de búsqueda de módulos
- [[Bash Scripting - Fundamentos Básicos]] — envolver CLIs de paquete en scripts shell
