---
title: "Python - Módulos, Paquetes y Distribución"
tags:
  - lang/python
  - concept/modules
  - concept/packages
  - concept/distribution
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Módulos, Paquetes y Distribución

> **Objetivo**: Organizar código en módulos/paquetes reutilizables y publicarlos en PyPI.

---

## 1. Módulos

### 1.1 Qué es un Módulo

Un **módulo** es un archivo `.py` que contiene definiciones (funciones, clases, variables) que se pueden importar en otros scripts.

```python
# math_ops.py
def sumar(a: int, b: int) -> int:
    return a + b

def restar(a: int, b: int) -> int:
    return a - b

PI = 3.14159

class Calculadora:
    def multiplicar(self, a, b):
        return a * b
```

### 1.2 Importación

```python
# Importar módulo completo
import math_ops
math_ops.sumar(2, 3)

# Importar nombres específicos
from math_ops import sumar, restar
sumar(2, 3)

# Importar con alias
import math_ops as mo
mo.sumar(2, 3)

# Importar todo (NO RECOMENDADO)
from math_ops import *  # contamina namespace
```

### 1.3 `if __name__ == "__main__"`

```python
# math_ops.py
def sumar(a, b):
    return a + b

if __name__ == "__main__":
    # Solo se ejecuta si se corre directamente: python math_ops.py
    print("Probando módulo:", sumar(1, 2))
# Si se importa: import math_ops -> NO se ejecuta
```

### 1.4 Rutas de Búsqueda (`sys.path`)

```python
import sys
sys.path  # lista de directorios donde busca módulos

# Añadir ruta temporal
sys.path.insert(0, "/ruta/a/mis/modulos")

# Variable de entorno permanente
# export PYTHONPATH="/ruta/a/mis/modulos:$PYTHONPATH"
```

---

## 2. Paquetes

### 2.1 Qué es un Paquete

Un **paquete** es un directorio que contiene módulos y un archivo `__init__.py` (opcional en Python 3.3+ para *namespace packages*).

```
mi_paquete/
├── __init__.py          # Inicialización del paquete
├── modulo_a.py
├── modulo_b.py
└── sub_paquete/
    ├── __init__.py
    └── modulo_c.py
```

### 2.2 `__init__.py`

```python
# mi_paquete/__init__.py

# Qué se expone al hacer: from mi_paquete import *
__all__ = ["sumar", "PI", "Calculadora"]

# Importaciones convenientes (aplanar API)
from .modulo_a import sumar, restar
from .modulo_b import PI

# Código de inicialización
print("Cargando mi_paquete...")
```

### 2.3 Importación de Paquetes

```python
# Importar submódulo
import mi_paquete.modulo_a
mi_paquete.modulo_a.sumar(1, 2)

# Importar nombre específico
from mi_paquete.modulo_a import sumar
from mi_paquete import sumar  # si __init__ lo expone

# Importaciones relativas (dentro del paquete)
# En modulo_b.py:
from . import modulo_a          # mismo nivel
from .modulo_a import sumar    # submódulo
from .. import algo            # nivel superior
from ..otro_paquete import x   # paquete hermano
```

### 2.4 Paquetes de Espacio de Nombres (Namespace Packages)

Python 3.3+ permite paquetes sin `__init__.py` que pueden dividirse en múltiples directorios (ej. `pkg_resources`, `google.cloud`).

```python
# Estructura:
# dir1/mi_paquete/mod_a.py
# dir2/mi_paquete/mod_b.py
# Ambos accesibles como mi_paquete.mod_a / mod_b
```

---

## 3. Distribución: `pyproject.toml` (PEP 621/660)

### 3.1 Estructura Mínima

```
mi_proyecto/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── mi_paquete/
│       ├── __init__.py
│       └── modulo.py
├── tests/
└── docs/
```

### 3.2 `pyproject.toml` Completo

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "mi-paquete"
version = "1.0.0"
description = "Descripción corta del paquete"
readme = "README.md"
license = {text = "MIT"}
authors = [
  {name = "Tu Nombre", email = "tu@email.com"}
]
maintainers = [
  {name = "Tu Nombre", email = "tu@email.com"}
]
keywords = ["ejemplo", "utilidades"]
classifiers = [
  "Development Status :: 4 - Beta",
  "Intended Audience :: Developers",
  "License :: OSI Approved :: MIT License",
  "Programming Language :: Python :: 3",
  "Programming Language :: Python :: 3.9",
  "Programming Language :: Python :: 3.10",
  "Programming Language :: Python :: 3.11",
  "Programming Language :: Python :: 3.12",
]
requires-python = ">=3.9"
dependencies = [
  "requests>=2.28",
  "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=7.0",
  "black>=23.0",
  "ruff>=0.1",
  "mypy>=1.0",
]
docs = [
  "sphinx>=7.0",
  "furo>=2023.0",
]

[project.urls]
Homepage = "https://github.com/usuario/mi-paquete"
Repository = "https://github.com/usuario/mi-paquete"
Issues = "https://github.com/usuario/mi-paquete/issues"
Documentation = "https://mi-paquete.readthedocs.io"

