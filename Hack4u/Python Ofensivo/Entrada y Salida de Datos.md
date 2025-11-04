-------
# Entrada por teclado y salida por Pantalla

La interacción del usuario a través de la consola es una habilidad esencial en Python, y en esta clase, exploraremos las funciones que permiten la entrada y salida de datos. Utilizaremos ‘**input()**‘ para recoger la entrada del teclado y ‘**print()**‘ para mostrar mensajes en la pantalla.

En cuanto al formato de texto, veremos cómo manejar y presentar la información de manera amigable. Esto incluye desde la manipulación básica de cadenas hasta técnicas más avanzadas de formateo de cadenas que permiten la inserción de variables y la alineación del texto.

La codificación de caracteres es un aspecto clave para garantizar que la entrada y salida manejen adecuadamente diferentes idiomas y conjuntos de caracteres, preservando la integridad de los datos y la claridad de la comunicación en la consola.

Dominar estas funciones es crucial para crear programas que requieran interacción con el usuario, y te brindarán las herramientas necesarias para construir aplicaciones interactivas robustas y fáciles de usar.

```python
#!/usr/bin/env python3

def main():

    nombre = input("\\n[+] Dime tu nombre: ")
    edad = int(input("\\n[+] Dime tu edad: ")) # Todo lo que se digite por consola sera un string por lo que siempre tenemos que cambiar el tipo de dato segun nesecitemos 
    print(f"Hola  {nombre} como estas?")
    print(f"El siguiente año cumples {edad + 1}")

if __name__ == "__main__":
    main()
```

Jugando con excepciones:

```python
#!/usr/bin/env python3

def main():

    while True:
        try:
            edad = int(input("\\n[+] Digita tu edad: "))
            print(f"Tu edad actual es: {edad}")
            break

        except ValueError:

            print("[!] Solo puede digitar valores numéricos enteros!!!!")

if __name__ == "__main__":
    main()

```

Poniendo los datos de la consola invisibles:

```python
#!/usr/bin/env python3

from getpass import getpass # Este no es un modulo integrado en python, es un script de python

def main():

    password = getpass("[+] Digita tu contraseña: ") #getpass es una especie de input pero que evita que lo que escribamos sea visible
    print(f"Tu contraseña es: {password}") # Esto se almacena igual que un input por lo que podemos usarlo luego

if __name__ == "__main__":
    main()

```

# Lectura y Escritura de Archivos

La lectura y escritura de archivos son operaciones fundamentales en la mayoría de los programas, y Python proporciona herramientas sencillas y poderosas para manejar archivos. En esta clase, aprenderemos cómo abrir, leer, escribir y cerrar archivos de manera eficiente y segura.
## **Manejo Básico de Archivos**

Explicaremos cómo utilizar la función ‘**open()**‘ para crear un objeto archivo y cómo los modos de apertura (‘**r**‘ para lectura, ‘**w**‘ para escritura, ‘**a**‘ para añadir y ‘**b**‘ para modo binario) afectan la manera en que trabajamos con esos archivos.
## **Lectura de Archivos**

Detallaremos cómo leer el contenido de un archivo en memoria, ya sea de una sola vez con el método ‘**read()**‘ o línea por línea con ‘**readline()**‘ o iterando sobre el objeto archivo.
## **Escritura en Archivos**

Examinaremos cómo escribir en un archivo usando métodos como ‘**write()**‘ o ‘**writelines()**‘, y cómo estos métodos difieren en cuanto a su manejo de strings y secuencias de strings.
## **Manejadores de Contexto**

Uno de los aspectos más importantes de la lectura y escritura de archivos en Python es el uso de manejadores de contexto, proporcionados por la declaración ‘**with**‘. Los manejadores de contexto garantizan que los recursos se manejen correctamente, abriendo el archivo y asegurándose de que, sin importar cómo o dónde termine el bloque de código, el archivo siempre se cierre adecuadamente. Esto ayuda a prevenir errores comunes como fugas de recursos o archivos que no se cierran si ocurre una excepción.

El uso de ‘**with open()**‘ no solo mejora la legibilidad del código, sino que también simplifica el manejo de excepciones al trabajar con archivos, haciendo el código más seguro y robusto.

Al concluir esta clase, tendrás una comprensión profunda de cómo interactuar con el sistema de archivos en Python, cómo procesar datos de archivos de texto y binarios, y las mejores prácticas para asegurar que los archivos se lean y escriban de manera efectiva. Estos conocimientos son vitales para una amplia gama de aplicaciones, desde el análisis de datos hasta la automatización de tareas y el desarrollo de aplicaciones web.

```jsx
#!/usr/bin/env python3

f = open("example.txt", "w") # r de leer, w de escribir(borra todo lo anterior), a de append(edita el archivo pero no lo borra lo anterior)

f.write("Hola mundo a otro archivo") # modificamos el archivo

f.close() # Cuando se habra un archivo siempre tenemos que cerrarlo para evitar problemas 

# Otra forma por la que podemos hacer lo mismo es la siguiente

with open("example.txt", "w") as file: # De esta forma no es nesesario cerrar el archivo ya que python lo hace por si mismo
    file.write("Hola desde la otra función")

```

```jsx
#!/usr/bin/env python3

with open("/etc/passwd", "r") as file:
    file_content = file.read()

print(file_content)

```

```jsx
#!/usr/bin/env python3

with open("/etc/passwd", "r") as file:
    file_content = file.read()

print(file_content)
```

```jsx
#!/usr/bin/env python3

with open("/etc/hosts", "r") as f : # otra forma cuanto tenemos archivos binarios donde no tenemos caracters leguibles usamos rb
    for line in f:
        print(line.strip()) # con ayuda de strip podemos quitar los saltos de lineas al final de print

```

