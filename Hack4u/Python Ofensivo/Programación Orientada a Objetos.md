------
# Clases y Objetos

La Programación Orientada a Objetos (POO) es un paradigma de programación que utiliza objetos y clases en su enfoque central. Es una manera de estructurar y organizar el código que refleja cómo los desarrolladores piensan sobre el mundo real y las entidades dentro de él. Pero para esto tenemos que tener claro conceptos como: [[Conceptos Básicos de Python]], [[Colecciones y Estructuras de Datos en Python]], [[Entrada y Salida de Datos]].
## **Clases**

Las clases son los fundamentos de la POO. Actúan como plantillas para la creación de objetos y definen atributos y comportamientos que los objetos creados a partir de ellas tendrán. En Python, una clase se define con la palabra clave ‘**class**‘ y proporciona la estructura inicial para todo objeto que se derive de ella.

**Instancias de Clase y Objetos**

Un objeto es una instancia de una clase. Cada vez que se crea un objeto, se está creando una instancia que tiene su propio espacio de memoria y conjunto de valores para los atributos definidos por su clase. Los objetos encapsulan datos y funciones juntos en una entidad discreta.
## **Métodos de Clase**

Los métodos de clase son funciones que se definen dentro de una clase y solo pueden ser llamados por las instancias de esa clase. Estos métodos son el mecanismo principal para interactuar con los objetos, permitiéndoles realizar operaciones o acciones, modificar su estado o incluso interactuar con otros objetos.

En esta clase, te proporcionaremos las herramientas y el entendimiento necesario para comenzar a diseñar y desarrollar tus propias clases y a crear instancias de esas clases en objetos funcionales. Aprenderemos cómo los métodos de clase operan y cómo puedes utilizarlos para dar vida al comportamiento de tus objetos en Python. Este conocimiento será esencial a medida que continúes aprendiendo y aplicando los principios de la POO en proyectos más complejos.

```python
#!/usr/bin/env python3

class Persona: # Definiendo una clase

    def __init__(self, nombre, edad): # Definiendo el constructor Persona.__init__(marcelo, nombre(Daniel), edad(20))

        # Estamos inicializando los valores
        self.nombre = nombre # daniel.nombre = "Daniel"
        self.edad = edad # daniel.edad = 20

    def saludo(self): # Definiendo un funcion cualquiera
        return f"Hola soy {self.nombre} y tengo {self.edad} años"

# Funcion Principal
def main():

    daniel = Persona("Daniel", 20) # Estamos creando un objetos, en este caso daniel
    dylan = Persona("Dylan", 9)   # Nuevo objeto dylan
    print(daniel.saludo()) # Estamos llamando al objeto daniel el cual cuenta con la función saludo
    print(dylan.saludo()) # Estamos llamando al objeto dylan

if __name__ == "__main__":
    main()

```

Otro Ejemplo puede ser el siguiente:

```python
#!/usr/bin/env python3

juegos = ["Super Mario Bros", "Zelda: Breth of the Wild", "Cyberpunk 2077", "Final Fantasy VII"]
tope = 100 

# Géneros

❯ catnp ejercicio2.py
#!/usr/bin/env python3

class Animal: # Creamos la clase

    def __init__(self, tipo, nombre, edad): # Definimos el constructor
        # Inicializamos los Datos
        self.tipo = tipo
        self.nombre = nombre
        self.edad = edad
    
    def datos(self): #Funcion cualquiera
        return f"Su nombre es {self.nombre} es un {self.tipo} y tiene {self.edad} años"

# Funcion Principal
def main():

    gato = Animal("Felino", "Flash", 5)
    gallina = Animal("Ave", "Turuleca", 2)
    print(gato.datos())
    print(gallina.datos())

if __name__ == "__main__":
    main()
```

Vamos a ver un ejemplo un poco mas complicado para terminar de comprender las clases:

```python
#!/urs/bin/env python3

class CuentaBancaria:

    def __init__(self,numero_cuenta, nombre, dinero=0): #Cuando nosotros inicamos con un valore como en este caso, el valor 0 solo entra en uso si no se le pasa nada a la clase
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
            print(f"A retirado {dinero}$ su saldo actual es de {self.dinero}$")
        else:
            print(f"No tiene ese valor en la cuenta")

# Funcion Principal
def main():

    manolo = CuentaBancaria("123413", "Manolo Zambrano", 10000)
    manolo.depositar(500)
    manolo.retirar(10500)

    daniel = CuentaBancaria("32452345", "Daniel Estrella", 5000)
    daniel.retirar(500)

if __name__ == "__main__":
    main()
```

Los siguientes ejercicio continúen el uso de decoradores y funciones especiales mediante las cuales podemos cumplir un objetivo:

```python
#!/usr/bin/env python3

class Rectangulo:
    def __init__(self, ancho, alto):
        self.ancho = ancho
        self.alto = alto

    @property # Este es un decorador el cual nos permite convertir el resultado de propiedad mas no en función
    def area(self):
        return self.ancho * self.alto

    def __str__(self): # Este es un metodo el cual se ejecuta por defecto si no se llama a ningun metodo especifico
        return f"\\n[+] Propiedades del rectángulo: [Anchor: {self.ancho}][Alto: {self.alto}]"

    def __eq__(self, otro): # Metodo especial el cual nos permite comprobar igualdad de valores en dos objetos

        return self.ancho == otro.ancho and self.alto == otro.alto
        

def main():
    area = Rectangulo(20, 80)
    area1 = Rectangulo(20, 80)
    print(f"El area del Rectangulo es {area.area}")
    print(area) # al no pasarle ningun metodo especifico se ejecuta el metodo __str__
    print(f"Los rectangulos son Iguales? {area == area1 }") # Esta definicion sin el metodod __eq__ no seria valido

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class Libro:
    
    besteseller_value = 5000 # Esta es una variable global en la clase
    iva=0.21

    def __init__(self, titulo, autor, precio):
        self.titulo = titulo
        self.autor = autor
        self.precio = precio

    @staticmethod # Este es un decorador que nos permite operar sin el objeto
    def es_besteseller(ventas):
        return ventas > Libro.besteseller_value # esta es la manera correcta de usar una variable global de la misma clase

    @classmethod # Este docorador hace que el metodo reciva a su propia clase como argumento
    def inclucion_iva(cls,precio):
        print(f"\\n[+] El precio con el iva incluido es de {precio*cls.iva + precio}") # Esto nos ayuda cuando ya tenemos muchas clases y tambien herencia

def main():
    mi_libro = Libro("48 Leyes", "Daniel", 16)
    print(mi_libro.es_besteseller(10000))
    mi_libro.inclucion_iva(100)

if __name__ == "__main__":
    main()
  
```

# Métodos estáticos y métodos de Clase

Los métodos estáticos y los métodos de clase son dos herramientas poderosas en la programación orientada a objetos en Python, que ofrecen flexibilidad en cómo se puede acceder y utilizar la funcionalidad asociada con una clase.
## **Métodos de Clase**

Se definen con el decorador ‘**@classmethod**‘, lo que les permite tomar la clase como primer argumento, generalmente nombrada ‘**cls**‘. Este acceso a la clase permite que los métodos de clase interactúen con la estructura de la clase en sí, como modificar atributos de clase que afectarán a todas las instancias. Se utilizan para tareas que requieren conocimiento del estado global de la clase, como la construcción de instancias de maneras específicas, también conocidos como métodos factory.
## **Métodos Estáticos**

Se definen con el decorador ‘**@staticmethod**‘ y no reciben un argumento implícito de referencia a la clase o instancia. Son similares a las funciones regulares definidas dentro del cuerpo de una clase. Se utilizan para funciones que, aunque conceptualmente pertenecen a la clase debido a la relevancia temática, no necesitan acceder a ningún dato específico de la clase o instancia. Proporcionan una manera de encapsular la funcionalidad dentro de una clase, manteniendo la cohesión y la organización del código.

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
    def sean(cls, marca):
        return cls(marca, "sean")

    def __str__(self):
        return f"La marca {self.marca} es un modelo {self.modelo}"

