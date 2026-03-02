---
aliases: ["Python Data Structures", "Listas, Tuplas, Conjuntos, Diccionarios"]
tags: [programacion/python]
estado: 🟢 Terminado
---
# Estructuras de Datos en Python

En esta clase, nos sumergiremos en profundidad en los tipos de datos más versátiles y utilizados en Python: listas, tuplas, conjuntos y diccionarios.

## Listas en Python

Las listas son estructuras de datos que nos permiten almacenar secuencias ordenadas de elementos. Son mutables, lo que significa que podemos modificarlas después de su creación, y son dinámicas, permitiéndonos añadir o quitar elementos de ellas.

### Características de las Listas

Vamos a explorar las características clave de las listas en Python, que incluyen su capacidad para:

*   Almacenar datos heterogéneos, es decir, pueden contener elementos de diferentes tipos (enteros, cadenas, flotantes y más) dentro de una misma lista.
*   Ser indexadas y segmentadas (slicing), lo que permite acceder a elementos específicos de la lista directamente a través de su índice.
*   Ser anidadas, es decir, una lista puede contener otras listas como elementos, lo que permite crear estructuras de datos complejas como matrices.

### Operaciones con Listas

También cubriremos las operaciones fundamentales que se pueden realizar con listas, como:

*   Añadir elementos con métodos como `append()` y `extend()`.
*   Eliminar elementos con métodos como `remove()` y `pop()`.
*   Ordenar las listas con el método `sort()` o la función incorporada `sorted()`.
*   Invertir los elementos con el método `reverse()` o la sintaxis de segmentación `[::-1]`.
*   Comprender las comprensiones de listas, una forma "pythonica" de crear y manipular listas de manera concisa y eficiente.

> [!info] `append()` vs `extend()`
> El método `append()` añade un único elemento al final de la lista. Si el elemento es una lista, la añade como un único elemento anidado. Por otro lado, `extend()` itera sobre un iterable (como otra lista o tupla) y añade cada uno de sus elementos individualmente al final de la lista actual.
> ```python
> lista_a = [1, 2]
> lista_b = [3, 4]
> lista_a.append(lista_b) # lista_a ahora es [1, 2, [3, 4]]
> lista_c = [5, 6]
> lista_d = [7, 8]
> lista_c.extend(lista_d) # lista_c ahora es [5, 6, 7, 8]
> ```

> [!tip] `remove()` vs `pop()`
> `remove(valor)` elimina la primera ocurrencia del `valor` especificado de la lista. Si el valor no se encuentra, lanza un `ValueError`.
> `pop(indice)` elimina y devuelve el elemento en la posición `indice` especificada. Si no se proporciona un índice, `pop()` elimina y devuelve el último elemento de la lista.

> [!info] `sort()` vs `sorted()`
> El método `list.sort()` ordena la lista *en su lugar* (modifica la lista original) y devuelve `None`. Es eficiente para ordenar listas grandes cuando no se necesita una copia de la lista original.
> La función `sorted(iterable)` devuelve una *nueva lista* ordenada a partir de los elementos del iterable, dejando el iterable original sin modificar. Es útil cuando se necesita mantener la lista original intacta o cuando se trabaja con otros tipos de iterables (como tuplas o conjuntos).

> [!tip] Invertir listas con `[::-1]`
> La sintaxis de segmentación `[::-1]` crea una *copia invertida* de la lista. Es una forma concisa y "pythonica" de invertir una secuencia sin modificar la original. Si necesitas invertir la lista en su lugar, usa `list.reverse()`.

> [!info] Comprensiones de Listas
> Las comprensiones de listas (`[expresion for elemento in iterable if condicion]`) ofrecen una sintaxis concisa para crear listas. Son generalmente más legibles y eficientes que los bucles `for` tradicionales para la construcción de listas, ya que Python puede optimizar su ejecución internamente.

### Métodos de Listas

Profundizaremos en la rica gama de métodos que Python ofrece para trabajar con listas y cómo estos métodos pueden ser utilizados para manipular listas de acuerdo a nuestras necesidades.

### Buenas Prácticas

Discutiremos las mejores prácticas en el manejo de listas, incluyendo cómo y cuándo usar listas en comparación con otros tipos de colecciones en Python, como tuplas, conjuntos y diccionarios.

