---
aliases: ["Manejo de Archivos Python", "Serialización Python", "Python Archivos"]
tags:
  - dev/backend/python
tipo: concepto
estado: terminado
creado: 2024-01-15
actualizado: 2026-10-11
relacionado:
  - "[[Python - Fundamentos y Sintaxis]]"
  - "[[Python - Estructuras de Datos]]"
  - "[[Python - Módulos, Paquetes y Distribución]]"
  - "[[Python - Librerías Estándar Esenciales]]"
  - "[[Python - Concurrencia Threading Multiprocessing Asyncio]]"
  - "[[Python - Testing, Calidad y Herramientas]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
fuente: "https://docs.python.org/es/3/library/filesys.html"
---

# Python - Manejo de Archivos y Serialización

Referencia profunda de E/S en Python: semántica de `open()` (modos, buffering, codificación, errores), context managers y por qué los archivos **deben** cerrarse, `pathlib` como API moderna, y el espectro completo de serialización (JSON, pickle, CSV, TOML, YAML, MessagePack, SQLite). Verificado con Python 3.14.

## `open()`: el contrato completo

```python
open(file, mode="r", buffering=-1, encoding=None, errors=None, newline=None, closefd=True)
```

### Tabla de modos (N completo)

| Modo | Lectura | Escritura | Trunca | Crea si no existe | Posición inicial | Requiere existir |
|---|---|---|---|---|---|---|
| `"r"` | ✅ | — | — | ❌ | inicio | ✅ (`FileNotFoundError`) |
| `"w"` | — | ✅ | ✅ **sí** | ✅ | inicio | — |
| `"a"` | — | ✅ | — | ✅ | **final** | — |
| `"x"` | — | ✅ | — | ❌ (falla si existe: `FileExistsError`) | inicio | ✅ |
| `"r+"` | ✅ | ✅ | — | ❌ | inicio | ✅ |
| `"w+"` | ✅ | ✅ | ✅ | ✅ | inicio | — |
| `"a+"` | ✅ | ✅ | — | ✅ | final (escritura) | — |

Modificadores: `"b"` (binario — **sin** encoding/newline, devuelve `bytes`), `"t"` (texto, default), `"U"` (obsoleto, no usar). Ejemplos: `"rb"`, `"wb"`, `"ab"`, `"r+b"`, `"wt"`. `"x"` es tu aliado para archivos-lock/caché: `try: f = open(p, "x") except FileExistsError: ...` — crea exactamente una vez sin race condition.

> [!warning] `"w"` trunca AL ABRIR, no al escribir
> `open(p, "w")` deja el archivo vacío **en el momento del open**, incluso si luego fallás antes de `write()`. Para escritura atómica: escribir a `p.with_suffix(".tmp")` y `os.replace(tmp, p)` (atómico en POSIX y Windows).

### Buffering: por qué `write` no es inmediato

| `buffering` | Comportamiento |
|---|---|
| `-1` (default) | tamaño por `io.DEFAULT_BUFFER_SIZE` (8 KiB típico) en texto; en binario = buffer size interno |
| `0` | solo binario, sin buffering (lectura/escritura inmediatas — raro) |
| `1`/`n` | línea (texto) o n bytes (binario) |

Implicaciones: `f.flush()` empuja al SO sin cerrar; `os.fsync(f.fileno())` empuja a **disco** (crítico en bancos/logs). Un `print` a archivo puede no verse en el disco hasta flush/close — por eso `with` cierra siempre.

### Codificación, `errors` y `newline`

```python
open("f.txt", "r", encoding="utf-8")
open("f.txt", "r", encoding="latin-1")                 # leer legacy sin romper
open("f.txt", "r", encoding="utf-8", errors="replace") # bytes ilegibles → '\ufffd'
open("f.csv", "r", encoding="utf-8", newline="")       # sin traducción de saltos (csv)
```

| `errors=` | Efecto | Cuándo |
|---|---|---|
| `strict` (default) | `UnicodeDecodeError` | default correcto: falla ruidoso |
| `replace` | carácter U+FFFD | inspección de datos rotos |
| `ignore` | descarta | **evitar**: pierde datos en silencio |
| `surrogateescape` | mapea a bytes 0xDC80-0xDCFF | round-trip de datos no validados (herramientas POSIX) |