def main():
    deportivo = print(Automovil.deportivos("Ferrari"))
    sean = print(Automovil.sean("Toyota"))

if __name__ == "__main__":
    main()
```

# Uso de self

El uso de self es uno de los aspectos más fundamentales y a la vez confusos para quienes se adentran en la Programación Orientada a Objetos (POO) en Python. Este identificador es crucial para entender cómo Python maneja los métodos y atributos dentro de sus clases y objetos.
## **Definición de ‘self’**

A nivel conceptual, ‘**self**‘ es una referencia al objeto actual dentro de la clase. Es el primer parámetro que se pasa a cualquier método de una clase en Python. A través de self, un método puede acceder y manipular los atributos del objeto y llamar a otros métodos dentro del mismo objeto.
## **Uso de ‘self’**

Cuando se crea una nueva instancia de una clase, Python pasa automáticamente la instancia recién creada como el primer argumento al método ‘****init****‘ y a otros métodos definidos en la clase que tienen self como su primer parámetro. Esto es lo que permite que un método opere con datos específicos del objeto y no con datos de la clase en general o de otras instancias de la clase.
## **Importancia de ‘self’**

El concepto de self es importante en la POO ya que asegura que los métodos y atributos se apliquen al objeto correcto. Sin self, no podríamos diferenciar entre las operaciones y datos de diferentes instancias de una clase.

En esta clase, nos enfocaremos en comprender a fondo cómo y por qué self es usado en Python, explorando su papel en la interacción con las instancias de la clase. Desarrollaremos una comprensión clara de cómo self permite que las clases en Python sean intuitivas y eficientes, manteniendo un estado consistente a través de las operaciones del objeto. Este conocimiento es esencial para trabajar con clases y objetos de manera efectiva y aprovechar la potencia de la POO.

# Herencia y Polimorfismo

La herencia y el polimorfismo son conceptos fundamentales en la programación orientada a objetos que permiten la creación de una estructura de clases flexible y reutilizable.
## **Herencia**

Es un principio de la POO que permite a una clase heredar atributos y métodos de otra clase, conocida como su clase base o superclase. La herencia facilita la reutilización de código y la creación de una jerarquía de clases. Las subclases heredan las características de la superclase, lo que permite que se especialicen o modifiquen comportamientos existentes.
## **Polimorfismo**

Este concepto se refiere a la habilidad de objetos de diferentes clases de ser tratados como instancias de una clase común. El polimorfismo permite que una función o método interactúe con objetos de diferentes clases y los trate como si fueran del mismo tipo, siempre y cuando compartan la misma interfaz o método. Esto significa que el mismo método puede comportarse de manera diferente en distintas clases, un concepto conocido como sobrecarga de métodos.

Ambos, la herencia y el polimorfismo, son piedras angulares de la POO y son ampliamente utilizados para diseñar sistemas que son fácilmente extensibles y mantenibles.

En esta clase, exploraremos cómo implementar herencia en Python y cómo se puede aprovechar el polimorfismo para escribir código más general y potente. Estos conceptos nos ayudarán a entender mejor cómo construir jerarquías de clases y cómo los diferentes objetos pueden interactuar entre sí de manera flexible.

```jsx
#!/usr/bin/env python3

class Animal:

    def __init__(self, nombre):
        self.nombre = nombre

    def hablar(self):
        
        raise NotImplementedError("Las Subclases deben implementar este metodo") # Este error se ejecuta cuando llamamos directo a la funcion animal la cual no cuenta con la funcion hablar implementada.

class Gato(Animal): # Estamos creando una clase la cual recive herencia de Animal Gato(Animal(gato, nombre))

    def hablar(self):
        return f"!Miau!"

class Perro(Animal):

    def hablar(self):
        return f"!Wuaw!"

