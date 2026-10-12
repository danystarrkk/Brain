---
aliases: ["Testing Python", "pytest", "Calidad Python", "Linting Python"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Patrones, Buenas Prácticas y Tipado]]"
  - "[[Python - Módulos, Paquetes y Distribución]]"
  - "[[Python - Concurrencia Threading Multiprocessing Asyncio]]"
  - "[[Python - Redes y HTTP]]"
  - "[[Python - Proyectos Prácticos POO]]"
  - "[[Python - Manejo de Archivos y Serialización]]"
fuente: "https://docs.pytest.org/"
---

# Python - Testing, Calidad y Herramientas

Referencia completa del pipeline de calidad de un proyecto Python: **pytest** a fondo (fixtures, marks, monkeypatch, conftest, organización), comparación con `unittest`, cobertura y lo que **no** prueba, filosofía del mocking, testing property-based, el estado 2026 de linting/formato (**ruff** vs flake8/black), type checking (mypy/pyright), pre-commit, CI, TDD, pirámide de tests, más las herramientas de seguridad, profiling y debugging. Verificado con Python 3.14 (`unittest`/`doctest`/`pyright` ejecutados en esta máquina).

> [!info] Por qué tests en Python (tipado dinámico) son doble de importantes
> En un lenguaje estáticamente tipado, el compilador captura familias enteras de errores gratis. En Python, eso lo hacen **mypy/pyright** (estáticos) y **pytest** (dinámicos); los dos se complementan, ninguno reemplaza al otro. Un proyecto Python sin tests + sin checker es un proyecto donde cualquier refactor puede romper silenciosamente cualquier cosa — ver [[Python - Patrones, Buenas Prácticas y Tipado]] para el rol de cada uno.

## pytest: el estándar

```bash
pip install pytest pytest-cov pytest-mock pytest-asyncio pytest-xdist hypothesis
```

### Qué hace distinto a `unittest` (la razón de su victoria)

| Característica | `unittest` (stdlib) | `pytest` |
|---|---|---|
| Sintaxis | `self.assertEqual(a, b)` — verboso | `assert a == b` — se lee como inglés |
| Captura de errores | `assertEqual` muestra valores | **Reescritura de asserts**: muestra las evaluaciones intermedias (`assert datos["total"] == 100` → `assert 90 == 100`) |
| Setup/teardown | `setUp`/`tearDown` por clase | **fixtures** componibles, con scopes y finalización |
| Fixtures | `addCleanup`, `patch` manual | Inyección por nombre de parámetro + `conftest.py` |
| Parametrizar | `subTest` o bucle | `@pytest.mark.parametrize` (IDs legibles en el fallo) |
| Descubrimiento | `test_*` en clases `Test*` | `test_*.py`, `test_*`, métodos `test_*` (sin herencia) |
| Marcas | decorators propios | `@pytest.mark.slow` + `--strict-markers` |
| Salida | OK/FAIL | Traza limpia, **solo** tu código en el traceback (`--tb=short`) |

Ejecutado en esta máquina (verificación real, aunque mínimo):

```
$ python3 -m unittest unittest_demo.py
test_suma ... ok
Ran 2 tests — OK
```

La misma suite en pytest es ~40% menos código y produce mejores mensajes de fallo.

### El assert: semántica y reescritura

```python
def test_suma():
    resultado = 2 + 3
    assert resultado == 5        # pytest reescribe: en fallo muestra 'assert 5 == 5'... y más

def test_excepciones():
    with pytest.raises(ValueError, match="ISBN mal formado"):
        parsear("basura")        # falla si NO lanza, o si lanza otro tipo

def test_multi_assert ():
    assert (1, 2) == (1, 2)      # por defecto una aserción por assert;
    assert "a" in {"a": 1}       # para múltiples: pytest.assume (plugin) o combinar
```

`pytest.raises` es un context manager **y** captura el error para examinarlo: `excinfo.value`, `excinfo.value.args`. `pytest.warns` hace lo mismo para `warnings`.

### Fixtures: el corazón de pytest

Una fixture es una **función con `@pytest.fixture`** que prepara (y opcionalmente limpia) contexto; los tests la piden **por nombre de parámetro**:

