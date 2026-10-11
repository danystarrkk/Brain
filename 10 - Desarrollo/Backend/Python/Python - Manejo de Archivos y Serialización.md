---
title: "Python - Manejo de Archivos y Serialización"
tags:
  - lang/python
  - concept/io
  - concept/serialization
  - concept/files
  - level/intermedio
  - type/reference
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Python - Manejo de Archivos y Serialización

> **Objetivo**: I/O robusto, context managers, y serialización de datos (JSON, pickle, CSV, YAML, TOML, MessagePack).

---

## 1. Lectura/Escritura Básica

### 1.1 Modos de Apertura

```python
# Texto
open("archivo.txt", "r")   # leer (default)
open("archivo.txt", "w")   # escribir (trunca)
open("archivo.txt", "a")   # append
open("archivo.txt", "x")   # crear exclusivo (falla si existe)
open("archivo.txt", "r+")  # leer/escribir (no trunca)

# Binario
open("archivo.bin", "rb")
open("archivo.bin", "wb")
open("archivo.bin", "ab")

# Codificación
open("archivo.txt", "r", encoding="utf-8")
open("archivo.txt", "w", encoding="utf-8", newline="")  # sin traducción \n
```

### 1.2 Context Manager (`with`) - OBLIGATORIO

```python
# ✅ Correcto: cierre automático incluso con excepciones
with open("archivo.txt", "r", encoding="utf-8") as f:
    contenido = f.read()

# Múltiples archivos
with open("entrada.txt", "r") as fin, open("salida.txt", "w") as fout:
    for linea in fin:
        fout.write(linea.upper())

# ❌ Incorrecto: riesgo de recurso no liberado
f = open("archivo.txt")
contenido = f.read()
f.close()  # Si hay error antes, no se cierra
```

---

## 2. Lectura de Archivos de Texto

```python
with open("archivo.txt", "r", encoding="utf-8") as f:
    # Leer todo
    contenido = f.read()           # string completo
    
    # Leer línea a línea (iterador perezoso - memoria eficiente)
    for linea in f:
        print(linea.rstrip("\n"))  # rstrip quita \n final
    
    # Leer todas las líneas (lista)
    lineas = f.readlines()         # incluye \n
    lineas = [l.rstrip("\n") for l in f]  # limpio
    
    # Leer primera línea
    primera = f.readline()
    
    # Iterar con índice
    for i, linea in enumerate(f, 1):
        print(f"{i}: {linea.rstrip()}")
```

---

## 2. Escritura de Archivos de Texto

```python
# Escribir string
with open("salida.txt", "w", encoding="utf-8") as f:
    f.write("Hola mundo\n")
    f.write("Segunda línea\n")

# Escribir múltiples líneas
lineas = ["línea 1", "línea 2", "línea 3"]
with open("salida.txt", "w", encoding="utf-8") as f:
    f.writelines(f"{l}\n" for l in lineas)  # writelines NO añade \n

# Print directo a archivo
with open("salida.txt", "w", encoding="utf-8") as f:
    print("Hola", file=f)
    print("Mundo", file=f)
    print(1, 2, 3, sep=" | ", file=f)
```

---

## 3. Archivos Binarios

```python
# Copiar archivo binario (imagen, zip, exe)
with open("entrada.png", "rb") as fin, open("copia.png", "wb") as fout:
    # Por chunks (memoria eficiente para archivos grandes)
    for chunk in iter(lambda: fin.read(8192), b""):
        fout.write(chunk)

# O shutil (más simple)
import shutil
shutil.copy2("origen.png", "copia.png")  # preserva metadata

# Leer bytes completos
with open("archivo.bin", "rb") as f:
    datos = f.read()  # bytes

# Escribir bytes
with open("salida.bin", "wb") as f:
    f.write(b"\x00\x01\x02\x03")
```

---

## 4. Rutas con `pathlib` (Python 3.4+) - RECOMENDADO

