-----
# **El intérprete de Python**

El intérprete de Python es el corazón del lenguaje de programación Python; es el motor que ejecuta el código que escriben los programadores. Cuando hablamos del “**intérprete de Python**“, nos referimos al programa que lee y ejecuta el código Python en tiempo real.
## **Funciones Clave del Intérprete de Python:**

- **Ejecución de Código**: El intérprete ejecuta el código escrito en Python línea por línea, lo que facilita la depuración y permite a los desarrolladores probar fragmentos de código de forma interactiva.
- **Modo Interactivo**: El intérprete puede usarse en un modo interactivo que permite a los usuarios ejecutar comandos de Python uno a uno y ver los resultados de inmediato, lo cual es excelente para el aprendizaje y la experimentación.
- **Modo de Script**: Además del modo interactivo, el intérprete puede ejecutar programas completos o scripts que se escriben en archivos con la extensión ‘**.py**‘.
- **Compilación a Bytecode**: Aunque Python es un lenguaje interpretado, internamente, el intérprete compila el código a bytecode antes de ejecutarlo, lo que mejora el rendimiento.
- **Máquina Virtual de Python**: El bytecode compilado se ejecuta en la Máquina Virtual de Python (Python Virtual Machine – PVM), que es una abstracción que hace que el código de Python sea portable y se pueda ejecutar en cualquier sistema operativo donde el intérprete esté disponible.
## **Ventajas del Intérprete de Python:**

- **Facilidad de Uso**: La capacidad de ejecutar código inmediatamente y de manera interactiva hace de Python una herramienta excelente para principiantes y para el desarrollo rápido de aplicaciones.
- **Portabilidad**: El intérprete de Python está disponible en múltiples plataformas, lo que significa que los programas de Python pueden ejecutarse en casi cualquier sistema sin modificaciones.
- **Extensibilidad**: El intérprete de Python permite la extensión con módulos escritos en otros lenguajes como C o C++, lo que puede ser utilizado para optimizar el rendimiento.

El intérprete de Python es una herramienta poderosa y flexible que hace que el lenguaje sea accesible y eficiente para una amplia variedad de aplicaciones de programación. Comprender cómo funciona el intérprete es fundamental para cualquier programador que desee dominar Python.

# Shebang y convenios en Python

En el desarrollo con Python, el shebang y los convenios de codificación son aspectos importantes que facilitan la escritura de scripts claros y portables.

**Shebang en Python:**

El shebang es una línea que se incluye al principio de un script ejecutable para indicar al sistema operativo con qué intérprete debe ejecutarse el archivo. En los scripts de Python, el shebang común es:

- **#!/usr/bin/env python3**

Esta línea le dice al sistema que utilice el entorno (**env**) para encontrar el intérprete de Python 3 y ejecutar el script con él. Es fundamental para asegurar que el script se ejecute con Python 3 en sistemas donde Python 2 todavía está presente.

**Convenios en Python:**

Los convenios de codificación son un conjunto de recomendaciones que guían a los desarrolladores de Python para escribir código claro y consistente. El más conocido es ‘**PEP 8**‘, que abarca:

- **Nombres de Variables**: Utilizar ‘**lower_case_with_underscores**‘ para nombres de variables y funciones, ‘**UPPER_CASE_WITH_UNDERSCORES**‘ para constantes, y ‘**CamelCase**‘ para clases.
- **Longitud de Línea**: Limitar las líneas a 79 caracteres para código y 72 para comentarios y docstrings.
- **Indentación**: Usar 4 espacios por nivel de indentación.
- **Espacios en Blanco**: Seguir las prácticas recomendadas sobre el uso de espacios en blanco, como no incluir espacios adicionales en listas, funciones y argumentos de funciones.
- **Importaciones**: Las importaciones deben estar en líneas separadas y agrupadas en el siguiente orden: módulos de la biblioteca estándar, módulos de terceros y luego módulos locales.
- **Compatibilidad entre Python 2 y 3**: Aunque Python 2 ha llegado al final de su vida útil, algunos convenios pueden seguirse para mantener la compatibilidad.

