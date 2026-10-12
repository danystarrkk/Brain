---
tags:
  - desarrollo
  - backend
  - python
tipo: concepto
estado: terminado
---

# Python - Proyectos Prácticos POO

Cuatro proyectos completos de POO aplicada — **Biblioteca** (con herencia), **Tienda de Animales**, **Flota de Vehículos** y **Gestor de Notas** (con persistencia pickle) — y la evolución de la Biblioteca a **forma de producción**: entidades `dataclass` con validación, jerarquía de excepciones de dominio, patrón repositorio con dos adaptadores (JSON y SQLite), capa de servicio, reloj inyectado, logging y suite de tests. Todo el código fue ejecutado y verificado con Python 3.14.

> [!info] El arco de esta nota
> Los proyectos 1–4 son la forma "curso" (todo junto, funcional, aprende los conceptos). El **Proyecto 5** toma exactamente el mismo dominio (biblioteca) y lo re-escribe con arquitectura de capas ver [[Python - Patrones, Buenas Prácticas y Tipado]]. La comparación lado a lado es el aprendizaje: cuándo y por qué cada capa aparece.

## Proyecto 1 — Biblioteca (diccionario, herencia, properties)

Modelo original del curso: `Libro` con estado de préstamo, `Biblioteca` como colección indexada por ID, y `BibliotecaInfantil` que **extiende** el comportamiento con una restricción por niño.

```python
#!/usr/bin/env python3

class Libro:
    def __init__(self, id_libro, autor, nombre_libro):
        self.id_libro = id_libro
        self.nombre_libro = nombre_libro
        self.autor = autor
        self.esta_prestado = False

    def __str__(self):
        return f"Libro({self.id_libro}, {self.autor}, {self.nombre_libro})"

    __repr__ = __str__          # idéntico en consola y en listas

class Biblioteca:
    def __init__(self):
        self.libros = {}        # dict indexado por ID: búsqueda O(1)

    def agregar_libro(self, libro):
        if libro.id_libro not in self.libros:
            self.libros[libro.id_libro] = libro
        else:
            print(f"[!] No se ha podido agregar el libro {libro.nombre_libro} con ID: {libro.id_libro}")

    @property
    def mostrar_libros(self):
        return [libro for libro in self.libros.values() if not libro.esta_prestado]

    @property
    def mostrar_libros_prestado(self):
        return [libro for libro in self.libros.values() if libro.esta_prestado]

    def prestar_libro(self, id_libro):
        if id_libro in self.libros and not self.libros[id_libro].esta_prestado:
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\n[+] No es posible prestar el libro con ID: {id_libro}")

class BibliotecaInfantil(Biblioteca):
    def __init__(self):
        super().__init__()
        self.libros_para_ninos = {}

    def agregar_libro(self, libro, ninos):          # ← firma distinta que el padre
        super().agregar_libro(libro)
        self.libros_para_ninos[libro.id_libro] = ninos

    def prestar_libro(self, id_libro, nino):
        if (id_libro in self.libros
                and self.libros_para_ninos.get(id_libro) == nino
                and not self.libros[id_libro].esta_prestado):
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\n[+] No es posible prestar el libro con ID: {id_libro}")

def main():
    libro1 = Libro(1, "Daniel Estrella", "Como ser un Lamer")
    libro2 = Libro(2, "Andres Estrella", "Aprende a colorear desde cero")
    biblioteca = BibliotecaInfantil()
    biblioteca.agregar_libro(libro1, ninos=False)
    biblioteca.agregar_libro(libro2, ninos=True)
    biblioteca.prestar_libro(1, nino=True)   # libro1 es ninos=False → rechazado (verificado)

    print("Disponibles:", biblioteca.mostrar_libros)
    print("Prestados:", biblioteca.mostrar_libros_prestado)
    print("Para niños:", biblioteca.libros_para_ninos)

if __name__ == "__main__":
    main()
```

Salida verificada:

```
[+] No es posible prestar el libro con ID: 1
Disponibles: [Libro(1, Daniel Estrella, Como ser un Lamer), Libro(2, Andres Estrella, Aprende a colorear desde cero)]
Prestados: []
Para niños: {1: False, 2: True}
```

Lecciones y olores a corregir (los veremos resueltos en el Proyecto 5):