def hacer_hablar(objeto):

    print(f"{objeto.nombre} dice {objeto.hablar()}") # Esto es un ejemplo de polimorfismo de como un metodo se comporta de forma distinta para diferentes clases

def main():

    gato = Gato("Firulais")
    perro = Perro("lulu")

    hacer_hablar(gato)
    hacer_hablar(perro)

if __name__ == "__main__":
    main()
```

```jsx
#!/usr/bin/env python3

class Automovil:

    def __init__(self, marca, modelo):
        self.marca = marca
        self.modelo = modelo

    def describir(self):
        return f"Veiculo: {self.marca} {self.modelo}"

class Coche(Automovil): # Concepto de herencia

    def describir(self):
        return f"Veiculo: {self.marca} {self.modelo}"

class Moto(Automovil): # Concepto de Herencia

    def describir(self):
        return f"Moto: {self.marca} {self.modelo}"

def describir_veiculo(veiculo): # Concepto de Polimorfismo

    print(veiculo.describir())

def main():

    coche = Coche("Toyota", "Corolla")
    moto = Moto("Honda", "CBR")

    describir_veiculo(coche)
    describir_veiculo(moto)

if __name__ == "__main__":
    main()
```

```jsx
#!/usr/bin/env python3

class Dispositivo:

    def __init__(self, modelo):
        self.modelo = modelo

    def scanner_vuln(self):
        raise NotImplementedError("Este metodo debe ser definido para el resto de sublcases existentes")

class Ordenador(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] Análisis de vulnerabilidades concluido: Actualización de software necesaria!!!"

class Router(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] Análisis de vulnerabilidades concluido: Múltiples puertos criticos detectados!!!"

class Telefono(Dispositivo): # Definiendo una clase con herencia

    def scanner_vuln(self):
        return f"[+] Múltiples aplicaciónes con permisos excesivos!!!!"

def tipo_dispositivo(dispositivo): # Uso de Polimorfismo

    print(dispositivo.scanner_vuln())

def main():

    pc = Ordenador("Dell XPS") # Creando objeto con la clase de herencia
    router = Router("TP-Link Archer C50")
    movil = Telefono("Samsun S9 Plus")

    tipo_dispositivo(pc) # Enviadon el objeto a la funcion con Polimorfismo
    tipo_dispositivo(router)
    tipo_dispositivo(movil)

if __name__ == "__main__":
    main()
```

```jsx
#!/usr/bin/env python3

class A:

    def __init__(self, x):
        self.x = x
        print(f"Valor en X: {x}")

class B(A):
    def __init__(self, x, y):
        self.y = y
        super().__init__(x)
        print(f"Valor en Y: {y}")

def main():
    # a = A() # Con esto llamamos al contructor de A
    b = B(2, 10) # Con esto llamamos al contructor de B

    # En algunos casos vamos a querer que tambien se ejecute el constructor o funcion de la clase padre y luego la modificación del constructor o funcion de la clase hija
    # Esto lo vamos a poder ralizar cuando aumentamos super().<funcion> para que se ejecute

if __name__ == "__main__":
    main()
```

```jsx
#!/usr/bin/env python3

class Persona:

    def __init__(self, nombre, edad):
        self.nombre = nombre
        self.edad = edad

    def saludo(self):
        return f"Hola, soy {self.nombre}, tengo {self.edad} años"

class Empleado(Persona): # Definiendo clase con herencia
    def __init__(self, nombre, edad, salario):
        super().__init__(nombre, edad) # gracias a super reutilizamos el consturctor de Persona para almacenar nombre y edad
        self.salario = salario

    def saludo(self):
        return f"{super().saludo()} y cobro {self.salario} dolares mensuales" # Con ayuda de super llamamos a la funcion saludo de persona

def main():

    persona = Empleado("Alicia", 23, 2500)
    print(persona.saludo())

if __name__ == "__main__":
    main()
