------
# **Proyecto de gestión de biblioteca de libros**

En este emocionante proyecto inicial, desarrollaremos un sistema de gestión para una biblioteca utilizando Python pero para esto tenemos que tener claro la [[Programación Orientada a Objetos]] .

Este sistema nos permitirá agregar nuevos libros a la biblioteca y gestionar el préstamo de estos. A través de la creación de este sistema, aplicaremos de manera práctica los fundamentos de la Programación Orientada a Objetos (POO), haciendo especial énfasis en conceptos clave como herencia y polimorfismo. Diseñaremos clases para representar diferentes aspectos de una biblioteca, como libros, miembros y transacciones de préstamo.

Con este proyecto, no solo consolidaremos nuestra comprensión de POO, sino que también mejoraremos nuestras habilidades de programación en Python, creando un sistema funcional y útil.

En esta clase, continuamos avanzando en nuestro emocionante primer proyecto en Python, el desarrollo de un sistema de gestión de biblioteca. Esta fase del proyecto nos permitirá profundizar en las funcionalidades que hemos comenzado a construir. Nos enfocaremos en perfeccionar las características existentes y en añadir nuevas funcionalidades.

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

    def __str__(self): # Metodo especial para controlar la salida de datos por defecto de los objetos
        return f"Libro({self.id_libro}, {self.autor}, {self.nombre_libro})"

    def __repr__(self): # Como el metodo especial que nos permite representar algo cuando llamamos al objeto y no se ejecuta str.
        return self.__str__()

class Biblioteca:
    #Definiendo Constructor
    def __init__(self):
        self.libros = {} # Creamos un diccionario de libros con la estructura {id_libro: El objeto}
    
    def agregar_libro(self, libro): # self es nuestra clase bibioteca y libro es el objeto que recive
        if libro.id_libro not in self.libros: # Comprovamos si libro se encuentra en self.libros
            self.libros[libro.id_libro] = libro
        else:
            print(f"\\n[!] No se a podido agregar el libro {libro.nombre} con id: {libro.id_libro}")

    @property # con esto convertimos la función en propiedad y ya no la llamamos como función
    def mostrar_libros(self):
        return [libro for libro in self.libros.values() if not libro.esta_prestado] # Esto almacena los libros que tenga en prestado false

    @property # con esto convertimos la función en propiedad y ya no la llamamos como función
    def mostrar_libros_prestado(self):
        return [libro for libro in self.libros.values() if libro.esta_prestado] # ESto almacena los libros que tengan prestado true

    def prestar_libro(self, id_libro):
        if id_libro in self.libros and not self.libros[id_libro].esta_prestado: # Si el id del libro esta en la lista libros y ese libro sea false en prestado  entoces 
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\\n[+] No es posible prestar el libro con id: {id_libro}")

class BibliotecaInfantil(Biblioteca):

    def __init__(self):

        super().__init__() # Gracias a super llamamos a los atributos del constructor del que erada en este caso Bliblioteca y evitamos sobreescribir el constructor
        self.libros_para_ninos = {}

    def agregar_libro(self, libro, ninos): # En este punto sobreescribimos un metodo y luego llamamos con super al metodo principal que se hereda.
        super().agregar_libro(libro)
        self.libros_para_ninos[libro.id_libro] = ninos # Estamos agregando el libro a nuestro diccionario

    def prestar_libro(self, id_libro, nino):
        if id_libro in self.libros and self.libros_para_ninos[id_libro] == nino and not self.libros[id_libro].esta_prestado: # Si el id del libro esta en la lista libros y ese libro sea false en prestado  entoces 
            self.libros[id_libro].esta_prestado = True
        else:
            print(f"\\n[+] No es posible prestar el libro con id: {id_libro}")

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

    print(f"\\n[+] Libros en la biblioteca {biblioteca.mostrar_libros}") # No usamos los parentesis porque converitmos las funciones en propiedades
    print(f"\\n[+] Libros prestados: {biblioteca.mostrar_libros_prestado}") # No usamos los parentesis proque llamamos las funciones en propiedades

    print(f"\\n[+] Libros para niños: {biblioteca.mostrar_libros_estado_libros_para_ninos}") # No usamos los parentesis proque llamamos las funciones en propiedades

if __name__ == "__main__":
    main()