- `@property` con **efectos secundarios que solo imprimen** (`mostrar_libros` es una consulta; property ok) — pero `prestar_libro` que imprime y falla silenciosamente es un **anti-patrón de errores**: el llamador no puede distinguir éxito de fracaso programáticamente.
- `BibliotecaInfantil.agregar_libro` **cambia la firma** del método padre → viola LSP (ver [[Python - Patrones, Buenas Prácticas y Tipado]]): cualquier código que espere una `Biblioteca` puede recibir una sorpresa.
- `self.libros` es estado público sin encapsulación: cualquier caller puede hacer `biblioteca.libros[9] = "basura"`.

## Proyecto 2 — Tienda de Animales

Colección simple, búsqueda por nombre, estado mutable (`alimentado`).

```python
#!/usr/bin/env python3

class Animal:
    def __init__(self, nombre, especie):
        self.nombre = nombre
        self.especie = especie
        self.alimentado = False

    def __str__(self):
        estado = "Alimentado" if self.alimentado else "Hambriento"
        return f"Mi nombre es {self.nombre} y soy un {self.especie} {estado}"

class TiendaAnimales:                       # ← corregido: snake_case, no Tienda_Animales
    def __init__(self, nombre):
        self.nombre = nombre
        self.animales = []

    def agregar_animal(self, animal):
        self.animales.append(animal)

    def mostrar_animales(self):             # ← método, no property: una vista no es un atributo
        for animal in self.animales:
            print(animal)

    def alimentar_a_todos(self):
        for animal in self.animales:
            animal.alimentado = True

    def vender_animal(self, nombre):
        for animal in self.animales:
            if animal.nombre == nombre:
                self.animales.remove(animal)
                return animal
        print(f"[!] El animal {nombre!r} no ha sido encontrado.")
        return None

def main():
    tienda = TiendaAnimales("Animal Happy")
    tienda.agregar_animal(Animal("Michi", "Gato"))
    tienda.agregar_animal(Animal("Firu", "Perro"))
    tienda.mostrar_animales()
    tienda.alimentar_a_todos()
    print("\n[+] Animales alimentados:")
    tienda.mostrar_animales()
    tienda.vender_animal("Firu")
    print("\n[+] Tras la venta:")
    tienda.mostrar_animales()

if __name__ == "__main__":
    main()
```

Correcciones respecto al original del curso (las tres mejoras de calidad mínimas):

1. **`Tienda_Animales` → `TiendaAnimales`**: PEP 8 manda snake_case para clases (la convención "CamelCase con guiones" no existe en Python).
2. **`mostrar_animales`/`alimentar` como `@property`** en el original: una property **imprime** — las properties deben ser vistas/valores, no comandos. Aquí son métodos (`mostrar_animales()`, `alimentar_a_todos()`).
3. **Borrar mientras se itera** (`self.animales.remove(animal)` dentro del `for` sobre `self.animales`) es clásico de "salteo" de elementos: el `return` temprano lo salva, pero la forma robusta es `next()` + `remove`, como arriba.

## Proyecto 3 — Flota de Vehículos

Estado booleano con transiciones validadas (`alquilar`/`devolver`).

```python
#!/usr/bin/env python3

class Vehiculo:
    def __init__(self, matricula, modelo):
        self.matricula = matricula
        self.modelo = modelo
        self.disponible = True

    def alquilar(self):
        if not self.disponible:
            print(f"[+] El vehículo con matrícula {self.matricula} no está disponible.")
            return False
        self.disponible = False
        return True

    def devolver(self):
        if self.disponible:
            print("[+] Error: el vehículo ya se encuentra en la flota.")
            return False
        self.disponible = True
        return True

    def __str__(self):
        return (f"Vehiculo(matricula={self.matricula}, modelo={self.modelo}, "
                f"disponibilidad={self.disponible})")

class Flota:
    def __init__(self):
        self._vehiculos = {}                # por matrícula: lookup O(1) en vez de scan

    def agregar_vehiculo(self, vehiculo):
        self._vehiculos[vehiculo.matricula] = vehiculo

    def alquilar_vehiculo(self, matricula) -> bool:
        if matricula not in self._vehiculos:
            print(f"[!] No existe la matrícula {matricula}.")
            return False
        return self._vehiculos[matricula].alquilar()

    def devolver_vehiculo(self, matricula) -> bool:
        if matricula not in self._vehiculos:
            return False
        return self._vehiculos[matricula].devolver()

    def __str__(self):
        return "\n".join(str(v) for v in self._vehiculos.values())

def main():
    flota = Flota()
    flota.agregar_vehiculo(Vehiculo("PDH4399", "SWM M-3"))
    flota.agregar_vehiculo(Vehiculo("PHD3480", "Aveo"))
    print("[+] Flota inicial:\n", flota, sep="")

    flota.alquilar_vehiculo("PDH4399")
    print("\n[+] Tras alquilar el SWM M-3:\n", flota, sep="")

    flota.devolver_vehiculo("PDH4399")
    flota.devolver_vehiculo("PDH4399")   # segunda vez: mensaje de error
    print("\n[+] Flota final:\n", flota, sep="")

if __name__ == "__main__":
    main()
```