`newline` controla la traducción `\n` ↔ `\r\n` (solo texto): `None` (default) traduce en lectura y usa `os.linesep`... no — en escritura escribe `\n` tal cual; `""` desactiva cualquier traducción (obligatorio con `csv` para no duplicar `\r`); `"x"` fuerza traducción a x. Verificado con `csv.writer`: el módulo csv por defecto termina filas con `\r\n` — con `newline=""` Python lo deja pasar limpio; sin él, en Windows se obtiene `\r\r\n`.

> [!info] Por qué los archivos deben cerrarse
> Cerrar libera: el descriptor de fichero del SO ( límite ~1024-4096 por proceso — `Too many open files`), el buffer del usuario (puede perderse escritura), y los locks. `with` usa el protocolo `__enter__`/`__exit__`: el cuerpo corre y **`finally` cierra incluso con excepción**. Nunca confíes en el GC de CPython para cerrar archivos: el colector de ciclos puede llegar tarde.

```python
# ✅ correcto
with open("archivo.txt", "r", encoding="utf-8") as f:
    contenido = f.read()

# varios a la vez
with open("in.txt") as fin, open("out.txt", "w") as fout:
    for linea in fin: fout.write(linea.upper())

# ❌ incorrecto: si read() lanza, f.close() nunca corre
f = open("archivo.txt"); contenido = f.read(); f.close()
```

## Patrones de lectura de texto

```python
with open("archivo.txt", encoding="utf-8") as f:
    contenido = f.read()            # TODO como str — archivos enteros en RAM
    f.seek(0)                       # rebobinar (texto: solo con seek(0) en la práctica)
    for linea in f:                 # ITERADOR PESOSO: lee buffer a buffer, O(1) memoria
        print(linea.rstrip("\n"))   # la línea conserva su \n
    lineas = f.readlines()          # lista de líneas (memoria = archivo)
    linea = f.readline()            # una línea; '' = EOF
    for i, linea in enumerate(f, 1):  # numerado
        ...
```

| Patrón | Memoria | Velocidad | Uso |
|---|---|---|---|
| `f.read()` | O(archivo) | rápido | archivos chicos/medianos (< ~100 MB) |
| `for linea in f` | **O(1)** | rápido | default para cualquier archivo grande |
| `f.readlines()` | O(archivo) | rápido | cuando necesitás la lista |
| `f.readline()` | O(línea) | medio | parsers manuales con lookahead |

Trampas: leer **dos veces** sin `seek(0)` da vacío (el puntero quedó al final); `read()` + `for` en el mismo `with` itera desde EOF; el EOF de `readline()` es `''` (falsy) — nunca compares con `None`.

```python
# Escritura
with open("salida.txt", "w", encoding="utf-8") as f:
    f.write("sin newline automático\n")   # write NO añade \n
    f.writelines(["tampoco\n", "añade\n"])  # writelines tampoco — nombre engañoso
with open("salida.txt", "a", encoding="utf-8") as f:
    print("con sep/end", file=f)          # print options a archivo
```

Copia binaria por chunks (memoria constante para archivos gigantes):

```python
with open("in.bin", "rb") as fin, open("out.bin", "wb") as fout:
    for chunk in iter(lambda: fin.read(8192), b""):   # sentinela b"" = EOF
        fout.write(chunk)
# atajo: shutil.copy2("in", "out") preserva metadata (mtime, permisos)
```

## `pathlib`: la API moderna de rutas (3.4+)

`pathlib.Path` object-oriented: `/` es composición de rutas, no string concat — portabilidad real entre Windows/POSIX.

```python
from pathlib import Path
p = Path("carpeta") / "archivo.txt"         # PosixPath('carpeta/archivo.txt')
p = Path.home() / "docs" / "config.json"
p = Path(__file__).parent / "data"          # relativo al script
```