Al final de esta clase, tendrás un conocimiento profundo de las listas en Python y estarás equipado con las técnicas para manejarlas eficazmente en tus programas. Con esta base sólida, podrás manipular colecciones de datos con confianza y aplicar esta habilidad central en tareas como la manipulación de datos, la automatización y el desarrollo de algoritmos.

```python
#!/usr/bin/env python3

puertos_tcp = [22, 21, 25, 80, 443, 8080, 445, 69]

puertos_tcp.append(1337) # Agregando un puerto nuevo a la lista
puertos_tcp[2] = 50 # Con esto logramos modificar el valor que corresponde al índice 2
puertos_tcp.insert(4, 25) # Con esto insertamos un valor en la posición 4 con valor 25

print(puertos_tcp)

### Iterar en una lista

for puerto in puertos_tcp:
    print(f"puerto {puerto}")

####
puertos_tcp.remove(22) # Eliminamos elemento de la lista
print(puertos_tcp)

####

puertos_tcp.sort() # Nos permite realizar un ordenamiento a la lista (en su lugar)
print(puertos_tcp)

#######

puertos_tcp.reverse() # Nos permite invertir la lista (en su lugar)
print(puertos_tcp)

#######
print(f"puerto {puertos_tcp[0]}") # Accediendo a la lista por su índice, el primer elemento
print(f"puerto {puertos_tcp[:3]}") # Accediendo a la lista por su índice, pero solo los 3 primeros elementos
print(f"puerto {puertos_tcp[3:]}") # Accediendo a la lista por su índice, pero a partir del tercer valor
print(f"puerto {puertos_tcp[-1]}") # Accediendo a la lista por su índice, nos muestra el último elemento
print(f"puerto {puertos_tcp[1:2]}") # Accediendo a la lista por su índice, nos muestra desde el elemento 1 (inclusive) hasta el elemento 2 (exclusivo)
print(f"puerto {puertos_tcp[1:2]}") # Repetido para demostración

########
len(puertos_tcp) # Esto nos devuelve el número total de elementos dentro de la lista
puertos_tcp.clear() # Con esto logramos limpiar la lista

#########

enumerate(puertos_tcp) # Esto nos devuelve tanto el índice como el valor, esto es útil en los bucles.
puertos_tcp.pop() # Esto nos elimina el último elemento y lo devuelve
puertos_tcp.pop(22) # También podemos eliminar un elemento específico por su índice, y este valor se puede almacenar

##########

newList = [port**2 for port in puertos_tcp] # Generamos una nueva lista con todos los valores elevados a la potencia 2

# puertos_tcp.lower() # Este método no es aplicable directamente a una lista de números o a la lista misma, solo a cadenas dentro de la lista.
# puertos_tcp.upper() # Este método no es aplicable directamente a una lista de números o a la lista misma, solo a cadenas dentro de la lista.

########### Combinando listas
names = ["Daniel", "Danna", "Vivi", "Andres", "Dome", "Dylan"]
edades = [20, 6, 40, 40, 9, 9]
more_names = ["Jaime", "Raquel", "Mateo", "Camila"]

for name, edad in zip(names, edades): # Con zip podemos combinar listas para poder iterar en ellas en paralelo
    print(f"{name} tiene {edad} años")

############# Extendiendo lista
names.extend(more_names) # Nos permite extender una lista a través de otra, añadiendo sus elementos individualmente
```

## Tuplas en Python

En esta clase, dedicaremos nuestro enfoque a las tuplas, una estructura de datos fundamental en Python que comparte algunas similitudes con las listas, pero se distingue por su inmutabilidad.

Las tuplas son colecciones ordenadas de elementos que no pueden modificarse una vez creadas. Esta característica las hace ideales para asegurar que ciertos datos permanezcan constantes a lo largo del ciclo de vida de un programa.

### Características de las Tuplas

*   **Inmutabilidad**: Una vez que se crea una tupla, no puedes cambiar, añadir o eliminar elementos. Esta inmutabilidad garantiza la integridad de los datos que deseas mantener constantes.
*   **Indexación y Segmentación (Slicing)**: Al igual que las listas, puedes acceder a los elementos de la tupla mediante índices y también puedes realizar operaciones de segmentación para obtener subsecuencias de la tupla.
*   **Heterogeneidad**: Las tuplas pueden contener elementos de diferentes tipos, incluyendo otras tuplas, lo que las hace muy versátiles.