```

# **Encapsulamiento y métodos especiales**

El encapsulamiento en la programación orientada a objetos (POO) maneja principalmente tres niveles de visibilidad para los atributos y métodos de una clase: públicos, protegidos y privados. En Python, esta distinción se realiza mediante convenciones en la nomenclatura, más que a través de estrictas restricciones de acceso como en otros lenguajes.
## **Atributos Públicos**

Son accesibles desde cualquier parte del programa y, por convención, no tienen un prefijo especial. Se espera que estos atributos sean parte de la interfaz permanente de la clase.
## **Atributos Protegidos**

Se indica con un único guion bajo al principio del nombre (por ejemplo, ‘**_atributo**‘). Esto es solo una convención y no impide el acceso desde fuera de la clase, pero se entiende que estos atributos están protegidos y no deberían ser accesibles directamente, excepto dentro de la propia clase y en subclases.
## **Atributos Privados**

En Python, los atributos privados se indican con un doble guion bajo al principio del nombre (por ejemplo, ‘**__atributo**‘). Esto activa un mecanismo de cambio de nombre conocido como ‘**name mangling**‘, donde el intérprete de Python altera internamente el nombre del atributo para hacer más difícil su acceso desde fuera de la clase.
## **Métodos Especiales**

Los métodos especiales en Python son también conocidos como métodos mágicos y son identificados por doble guion bajo al inicio y al final (‘****metodo****‘). Permiten a las clases en Python emular el comportamiento de los tipos incorporados y responder a operadores específicos. Por ejemplo, el método ‘****init****‘ se utiliza para inicializar una nueva instancia de una clase, ‘****str****‘ se invoca para una representación en forma de cadena legible del objeto y ‘****len****‘ devuelve la longitud del objeto.

Algunos métodos especiales importantes en POO son:

- ****init**(self, […])**: Inicializa una nueva instancia de la clase.
- ****str**(self)**: Devuelve una representación en cadena de texto del objeto, invocado por la función ‘**str(object)**‘ y ‘**print**‘.
- ****repr**(self)**: Devuelve una representación del objeto que debería, en teoría, ser una expresión válida de Python, invocado por la función ‘**repr(object)**‘.
- ****eq**(self, other)**: Define el comportamiento del operador de igualdad ‘**==**‘.
- ****lt**(self, other), **le**(self, other), **gt**(self, other), **ge**(self, other)**: Definen el comportamiento de los operadores de comparación (**<**, **<=**, **>** y **>=** respectivamente).
- ****add**(self, other), **sub**(self, other), **mul**(self, other), etc.**: Definen el comportamiento de los operadores aritméticos (**+**, **–**, ****, etc.).

El encapsulamiento y los métodos especiales son herramientas poderosas que, cuando se utilizan correctamente, pueden mejorar la seguridad, la flexibilidad y la claridad en la construcción de aplicaciones. A lo largo de esta clase, exploraremos en detalle cómo implementar y utilizar estos conceptos y métodos para crear clases robustas y mantenibles en Python.

```jsx
#!/usr/bin/env python3

class Ejemplo:

    def __init__(self):
        # Atributo protegido
        self._atributo_protegito = "Soy un atributo protegido y no deberías poder verme"
        # Atributo Privado
        self.__atributo_privado = "Soy un atributo privado y no deves verme"

def main():
    ejemplo = Ejemplo()
    print(ejemplo._atributo_protegito) # El objetivo es no referenciarlo, pero si podemos llamarlo
    #print(ejemplo.__atributo_privado) # Esto nos da un error y no podemos usarlo por el echo de ser privado

if __name__ == "__main__":
    main()
```

```jsx
#!/usr/bin/env python3

class Coche:

    def __init__(self, marca, modelo):
        self.marca = marca
        self.modelo = modelo
        self.__kilometrage = 0 # Atributo de tipo privado

    def conducir(self, kilometros):

        if kilometros >= 0:
            self.__kilometrage += kilometros
        else:
            print(f"\\n[!] Los kilometros deven ser mayores a {kilometros}")

    def mostrar_kilometros(self):
        
        return self.__kilometrage # Desde dentro de la clase lo podemos usar pero fuera de la misma no, lo mismo con los protegidos