```python
from pathlib import Path

# Crear paths
p = Path("carpeta") / "archivo.txt"
p = Path.home() / "documentos" / "archivo.txt"
p = Path(__file__).parent / "data" / "config.json"

# Operaciones comunes
p.exists()              # True/False
p.is_file()             # True si es archivo
p.is_dir()              # True si es directorio
p.stat()                # info (tamaño, mtime, etc.)
p.stat().st_size        # tamaño en bytes
p.suffix                # ".txt"
p.stem                  # "archivo" (sin extensión)
p.parent                # directorio padre
p.name                  # "archivo.txt"
p.absolute()            # Path absoluto
p.resolve()             # resuelve symlinks

# Lectura/escritura directa (Python 3.5+)
texto = p.read_text(encoding="utf-8")
p.write_text("contenido\n", encoding="utf-8")
bytes_data = p.read_bytes()
p.write_bytes(b"\x00\x01")

# Directorio
p.mkdir(parents=True, exist_ok=True)  # crea recursivamente
p.rmdir()                             # elimina directorio vacío
shutil.rmtree(p)                      # elimina recursivamente

# Iterar directorio
for archivo in p.iterdir():           # solo nivel 1
for archivo in p.rglob("*.py"):       # recursivo con patrón
for archivo in p.glob("**/*.txt"):    # recursivo glob

# Renombrar/mover
p.rename("nuevo_nombre.txt")
p.replace("destino.txt")              # sobrescribe si existe
```

---

## 5. Serialización JSON

### 5.1 Básico

```python
import json

data = {
    "nombre": "Juan",
    "edad": 30,
    "activo": True,
    "tags": ["python", "dev"],
    "metadata": {"ciudad": "Madrid", "pais": "ES"}
}

# Serializar (dumps) / Escribir archivo (dump)
json_str = json.dumps(data, ensure_ascii=False, indent=2)
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# Deserializar (loads) / Leer archivo (load)
data2 = json.loads(json_str)
with open("data.json", "r", encoding="utf-8") as f:
    data2 = json.load(f)
```

### 5.2 Opciones Avanzadas

```python
# Clase personalizada -> JSON
class Usuario:
    def __init__(self, nombre, email):
        self.nombre = nombre
        self.email = email

class UsuarioEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Usuario):
            return {"__class__": "Usuario", "nombre": obj.nombre, "email": obj.email}
        return super().default(obj)

json.dumps(usuario, cls=UsuarioEncoder)
# '{"__class__": "Usuario", "nombre": "Juan", "email": "juan@ejemplo.com"}'

# Decodificar con hook
def usuario_decoder(dct):
    if dct.get("__class__") == "Usuario":
        return Usuario(dct["nombre"], dct["email"])
    return dct

json.loads(json_str, object_hook=usuario_decoder)
```

### 5.3 Fechas y Tipos No Serializables

```python
from datetime import datetime, date
from decimal import Decimal
from uuid import UUID

data = {
    "fecha": datetime(2024, 1, 15, 10, 30),
    "precio": Decimal("19.99"),
    "id": UUID("12345678-1234-5678-1234-567812345678")
}

# Encoder personalizado
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, UUID):
            return str(obj)
        if hasattr(obj, "__dict__"):
            return obj.__dict__
        return super().default(obj)

json.dumps(data, cls=CustomEncoder)
# {"fecha": "2024-01-15T10:30:00", "precio": 19.99, "id": "12345678-1234-5678-1234-567812345678"}
```

---

## 6. Serialización Pickle (Solo Python)

```python
import pickle

data = {"clave": "valor", "lista": [1, 2, 3], "objeto": MiClase()}

# Serializar
pickle_bytes = pickle.dumps(data, protocol=pickle.HIGHEST_PROTOCOL)
with open("data.pkl", "wb") as f:
    pickle.dump(data, f, protocol=pickle.HIGHEST_PROTOCOL)

# Deserializar
data2 = pickle.loads(pickle_bytes)
with open("data.pkl", "rb") as f:
    data2 = pickle.load(f)

# Protocolos
# protocol=0: ASCII (legacy)
# protocol=1: binario antiguo
# protocol=2: Python 2.3+
# protocol=3: Python 3.0+
# protocol=4: Python 3.4+ (eficiente para objetos grandes)
# protocol=5: Python 3.8+ (out-of-band buffers)
# HIGHEST_PROTOCOL = mejor disponible
```

> ⚠️ **ADVERTENCIA**: `pickle` **NO es seguro** para datos no confiables. Permite ejecución de código arbitrario (`__reduce__`). **Nunca** `pickle.load()` datos de fuentes no confiables. Usa JSON para APIs/datos externos.

---

## 7. CSV (Valores Separados por Comas)