| Grupo | API | Devuelve |
|---|---|---|
| Componentes | `.name`, `.stem`, `.suffix`, `.suffixes`, `.parent`, `.parts` | str/tupla |
| Tests | `.exists()`, `.is_file()`, `.is_dir()`, `.is_symlink()`, `.is_relative_to(q)` (3.9+) | bool |
| Metadata | `.stat()` (`.st_size`, `.st_mtime`), `.owner()`, `.chmod()` | `os.stat_result` |
| Lectura/Escritura | `.read_text(enc)`, `.write_text(s, enc)`, `.read_bytes()`, `.write_bytes(b)` | str/bytes |
| Directorios | `.mkdir(parents=True, exist_ok=True)`, `.rmdir()`, `.iterdir()`, `.glob("*.py")`, `.rglob("*.py")`, `.walk()` (3.12+) | iterador Path |
| Movimiento | `.rename(dest)`, `.replace(dest)` (sobrescribe), `.touch()`, `.unlink(missing_ok=True)` | — |
| Conversión | `.absolute()`, `.resolve()` (symlinks + `..`), `.as_uri()`, `Path.cwd()` | Path |

Verificado: `p.write_text("línea1\nlínea2\n", encoding="utf-8")` → `p.read_text()` lo devuelve exacto; `p.stat().st_size` coincide con bytes UTF-8. Escritura atómica idiomática:

```python
def escribir_atomico(path: Path, texto: str):
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(texto, encoding="utf-8")
    tmp.replace(path)              # os.replace por debajo: atómico
```

### `pathlib` vs `os`/`shutil`: tabla de equivalencias

| Tarea | `os` / `os.path` (legacy) | `pathlib` (moderno) |
|---|---|---|
| Unir rutas | `os.path.join(a, b)` | `a / b` |
| Existencia | `os.path.exists(p)` | `p.exists()` |
| Tamaño | `os.path.getsize(p)` | `p.stat().st_size` |
| Listar dir | `os.listdir(d)` | `list(p.iterdir())` |
| Recursivo | `os.walk(d)` | `p.walk()` (3.12+) o `p.rglob("*")` |
| Crear dirs | `os.makedirs(d, exist_ok=True)` | `p.mkdir(parents=True, exist_ok=True)` |
| Copiar | `shutil.copy2(src, dst)` | `shutil.copy2(src, p)` (acepta Path) |
| Borrar árbol | `shutil.rmtree(d)` | `shutil.rmtree(d)` (pathlib no tiene) |
| Mover | `shutil.move(a, b)` | `a.replace(b)` o `shutil.move` |
| Split ext | `os.path.splitext(p)` | `p.suffix` / `p.stem` |
| Absoluta | `os.path.abspath(p)` | `p.resolve()` |

Regla 2026: **`pathlib` por defecto**; `os`/`shutil` para operaciones que pathlib no cubre (rmtree, copy, walk viejo). `os.environ`, `os.getpid()`, `os.walk` siguen siendo de `os` — el módulo no desaparece.

## Context managers personalizados

```python
from contextlib import contextmanager, closing, suppress, ExitStack

@contextmanager
def abrir_archivo(ruta, modo="r", encoding="utf-8"):
    """Generator-based: yield = valor del with; try/finally = cleanup."""
    f = open(ruta, modo, encoding=encoding)
    try:
        yield f
    finally:
        f.close()

with abrir_archivo("archivo.txt") as f:
    print(f.read())
```

La forma clase (más control, soporta `as` + supresión de excepciones):

```python
class MiRecurso:
    def __init__(self, *args): ...
    def __enter__(self): return self       # el valor de `as`
    def __exit__(self, exc_type, exc, tb):
        self.cerrar()
        return False                       # True = SUPRIME la excepción (usar con cuidado)
```

| Herramienta de `contextlib` | Para qué |
|---|---|
| `contextmanager` | decorador generator → CM con 5 líneas |
| `ExitStack()` | apila N CMs dinámicos (`with ExitStack() as st: f = st.enter_context(open(p))`) |
| `closing(x)` | cierra cualquier objeto con `.close()` |
| `suppress(Exception)` | `with suppress(FileNotFoundError): os.unlink(p)` |
| `redirect_stdout(buf)` | capturar prints (tests) |
| `AsyncExitStack` | lo mismo para `async with` |

> [!tip] `__exit__` devolviendo True silencia excepciones
> Solo usalo para supresión **intencional** (p.ej. `suppress`). Un CM que devuelve True por accidente convierte errores críticos en no-op.

## Serialización: el mapa completo

### JSON (`json`) — intercambio estándar

```python
import json
s = json.dumps(data, ensure_ascii=False, indent=2, sort_keys=False)
data2 = json.loads(s)
json.dump(data, f, ensure_ascii=False, indent=2)   # a archivo (texto abierto)
data3 = json.load(f)
```