```python
import pytest

@pytest.fixture
def usuario():                      # scope="function" (default): nueva por test
    return Usuario("Ana", "ana@e.com")

@pytest.fixture
def db(tmp_path):                   # fixture de la stdlib: dir temporal por test
    conn = sqlite3.connect(tmp_path / "test.db")
    yield conn                      # ← yield = hay teardown después del test
    conn.close()                    # se ejecuta SIEMPRE, incluso si el test falla

def test_usuario_creado(usuario, db):     # inyección por nombre
    db.execute("INSERT INTO usuarios ...")
    ...
```

**Scopes** (el orden de creación):

| Scope | Creada | Uso típico |
|---|---|---|
| `function` (default) | Por cada test | Aislamiento puro |
| `class` | Una vez por clase `Test*` | Contexto caro compartido por tests relacionados |
| `module` | Una vez por archivo | Módulo importado/setup caro |
| `package` | Una vez por package | Raro |
| `session` | Una vez por corrida completa | Conexión a BD de testing, servidor embebido |

> [!warning] Scope amplio = estado compartido = tests frágiles
> Una fixture `session` que muta un dict global hace que el test 47 dependa del test 12. Regla: **compartí solo lo que es caro e inmutable** (conexiones, clientes); los datos de entrada siempre fresh. Y el teardown de `yield` corre incluso con fallos — pero no con `SIGKILL`; ahí solo queda el diseño idempotente.

**Factory fixtures** (fixtures que producen fixtures):

```python
@pytest.fixture
def crear_usuario():                     # el factory
    def _crear(nombre="Ana", rol="user"):
        return Usuario(nombre=nombre, rol=rol)
    return _crear

def test_admin(crear_usuario):
    admin = crear_usuario(rol="admin")
    assert admin.es_admin()
```

**Parametrizar fixtures**: `@pytest.fixture(params=[...])` genera un test por valor; el parámetro `request` lo entrega (`request.param`).

### Parametrización y marks

```python
@pytest.mark.parametrize("a,b,esperado", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_sumar(a, b, esperado):
    assert sumar(a, b) == esperado       # 4 tests con IDs: test_sumar[2-3-5]...

@pytest.mark.slow
def test_carga_masiva():
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="solo POSIX")
def test_signals(): ...

@pytest.mark.xfail(reason="bug conocido #123")    # esperado fallar; XPASS = señal de arreglo
def test_bug_viejo(): ...
```

```toml
# pyproject.toml — configuración canónica
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
addopts = "-v --strict-markers --strict-config"
markers = [
    "slow: tests largos; correr con -m 'not slow'",
    "integration: requieren servicios externos",
]
asyncio_mode = "auto"          # pytest-asyncio: async def test_* sin decorator
```

`--strict-markers` convierte los typos de mark (`@pytest.mark.sloww`) en errores — sin él, marks mal escritos se ignoran silenciosamente y tu `pytest -m "not slow"` deja correr los lentos.

### `monkeypatch`: el reemplazo seguro (mejor que patch suelto)

```python
def test_config(monkeypatch, tmp_path):
    monkeypatch.setenv("APP_ENV", "test")                 # entorno
    monkeypatch.delenv("HOME", raising=False)
    monkeypatch.setattr("mi_modulo.time.sleep", lambda s: None)   # por string
    monkeypatch.setattr(time, "sleep", lambda s: None)             # por objeto
    monkeypatch.setitem(os.environ, "K", "v")
    monkeypatch.chdir(tmp_path)                           # cwd temporal
    assert config().env == "test"
    # TODO se revierte SOLO al terminar el test — automático, sin teardown manual
```

`monkeypatch` es scope-function y se **revierte solo**; no deja parches pegados cuando el test falla a mitad (el riesgo clásico de `unittest.mock.patch` mal anidado).

### `conftest.py`: la jerarquía de fixtures

Los fixtures definidos en `conftest.py` están disponibles en **toda** la rama de directorios sin imports:

```
tests/
├── conftest.py          # fixtures compartidas: db, cliente_api, usuario
├── unit/
│   ├── conftest.py      # extras solo de unit
│   └── test_reglas.py
└── integration/
    └── test_api.py      # usa db y cliente_api del conftest raíz
```