def main():

    coche = Coche("Toyota", "Corolla")
    coche.conducir(150)
    print(coche.mostrar_kilometros())

if __name__ == "__main__":
    main()
```
### Empleo de métodos especiales:

```jsx
#!/usr/bin/env python3

class Libro:
    
    def __init__(self, autor, titulo):
        self.autor = autor
        self.titulo = titulo

    def __str__(self): # Metodo especial que nos permite imprimir información si solo se llama al objeto
        return f"El libro {self.titulo} a sido escrito por {self.autor}"

    def __eq__(self,otro):
        return self.autor == otro.autor and self.titulo == otro.titulo

def main():
    mi_libro = Libro("Daniel Estrella", "Como se un Hacker")
    libro_dos = Libro("Dylan Estrella", "Como se el mejor jugador")
    print(mi_libro)
    print(libro_dos)

    print(f"Son iguales los dos libros? {mi_libro == libro_dos}")

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
        self.__saldo = saldo_inicial # Definimos atributo privado para no poderlo modificar sin autorización

    def depositar_dinero(self, dinero):

        if dinero > 0:
            self.__saldo += dinero

    def retirar_dinero(self, dinero):

        if dinero <= self.__saldo and dinero > 0:
            self.__saldo = self.__saldo - dinero
        else:
            print(f"\\n[!] No puedes retirar la cantidad de {dinero} $\\n")

    def mostrar_dinero(self):

        return f"[+] El saldo actual de la cuenta es de: {self.__saldo}"

def main():
    manolo = CuentaBancaria("12341", "Manolo Vieria", 1000)
    manolo.depositar_dinero(100)
    manolo.retirar_dinero(1500)
    print(manolo.mostrar_dinero())

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class Caja:

    def __init__(self, *frutas): # De esta forma creamos una tupla en la cual no importa el numero de elementos que le pasemos estos se almacenaran alli

        self.frutas = frutas # Por lo tanto se almacenara un tupla con todas las frutas que le pasemos al arreglo

    def mostrar_items(self): # Este metodo nos ayuda a listar la fruta que le pasamos

        for fruta in self.frutas:
            print(fruta)

    def __len__(self): # Con esto podemos retornar la longitud de diferentes variables almacenadas dentro del objeto

        return len(self.frutas)

def main():
    caja = Caja("Manzana", "Platano", "Kiwi", "Pera")
    caja.mostrar_items()
    print(len(caja)) # Esto lo podemos hacer solo si tenemos declarado el metodo especial len

if __name__ == "__main__":
    main()
```

En este programa hacemos uso de métodos especiales y además implementamos el método join() para iterar en una tupla y separa todo por lo que nosotros queramos.

```python
#!/usr/bin/env python3

class Pizza:

    def __init__(self, size, *ingredientes):
        self.size = size
        self.ingredientes = ingredientes

    def descripcion(self):
    
        print(f"Esta pizza es de {self.size} y los ingredientes son: {', '.join(self.ingredientes)}")

def main():
    pizza = Pizza(12, "Chirizo", "Jamon", "Bacon", "Queso", "Cebolla")
    pizza.descripcion()

if __name__ == "__main__":
    main()
```

En este programa hacemos uso de otro método especial el cual es getitems:

```python
#!/usr/bin/env python3

class MiLista:

    def __init__(self):
        self.data = [1, 2, 3, 4, 5]

    def __getitem__(self, index): # Esto nos permite devolver el valor de una lista o cualquiervariable siempre que le pasemos el indice

        return self.data[index]

def main():
    lista = MiLista()
    print(lista.data) # De esta forma podemos imprimier los elementos almacenados en una  vaiable de nuestra clase
    print(lista.data[1]) # Al ser una lista podemos llamar atraves de su indice

    ## Implementación del metodod especial
    print(lista[2]) # Esto solo es valido cuando tenemos el metodo __getitem__ declarada en nuestra clase

