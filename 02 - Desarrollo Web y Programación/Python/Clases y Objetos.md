---
aliases:
  - POO en Python
  - Programación Orientada a Objetos
tags:
  - programacion/python
estado: 🟢 Terminado
---
# Clases y Objetos

La Programación Orientada a Objetos (POO) es un paradigma de programación que utiliza objetos y clases en su enfoque central. Es una manera de estructurar y organizar el código que refleja cómo los desarrolladores piensan sobre el mundo real y las entidades dentro de él.

> [!info]
> La POO se basa en cuatro pilares fundamentales: **Encapsulamiento**, **Herencia**, **Polimorfismo** y **Abstracción**. Estos principios buscan mejorar la modularidad, la reutilización y la mantenibilidad del código.

## Clases

Las clases son los fundamentos de la POO. Actúan como plantillas para la creación de objetos y definen atributos y comportamientos que los objetos creados a partir de ellas tendrán. En Python, una clase se define con la palabra clave `class` y proporciona la estructura inicial para todo objeto que se derive de ella.

**Instancias de Clase y Objetos**

Un objeto es una instancia de una clase. Cada vez que se crea un objeto, se está creando una instancia que tiene su propio espacio de memoria y conjunto de valores para los atributos definidos por su clase. Los objetos encapsulan datos y funciones juntos en una entidad discreta.

## Métodos de Clase

Los métodos de clase son funciones que se definen dentro de una clase y solo pueden ser llamados por las instancias de esa clase. Estos métodos son el mecanismo principal para interactuar con los objetos, permitiéndoles realizar operaciones o acciones, modificar su estado o incluso interactuar con otros objetos.

En esta clase, te proporcionaremos las herramientas y el entendimiento necesario para comenzar a diseñar y desarrollar tus propias clases y a crear instancias de esas clases en objetos funcionales. Aprenderemos cómo los métodos de clase operan y cómo puedes utilizarlos para dar vida al comportamiento de tus objetos en Python. Este conocimiento será esencial a medida que continúes aprendiendo y aplicando los principios de la POO en proyectos más complejos.

```python
#!/usr/bin/env python3

class Persona: # Definiendo una clase

    def __init__(self, nombre, edad): # Definiendo el constructor Persona.__init__(marcelo, nombre(Daniel), edad(20))

        # Estamos inicializando los valores
        self.nombre = nombre # daniel.nombre = "Daniel"
        self.edad = edad # daniel.edad = 20

    def saludo(self): # Definiendo una función cualquiera
        return f"Hola soy {self.nombre} y tengo {self.edad} años"

# Función Principal
def main():

    daniel = Persona("Daniel", 20) # Estamos creando un objeto, en este caso daniel
    dylan = Persona("Dylan", 9)   # Nuevo objeto dylan
    print(daniel.saludo()) # Estamos llamando al objeto daniel el cual cuenta con la función saludo
    print(dylan.saludo()) # Estamos llamando al objeto dylan

if __name__ == "__main__":
    main()

```

> [!info]
> El método `__init__` es el constructor de la clase. Se invoca automáticamente cuando se crea una nueva instancia de la clase. El primer parámetro, `self`, es una referencia a la instancia recién creada, permitiendo inicializar sus atributos.

Otro ejemplo puede ser el siguiente:

```python
#!/usr/bin/env python3

juegos = ["Super Mario Bros", "Zelda: Breath of the Wild", "Cyberpunk 2077", "Final Fantasy VII"]
tope = 100 

# Géneros

# El siguiente comando 'catnp ejercicio2.py' fue omitido ya que no es parte del código Python.

class Animal: # Creamos la clase

    def __init__(self, tipo, nombre, edad): # Definimos el constructor
        # Inicializamos los Datos
        self.tipo = tipo
        self.nombre = nombre
        self.edad = edad
    
    def datos(self): # Función cualquiera
        return f"Su nombre es {self.nombre} es un {self.tipo} y tiene {self.edad} años"

# Función Principal
def main():

    gato = Animal("Felino", "Flash", 5)
    gallina = Animal("Ave", "Turuleca", 2)
    print(gato.datos())
    print(gallina.datos())

if __name__ == "__main__":
    main()
```

Vamos a ver un ejemplo un poco más complicado para terminar de comprender las clases:

```python
#!/usr/bin/env python3

class CuentaBancaria:

    def __init__(self, numero_cuenta, nombre, dinero=0): # Cuando inicializamos con un valor predeterminado, como en este caso, el valor 0 solo se utiliza si no se proporciona ningún argumento a la clase.
        self.numero_cuenta = numero_cuenta
        self.nombre = nombre
        self.dinero = dinero

    def depositar(self, dinero):
        print(f"Depositando {dinero}$")
        self.dinero += dinero
        print(f"El dinero actual de la cuenta es de: {self.dinero}")

    def retirar(self, dinero):
        if self.dinero >= dinero:
            print(f"Saldo actual {self.dinero}$")
            self.dinero-=dinero
            print(f"Ha retirado {dinero}$ su saldo actual es de {self.dinero}$")
        else:
            print(f"No tiene ese valor en la cuenta")

# Función Principal
def main():

    manolo = CuentaBancaria("123413", "Manolo Zambrano", 10000)
    manolo.depositar(500)
    manolo.retirar(10500)

    daniel = CuentaBancaria("32452345", "Daniel Estrella", 5000)
    daniel.retirar(500)

if __name__ == "__main__":
    main()
```