Reglas: `conftest.py` **nunca** se importa explícitamente (pytest lo carga solo); plugins propios van en `conftest` con `@pytest.fixture`; la lógica reutilizable de verdad va en un módulo `tests/helpers.py` importado.

### Organizar tests (layout de un proyecto real)

```
tests/
├── conftest.py               # fixtures de sesión y datos base
├── unit/
│   ├── test_entidades.py     # dominio puro: sin I/O, milisegundos
│   └── test_servicios.py     # con repos fake
├── integration/
│   ├── test_repositorio_json.py    # disco real (tmp_path)
│   └── test_api.py                 # TestClient de FastAPI / servidor embebido
└── e2e/
    └── flujo_completo.py           # pocos: alta→préstamo→devolución
```

Nombres que cuentan la historia: `test_<unidad>_<escenario>_<resultado>` → `test_prestamo_libro_prestado_lanza_error`.

### `tmp_path`, `capsys`, `caplog`, `request`

| Fixture stdlib | Para qué |
|---|---|
| `tmp_path` | Directorio temporal `Path` por test (reemplaza a tempfiles manuales) |
| `tmp_path_factory` | Versión session-scope |
| `capsys` / `capdb` | Captura `print`/`stdout+stderr` para assertear salidas |
| `caplog` | Captura `logging` (`caplog.records`, `caplog.text`) |
| `monkeypatch` | Ver arriba |
| `request` | Metadata del test (`request.param`, `request.node.name`) |
| `pytester` | Tests **de** pytest (plugins) |

```python
def test_cli_imprime(capsys):
    main()                            # main usa print()
    out = capsys.readouterr().out
    assert "Listo" in out

def test_errores_logueados(caplog):
    with caplog.at_level(logging.ERROR, logger="mi_app"):
        procesar_malo()
    assert any("timeout" in r.message for r in caplog.records)
```

## Cobertura: `pytest-cov` y lo que NO prueba

```bash
pytest --cov=mi_paquete --cov-report=term-missing --cov-report=html --cov-fail-under=80
# equivalente manual: coverage run -m pytest && coverage report -m && coverage html
```

```toml
[tool.coverage.run]
branch = true                    # cobertura de RAMAS: if/else cada uno
source = ["mi_paquete"]
[tool.coverage.report]
exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError", "pragma: no cover"]
```

> [!warning] "El 90% de cobertura" ≠ "el 90% testeado"
> Cobertura mide **líneas ejecutadas**, no **aserciones correctas**: un test que llama a la función sin `assert` cubre el 100% y prueba nada. Y con `branch=true` cuenta ramas, pero sigue sin saber si el resultado fue el correcto. Herramientas que van más allá: **mutación** (`mutmut`, `cosmic-ray`: si cambiar `<=` por `<` sobrevive a toda la suite, hay un hueco) y property-based testing (abajo). Umbrales sanos: 80% global, 100% en dominio financiero/crítico, y **cero** tolerancia a código sin asserts.

## Mocking: filosofía y cuándo NO mockear

```python
from unittest.mock import patch, MagicMock, AsyncMock, call

def test_servicio_usa_repo():
    repo_falso = MagicMock()
    repo_falso.obtener.return_value = Usuario(id=1, nombre="Ana")
    servicio = ServicioUsuarios(repo=repo_falso)         # inyección: ni patch
    assert servicio.saludo() == "Hola, Ana"
    repo_falso.obtener.assert_called_once_with(1)

# patch por string del camino de import — SOLO cuando no podés inyectar
@patch("mi_modulo.requests.get")
def test_cliente(mock_get):
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"ok": True}
    assert cliente() == {"ok": True}

# async: AsyncMock (3.8+); un Mock normal en await lanza TypeError
@patch("mi_modulo.api_llamar", new_callable=AsyncMock)
async def test_async(mock_api):
    mock_api.return_value = 42
    assert await api_llamar() == 42
```

Cuándo **no** mockear:

1. **Objetos propios** → pasá fakes/estados reales baratos (un `Usuario` de verdad, un repo en `tmp_path`): los mocks de tus clases duplican su implementación y se rompen con cada refactor legítimo.
2. **La biblioteca que probás** → testear `requests` con mocks de requests prueba el mock.
3. **Datos** → fixtures reales (fábricas) > mocks de retorno.
4. Lo único legítimo de mockear: **fronteras que no controlás** (red, reloj, UUID, SMTP), y ahí el patrón de Python es primero **inyectar** (un callable `enviar_email` como parámetro) y solo después `patch`.

El test doble de oro: mockea **poco**, aserta **sobre tus objetos**, no sobre los llamados al mock — salvo para verificar interacciones que son el contrato (`assert_called_once_with`).

## Property-based testing (hypothesis)

Genera **cientos de entradas** aleatorias/borde y busca contraejemplos:

```python
from hypothesis import given, strategies as st, settings

@given(st.integers(), st.integers())
@settings(max_examples=500)
def test_suma_conmutativa(a, b):
    assert suma(a, b) == suma(b, a)

@given(st.text())
def test_parseo_no_crumpe(s):
    try: parsear(s)
    except ValueError: pass          # cualquier texto: o parsea o ValueError, nunca otra cosa
```

Hypothesis guarda los ejemplos encontrados (`.hypothesis/`) y **reproduce** el fallo en cada corrida. El caso de uso: algoritmos puros (serialización, parsing, orden, matemática de dominio). Verificado con `st.text()`/`st.integers()`: descubre casos borde (strings vacíos, unicode, INT_MIN) que ningún humano escribe en parametrize.

## Linting y formato: estado 2026

**ruff** reemplazó de facto a flake8+isort+pyupgrade+black en la mayoría de proyectos: escrito en Rust, ~10-100× más rápido, un solo binario, y su formatter es compatible con black.

| Herramienta | Rol | Estado 2026 |
|---|---|---|
| **ruff** (lint + format) | Lint, isort, pyupgrade, bugbear + formatter black-compatible | **Estándar moderno** |
| black | Formatter opinionado | Legítimo si ya está en el proyecto; migración trivial a `ruff format` |
| flake8 | Lint | Legado; ruff cubre sus reglas |
| isort | Imports | Absorbido por `ruff check --select I` |
| pylint | Lint con análisis de diseño | Nicho (métricas de diseño); ruff lo cubre parcialmente |

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py313"
[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "C4", "SIM", "FURB", "T20", "S", "N"]
ignore = ["E501"]                 # el formatter decide el largo
[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]              # asserts permitidos en tests
[tool.ruff.format]
quote-style = "double"
```

```bash
ruff check . --fix          # lint con autofix
ruff format .               # formato black-compatible
ruff check --select I .     # solo orden de imports (como isort)
```

`T20` (print) y `S` (bandit: seguridad) son las reglas que más bugs revelan en revisión.

## Type checking: mypy y pyright

Verificado en esta máquina con `pyright`:

```
$ pyright typecheck_demo.py
  Type of "doble(3)" is "int"
  error: Argument of type "Literal['hola']" cannot be assigned to parameter "n" of type "int"
1 error, 0 warnings
```

Cazó el `doble("hola")` **antes** de ejecutar nada — y `reveal_type` informó el tipo inferido. Configuración (ver detalle en [[Python - Patrones, Buenas Prácticas y Tipado]]):

```toml
[tool.mypy]
strict = true
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

| | mypy | pyright |
|---|---|---|
| Velocidad | Media | Muy rápida |
| Ecosistema | Plugins (Django, attrs...) | Pylance default en VS Code |
| PEP 695 | Soporte bueno | Soporte excelente |
| Veredicto | Estándar de facto | Mejor DX; basedpyright = modo estricto extra |

En CI: `mypy .` **o** `pyright --outputjson`; en editor: el checker corriendo en vivo.

## Seguridad estática y dependencias

```bash
pip install bandit pip-audit semgrep
bandit -r . -ll                          # patrones inseguros (eval, pickle, bind 0.0.0.0...)
bandit -r . -f json -o report.json
pip-audit                                # CVEs en dependencias instaladas
safety check                            # alternativa con base de datos comercial
semgrep --config=p/security-audit .     # reglas de patrones + p/secrets (cazasecretos)
```