El cumplimiento de estos convenios no solo mejora la legibilidad del código, sino que también facilita la colaboración entre desarrolladores y el mantenimiento a largo plazo del software.

El uso adecuado del shebang y la adhesión a los convenios de Python son señales de un desarrollador cuidadoso y profesional. Integrar estos aspectos en tus prácticas de codificación es crucial para el desarrollo de software efectivo y eficiente en Python.

# Variables y Tipos de Datos

Las variables en Python son como nombres que se le asignan a los datos que manejamos. Piensa en una variable como un nombre que pones a un valor, para poder referirte a él y utilizarlo en diferentes partes de tu código.

En la clase actual, vamos a enfocarnos en comprender las variables y algunos de los tipos de datos fundamentales en Python. Estos conceptos son esenciales, ya que nos permiten almacenar y manipular la información en nuestros programas.
## **Variables**

Una variable en Python es como un nombre que se le asigna a un dato. No es necesario declarar el tipo de dato, ya que Python es inteligente para inferirlo.
## **Cadenas (Strings)**

Las cadenas son secuencias de caracteres que se utilizan para manejar texto. Son inmutables, lo que significa que una vez creadas, no puedes cambiar sus **caracteres individuales**.
## **Números**

Python maneja varios tipos numéricos, pero nos centraremos principalmente en:

- **Enteros (Integers)**: Números sin parte decimal.
- **Flotantes (Floats)**: Números que incluyen decimales.
## **Listas**

Las listas son colecciones ordenadas y mutables que pueden contener elementos de diferentes tipos. Son ideales para almacenar y acceder a secuencias de datos.

Y para trabajar con estas listas, así como con cadenas y rangos de números, utilizaremos los bucles ‘**for**‘, que nos permiten iterar sobre cada elemento de una secuencia de manera eficiente.

Estas son solo algunas de las estructuras de datos con las que trabajaremos por el momento. A medida que avancemos en las próximas clases, exploraremos más tipos de datos y estructuras más complejas, ampliando nuestras herramientas para resolver problemas y construir programas más sofisticados.