```
# Proyecto de gestión de animales en tienda

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

        print(f"\\n[!] El animal nunca a sido encontrado ")

def main():

    tienda = Tienda_Animales("Animal Happy")

    gato = Animal("Michi", "Gato")
    perro = Animal("firu", "Perro")

    tienda.agregar_animal(gato)
    tienda.agregar_animal(perro)

    tienda.mostrar_animales
    tienda.alimentar

    print(f"\\n[+] Mostrando animales alimentados\\n")

    tienda.mostrar_animales

    tienda.vender_animal("firu")

    print(f"\\n[+] Mostrando los animales restantes despues de la venta: \\n")

    tienda.mostrar_animales

if __name__ == "__main__":
    main()
```

# **Proyecto de administración de flota de vehículos**

En este proyecto de Python, nos embarcamos en el desafío de crear un sistema de administración para una flota de vehículos, aplicando nuevamente los principios de la Programación Orientada a Objetos. Este proyecto se centrará en el diseño y la implementación de dos clases interactivas, que trabajarán en conjunto para gestionar eficientemente una flota de vehículos.

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
            print(f"[+] El vehiculo con matricula {self.matricula} no esta disponible")

    def devolver(self):
        if not self.disponible:
            self.disponible = True
        else:
            print("\\n[+] Error en la matricual, el vehiculo ya se encuentra en la flota!!!!")

    def __str__(self):
        return f"Vehiculo(matricula={self.matricula}, modelo={self.modelo}, disponibilidad={self.disponible})"

class Flota:
    
    def __init__(self):
        self.vehiculos = [];

    
    def agregar_vehiculo(self, vehiculo):
        self.vehiculos.append(vehiculo)

    def __str__(self):
        return "\\n".join(str(vehiculo) for vehiculo in self.vehiculos)

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
    
    print(f"\\n[+] Flota inicial:\\n")
    print(flota)

    flota.alquilar_vehiculo("PDH4399")
    
    print(f"\\n Mostrando la flota despues de alquilar el SWM M-3: \\n")
    print(flota)

    print(f"\\n[+] Devolviendo el vehiculo: \\n")

    flota.devolver_vehiculo("PDH4399")
    print(flota)

    flota.devolver_vehiculo("PDH4399")

if __name__ == "__main__":
    main()

```

# **Proyecto de gestión de notas**

En este nuevo proyecto de Python, nos embarcamos en la construcción de un gestor de notas avanzado, aprovechando las técnicas de la Programación Orientada a Objetos (POO). Nuestro objetivo es crear un sistema interactivo que nos permita manejar eficientemente una variedad de notas.

El corazón de nuestro proyecto será un menú interactivo desde el cual podremos realizar múltiples acciones. Entre estas, incluiremos la capacidad de crear nuevas notas que serán almacenadas en disco, visualizar todas las notas existentes, buscar notas específicas por ciertos criterios y eliminar notas que ya no sean necesarias.

Para lograr esto, definiremos varias clases y métodos que colaborarán para manejar las diferentes operaciones relacionadas con las notas. Una parte crucial de nuestro proyecto será la implementación de módulos múltiples, lo que nos permitirá organizar mejor nuestro código y hacerlo más mantenible.

Además, exploraremos los conceptos de serialización y deserialización de datos utilizando Pickle. Esto nos permitirá guardar y recuperar eficientemente el estado de nuestras notas desde y hacia el disco, facilitando la persistencia de los datos a través de las sesiones de uso del gestor.

Este proyecto no solo fortalecerá nuestras habilidades en POO y manejo de datos, sino que también nos proporcionará experiencia práctica en la creación de aplicaciones Python que interactúan con el sistema de archivos y gestionan la información de manera eficiente.

En esta clase, concluiremos nuestro proyecto de gestor de notas en Python utilizando Pickle. Tras haber sentado las bases en nuestra sesión anterior, ahora es momento de integrar y finalizar todos los componentes de nuestro sistema. Vamos a dedicar esta clase a revisar y perfeccionar las clases principales, asegurándonos de que todas las funcionalidades clave, como crear, visualizar, buscar y eliminar notas, funcionen de manera fluida.

Un enfoque importante será la implementación efectiva de la serialización y deserialización con Pickle. Esta técnica es crucial para la persistencia de las notas en el disco, permitiendo que se guarden de forma segura y se recuperen en futuras sesiones del programa.

Al concluir esta clase, habremos completado un gestor de notas funcional y eficiente. Este proyecto no solo refuerza nuestras habilidades en programación orientada a objetos y manejo de datos, sino que también nos proporciona experiencia práctica en la creación de aplicaciones Python interactivas y robustas.

Archivo [main.py](http://main.py)

```python
#!/usr/bin/env python3

