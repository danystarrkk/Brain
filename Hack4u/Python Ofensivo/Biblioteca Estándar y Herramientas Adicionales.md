------
En el manejo de Bibliotecas tenemos que tener claro conceptos como: [[Programación Orientada a Objetos]], [[Manejo de Librerías]].
# Manejo de fechas y horas

La biblioteca ‘**datetime**‘ en Python es una de las principales herramientas para trabajar con fechas y horas. Aquí hay algunos aspectos clave sobre esta biblioteca:

- **Tipos de Datos Principales**: ‘**datetime**‘ incluye varios tipos de datos, como ‘**date**‘ (para fechas), ‘**time**‘ (para horas), ‘**datetime**‘ (para fechas y horas combinadas), y ‘**timedelta**‘ (para representar diferencias de tiempo).
- **Manipulación de Fechas y Horas**: Permite realizar operaciones como sumar o restar días, semanas, o meses a una fecha, comparar fechas, o extraer componentes como el día, mes, o año de una fecha específica.
- **Zonas Horarias**: A través del módulo ‘**pytz**‘ que se integra con ‘**datetime**‘, se pueden manejar fechas y horas en diferentes zonas horarias, lo que es crucial para aplicaciones que requieren precisión a nivel global.
- **Formateo y Análisis**: ‘**datetime**‘ permite convertir fechas y horas a strings y viceversa, utilizando códigos de formato específicos. Esto es útil para mostrar fechas y horas en formatos legibles o para parsear strings que representan fechas/horas.
- **Facilidad de Uso**: A pesar de su potencia y flexibilidad, datetime es relativamente fácil de usar, lo que la hace accesible incluso para programadores principiantes.
- **Amplia Aplicación**: Desde registros de eventos hasta cálculos de períodos de tiempo, datetime es indispensable en una variedad de aplicaciones, como sistemas de reservas, análisis de datos temporales, y más.

En resumen, datetime es una biblioteca integral y robusta para el manejo de fechas y horas en Python, ofreciendo una amplia gama de funcionalidades esenciales para el manejo de datos temporales en la programación.

```python
#!/usr/bin/evn python3

import datetime #libreria nesesaria

ahora = datetime.datetime.now() # Con esto nos da la fecha y hora acutal

# Podemos imprimier las partes de una datetime que en este caso es ahora
year = ahora.year
mes = ahora.month
dia = ahora.day
hora = ahora.hour
minuto = ahora.minute
segundo = ahora.second

date = datetime.date(2023, 5, 14) # Nos permite imprimir una fecha que queramos

time = datetime.time(14, 15, 15) # Nos permite imprimier la hora que queramos

date_time = datetime.datetime(2023, 6 ,12, 14,15,15) # con esto imprimimos tanto una fecha y una hora

print(ahora)
```

# Expresiones Regulares

La librería ‘**re**‘ en Python proporciona un conjunto completo de herramientas para trabajar con expresiones regulares, que son patrones de cadenas diseñados para la búsqueda y manipulación de texto.

Aquí hay varios aspectos importantes de esta librería:

- **Funciones Básicas**: ‘**re**‘ incluye funciones como ‘**search**‘ (para buscar un patrón dentro de una cadena), ‘**match**‘ (para verificar si una cadena comienza con un patrón específico), ‘**findall**‘ (para encontrar todas las ocurrencias de un patrón), y ‘**sub**‘ (para reemplazar partes de una cadena que coinciden con un patrón).
- **Compilación de Patrones**: Permite compilar expresiones regulares en objetos de patrón, lo que puede mejorar el rendimiento cuando se usan repetidamente.
- **Grupos y Captura**: Ofrece la capacidad de definir grupos dentro de patrones de expresiones regulares, lo que facilita extraer partes específicas de una cadena que coinciden con subpatrones.
- **Flags**: Soporta modificadores que alteran la forma en que las expresiones regulares son interpretadas y coincididas, como ignorar mayúsculas y minúsculas o permitir el modo multilínea.
- **Patrones Complejos**: Permite la creación de patrones complejos utilizando una variedad de símbolos y secuencias especiales, como cuantificadores, aserciones y clases de caracteres.
- **Aplicaciones Prácticas**: Las expresiones regulares son extremadamente útiles en tareas como la validación de formatos (por ejemplo, direcciones de correo electrónico), el análisis de registros (logs), el procesamiento de lenguaje natural, y la limpieza y preparación de datos.
- **Curva de Aprendizaje**: Aunque potentes, las expresiones regulares pueden ser complejas y requieren una curva de aprendizaje. Sin embargo, una vez dominadas, se convierten en una herramienta invaluable en el arsenal de cualquier programador.

