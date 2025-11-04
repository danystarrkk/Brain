-----
# Listas en Python

En esta clase, nos sumergiremos en profundidad en uno de los tipos de datos más versátiles y utilizados en Python.

Las listas son estructuras de datos que nos permiten almacenar secuencias ordenadas de elementos. Son mutables, lo que significa que podemos modificar después de su creación, y son dinámicas, permitiéndonos añadir o quitar elementos de ellas.
## **Características de las Listas**

Vamos a explorar las características clave de las listas en Python, que incluyen su capacidad para:

- Almacenar datos heterogéneos, es decir, pueden contener elementos de diferentes tipos (enteros, cadenas, flotantes y más) dentro de una misma lista.
- Ser indexadas y cortadas, lo que permite acceder a elementos específicos de la lista directamente a través de su índice.
- Ser anidadas, es decir, una lista puede contener otras listas como elementos, lo que permite crear estructuras de datos complejas como matrices.

## **Operaciones con Listas**

También cubriremos las operaciones fundamentales que se pueden realizar con listas, como:

- Añadir elementos con métodos como ‘**append()**‘ y ‘**extend()**‘.
- Eliminar elementos con métodos como ‘**remove()**‘ y ‘**pop()**‘.
- Ordenar las listas con el método ‘**sort()**‘ o la función incorporada ‘**sorted()**‘.
- Invertir los elementos con el método ‘**reverse()**‘ o la sintaxis de corte ‘**[::-1]**‘.
- Comprender las comprensiones de listas, una forma “pythonica” de crear y manipular listas de manera concisa y eficiente.
## **Métodos de Listas**

Profundizaremos en la rica gama de métodos que Python ofrece para trabajar con listas y cómo estos métodos pueden ser utilizados para manipular listas de acuerdo a nuestras necesidades.

**Buenas Prácticas**

Discutiremos las mejores prácticas en el manejo de listas, incluyendo cómo y cuándo usar listas en comparación con otros tipos de colecciones en Python, como tuplas, conjuntos y diccionarios.

Al final de esta clase, tendrás un conocimiento profundo de las listas en Python y estarás equipado con las técnicas para manejarlas eficazmente en tus programas. Con esta base sólida, podrás manipular colecciones de datos con confianza y aplicar esta habilidad central en tareas como la manipulación de datos, la automatización y el desarrollo de algoritmos.

```python
#!/usr/bin/env Python3

puertos_tcp = [22, 21, 25, 80, 443, 8080, 445, 69]

puertos_tcp.append(1337) # agregando un puerto nuevo a la lista
puertos_tcp[2] = 50 # Con esto logramos modificar el valor que corresponde al indice 2
puertos_tcp.insert(4, 25) # Con esto incertamos un valor en la posicion 4 con valor 25

print(puertos_tcp)

### Iterar en una lista

for puerto in puertos_tcp:
    print(f"puerto {puerto}")

####
puertos_tcp.remove(22) # Eliminamos elemento de la lista
print(puertos_tcp)

####

puertos_tcp.sort() #Nos permite realizar un ordenamiento a la lista
print(puertos_tcp)

#######

puertos_tcp.reverse() # Nos permite reversear la lista
print(puertos_tcp)

#######
print(f"puerto {puertos_tcp[0]}") # Accediendo a la lista por su indice
print(f"puerto {puertos_tcp[:3]}") # Accediendo a la lista por su indice, pero solo los 3 primeros
print(f"puerto {puertos_tcp[3:]}") # Accediendo a la lista por su indice, pero apartir del tercer valor 
print(f"puerto {puertos_tcp[-1]}") # Accediendo a la lista por su indice, pero nos muestra el ultimo elemento
print(f"puerto {puertos_tcp[1:2]}") # Accediendo a la lista por su indice, pero nos muestra desde el elemento 1 al elemento 2
print(f"puerto {puertos_tcp[1:2]}") # Accediendo a la lista por su indice, pero nos muestra desde el elemento 1 al elemento 2

########
len(puertos_tcp) # Esto nos devuelv el numero total de elemtnos dentro de la lista
puertos_tcp.clear() # Con esto logramos limpiar la lista

#########

enumerate(puertos_tcp) # Esto nos devueve tanto el indice como el valor, esto es ultil en los bucles.
puertos_tcp.pop() # Esto nos elimina el ultimo elemento
puertos_tcp.pop(22) # Tambien podemos eliminar una variable especifica, esta variable la podemos almacenar

##########

newList = [port**2 for port in puertos_tcp] # Generamos una nueva lista con todos los valores con potencia 2

#puertos_tcp.lower() # Nos permite poner todo en minusculas, incluyendo a los strings de las listas
#puertos_tcp.upper() # Nos pemite poner todo en mayúsculas, incluyendo a los strings de las listas

########### Combiando listas
names = ["Daniel", "Danna", "Vivi", "Andres", "Dome", "Dylan"]
edades = [20, 6, 40, 40, 9, 9]
more_names = ["Jaime", "Raquel", "Mateo", "Camila"]

for name, edad in zip(names, edades): # Con zip podemos combinar listas para poder iterar en ellas
    print(f"{name} tiene {edad} años")

############# Extendiendo lista
names.extend(more_names) # Nos permite extender una lista a través de otra
```
# Tuplas