if __name__ == "__main__":
    main()
```

En este caso estoy implementando el método especial call :

```python
#!/usr/bin/env python3

class Saludo:
    
    def __init__(self, saludo):
        self.saludo = saludo

    def __call__(self, nombre): # Este es un metodo especial el cual me permite devolver una llamada a la clase con lo que se a solicitado

        return f"{self.saludo} {nombre}!"

def main():
    hola = Saludo("!Hola")
    print(hola("Luis")) # Esta forma de llamar a al objeto solo es valida cuando empleamos metodos especiales como el de __call__

if __name__ == "__main__":
    main()
```

Estamos haciendo uso del método add el cual va en conjunto con el método str para operar correctamente:

```python
#!/urs/bin/env python3

class Punto:

    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __add__(self, object): # Como la suma es atraves de dos objetos vamos atener que representar los dos uno que es self y otro que es object

        return Punto(self.x + object.x, self.y + object.y) # De esta forma podemos crear un objeto temporar el cual se retorna atraves de esta función

    def __str__(self): # Si lo pensamos este metodo nos ayuda a retorna de manera correcta valores de un objeto cuando usamoe el return.

        return f"({self.x}, {self.y})" # Podemos retornar el objeto temporal ya que esta función se centra en todas las varialbles las cuales se retorna,
        # y al retornarce un objeto este metodo especial lo acoje y lo imprime de manera correcta.

def main():
    p1 = Punto(2, 8)
    p2 = Punto(4, 9)

    print(p1 + p2) # Esto no es valido a menos a menos que se implemente el metodo especial __add__ para poder operar la suma y __str__ para que python sepa como devolver el resultado

if __name__ == "__main__":
    main()
```

Empleamos métodos como los son el método iter y el método add podemos usar un objeto como un iterador:

```python
#!/usr/bin/evn python3

class Contador:
    
    def __init__(self, limite):
        self.limite = limite

    def __iter__(self): # Este metodo nos devuleve un iterable

        self.contador = 0
        return self

    def __next__(self): # Este metodo nos permite aumetar la vairble contador

        if self.contador < self.limite:
            self.contador += 1
            return self.contador
        else:
            raise StopIteration

def main():

    c = Contador(15)
    for i in c: # con esto podemos usar el objeto como un iterador.
        print(i)

if __name__ == "__main__":
    main()
```

# Decoradores y Propiedades

En esta clase, profundizaremos en estas poderosas características de Python que mejoran significativamente la forma en que podemos manejar y modificar el comportamiento de nuestras clases y funciones.
## **Decoradores**

Los decoradores en Python son una forma elegante de modificar las funciones o métodos. Se utilizan para extender o alterar el comportamiento de la función sin cambiar su código fuente. Un decorador es en sí mismo una función que toma otra función como argumento y devuelve una nueva función que, opcionalmente, puede agregar alguna funcionalidad antes y después de la función original.

**Propiedades**

Las propiedades son un caso especial de decoradores que permiten a los desarrolladores añadir “**getter**“, “**setter**” y “**deleter**” a los atributos de una clase de manera elegante, controlando así el acceso y la modificación de los datos. En Python, esto se logra con el decorador ‘**@property**‘, que transforma un método para acceder a un atributo como si fuera un atributo público.
## **Getters y Setters**

- El “**getter**” obtiene el valor de un atributo manteniendo el encapsulamiento y permitiendo que se ejecute una lógica adicional durante el acceso.
- El “**setter**” establece el valor de un atributo y puede incluir validación o procesamiento antes de que el cambio se refleje en el estado interno del objeto.
- El “**deleter**” puede ser utilizado para definir un comportamiento cuando un atributo es eliminado con del.

Durante la clase, discutiremos cómo los decoradores pueden ser aplicados no solo para métodos y propiedades, sino también para funciones en general. También exploraremos cómo las propiedades se pueden utilizar para crear una interfaz pública para atributos privados/ protegidos, mejorando la encapsulación y manteniendo la integridad de los datos de una clase.

Este conocimiento es crucial para escribir código Python idiomático y eficiente, aprovechando al máximo lo que el lenguaje tiene para ofrecer en términos de flexibilidad y potencia en el diseño de software

```python
#!/usr/bin/env python3