import os # Os nos ayuda a ejecutar comandos
from gestor_notas import GestorNotas # Esto importa las clases de otros archivos

def main():

    gestor = GestorNotas()

    while True:

        print(f"\\n{'-' * 15}\\n     MENÚ\\n{'-'*15}\\n")
        print("1.Agregar una nota")
        print("2.Leer todas las notas")
        print("3.Buscar por una nota")
        print("4.Elimnar una nota")
        print("5.Salir")

        opt = input("\\n[+] Escoge una opción: ")

        if opt == "1":
            contenido = input("\\n[+] Contenido de la nota: ")
            gestor.agregar_nota(contenido)

        elif opt == "2":

            notas = gestor.leer_notas()

            os.system('cls' if os.name == 'nt' else 'clear') # os.name nos devuelve el tipo de sistema: nt(windows) y posix(linux)
            print(f"\\nMostarndo todas las notas almacenadas:\\n")

            for i, nota in enumerate(notas):
                print(f"Nota {i+1}: {nota}")
        elif opt == "3":
            texto_busquda = input("[+] Ingresa el texto a buscar en las notas: ")
            notas = gestor.buscar_nota(texto_busquda)

            print(f"\\n[+] Mostrando las Notas que coinciden con la busqueda: ")
            
            for i, nota in enumerate(notas):
                print(f"\\n{i+1}: {nota}")

        elif opt == "4":
            index = int(input(f"\\n[+] Introduce el numero de la nota que quieres eliminar: "))
            gestor.eliminar_nota(index)

        elif opt == "5":
            break
        else:
            print("\\n[!] La opción indicada es incorrecta\\n")

        input(f"\\n[+] Presiona enter para continuar......")
        
        os.system('cls' if os.name == 'nt' else 'clear') # os.name nos devuelve el tipo de sistema: nt(windows) y posix(linux)

if __name__ == "__main__":
    main()
```

Archivo gestor_notas.py

```python
#!/usr/bin/env python3

import os # Os nos ayuda a ejecutar comandos
from gestor_notas import GestorNotas # Esto importa las clases de otros archivos

def main():

    gestor = GestorNotas()

    while True:

        print(f"\\n{'-' * 15}\\n     MENÚ\\n{'-'*15}\\n")
        print("1.Agregar una nota")
        print("2.Leer todas las notas")
        print("3.Buscar por una nota")
        print("4.Elimnar una nota")
        print("5.Salir")

        opt = input("\\n[+] Escoge una opción: ")

        if opt == "1":
            contenido = input("\\n[+] Contenido de la nota: ")
            gestor.agregar_nota(contenido)

        elif opt == "2":

            notas = gestor.leer_notas()

            os.system('cls' if os.name == 'nt' else 'clear') # os.name nos devuelve el tipo de sistema: nt(windows) y posix(linux)
            print(f"\\nMostarndo todas las notas almacenadas:\\n")

            for i, nota in enumerate(notas):
                print(f"Nota {i+1}: {nota}")
        elif opt == "3":
            texto_busquda = input("[+] Ingresa el texto a buscar en las notas: ")
            notas = gestor.buscar_nota(texto_busquda)

            print(f"\\n[+] Mostrando las Notas que coinciden con la busqueda: ")
            
            for i, nota in enumerate(notas):
                print(f"\\n{i+1}: {nota}")

        elif opt == "4":
            index = int(input(f"\\n[+] Introduce el numero de la nota que quieres eliminar: "))
            gestor.eliminar_nota(index)

        elif opt == "5":
            break
        else:
            print("\\n[!] La opción indicada es incorrecta\\n")

        input(f"\\n[+] Presiona enter para continuar......")
        
        os.system('cls' if os.name == 'nt' else 'clear') # os.name nos devuelve el tipo de sistema: nt(windows) y posix(linux)

if __name__ == "__main__":
    main()
```

Archivo [notas.py](http://notas.py)

```python
#!/usr/bin/env python3

class Nota:

    def __init__(self, contenido):
        self.contenido = contenido

    def __str__(self):
        return f"{self.contenido}"
```