```python
#!/usr/bin/python3

################## Strings #####################

ip_address = "mi cadena" # Para definir las cadenas solo es nesesario los ""
print(ip_address)        # Imprimimos cadena
print(type(ip_address))  # Verificamos que sea String (STR)

################## Numeros #################

number = 20 #Definimos una variable de tipo numero pero entero
print(number) #Imprimimos el dato que creamos
print(type(number)) # Verificamos que sea de tipo entero(INT)

### Tipo flotante

number_float = 1.66666 # Definimos el dato de tipo flotante o (float)
print(number_float) # Imprimimos el dato que creamos
print(type(number_float) ) # Verificamos que se a de tipo Flotante(float)

################### Boleanos #################

bolean_dato = True # Definimos el dato de tipo bolean, Solo tiene dos tipos (True/False)
print(bolean_dato) # Imprimimos el dato que creamos
print(type(bolean_dato)) # Verificamos que sea de tipo booleano

################## Cambios de Datos ####################!/usr/bin/env python3

numero_string = "10"
print(type(numero_string))

numero_string = int(numero_string)
print(type(numero_string))

numero_string = float(numero_string / 4)
print(type(numero_string) )

############## Listas #################3 

my_ports = [] #Declarando una Lista vacia
my_ports2 = [1, 2, 22, 3, 4, 5, 22] # Declarando una lista llena

my_ports.append(22) # Incertamos un valor en la lista
my_ports.append(40)
my_ports.append(22) # Incertamos un valor en la lista
my_ports.append(25)
my_ports.append(22) # Incertamos un valor en la lista
my_ports.append(80)
my_ports.append(22) # Incertamos un valor en la lista

my_ports2.extend([84, 85]) #Agregando mas valores a la lista
my_ports2 += [86,87] # Agrengaod mas valores a al lista

del my_ports[0] # Eliminado del indice 0

print(my_ports) # Imprimiendo toda la lista

print(my_ports[0]) # Imprimiendo solo el elemento 0
print(my_ports[1]) # Imprimiendo solo el elemento 1
print(my_ports[2]) # Imprimiendo solo el elemento 2
print(my_ports[3]) # Imprimiendo solo el elemento 3

print(len(my_ports)) # Nos permite ver el tamaño de la lista

for port in my_ports: # con un bucle es mas facil recorrer los elementos de la lista
    print(f"Puerto: {port}")

my_ports2 = sorted(my_ports2) # Nos permite ordenar la lista.
print(my_ports2[:2]) # con esto nosostros imprimimos solo los 2 primeros elementos
print(my_ports2[0:3]) # Con esto nos imprime del primer al 3-1 elementos, si hablamso de indices
print(my_ports2[2:4]) # Nos muestra del indice 2 al 4-1

print(my_ports2[2:]) # Nos muestra desde el segundo elemento sin incluirlo.
print(my_ports2[-1]) # Con esto nosotros podemos listaar desde el ultimo.
print(my_ports2[-2:]) # desde el ante penultimo elemento en adelante
print(my_ports2[:-2]) # Imprime del todo menos los dos ultimos elementos

# Las listas pueden tener cualquier tipo de dato. 

my_ports2.pop() # Elimina el ultimo elemento de la lista.

my_ports2.index(22) #Nos permite saver el indice de un elemento perteneciente a la lista, en el caso de tener elementos repetido nos indica el indice del primer elemento encontrado

enumerate(my_ports2) # Este objeto nos devuelve dos valoes, primero el indice y luego sus valores

for x,y in enumerate(my_ports2):
    print(x) # Imprime primero el indice.
    print(y) # Inprime segundo el elemeto.

indices = [x for x, y in enumerate(my_ports2) if y == 22] # Esta es una función lambda con la cual cramos una lista en donde guardamos los indices de los elementos donde tenemos un 22  

my_ports2.count(22) # Nos permite contar el numeo de veses que se repite un valor en una lista

set(my_ports2) # Este es otro tipo de dato, en este no se puede tener valores repetidos por lo que podemos cambiar a este dato para eliminar repetidos y luego regresar a lista
list(my_ports2)

max(my_ports2) # Nos permite obtener el valor mas alto de la lista.
min(my_ports2) # Nos permite obtener el valor mas bajo de la lista
sum(my_ports2) # Nos premite obtener la suma de toda la lista.
round(10.5) # Nos permite obtener el redondeo de un valor decimal o float

###################
```

# Operadores

Los operadores aritméticos son símbolos que Python utiliza para realizar cálculos matemáticos.
## Los fundamentales son:

- **Suma (+)**: No solo suma números, sino que también une secuencias como cadenas y listas, creando una nueva secuencia que es la combinación de ambas.
- **Resta (-)**: Se utiliza para restar un número de otro. Con listas, su uso es menos directo y generalmente no se aplica como operador directo.
- **Multiplicación (*)**: Cuando se multiplica un número por otro, obtenemos el producto. Con cadenas y listas, este operador repite los elementos la cantidad de veces especificada.
- **División (/)**: Divide un número entre otro y el resultado es siempre un número flotante, incluso si los números son enteros.
- **Exponente (**):** Eleva un número a la potencia de otro. Por ejemplo, ‘**2 ** 3**‘ resultará en 8. Este operador es menos común en operaciones con cadenas o listas.
## **Operaciones con Cadenas**

En Python, las cadenas son objetos que representan secuencias de caracteres y se pueden manipular usando operadores aritméticos:

- **Concatenación (+)**: Une varias cadenas en una sola. Por ejemplo, ‘Hola’ + ‘ ‘ + ‘Mundo’ se convierte en ‘Hola Mundo’.
- **Repetición (*)**: Crea repeticiones de la misma cadena. ‘Hola’ * 3 generará ‘HolaHolaHola’.
## **Operaciones con Listas**

Las listas son colecciones ordenadas y mutables de elementos:

- **Concatenación (+)**: Similar a las cadenas, unir dos listas las combina en una nueva lista.
- **Repetición (*)**: Repite todos los elementos de la lista un número determinado de veces.
## **Funciones Especiales para Listas**

- **Zip**: Toma dos o más listas y las empareja, creando una lista de tuplas. Cada tupla contiene elementos de las listas originales que ocupan la misma posición.
- **Map**: Aplica una función específica a cada elemento de un iterable, lo que resulta útil para transformar los datos contenidos.