En esta clase, dedicaremos nuestro enfoque a las tuplas, una estructura de datos fundamental en Python que comparte algunas similitudes con las listas, pero se distingue por su inmutabilidad.

Las tuplas son colecciones ordenadas de elementos que no pueden modificarse una vez creadas. Esta característica las hace ideales para asegurar que ciertos datos permanezcan constantes a lo largo del ciclo de vida de un programa.
## **Características de las Tuplas**

- **Inmutabilidad**: Una vez que se crea una tupla, no puedes cambiar, añadir o eliminar elementos. Esta inmutabilidad garantiza la integridad de los datos que desea mantener constantes.
- **Indexación y Slicing**: Al igual que las listas, puedes acceder a los elementos de la tupla mediante índices y también puedes realizar operaciones de slicing para obtener subsecuencias de la tupla.
- **Heterogeneidad**: Las tuplas pueden contener elementos de diferentes tipos, incluyendo otras tuplas, lo que las hace muy versátiles.
## **Operaciones con Tuplas**

Aunque no puedes modificar una tupla, hay varias operaciones que puedes realizar:

- **Empaquetado y Desempaquetado de Tuplas**: Las tuplas permiten asignar y desasignar sus elementos a múltiples variables de forma simultánea.
- **Concatenación y Repetición**: Similar a las listas, puedes combinar tuplas usando el operador ‘**+**‘ y repetir los elementos de una tupla un número determinado de veces con el operador ‘****‘.
- **Métodos de Búsqueda**: Puedes usar métodos como ‘**index()**‘ para encontrar la posición de un elemento y ‘**count()**‘ para contar cuántas veces aparece un elemento en la tupla.
## **Uso de Tuplas en Python**

- **Funciones y Asignaciones Múltiples**: Las tuplas son muy útiles cuando una función necesita devolver múltiples valores o cuando se realizan asignaciones múltiples en una sola línea.
- **Estructuras de Datos Fijas**: Se usan para crear estructuras de datos que no deben cambiar, como los días de la semana o las coordenadas de un punto en el espacio.
## **Buenas Prácticas**

Abordaremos cuándo es más apropiado utilizar tuplas en lugar de listas y cómo la elección de una tupla sobre una lista puede afectar la claridad y la seguridad del código.

Al concluir esta clase, tendrás un entendimiento claro de qué son las tuplas, cómo y cuándo utilizarlas en tus programas, y las prácticas recomendadas para trabajar con este tipo de datos inmutable. Las tuplas son una herramienta poderosa en Python, y saber cómo utilizarlas te permitirá escribir código más seguro y eficiente

```python

tupla = (1, 2, 3, 4, 5, 6) # Definiendo una tupla
tupla2 = (5,) # Al definir una tupla de un elemento se deve incluir una coma
print(type(tupla))

# En el caso de uso es casi similar a las listas, pero nos son mutables por lo que lo siguiente no se puede hacer
# insert(), extend(), remove(), appden()

#######################

mi_tupla = (1, 2, 3, 4)
a, b, c, d = mi_tupla

print(a)
print(b)
print(c)
print(d)

#### Modificar tupla de forma indirecta ######
mi_primera_tupla = (1, 2, 3, 4)
mi_segunda_tupla = (5, 6, 7, 8, 9)
mi_tercera_tupla = mi_primera_tupla + mi_segunda_tupla

print(mi_tercera_tupla)

# Por lo tanto para agregar valores a tuplas se deve crear una nueva y sumarla u operar pero entre tuplas
mi_tupla1 = (1, 2, 3 ,4, 5, 6, 7, 8, 9)

numeros_pares = tuple(i for i in mi_tupla1 if i %2 == 0) # Atraves de una lambda fuction podemos llegar a crear una nueva tupla
print(numeros_pares)

## Las tuplas las podemos usar en un programa el cual tenga usuario y a este asignarle el valor de tupla para así evitar cambios no autorizados a los usuarios
```

