---
aliases:
  - Python I/O
  - Python Archivos
  - Python Cadenas
tags:
  - programacion/python
estado: 🟢 Terminado
---
# Entrada por Teclado y Salida por Pantalla

La interacción del usuario a través de la consola es una habilidad esencial en Python, y en esta clase, exploraremos las funciones que permiten la entrada y salida de datos. Utilizaremos `input()` para recoger la entrada del teclado y `print()` para mostrar mensajes en la pantalla.

En cuanto al formato de texto, veremos cómo manejar y presentar la información de manera amigable. Esto incluye desde la manipulación básica de cadenas hasta técnicas más avanzadas de formateo de cadenas que permiten la inserción de variables y la alineación del texto.

La codificación de caracteres es un aspecto clave para garantizar que la entrada y salida manejen adecuadamente diferentes idiomas y conjuntos de caracteres, preservando la integridad de los datos y la claridad de la comunicación en la consola.

Dominar estas funciones es crucial para crear programas que requieran interacción con el usuario, y te brindarán las herramientas necesarias para construir aplicaciones interactivas robustas y fáciles de usar.

```python
#!/usr/bin/env python3

def main():

    nombre = input("\n[+] Dime tu nombre: ")
    # > [!info] La función `input()` siempre devuelve una cadena de texto (string).
    # > [!tip] Es fundamental realizar la conversión de tipo (type casting) si se espera un valor numérico o de otro tipo, como se hace con `int()` a continuación.
    edad = int(input("\n[+] Dime tu edad: ")) # Todo lo que se digite por consola será un string, por lo que siempre tenemos que cambiar el tipo de dato según necesitemos.
    print(f"Hola {nombre} ¿cómo estás?")
    print(f"El siguiente año cumples {edad + 1}")

if __name__ == "__main__":
    main()
```

## Jugando con Excepciones

```python
#!/usr/bin/env python3

def main():

    while True:
        try:
            edad = int(input("\n[+] Digita tu edad: "))
            print(f"Tu edad actual es: {edad}")
            break

        except ValueError:
            # > [!warning] `ValueError` se lanza cuando una operación o función recibe un argumento que tiene el tipo correcto pero un valor inapropiado.
            # > [!info] En este caso, `int()` no puede convertir la entrada a un entero, indicando que el usuario no ingresó un número válido.
            print("[!] Solo puedes digitar valores numéricos enteros!!!!")

if __name__ == "__main__":
    main()

```

## Poniendo los Datos de la Consola Invisibles

```python
#!/usr/bin/env python3

from getpass import getpass # Este no es un módulo integrado en Python, es un script de Python.

def main():

    # > [!tip] `getpass()` es ideal para solicitar contraseñas o información sensible, ya que evita que los caracteres se muestren en la consola, mejorando la seguridad básica.
    password = getpass("[+] Digita tu contraseña: ") # getpass es una especie de input pero que evita que lo que escribamos sea visible.
    print(f"Tu contraseña es: {password}") # Esto se almacena igual que un input, por lo que podemos usarlo luego.

if __name__ == "__main__":
    main()

```

# Lectura y Escritura de Archivos

La lectura y escritura de archivos son operaciones fundamentales en la mayoría de los programas, y Python proporciona herramientas sencillas y poderosas para manejar archivos. En esta clase, aprenderemos cómo abrir, leer, escribir y cerrar archivos de manera eficiente y segura.

## Manejo Básico de Archivos

Explicaremos cómo utilizar la función `open()` para crear un objeto archivo y cómo los modos de apertura (`r` para lectura, `w` para escritura, `a` para añadir y `b` para modo binario) afectan la manera en que trabajamos con esos archivos.

## Lectura de Archivos

Detallaremos cómo leer el contenido de un archivo en memoria, ya sea de una sola vez con el método `read()` o línea por línea con `readline()` o iterando sobre el objeto archivo.

## Escritura en Archivos