Asimismo, otro de los conceptos que mencionamos es el de ‘**TypeCast**‘. El TypeCast, o conversión de tipo, es el proceso mediante el cual se cambia una variable de un tipo de dato a otro.

En Python, esto se realiza de manera muy directa, utilizando el nombre del tipo de dato como una función para realizar la conversión. Por ejemplo, convertir una cadena a un entero se hace pasando la cadena como argumento a la función **int()**, y transformar un número a una cadena se hace con la función **str()**. Esta capacidad de cambiar el tipo de dato es especialmente útil cuando se necesita estandarizar los tipos de datos para operaciones específicas o para cumplir con los requisitos de las estructuras de datos.

A medida que progresemos, ampliaremos nuestro repertorio para incluir operaciones más complejas y explorar otros tipos de datos y estructuras en Python.

```python
#!/usr/bin/python3

first_number = 5
second_number = 8

result = first_number + second_number # Suma
result = first_number - second_number # Resta
result = first_number * second_number # Multiplicación
result = first_number / second_number # Divición
result = first_number % second_number # Reciduo de la divición

potencia = first_number ** 2 # Esto se lee 5 al cuadrado.
potencia2 = 12 ** 7

print("{:,}".format(potencia2)) # Podemos formatear la cadena y decirle que nos ponga comas para representar mejor los valores numericos

# Digamos que no nos gustan las comas y queremos poner puntos, para esto hacemos lo siguiente:

print("{:,}".format(potencia2).replace(",",".")) # Como vemos con ayuda de replace cambiamos las comas por puntos

#### Operatorias con strings

first_str= "hola"
second_str = " "
third_str = "mundo"

frace = first_str + second_str + third_str
print(frace)

print(first_str*3) # Nos repite el valor del str por 3
print(third_str[:3]) # A los estrings tambien podemos tratarles como listas

###########################

odd_numbers= [1, 3, 5, 7, 9]
eve_numbers= [2, 4, 6, 8, 10]

result = odd_numbers + eve_numbers 

print(result)

result2 =zip(odd_numbers, eve_numbers) # Esto nos devuelve una lista de elementos en formato tupla formando la tupla con la misma posición

for elements in result2:
    print(elements)

result2 = list(map(sum, zip(odd_numbers, eve_numbers))) # map nos permite sumar con sum los valores de las tuplas y hacemos listas
print(result2)
```

# String Formating

Python proporciona varias maneras de formatear cadenas, permitiendo insertar variables en ellas, así como controlar el espaciado, alineación y precisión de los datos mostrados. Aquí están las técnicas de formateo de cadenas que exploraremos:
## **Operador % (Porcentaje)**

También conocido como “i**nterpolación de cadenas**“, este método clásico utiliza marcadores de posición como ‘**%s**‘ para cadenas, ‘**%d**‘ para enteros, o ‘**%f**‘ para números de punto flotante.
## **Método format()**

Introducido en Python 2.6, permite una mayor flexibilidad y claridad. Utiliza llaves ‘**{}**‘ como marcadores de posición dentro de la cadena y puede incluir detalles sobre el formato de la salida.

## **F-Strings (Literal String Interpolation)**

Disponible desde Python 3.6, los F-Strings ofrecen una forma concisa y legible de incrustar expresiones dentro de literales de cadena usando la letra ‘**f**‘ antes de las comillas de apertura y llaves para indicar dónde se insertarán las variables o expresiones.

En esta clase, nos enfocaremos en cómo utilizar cada uno de estos métodos para formatear cadenas efectivamente, así como las situaciones en las que cada uno podría ser más apropiado. Al final, tendrás las herramientas para presentar información de manera profesional en tus programas de Python.