> [!info] Beneficios de la Inmutabilidad de las Tuplas
> La inmutabilidad de las tuplas ofrece varias ventajas:
> *   **Seguridad de Datos**: Una vez definida, la tupla no puede ser alterada, lo que previene modificaciones accidentales.
> *   **Hashable**: Las tuplas que contienen solo elementos inmutables son *hashable*, lo que significa que pueden ser usadas como claves en diccionarios o elementos en conjuntos. Las listas, al ser mutables, no pueden ser hashable.
> *   **Eficiencia**: A menudo, las tuplas pueden ser ligeramente más eficientes en términos de rendimiento y uso de memoria que las listas para colecciones de datos fijos.
> *   **Seguridad en Concurrencia**: En entornos multi-hilo, las tuplas son inherentemente "thread-safe" ya que no pueden ser modificadas por múltiples hilos simultáneamente.

### Operaciones con Tuplas

Aunque no puedes modificar una tupla, hay varias operaciones que puedes realizar:

*   **Empaquetado y Desempaquetado de Tuplas**: Las tuplas permiten asignar y desasignar sus elementos a múltiples variables de forma simultánea.
*   **Concatenación y Repetición**: Similar a las listas, puedes combinar tuplas usando el operador `+` y repetir los elementos de una tupla un número determinado de veces con el operador `*`.
*   **Métodos de Búsqueda**: Puedes usar métodos como `index()` para encontrar la posición de un elemento y `count()` para contar cuántas veces aparece un elemento en la tupla.

### Uso de Tuplas en Python

*   **Funciones y Asignaciones Múltiples**: Las tuplas son muy útiles cuando una función necesita devolver múltiples valores o cuando se realizan asignaciones múltiples en una sola línea.
*   **Estructuras de Datos Fijas**: Se usan para crear estructuras de datos que no deben cambiar, como los días de la semana o las coordenadas de un punto en el espacio.

### Buenas Prácticas

Abordaremos cuándo es más apropiado utilizar tuplas en lugar de listas y cómo la elección de una tupla sobre una lista puede afectar la claridad y la seguridad del código.

Al concluir esta clase, tendrás un entendimiento claro de qué son las tuplas, cómo y cuándo utilizarlas en tus programas, y las prácticas recomendadas para trabajar con este tipo de datos inmutable. Las tuplas son una herramienta poderosa en Python, y saber cómo utilizarlas te permitirá escribir código más seguro y eficiente.

```python
tupla = (1, 2, 3, 4, 5, 6) # Definiendo una tupla
tupla2 = (5,) # Al definir una tupla de un solo elemento, se debe incluir una coma para distinguirla de una expresión entre paréntesis.
print(type(tupla))

# En el caso de uso es casi similar a las listas, pero no son mutables por lo que lo siguiente no se puede hacer:
# insert(), extend(), remove(), append()

#######################

mi_tupla = (1, 2, 3, 4)
a, b, c, d = mi_tupla # Desempaquetado de tuplas

print(a)
print(b)
print(c)
print(d)

#### Modificar tupla de forma indirecta ######
mi_primera_tupla = (1, 2, 3, 4)
mi_segunda_tupla = (5, 6, 7, 8, 9)
mi_tercera_tupla = mi_primera_tupla + mi_segunda_tupla # Concatenación de tuplas, creando una nueva tupla

print(mi_tercera_tupla)

# Por lo tanto, para "agregar" valores a tuplas, se debe crear una nueva tupla a partir de las existentes o de una operación.
mi_tupla1 = (1, 2, 3 ,4, 5, 6, 7, 8, 9)

numeros_pares = tuple(i for i in mi_tupla1 if i % 2 == 0) # A través de una comprensión de generador convertida a tupla, podemos crear una nueva tupla.
print(numeros_pares)

## Las tuplas las podemos usar en un programa el cual tenga usuario y a este asignarle el valor de tupla para así evitar cambios no autorizados a los usuarios.
```

## Conjuntos o Sets en Python

En esta clase, nos adentraremos en los conjuntos, conocidos en Python como `sets`. Los conjuntos son una colección de elementos sin orden y sin elementos repetidos, inspirados en la teoría de conjuntos de las matemáticas. Son ideales para la gestión de colecciones de elementos únicos y operaciones que requieren eliminar duplicados o realizar comparaciones de conjuntos.

### Características de los Conjuntos