Examinaremos cómo escribir en un archivo usando métodos como `write()` o `writelines()`, y cómo estos métodos difieren en cuanto a su manejo de strings y secuencias de strings.

## Manejadores de Contexto

Uno de los aspectos más importantes de la lectura y escritura de archivos en Python es el uso de manejadores de contexto, proporcionados por la declaración `with`. Los manejadores de contexto garantizan que los recursos se manejen correctamente, abriendo el archivo y asegurándose de que, sin importar cómo o dónde termine el bloque de código, el archivo siempre se cierre adecuadamente. Esto ayuda a prevenir errores comunes como fugas de recursos o archivos que no se cierran si ocurre una excepción.

El uso de `with open()` no solo mejora la legibilidad del código, sino que también simplifica el manejo de excepciones al trabajar con archivos, haciendo el código más seguro y robusto.

Al concluir esta clase, tendrás una comprensión profunda de cómo interactuar con el sistema de archivos en Python, cómo procesar datos de archivos de texto y binarios, y las mejores prácticas para asegurar que los archivos se lean y escriban de manera efectiva. Estos conocimientos son vitales para una amplia gama de aplicaciones, desde el análisis de datos hasta la automatización de tareas y el desarrollo de aplicaciones web.

```python
#!/usr/bin/env python3

# > [!info] Modos de apertura de archivos:
# > - `r`: Lectura (por defecto). El archivo debe existir.
# > - `w`: Escritura. Crea el archivo si no existe, o lo trunca (borra su contenido) si ya existe.
# > - `a`: Añadir. Crea el archivo si no existe, o añade contenido al final si ya existe.
# > - `x`: Creación exclusiva. Falla si el archivo ya existe.
# > - `b`: Modo binario. Se usa en combinación con `r`, `w`, `a` (ej. `rb`, `wb`).
# > - `t`: Modo texto (por defecto). Se usa en combinación con `r`, `w`, `a` (ej. `rt`, `wt`).

f = open("example.txt", "w") # r de leer, w de escribir (borra todo lo anterior), a de append (edita el archivo pero no borra lo anterior).

f.write("Hola mundo a otro archivo") # Modificamos el archivo.

f.close() # Cuando se abre un archivo, siempre tenemos que cerrarlo para evitar problemas de recursos o corrupción de datos.

# Otra forma por la que podemos hacer lo mismo es la siguiente:

with open("example.txt", "w") as file: # De esta forma no es necesario cerrar el archivo, ya que Python lo hace por sí mismo automáticamente al salir del bloque `with`.
    file.write("Hola desde la otra función")

```

```python
#!/usr/bin/env python3

with open("/etc/passwd", "r") as file:
    file_content = file.read()

print(file_content)

```

```python
#!/usr/bin/env python3

with open("/etc/passwd", "r") as file:
    file_content = file.read()

print(file_content)
```

```python
#!/usr/bin/env python3

with open("/etc/hosts", "r") as f : # Cuando tenemos archivos binarios donde no tenemos caracteres legibles, usamos `rb`.
    for line in f:
        # > [!tip] El método `strip()` elimina los espacios en blanco (incluyendo saltos de línea `\n`, tabulaciones `\t`, etc.) del principio y del final de la cadena.
        print(line.strip()) # Con ayuda de strip podemos quitar los saltos de línea al final de print.

```

```python
#!/usr/bin/env python3

with open("/usr/share/wordlists/rockyou.txt", "rb") as f:
    # > [!info] `readline()` lee una sola línea del archivo. Si el archivo está en modo binario (`rb`), la línea se devuelve como un objeto `bytes`.
    first_linea = f.readline() # Con esto hacemos que solo nos almacene la primera línea de nuestro archivo.

# > [!tip] Cuando se lee un archivo en modo binario (`rb`), el contenido se obtiene como `bytes`. Para mostrarlo como texto legible, es necesario decodificarlo.
# > [!info] `decode()` convierte un objeto `bytes` a una cadena de texto (string) utilizando una codificación específica (por defecto, UTF-8).
print(first_linea.strip().decode()) # El strip es por el salto de línea y el decode es para decodificar.
```

