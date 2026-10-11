---
title: "Python - Testing, Calidad y Herramientas"
tags:
  - lang/python
  - concept/testing
  - concept/quality
  - concept/linting
  - concept/type-checking
  - level/intermedio
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Python - Testing, Calidad y Herramientas

> **Objetivo**: Pipeline completo de calidad: tests, linting, type checking, formatting, CI/CD.

---

## 1. Testing

### 1.1 `pytest` (Estándar Moderno)

```bash
pip install pytest pytest-cov pytest-mock pytest-asyncio pytest-xdist
```

```python
# test_ejemplo.py
import pytest
from mi_modulo import sumar, Usuario, calcular_descuento

# Fixtures (setup/teardown)
@pytest.fixture
def usuario_base():
    return Usuario("Juan", "juan@ejemplo.com")

@pytest.fixture(scope="session")
def db_connection():
    conn = crear_conexion_test()
    yield conn
    conn.close()

# Tests básicos
def test_sumar():
    assert sumar(2, 3) == 5
    assert sumar(-1, 1) == 0

# Parametrización
@pytest.mark.parametrize("a,b,esperado", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_sumar_parametrizado(a, b, esperado):
    assert sumar(a, b) == esperado

# Excepciones
def test_division_por_cero():
    with pytest.raises(ZeroDivisionError):
        1 / 0

# Mocking
def test_usuario_externo(usuario_base, mocker):
    mock_api = mocker.patch("mi_modulo.api_externa.obtener_datos")
    mock_api.return_value = {"rol": "admin"}
    
    resultado = usuario_base.obtener_rol()
    assert resultado == "admin"
    mock_api.assert_called_once_with(usuario_base.id)

# Async tests
@pytest.mark.asyncio
async def test_asyncio():
    async def tarea():
        await asyncio.sleep(0.01)
        return "ok"
    assert await tarea() == "ok"

# Markers
@pytest.mark.slow
def test_lento():
    time.sleep(2)
    assert True

@pytest.mark.integration
def test_integracion_db():
    pass

# Skip condicional
@pytest.mark.skipif(sys.platform == "win32", reason="Solo Linux")
def test_solo_linux():
    pass

# Configuración pytest.ini / pyproject.toml
"""
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --strict-markers --cov=mi_paquete --cov-report=term-missing --cov-fail-under=80"
markers =
    slow: tests lentos
    integration: tests de integración
    unit: tests unitarios
"""
```

### 1.2 `unittest` (Built-in)

```python
import unittest
from unittest.mock import patch, MagicMock, AsyncMock

class TestCalculadora(unittest.TestCase):
    def setUp(self):
        self.calc = Calculadora()
    
    def tearDown(self):
        pass
    
    def test_sumar(self):
        self.assertEqual(self.calc.sumar(2, 3), 5)
    
    def test_division_cero(self):
        with self.assertRaises(ZeroDivisionError):
            self.calc.dividir(1, 0)
    
    @patch('mi_modulo.servicio_externo')
    def test_con_mock(self, mock_servicio):
        mock_servicio.obtener.return_value = {"datos": "test"}
        resultado = self.calc.usar_servicio()
        self.assertEqual(resultado, "test")
        mock_servicio.obtener.assert_called_once()

if __name__ == "__main__":
    unittest.main()
```

### 1.3 Coverage y Reporting

```bash
# Coverage
pytest --cov=mi_paquete --cov-report=html --cov-report=term-missing --cov-fail-under=80

# Solo coverage
coverage run -m pytest
coverage report -m
coverage html
```

---

## 2. Linting y Formateo

### 2.1 `ruff` (Linter Rápido, Moderno)

```bash
pip install ruff
```

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py39"
select = [
    "E",   # pycodestyle
    "F",   # pyflakes
    "I",   # isort
    "W",   # pyflakes warnings
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "FURB", # refurb
    "SIM", # flake8-simplify
    "T20", # flake8-print
]
ignore = ["E501"]  # line too long (handled by formatter)
per-file-ignores = {
    "tests/*": ["S101"],  # allow assert in tests
}

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
```

```bash
# Lint
ruff check .
ruff check --fix .  # auto-fix

