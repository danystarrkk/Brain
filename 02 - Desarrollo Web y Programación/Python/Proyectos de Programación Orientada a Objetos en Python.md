---
aliases:
  - Proyectos Python POO
  - Gestión de Biblioteca Python
  - Gestión de Animales Python
  - Gestión de Flota Python
  - Gestor de Notas Python
tags:
  - programacion/python
estado: 🟢 Terminado
---
# Proyecto de Gestión de Biblioteca de Libros

Este sistema nos permitirá agregar nuevos libros a la biblioteca y gestionar el préstamo de estos. A través de la creación de este sistema, aplicaremos de manera práctica los fundamentos de la Programación Orientada a Objetos (POO), haciendo especial énfasis en conceptos clave como herencia y polimorfismo. Diseñaremos clases para representar diferentes aspectos de una biblioteca, como libros, miembros y transacciones de préstamo.

Con este proyecto, no solo consolidaremos nuestra comprensión de POO, sino que también mejoraremos nuestras habilidades de programación en Python, creando un sistema funcional y útil.

En esta clase, continuamos avanzando en nuestro emocionante primer proyecto en Python: el desarrollo de un sistema de gestión de biblioteca. Esta fase del proyecto nos permitirá profundizar en las funcionalidades que hemos comenzado a construir. Nos enfocaremos en perfeccionar las características existentes y en añadir nuevas funcionalidades.

Al final de esta clase, nuestro objetivo es concluir el proyecto. Nos aseguraremos de que todos los conceptos clave de la Programación Orientada a Objetos, como la herencia, el polimorfismo y la utilización de clases, estén integrados de manera efectiva. Esta clase no solo consolida lo aprendido hasta ahora, sino que también culmina en la finalización de un sistema funcional y bien estructurado.

```python
#!/usr/bin/env python3

class Libro:
    # Constructor de la clase
    def __init__(self, id_libro, autor, nombre_libro):
        self.id_libro = id_libro
        self.nombre_libro = nombre_libro
        self.autor = autor
        self.esta_prestado = False

    def __str__(self): # Método especial para controlar la salida de datos por defecto de los objetos
        return f"Libro({self.id_libro}, {self.autor}, {self.nombre_libro})"

    def __repr__(self): # Método especial que nos permite representar un objeto cuando se llama directamente o en colecciones, y no se ejecuta __str__.
        return self.__str__()

class Biblioteca:
    #Definiendo Constructor
    def __init__(self):
        self.libros = {} # Creamos un diccionario de libros con la estructura {id_libro: objeto_libro}
    
    def agregar_libro(self, libro): # 'self' es una referencia a la instancia de la clase Biblioteca, y 'libro' es el objeto Libro que se recibe.
        if libro.id_libro not in self.libros: # Comprobamos si el libro se encuentra en self.libros
            self.libros[libro.id_libro] = libro
        else:
            print(f"\n[!] No se ha podido agregar el libro {libro.nombre_libro} con ID: {libro.id_libro}")

    @property # Con esto convertimos la función en una propiedad, permitiendo acceder a ella como un atributo sin paréntesis.
    def mostrar_libros(self):
        return [libro for libro in self.libros.values() if not libro.esta_prestado] # Esto almacena los libros cuyo estado 'esta_prestado' es False.

    @property # Con esto convertimos la función en una propiedad, permitiendo acceder a ella como un atributo sin paréntesis.
    def mostrar_libros_prestado(self):
        return [libro for libro in self.libros.values() if libro.esta_prestado] # Esto almacena los libros cuyo estado 'esta_prestado' es True.

    def prestar_libro(self, id_libro):
        if id_libro in self.libros and not self.libros[id_libro].esta_prestado: # Si el ID del libro está en el diccionario 'self.libros' y el libro no está prestado, entonces...
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\n[+] No es posible prestar el libro con ID: {id_libro}")

class BibliotecaInfantil(Biblioteca):

    def __init__(self):

        super().__init__() # Gracias a 'super()', llamamos al constructor de la clase padre (Biblioteca) para inicializar sus atributos y evitar sobrescribir el constructor.
        self.libros_para_ninos = {}

    def agregar_libro(self, libro, ninos): # En este punto sobrescribimos un método y luego llamamos con 'super()' al método principal que se hereda.
        super().agregar_libro(libro)
        self.libros_para_ninos[libro.id_libro] = ninos # Estamos agregando el libro a nuestro diccionario, asociándolo con un indicador para niños.

    def prestar_libro(self, id_libro, nino):
        if id_libro in self.libros and self.libros_para_ninos[id_libro] == nino and not self.libros[id_libro].esta_prestado: # Si el ID del libro está en 'self.libros', coincide con el indicador 'nino' y no está prestado, entonces...
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

    biblioteca.agregar_libro(libro1, ninos = False)
    biblioteca.agregar_libro(libro2, ninos = True)

    biblioteca.prestar_libro(1, nino = True)

    print(f"\n[+] Libros en la biblioteca {biblioteca.mostrar_libros}") # No usamos los paréntesis porque convertimos las funciones en propiedades.
    print(f"\n[+] Libros prestados: {biblioteca.mostrar_libros_prestado}") # No usamos los paréntesis porque llamamos a las funciones como propiedades.

    print(f"\n[+] Libros para niños: {biblioteca.mostrar_libros_estado_libros_para_ninos}") # No usamos los paréntesis porque llamamos a las funciones como propiedades.

if __name__ == "__main__":
    main()
```