Los siguientes ejercicios continúan el uso de decoradores y funciones especiales mediante las cuales podemos cumplir un objetivo:

```python
#!/usr/bin/env python3

class Rectangulo:
    def __init__(self, ancho, alto):
        self.ancho = ancho
        self.alto = alto

    @property # Este es un decorador el cual nos permite convertir el resultado de un método en una propiedad, no en una función.
    def area(self):
        return self.ancho * self.alto

    def __str__(self): # Este es un método especial que se ejecuta por defecto si no se llama a ningún método específico para la representación en cadena del objeto.
        return f"\n[+] Propiedades del rectángulo: [Ancho: {self.ancho}][Alto: {self.alto}]"

    def __eq__(self, otro): # Método especial que nos permite comprobar la igualdad de valores entre dos objetos.

        return self.ancho == otro.ancho and self.alto == otro.alto
        

def main():
    area = Rectangulo(20, 80)
    area1 = Rectangulo(20, 80)
    print(f"El área del Rectángulo es {area.area}")
    print(area) # Al no pasarle ningún método específico, se ejecuta el método __str__
    print(f"Los rectángulos son iguales? {area == area1 }") # Esta comparación sin el método __eq__ no sería válida.

if __name__ == "__main__":
    main()
```

> [!tip]
> El decorador `@property` permite acceder a un método como si fuera un atributo, lo que es útil para calcular valores dinámicamente o para implementar getters sin cambiar la forma en que se accede al atributo. El método `__str__` proporciona una representación legible del objeto, mientras que `__eq__` define cómo se comparan dos objetos para la igualdad (`==`).

```python
#!/usr/bin/env python3

class Libro:
    
    bestseller_value = 5000 # Esta es una variable de clase (atributo de clase).
    iva=0.21

    def __init__(self, titulo, autor, precio):
        self.titulo = titulo
        self.autor = autor
        self.precio = precio

    @staticmethod # Este es un decorador que nos permite operar sin la necesidad de una instancia del objeto.
    def es_bestseller(ventas):
        return ventas > Libro.bestseller_value # Esta es la manera correcta de usar un atributo de clase.

    @classmethod # Este decorador hace que el método reciba a su propia clase como primer argumento.
    def inclusion_iva(cls,precio):
        print(f"\n[+] El precio con el IVA incluido es de {precio*cls.iva + precio}") # Esto es útil cuando se trabaja con jerarquías de clases y herencia.

def main():
    mi_libro = Libro("48 Leyes", "Daniel", 16)
    print(mi_libro.es_bestseller(10000))
    mi_libro.inclusion_iva(100)

if __name__ == "__main__":
    main()
  
```

# Métodos Estáticos y Métodos de Clase

Los métodos estáticos y los métodos de clase son dos herramientas poderosas en la programación orientada a objetos en Python, que ofrecen flexibilidad en cómo se puede acceder y utilizar la funcionalidad asociada con una clase.

## Métodos de Clase

Se definen con el decorador `@classmethod`, lo que les permite tomar la clase como primer argumento, generalmente nombrada `cls`. Este acceso a la clase permite que los métodos de clase interactúen con la estructura de la clase en sí, como modificar atributos de clase que afectarán a todas las instancias. Se utilizan para tareas que requieren conocimiento del estado global de la clase, como la construcción de instancias de maneras específicas, también conocidos como métodos *factory*.

## Métodos Estáticos

Se definen con el decorador `@staticmethod` y no reciben un argumento implícito de referencia a la clase o instancia. Son similares a las funciones regulares definidas dentro del cuerpo de una clase. Se utilizan para funciones que, aunque conceptualmente pertenecen a la clase debido a la relevancia temática, no necesitan acceder a ningún dato específico de la clase o instancia. Proporcionan una manera de encapsular la funcionalidad dentro de una clase, manteniendo la cohesión y la organización del código.

Ambos métodos contribuyen a un diseño de software más limpio y modular, permitiendo una clara separación entre la funcionalidad que opera con respecto a la clase en su totalidad y la funcionalidad que es independiente de las instancias de clase y de la clase misma. La elección entre utilizar un método de clase o un método estático a menudo depende del requisito específico de acceso o no a la clase o a sus instancias.