*   **Unicidad**: Los conjuntos automáticamente descartan elementos duplicados, lo que los hace perfectos para recolectar elementos únicos.
*   **Desordenados**: A diferencia de las listas y las tuplas, los conjuntos no mantienen los elementos en ningún orden específico. El orden de los elementos puede variar.
*   **Mutabilidad**: Los elementos de un conjunto pueden ser agregados o eliminados, pero los elementos mismos que contiene el conjunto deben ser inmutables (por ejemplo, no puedes tener un conjunto de listas, ya que las listas se pueden modificar).

> [!warning] Elementos de un Conjunto deben ser Inmutables
> Los elementos que se almacenan dentro de un conjunto deben ser *hashable*. Esto significa que deben ser inmutables (como números, cadenas, tuplas). No se pueden almacenar objetos mutables (como listas, diccionarios u otros conjuntos mutables) directamente como elementos de un conjunto, ya que su valor hash podría cambiar.

### Operaciones con Conjuntos

Exploraremos las operaciones básicas de conjuntos que Python facilita, como:

*   **Adición y Eliminación**: Añadir elementos con `add()` y eliminar elementos con `remove()` o `discard()`.
*   **Operaciones de Conjuntos**: Realizar uniones, intersecciones, diferencias y diferencias simétricas utilizando métodos u operadores respectivos.
*   **Pruebas de Pertenencia**: Comprobar rápidamente si un elemento es miembro de un conjunto.
*   **Inmutabilidad Opcional**: Usar el tipo `frozenset` para crear conjuntos que no se pueden modificar después de su creación.

> [!tip] `add()` vs `update()`
> `add(elemento)` añade un único `elemento` al conjunto. Si el elemento ya existe, no hace nada.
> `update(iterable)` añade todos los elementos de un `iterable` (como una lista, tupla o cadena) al conjunto. Es equivalente a realizar `add()` para cada elemento del iterable.

> [!info] `remove()` vs `discard()`
> Ambos métodos eliminan un elemento específico del conjunto.
> `remove(elemento)`: Elimina el `elemento` del conjunto. Si el elemento no está presente, lanza un `KeyError`.
> `discard(elemento)`: Elimina el `elemento` del conjunto. Si el elemento no está presente, no hace nada y no lanza ningún error. `discard()` es más seguro cuando no estás seguro de si el elemento existe.

> [!info] `frozenset`
> Un `frozenset` es una versión inmutable de un conjunto. Una vez creado, no se pueden añadir ni eliminar elementos. Esto los hace hashable, lo que significa que un `frozenset` puede ser un elemento de otro conjunto o una clave de diccionario.

### Uso de Conjuntos en Python

*   **Eliminación de Duplicados**: Son útiles cuando necesitas asegurarte de que una colección no tenga elementos repetidos.
*   **Relaciones entre Colecciones**: Facilitan la comprensión y el manejo de relaciones matemáticas entre colecciones, como subconjuntos y superconjuntos.
*   **Rendimiento de Búsqueda**: Proporcionan una búsqueda de elementos más rápida que las listas o las tuplas (en promedio O(1)), lo que es útil para grandes volúmenes de datos.

### Buenas Prácticas

Discutiremos cuándo es beneficioso usar conjuntos en lugar de otras estructuras de datos y cómo su uso puede influir en la eficiencia del programa.

Al final de esta clase, tendrás una comprensión completa de los conjuntos en Python y cómo pueden ser utilizados para hacer tu código más eficiente y lógico, aprovechando sus propiedades únicas para manejar datos. Con este conocimiento, podrás implementar estructuras de datos complejas y operaciones que requieren lógica de conjuntos.