```python
#!/usr/bin/env python3

name = "stark"
rol = "hacker"
edad = 21

# Esta es una forma de formatear la cadena y poner cualquier valor
print(f"Hola mi nombre es {name}")

# Esta es otra forma de formatear la cadena  
print("Hola, mi nombre es %s y soy un %s. Actualmente tengo %d" % (name, rol, edad)) # usamos el %s ya que el dato es de tipo string, si es de tipo int seria %d

# Esta es otra forma de formatear la cadena
print("Hola, soy {} y tengo {} años".format(name, edad)) # Con el .format pasamos las variables que se rellenaran en orden
print("Hola, soy {0} y tengo {1} años y mi nombre real es {0}".format(name, edad)) # Con el .format pasamos las variables que se rellenaran en orden, si queremos reusar variables usamos indices
```

# Control de Flujo

Los conceptos vistos en esta clase son esenciales para entender cómo crear programas en Python que puedan tomar decisiones y repetir acciones hasta cumplir ciertos criterios. Aquí es donde nuestros programas obtienen la capacidad de responder a diferentes situaciones y datos.
## **Condicionales**

Los condicionales son estructuras de control que permiten ejecutar diferentes bloques de código dependiendo de si una o más condiciones son verdaderas o falsas. En Python, las declaraciones condicionales más comunes son ‘**if**‘, ‘**elif**‘ y ‘**else**‘.

- **if**: Evalúa si una condición es verdadera y, de ser así, ejecuta un bloque de código.
- **elif**: Abreviatura de “**else if**“, se utiliza para verificar múltiples expresiones sólo si las anteriores no son verdaderas.
- **else**: Captura cualquier caso que no haya sido capturado por las declaraciones ‘**if**‘ y ‘**elif**‘ anteriores.
## **Bucles**

Los bucles permiten ejecutar un bloque de código repetidamente mientras una condición sea verdadera o para cada elemento en una secuencia. Los dos tipos principales de bucles que utilizamos en Python son ‘**for**‘ y ‘**while**‘.

- **for**: Se usa para iterar sobre una secuencia (como una lista, un diccionario, una tupla o un conjunto) y ejecutar un bloque de código para cada elemento de la secuencia.
- **while**: Ejecuta un bloque de código repetidamente mientras una condición específica se mantiene verdadera.
## **Control de Flujo en Bucles**

Existen declaraciones de control de flujo que pueden modificar el comportamiento de los bucles, como ‘**break**‘, ‘**continue**‘ y ‘**pass**‘.

- **break**: Termina el bucle y pasa el control a la siguiente declaración fuera del bucle.
- **continue**: Omite el resto del código dentro del bucle y continúa con la siguiente iteración.
- **pass**: No hace nada, se utiliza como una declaración de relleno donde el código eventualmente irá, pero no ha sido escrito todavía.

En esta clase, profundizaremos en cada uno de estos aspectos con ejemplos detallados. Aprenderemos cómo tomar decisiones dentro de nuestros programas y cómo automatizar tareas repetitivas. Esto nos dará la base para escribir programas que pueden manejar tareas complejas y responder dinámicamente a los datos de entrada. Al final de la clase, estarás equipado con el conocimiento para controlar el flujo de tus programas de Python de manera eficiente y efectiva.