```jsx
#!/usr/bin/env python3

with open("/usr/share/wordlists/rockyou.txt", "rb") as f:
    first_linea = f.readline() # con esto hacemos que solo nos almacene la primera linea de nuestro archivo

print(first_linea.strip().decode()) # El strip es por el salto de lina y el decode es para decodificar.
```

```jsx
#!/usr/bin/env python3

# Esta es una forma un poco rarra pero funcional de ingresar contenido a un archivo

with open("prueba.txt", "w") as f:
    print("Hola Mundo", file=f)

```

```python
#!/usr/bin/env python3

# En este caso nos estamos copiando todo el contenido del archivo binario(imagen) y a través de otra operatoria vamos a copiarlo a otro archivo.

with open("/home/stark/Pictures/wallhaven-zyj9yj_1920x1080.png", "rb") as f_in, open("image.png", "wb") as f_out:
    file_content = f_in.read()
    f_out.write(file_content) # Esto nos ayuda a introducir el contenido de file_content en f_out
```

```python
#!/usr/bin/env python3

# Esta seria la manera en la cual  vamos a controlar el error.

try:
    with open("puerba.txt", "r") as f:
        print(f.read())

except FileNotFoundError:
    print("\\n [!] No ha sido posible encontrar este archivo")
```

# Formateo de Cadenas

El formateo de cadenas y la manipulación de texto son habilidades esenciales en Python, especialmente en aplicaciones que involucran la presentación de datos al usuario o el procesamiento de información textual. En esta clase, nos centraremos en las técnicas y herramientas que Python ofrece para trabajar con cadenas de texto.
## **Formateo de Cadenas**

Aprenderemos los distintos métodos de formateo de cadenas que Python proporciona, incluyendo:

- **Formateo Clásico**: A través del operador %, similar al ‘**printf**‘ en C.
- **Método format()**: Un enfoque versátil que ofrece numerosas posibilidades para formatear y alinear texto, rellenar caracteres, trabajar con números y más.
- **F-strings (Literal String Interpolation)**: Introducido en Python 3.6, este método permite incrustar expresiones dentro de cadenas de texto de una manera concisa y legible.

# **Manipulación de Texto**

Exploraremos las funciones y métodos incorporados para la manipulación de cadenas, que incluyen:

- **Métodos de Búsqueda y Reemplazo**: Como ‘**find()**‘, ‘**index()**‘, ‘**replace()**‘ y métodos de expresiones regulares.
- **Métodos de Prueba**: Para verificar el contenido de la cadena, como ‘**isdigit()**‘, ‘**isalpha()**‘, ‘**startswith()**‘, y ‘**endswith()**‘.
- **Métodos de Transformación**: Que permiten cambiar el caso de una cadena, dividirla en una lista de subcadenas o unirlas, como ‘**upper()**‘, ‘**lower()**‘, ‘**split()**‘, y ‘**join()**‘.

También veremos cómo trabajar con cadenas Unicode en Python, lo que es esencial para aplicaciones modernas que necesitan soportar múltiples idiomas y caracteres especiales.

Al final de esta clase, tendrás una comprensión completa de cómo dar formato a las cadenas para la salida de datos y cómo realizar operaciones comunes de manipulación de texto. Estas habilidades son fundamentales para la creación de aplicaciones que necesitan una interfaz de usuario sofisticada y para el procesamiento de datos en aplicaciones de backend.

```python
#!/usr/bin/env python3

cadena = "Hola mundo!!!"
cadena2 = "hola;mundo"
nuva_cadena = cadena.split() # Nos crear una cadena de las palabras usando por defecto como delimitador el espacio
nuva_cadena1 = cadena2.split(';')

#print(cadena.strip()) # Strip quita todo tipo de estacios al comienzo y al fina de el strig
print(cadena.lower()) # Todo en minustulas
print(cadena.upper()) # Todod en mayusculas
print(cadena.replace("o", "x")) # Esto es para remplasos y nos cambia en este caso las o por x
print(nuva_cadena)
print(nuva_cadena1)

print(cadena.startswith('H')) # Estamos diciendo "La cadena comienza con H? " y nos devuelve un true o falce
print(cadena.endswith('!')) # Estamos diciendo "La cadena termia con !? " y nos devuelve un true o falce

print(cadena.find("Hola")) # Buscamos por una palabra pero nos devuelve la posición desde donde comienza la cadena, y si no existe nos da -1

print(cadena.index("mundo")) # Esto hace lo mismo que find pero si no encuentra la palabra nos devolvera una excepciónj

print(cadena.count("o")) # Esto nos ayuda a contar ciertos caracteres en un string

cadena3 = ["Hola", "mundo"]
nombres = ["Daniel", "Andres", "Dylan"]
print(' '.join(cadena3)) # con punto joint unimos los elementos de la cadena por un espacio
print(', '.join(nombres))

print(cadena.capitalize()) # la primera letra de la palabra la pone en mayuscula
print(cadena.title()) # Cambia todas las primeras letras de cada palabra por mayusculas
print(cadena.swapcase()) # Inverte la mayusculas por minusculas y las minusculas por mayusculas

print(cadena.isalpha()) # Nos identifica si son caracteres
print(cadena.isnumeric())
print(cadena.isdigit())
print(cadena.islower())
print(cadena.isupper())
print(cadena.istitle())

print(cadena.replace(" ", "")) # Con esto podemos eliminar ciertos tipos de caracteres

tabla = cadena.maketrans("Hola", "Dios") # es una forma de sustituri varios caracteres a la ves, parecido a un tr
cadena_nueva = cadena.translate(tabla)
print(cadena_nueva)
```