En resumen, la librería ‘**re**‘ en Python es una herramienta esencial para cualquier tarea que implique procesamiento complejo de cadenas de texto, proporcionando una forma poderosa y flexible de buscar, analizar, y manipular datos basados en texto.

```python
#!/usr/bin/env python3

import re

text = "Mi gato está en el tejado y mi otro gato está en el jardin"

matches = re.findall("gato", text) # Esto nos permite listar y guardar en una lista las veses que aparaece la palabra

text2 = "Hoy estamos en fecha 10/10/2023, mañana estaremos a 11/10/2023"

matches = re.findall("\\d{2}\\/\\d{2}\\/\\d{4}", text2) # Esto no es mas que un patron de busqueda el cual dice "trae \\d{2}(digito de 2 valor)\\/(escapamos la /) otro digito de dos vamor y por ultimo uno de cuatro"

text3 = "Los usuario pueden contactarnos a soporte@danystarrkk.ecuador o a soporte@andresestrella26.tech"

matches = re.findall("(\\w+)@(\\w+\\.\\w{2,})", text3) # el \\w es de caracteres alfanumericos y \\w+ de caracteres alphanumericos y mas osea los que encuentre
# por ultimo definimos lo que es \\w{2,} caracteres alphanumericos con un minimos de 2 y un maximo no definino para no limitar lo que se tenga luego

text4 = "Mi gato esta en el tejado y mi perro esta en el jardín"
nuevo_texto = re.sub("gato", "perro", text) # Esto realiza una sustitución pero no multiple solo con la primera conincidencia por lo tanto o se altera en toda la linea

text5 = "Campo1,Campo2,Campo3,Campo4,Campo5" # Nos permite a través de un delimintador crear una cadena de diferes valores
nuevo_texto = re.split(",", text5)

print(nuevo_texto)
```

Programa de prueba:

```python
#!/usr/bin/env python3

import re

def validar_correo(correo):

    # La r es de raw y no sirve para que python pueda entender los diferentes parametros como \\b que nos indica con que queremos que empiece o termine
    patron = r"\\b[A-Za-z0-9._+-]+@[A-Za-z0-9]+\\.[A-Za-z]{2,}\\b" # Esto nos permite crear un patron que dice "[todo lo que esta aqui se engloba] @ [todo lo que esta aqui se engloba]
    #\\.(escapamos el .)[todo lo que esta aqui se engloba]{2,} pero puede tener un minimo de 2 caracteres "

    
    if re.findall(patron, correo): # En este punto es en el cual se realiza la comparatoria y nos devuelve el contenido, al usar if se devuelve contenido es true y entra si no no lo hace
        return True

    else:
        return False

print(validar_correo("soporte@A.io"))
```

Programa de prueba2:

```python
#!/usr/bin/env python3

import re

texto = "Hos estamos a día 10/10/2023 y mañana estaremos a 11/10/2023"
patron = r"\\b(\\d{2}\\/\\d{2}\\/\\d{4})\\b"

#print(re.findall(patron, texto))

for match in re.finditer(patron, texto):
    print(f"la fecha es: {match.group(0)} la cual comienza en la posición {match.start()} y termina en la posición {match.end()}")
    #tanto match.start como match.end nos pemite identificar en donde comienza la cadena y en donde termina, el valor del indice donde comienza
```

# Manejo de Archivos y Directorios

La librería ‘**os**‘ y el módulo ‘**shutil**‘ en Python son herramientas fundamentales para interactuar con el sistema de archivos, especialmente en lo que respecta a la creación y eliminación de archivos y directorios.

Aquí tienes una descripción detallada de ambas:
## **Librería os**