```python
#!/usr/bin/env python3

class Calculadora:

    @staticmethod
    def suma(num1, num2):
        return num1 + num2

    @staticmethod
    def resta(num1, num2):
        return num1 - num2

    @staticmethod
    def multiplicacion(num1, num2):
        return num1 * num2

    @staticmethod
    def dividir(num1, num2):
        if num2 != 0:
            return num1 / num2
        else:
            return "El divisor no puede ser 0"

def main():
    print(Calculadora.suma(2, 8))
    print(Calculadora.resta(8, 2))
    print(Calculadora.multiplicacion(8, 2))
    print(Calculadora.dividir(8, 2))

if __name__ == "__main__":
    main()
```

> [!info]
> Los métodos estáticos son ideales para funciones de utilidad que están lógicamente relacionadas con una clase, pero que no necesitan acceder a los datos de la instancia (`self`) ni a los datos de la clase (`cls`). Se invocan directamente desde la clase, como `Calculadora.suma(2, 8)`.

```python
#!/usr/bin/env python3

class Automovil:

    def __init__(self, marca, modelo):
        self.marca = marca
        self.modelo = modelo

    @classmethod
    def deportivos(cls, marca):
        return cls(marca, "Deportivo")

    @classmethod
    def sedan(cls, marca):
        return cls(marca, "Sedan")

    def __str__(self):
        return f"La marca {self.marca} es un modelo {self.modelo}"

def main():
    deportivo = Automovil.deportivos("Ferrari")
    sedan = Automovil.sedan("Toyota")
    print(deportivo)
    print(sedan)

if __name__ == "__main__":
    main()
```

> [!info]
> Los métodos de clase (`@classmethod`) reciben la clase (`cls`) como primer argumento. Son comúnmente utilizados como "constructores alternativos" o "métodos factory" para crear instancias de la clase de diferentes maneras, como se ve en `deportivos()` y `sedan()`.

# Uso de self

El uso de `self` es uno de los aspectos más fundamentales y a la vez confusos para quienes se adentran en la Programación Orientada a Objetos (POO) en Python. Este identificador es crucial para entender cómo Python maneja los métodos y atributos dentro de sus clases y objetos.

## Definición de 'self'

A nivel conceptual, `self` es una referencia al objeto actual dentro de la clase. Es el primer parámetro que se pasa a cualquier método de una clase en Python. A través de `self`, un método puede acceder y manipular los atributos del objeto y llamar a otros métodos dentro del mismo objeto.

## Uso de 'self'

Cuando se crea una nueva instancia de una clase, Python pasa automáticamente la instancia recién creada como el primer argumento al método `__init__` y a otros métodos definidos en la clase que tienen `self` como su primer parámetro. Esto es lo que permite que un método opere con datos específicos del objeto y no con datos de la clase en general o de otras instancias de la clase.

## Importancia de 'self'

El concepto de `self` es importante en la POO ya que asegura que los métodos y atributos se apliquen al objeto correcto. Sin `self`, no podríamos diferenciar entre las operaciones y datos de diferentes instancias de una clase.

En esta clase, nos enfocaremos en comprender a fondo cómo y por qué `self` es usado en Python, explorando su papel en la interacción con las instancias de la clase. Desarrollaremos una comprensión clara de cómo `self` permite que las clases en Python sean intuitivas y eficientes, manteniendo un estado consistente a través de las operaciones del objeto. Este conocimiento es esencial para trabajar con clases y objetos de manera efectiva y aprovechar la potencia de la POO.

# Herencia y Polimorfismo

La herencia y el polimorfismo son conceptos fundamentales en la programación orientada a objetos que permiten la creación de una estructura de clases flexible y reutilizable.

## Herencia

Es un principio de la POO que permite a una clase heredar atributos y métodos de otra clase, conocida como su clase base o superclase. La herencia facilita la reutilización de código y la creación de una jerarquía de clases. Las subclases heredan las características de la superclase, lo que permite que se especialicen o modifiquen comportamientos existentes.

## Polimorfismo

Este concepto se refiere a la habilidad de objetos de diferentes clases de ser tratados como instancias de una clase común. El polimorfismo permite que una función o método interactúe con objetos de diferentes clases y los trate como si fueran del mismo tipo, siempre y cuando compartan la misma interfaz o método. Esto significa que el mismo método puede comportarse de manera diferente en distintas clases, un concepto conocido como sobrecarga de métodos.

Ambos, la herencia y el polimorfismo, son piedras angulares de la POO y son ampliamente utilizados para diseñar sistemas que son fácilmente extensibles y mantenibles.

En esta clase, exploraremos cómo implementar herencia en Python y cómo se puede aprovechar el polimorfismo para escribir código más general y potente. Estos conceptos nos ayudarán a entender mejor cómo construir jerarquías de clases y cómo los diferentes objetos pueden interactuar entre sí de manera flexible.