```python
import csv

# Leer
with open("datos.csv", "r", encoding="utf-8", newline="") as f:
    reader = csv.reader(f, delimiter=",")
    header = next(reader)  # primera fila
    for fila in reader:
        print(fila)  # lista de strings

# Leer como dict (recomendado)
with open("datos.csv", "r", encoding="utf-8", newline="") as f:
    reader = csv.DictReader(f)  # usa primera fila como claves
    for fila in reader:
        print(fila["nombre"], fila["edad"])

# Escribir
with open("salida.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["nombre", "edad", "ciudad"])
    writer.writerows([
        ["Juan", 30, "Madrid"],
        ["Ana", 25, "Barcelona"]
    ])

# DictWriter
with open("salida.csv", "w", encoding="utf-8", newline="") as f:
    fieldnames = ["nombre", "edad", "ciudad"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({"nombre": "Juan", "edad": 30, "ciudad": "Madrid"})
    writer.writerows([
        {"nombre": "Ana", "edad": 25, "ciudad": "Barcelona"}
    ])

# Opciones
csv.reader(f, delimiter=";", quotechar='"', quoting=csv.QUOTE_MINIMAL)
```

---

## 8. YAML (Requiere PyYAML)

```bash
pip install pyyaml
```

```python
import yaml

data = {
    "servidor": {"host": "localhost", "puerto": 8080},
    "base_datos": {"nombre": "app", "usuario": "admin"},
    "features": ["auth", "cache", "logging"]
}

# Escribir
with open("config.yaml", "w", encoding="utf-8") as f:
    yaml.dump(data, f, allow_unicode=True, sort_keys=False)

# Leer
with open("config.yaml", "r", encoding="utf-8") as f:
    data = yaml.safe_load(f)  # safe_load evita código arbitrario

# Múltiples documentos
yaml.dump_all([doc1, doc2], f)
docs = list(yaml.safe_load_all(f))
```

> ⚠️ **SIEMPRE usar `yaml.safe_load()`**, nunca `yaml.load()` (ejecuta código arbitrario).

---

## 9. TOML (Python 3.11+ `tomllib` / `tomli`)

```bash
pip install tomli  # Python < 3.11
# Python 3.11+: import tomllib (solo lectura)
```

```toml
# config.toml
[servidor]
host = "localhost"
puerto = 8080

[base_datos]
nombre = "app"
usuario = "admin"
password = "secret"

features = ["auth", "cache", "logging"]
```

```python
# Python 3.11+
import tomllib
with open("config.toml", "rb") as f:
    config = tomllib.load(f)

# Python < 3.11
import tomli
with open("config.toml", "rb") as f:
    config = tomli.load(f)

# Escritura (requiere tomli-w)
import tomli_w
with open("config.toml", "wb") as f:
    tomli_w.dump(config, f)
```

---

## 10. MessagePack (Binario, Rápido, Compacto)

```bash
pip install msgpack
```

```python
import msgpack

data = {"usuarios": [{"id": 1, "nombre": "Juan"}, {"id": 2, "nombre": "Ana"}]}

# Serializar
packed = msgpack.packb(data, use_bin_type=True)
with open("data.msgpack", "wb") as f:
    f.write(packed)

# Deserializar
unpacked = msgpack.unpackb(packed, raw=False)
with open("data.msgpack", "rb") as f:
    unpacked = msgpack.unpackb(f.read(), raw=False)
```

> **Ventajas**: Binario, más rápido y compacto que JSON. Ideal para comunicación entre servicios (microservicios, Redis, colas).

---

## 11. SQLite (Base de Datos Ligera Integrada)

```python
import sqlite3

# Conexión (archivo o :memory:)
conn = sqlite3.connect("app.db")
conn.row_factory = sqlite3.Row  # acceso por nombre de columna

cursor = conn.cursor()

# Crear tabla
cursor.execute("""
    CREATE TABLE IF NOT EXISTS usuarios (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nombre TEXT NOT NULL,
        email TEXT UNIQUE,
        edad INTEGER,
        creado TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
""")

# Insertar (parámetros seguros - previene SQL injection)
cursor.execute("INSERT INTO usuarios (nombre, email, edad) VALUES (?, ?, ?)",
               ("Juan", "juan@ejemplo.com", 30))

# Múltiples
usuarios = [("Ana", "ana@ejemplo.com", 25), ("Pedro", "pedro@ejemplo.com", 35)]
cursor.executemany("INSERT INTO usuarios (nombre, email, edad) VALUES (?, ?, ?)", usuarios)

conn.commit()

# Consultar
cursor.execute("SELECT * FROM usuarios WHERE edad > ?", (25,))
for row in cursor.fetchall():
    print(dict(row))  # Row -> dict

# Transacciones
with conn:
    cursor.execute("UPDATE usuarios SET edad = edad + 1 WHERE id = ?", (1,))
    # commit/rollback automático

conn.close()
```