- **Funcionalidades Básicas**: ‘os’ proporciona una interfaz rica y variada para interactuar con el sistema operativo subyacente. Permite realizar operaciones como la creación y eliminación de archivos y directorios, así como la manipulación de rutas y el manejo de la información del sistema de archivos.
## **Creación y Eliminación de Archivos y Directorios**

- **Creación de Directorios**: Utilizando ‘**os.mkdir()**‘ u ‘**os.makedirs()**‘, se pueden crear directorios individuales o múltiples directorios (y subdirectorios) respectivamente.
- **Eliminación**: ‘**os.remove()**‘ se usa para eliminar archivos, mientras que ‘**os.mkdir()**‘ y ‘**os.removedirs()**‘ permiten eliminar directorios y directorios con subdirectorios, respectivamente.
- **Gestión de Rutas**: La sublibrería ‘**os.path**‘ es crucial para manipular rutas de archivos y directorios, como unir rutas, obtener nombres de archivos, verificar si un archivo o directorio existe, etc.
## **Módulo shutil**

- **Operaciones de Alto Nivel**: Mientras que os se enfoca en operaciones básicas, ‘**shutil**‘ proporciona funciones de nivel superior, más orientadas a tareas complejas y operaciones en lotes.
- **Copiar y Mover Archivos y Directorios**: ‘**shutil**‘ es especialmente útil para copiar y mover archivos y directorios. Funciones como ‘**shutil.copy()**‘, ‘**shutil.copytree()**‘, y ‘**shutil.move()**‘ facilitan estas tareas.
- **Eliminación de Directorios**: Aunque ‘**os**‘ puede eliminar directorios, ‘**shutil.rmtree()**‘ es una herramienta más poderosa para eliminar un directorio completo y todo su contenido.
- **Manejo de Archivos Temporales**: ‘**shutil**‘ también ofrece funcionalidades para trabajar con archivos temporales, lo que es útil para operaciones que requieren almacenamiento temporal de datos.

En resumen, ‘**os**‘ y ‘**shutil**‘ en Python son bibliotecas complementarias para la gestión de archivos y directorios. Mientras ‘**os**‘ ofrece una gran flexibilidad para operaciones básicas y de bajo nivel, ‘**shutil**‘ brinda herramientas más potentes y de alto nivel, adecuadas para tareas complejas y operaciones en lotes. Juntas, forman un conjunto integral de herramientas para la manipulación eficaz del sistema de archivos en Python.

```python
#!/usr/bin/env python3

import os, shutil

if os.path.exists("mi_directorio"):
    print(f"\\n[+] El archivo existe\\n")
else:
    print(f"\\n[+] El archivo no existe\\n")

## si no existe podemos usar

if not os.path.exists("mi_directorio"):
    os.mkdir("mi_directorio")

###### Creando directorios anidandos

if not os.path.exists("mi_directorio/mi_subdirectorio"): # Comprovamos su existencia
    os.makedirs("mi_directorio/mi_subdirectorio") # Creamos los directorios anidandos

### Listadon los recursos de una ruta

print(f"[+] Listando recurso: \\n")

recursos = os.listdir() # Listamos los recursos y los almacenamos en una variable, estos se almacenan en una lista

for recurso in recursos:
    print(recurso)

##### Borrar recursos

if os.path.exists("file1.txt"):
    os.remove("file1.txt")

if os.path.exists("mi_directorio1"):
    os.rmdir("mi_directorio1") # solo podemos eliminar los directorios basios en el sistema.

##### Borrar directorios que almacenan contenido

if os.path.exists("mi_directorio"):
    shutil.rmtree("mi_directorio") # Nos permite eliminar directorios con  contenido

##### Renombrar archivos
if os.path.exists("file2.txt"):
    os.rename("file2.txt", "cambiado.txt")

## Tamaño de los archivos

if os.path.exists("/etc/passwd"):
    tamano = os.path.getsize("/etc/passwd") # representa el tamaño en bytes

print(tamano)

```

Practica:

```python
#!/usr/bin/env python3

import os

ruta = os.path.join("mi_directorio", "mi_archivo.txt") # para pasarle esto debemos seguir la estructura [ruta, archio final]

print(f"\\n[+] Ruta: {ruta}\\n")

archivo = os.path.basename(ruta)

print(archivo)

directorio = os.path.dirname(ruta) # Nos almacena el directio que que contine
print(f"\\n[+] Nombre del directorio: {directorio}\\n")

directorio , archivo = os.path.split(ruta) # Con esto igualamos atraves de la lista que se genera las variables

print(f"\\n El directorio es: {directorio}")

print(f"\\n El archivo es: {archivo}")
```

# Conexiones de Red y Protocolos

Los protocolos **TCP** (Transmission Control Protocol) y **UDP** (User Datagram Protocol) son fundamentales en la comunicación de red, y la librería ‘**socket**‘ en Python proporciona las herramientas necesarias para interactuar con ellos. Aquí tienes una descripción detallada de ambos protocolos y el uso de ‘**socket**‘:
## **Protocolo TCP**

- **Orientado a la Conexión**: TCP es un protocolo orientado a la conexión, lo que significa que establece una conexión segura y confiable entre el emisor y el receptor antes de la transmisión de datos.
- **Fiabilidad y Control de Flujo**: Garantiza la entrega de datos sin errores y en el mismo orden en que se enviaron. También gestiona el control de flujo y la corrección de errores.
- **Uso en Aplicaciones**: Es ampliamente utilizado en aplicaciones que requieren una entrega fiable de datos, como navegadores web, correo electrónico, y transferencia de archivos.
## **Protocolo UDP**

- **No Orientado a la Conexión**: A diferencia de TCP, UDP es un protocolo no orientado a la conexión. Envía datagramas (paquetes de datos) sin establecer una conexión previa.
- **Rápido y Ligero**: UDP es más rápido y tiene menos sobrecarga que TCP, ya que no verifica la llegada de paquetes ni mantiene el orden de los mismos.
- **Uso en Aplicaciones**: Ideal para aplicaciones donde la velocidad es crucial y se pueden tolerar algunas pérdidas de datos, como juegos en línea, streaming de vídeo y voz sobre IP (VoIP).
## **Librería ‘socket’ en Python**

La librería ‘**socket**‘ en Python es una herramienta esencial para la programación de comunicaciones en red. Permite a los desarrolladores crear aplicaciones que pueden enviar y recibir datos a través de la red, ya sea en una red local o a través de Internet. Aquí tienes una visión general de la librería ‘**socket**‘:

- **Creación de sockets**: La librería ‘**socket**‘ proporciona clases y funciones para crear sockets, que son puntos finales de comunicación. Puedes crear sockets tanto para el protocolo TCP (Transmission Control Protocol) como para UDP (User Datagram Protocol).
- **Conexiones TCP**: Puedes utilizar ‘**socket**‘ para establecer conexiones TCP, que son conexiones confiables y orientadas a la conexión. Esto es útil para aplicaciones que requieren transferencia de datos confiable, como la transmisión de archivos o la comunicación cliente-servidor.
- **Comunicación UDP**: La librería ‘**socket**‘ también admite la comunicación mediante UDP, que es un protocolo de envío de mensajes sin conexión. Es adecuado para aplicaciones que necesitan una comunicación rápida y eficiente, como juegos en línea o aplicaciones de transmisión de video en tiempo real.
- **Funciones de envío y recepción**: Puedes utilizar métodos como ‘**send()**‘ y ‘**recv()**‘ para enviar y recibir datos a través de sockets. Esto te permite transferir información entre dispositivos de manera eficiente.
- **Gestión de conexiones**: La librería ‘**socket**‘ incluye métodos como ‘**bind()**‘ para asociar un socket a una dirección y puerto específicos, y ‘**listen()**‘ para poner un socket en modo de escucha, lo que le permite aceptar conexiones entrantes.
- **Conexiones cliente-servidor**: Con ‘**socket**‘, puedes crear aplicaciones cliente-servidor, donde un programa actúa como servidor esperando conexiones entrantes y otro actúa como cliente para conectarse al servidor.