# Format
ruff format .
ruff format --check .  # solo verificar
```

### 2.2 `black` (Formateador Opinionado)

```bash
pip install black
```

```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ["py39"]
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | build
  | dist
)/
'''
```

```bash
black .
black --check .  # solo verificar (CI)
```

### 2.3 `isort` (Ordenar Imports) - Ahora integrado en ruff

```bash
# Si no usas ruff
pip install isort
isort .
isort --check-only .
```

---

## 3. Type Checking

### 3.1 `mypy` (Estándar)

```bash
pip install mypy
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true
strict_equality = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = "sqlalchemy.*"
ignore_missing_imports = true
```

```bash
mypy .
mypy --strict .
mypy --install-types --non-interactive .
```

### 3.2 `pyright` (Microsoft, Más Rápido)

```bash
pip install pyright
# o: npm install -g pyright
pyright
```

### 3.3 `pyre` (Facebook, Muy Rápido)

```bash
pip install pyre-check
pyre init
pyre check
```

---

## 4. Seguridad

### 4.1 `bandit` (Análisis Estático Seguridad)

```bash
pip install bandit
bandit -r .
bandit -r . -f json -o bandit-report.json
```

### 4.2 `pip-audit` / `safety` (Vulnerabilidades Dependencias)

```bash
pip install pip-audit
pip-audit

pip install safety
safety check
safety check --json
```

### 4.3 `semgrep` (Patrones de Código)

```bash
pip install semgrep
semgrep --config=auto .
semgrep --config=p/security-audit .
semgrep --config=p/secrets .
```

---

## 5. Pre-commit Hooks

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
  
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-json
      - id: check-merge-conflict
      - id: debug-logger
      - id: detect-private-key
  
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
  
  - repo: https://github.com/pycqa/isort
    rev: 5.13.0
    hooks:
      - id: isort
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests, types-PyYAML]

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: [-r, ., -ll]
```

```bash
pre-commit install
pre-commit run --all-files
```

---

## 6. CI/CD: GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      
      - name: Install dependencies
        run: |
          pip install -e ".[dev]"
      
      - name: Lint with Ruff
        run: ruff check .
      
      - name: Format with Ruff
        run: ruff format --check .
      
      - name: Type check with MyPy
        run: mypy .
      
      - name: Security check with Bandit
        run: bandit -r . -ll
      
      - name: Run tests with pytest
        run: pytest --cov=mi_paquete --cov-report=xml --cov-fail-under=80
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
```

---

## 7. Métricas de Complejidad

```bash
pip install radon xenon
```

```bash
# Complejidad ciclomática
radon cc . -a -nb

# Raw metrics
radon raw .

# Maintainability index
radon mi .

# Xenon (fail si complejidad alta)
xenon --max-absolute B --max-modules A --max-average A .
```

---

## 8. Documentación

```bash
pip install sphinx sphinx-autodoc2 furo
```

```bash
# Inicializar
sphinx-quickstart docs
```

```python
# conf.py extensions
extensions = [
    "sphinx.ext.autodoc",
    "sphinx.ext.napoleon",
    "sphinx.ext.viewcode",
    "sphinx.ext.intersphinx",
    "sphinx_autodoc2",
]

html_theme = "furo"
```

```bash
# Build
cd docs && make html
```

---

## 9. Profiling y Performance

```bash
pip install snakeviz py-spy scalene
```

```bash
# cProfile
python -m cProfile -o profile.stats mi_script.py
snakeviz profile.stats

# py-spy (sampling, bajo overhead)
py-spy record -o profile.svg -- python mi_script.py
py-spy top -- python mi_script.py

# Scalene (CPU + Memory + GPU)
python -m scalene mi_script.py
```

---

## 9. Debugging Avanzado

```python
# pdb
import pdb; pdb.set_trace()

# breakpoint() (Python 3.7+)
breakpoint()

# ipdb (mejor UX)
pip install ipdb
import ipdb; ipdb.set_trace()

# PuDB (GUI TUI)
pip install pudb
import pudb; pudb.set_trace()

# remote debugging
pip install debugpy
# python -m debugpy --listen 5678 --wait-for-client mi_script.py
```

---

## 9. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Sintaxis base
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - Tests async
- [Módulos y Paquetes](./Python%20-%20M%C3%B3dulos%2C%20Paquetes%20y%20Distribuci%C3%B3n.md) - Distribución con tests
- [Patrones y Buenas Prácticas](./Python%20-%20Patrones%2C%20Buenas%20Pr%C3%A1cticas%20y%20Tipado.md) - Arquitectura testeable
- [CI/CD](./Python%20-%20M%C3%B3dulos%2C%20Paquetes%20y%20Distribuci%C3%B3n.md#7-publicación-automatizada-cicd) - Pipeline completo

---

*Etiquetas: #python #testing #pytest #unittest #coverage #mocking #mypy #pyright #ruff #black #isort #pre-commit #bandit #semgrep #pip-audit #safety #radon #xenon #snakeviz #py-spy #scalene #pdb #ipdb #pudb #debugpy #profiling #profiling #ci-cd #github-actions #codecov #code-quality #linting #formatting #type-checking #static-analysis #security*