Verificado — semántica fina:

| Hecho verificado | Implicación |
|---|---|
| `json.dumps({"b":1,"a":2})` preserva orden `{'b','a'}` | round-trip ordenado desde 3.7 |
| `json.dumps({1: "x"})` → `'{"1": "x"}'` y `loads` devuelve claves **str** | claves int pierden el tipo — usa listas de pares si importa |
| `json.dumps({"t": datetime(...)}, default=str)` funciona | `default=` se invoca para tipos no serializables (solo valores; claves usan `default` en 3.x para keys no str/int/float/bool/None) |
| `separators=(",",":")` elimina espacios | json compacto para wire |
| `default` + `object_hook` | (de)serialización de clases custom |

Custom encoder completo (fechas, Decimal, UUID):

```python
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, (datetime, date)): return obj.isoformat()
        if isinstance(obj, Decimal): return float(obj)
        if isinstance(obj, UUID): return str(obj)
        if hasattr(obj, "__dict__"): return obj.__dict__
        return super().default(obj)     # TypeError con mensaje útil si nada aplica

def hook(dct):                          # decodificar marcas
    if dct.get("__class__") == "Usuario":
        return Usuario(dct["nombre"], dct["email"])
    return dct
json.loads(s, object_hook=hook)
```

Para máximo rendimiento: `orjson` (externo, C/Rust, 3-10× más rápido, serializa datetime/UUID nativos; `orjson.dumps` devuelve `bytes` — abrir archivo en `"wb"`). `ujson` similar. El stdlib es suficiente salvo volúmenes grandes.

### pickle — peligroso por diseño

```python
import pickle
b = pickle.dumps(data, protocol=pickle.HIGHEST_PROTOCOL)   # protocolo 5 desde 3.8
data2 = pickle.loads(b)
pickle.dump(data, open("d.pkl", "wb"))     # ¡siempre "rb"/"wb"! pickle es binario
```

| Protocolo | Versión | Nota |
|---|---|---|
| 0 | legado | ASCII, lento, grande |
| 2 | 2.3+ | baseline histórico |
| 3 | 3.0+ | default en Py3 temprano |
| 4 | 3.4+ | objetos grandes, amplio soporte |
| 5 | 3.8+ | out-of-band buffers (numpy grande sin copia) |

> [!warning] `pickle.load` = ejecución de código arbitrario
> Verificado con `pickletools.dis` — un objeto con `__reduce__` devuelve `(print, ("mensaje",))` y el bytecode del pickle contiene literalmente el opcode `REDUCE` que **llama a `builtins.print`** al deserializar. Un pickle hostil puede devolver `(os.system, ("rm -rf /",))`. Reglas: 1) **nunca** cargar pickle de fuente no confiable (red, uploads, caché compartido); 2) para datos externos, JSON; 3) si necesitás binario seguro, `JSON` + schema o `msgpack`; 4) caché local propio con hash de integridad, y aun así, es confianza implícita.

### CSV (`csv`)

```python
import csv
with open("datos.csv", "r", encoding="utf-8", newline="") as f:    # newline="" OBLIGATORIO
    reader = csv.reader(f, delimiter=",", quotechar='"')
    header = next(reader)                       # primera fila a mano
    for fila in reader:                         # lista de strings (siempre str)
        print(fila)

with open("datos.csv", encoding="utf-8", newline="") as f:
    for fila in csv.DictReader(f):              # dict por clave = header
        print(fila["nombre"], fila["edad"])

with open("out.csv", "w", encoding="utf-8", newline="") as f:
    w = csv.writer(f)
    w.writerow(["nombre", "edad"])
    w.writerows([["Juan", 30], ["Ana", 25]])
with open("out.csv", "w", encoding="utf-8", newline="") as f:
    w = csv.DictWriter(f, fieldnames=["nombre", "edad"])
    w.writeheader()
    w.writerow({"nombre": "Ana", "edad": 25})
```

Detalles: los valores se convierten a str al escribir; `csv.Sniffer` adivina delimitador; dialectos/`quoting=csv.QUOTE_ALL` para Excel-europeo (que usa `;`); `excel-tab` predefinido. Archivos grandes: `csv` es streaming-friendly, itera el reader directamente.