# Conjuntos o Sets

En esta clase, nos adentraremos en los conjuntos, conocidos en Python como ‘**sets**‘. Los conjuntos son una colección de elementos sin orden y sin elementos repetidos, inspirados en la teoría de conjuntos de las matemáticas. Son ideales para la gestión de colecciones de elementos únicos y operaciones que requieren eliminar duplicados o realizar comparaciones de conjuntos.
## **Características de los Conjuntos**

- **Unicidad**: Los conjuntos automáticamente descartan elementos duplicados, lo que los hace perfectos para recolectar elementos únicos.
- **Desordenados**: A diferencia de las listas y las tuplas, los conjuntos no mantienen los elementos en ningún orden específico.
- **Mutabilidad**: Los elementos de un conjunto pueden ser agregados o eliminados, pero los elementos mismos deben ser inmutables (por ejemplo, no puedes tener un conjunto de listas, ya que las listas se pueden modificar).
## **Operaciones con Conjuntos**

Exploraremos las operaciones básicas de conjuntos que Python facilita, como:

- **Adición y Eliminación**: Añadir elementos con ‘**add()**‘ y eliminar elementos con ‘**remove()**‘ o ‘**discard()**‘.
- **Operaciones de Conjuntos**: Realizar uniones, intersecciones, diferencias y diferencias simétricas utilizando métodos o operadores respectivos.
- **Pruebas de Pertenencia**: Comprobar rápidamente si un elemento es miembro de un conjunto.
- **Inmutabilidad Opcional**: Usar el tipo ‘**frozenset**‘ para crear conjuntos que no se pueden modificar después de su creación.
## **Uso de Conjuntos en Python**

- **Eliminación de Duplicados**: Son útiles cuando necesitas asegurarte de que una colección no tenga elementos repetidos.
- **Relaciones entre Colecciones**: Facilitan la comprensión y el manejo de relaciones matemáticas entre colecciones, como subconjuntos y superconjuntos.
- **Rendimiento de Búsqueda**: Proporcionan una búsqueda de elementos más rápida que las listas o las tuplas, lo que es útil para grandes volúmenes de datos.
## **Buenas Prácticas**

Discutiremos cuándo es beneficioso usar conjuntos en lugar de otras estructuras de datos y cómo su uso puede influir en la eficiencia del programa.

Al final de esta clase, tendrás una comprensión completa de los conjuntos en Python y cómo pueden ser utilizados para hacer tu código más eficiente y lógico, aprovechando sus propiedades únicas para manejar datos. Con este conocimiento, podrás implementar estructuras de datos complejas y operaciones que requieren lógica de conjuntos.

```python
#!/usr/bin/env python3

mi_conjunto = {1, 2, 3, 4} # Definimos el conjunto o sets
mi_conjunto2 = {1, 4, 6, 8, 9}
mi_conjunto.update({4,5,6}) # en este punto podemos agregar mas valores a nuestro conjunto

mi_conjunto.discard(4) # De esta forma odemos eleminar valores de nuestros conjuntos

print(mi_conjunto)
# Cuando nosotros agregemos elementos los sets no almacenan de manera ordenada.

conjunto_final = mi_conjunto.intersection(mi_conjunto2) # con esto podemos almacenar solo los valores repetidos entre los dos conjuntos

conjunto_final = mi_conjunto.union(mi_conjunto2) # Une los dos conjunto y elemina los elementos repetidos porque los sets no continen valores repetidos, recuerda que este tipo de dato no se ordena 

conjunto_final = mi_conjunto.issubset(mi_conjunto2) # Esto nos devuelve un valor booleano y verifica si el segundo conjunto es subconjunto del primero

conjutno_final = mi_conjunto.difference(mi_conjunto2) # Bueno esto nos devuelve los valores que son diferentes de mi_conjunto y mi_conjunto2, solo de mi_conjunto

print(conjunto_final)

# Los cundjuntos nos pueden servir para eliminar valores repetidos.

############# Verificando valores de un conjunto

print(123 in mi_conjunto) # Esto nos permite verificar si tenemos 1,2 y 3 en el conjunto
```

# Diccionarios

En esta clase, nos centraremos en los diccionarios, una de las estructuras de datos más poderosas y flexibles de Python. Los diccionarios en Python son colecciones desordenadas de pares clave-valor. A diferencia de las secuencias, que se indexan mediante un rango numérico, los diccionarios se indexan con claves únicas, que pueden ser cualquier tipo inmutable, como cadenas o números.
## Características de los Diccionarios