```python
#!/usr/bin/env python3

class Animal:

    def __init__(self, nombre):
        self.nombre = nombre

    def hablar(self):
        
        raise NotImplementedError("Las subclases deben implementar este método") # Este error se ejecuta cuando llamamos directamente a la función 'hablar' de la clase base 'Animal', la cual no cuenta con una implementación concreta.

class Gato(Animal): # Estamos creando una clase que recibe herencia de Animal Gato(Animal(gato, nombre))

    def hablar(self):
        return f"!Miau!"

class Perro(Animal):

    def hablar(self):
        return f"!Guau!"

def hacer_hablar(objeto):

    print(f"{objeto.nombre} dice {objeto.hablar()}") # Esto es un ejemplo de polimorfismo, de cómo un método se comporta de forma distinta para diferentes clases.

def main():

    gato = Gato("Firulais")
    perro = Perro("Lulú")

    hacer_hablar(gato)
    hacer_hablar(perro)

if __name__ == "__main__":
    main()
```

> [!info]
> La herencia permite a `Gato` y `Perro` reutilizar el constructor de `Animal`. El polimorfismo se demuestra con la función `hacer_hablar`, que puede aceptar cualquier objeto que tenga un método `hablar`, sin importar su tipo específico, y el comportamiento de `hablar` se adapta a la subclase.

```python
#!/usr/bin/env python3

class Automovil:

    def __init__(self, marca, modelo):
        self.marca = marca
        self.modelo = modelo

    def describir(self):
        return f"Vehículo: {self.marca} {self.modelo}"

class Coche(Automovil): # Concepto de herencia

    def describir(self):
        return f"Vehículo: {self.marca} {self.modelo}"

class Moto(Automovil): # Concepto de Herencia

    def describir(self):
        return f"Moto: {self.marca} {self.modelo}"

def describir_vehiculo(vehiculo): # Concepto de Polimorfismo

    print(vehiculo.describir())

def main():

    coche = Coche("Toyota", "Corolla")
    moto = Moto("Honda", "CBR")

    describir_vehiculo(coche)
    describir_vehiculo(moto)

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class Dispositivo:

    def __init__(self, modelo):
        self.modelo = modelo

    def scanner_vuln(self):
        raise NotImplementedError("Este método debe ser definido para el resto de subclases existentes")

class Ordenador(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] Análisis de vulnerabilidades concluido: ¡Actualización de software necesaria!"

class Router(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] Análisis de vulnerabilidades concluido: ¡Múltiples puertos críticos detectados!"

class Telefono(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] ¡Múltiples aplicaciones con permisos excesivos!"

def tipo_dispositivo(dispositivo): # Uso de Polimorfismo

    print(dispositivo.scanner_vuln())

def main():

    pc = Ordenador("Dell XPS") # Creando objeto con la clase de herencia
    router = Router("TP-Link Archer C50")
    movil = Telefono("Samsung S9 Plus")

    tipo_dispositivo(pc) # Enviando el objeto a la función con Polimorfismo
    tipo_dispositivo(router)
    tipo_dispositivo(movil)

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class A:

    def __init__(self, x):
        self.x = x
        print(f"Valor en X: {x}")

class B(A):
    def __init__(self, x, y):
        self.y = y
        super().__init__(x) # Llama al constructor de la clase padre A
        print(f"Valor en Y: {y}")

def main():
    # a = A() # Con esto llamamos al constructor de A
    b = B(2, 10) # Con esto llamamos al constructor de B

    # En algunos casos vamos a querer que también se ejecute el constructor o función de la clase padre y luego la modificación del constructor o función de la clase hija.
    # Esto lo vamos a poder realizar cuando aumentamos super().<funcion> para que se ejecute.

if __name__ == "__main__":
    main()
```

> [!tip]
> La función `super()` se utiliza para llamar a métodos de la clase padre (superclase) desde una subclase. Es especialmente útil en el método `__init__` de la subclase para asegurar que los atributos de la superclase sean inicializados correctamente antes de que la subclase añada sus propios atributos.

```python
#!/usr/bin/env python3

class Persona:

    def __init__(self, nombre, edad):
        self.nombre = nombre
        self.edad = edad

    def saludo(self):
        return f"Hola, soy {self.nombre}, tengo {self.edad} años"

class Empleado(Persona): # Definiendo clase con herencia
    def __init__(self, nombre, edad, salario):
        super().__init__(nombre, edad) # Gracias a super() reutilizamos el constructor de Persona para inicializar nombre y edad.
        self.salario = salario

    def saludo(self):
        return f"{super().saludo()} y cobro {self.salario} dólares mensuales" # Con ayuda de super() llamamos al método saludo de la clase Persona.

def main():

    persona = Empleado("Alicia", 23, 2500)
    print(persona.saludo())

if __name__ == "__main__":
    main()
```

# Encapsulamiento y Métodos Especiales

El encapsulamiento en la programación orientada a objetos (POO) maneja principalmente tres niveles de visibilidad para los atributos y métodos de una clase: públicos, protegidos y privados. En Python, esta distinción se realiza mediante convenciones en la nomenclatura, más que a través de estrictas restricciones de acceso como en otros lenguajes.

## Atributos Públicos

Son accesibles desde cualquier parte del programa y, por convención, no tienen un prefijo especial. Se espera que estos atributos sean parte de la interfaz permanente de la clase.