### TOML (config; lectura stdlib desde 3.11)

```python
# Python 3.11+
import tomllib
with open("config.toml", "rb") as f:            # TOML se lee como BINARIO
    config = tomllib.load(f)
# Python < 3.11: pip install tomli (API idéntica)
# Escritura: pip install tomli_w (el stdlib es solo lectura)
import tomli_w
open("out.toml", "wb").write(tomli_w.dumps(config).encode())
```

TOML es el formato de `pyproject.toml` (ver [[Python - Módulos, Paquetes y Distribución]]) — ver ese note para el rol en distribución.

### YAML (externo, PyYAML)

```python
import yaml
data = yaml.safe_load(open("config.yaml"))       # SIEMPRE safe_load
yaml.dump(data, open("out.yaml", "w"), allow_unicode=True, sort_keys=False)
docs = list(yaml.safe_load_all(f))               # multi-documento
```

> [!warning] `yaml.load()` sin `Loader=SafeLoader` ejecuta objetos Python arbitrarios (`!!python/object/apply:os.system`). `safe_load` es la única variante aceptable con datos ajenos. Idéntico threat model que pickle.

### MessagePack (binario, externo)

```python
import msgpack
packed = msgpack.packb(data, use_bin_type=True)     # str→str, bytes→bin (compat)
unpacked = msgpack.unpackb(packed, raw=False)       # raw=False: claves str
```

JSON-compatible pero binario: ~30-50% más pequeño y rápido. Ideal para colas/Redis/RPC entre servicios Python. Sin schema (a diferencia de protobuf).

### SQLite (base de datos embebida)

```python
import sqlite3
conn = sqlite3.connect("app.db")
conn.row_factory = sqlite3.Row                # filas con acceso por nombre
with conn:                                     # transacción: commit/rollback automático
    conn.execute("INSERT INTO usuarios (nombre, edad) VALUES (?, ?)", ("Juan", 30))
    conn.executemany("INSERT ... VALUES (?, ?)", filas)
rows = conn.execute("SELECT * FROM usuarios WHERE edad > ?", (25,)).fetchall()
print(dict(rows[0]))                           # Row → dict
conn.close()
```

- **Parámetros `?` siempre**: previene SQL injection por completo (nunca f-strings en SQL).
- `with conn:` gestiona la transacción; para contexto de conexión usa `contextlib.closing` o el propio conn-manager de 3.12+ (`with sqlite3.connect(...) as conn` cierra además transacciones).
- `:memory:` para tests; `detect_types`, `PRAGMA journal_mode=WAL` para concurrencia (ver [[Python - Concurrencia Threading Multiprocessing Asyncio]] para hilos — sqlite3 es thread-safe con check_same_thread=False + locks).

### Comparativa de formatos

| Formato | Tipo | Velocidad | Tamaño | Legible | Tipos | Seguro con datos ajenos | Uso |
|---|---|---|---|---|---|---|---|
| JSON | texto | media | medio | ✅ | limitado | ✅ | APIs, config, web |
| TOML | texto | media | pequeño | ✅ | bueno | ✅ | config, pyproject |
| YAML | texto | lenta | medio | ✅ | bueno | ⚠️ solo `safe_load` | config humana, CI |
| CSV | texto | rápida | pequeño | ✅ | plano | ✅ | tablas, Excel |
| pickle | binario | rápida | medio | ❌ | Python completo | ❌ **NO** | caché local propia |
| MessagePack | binario | **muy rápida** | **pequeño** | ❌ | bueno | ✅ | microservicios, colas |
| Protobuf | binario | muy rápida | muy pequeño | ❌ | schema estricta | ✅ | gRPC, alto rendimiento |
| SQLite | binario | — | — | SQL | relacional | ✅ | datos locales persistentes |

## Archivos temporales

```python
import tempfile
with tempfile.NamedTemporaryFile(mode="w+", delete=True, encoding="utf-8") as tmp:
    tmp.write("temporal"); tmp.flush()
    print(tmp.name)                       # ruta; se borra al salir del with
with tempfile.TemporaryDirectory() as d:  # directorio + todo su contenido
    p = Path(d) / "x.json"; p.write_text("{}")
# ojito Windows: NamedTemporaryFile no puede reabrirse por nombre mientras está abierto
# → usa delete=False + os.unlink manual, o TemporaryDirectory
```