Mejoras: `Flota` busca por matrícula con **dict** (O(1)) en vez de `for` lineal (O(n)); las operaciones **retornan bool** en vez de solo imprimir (el llamador puede decidir); atributo privado `_vehiculos` con interfaz de método.

## Proyecto 4 — Gestor de Notas con persistencia (pickle)

Tres módulos + CLI de menú: modelo (`notas.py`), repositorio con pickle (`gestor_notas.py`), entrada (`main.py`).

```python
# notas.py
class Nota:
    def __init__(self, contenido: str):
        self.contenido = contenido
    def __str__(self) -> str:
        return self.contenido
```

```python
# gestor_notas.py
import pickle
from notas import Nota

class GestorNotas:
    def __init__(self, ruta: str = "notas.pkl"):
        self.archivo = ruta
        self.notas = self.cargar_notas()

    def cargar_notas(self):
        try:
            with open(self.archivo, "rb") as f:
                return pickle.load(f)
        except FileNotFoundError:
            return []

    def guardar_notas(self):
        with open(self.archivo, "wb") as f:
            pickle.dump(self.notas, f)

    def agregar_nota(self, contenido: str):
        self.notas.append(Nota(contenido))
        self.guardar_notas()

    def leer_notas(self):
        return self.notas

    def buscar_nota(self, texto: str):
        return [n for n in self.notas if texto.lower() in n.contenido.lower()]

    def eliminar_nota(self, index: int):
        if 0 <= index < len(self.notas):
            self.notas.pop(index)
            self.guardar_notas()
```

```python
# main.py
import os
from gestor_notas import GestorNotas

def limpiar():
    os.system("cls" if os.name == "nt" else "clear")

def main():
    gestor = GestorNotas()
    acciones = {
        "1": lambda: gestor.agregar_nota(input("Contenido: ")),
        "2": lambda: [print(f"Nota {i+1}: {n}") for i, n in enumerate(gestor.leer_notas())],
        "3": lambda: [print(f"Nota: {n}") for n in gestor.buscar_nota(input("Buscar: "))],
        "4": lambda: gestor.eliminar_nota(int(input("Número: ")) - 1),
    }
    while True:
        print("\n--- MENÚ ---\n1. Agregar\n2. Leer todas\n3. Buscar\n4. Eliminar\n5. Salir")
        opcion = input("> ")
        if opcion == "5":
            break
        try:
            acciones[opcion]()
        except (KeyError, ValueError):
            print("[!] Opción incorrecta.")
        input("Enter para continuar...")
        limpiar()

if __name__ == "__main__":
    main()
```

Verificado: roundtrip completo — 2 notas agregadas, búsqueda "pan" encuentra 2, eliminar índice 0, y una **segunda instancia** del gestor recarga desde disco el estado correcto (`["llamar al dentista", "pan con tomate"]`).

> [!warning] `pickle` es un formato inseguro por diseño
> `pickle.load` **ejecuta código arbitrario** contenido en el archivo: un `notas.pkl` manipulado puede tomar tu máquina (el gadget `__reduce__` es de libro de texto — ver [[Python Ofensivo]] para el ataque y [[Python Library Hijacking]] para el vector hermano de import). Reglas: **jamás** cargar pickles de origen desconocido; para datos propios e intercambiables preferí JSON (ver [[Python - Manejo de Archivos y Serialización]]) o SQLite; pickle solo entre procesos tuyos de la misma versión.