```python
#!/usr/bin/env python3

## Operadores ##
#    <=
#    >=
#    ==
#    !=
#    <
#    >
#    and
#    or
#
################

for i in range(5): # Esta es la estructura de un bucle for, el range va desde 0 a 4 dandones 5 valores
    print(i)

# Podemos iterar sobre varias cosas como: Listas, tuplas, diccionarios, strings y el concepto siempre sera el mismo.

names = ["Marcelo", "Lobotec", "HackerMate", "TheGoodHacker"]
for i in names:
    print(f"name: {i}")

for indice,nombre in enumerate(names): # Tenemos que saver que enumerate nos devuleve dos valores que son el iterador y su valores en ese orden
    print(f"Indice: {indice} -> Valor: {nombre}")

frutas = {"manzana": 1, "peras": 5, "platanos": 7} # Esto es la definición de un diccionarios el cual tiene una clave y un valor en ese orden

for fruta,cantidad in frutas.items(): # esta es la forma para iterar tanto por su clave como por su valor en diccionarios
    print(f"Tenemos {cantidad} de {fruta}")

########### Bucle While ########

i=0
while i < 5:  # Esta es la estructura de un bucle while, recordar siempre incrementar nuestro indicador para que no sea un bucle infinito
    print(i)
    i+=1

###### Else en el bucle ##########
# Tenemos que saver que un else luego de un bucle siempre se ejecutara amenos que este sea interrumpido
while i<10:
    if i==10:
        print("El bucle termina antes de tiempo")
        break

    i+=1
else:
    print("El bucle termino correctamente")

############  Bucles Anidados #########

my_list = [[1,4,5], [2,4,8], [9,4,1]]

for elemento in my_list:
    print(f"[+] Vamos a desglozar la lista {elemento}")
    for valores in elemento:
        print(valores)

############# Listas de Compresión (for) ###########
odd_list = [1, 3, 5, 7, 9]

cuadrado = [i**2 for i in odd_list] #Esta es la forma de crear una lista de comprensión

print(cuadrado)

############# Salir antes de tiempo de un bucle ##########

for i in range(10):
    print(i)

    if i == 5:
        break       #A través de break nosotros podemos romper un bucle y salir cuando nosotros queramos

for i in range(10):
    if i == 5:
        continue     # A través de continue nosotros podemos envitar ejecutar cierto codigo y continuar con lo demas

    print(i)

############## if else / elif #############

for i in range(20):
    if i==5:    # esta es la estructura de un condicional
        print(f"Este es una valor {i}")

    else:
        print(f"Esto no es un 5")

if i == 10:
    print(f"El valor es igual a 10")
elif i==5:      # El elif no es mas que un else al cual le podemos asosciar una condición al igual que al if
    print(f"El valor es igual a 5")
else:
    print(f"El valor es igual a {i}")

######## condicional en una linea u Operadores Ternarios ########

edad=20
mensaje = "Eres mayor de edad" if edad >= 18 else "Eres menor de edad"
print(f"Tu tienes {edad} y {mensaje}")

############# else en bucles #############

for i in range (20):
    if i==20:
        break

else:
    print("El bucle termino correctamente") # un else en un bucle va a ejecutarse siempre que que el mismo concluya de manera exitosa

####### Determinar valores sin iterar ########
numeros = [1, 2, 4, 6, 6, 7, 9]

if 6 in numeros:
    print(f"el número 6 esta en la lista")
else:
    print("El número 6 no esta en la lista")

#############  Condicional anidados #########
numero=10
numero2=11

if numero == 10:
    pass # esto es equivalente a un continuo
    if numero2 == 11:
        print("Los condicionales estan anidados")
```

# Funciones y ámbito de las variables

En esta clase nos sumergimos en dos conceptos fundamentales de la programación en Python que potencian la modularidad y la gestión eficaz de los datos dentro de nuestros programas.
## **Funciones**

Las funciones son bloques de código reutilizables diseñados para realizar una tarea específica. En Python, se definen usando la palabra clave ‘**def**‘ seguida de un nombre descriptivo, paréntesis que pueden contener parámetros y dos puntos. Los parámetros son “**variables de entrada**” que pueden cambiar cada vez que se llama a la función. Esto permite a las funciones operar con diferentes datos y producir resultados correspondientes.

Las funciones pueden devolver valores al programa principal o a otras funciones mediante la palabra clave ‘**return**‘. Esto las hace increíblemente versátiles, ya que pueden procesar datos y luego pasar esos datos modificados a otras partes del programa.

## **Ámbito de las Variables (Scope)**

El ámbito de una variable se refiere a la región de un programa donde esa variable es accesible. En Python, hay dos tipos principales de ámbitos:

- **Local**: Las variables definidas dentro de una función tienen un ámbito local, lo que significa que solo pueden ser accesadas y modificadas dentro de la función donde fueron creadas.
- **Global**: Las variables definidas fuera de todas las funciones tienen un ámbito global, lo que significa que pueden ser accesadas desde cualquier parte del programa. Sin embargo, para modificar una variable global dentro de una función, se debe declarar como global.

Durante esta clase, exploraremos cómo definir y llamar funciones, cómo pasar información a las funciones a través de argumentos, y cómo las variables interactúan con diferentes ámbitos. También veremos las mejores prácticas para definir funciones claras y concisas y cómo el correcto manejo del ámbito de las variables puede evitar errores y complicaciones en el código.