```python
#!/usr/bin/env python3

mi_conjunto = {1, 2, 3, 4} # Definimos el conjunto o set
mi_conjunto2 = {1, 4, 6, 8, 9}
mi_conjunto.update({4, 5, 6}) # En este punto podemos agregar más valores a nuestro conjunto. Los duplicados se ignoran automáticamente.

mi_conjunto.discard(4) # De esta forma podemos eliminar valores de nuestros conjuntos. No lanza error si el elemento no existe.

print(mi_conjunto)
# Cuando nosotros agregamos elementos, los sets no garantizan un orden específico.

conjunto_final = mi_conjunto.intersection(mi_conjunto2) # Con esto podemos almacenar solo los valores repetidos (intersección) entre los dos conjuntos.
print(f"Intersección: {conjunto_final}")

conjunto_final = mi_conjunto.union(mi_conjunto2) # Une los dos conjuntos y elimina los elementos repetidos porque los sets no contienen valores repetidos. Recuerda que este tipo de dato no se ordena.
print(f"Unión: {conjunto_final}")

conjunto_final = mi_conjunto.issubset(mi_conjunto2) # Esto nos devuelve un valor booleano y verifica si el primer conjunto es un subconjunto del segundo.
print(f"¿mi_conjunto es subconjunto de mi_conjunto2?: {conjunto_final}")

conjunto_final = mi_conjunto.difference(mi_conjunto2) # Esto nos devuelve los valores que están en mi_conjunto pero no en mi_conjunto2.
print(f"Diferencia (mi_conjunto - mi_conjunto2): {conjunto_final}")

# Los conjuntos nos pueden servir para eliminar valores repetidos de una colección.

############# Verificando valores de un conjunto

print(123 in mi_conjunto) # Esto nos permite verificar rápidamente si el valor 123 está en el conjunto.
```

## Diccionarios en Python

En esta clase, nos centraremos en los diccionarios, una de las estructuras de datos más poderosas y flexibles de Python. Los diccionarios en Python son colecciones desordenadas de pares clave-valor. A diferencia de las secuencias, que se indexan mediante un rango numérico, los diccionarios se indexan con claves únicas, que pueden ser cualquier tipo inmutable, como cadenas o números.

### Características de los Diccionarios

*   **Desordenados**: Los elementos en un diccionario no están ordenados y no se accede a ellos mediante un índice numérico, sino a través de claves únicas. (Nota: A partir de Python 3.7, los diccionarios mantienen el orden de inserción, pero conceptualmente no se basan en un índice numérico).
*   **Dinámicos**: Se pueden agregar, modificar y eliminar pares clave-valor.
*   **Claves Únicas**: Cada clave en un diccionario es única, lo que previene duplicaciones y sobrescrituras accidentales.
*   **Valores Accesibles**: Los valores no necesitan ser únicos y pueden ser de cualquier tipo de dato.

> [!warning] Claves de Diccionario deben ser Inmutables
> Las claves de un diccionario deben ser *hashable*, lo que implica que deben ser inmutables. Esto significa que no puedes usar objetos mutables como listas, conjuntos o diccionarios como claves. Sin embargo, los valores asociados a esas claves pueden ser de cualquier tipo, incluyendo mutables.

### Operaciones con Diccionarios

Durante la clase, exploraremos cómo realizar operaciones básicas y avanzadas con diccionarios:

*   **Agregar y Modificar**: Cómo agregar nuevos pares clave-valor y modificar valores existentes.
*   **Eliminar**: Cómo eliminar pares clave-valor usando `del` o el método `pop()`.
*   **Métodos de Diccionario**: Utilizar métodos como `keys()`, `values()`, y `items()` para acceder a las claves, valores o ambos en forma de pares.
*   **Comprensiones de Diccionarios**: Una forma elegante y concisa de construir diccionarios basados en secuencias o rangos.

> [!tip] `del` vs `pop()` para eliminar elementos
> `del diccionario[clave]`: Elimina el par clave-valor especificado. Si la clave no existe, lanza un `KeyError`.
> `diccionario.pop(clave)`: Elimina el par clave-valor especificado y *devuelve el valor* asociado a la clave. Si la clave no existe, lanza un `KeyError` a menos que se proporcione un segundo argumento como valor por defecto (`diccionario.pop(clave, valor_por_defecto)`).

> [!info] `keys()`, `values()`, `items()`
> Estos métodos devuelven "objetos vista" (view objects) que proporcionan una vista dinámica de las claves, valores o pares clave-valor del diccionario. Estos objetos vista reflejan los cambios en el diccionario subyacente.
> *   `dict.keys()`: Vista de todas las claves.
> *   `dict.values()`: Vista de todos los valores.
> *   `dict.items()`: Vista de todos los pares (clave, valor) como tuplas.

> [!tip] Comprensiones de Diccionarios
> Similar a las comprensiones de listas, las comprensiones de diccionarios (`{clave_expresion: valor_expresion for elemento in iterable if condicion}`) permiten crear diccionarios de forma concisa y eficiente. Son ideales para transformar o filtrar datos en un nuevo diccionario.

### Uso de Diccionarios en Python