> [!info] Métodos `__str__` y `__repr__`
> - `__str__`: Proporciona una representación legible del objeto, destinada a ser consumida por humanos (ej. `print()`).
> - `__repr__`: Proporciona una representación inequívoca del objeto, destinada a ser consumida por desarrolladores (ej. depuración, `eval()`). Si `__str__` no está definido, `__repr__` se usa como fallback para `print()`. Es buena práctica que `__repr__` devuelva una cadena que, si es posible, pueda usarse para recrear el objeto.

> [!tip] Decorador `@property`
> El decorador `@property` permite acceder a un método como si fuera un atributo. Esto es útil para encapsular la lógica de acceso a datos, permitiendo realizar cálculos o validaciones internas sin cambiar la forma en que se accede al dato desde fuera de la clase. Mejora la legibilidad y el diseño de la API de la clase.

> [!info] Uso de `super().__init__()` en Herencia
> La función `super()` en Python se utiliza para llamar a métodos de la clase padre (superclase) desde una subclase. En el constructor `__init__` de `BibliotecaInfantil`, `super().__init__()` asegura que el constructor de la clase `Biblioteca` se ejecute, inicializando `self.libros = {}` antes de que `BibliotecaInfantil` añada sus propios atributos (`self.libros_para_ninos`). Esto es crucial para una correcta inicialización y para evitar la duplicación de código.

> [!tip] Polimorfismo en `agregar_libro` y `prestar_libro`
> La clase `BibliotecaInfantil` demuestra polimorfismo al sobrescribir los métodos `agregar_libro` y `prestar_libro` de su clase padre `Biblioteca`. Aunque ambos métodos tienen el mismo nombre, su comportamiento se adapta para incluir la lógica específica de los libros para niños, manteniendo una interfaz común pero con implementaciones distintas.

# Proyecto de Gestión de Animales en Tienda

En este proyecto, continuamos explorando la Programación Orientada a Objetos (POO), esta vez con un enfoque diferente. Sin emplear herencia ni polimorfismo, nos centraremos en la creación de un sistema de gestión de animales en una tienda.

El proyecto involucrará el diseño y la implementación de dos clases principales que trabajarán de manera sinérgica. Estas clases nos permitirán realizar una variedad de operaciones esenciales en la gestión de una tienda de animales. Entre las funcionalidades que desarrollaremos, se incluyen métodos para agregar nuevos animales al inventario, mostrar los animales disponibles, alimentar a los animales y gestionar su venta.

A través de este proyecto, no solo afianzaremos nuestras habilidades en POO, sino que también demostraremos cómo diferentes clases pueden colaborar de forma efectiva para lograr objetivos comunes. Este sistema nos dará una perspectiva práctica y realista sobre cómo la POO puede ser utilizada para resolver problemas y gestionar información en un contexto de negocios.

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

> [!info] Composición de Objetos
> En este proyecto, la clase `Tienda_Animales` utiliza la composición al contener una lista de objetos `Animal`. Esto significa que una `Tienda_Animales` *tiene* `Animales`. La composición es una forma poderosa de construir objetos complejos a partir de objetos más simples, promoviendo la reutilización de código y una clara separación de responsabilidades, sin necesidad de herencia.