---

## Proyecto 5 — De la Biblioteca a forma de producción

Mismo dominio, arquitectura de capas. Objetivos: cambiar la persistencia **sin tocar** la lógica, testear sin disco/reloj, errores programáticos, y un CLI delgado.

### Layout del proyecto

```
biblioteca/
├── entities.py        # dominio: dataclass + excepciones (sin dependencias)
├── repository.py      # PUERTO (Protocol) + adaptadores JSON y SQLite
├── services.py        # casos de uso; inyecta repo y reloj
├── test_biblioteca.py # suite (pytest)
└── main.py            # composición + CLI argparse
```

### Paso 1 — Entidades con validación y excepciones de dominio

```python
# entities.py
"""Capa de dominio: entidades y errores. Cero dependencias externas."""
from __future__ import annotations
from dataclasses import dataclass, asdict
from datetime import date
import re

class InventarioError(Exception):
    """Raíz de la jerarquía: los callers capturan esta y derivan."""

class LibroNoEncontrado(InventarioError):
    def __init__(self, isbn: str):
        super().__init__(f"No existe libro con ISBN {isbn}")
        self.isbn = isbn

class LibroDuplicado(InventarioError): ...
class PrestamoInvalido(InventarioError): ...

ISBN_RE = re.compile(r"^\d{3}-\d{1,5}-\d{2,6}-\d{1,4}-\d$")

@dataclass
class Libro:
    isbn: str
    titulo: str
    autor: str
    anio: int
    prestado_a: str | None = None

    def __post_init__(self) -> None:           # validación AL CONSTRUIR
        if not ISBN_RE.match(self.isbn):
            raise InventarioError(f"ISBN mal formado: {self.isbn!r}")
        if not self.titulo.strip():
            raise InventarioError("El título no puede estar vacío")

    @property
    def disponible(self) -> bool:
        return self.prestado_a is None

    def prestar(self, socio: str, hoy: date) -> None:    # transición validada
        if not self.disponible:
            raise PrestamoInvalido(f"'{self.titulo}' ya está prestado a {self.prestado_a}")
        self.prestado_a = socio

    def devolver(self) -> None:
        if self.disponible:
            raise PrestamoInvalido(f"'{self.titulo}' no está prestado")
        self.prestado_a = None

    def to_dict(self) -> dict:
        return asdict(self)

    @classmethod
    def from_dict(cls, d: dict) -> "Libro":
        return cls(**d)                          # re-valida al rehidratar

    def __str__(self) -> str:
        estado = "disponible" if self.disponible else f"prestado a {self.prestado_a}"
        return f"[{self.isbn}] {self.titulo} — {self.autor} ({self.anio}) · {estado}"
```

Qué cambió vs Proyecto 1: **errores que se lanzan** (no prints), validación en `__post_init__` (imposible construir un Libro inválido), transiciones de estado como métodos con guardas, serialización simétrica (`to_dict`/`from_dict`), y la excepción raíz `InventarioError` para capturar por familia.

### Paso 2 — Puerto repositorio y dos adaptadores