En resumen, la librería ‘**socket**‘ en Python proporciona las herramientas necesarias para desarrollar aplicaciones de red, permitiendo la comunicación entre dispositivos a través de diferentes protocolos y ofreciendo control sobre la transferencia de datos. Es una parte fundamental de la programación de redes en Python y se utiliza en una amplia variedad de aplicaciones, desde servidores web hasta aplicaciones de chat y juegos en línea.

server v1:

```python
#!/usr/bin/env python3

import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # este es una especie de descriptor el cual nos va a permitir aceptar conexiónes entrantes y tambien nos permite
# establecer el servicio en escucha para que otros se conecten a nosotros. El (socket.AF_INET)-> esto define la familia de direcciones que acepta y (socket.SOCK_STREAM) define
# que vamos a estar trabajando con conexiónes del tipo TCP

server_adderss = ('localhost', 1234) # definimos el server_adderss a traves de una tupla en donde describimos equipo local(localhost) y el puerto(1234)
server_socket.bind(server_adderss) # con ayuda de bind nos ponemos en escucha

server_socket.listen(1) # en este punto limitamos la cola de espera o limite de conexiónes para interactuar solo con una persona a la vez

while True:

    client_socket, client_addres = server_socket.accept() # Al aceptar la conexión este deve devolvernos dos partes el client_socket con el cual nos permite comunicarnos con else:
    # y el client_addres que es una valiable a través de la cual vamos apercivir la ip del cliente que se conecta con nostros y el puerto aleatorio que se habre la conexión.

    data = client_socket.recv(1024) # Estafuncion nos permite recivir el mensaje del cliente que se conecto y devemos definir el tamaño de la información en bytes, por esto
    # es el 1024 que se define.

    print(f"\\n[+] Mensaje recibido del cliente: {data.decode()}")
    print(f"[+] Información del cliente que se a comunicado con nosotros: \\n {client_addres}")

    client_socket.sendall(f"Un saludo crack\\n".encode()) # Nosotros en este punto logramos enviar un mensaje asiendo uso del client_socket que se consigio al entablar la conexión

    client_socket.close()
```

cliente v1:

```python
#!/usr/bin/env python3

import socket

client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_addres = ("localhost", 1234) # definimos en una tupla los parametros del servidor a conectarse
client_socket.connect(server_addres) # con esto establecemos una conexión con el servidor.

try:
    mesage = b"Este es un emnsaje de prueba que envio al servidor" # Generamos un mensaje en bytes
    client_socket.sendall(mesage) # Enviamos el mensaje al servidor
    data =  client_socket.recv(1024)

    print(f"[+] El servidor nos a respondido con un: {data.decode()}") #Imprimimos el mensaje sin olvidarnos del decode para que no este en formato bytes

finally: # recordemos que finally se ejecuta sin importa lo que pase en try es por eso que lo usamos para cerra el socket
    client_socket.close()

```

server Final:

```python
#!/usr/bin/env python3

import socket #importamos socket para podeer jugar con puertos

def start_server():

    host = 'localhost'
    port = 1234

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s: # De esta forma al igual que con archivos el socket o puerto se cerrara solo
        s.bind((host, port)) # Recordemos que devemos pasarle una tupla a nuestro bind par aquedar en escha.
        print(f"\\n[+] Servidor en escucha en {host} por el puerto {port}")
        s.listen(1) # Nos permite definir el numero de conexiónes en simultaneo
        conn, addr = s.accept() # Recordemos que el accept nos devuleve dos valores el socket y la dirección.

        with conn: # una ves salgamos del with se cierra el descriptor de archivo
            print(f"\\n[+] Se ha conectado un nuevo cliente: {addr}")
            while True:
                data = conn.recv(1024) # nos permite recivir mensajes y ademas devemos definir lo que es el tamaño maximo a recivir
                if not data: # en pocas decimos que si el cliente no nos envia nada salimos del bucle
                    break
                conn.sendall(data) # con sendall estamos enviando información.

start_server()
```

cliente Final:

```python
#!/usr/bin/evn python3

import socket

def start_client():

    host = "localhost"
    port = 1234

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:

        s.connect((host, port))
        s.sendall(b"Hola, servidor!!!")
        data = s.recv(1024)

    print(f"[+] Mensaje recibido del servidor: {data.decode()}")

start_client()

```

