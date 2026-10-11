---
title: "Python - Proyectos Prácticos POO"
tags:
  - lang/python
  - concept/oop
  - concept/projects
  - concept/practice
  - level/intermedio
  - type/pattern
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Proyectos Prácticos POO

> **Objetivo**: Aplicar POO en proyectos reales: Biblioteca, Tienda Animales, Flota Vehículos, Gestor Notas.

---

## 1. Gestión de Biblioteca

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

    def __repr__(self):
        return self.__str__()

class Biblioteca:
    def __init__(self):
        self.libros = {}
    
    def agregar_libro(self, libro):
        if libro.id_libro not in self.libros:
            self.libros[libro.id_libro] = libro
        else:
            print(f"\n[!] No se ha podido agregar el libro {libro.nombre_libro} con ID: {libro.id_libro}")

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

    def agregar_libro(self, libro, ninos):
        super().agregar_libro(libro)
        self.libros_para_ninos[libro.id_libro] = ninos

    def prestar_libro(self, id_libro, nino):
        if id_libro in self.libros and self.libros_para_ninos[id_libro] == nino and not self.libros[id_libro].esta_prestado:
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\n[+] No es posible prestar el libro con ID: {id_libro}")

    @property
    def mostrar_libros_estado_libros_para_ninos(self):
        return self.libros_para_ninos

def main():
    libro1 = Libro(1, "Daniel Estrella", "Como ser un Lamer")
    libro2 = Libro(2, "Andres Estrella", "Aprende a colorear desde cero")

    biblioteca = BibliotecaInfantil()
    biblioteca.agregar_libro(libro1, ninos=False)
    biblioteca.agregar_libro(libro2, ninos=True)
    biblioteca.prestar_libro(1, nino=True)

    print(f"\n[+] Libros en la biblioteca {biblioteca.mostrar_libros}")
    print(f"\n[+] Libros prestados: {biblioteca.mostrar_libros_prestado}")
    print(f"\n[+] Libros para niños: {biblioteca.mostrar_libros_estado_libros_para_ninos}")

if __name__ == "__main__":
    main()
```

---

## 2. Tienda de Animales

```python
#!/usr/bin/env python3

class Animal:
    def __init__(self, nombre, especie):
        self.nombre = nombre
        self.especie = especie
        self.alimentado = False

    def __str__(self):
        return f"Mi nombre es {self.nombre} y soy un {self.especie} {'Alimentado' if self.alimentado else 'Hambriento'}"

class Tienda_Animales:
    def __init__(self, nombre):
        self.nombre = nombre
        self.animales = []

    def agregar_animal(self, animal):
        self.animales.append(animal)

    @property
    def mostrar_animales(self):
        for animal in self.animales:
            print(animal)

    @property
    def alimentar(self):
        for animal in self.animales:
            animal.alimentado = True

    def vender_animal(self, nombre):
        for animal in self.animales:
            if animal.nombre == nombre:
                animal.alimentado = False
                self.animales.remove(animal)
                return
        print(f"\n[!] El animal no ha sido encontrado.")

def main():
    tienda = Tienda_Animales("Animal Happy")
    gato = Animal("Michi", "Gato")
    perro = Animal("firu", "Perro")
    tienda.agregar_animal(gato)
    tienda.agregar_animal(perro)
    tienda.mostrar_animales
    tienda.alimentar
    print(f"\n[+] Mostrando animales alimentados\n")
    tienda.mostrar_animales
    tienda.vender_animal("firu")
    print(f"\n[+] Mostrando los animales restantes después de la venta:\n")
    tienda.mostrar_animales

if __name__ == "__main__":
    main()
```

---

## 3. Flota de Vehículos

```python
#!/usr/bin/env python3

class Vehiculo:
    def __init__(self, matricula, modelo):
        self.matricula = matricula
        self.modelo = modelo
        self.disponible = True
    
    def alquilar(self):
        if self.disponible:
            self.disponible = False
        else:
            print(f"[+] El vehículo con matrícula {self.matricula} no está disponible.")

    def devolver(self):
        if not self.disponible:
            self.disponible = True
        else:
            print("\n[+] Error en la matrícula, el vehículo ya se encuentra en la flota.")

    def __str__(self):
        return f"Vehiculo(matricula={self.matricula}, modelo={self.modelo}, disponibilidad={self.disponible})"