[tool.setuptools.packages.find]
where = ["src"]
exclude = ["tests*"]

[tool.setuptools.package-dir]
"" = "src"

[tool.black]
line-length = 88
target-version = ["py39"]

[tool.ruff]
line-length = 88
target-version = "py39"
select = ["E", "F", "I", "W", "UP", "B", "C4", "FURB"]

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --cov=mi_paquete --cov-report=term-missing"
```

---

## 4. Construcción y Publicación

### 4.1 Herramientas

```bash
# Instalar herramientas de build
pip install build twine

# O usar pipx para herramientas aisladas
pipx install build
pipx install twine
```

### 4.2 Construir Distribuciones

```bash
# Desde raíz del proyecto (donde está pyproject.toml)
python -m build

# Genera en dist/:
# mi_paquete-1.0.0.tar.gz     (source distribution - sdist)
# mi_paquete-1.0.0-py3-none-any.whl  (wheel - binario)
```

### 4.3 Verificar Paquete

```bash
# Ver contenido
tar -tzf dist/mi_paquete-1.0.0.tar.gz
unzip -l dist/mi_paquete-1.0.0-py3-none-any.whl

# Verificar metadatos
twine check dist/*
```

### 4.4 Publicar en PyPI

```bash
# TestPyPI (primero probar)
twine upload --repository testpypi dist/*

# Instalar desde TestPyPI para probar
pip install --index-url https://test.pypi.org/simple/ mi-paquete

# PyPI oficial
twine upload dist/*

# Con token de API (recomendado)
twine upload --user __token__ --password pypi-AgEIcHl... dist/*
```

---

## 5. Versionado Semántico (SemVer)

```
MAJOR.MINOR.PATCH
1.0.0

MAJOR: Cambios incompatibles (breaking changes)
MINOR: Nueva funcionalidad compatible hacia atrás
PATCH: Fixes de bugs compatibles
```

```toml
# pyproject.toml - versión dinámica desde git tags
[tool.setuptools.dynamic]
version = {attr = "mi_paquete.__version__"}
```

```python
# src/mi_paquete/__init__.py
__version__ = "1.2.3"
```

---

## 6. Instalación Local y Desarrollo

```bash
# Instalar en modo desarrollo (editable)
pip install -e .              # desde raíz del proyecto
pip install -e ".[dev]"       # con dependencias de desarrollo

# Instalar desde git
pip install git+https://github.com/usuario/mi-paquete.git
pip install git+https://github.com/usuario/mi-paquete.git@v1.0.0
pip install git+https://github.com/usuario/mi-paquete.git@main#egg=mi_paquete

# Instalar desde archivo local
pip install ./dist/mi_paquete-1.0.0-py3-none-any.whl
pip install ./dist/mi_paquete-1.0.0.tar.gz
```

---

## 6. Dependencias y `requirements.txt`

```bash
# Generar requirements desde entorno actual
pip freeze > requirements.txt

# Instalar desde requirements
pip install -r requirements.txt

# Separar dependencias
# requirements.txt (producción)
# requirements-dev.txt (desarrollo)
pip install -r requirements-dev.txt
```

---

## 7. Publicación Automatizada (CI/CD)

### 7.1 GitHub Actions - Publicar en PyPI

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI
on:
  release:
    types: [published]

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install build tools
        run: pip install build twine
      - name: Build package
        run: python -m build
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

### 7.2 Tokens de API Seguros

```bash
# En PyPI: Account settings > API tokens > Add token
# Scope: "Entire account" o "Specific project"
# En GitHub: Settings > Secrets > PYPI_API_TOKEN
```

---

## 8. Licencias Comunes

| Licencia | Tipo | Uso Comercial | Modificación | Distribución | Patent Grant |
|----------|------|---------------|--------------|--------------|--------------|
| **MIT** | Permisiva | ✅ | ✅ | ✅ | ❌ |
| **Apache 2.0** | Permisiva | ✅ | ✅ | ✅ | ✅ |
| **BSD 3-Clause** | Permisiva | ✅ | ✅ | ✅ | ❌ |
| **GPL v3** | Copyleft | ✅ | ✅ | ✅ (misma licencia) | ✅ |
| **LGPL v3** | Copyleft débil | ✅ | ✅ | ✅ (misma licencia) | ✅ |
| **MPL 2.0** | Copyleft archivo | ✅ | ✅ | ✅ (misma licencia) | ✅ |

> **Recomendación**: MIT o Apache 2.0 para librerías; GPL v3 para aplicaciones completas.

---

## 9. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Importaciones básicas
- [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) - Tipos exportables
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - Tests en CI/CD
- [Patrones y Buenas Prácticas](./Python%20-%20Patrones%2C%20Buenas%20Pr%C3%A1cticas%20y%20Tipado.md) - Arquitectura de paquetes

---

*Etiquetas: #python #modulos #paquetes #distribucion #pypi #pyproject-toml #setuptools #wheel #sdist #semver #ci-cd #github-actions #licencias*