Desordenados: Los elementos en un diccionario no están ordenados y no se accede a ellos mediante un índice numérico, sino a través de claves únicas. Dinámicos: Se pueden agregar, modificar y eliminar pares clave-valor. Claves Únicas: Cada clave en un diccionario es única, lo que previene duplicaciones y sobrescrituras accidentales. Valores Accesibles: Los valores no necesitan ser únicos y pueden ser de cualquier tipo de dato. Operaciones con Diccionarios
## Durante la clase, exploraremos cómo realizar operaciones básicas y avanzadas con diccionarios:

Agregar y Modificar: Cómo agregar nuevos pares clave-valor y modificar valores existentes. Eliminar: Cómo eliminar pares clave-valor usando del o el método ‘pop()‘. Métodos de Diccionario: Utilizar métodos como ‘keys()‘, ‘values()‘, y ‘items()‘ para acceder a las claves, valores o ambos en forma de pares. Comprensiones de Diccionarios: Una forma elegante y concisa de construir diccionarios basados en secuencias o rangos. Uso de Diccionarios en Python

Almacenamiento de Datos Estructurados: Ideales para almacenar y organizar datos que están relacionados de manera lógica, como una base de datos en memoria. Búsqueda Eficiente: Los diccionarios son altamente optimizados para recuperar valores cuando se conoce la clave, proporcionando tiempos de búsqueda muy rápidos. Flexibilidad: Pueden ser anidados, lo que significa que los valores dentro de un diccionario pueden ser otros diccionarios, listas o cualquier otro tipo de dato. Buenas Prácticas

Enfatizaremos las mejores prácticas para trabajar con diccionarios, incluyendo la selección de claves adecuadas y el manejo de errores comunes, como intentar acceder a claves que no existen.

Al final de esta clase, tendrás una comprensión completa de los diccionarios y estarás listo para utilizarlos para gestionar eficazmente los datos dentro de tus programas. Los diccionarios son una herramienta esencial en Python y saber cómo utilizarlos te abrirá la puerta a un nuevo nivel de programación.

```python
#!/usr/bin/evn python3

# definiendo un diccionario, recordemos que los diccionarios estan definidmos por una clave: valor
# Los diccionarios no se encuentra enumerados por lo que no cuentan con un indice, por lo que para acceder a sus valres nos movemos entre sus claves
mi_diccionario = {"nombre": "stark", "edad": 20, "pais": "Ecuador"}

print(mi_diccionario["nombre"]) # imprimiendo el valor a través de la clave nombre
print(mi_diccionario["edad"]) 
print(mi_diccionario["pais"]) 

# Podemos alterar los valores a través de las claves 
mi_diccionario["pais"] = "Argentina"
print(mi_diccionario["pais"])

# Podemos agregar valres a nuestro diccionario de la siguiente manera

mi_diccionario["sexo"] = "masculino"
print(mi_diccionario)

# eleminar valores a mi diccionario

del mi_diccionario["edad"] 
print(mi_diccionario)

# Verificar si contenemos ciertas claves

if "Argentina" in mi_diccionario:
    print("Esta clave si existe en mi diccionario")
else:
    print("Esta clave no existe")
# otra forma puede ser
"Argentina" in mi_diccionario
# Iterar sobre un diccionario

for key, value in mi_diccionario.items(): # Con ayuda de la funcion items podemos iterar por el diccionario
    print(f"Para la clave: {key}, tenemos el valor: {value}")

# Podemos usar algunas variables del las listas y tuplas como len(), clear(),

# Diccionarios de comprencion

cuadrados = {x:x**2 for x in range(6) } # Podemos usar esta sintaxis para crear siertos diccionarios sencillos y con funciones especificas
print(cuadrados)

# Listar solo claves o solo valores

print(mi_diccionario.keys())
print(mi_diccionario.values())

## Uso de get

print(mi_diccionario.get("nombre", "No encontrado")) # con get podemos decir: "Si encontras la clave nombre imprimela", si no la tienes imprime "No encotrado"

# Actualizando diccionarios

mi_diccionario2 = {"mascota": "Perro", "Oficio": "Estudiante"}

mi_diccionario.update(mi_diccionario2)
print(mi_diccionario)

# Diccionarios anidados

mi_dict = { 
    "nombres": "stark",
    "hobbies": {"primero": "música", "segundo": "fútbol"},
    "edad": 20
}

print(mi_dict)
```