Complemento: `pip install pip-compile`/`uv` para **fijar versiones** (`requirements.txt` hash-pinadas) — la mitad de la seguridad de supply chain es no instalar sorpresas. Ver [[Python - Módulos, Paquetes y Distribución]].

## pre-commit: la puerta antes del CI

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.15.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-json
      - id: check-merge-conflict
      - id: debug-logger          # print() olvidado
      - id: detect-private-key
```

```bash
pip install pre-commit
pre-commit install                  # por clon
pre-commit run --all-files          # limpieza inicial
```

El hook corre los **mismos binarios** que el CI (siempre via `rev` fijado): cero "en mi máquina pasa".

## CI: GitHub Actions (pipeline mínimo)

```yaml
name: CI
on: [push, pull_request]
jobs:
  calidad:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.12", "3.13", "3.14"]   # versiones soportadas reales
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: ruff format --check .
      - run: mypy .
      - run: bandit -r . -ll
      - run: pytest --cov=mi_paquete --cov-report=xml --cov-fail-under=80
      - uses: codecov/codecov-action@v4
        with: { files: ./coverage.xml }
```

Ordén de fallo: **lint (segundos) → tipos (segundos) → tests rápidos → tests lentos**: el feedback más rápido primero; `pytest -m "not slow"` en PRs y la suite completa en main/nightly.

## TDD aplicado a Python

Ciclo rojo-verde-refactor con pytest:

1. **Rojo**: escribí el test mínimo de un comportamiento que aún no existe → `pytest` → falla con `AttributeError`/`AssertionError` (la falla **esperada** es evidencia).
2. **Verde**: la implementación mínima (a veces un `return 0` placeholder) → pasa.
3. **Refactor**: extraer, renombrar, tipar — el test sigue verde.
4. Repetir; los tests quedan como documentación viva (ver [[Python - Proyectos Prácticos POO]] Proyecto 5: cada capa se construyó así).

TDD no aplica igual a: exploración/spike (primero descubrí, después testeá), GUI pixel-perfect (smoke tests), y código declarativo trivial. Tampoco es excusa para no testear lo heredado: **caracterización** — escribí tests que congelen el comportamiento actual antes de tocar.

## Pirámide de tests para proyectos Python

| Nivel | Proporción | Ejemplo | Herramienta |
|---|---|---|---|
| Unitario (dominio puro) | 70%, <50 ms c/u | Reglas de `Libro.prestar` | pytest |
| Integración (un subsistema) | 20% | Repositorio JSON con `tmp_path` | pytest |
| E2E (flujo completo) | <10% | Alta→préstamo→devolución vía CLI/API | pytest + servidor embebido |

Lo que NO es la pirámide: tests de UI por captura de pantalla (flaky) ni tests que solo llaman funciones sin asserts ("cobertura de mentira").

## Complejidad y métricas de diseño

```bash
pip install radon xenon
radon cc . -a          # complejidad ciclomática promedio por función (A-F; <B sano)
radon mi .             # Maintainability Index (0-100; <20 = rojo)
radon raw .            # LOC, comentadas, blancas
xenon --max-absolute B --max-modules A --max-average A .   # gate de CI: falla si degrada
```

La **complejidad ciclomática** (ramas `if/for/while/and/or`) es el mejor predictor de bugs por función: >10 = candidata a extraerse (ver tabla de olores). `xenon` convierte eso en regla de CI en lugar de consejo.

## Profiling y debugging

```bash
pip install py-spy scalene snakeviz ipdb
python -m cProfile -o perf.stats mi_script.py && snakeviz perf.stats   # árbol de llamadas
py-spy record -o profile.svg -- python mi_script.py   # sampling sin reiniciar; -d dump en vivo
python -m scalene mi_script.py                        # CPU + memoria + GPU, con atribución por línea
```

```python
breakpoint()                # pdb/ipdb según PYTHONBREAKPOINT (3.7+)
import pdb; pdb.set_trace() # pdb duro
import ipdb; ipdb.set_trace()
# remote: python -m debugpy --listen 5678 --wait-for-client mi_script.py  → attach del IDE
```

Cuándo cada uno: `breakpoint()` para desarrollo interactivo; `py-spy` para procesos que ya corren en prod (latencia, hilos colgados — ver Concurrencia); `scalene` para "¿por qué lento y qué come RAM?"; `cProfile` para ranking completo de llamadas.

## Documentación de la librería (Sphinx)

```bash
pip install sphinx furo sphinx-autodoc2
sphinx-quickstart docs
```

```python
# conf.py: extensions mínimas
extensions = ["sphinx.ext.autodoc", "sphinx.ext.napoleon",   # napoleon: docstrings Google/NumPy
              "sphinx.ext.viewcode", "sphinx.ext.intersphinx"]