## En el caso de ser conexiones de tipo UDP es de la siguiente manera:
### **Manejadores de contexto con conexiones**

Los manejadores de contexto (‘**with**‘ en Python) se utilizan para garantizar que los recursos se gestionen de manera adecuada. En el contexto de las conexiones de socket, un manejador de contexto se encarga de abrir y cerrar el socket de manera segura. Esto evita que los recursos del sistema se queden en uso indefinidamente y asegura una gestión adecuada de las conexiones.
### **Diferencias entre send y sendall**

- **send(data)**: El método ‘**send()**‘ se utiliza para enviar una cantidad específica de datos a través del socket. Puede no enviar todos los datos en una sola llamada y puede ser necesario llamarlo múltiples veces para enviar todos los datos.
- **sendall(data)**: El método ‘**sendall()**‘ se utiliza para enviar todos los datos especificados a través del socket. Realiza llamadas repetidas a ‘**send()**‘ internamente para garantizar que todos los datos se envíen por completo sin pérdidas.

La elección entre ‘**send**‘ y ‘**sendall**‘ depende de si se necesita garantizar la entrega completa de los datos o si se permite que los datos se envíen en fragmentos. send puede enviar datos en fragmentos, mientras que sendall garantiza que todos los datos se envíen sin pérdida.

server.py

```python
#!/usr/bin/env python3

import socket

def start_udp_server():
    
    host = "localhost"
    port = 1234

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s: # En el caso del tipo de conexión UDP usamos SOCK_DGRAM.
        s.bind((host, port))
        
        print(f"[+] Servidor UDP iniciado en {locals}-{port}")

        while True:
            data, addr = s.recvfrom(1024) # Como UDP no esta toalmente orientado a conexiónes no es nesesario intablar un three way handshake
            # Es por eso que no hacemos uso del accept y del listen. Con recvfrom nos devuleve las mensajes transmitidos y en addr devuelve la información como el host y port
            print(f"\\n[+] Mensaje enviado por el cliente: {data.decode()}")
            print(f"[+] Información del cliente que nos a enviado el mensaje: {addr}")

start_udp_server()
```

client.py

```python
#!/usr/bin/env python3

import socket

def start_udp_client():

    host = "localhost"
    port = 1234

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s: # Realizamos esto para que la conexión se cierre al temirnal
        # s.sendto(b"Hola aqui esta el cliente", (host, port)) # Jugamos con sendto porque nos permite inviar el mensaje y la tupla del host y port
        # Los mensajes con tildes van a causar un error y para solucionar esto vamos a tener que defirnir de la siguiente manera:

        mensaje ="Hola aquí esta el cliente".encode("utf-8") # de esta forma es una vuena practica realizar para poder enviar tildes y caracteres no acii
        s.sendto(mensaje, (host, port)) 

start_udp_client()

```

## Sockets

La función ‘**setsockopt**‘ en la programación de redes juega un papel crucial al permitir a los desarrolladores ajustar y controlar varios aspectos de los sockets. Los sockets son fundamentales en la comunicación de red, proporcionando un punto final para el envío y recepción de datos en una red.

**Niveles en setsockopt**

Cuando utilizas ‘setsockopt’, puedes especificar diferentes niveles de configuración, que determinan el ámbito y la aplicación de las opciones que estableces:

- **Nivel de Socket (SOL_SOCKET)**: Este nivel afecta a las opciones aplicables a todos los tipos de sockets, independientemente del protocolo que estén utilizando. Las opciones en este nivel controlan aspectos generales del comportamiento del socket, como el tiempo de espera, el tamaño del buffer, y el reuso de direcciones y puertos.
- **Nivel de Protocolo**: Este nivel permite configurar opciones específicas para un protocolo de red en particular, como TCP o UDP. Por ejemplo, puedes ajustar opciones relacionadas con la calidad del servicio, la forma en que se manejan los paquetes de datos, o características específicas de un protocolo.

**socket.SOL_SOCKET**

‘**socket.SOL_SOCKET**‘ es una constante en muchos lenguajes de programación que se usa con ‘setsockopt’ para indicar que las opciones que se van a ajustar son a nivel de socket. Esto significa que las opciones aplicadas en este nivel afectarán a todas las operaciones de red realizadas a través del socket, sin importar el protocolo de transporte específico (como TCP o UDP) que esté utilizando.