> [!tip] Representación de Objetos con `__str__`
> El método `__str__` en la clase `Animal` es fundamental para proporcionar una representación legible del estado de cada animal. Cuando se imprime un objeto `Animal` (por ejemplo, `print(animal)`), Python llama automáticamente a este método, lo que facilita la depuración y la visualización del inventario de la tienda.

# Proyecto de Administración de Flota de Vehículos

En este proyecto de Python, nos embarcamos en el desafío de crear un sistema de administración para una flota de vehículos, aplicando nuevamente los principios de la Programación Orientada a Objetos. Este proyecto se centrará en el diseño y la implementación de dos clases interactivas que trabajarán en conjunto para gestionar eficientemente una flota de vehículos.

Dentro de las capacidades de nuestro sistema, incluiremos métodos para agregar nuevos vehículos a la flota, lo que nos permitirá expandir y diversificar nuestras opciones de alquiler. Además, desarrollaremos funcionalidades para el alquiler de vehículos, brindando a los clientes la posibilidad de elegir entre diferentes tipos de vehículos según sus necesidades.

Otra característica clave será la gestión de la devolución de vehículos. Este método facilitará el proceso de retorno de los vehículos alquilados, asegurando un control eficiente y organizado de la flota.

Este proyecto no solo fortalecerá nuestra comprensión y habilidad en POO, sino que también nos proporcionará valiosas lecciones sobre cómo las clases pueden interactuar para administrar operaciones complejas en un entorno empresarial real.

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

> [!info] Gestión de Estado de Objetos
> La clase `Vehiculo` gestiona su propio estado a través del atributo `self.disponible`. Los métodos `alquilar()` y `devolver()` modifican este estado, lo que es un ejemplo clave de cómo los objetos encapsulan datos y comportamiento. Esta encapsulación permite que cada vehículo sea una entidad autónoma que sabe si está disponible o no, simplificando la lógica de la `Flota`.

> [!tip] Método `__str__` para Colecciones
> El método `__str__` en la clase `Flota` es un excelente ejemplo de cómo representar una colección de objetos de manera legible. Al usar `"\n".join(str(vehiculo) for vehiculo in self.vehiculos)`, se itera sobre cada `Vehiculo` en la lista, se llama a su propio método `__str__` y se unen todas las representaciones con saltos de línea, creando una salida clara de toda la flota.

# Proyecto de Gestión de Notas

En este nuevo proyecto de Python, nos embarcamos en la construcción de un gestor de notas avanzado, aprovechando las técnicas de la Programación Orientada a Objetos (POO). Nuestro objetivo es crear un sistema interactivo que nos permita manejar eficientemente una variedad de notas.

El corazón de nuestro proyecto será un menú interactivo desde el cual podremos realizar múltiples acciones. Entre estas, incluiremos la capacidad de crear nuevas notas que serán almacenadas en disco, visualizar todas las notas existentes, buscar notas específicas por ciertos criterios y eliminar notas que ya no sean necesarias.

Para lograr esto, definiremos varias clases y métodos que colaborarán para manejar las diferentes operaciones relacionadas con las notas. Una parte crucial de nuestro proyecto será la implementación de módulos múltiples, lo que nos permitirá organizar mejor nuestro código y hacerlo más mantenible.

Además, exploraremos los conceptos de serialización y deserialización de datos utilizando Pickle. Esto nos permitirá guardar y recuperar eficientemente el estado de nuestras notas desde y hacia el disco, facilitando la persistencia de los datos a través de las sesiones de uso del gestor.

Este proyecto no solo fortalecerá nuestras habilidades en POO y manejo de datos, sino que también nos proporcionará experiencia práctica en la creación de aplicaciones Python que interactúan con el sistema de archivos y gestionan la información de manera eficiente.

En esta clase, concluiremos nuestro proyecto de gestor de notas en Python utilizando Pickle. Tras haber sentado las bases en nuestra sesión anterior, ahora es momento de integrar y finalizar todos los componentes de nuestro sistema. Vamos a dedicar esta clase a revisar y perfeccionar las clases principales, asegurándonos de que todas las funcionalidades clave, como crear, visualizar, buscar y eliminar notas, funcionen de manera fluida. Un enfoque importante será la implementación efectiva de la serialización y deserialización con Pickle. Esta técnica es crucial para la persistencia de las notas en el disco, permitiendo que se guarden de forma segura y se recuperen en futuras sesiones del programa. Al concluir esta clase, habremos completado un gestor de notas funcional y eficiente. Este proyecto no solo refuerza nuestras habilidades en programación orientada a objetos y manejo de datos, sino que también nos proporciona experiencia práctica en la creación de aplicaciones Python interactivas y robustas.