def mi_decorador(funcion): # esta es una funcion de orden superior

    def envoltura(): # Esta es una funcion con un nombre que nosotros queramos que nos permite alterar antes y despues de otra función

        print("Estoy saludo en la envoltura del decorador antes de llamar a la función") # Esto es antes de saludo
        funcion() # Esta es la funcion saludo que ahora toma el nombre de funcion
        print("Estoy saludo en la envoltura del decorador despues de llamar a la función") # Esto es despues de saludo

    return envoltura # se retorn la funcion envoltura ya que esta cuenta con las modificaciones

@mi_decorador # Esta es la forma de definir un decorador para una función
def saludo():

    print("Hola, estoy saludando dentro de la funció saludo")

def main():

    saludo()

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

class Persona:

    def __init__(self, nombre, edad):
        self._nombre = nombre # Esto es una varible protegida py aunque se pueda llamar no se debe hacer
        self._edad = edad

    @property # Nos permite crear una nueva propiedad, recordemos que cuando usamos este decorador lo llamamos como una propiedad
    def edad(self): # Getter, Con esto enviamos el valor de un atributo protegio o privado
        return self._edad

    @edad.setter # creacion de un setter, Este tipo de decorador al igual que con el getter lo llamamos como una propiedad y no como una funcioón
    def edad(self, edad): # creamos la funcion con la cual ya podemos asignar un valor internamente
        if edad > 0:
            self._edad = edad
        else:
            raise ValueError("[!] La edad no puede ser un numero menor a 0")

    # Tanto con getter como con setter vamos a mantener protegidos los valores originales de las variables protegidas o vaiables privadas

def main():
    manolo = Persona("Manolo", 23)
    manolo.edad = -1 # setter, nos permite ingresar un valor y lo hacemos como si fuera una pripiedad porque recordemos que no podemos llamarlo como un valor
    print(manolo.edad) # getter, nos premite ver o obtener el valor  y lo hacemos como si fuera una pripiedad porque recordemos que no podemos llamarlo como un valor

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

import time # de esta forma podemos importar librerias

def cronometro(function):
    def envoltura(num): # Cuando hacemos uso de docoradores con funciónes las cuales tienen parametros, tenemos que pasarle un parametro a envoltura y a la function

        inicio = time.time()
        function(num)
        final = time.time()
        
        resultado = final - inicio

        print(f"Tiempo total de ejecucion en la función {function.__name__}: {round(resultado, 1)}") # la funcion name nos permite imprimir el nombre de la funcion

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

```python
#!/usr/bin/env python3

# *args

def suma(*args): # Esto lo usamos especificamente cuando usamos de varios valores, ya que *args creara una tupla de estos
    return sum(args)

def main():
    print(suma(2, 3, 4, 6, 12, 14, 32))

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3

# **kwargs

def presentacion(**kwargs): # Con esto ya no almacenamos en una tupla sino en un diccionario

    for key, value in kwargs.items():
        print(f"{key} : {value}\\n")

def main():
    presentacion(nombre = "Daniel", edad=20, ciudad="Quito", profecion="Estudiante")

if __name__ == "__main__":
    main()

```

```python
#!/usr/bin/env python3

# **kwargs

def presentacion(**kwargs): # Con esto ya no almacenamos en una tupla sino en un diccionario

    for key, value in kwargs.items():
        print(f"{key} : {value}\\n")

def main():
    presentacion(nombre = "Daniel", edad=20, ciudad="Quito", profecion="Estudiante")

if __name__ == "__main__":
    main()
```

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
    print("\\n\\n")
    c.radio = 10
    print(c.radio)
    print(c.diametro)
    print(c.area)

if __name__ == "__main__":
    main()

```