**socket.SO_REUSEADDR**

‘**socket.SO_REUSEADDR**‘ es otra opción comúnmente utilizada en setsockopt. Esta opción es muy útil en el desarrollo de aplicaciones de red. Lo que hace es permitir que un socket se enlace a un puerto que todavía está siendo utilizado por un socket que ya no está activo. Esto es particularmente útil en situaciones donde un servidor se reinicia y sus sockets aún están en un estado de “espera de cierre” (**TIME_WAIT**), lo que podría impedir que el servidor se vuelva a enlazar al mismo puerto.

Al establecer ‘**SO_REUSEADDR**‘, el sistema operativo permite reutilizar el puerto inmediatamente, lo que facilita la reanudación rápida de los servicios del servidor.

En resumen, ‘**setsockopt**‘ con diferentes niveles y opciones, como ‘**SOL_SOCKET**‘ y ‘**SO_REUSEADDR**‘, proporciona una flexibilidad significativa en la configuración de sockets para una comunicación de red eficiente y efectiva.

server.py

```python
#!/usr/bin/env python3

import socket
import threading
import pdb # Degugging

class ClientThread(threading.Thread): # creamos la clase y hacemos uso de herencia para poder usar metodos y propiedades de la origina

    def __init__(self, client_sock, client_addr):
        super().__init__() # llamamos al contructor de la clase padre
        # inicializamos las variables
        self.client_sock = client_sock
        self.client_addr = client_addr

        print(f"[+] Nuevo cliente conectado")
    
    # Este es el metodo que sobreescrivimos
    def run(self):

        message = ""

        while True:
            data = self.client_sock.recv(1024) # recivmos el mensaje
            message = data.decode() # le hacemos un decode para verlo en formato normal, recordemos que cuando hacemos el decode se arrastar un salto de linea

            #pdb.set_trace() # Breakpoint Esto nos permite frenar el programa y ver los valores

            if message.strip() == "bye": # establecemos que si el cliente ingresa un bye se termine la conexión osea el bucle, usamos strip para eleminar el salto de linea que nos agrega decode
                break

            print(f"[+] Mensaje enviado por el cliente: {message.strip()}") # Traza para ver que el mensaje del cliente
            self.client_sock.send(data) # enviamos lo mismo que nos envia el cliente

        print(f"[!] El cliente: {self.client_addr} desconectado") # nos permite ver los datos de la conexión
        self.client_sock.close() # cerramos la conexión
            

HOST = 'localhost'
PORT = 1234

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1) # Con esto podemos manipular un nivel a travéz de 3 campos donde (nivel, propiedad, igualdad)
    # la propiedad de SO_REUSEADDR nos permite reutilizar el estado de espera y envez de esperar nos permite utilizarla.
    server_socket.bind((HOST, PORT))

    print(f"[+] En espera de conexiones entrantes")
    
    while True:
        server_socket.listen() # como estamos configurando hilos para las conexiónes podemos dejar sin valor a listen
        client_sock, client_addr = server_socket.accept() # nos devuelve valores de (sock, host, port)
        new_thread = ClientThread(client_sock, client_addr) # instanciamos una clase para poder modificar a nuestro gusto la misma.
        new_thread.start() # Esto nos permite llamar a un metodo run pero lo vamos a sobre escribir en la clase
```

client.py

```python
#!/usr/bin/env python3

import socket

def start_client():
    
    host = "localhost"
    port =1234

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host, port)) # enviamos la tupla para conectarnos

        while True:
            message = input("\\n [+] Introduce el mensaje: ")
            s.sendall(message.encode()) # con esto enviamos el mensaje

            if message == "bye":
                break

            data = s.recv(1024)
            print(f"[+] El mensaje de espuesta es: {data.decode()} ")

start_client()
```

El uso de hilos con ‘**threading**‘ en Python es crucial para gestionar múltiples clientes en aplicaciones de red que utilizan sockets, especialmente en servidores.

Aquí te explico en detalle por qué es necesario:

- **Concurrencia y Manejo de Múltiples Conexiones**: Los servidores de red a menudo necesitan manejar múltiples conexiones de clientes simultáneamente. Sin hilos, un servidor tendría que atender a un cliente a la vez, lo cual es ineficiente y no escalable. Con ‘**threading**‘, cada cliente puede ser manejado en un hilo separado, permitiendo al servidor atender múltiples solicitudes al mismo tiempo.
- **Bloqueo de Operaciones de Red**: Las operaciones de red, como ‘**recv**‘ y ‘**accept**‘, suelen ser bloqueantes. Esto significa que el servidor se detendrá en estas operaciones hasta que se reciba algo de la red. Si un cliente se demora en enviar datos, esto puede bloquear todo el servidor, impidiendo que atienda a otros clientes. Con hilos, cada cliente tiene su propio hilo de ejecución, por lo que la lentitud o el bloqueo de uno no afecta a los demás.
- **Escalabilidad**: Los hilos permiten a los desarrolladores crear servidores que escalan bien con el número de clientes. Al asignar un hilo a cada cliente, el servidor puede manejar muchos clientes a la vez, ya que cada hilo ocupa relativamente pocos recursos del sistema.
- **Simplicidad en el Diseño de la Aplicación**: Aunque existen modelos alternativos para manejar la concurrencia (como la programación asíncrona), el uso de hilos puede simplificar el diseño y la lógica de la aplicación. Cada hilo puede ser diseñado como si estuviera manejando solo un cliente, lo que facilita la programación y el mantenimiento del código.
- **Uso Eficiente de Recursos de CPU en Sistemas Multi-Core**: Los hilos pueden ejecutarse en paralelo en diferentes núcleos de un procesador multi-core, lo que permite a un servidor aprovechar mejor el hardware moderno y manejar más eficientemente varias conexiones al mismo tiempo.
- **Independencia y Aislamiento de Clientes**: Cada hilo opera de manera independiente, lo que significa que un problema en un hilo (como un error o una excepción) no necesariamente afectará a los demás. Esto proporciona un aislamiento efectivo entre las conexiones de los clientes, mejorando la robustez del servidor.

En resumen, el uso de ‘**threading**‘ para manejar múltiples clientes en aplicaciones basadas en sockets es esencial para lograr una alta concurrencia, escalabilidad y un diseño eficiente que aproveche al máximo los recursos del sistema y proporcione un servicio fluido y estable a múltiples clientes simultáneamente.

Server.py

```python
#!/usr/bin/env python3

import socket

def star_chart_server():
    host = "localhost"
    port = 1234

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Genreamos la conexión tipo UTP
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)# es una especia de TIME_WAIT

    server.bind((host, port))
    server.listen(1)

    print(f"\\n[+] Servidor listo para aceptar una conexión......")

    client_socket, client_addr = server.accept()

    print(f"\\n[+] Se a conectado el cliente: {client_addr}")

    while True:

        client_message = client_socket.recv(1024).strip().decode() # Recivimos el mensaje y podemos quitar el salto de linea y deuna ves tambien decodearlo
        print(f"\\n[+] El mensaje del cliente: {client_message}")

        if client_message == "bye":
            break
    
        server_message = input(f"\\n[+] Responde al cliente: ")
        client_socket.send(server_message.encode()) # Enviamos el mensaje al cliente
 
    client_socket.close() # Serramos el servidor una vez que termina el bucle
    

star_chart_server()

```

Client.py

```python
#!/usr/bin/env python3

import socket

def start_chat_client():

    host = "localhost"
    port = 1234

    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client_socket.connect((host, port)) # con esto nos conectamos directamente al servidor

    while True:
        client_messege = input(f"\\n[+] Mensaje a enviar al servidor: ") # definimos un mensaje para enviarlo al servidor
        client_socket.send(client_messege.encode()) # Recordemos que los mensajes tiene que estar en bitcode

        if client_messege == 'byte':
            break

        server_messege = client_socket.recv(1024).strip().decode() # Recivimos el mensaje del servidor
        print(f"\\n[+] Mensaje del servidor: {server_messege}") # Imprimimos el mensaje del servidor

    client_socket.close() # cerramos la conexión

start_chat_client()
```