## Atributos Protegidos

Se indican con un único guion bajo al principio del nombre (por ejemplo, `_atributo`). Esto es solo una convención y no impide el acceso desde fuera de la clase, pero se entiende que estos atributos están protegidos y no deberían ser accesibles directamente, excepto dentro de la propia clase y en subclases.

## Atributos Privados

En Python, los atributos privados se indican con un doble guion bajo al principio del nombre (por ejemplo, `__atributo`). Esto activa un mecanismo de cambio de nombre conocido como 'name mangling', donde el intérprete de Python altera internamente el nombre del atributo para hacer más difícil su acceso desde fuera de la clase.

## Métodos Especiales

Los métodos especiales en Python son también conocidos como métodos mágicos y son identificados por doble guion bajo al inicio y al final (`__metodo__`). Permiten a las clases en Python emular el comportamiento de los tipos incorporados y responder a operadores específicos. Por ejemplo, el método `__init__` se utiliza para inicializar una nueva instancia de una clase, `__str__` se invoca para una representación en forma de cadena legible del objeto y `__len__` devuelve la longitud del objeto.

Algunos métodos especiales importantes en POO son:

- `__init__(self, [...])`: Inicializa una nueva instancia de la clase.
- `__str__(self)`: Devuelve una representación en cadena de texto del objeto, invocado por la función `str(object)` y `print`.
- `__repr__(self)`: Devuelve una representación del objeto que debería, en teoría, ser una expresión válida de Python, invocado por la función `repr(object)`.
- `__eq__(self, other)`: Define el comportamiento del operador de igualdad `==`.
- `__lt__(self, other)`, `__le__(self, other)`, `__gt__(self, other)`, `__ge__(self, other)`: Definen el comportamiento de los operadores de comparación (`<`, `<=`, `>`, y `>=` respectivamente).
- `__add__(self, other)`, `__sub__(self, other)`, `__mul__(self, other)`, etc.: Definen el comportamiento de los operadores aritméticos (`+`, `–`, `*`, etc.).

El encapsulamiento y los métodos especiales son herramientas poderosas que, cuando se utilizan correctamente, pueden mejorar la seguridad, la flexibilidad y la claridad en la construcción de aplicaciones. A lo largo de esta clase, exploraremos en detalle cómo implementar y utilizar estos conceptos y métodos para crear clases robustas y mantenibles en Python.

```python
#!/usr/bin/env python3

class Ejemplo:

    def __init__(self):
        # Atributo protegido
        self._atributo_protegido = "Soy un atributo protegido y no deberías poder verme"
        # Atributo Privado
        self.__atributo_privado = "Soy un atributo privado y no debes verme"

def main():
    ejemplo = Ejemplo()
    print(ejemplo._atributo_protegido) # El objetivo es no referenciarlo directamente, aunque técnicamente se pueda acceder.
    #print(ejemplo.__atributo_privado) # Esto generará un AttributeError porque el name mangling hace que el acceso directo sea más difícil.

if __name__ == "__main__":
    main()
```

> [!warning]
> Aunque Python no impone una encapsulación estricta como otros lenguajes, el uso de `_` (protegido) y `__` (privado, con *name mangling*) es una convención importante. Acceder directamente a atributos protegidos o privados desde fuera de la clase rompe el principio de encapsulamiento y puede llevar a un código menos mantenible y propenso a errores.

```python
#!/usr/bin/env python3

class Coche:

    def __init__(self, marca, modelo):
        self.marca = marca
        self.modelo = modelo
        self.__kilometraje = 0 # Atributo de tipo privado

    def conducir(self, kilometros):

        if kilometros >= 0:
            self.__kilometraje += kilometros
        else:
            print(f"\n[!] Los kilómetros deben ser mayores a {kilometros}")

    def mostrar_kilometros(self):
        
        return self.__kilometraje # Desde dentro de la clase podemos acceder a él, pero fuera de la misma no se recomienda el acceso directo, al igual que con los atributos protegidos.

def main():

    coche = Coche("Toyota", "Corolla")
    coche.conducir(150)
    print(coche.mostrar_kilometros())

if __name__ == "__main__":
    main()
```

### Empleo de métodos especiales:

```python
#!/usr/bin/env python3

class Libro:
    
    def __init__(self, autor, titulo):
        self.autor = autor
        self.titulo = titulo

    def __str__(self): # Método especial que nos permite definir la representación en cadena de texto del objeto cuando se imprime o se convierte a str().
        return f"El libro {self.titulo} ha sido escrito por {self.autor}"

    def __eq__(self,otro):
        return self.autor == otro.autor and self.titulo == otro.titulo

def main():
    mi_libro = Libro("Daniel Estrella", "Cómo ser un Hacker")
    libro_dos = Libro("Dylan Estrella", "Cómo ser el mejor jugador")
    print(mi_libro)
    print(libro_dos)

    print(f"¿Son iguales los dos libros? {mi_libro == libro_dos}")

if __name__ == "__main__":
    main()
```