Comprender estos conceptos es esencial para escribir programas claros, eficientes y mantenibles en Python. Al finalizar la clase, tendrás una sólida comprensión de cómo estructurar tu código y cómo gestionar las variables para que tus programas funcionen de manera impecable.

Las funciones son bloques de código, los cuales son reutilizables y que realizan un tarea especifica, Para definir una función la sintaxis es la siguiente:

```python
#!/usr/bin/env python3

def saludo(): ## Definiendo a la funciónes
    print("\\nHola mundo")

saludo() ## Llamando a la función

#########################

def saludo2(nombre): # Definiendo una función en base a una variable
    print(f"Hola {nombre}")

saludo2("Daniel") # LLamando a la función y pasando un paramentro

#############################
def suma(n1, n2): # Definiendo una función con varias variables
    print(f"La suma total es de {n1 + n2}")

suma(5,10) # llamando a la función con sus parametros

#############################
def multiplicacion(n1, n2): # Definiendo la función
    return n1+n2            # return nos regresa nos envia nos retorna el valor o valores que nosostros dispongamos, en este caso la suma

print(f"La suma de los valores es {multiplicacion(10,8)}")

############# Hambitos de la Varibles ####################

variable_global = "Soy una variable Global" # Este tipo de variables se pueden usar tanto dentro como fuera de la función.

def mi_funcion():
    global varible_local # De esta forma podemos pasar las variables locales a globales.
    variable_local = "Soy una variable local" # Este es un tipo de variable la cual solo existe dentro de nuestra función y no fuera de la misma.
    print(variable_local)

mi_funcion()

##############################################################
```

# Funciones lambda anónimas

Esta clase se centra en una característica poderosa y expresiva de Python que permite la creación de funciones en una sola línea: las funciones lambda.
## **Funciones Lambda**

Las funciones lambda son también conocidas como funciones anónimas debido a que no se les asigna un nombre explícito al definirlas. Se utilizan para crear pequeñas funciones en el lugar donde se necesitan, generalmente para una operación específica y breve. En Python, una función lambda se define con la palabra clave ‘**lambda**‘, seguida de una lista de argumentos, dos puntos y la expresión que desea evaluar y devolver.

Una de las ventajas de las funciones lambda es su simplicidad sintáctica, lo que las hace ideal para su uso en operaciones que requieren una función por un breve momento y para casos donde la definición de una función tradicional completa sería excesivamente verbosa.
## **Usos comunes de las Funciones Lambda**

- **Con funciones de orden superior**: Como aquellas que requieren otra función como argumento, por ejemplo, ‘**map()**‘, ‘**filter()**‘ y ‘**sorted()**‘.
- **Operaciones simples**: Para realizar cálculos o acciones rápidas donde una función completa sería innecesariamente larga.
- **Funcionalidad en línea**: Cuando se necesita una funcionalidad simple sin la necesidad de reutilizarla en otro lugar del código.

En esta clase, aprenderemos cómo y cuándo utilizar las funciones lambda de manera efectiva, además de entender cómo pueden ayudarnos a escribir código más limpio y eficiente. Aunque su utilidad es amplia, también discutiremos las limitaciones de las funciones lambda y cómo el abuso de estas puede llevar a un código menos legible.

Al dominar las funciones lambda, ampliarás tu conjunto de herramientas de programación en Python, permitiéndote escribir código más conciso y funcional.

Este tipo de funciones nos permite simplificar una función normal la cual no tiene nombre (por eso anónima) y hacerla mucho mas pequeña y sencilla.

Las funciones lambda se realizan de la siguiente manera:

```python
#!/usr/bin/env python3

funcion = lambda: "Hola mundo"   # De esta forma definimos las funciones lambda
print(funcion()) # la variable que almacena la funcion lambda tiene que definirse como una función

cuadrado = lambda x: x**2 # De esta manera podemos pasarle variables a nuestras funciónes lambda
print(cuadrado(6))

###############

numeros = [1, 2, 3, 4, 5]

cuadrados = list(map(lambda x: x**2, numeros))  # La funcion lambda eleva los valores, con map le pasamos un iterable que es la lista y luego convierto todo a una lista.
print(cuadrados)

################

numeros2 = [1, 2, 3, 4, 5]

cuadrados2 = list(filter(lambda x: x % 2 != 0, numeros2))

print(cuadrados2)
```