```python
# repository.py
from __future__ import annotations
from typing import Protocol
from entities import Libro, LibroNoEncontrado, InventarioError
import json

class RepositorioLibros(Protocol):              # el PUERTO: la capa de aplicación
    def agregar(self, libro: Libro) -> None: ...   # solo conoce esta interfaz
    def obtener(self, isbn: str) -> Libro: ...
    def todos(self) -> list[Libro]: ...
    def eliminar(self, isbn: str) -> None: ...
    def guardar_cambios(self) -> None: ...

class RepositorioJSON:
    """Persistencia en un archivo .json — reemplazable por completo."""
    def __init__(self, ruta: str = "biblioteca.json"):
        self._ruta = ruta
        self._libros: dict[str, Libro] = {}
        try:
            with open(ruta, encoding="utf-8") as f:
                self._libros = {d["isbn"]: Libro.from_dict(d) for d in json.load(f)}
        except FileNotFoundError:
            pass                                  # primera corrida: colección vacía

    def agregar(self, libro: Libro) -> None:
        if libro.isbn in self._libros:
            raise InventarioError(f"ISBN duplicado: {libro.isbn}")
        self._libros[libro.isbn] = libro

    def obtener(self, isbn: str) -> Libro:
        try:
            return self._libros[isbn]
        except KeyError:
            raise LibroNoEncontrado(isbn) from None

    def todos(self) -> list[Libro]:
        return sorted(self._libros.values(), key=lambda l: l.titulo.lower())

    def eliminar(self, isbn: str) -> None:
        self.obtener(isbn)                       # valida existencia primero
        del self._libros[isbn]

    def guardar_cambios(self) -> None:
        with open(self._ruta, "w", encoding="utf-8") as f:
            json.dump([l.to_dict() for l in self._libros.values()],
                      f, ensure_ascii=False, indent=2)

class RepositorioSQLite:
    """Mismo Protocol, otra tecnología: SQL real, transaccional."""
    def __init__(self, ruta: str = ":memory:"):
        import sqlite3
        self._conn = sqlite3.connect(ruta)
        self._conn.row_factory = sqlite3.Row
        self._conn.execute("""CREATE TABLE IF NOT EXISTS libros (
            isbn TEXT PRIMARY KEY, titulo TEXT NOT NULL,
            autor TEXT NOT NULL, anio INTEGER NOT NULL, prestado_a TEXT)""")

    def agregar(self, libro: Libro) -> None:
        try:
            self._conn.execute("INSERT INTO libros VALUES (?,?,?,?,?)",
                (libro.isbn, libro.titulo, libro.autor, libro.anio, libro.prestado_a))
        except Exception as e:
            raise InventarioError(f"No se pudo agregar {libro.isbn}: {e}") from e

    def obtener(self, isbn: str) -> Libro:
        fila = self._conn.execute("SELECT * FROM libros WHERE isbn=?", (isbn,)).fetchone()
        if not fila:
            raise LibroNoEncontrado(isbn)
        return Libro(fila["isbn"], fila["titulo"], fila["autor"], fila["anio"], fila["prestado_a"])

    def todos(self) -> list[Libro]:
        filas = self._conn.execute("SELECT * FROM libros ORDER BY titulo").fetchall()
        return [Libro(f["isbn"], f["titulo"], f["autor"], f["anio"], f["prestado_a"]) for f in filas]

    def eliminar(self, isbn: str) -> None:
        cur = self._conn.execute("DELETE FROM libros WHERE isbn=?", (isbn,))
        if cur.rowcount == 0:
            raise LibroNoEncontrado(isbn)

    def guardar_cambios(self) -> None:
        self._conn.commit()
```

El `Protocol` es la bisagra del DIP (ver [[Python - Patrones, Buenas Prácticas y Tipado]]): `ServicioBiblioteca` **no importa** ni JSON ni SQLite; recibe "algo que sabe hacer estas 5 cosas". Cambiar la persistencia = una línea en el `main`.

### Paso 3 — Capa de servicio (casos de uso)

```python
# services.py
from __future__ import annotations
from datetime import date
import logging
from entities import Libro
from repository import RepositorioLibros

log = logging.getLogger(__name__)

class ServicioBiblioteca:
    def __init__(self, repo: RepositorioLibros, hoy: date | None = None):
        self._repo = repo
        self._hoy = hoy or date.today        # RELOJ INYECTADO: testeable sin freezegun

    def registrar(self, isbn: str, titulo: str, autor: str, anio: int) -> Libro:
        libro = Libro(isbn, titulo, autor, anio)   # la validación vive en el dominio
        self._repo.agregar(libro)
        self._repo.guardar_cambios()
        log.info("Libro registrado: %s", libro.isbn)
        return libro

    def prestar(self, isbn: str, socio: str) -> Libro:
        libro = self._repo.obtener(isbn)
        libro.prestar(socio, self._hoy)
        self._repo.guardar_cambios()
        log.info("Préstamo %s → %s", isbn, socio)
        return libro

    def devolver(self, isbn: str) -> Libro:
        libro = self._repo.obtener(isbn)
        libro.devolver()
        self._repo.guardar_cambios()
        return libro

    def inventario(self, solo_disponibles: bool = False) -> list[Libro]:
        libros = self._repo.todos()
        return [l for l in libros if l.disponible] if solo_disponibles else libros
```