html_theme = "furo"
```

Los docstrings estilo Google/NumPy (ver [[Python - Fundamentos y Sintaxis]]) alimentan `autodoc`: la documentación se genera desde el código y no se desincroniza. Para proyectos pequeños: README + `pydoc` alcanzan.

## Errores comunes

1. **Tests que dependen de otros tests** (orden de ejecución): si `pytest -p no:randomly` revienta con orden aleatorio, hay estado global sucio — fixtures fresh.
2. **`sleep` como sincronización** en tests concurrentes → joins/Events; ver [[Python - Concurrencia Threading Multiprocessing Asyncio]].
3. **Mockear objetos propios** → fakes + inyección; el mock se rompe con cada refactor.
4. **`patch("ruta.incorrecta")`** → el patch no aplica y el test "pasa" contra el código real (falso verde). El camino es el de **importación**, no el de definición.
5. **Cobertura sin aserciones** → ver sección de cobertura; los mutantes sobrevivientes lo delatan.
6. **Marks sin `--strict-markers`** → typos silenciosos.
7. **`assert` en producción apagado con `python -O`** → los asserts de validación de negocio no son `assert`: excepciones.
8. **Tests que abren la red** → CI sin internet = rojos aleatorios; `responses`/`pytest-httpx`/servidor local.
9. **`--cov-fail-under` bajo + refactors** → subilo incrementalmente; bajarlo es una señal de alarma del equipo.
10. **Fixture scope amplio con mutación** → efectos fantasma entre tests.

## Cheatsheet

| Tarea | Comando/código | Nota |
|---|---|---|
| Correr | `pytest` / `pytest -x` / `pytest -k "patron"` / `pytest -m "not slow"` | `-x`: para en el primero |
| Fixture + cleanup | `@pytest.fixture` + `yield` | Teardown automático |
| Contexto caro | `scope="session"` | Solo lo inmutable |
| Parametrizar | `@pytest.mark.parametrize("a,b", [...])` | IDs legibles |
| Excepción | `with pytest.raises(ValueError, match="..."):` | — |
| Parche reversible | `monkeypatch.setattr/setenv` | Se revierte solo |
| Capturar stdout | `capsys.readouterr().out` | — |
| Capturar logs | `caplog.at_level(...)` | — |
| Datos temporales | `tmp_path` | `Path` por test |
| Async | `asyncio_mode = "auto"` + `async def test_*` | pytest-asyncio |
| Cobertura | `pytest --cov=paquete --cov-fail-under=80` | `branch = true` |
| Lint+formato | `ruff check --fix . && ruff format .` | 2026 default |
| Tipos | `mypy --strict .` / `pyright` | Ambos, o el del equipo |
| Seguridad | `bandit -r .`, `pip-audit` | CI |
| Hooks | `pre-commit install` | Mismos binarios que CI |
| Debug | `breakpoint()` | `PYTHONBREAKPOINT=0` lo desactiva |
| Perf | `py-spy record`, `scalene` | Sampling sin reiniciar |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — asserts, docstrings, `breakpoint`, la base que se testea
- [[Python - Patrones, Buenas Prácticas y Tipado]] — arquitectura testeable, DI/Protocol, mypy profundo
- [[Python - Proyectos Prácticos POO]] — la suite del Proyecto 5 como ejemplo de pirámide completa
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — `pytest-asyncio`, tests deterministas
- [[Python - Redes y HTTP]] — `responses`/`pytest-httpx`: clientes sin red
- [[Python - Módulos, Paquetes y Distribución]] — `pyproject.toml` como casa de toda esta configuración
- [[Python - Manejo de Archivos y Serialización]] — `tmp_path` y tests de roundtrip