# Manejo de errores y excepciones

En esta clase, abordaremos el manejo de errores y excepciones, un aspecto crítico para la creación de programas robustos y confiables en Python. Los errores son inevitables en la programación, pero manejarlos correctamente es lo que diferencia a un buen programa de uno que falla constantemente.
## **Manejo de Errores**

Los errores pueden ocurrir por muchas razones: errores de código, datos de entrada incorrectos, problemas de conectividad, entre otros. En lugar de permitir que un programa falle con un error, Python nos proporciona herramientas para ‘atrapar’ estos errores y manejarlos de manera controlada, evitando así que el programa se detenga inesperadamente y permitiendo reaccionar de manera adecuada.
## **Excepciones**

Una excepción en Python es un evento que ocurre durante la ejecución de un programa que interrumpe el flujo normal de las instrucciones del programa. Cuando el intérprete se encuentra con una situación que no puede manejar, ‘levanta’ o ‘arroja’ una excepción.
### **Bloques try y except**

Para manejar las excepciones, utilizamos los bloques ‘**try**‘ y ‘**except**‘. Un bloque ‘**try**‘ contiene el código que puede producir una excepción, mientras que un bloque ‘**except**‘ captura la excepción y contiene el código que se ejecuta cuando se produce una.
### **Otras Palabras Clave de Manejo de Excepciones**

- **else**: Se puede usar después de los bloques ‘**except**‘ para ejecutar código si el bloque ‘**try**‘ no generó una excepción.
- **finally**: Se utiliza para ejecutar código que debe correr independientemente de si se produjo una excepción o no, como cerrar un archivo o una conexión de red.
### **Levantar Excepciones**

También es posible ‘levantar’ una excepción intencionalmente con la palabra clave ‘**raise**‘, lo que permite forzar que se produzca una excepción bajo condiciones específicas.

En esta clase, aprenderemos a identificar diferentes tipos de excepciones y cómo manejarlas de manera específica. También exploraremos cómo utilizar la declaración ‘**raise**‘ para crear excepciones que ayuden a controlar el flujo del programa y evitar estados erróneos o datos corruptos.

Al final de esta clase, tendrás las habilidades para escribir programas que manejen situaciones inesperadas de manera elegante y mantengan una ejecución limpia y controlada, incluso cuando se encuentren con problemas imprevistos.

A lo que nos referimos por el manejo de errores es al controlar la salida o como se describe ese error, ser mas descriptivo para el usuario y evitar que nuestros programas tengan problemas o cerrados absurdos.

```python
#!/usr/bin/env python3

try: # tenemos que leerlo como: si al ejecutar lo siguiente da error
    num = 5/0

except: # Entonces ejecutame lo siguiten
    print("No se puede dividir un valor entre cero!!!!")

# Podemos ser mucho mas especificos al momento de manejar excepciones

try:
    num = "hola"/5

except TypeError: # En este punto nos encargamos de especifiar el erro que esperamos.
    print("Solo es posible dividir numeros")

# En el manejo de execpciones podemos aumentar un else el cual se ejecutara siempre y cuando no se genere una execpción
try:
    num = 20/5

except TypeError: # En este punto nos encargamos de especifiar el erro que esperamos.
    print("Solo es posible dividir numeros")

else: 
    print("la divicicón si se realiza")

# Tenemos otro concepto que es el finally el cual siempre se ejecuta

try: # tenemos que leerlo como: si al ejecutar lo siguiente da error
    num = 5/0

except: # Entonces ejecutame lo siguiten
    print("No se puede dividir un valor entre cero!!!!")

finally:
    print("Este código siempre se ejecuta sin importar nada")

# Por ultimo las excepciones las poddemos lanzar de la siguiente forma:

x= -5
if x < 0:
    raise Exception("NO es posible usar numeros Negativos!") # esto es lo que se conoce como un stac trace y siempre es nesesario el raise

```