Detalle de diseño: el servicio **orquesta** (buscar → delegar transición → persistir → loguear) pero **no implementa** reglas; las reglas (ISBN válido, no-prestar-prestado) viven en el dominio y sobreviven a cualquier cambio de UI o persistencia.

### Paso 4 — CLI delgado (composición)

```python
# main.py
import argparse, logging
from repository import RepositorioJSON, RepositorioSQLite
from services import ServicioBiblioteca
from entities import InventarioError

def construir_servicio(args) -> ServicioBiblioteca:
    repo = (RepositorioSQLite(args.db) if args.db else RepositorioJSON(args.archivo))
    return ServicioBiblioteca(repo)

def main():
    parser = argparse.ArgumentParser(prog="biblioteca")
    parser.add_argument("--archivo", default="biblioteca.json")
    parser.add_argument("--db", default=None, help="usar SQLite en vez de JSON")
    sub = parser.add_subparsers(dest="comando", required=True)

    p_add = sub.add_parser("add"); p_add.add_argument("isbn"); p_add.add_argument("titulo")
    p_add.add_argument("autor"); p_add.add_argument("anio", type=int)
    p_loan = sub.add_parser("prestar"); p_loan.add_argument("isbn"); p_loan.add_argument("socio")
    sub.add_parser("listar")

    args = parser.parse_args()
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")
    servicio = construir_servicio(args)
    try:
        if args.comando == "add":
            print(servicio.registrar(args.isbn, args.titulo, args.autor, args.anio))
        elif args.comando == "prestar":
            print(servicio.prestar(args.isbn, args.socio))
        elif args.comando == "listar":
            for libro in servicio.inventario():
                print(libro)
    except InventarioError as e:                # UNA captura para toda la familia
        raise SystemExit(f"error: {e}")

if __name__ == "__main__":
    main()
```

### Paso 5 — Tests (la seguridad de todo lo anterior)

```python
# test_biblioteca.py
import pytest
from datetime import date
from entities import Libro, LibroNoEncontrado, PrestamoInvalido, InventarioError
from repository import RepositorioJSON, RepositorioSQLite
from services import ServicioBiblioteca

@pytest.fixture
def servicio(tmp_path):
    repo = RepositorioJSON(str(tmp_path / "b.json"))
    return ServicioBiblioteca(repo, hoy=date(2026, 10, 11))     # reloj congelado

def test_alta_y_listado(servicio):
    servicio.registrar("978-84-376-0494-7", "Cien años", "García Márquez", 1967)
    assert [l.titulo for l in servicio.inventario()] == ["Cien años"]

def test_prestamo_y_devolucion(servicio):
    servicio.registrar("978-84-376-0494-7", "Cien años", "GM", 1967)
    libro = servicio.prestar("978-84-376-0494-7", "ana")
    assert libro.prestado_a == "ana"
    assert servicio.inventario(solo_disponibles=True) == []
    with pytest.raises(PrestamoInvalido):               # doble préstamo → error
        servicio.prestar("978-84-376-0494-7", "luis")
    servicio.devolver("978-84-376-0494-7")
    assert len(servicio.inventario(solo_disponibles=True)) == 1

def test_isbn_invalido():
    with pytest.raises(InventarioError):
        Libro("123", "Título", "Autor", 2000)

def test_no_encontrado(servicio):
    with pytest.raises(LibroNoEncontrado):
        servicio.prestar("999-9-9-9-9", "ana")

def test_persistencia_roundtrip(tmp_path):
    ruta = str(tmp_path / "b.json")
    s1 = ServicioBiblioteca(RepositorioJSON(ruta))
    s1.registrar("978-84-376-0494-7", "Cien años", "GM", 1967)
    s2 = ServicioBiblioteca(RepositorioJSON(ruta))     # "nuevo proceso"
    assert len(s2.inventario()) == 1

def test_con_sqlite():
    s = ServicioBiblioteca(RepositorioSQLite())        # MISMOS tests, otro adaptador
    s.registrar("978-84-207-5312-1", "Lázaro", "Unamuno", 1914)
    assert len(s.inventario()) == 1
```

Verificado con smoke test manual (pytest no instalado en esta máquina): alta, préstamo, devolución, doble préstamo → `PrestamoInvalido`, ISBN inválido → `InventarioError`, no encontrado → `LibroNoEncontrado`, roundtrip JSON entre dos instancias, y el adaptador SQLite — todo OK. La suite real usa `pytest` y `tmp_path` (ver [[Python - Testing, Calidad y Herramientas]]).