Implementamos variables privadas a nuestro programa:

```python
#!/usr/bin/env python3

class CuentaBancaria:

    def __init__(self, num_cuenta, titular, saldo_inicial=0):
        self.num_cuenta = num_cuenta
        self.titular = titular
        self.__saldo = saldo_inicial # Definimos un atributo privado para controlar su modificación y acceso.

    def depositar_dinero(self, dinero):

        if dinero > 0:
            self.__saldo += dinero

    def retirar_dinero(self, dinero):

        if dinero <= self.__saldo and dinero > 0:
            self.__saldo = self.__saldo - dinero
        else:
            print(f"\n[!] No puedes retirar la cantidad de {dinero} $\n")

    def mostrar_dinero(self):

        return f"[+] El saldo actual de la cuenta es de: {self.__saldo}"

def main():
    manolo = CuentaBancaria("12341", "Manolo Vieira", 1000)
    manolo.depositar_dinero(100)
    manolo.retirar_dinero(1500)
    print(manolo.mostrar_dinero())

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class Caja:

    def __init__(self, *frutas): # De esta forma creamos una tupla en la que se almacenarán un número variable de elementos que le pasemos.

        self.frutas = frutas # Por lo tanto, se almacenará una tupla con todas las frutas que le pasemos.

    def mostrar_items(self): # Este método nos ayuda a listar las frutas que le pasamos.

        for fruta in self.frutas:
            print(fruta)

    def __len__(self): # Con esto podemos retornar la longitud de la colección de frutas almacenadas dentro del objeto.

        return len(self.frutas)

def main():
    caja = Caja("Manzana", "Plátano", "Kiwi", "Pera")
    caja.mostrar_items()
    print(len(caja)) # Esto solo es posible si tenemos declarado el método especial __len__.

if __name__ == "__main__":
    main()
```

> [!info]
> El método especial `__len__` permite que un objeto responda a la función `len()`. Es útil para colecciones personalizadas, haciendo que se comporten de manera similar a las listas o tuplas integradas de Python.

En este programa hacemos uso de métodos especiales y además implementamos el método `join()` para iterar en una tupla y separa los elementos por el delimitador que especifiquemos.

```python
#!/usr/bin/env python3

class Pizza:

    def __init__(self, size, *ingredientes):
        self.size = size
        self.ingredientes = ingredientes

    def descripcion(self):
    
        print(f"Esta pizza es de {self.size} y los ingredientes son: {', '.join(self.ingredientes)}")

def main():
    pizza = Pizza(12, "Chorizo", "Jamón", "Bacon", "Queso", "Cebolla")
    pizza.descripcion()

if __name__ == "__main__":
    main()
```

En este programa hacemos uso de otro método especial el cual es `__getitem__`:

```python
#!/usr/bin/env python3

class MiLista:

    def __init__(self):
        self.data = [1, 2, 3, 4, 5]

    def __getitem__(self, index): # Esto nos permite acceder a los elementos de una lista o cualquier secuencia mediante su índice, como si el objeto fuera una lista.

        return self.data[index]

def main():
    lista = MiLista()
    print(lista.data) # De esta forma podemos imprimir los elementos almacenados en una variable de nuestra clase.
    print(lista.data[1]) # Al ser una lista, podemos acceder a sus elementos a través de su índice.

    ## Implementación del método especial
    print(lista[2]) # Esto solo es válido cuando tenemos el método __getitem__ declarado en nuestra clase.

if __name__ == "__main__":
    main()
```

> [!info]
> El método `__getitem__` permite que las instancias de una clase soporten la indexación (`objeto[indice]`), lo que las hace comportarse como secuencias (listas, tuplas, etc.).

En este caso estoy implementando el método especial `__call__`:

```python
#!/usr/bin/env python3

class Saludo:
    
    def __init__(self, saludo):
        self.saludo = saludo

    def __call__(self, nombre): # Este es un método especial que permite que una instancia de la clase sea llamada como si fuera una función.

        return f"{self.saludo} {nombre}!"

def main():
    hola = Saludo("!Hola")
    print(hola("Luis")) # Esta forma de llamar al objeto solo es válida cuando empleamos métodos especiales como el de __call__.

if __name__ == "__main__":
    main()
```

> [!info]
> El método `__call__` hace que un objeto sea "callable". Esto significa que puedes usar los paréntesis `()` directamente en la instancia del objeto, como si fuera una función, permitiendo una sintaxis más limpia para ciertas operaciones.

Estamos haciendo uso del método `__add__` el cual, para una representación legible del resultado, va en conjunto con el método `__str__`.