Archivo [main.py](http://main.py)

```python
#!/usr/bin/env python3

import os # El módulo 'os' nos ayuda a interactuar con el sistema operativo, por ejemplo, para ejecutar comandos.
from gestor_notas import GestorNotas # Esto importa la clase 'GestorNotas' desde el archivo 'gestor_notas.py'.

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

            os.system('cls' if os.name == 'nt' else 'clear') # 'os.name' devuelve el tipo de sistema operativo: 'nt' para Windows y 'posix' para sistemas tipo Unix (Linux, macOS).
            print(f"\nMostrando todas las notas almacenadas:\n")

            for i, nota in enumerate(notas):
                print(f"Nota {i+1}: {nota}")
        elif opt == "3":
            texto_busquda = input("[+] Ingresa el texto a buscar en las notas: ")
            notas = gestor.buscar_nota(texto_busquda)

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
        
        os.system('cls' if os.name == 'nt' else 'clear') # 'os.name' devuelve el tipo de sistema operativo: 'nt' para Windows y 'posix' para sistemas tipo Unix (Linux, macOS).

if __name__ == "__main__":
    main()
```

> [!info] Limpieza de Consola Multiplataforma
> La línea `os.system('cls' if os.name == 'nt' else 'clear')` es un truco común en Python para limpiar la pantalla de la consola de forma multiplataforma. `os.name` devuelve `'nt'` para Windows y `'posix'` para sistemas Unix-like (Linux, macOS). Así, ejecuta `cls` en Windows y `clear` en otros sistemas, proporcionando una experiencia de usuario más limpia.

> [!tip] Modularización del Código
> La separación del código en archivos como `main.py`, `gestor_notas.py` y `notas.py` es un ejemplo de modularización. Esto mejora la organización, la legibilidad y la mantenibilidad del proyecto. Cada módulo tiene una responsabilidad específica, lo que facilita la colaboración y la depuración.

Archivo gestor_notas.py

```python
#!/usr/bin/env python3

import os # El módulo 'os' nos ayuda a interactuar con el sistema operativo, por ejemplo, para ejecutar comandos.
from gestor_notas import GestorNotas # Esto importa la clase 'GestorNotas' desde el archivo 'gestor_notas.py'.

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

            os.system('cls' if os.name == 'nt' else 'clear') # 'os.name' devuelve el tipo de sistema operativo: 'nt' para Windows y 'posix' para sistemas tipo Unix (Linux, macOS).
            print(f"\nMostrando todas las notas almacenadas:\n")

            for i, nota in enumerate(notas):
                print(f"Nota {i+1}: {nota}")
        elif opt == "3":
            texto_busquda = input("[+] Ingresa el texto a buscar en las notas: ")
            notas = gestor.buscar_nota(texto_busquda)

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
        
        os.system('cls' if os.name == 'nt' else 'clear') # 'os.name' devuelve el tipo de sistema operativo: 'nt' para Windows y 'posix' para sistemas tipo Unix (Linux, macOS).

if __name__ == "__main__":
    main()
```

> [!warning] Contenido Duplicado en `gestor_notas.py`
> El código proporcionado para `gestor_notas.py` es idéntico al de `main.py`. En una implementación real, `gestor_notas.py` debería contener la definición de la clase `GestorNotas` (y su lógica de persistencia con Pickle), mientras que `main.py` se encargaría de la interfaz de usuario y la interacción con el objeto `GestorNotas` importado.

> [!info] Serialización con Pickle
> Pickle es un módulo de Python que implementa protocolos binarios para serializar y deserializar objetos Python. Esto permite convertir un objeto Python en una secuencia de bytes (serialización) para almacenarlo en un archivo o transmitirlo a través de la red, y luego reconstruir el objeto original a partir de esa secuencia de bytes (deserialización). Es ideal para la persistencia de datos de objetos complejos en aplicaciones Python.

Archivo `notas.py`

```python
#!/usr/bin/env python3

class Nota:

    def __init__(self, contenido):
        self.contenido = contenido

    def __str__(self): # Define la representación en cadena del objeto Nota.
        return f"{self.contenido}"
```