---

## 12. Context Managers Personalizados

```python
from contextlib import contextmanager

@contextmanager
def abrir_archivo(ruta, modo="r", encoding="utf-8"):
    """Context manager personalizado."""
    f = open(ruta, modo, encoding=encoding)
    try:
        yield f
    finally:
        f.close()

# Uso
with abrir_archivo("archivo.txt") as f:
    print(f.read())

# Clase-based (más control)
class MiArchivo:
    def __init__(self, ruta, modo="r"):
        self.ruta = ruta
        self.modo = modo
        self.f = None
    
    def __enter__(self):
        self.f = open(self.ruta, self.modo)
        return self.f
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.f.close()
        return False  # no suprimir excepciones
```

---

## 13. Archivos Temporales

```python
import tempfile

# Archivo temporal (se elimina al cerrar)
with tempfile.NamedTemporaryFile(mode="w+", delete=True, encoding="utf-8") as tmp:
    tmp.write("datos temporales")
    tmp.flush()  # asegurar escritura en disco
    print(tmp.name)  # ruta del archivo
    # se elimina automáticamente al salir del with

# Directorio temporal
with tempfile.TemporaryDirectory() as tmpdir:
    print(tmpdir)
    # se elimina recursivamente al salir

# Archivo temporal persistente (no auto-delete)
tmp = tempfile.NamedTemporaryFile(mode="w+", delete=False, encoding="utf-8")
tmp.write("persistente")
tmp.close()
# recordar: os.unlink(tmp.name) cuando termine
```

---

## 14. Comparativa Formatos

| Formato | Tipo | Velocidad | Tamaño | Legible | Tipos Complejos | Streaming | Uso Típico |
|---------|------|-----------|--------|---------|-----------------|-----------|------------|
| **JSON** | Texto | Media | Media | ✅ | Limitado | ✅ | APIs, config, web |
| **Pickle** | Binario | Rápida | Media | ❌ | **Completo** | ❌ | Cache Python interno |
| **CSV** | Texto | Rápida | Pequeño | ✅ | Plano | ✅ | Datos tabulares, Excel |
| **YAML** | Texto | Lenta | Media | ✅ | Bueno | ❌ | Config, CI/CD |
| **TOML** | Texto | Media | Pequeño | ✅ | Bueno | ❌ | Config moderna (pyproject.toml) |
| **MessagePack** | Binario | **Muy rápida** | **Muy pequeño** | ❌ | Bueno | ✅ | Microservicios, Redis, RPC |
| **Protocol Buffers** | Binario | Muy rápida | Muy pequeño | ❌ | **Schema** | ✅ | gRPC, APIs alto rendimiento |
| **SQLite** | Binario | N/A | N/A | SQL | Relacional | Parcial | Datos locales, embebido |

---

## 15. Referencias Cruzadas

- [Fundamentos y Sintaxis](./Python%20-%20Fundamentos%20y%20Sintaxis.md) - Context managers, excepciones
- [Estructuras de Datos](./Python%20-%20Estructuras%20de%20Datos.md) - Tipos serializables
- [Módulos y Paquetes](./Python%20-%20M%C3%B3dulos%2C%20Paquetes%20y%20Distribuci%C3%B3n.md) - Distribución
- [Testing y Calidad](./Python%20-%20Testing%2C%20Calidad%20y%20Herramientas.md) - Tests de serialización
- [Concurrencia](./Python%20-%20Concurrencia%20Threading%20Multiprocessing%20Asyncio.md) - I/O asíncrono

---

*Etiquetas: #python #io #serialization #json #pickle #csv #yaml #toml #msgpack #sqlite #pathlib #context-manager #tempfile #archivos-binarios #contextlib*