```python
#!/usr/bin/env python3

class Punto:

    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __add__(self, other): # Como la suma es entre dos objetos, tendremos que representar ambos: uno que es self y otro que es other.

        return Punto(self.x + other.x, self.y + other.y) # De esta forma podemos crear un objeto temporal que se retorna a través de esta función.

    def __str__(self): # Si lo pensamos, este método nos ayuda a retornar de manera correcta la representación en cadena de un objeto cuando lo imprimimos o convertimos a str().

        return f"({self.x}, {self.y})" # Podemos retornar el objeto temporal, ya que esta función se encarga de la representación en cadena de todas las variables que se retornan, y al retornarse un objeto, este método especial lo procesa y lo imprime de manera correcta.

def main():
    p1 = Punto(2, 8)
    p2 = Punto(4, 9)

    print(p1 + p2) # Esto no es válido a menos que se implemente el método especial __add__ para poder operar la suma y __str__ para que Python sepa cómo devolver el resultado de forma legible.

if __name__ == "__main__":
    main()
```

> [!info]
> El método `__add__` sobrecarga el operador `+` para objetos de la clase, permitiendo definir cómo se suman. Combinado con `__str__`, proporciona una forma intuitiva de realizar operaciones aritméticas y mostrar sus resultados de manera legible.

Empleamos métodos como lo son el método `__iter__` y el método `__next__` podemos usar un objeto como un iterador:

```python
#!/usr/bin/env python3

class Contador:
    
    def __init__(self, limite):
        self.limite = limite

    def __iter__(self): # Este método nos devuelve un iterable.

        self.contador = 0
        return self

    def __next__(self): # Este método nos permite aumentar la variable contador.

        if self.contador < self.limite:
            self.contador += 1
            return self.contador
        else:
            raise StopIteration

def main():

    c = Contador(15)
    for i in c: # Con esto podemos usar el objeto como un iterador.
        print(i)

if __name__ == "__main__":
    main()
```

> [!info]
> Los métodos `__iter__` y `__next__` son fundamentales para hacer que un objeto sea un iterador. `__iter__` debe devolver el propio objeto iterador (o un nuevo iterador), y `__next__` debe devolver el siguiente elemento de la secuencia. Cuando no hay más elementos, `__next__` debe lanzar `StopIteration`.

# Decoradores y Propiedades

En esta clase, profundizaremos en estas poderosas características de Python que mejoran significativamente la forma en que podemos manejar y modificar el comportamiento de nuestras clases y funciones.

## Decoradores

Los decoradores en Python son una forma elegante de modificar funciones o métodos. Se utilizan para extender o alterar el comportamiento de la función sin cambiar su código fuente. Un decorador es en sí mismo una función que toma otra función como argumento y devuelve una nueva función que, opcionalmente, puede agregar alguna funcionalidad antes y después de la función original.

**Propiedades**

Las propiedades son un caso especial de decoradores que permiten a los desarrolladores añadir "getter", "setter" y "deleter" a los atributos de una clase de manera elegante, controlando así el acceso y la modificación de los datos. En Python, esto se logra con el decorador `@property`, que transforma un método para acceder a un atributo como si fuera una propiedad pública.

## Getters y Setters

- El "getter" obtiene el valor de un atributo, manteniendo el encapsulamiento y permitiendo que se ejecute una lógica adicional durante el acceso.
- El "setter" establece el valor de un atributo y puede incluir validación o procesamiento antes de que el cambio se refleje en el estado interno del objeto.
- El "deleter" puede ser utilizado para definir un comportamiento cuando un atributo es eliminado con `del`.

Durante la clase, discutiremos cómo los decoradores pueden ser aplicados no solo para métodos y propiedades, sino también para funciones en general. También exploraremos cómo las propiedades se pueden utilizar para crear una interfaz pública para atributos privados/protegidos, mejorando la encapsulación y manteniendo la integridad de los datos de una clase.

Este conocimiento es crucial para escribir código Python idiomático y eficiente, aprovechando al máximo lo que el lenguaje tiene para ofrecer en términos de flexibilidad y potencia en el diseño de software.

```python
#!/usr/bin/env python3

def mi_decorador(funcion): # Esta es una función de orden superior.

    def envoltura(): # Esta es una función interna (closure) que nos permite añadir lógica antes y después de la función original.

        print("Estoy saludando en la envoltura del decorador antes de llamar a la función") # Esto es antes de saludo
        funcion() # Esta es la función saludo que ahora toma el nombre de funcion
        print("Estoy saludando en la envoltura del decorador después de llamar a la función") # Esto es después de saludo

    return envoltura # Se retorna la función envoltura, ya que esta contiene las modificaciones.

@mi_decorador # Esta es la forma de aplicar un decorador a una función.
def saludo():

    print("Hola, estoy saludando dentro de la función `saludo`")

def main():

    saludo()

if __name__ == "__main__":
    main()
```

> [!info]
> Los decoradores son funciones que envuelven a otras funciones, permitiendo añadir funcionalidad extra antes o después de la ejecución de la función original sin modificar su código directamente. Son una forma de metaprogramación.