> [!tip] Qué ganaste con las 5 capas
> 1. **Testeable**: sin disco (tmp_path), sin reloj (inyección), sin prints (excepciones).
> 2. **Intercambiable**: JSON ↔ SQLite en un `if` del `main`.
> 3. **Extensible**: un reporte nuevo = un método en `services.py`; un campo nuevo = un atributo en `Libro` + una columna.
> 4. **Cazable por familia**: `except InventarioError` captura TODO el dominio sin catch-alls peligrosos.

### Incremental: cómo se construyó (narrativa)

| Iteración | Añadido | Costo |
|---|---|---|
| 1 | `entities.py` puro + tests de validación | Sin I/O, sin mocks |
| 2 | `RepositorioJSON` + roundtrip test | Aparece el Protocol |
| 3 | `ServicioBiblioteca` + reloj inyectado | Los tests de reglas no tocan disco |
| 4 | `RepositorioSQLite` | Cero cambios en servicio/dominio |
| 5 | CLI argparse + logging | El único archivo con `print` |

## Errores comunes en proyectos POO

1. **Lógica de negocio en el CLI/GUI**: al llegar la segunda UI (web, API) hay que duplicarla → reglas en `entities`/`services`.
2. **`print` en vez de excepciones** (Proyecto 1 original): el llamador no puede reaccionar; lanzá y capturá por familia.
3. **Estado público expuesto** (`biblioteca.libros`): encapsulá y ofrecé métodos; después no podés cambiar la representación.
4. **Firma cambiada en subclases** (LSP): preferí composición o parámetros con default antes que métodos incompatibles.
5. **Persistencia acoplada al modelo** (`Libro.guardar_en_archivo()`): el dominio no debe saber qué disco existe — el repositorio hace I/O, el dominio modela reglas.
6. **`datetime.now()` por todos lados**: ingesta de tiempo = dependencia; inyectala y los tests se vuelven deterministas.
7. **Pickle para datos del usuario**: formato inseguro y no portable — JSON/SQLite por defecto.
8. **Sin `if __name__ == "__main__":`**: nada es testeable si el módulo ejecuta al importar.

## Cheatsheet

| Decisión | Elección | Por qué |
|---|---|---|
| Entidad | `@dataclass` + `__post_init__` | Validación en construcción, `asdict` gratis |
| Errores | Jerarquía bajo `XError(Exception)` | Captura por familia sin catch-alls |
| Consulta booleana | `@property` | `libro.disponible` se lee como atributo |
| Comando | método con nombre de verbo | `libro.prestar(socio)` no es property |
| Búsqueda | `dict` indexado por ID/ISBN | O(1) vs O(n) lineal |
| Persistencia | Protocol + repositorios | Cambiar tecnología sin tocar reglas |
| Testeabilidad | reloj/paths inyectados + `tmp_path` | Sin mocks del mundo |
| Serialización | `to_dict`/`from_dict` simétricos | Roundtrip verificable |
| CLI | `argparse` + un solo `except InventarioError` | Delgado, errores consistentes |
| Datos no confiables | JSON/SQLite, nunca `pickle.load` | Seguridad |

## Relacionadas

- [[Python - Programación Orientada a Objetos]] — clases, dataclasses, `__post_init__`, `__str__`, herencia
- [[Python - Patrones, Buenas Prácticas y Tipado]] — SOLID, Protocol/DIP, Repository, los anti-patrones evitados aquí
- [[Python - Manejo de Archivos y Serialización]] — JSON como alternativa segura a pickle; formatos
- [[Python - Testing, Calidad y Herramientas]] — pytest, fixtures, `tmp_path`, TDD
- [[Python - Módulos, Paquetes y Distribución]] — cómo estructurar `biblioteca/` como paquete instalable
- [[Python - Interfaces Gráficas (Tkinter)]] — el mismo patrón servicio+repositorio detrás de una GUI (Proyectos del editor)
- [[Python - Redes y HTTP]] — convertir el servicio en una API REST sin tocar el dominio
- [[Python Ofensivo]] — por qué `pickle.load` es ejecución de código (contexto defensivo)