```python
#!/usr/bin/env python3

# Esta es una forma un poco rara pero funcional de ingresar contenido a un archivo.

with open("prueba.txt", "w") as f:
    # > [!info] El argumento `file=f` en la función `print()` redirige la salida estándar (normalmente la consola) a un objeto de archivo específico.
    # > [!tip] Esto es útil para escribir texto formateado directamente en un archivo sin usar `f.write()`.
    print("Hola Mundo", file=f)

```

```python
#!/usr/bin/env python3

# En este caso nos estamos copiando todo el contenido del archivo binario (imagen) y a través de otra operatoria vamos a copiarlo a otro archivo.

with open("/home/stark/Pictures/wallhaven-zyj9yj_1920x1080.png", "rb") as f_in, open("image.png", "wb") as f_out:
    file_content = f_in.read()
    # > [!info] Para archivos binarios (como imágenes), es crucial usar los modos `rb` y `wb` para leer y escribir los datos byte a byte, preservando la integridad del archivo.
    f_out.write(file_content) # Esto nos ayuda a introducir el contenido de file_content en f_out.
```

```python
#!/usr/bin/env python3

# Esta sería la manera en la cual vamos a controlar el error.

try:
    with open("prueba.txt", "r") as f:
        print(f.read())

except FileNotFoundError:
    # > [!warning] `FileNotFoundError` es una excepción específica que se lanza cuando se intenta acceder a un archivo o directorio que no existe.
    # > [!tip] Usar bloques `try-except` es una buena práctica para manejar errores esperados, como la ausencia de un archivo, y hacer que el programa sea más robusto.
    print("\n [!] No ha sido posible encontrar este archivo")
```

# Formateo de Cadenas

El formateo de cadenas y la manipulación de texto son habilidades esenciales en Python, especialmente en aplicaciones que involucran la presentación de datos al usuario o el procesamiento de información textual. En esta clase, nos centraremos en las técnicas y herramientas que Python ofrece para trabajar con cadenas de texto.

## Formateo de Cadenas

Aprenderemos los distintos métodos de formateo de cadenas que Python proporciona, incluyendo:

-   **Formateo Clásico**: A través del operador `%`, similar al `printf` en C.
> [!info] El formateo con `%` es el método más antiguo y menos recomendado para código nuevo, pero aún se encuentra en código legado. Es propenso a errores si los tipos no coinciden.
-   **Método `format()`**: Un enfoque versátil que ofrece numerosas posibilidades para formatear y alinear texto, rellenar caracteres, trabajar con números y más.
> [!tip] El método `str.format()` ofrece mayor flexibilidad y legibilidad que el operador `%`, permitiendo el uso de argumentos posicionales, de palabra clave y especificaciones de formato avanzadas.
-   **F-strings (Literal String Interpolation)**: Introducido en Python 3.6, este método permite incrustar expresiones dentro de cadenas de texto de una manera concisa y legible.
> [!info] Las f-strings son la forma más moderna y recomendada de formatear cadenas en Python debido a su concisión, legibilidad y rendimiento. Permiten incrustar expresiones Python directamente dentro de la cadena.

## Manipulación de Texto

Exploraremos las funciones y métodos incorporados para la manipulación de cadenas, que incluyen:

-   **Métodos de Búsqueda y Reemplazo**: Como `find()`, `index()`, `replace()` y métodos de expresiones regulares.
-   **Métodos de Prueba**: Para verificar el contenido de la cadena, como `isdigit()`, `isalpha()`, `startswith()`, y `endswith()`.
-   **Métodos de Transformación**: Que permiten cambiar el caso de una cadena, dividirla en una lista de subcadenas o unirlas, como `upper()`, `lower()`, `split()`, y `join()`.