```python
#!/usr/bin/env python3

class Persona:

    def __init__(self, nombre, edad):
        self._nombre = nombre # Esto es una variable protegida y, aunque se pueda acceder, no se debe hacer directamente.
        self._edad = edad

    @property # Nos permite crear una nueva propiedad. Recordemos que cuando usamos este decorador lo llamamos como una propiedad.
    def edad(self):
        return self._edad # Getter: Con esto obtenemos el valor de un atributo protegido o privado.

    @edad.setter # Creación de un setter: Este tipo de decorador, al igual que con el getter, lo llamamos como una propiedad y no como una función.
    def edad(self, edad): # Creamos la función con la cual ya podemos asignar un valor internamente.
        if edad > 0:
            self._edad = edad
        else:
            raise ValueError("[!] La edad no puede ser un número menor a 0")

    # Tanto con getter como con setter, vamos a mantener protegidos los valores originales de las variables protegidas o privadas.

def main():
    manolo = Persona("Manolo", 23)
    manolo.edad = -1 # Setter: Nos permite ingresar un valor y lo hacemos como si fuera una propiedad, porque recordemos que no podemos llamarlo como un método.
    print(manolo.edad) # Getter: Nos permite ver u obtener el valor y lo hacemos como si fuera una propiedad, porque recordemos que no podemos llamarlo como un método.

if __name__ == "__main__":
    main()
```

> [!tip]
> El decorador `@property` y sus métodos asociados (`@.setter`, `@.deleter`) son la forma "pythónica" de implementar getters y setters. Permiten controlar el acceso y la modificación de atributos, añadiendo lógica de validación o transformación, sin que el usuario de la clase tenga que llamar a métodos explícitos como `get_edad()` o `set_edad()`.

```python
#!/usr/bin/env python3

import time # De esta forma podemos importar librerías.

def cronometro(function):
    def envoltura(*args, **kwargs): # Cuando hacemos uso de decoradores con funciones que tienen parámetros, tenemos que pasarle los mismos parámetros a la función envoltura y a la función original.

        inicio = time.time()
        result = function(*args, **kwargs)
        final = time.time()
        
        resultado = final - inicio

        print(f"Tiempo total de ejecución en la función {function.__name__}: {round(resultado, 1)}") # La propiedad __name__ nos permite imprimir el nombre de la función.
        return result

    return envoltura

@cronometro
def pausa_corta(num):
    time.sleep(num)

@cronometro
def pausa_larga(num):
    time.sleep(num)

def main():

    pausa_corta(2)
    pausa_larga(3)

if __name__ == "__main__":
    main()
```

> [!info]
> Para crear decoradores que acepten argumentos en la función decorada, es crucial que la función `envoltura` (wrapper) acepte `*args` y `**kwargs`. Esto asegura que el decorador pueda manejar cualquier número de argumentos posicionales y de palabra clave que se pasen a la función original.

```python
#!/usr/bin/env python3

# *args

def suma(*args): # Esto lo usamos específicamente cuando pasamos un número variable de valores, ya que *args creará una tupla con ellos.
    return sum(args)

def main():
    print(suma(2, 3, 4, 6, 12, 14, 32))

if __name__ == "__main__":
    main()
```

> [!tip]
> `*args` permite que una función acepte un número variable de argumentos posicionales. Estos argumentos se empaquetan en una tupla dentro de la función, lo que facilita el manejo de entradas flexibles.

```python
#!/usr/bin/env python3

# **kwargs

def presentacion(**kwargs): # Con esto ya no almacenamos en una tupla, sino en un diccionario.

    for key, value in kwargs.items():
        print(f"{key} : {value}\n")

def main():
    presentacion(nombre = "Daniel", edad=20, ciudad="Quito", profesion="Estudiante")

if __name__ == "__main__":
    main()

```

```python
#!/usr/bin/env python3

# **kwargs

def presentacion(**kwargs): # Con esto ya no almacenamos en una tupla, sino en un diccionario.

    for key, value in kwargs.items():
        print(f"{key} : {value}\n")

def main():
    presentacion(nombre = "Daniel", edad=20, ciudad="Quito", profesion="Estudiante")

if __name__ == "__main__":
    main()

```

> [!tip]
> `**kwargs` permite que una función acepte un número variable de argumentos de palabra clave. Estos argumentos se empaquetan en un diccionario dentro de la función, lo que es útil para configuraciones o datos con nombres específicos.

```python
#!/usr/bin/env python3

class Circunferencia:

    def __init__(self, radio):
        self._radio = radio
        
    @property # Getter
    def radio(self):
        return self._radio

    @property # Getter
    def diametro(self):
        return 2 * self._radio

    @property # Getter
    def area(self):
        return 3.1415 * (self._radio ** 2)

    @radio.setter # Setter
    def radio(self, valor):
        self._radio = valor

def main():
    c = Circunferencia(5)
    print(c.radio)
    print(c.diametro)
    print(c.area)
    print("\n\n")
    c.radio = 10
    print(c.radio)
    print(c.diametro)
    print(c.area)

if __name__ == "__main__":
    main()

```