class Flota:
    def __init__(self):
        self.vehiculos = []
    
    def agregar_vehiculo(self, vehiculo):
        self.vehiculos.append(vehiculo)

    def __str__(self):
        return "\n".join(str(vehiculo) for vehiculo in self.vehiculos)

    def alquilar_vehiculo(self, matricula):
        for vehiculo in self.vehiculos:
            if vehiculo.matricula == matricula:
                vehiculo.alquilar()

    def devolver_vehiculo(self, matricula):
        for vehiculo in self.vehiculos:
            if vehiculo.matricula == matricula:
                vehiculo.devolver()

def main():
    flota = Flota()
    flota.agregar_vehiculo(Vehiculo("PDH4399", "SWM M-3"))
    flota.agregar_vehiculo(Vehiculo("PHD3480", "Aveo"))
    
    print(f"\n[+] Flota inicial:\n")
    print(flota)

    flota.alquilar_vehiculo("PDH4399")
    print(f"\n Mostrando la flota después de alquilar el SWM M-3:\n")
    print(flota)

    print(f"\n[+] Devolviendo el vehículo:\n")
    flota.devolver_vehiculo("PDH4399")
    print(flota)
    flota.devolver_vehiculo("PDH4399")

if __name__ == "__main__":
    main()
```

---

## 4. Gestor de Notas (con Pickle)

```python
#!/usr/bin/env python3
import os
from gestor_notas import GestorNotas

def main():
    gestor = GestorNotas()
    while True:
        print(f"\n{'-' * 15}\n     MENÚ\n{'-'*15}\n")
        print("1. Agregar una nota")
        print("2. Leer todas las notas")
        print("3. Buscar por una nota")
        print("4. Eliminar una nota")
        print("5. Salir")
        opt = input("\n[+] Escoge una opción: ")
        if opt == "1":
            contenido = input("\n[+] Contenido de la nota: ")
            gestor.agregar_nota(contenido)
        elif opt == "2":
            notas = gestor.leer_notas()
            os.system('cls' if os.name == 'nt' else 'clear')
            print(f"\nMostrando todas las notas almacenadas:\n")
            for i, nota in enumerate(notas):
                print(f"Nota {i+1}: {nota}")
        elif opt == "3":
            texto_busqueda = input("[+] Ingresa el texto a buscar en las notas: ")
            notas = gestor.buscar_nota(texto_busqueda)
            print(f"\n[+] Mostrando las notas que coinciden con la búsqueda: ")
            for i, nota in enumerate(notas):
                print(f"\n{i+1}: {nota}")
        elif opt == "4":
            index = int(input(f"\n[+] Introduce el número de la nota que quieres eliminar: "))
            gestor.eliminar_nota(index)
        elif opt == "5":
            break
        else:
            print("\n[!] La opción indicada es incorrecta.\n")
        input(f"\n[+] Presiona Enter para continuar......")
        os.system('cls' if os.name == 'nt' else 'clear')

if __name__ == "__main__":
    main()
```

```python
# gestor_notas.py
#!/usr/bin/env python3
import pickle
from notas import Nota

class GestorNotas:
    def __init__(self):
        self.archivo = "notas.pkl"
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
    
    def agregar_nota(self, contenido):
        self.notas.append(Nota(contenido))
        self.guardar_notas()
    
    def leer_notas(self):
        return self.notas
    
    def buscar_nota(self, texto):
        return [n for n in self.notas if texto.lower() in n.contenido.lower()]
    
    def eliminar_nota(self, index):
        if 0 <= index < len(self.notas):
            self.notas.pop(index)
            self.guardar_notas()
```

```python
# notas.py
#!/usr/bin/env python3
class Nota:
    def __init__(self, contenido):
        self.contenido = contenido
    def __str__(self):
        return f"{self.contenido}"
```