También veremos cómo trabajar con cadenas Unicode en Python, lo que es esencial para aplicaciones modernas que necesitan soportar múltiples idiomas y caracteres especiales.

Al final de esta clase, tendrás una comprensión completa de cómo dar formato a las cadenas para la salida de datos y cómo realizar operaciones comunes de manipulación de texto. Estas habilidades son fundamentales para la creación de aplicaciones que necesitan una interfaz de usuario sofisticada y para el procesamiento de datos en aplicaciones de backend.

```python
#!/usr/bin/env python3

cadena = "Hola mundo!!!"
cadena2 = "hola;mundo"
nueva_cadena = cadena.split() # Nos crea una lista de palabras usando por defecto como delimitador el espacio.
nueva_cadena1 = cadena2.split(';')

#print(cadena.strip()) # Strip quita todo tipo de espacios al comienzo y al final del string.
print(cadena.lower()) # Todo en minúsculas.
print(cadena.upper()) # Todo en mayúsculas.
print(cadena.replace("o", "x")) # Esto es para reemplazos y nos cambia en este caso las 'o' por 'x'.
print(nueva_cadena)
print(nueva_cadena1)

print(cadena.startswith('H')) # Estamos diciendo "¿La cadena comienza con 'H'?" y nos devuelve un `True` o `False`.
print(cadena.endswith('!')) # Estamos diciendo "¿La cadena termina con '!'?" y nos devuelve un `True` o `False`.

print(cadena.find("Hola")) # Buscamos por una palabra pero nos devuelve la posición desde donde comienza la subcadena, y si no existe nos da -1.
# > [!info] `find()` devuelve el índice de la primera ocurrencia de la subcadena. Si no se encuentra, devuelve -1.

print(cadena.index("mundo")) # Esto hace lo mismo que `find()` pero si no encuentra la palabra nos devolverá una excepción `ValueError`.
# > [!warning] A diferencia de `find()`, `index()` lanzará un `ValueError` si la subcadena no se encuentra, lo que puede ser útil para un manejo de errores explícito.

print(cadena.count("o")) # Esto nos ayuda a contar ciertos caracteres en un string.

cadena3 = ["Hola", "mundo"]
nombres = ["Daniel", "Andres", "Dylan"]
print(' '.join(cadena3)) # Con `.join()` unimos los elementos de la cadena por un espacio.
# > [!tip] El método `str.join(iterable)` es muy eficiente para concatenar elementos de un iterable (como una lista) usando la cadena sobre la que se llama como separador.
print(', '.join(nombres))

print(cadena.capitalize()) # La primera letra de la palabra la pone en mayúscula.
print(cadena.title()) # Cambia todas las primeras letras de cada palabra por mayúsculas.
print(cadena.swapcase()) # Invierte las mayúsculas por minúsculas y las minúsculas por mayúsculas.

print(cadena.isalpha()) # Nos identifica si son caracteres alfabéticos.
print(cadena.isnumeric()) # Nos identifica si son caracteres numéricos (solo dígitos Unicode).
print(cadena.isdigit()) # Nos identifica si son dígitos decimales.
print(cadena.islower()) # Nos identifica si todos los caracteres alfabéticos están en minúsculas.
print(cadena.isupper()) # Nos identifica si todos los caracteres alfabéticos están en mayúsculas.
print(cadena.istitle()) # Nos identifica si la cadena está en formato de título (primera letra de cada palabra en mayúscula, el resto en minúscula).

print(cadena.replace(" ", "")) # Con esto podemos eliminar ciertos tipos de caracteres.

# > [!info] `str.maketrans()` crea una tabla de traducción que se usa con `str.translate()`.
# > [!tip] Es muy eficiente para reemplazar múltiples caracteres en una sola pasada, similar a la utilidad `tr` en sistemas Unix/Linux.
tabla = cadena.maketrans("Hola", "Dios") # Es una forma de sustituir varios caracteres a la vez, similar a `tr`.
cadena_nueva = cadena.translate(tabla)
print(cadena_nueva)
```