*   **Almacenamiento de Datos Estructurados**: Ideales para almacenar y organizar datos que están relacionados de manera lógica, como una base de datos en memoria.
*   **Búsqueda Eficiente**: Los diccionarios son altamente optimizados para recuperar valores cuando se conoce la clave, proporcionando tiempos de búsqueda muy rápidos (en promedio O(1)).
*   **Flexibilidad**: Pueden ser anidados, lo que significa que los valores dentro de un diccionario pueden ser otros diccionarios, listas o cualquier otro tipo de dato.

### Buenas Prácticas

Enfatizaremos las mejores prácticas para trabajar con diccionarios, incluyendo la selección de claves adecuadas y el manejo de errores comunes, como intentar acceder a claves que no existen.

Al final de esta clase, tendrás una comprensión completa de los diccionarios y estarás listo para utilizarlos para gestionar eficazmente los datos dentro de tus programas. Los diccionarios son una herramienta esencial en Python y saber cómo utilizarlos te abrirá la puerta a un nuevo nivel de programación.

```python
#!/usr/bin/env python3

# Definiendo un diccionario. Recordemos que los diccionarios están definidos por pares clave: valor.
# Los diccionarios no se encuentran enumerados por lo que no cuentan con un índice; para acceder a sus valores, nos movemos entre sus claves.
mi_diccionario = {"nombre": "stark", "edad": 20, "pais": "Ecuador"}

print(mi_diccionario["nombre"]) # Imprimiendo el valor a través de la clave "nombre"
print(mi_diccionario["edad"])
print(mi_diccionario["pais"])

# Podemos alterar los valores a través de las claves
mi_diccionario["pais"] = "Argentina"
print(mi_diccionario["pais"])

# Podemos agregar valores a nuestro diccionario de la siguiente manera
mi_diccionario["sexo"] = "masculino"
print(mi_diccionario)

# Eliminar valores de mi diccionario
del mi_diccionario["edad"]
print(mi_diccionario)

# Verificar si contiene ciertas claves
if "Argentina" in mi_diccionario: # Esto verifica si "Argentina" es una CLAVE en el diccionario, no un valor.
    print("Esta clave sí existe en mi diccionario")
else:
    print("Esta clave no existe")
# Otra forma puede ser
print("Argentina" in mi_diccionario) # Devuelve True/False

# Iterar sobre un diccionario
for key, value in mi_diccionario.items(): # Con ayuda de la función items() podemos iterar por el diccionario, obteniendo clave y valor.
    print(f"Para la clave: {key}, tenemos el valor: {value}")

# Podemos usar algunas funciones de las listas y tuplas como len(), clear().
print(f"Número de elementos en el diccionario: {len(mi_diccionario)}")
mi_diccionario.clear() # Vacía el diccionario
print(f"Diccionario después de clear(): {mi_diccionario}")

# Volvemos a inicializar el diccionario para los siguientes ejemplos
mi_diccionario = {"nombre": "stark", "edad": 20, "pais": "Ecuador"}

# Diccionarios por comprensión
cuadrados = {x: x**2 for x in range(6)} # Podemos usar esta sintaxis para crear ciertos diccionarios sencillos y con funciones específicas.
print(f"Diccionario de cuadrados: {cuadrados}")

# Listar solo claves o solo valores
print(f"Claves del diccionario: {mi_diccionario.keys()}")
print(f"Valores del diccionario: {mi_diccionario.values()}")

## Uso de get()
print(mi_diccionario.get("nombre", "No encontrado")) # Con get() podemos decir: "Si encuentras la clave 'nombre', imprímela"; si no la tienes, imprime "No encontrado".
print(mi_diccionario.get("ciudad", "No encontrado")) # Ejemplo de clave no existente

# Actualizando diccionarios
mi_diccionario2 = {"mascota": "Perro", "Oficio": "Estudiante"}

mi_diccionario.update(mi_diccionario2) # Añade o actualiza pares clave-valor de mi_diccionario2 en mi_diccionario.
print(f"Diccionario actualizado: {mi_diccionario}")

# Diccionarios anidados
mi_dict = {
    "nombres": "stark",
    "hobbies": {"primero": "música", "segundo": "fútbol"},
    "edad": 20
}

print(f"Diccionario anidado: {mi_dict}")
print(f"Primer hobby: {mi_dict['hobbies']['primero']}")
```