Para archivos "secretos" usa `tempfile.mkstemp()` (permisos 0600) o `NamedTemporaryFile` con `dir=` controlado. Patrón atómico combinado: temporales + `os.replace`.

## Errores comunes

1. **`UnicodeDecodeError` al leer texto legacy** — abrir con `encoding="latin-1"` o `errors="replace"`; UTF-8 es default pero los archivos viejos de Windows (CP1252) mienten.
2. **`\r\r\n` en CSV/Windows** — olvidar `newline=""` con `csv`; verificado que csv escribe `\r\n` por defecto.
3. **`FileNotFoundError` con path relativo** — el cwd del proceso no es donde crees; usa `Path(__file__).parent / ...` para assets del script.
4. **`IsADirectoryError` / `NotADirectoryError`** — `open()` exige archivo; directorios se manejan con `pathlib.iterdir`.
5. **`OSError: [Errno 28] No space left`** o `Errno 118` — manejar `OSError` (padre de la familia de errores de archivos) con reintentos/limpieza.
6. **`Too many open files`** — handles sin cerrar en bucles; revisar `with`.
7. **JSON con NaN/Infinity** — el stdlib los emite (`NaN`), pero no es JSON válido para otros parsers: `allow_nan=False` para forzar validez.
8. **Pickle en texto** — `open("f.pkl")` sin `"b"` corrompe; pickle siempre binario.
9. **`load()` de YAML con `yaml.load`** — carga arbitraria; `safe_load` siempre.
10. **Leer dos veces sin `seek(0)`** — el puntero quedó al final; segundo `for` = vacío sin error.

## Recetas prácticas

```python
# 1. Procesar archivo gigante con memoria constante
from pathlib import Path
def contar_palabras(path):
    from collections import Counter
    with open(path, encoding="utf-8") as f:
        return Counter(p.lower() for line in f for p in line.split())

# 2. Config en capas: TOML + env + defaults
import os, tomllib
def cargar_config(p=Path("config.toml")):
    base = {"debug": False, "puerto": 8080}
    if p.exists():
        base |= tomllib.load(open(p, "rb"))
    base["puerto"] = int(os.environ.get("PORT", base["puerto"]))
    return base

# 3. Backup antes de escribir (pattern de scripts de ops)
def con_backup(path: Path):
    if path.exists(): path.replace(path.with_suffix(path.suffix + ".bak"))

# 4. Hash de archivo grande (3.11+: file_digest)
import hashlib
with open("iso.bin", "rb") as f:
    sha = hashlib.file_digest(f, "sha256").hexdigest()

# 5. Concatenar/generar CSV desde dicts
import csv, io
buf = io.StringIO()
w = csv.DictWriter(buf, fieldnames=["id", "valor"])
w.writeheader(); w.writerows(filas)
csv_bytes = buf.getvalue().encode("utf-8")

# 6. JSONL (línea-json) — streaming de logs/datasets
with open("data.jsonl", "w", encoding="utf-8") as f:
    for reg in registros:
        f.write(json.dumps(reg, ensure_ascii=False) + "\n")   # una línea = un registro
```

## Rendimiento

- Texto vs binario: binario + `decode` manual es ~2× más rápido para transformaciones pesadas (evita la traducción de newlines); para IO puro da igual.
- `read()` completo + procesar en memoria gana para archivos < caché; streaming gana con gigantes.
- `json` stdlib es C y decente; para >100 MB o rate alto, medí con `orjson` (`timeit` de referencia: orjson 3-10×).
- `pathlib` tiene overhead vs `os.path` en hot loops de millones de operaciones — irrelevante salvo benchmark demuestre lo contrario.

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — encoding de str/bytes, context managers como protocolo
- [[Python - Estructuras de Datos]] — qué tipos serializar y sus copias
- [[Python - Módulos, Paquetes y Distribución]] — `pyproject.toml` (TOML) y recursos del paquete
- [[Python - Librerías Estándar Esenciales]] — `tempfile`, `shutil`, `sqlite3`, `hashlib` en detalle
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — I/O no bloqueante y asyncio files
- [[Python - Testing, Calidad y Herramientas]] — fixtures con tmp_path, tests de parsers
- [[Comandos Básicos de Sistema y Red en Linux]] — el lado shell de la manipulación de archivos
