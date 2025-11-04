-----
# Librería os y sys

Las bibliotecas ‘**os**‘ y ‘**sys**‘ de Python son herramientas esenciales para cualquier desarrollador que busque interactuar eficazmente con el sistema operativo y gestionar el entorno de ejecución de sus programas. Estas bibliotecas proporcionan una amplia gama de funcionalidades que permiten una mayor flexibilidad y control en el desarrollo de software.
## **Biblioteca os**

La biblioteca ‘**os**‘ en Python es una herramienta poderosa para interactuar con el sistema operativo. Proporciona una interfaz portátil para usar funcionalidades dependientes del sistema operativo, lo que significa que los programas pueden funcionar en diferentes sistemas operativos sin cambios significativos en el código. Algunas de sus capacidades incluyen:

- **Manipulación de Archivos y Directorios**: Permite realizar operaciones como crear, eliminar, mover archivos y directorios, y consultar sus propiedades.
- **Ejecución de Comandos del Sistema**: Facilita la ejecución de comandos del sistema operativo desde un programa Python.
- **Gestión de Variables de Entorno**: Ofrece funciones para leer y modificar las variables de entorno del sistema.
- **Obtención de Información del Sistema**: Proporciona métodos para obtener información relevante sobre el sistema operativo, como la estructura de directorios, detalles del usuario, procesos, etc.
## **Biblioteca sys**

La biblioteca ‘**sys**‘ es fundamental para interactuar con el entorno de ejecución del programa Python. A diferencia de ‘**os**‘, que se centra en el sistema operativo, ‘**sys**‘ está más orientada a la interacción con el intérprete de Python. Sus principales usos incluyen:

- **Argumentos de Línea de Comandos**: Permite acceder y manipular los argumentos que se pasan al programa Python desde la línea de comandos.
- **Gestión de la Salida del Programa**: Facilita el control sobre la salida estándar (**stdout**) y la salida de error (**stderr**), lo cual es esencial para la depuración y la presentación de resultados.
- **Información del Intérprete**: Ofrece acceso a configuraciones y funcionalidades relacionadas con el intérprete de Python, como la versión de Python en uso, la lista de módulos importados y la gestión de la ruta de búsqueda de módulos.

Ambas bibliotecas son cruciales para el desarrollo de aplicaciones Python que requieren interacción avanzada con el entorno de sistema y el intérprete. Su comprensión y uso adecuado permite a los desarrolladores escribir código más robusto, portable y eficiente.

librería os:

```python
#!/usr/bin/env python3

import os

directorio_actual = os.getcwd() # Esto nos permite obtener la ruta actual de trabajo

print(f"\\n{directorio_actual}")

files = os.listdir(directorio_actual) # Esto nos permite listar los directorios y los almacena como lista

print(f"\\n{files}")

# os.mkdir("mi_directorio") # Nos permite crear un directorio en una ruta o directorio actual
files = os.listdir(directorio_actual) # usamos para ver si se crea el directorio
print(f"\\n{files}")

print(f"\\n Existe el archivo mi_archivo.txt -> {os.path.exists('mi_archivo.txt')}") # Verifica la existencia de archivos y directorios
print(f"\\n Existe el directorio mi_directorio -> {os.path.exists('mi_directorio')}")

get_env = os.getenv('SHELL') # Nos permite ver varialbes de entorno

print(f"\\n Valor de la variable de entorno: SHELL -> {get_env}")
```

librería sys:

```python
#!/usr/bin/env python3

import sys

print(f"\\n[+] Nombre del script: {sys.argv[0]}") # Esto nos permite manejar argumentos, en este caso el 0 es el nombre del script
print(f"\\n[+] Argumentos totales que se estan pasando a nuestro programa: {len(sys.argv)}") # Nos permite ver el numero de argumentos que se le pasa al scrpt
print(f"\\n[+] Mostrando todos los argumentos: {', '.join(parameter for parameter in sys.argv)}") # de manera general sys.argv nos retorna una lista

print(f"\\n[+] Mostrando las rutas de python: {', '.join(element for element in sys.path)}") # sys.path nos permie ver el path de python

sys.exit(1) # Esto nos permite salir del programa con un código de estado 1, no exitoso

```

# Librería requests

La biblioteca ‘**requests**‘ en Python es una de las herramientas más populares y poderosas para realizar solicitudes HTTP. Su diseño es intuitivo y fácil de usar, lo que la hace una opción preferida para interactuar con APIs y servicios web.
## **Introducción a requests**

‘**requests**‘ es una biblioteca de Python que simplifica enormemente el proceso de enviar solicitudes HTTP. Está diseñada para ser más fácil de usar que las opciones incorporadas en Python, como ‘**urllib**‘, proporcionando una API más amigable.
## **Características Principales**

- **Simplicidad y Facilidad de Uso**: Con requests, enviar solicitudes GET, POST, PUT, DELETE, entre otras, se puede realizar en pocas líneas de código. Su sintaxis es clara y concisa.
- **Gestión de Parámetros URL**: Permite manejar parámetros de consulta y cuerpos de solicitud con facilidad, automatizando la codificación de URL.
- **Manejo de Respuestas**: ‘**requests**‘ facilita la interpretación de respuestas HTTP, proporcionando un objeto de respuesta que incluye el contenido, el estado, los encabezados, y más.
- **Soporte para Autenticaciones**: Ofrece soporte integrado para diferentes formas de autenticación, incluyendo autenticación básica, digest, y OAuth.
- **Manejo de Sesiones y Cookies**: Permite mantener sesiones y gestionar cookies, lo cual es útil para interactuar con sitios web que requieren autenticación o mantienen estado.
- **Soporte para SSL**: ‘**requests**‘ maneja SSL (Secure Sockets Layer) y TLS (Transport Layer Security), permitiendo realizar solicitudes seguras a sitios HTTPS.
- **Manejo de Excepciones y Errores**: Proporciona métodos para manejar y reportar errores de red y HTTP de manera efectiva.
## **Uso Práctico**

La biblioteca se utiliza ampliamente para interactuar con APIs RESTful, automatizar interacciones con sitios web, y en tareas de scraping web. Sus capacidades para manejar solicitudes complejas y sus características de seguridad la hacen ideal para una amplia gama de aplicaciones, desde scripts simples hasta sistemas empresariales complejos.
## **Conclusión**

La comprensión y el uso efectivo de ‘**requests**‘ son habilidades esenciales para cualquier desarrollador Python que trabaje con HTTP y APIs web. Esta biblioteca no solo facilita la realización de tareas relacionadas con la red, sino que también ayuda a escribir código más limpio y mantenible.

```python
#!/usr/bin/evn python3

import requests 

response = requests.get("<https://google.es>") # con esto generamos una petición

print(f"[+] Status code: {response.status_code}") # nos permite ver el código de estado de la petición
print(f"[+] mostrando el código fuente de la respues:\\n {response.text}\\n") # response.text nos permite ver el contenido de la petición

with open("index.html", "w") as f:
    f.write(response.text)
```

```python
#!/usr/bin/env python3

import requests
from requests.api import request

values = {'key1': 'value1', 'key2': 'value2', 'key3': 'value3'} # estosn valores que podemos pasarlos en la petición
headers = {'User-Agent': 'hola hack'}
response = requests.get("<https://httpbin.org/get>", params=values, headers = headers, timeout=1) # Podemos pasarle parametros en formato de llaves cuando son mas de uno pro facilidad

try:
    response = requests.get("<https://httpbin.org/get>", params=values, headers = headers, timeout=1) # Podemos pasarle parametros en formato de llaves cuando son mas de uno pro facilidad
    # con el paametro headers podemos modificar las caveceras de la petición
    # Podemos atraves de timeout defirnir el tiempo de espera para la petición y crear una exepción de la siguiente manera:
    response.raise_for_status() # Esta es excepción

except requests.Timeout: # este tipo de excepcion cuando no carga
    print("\\n[!] No a respondido la web......" )

except requests.HTTPError as http_err: # Este tipo de excepción es cuando la web 
    print(f"[!] Error HTTP: {http_err}")

except requests.RequestException as err: # Este tipo de error engloba a muchos por lo que es bueno usarlo para evitar errores generales
    print("\\n[!] error: {err}")

print(f"\\n[+] URL final: {response.url}") # Esto nos permite ver toda la url que se compone
print(response.text)

print(response.request.headers) # Nos permite ver las caveceras del servidor cuando nos responde, es por eso que usamos el objeto response

# si queremos hacer una petición por post seria:

payload = {'key1': 'value1', 'key2': 'value2', 'key3': 'value3'} 

response = requests.post("<https://httpbin.org/post>", data=payload) # como vemos cuando tramitamos por post devemos pasale data mas no params como en get
print(f"\\n[+] URL final: {response.url}") # Esto nos permite ver toda la url que se compone
print(response.text)

```

```python
#!/usr/bin/env python3

import requests 

respons = requests.get("<https://httpbin.org/get>")

data = respons.json() # Esto nos devuelve los datos en listas por lo que es mucho mas manejable los datos json

if 'headers' in data and 'User-Agent' in data['headers']:
    user_agent = data['headers']['User-Agent']# Al data ser un diccionario lo podemos tratar como tal para ver datos
    print(f"\\n[+] El User-Agent es: {user_agent}")
else:
    print(f"\\n[+] No existe el User-Agent")
```

## **Curiosidades y Aspectos Complementarios de requests**

- **Orígenes y Popularidad**: requests fue creada por Kenneth Reitz en 2011. Su diseño enfocado en la simplicidad y la legibilidad rápidamente la convirtió en una de las bibliotecas más populares en Python. Su lema es “**HTTP for Humans**“, reflejando su objetivo de hacer las solicitudes HTTP accesibles y fáciles para los desarrolladores.
- **Comunidad y Contribuciones**: requests es un proyecto de código abierto y ha recibido contribuciones de numerosos desarrolladores. Esto asegura su constante actualización y adaptación a las nuevas necesidades y estándares de la web.
- **Inspiración en Otros Lenguajes**: El diseño de requests se inspira en otras bibliotecas HTTP de alto nivel de otros lenguajes de programación, buscando combinar lo mejor de cada uno para crear una experiencia de usuario óptima en Python.
- **Extensibilidad**: Aunque requests es poderosa por sí sola, su funcionalidad se puede ampliar con varios complementos. Esto incluye adaptadores para diferentes tipos de autenticación, soporte para servicios como AWS, o herramientas para aumentar su rendimiento.
- **Uso en la Educación y la Industria**: Debido a su simplicidad y potencia, requests se ha convertido en una herramienta de enseñanza estándar para la programación de red en Python. Además, es ampliamente utilizada en la industria para desarrollar aplicaciones que requieren comunicación con servicios web.
- **Casos de Uso Diversos**: Desde la automatización de tareas y el scraping web hasta el testing y la integración con APIs, requests tiene un rango de aplicaciones muy amplio. Su versatilidad la hace adecuada tanto para proyectos pequeños como para aplicaciones empresariales a gran escala.
- **Soporte para Proxies y Timeouts**: requests ofrece un control detallado sobre aspectos como proxies y timeouts, lo cual es crucial en entornos de producción donde la gestión del tráfico de red y la eficiencia son importantes.
- **Manejo Eficiente de Excepciones**: Proporciona una forma clara y consistente de manejar errores de red y HTTP, lo que ayuda a los desarrolladores a escribir aplicaciones más robustas y confiables.

En resumen, requests no solo es una biblioteca de alto nivel para solicitudes HTTP en Python, sino que también es un ejemplo brillante de diseño de software y colaboración comunitaria. Su facilidad de uso, junto con su potente funcionalidad, la convierte en una herramienta indispensable para cualquier desarrollador que trabaje con Python en el ámbito de la web.

```python
#!/usr/bin/env python3

import requests

url = "<https://httpbin.org/post>"
my_file = {'Archivo': open("file.txt", 'r')} # Con esto almacenamos en un diccionario el nombre que nosostros queramos y el contenido, por eso lo habrimos con escritura

response = requests.post(url, files=my_file) # con el campo files podemos enviar archivos

print(response.text)
```

```python
#!/usr/bin/env python3

import requests

url = "<https://httpbin.org/cookies>"

set_cookies_url = "<https://httpbin.org/cookies/set/my_cookies/123123>"

s =requests.Session() # Con esto creamos una session antes de generar la conexión para que siempre nos arrastre las cookies de session

response = s.get(set_cookies_url)
response = s.get(url)

print(response.text)
```

```python
#!/usr/bin/env python3

import requests

response = requests.get("<https://httpbin.org/basic-auth/foo/bar>", auth=('foo', 'bar')) # A través de auth podemos pasarle una tupla la cual contenga, (user, passwd)

print(f"El código de estado es: {response.status_code}") # con .status_code podemos imprimir el código de estado.
```

```python
#!/usr/bin/env python3

import requests

cookies = dict(cookies_are='working') # esto es un diccionario {cookies_are: 'working'}

response = requests.get("<https://httpbin.org/cookies>", cookies=cookies) # para enviar las cookies vamos a usar el diccioario que creamos anteriormente

print(response.text)
```

```python
#!/usr/bin/env python3

from requests import Request, Session

url = "<https://httpbin.org/get>"

s = Session() # Generamos la session de esta manera solo por la forma de importar las lisbrerias

headers = {'Custom-Header': 'my_custom_header'} # Creamos un nuevo header

req = Request('GET', url, headers=headers) #Enviamos de esta manera ya que se importo de forma diferente la libreria

prepped = req.prepare() # Con esto lo que se hace es preparar la solicitud antes de envairla.

prepped.headers['Custom-Header'] = 'my_headers_changet' # Con esto modificamos gracias a prepped
prepped.headers['Another-Header'] = 'this_my_another_header'
response = s.send(prepped) # obiamente tenemos que enviar prepped ya que es la solicitud preparada

print(response.text)
```

```python
#!/usr/bin/env python3

import requests

url = '<http://github.com>'

r = requests.get(url) # Nos permite evitar que nos realice una redirecicón y lo hacemos con ayuda de allow_redirects=False

for redirect in r.history: # con ayuda de r.history podemos ver los tipos de redireccionamiento el cual nos permite ver las paginas por las que pasamos
    print(f"La pagina es: {redirect.url}") 
```

```python
#!/usr/bin/env python3

import requests

with requests.Session() as session: # con ayuda de with vamos a mantener la session activa 
    response1 = session.get("<http://httpbin.org/get>")
    print(response1.text)

    response2 = requests.get("<http://httpbin.org/get>")
    print(response2.text)
```

```python
#!/usr/bin/env python3

import requests

with requests.Session() as session:
    session.auth = ('foo', 'bar') # definimos los valores de la auth

    response = session.get("<https://httpbin.org/basic-auth/foo/bar>") # Generamos la primera session
    print(response.text)

    response2 = session.get("<https://httpbin.org/basic-auth/foo/bar>") # En este punto ya se arrastra la cookie de session gracia a que with lo mantiene activo
    print(response2.text)

```

```python
#!/usr/bin/env python3

import requests

requests.packages.urlib3.disable_warning(requests.packages.urlib3.exceptions.InsecureRequestWarning)
response = requests.get("hhttps://45.4.174.194/", verify=False)
```

# **Librería Urllib3**

‘**urllib3**‘ es una biblioteca de Python ampliamente utilizada para realizar solicitudes HTTP y HTTPS. Es conocida por su robustez y sus numerosas características, que la hacen una herramienta versátil para una variedad de aplicaciones de red. A continuación, se presenta una descripción detallada de ‘**urllib3**‘ y sus capacidades.
## **Descripción Detallada de la Biblioteca urllib3**

### **Funcionalidades Clave**

- **Gestión de Pool de Conexiones**: Una de las características más destacadas de ‘**urllib3**‘ es su manejo de pools de conexiones, lo que permite reutilizar y mantener conexiones abiertas. Esto es eficiente en términos de rendimiento, especialmente cuando se hacen múltiples solicitudes al mismo host.
- **Soporte para Solicitudes HTTP y HTTPS**: ‘**urllib3**‘ ofrece un soporte sólido para realizar solicitudes tanto HTTP como HTTPS, brindando la flexibilidad necesaria para trabajar con una variedad de servicios web.
- **Reintentos Automáticos y Redirecciones**: Viene con un sistema incorporado para manejar reintentos automáticos y redirecciones, lo cual es esencial para mantener la robustez de las aplicaciones en entornos de red inestables.
- **Manejo de Diferentes Tipos de Autenticación**: Proporciona soporte para varios esquemas de autenticación, incluyendo la autenticación básica y digest, lo que la hace apta para interactuar con una amplia gama de APIs y servicios web.
- **Soporte para Características Avanzadas del HTTP**: Incluye soporte para características como la compresión de contenido, el streaming de solicitudes y respuestas, y la manipulación de cookies, ofreciendo así un control detallado sobre las operaciones de red.
- **Gestión de SSL/TLS**: ‘**urllib3**‘ tiene capacidades avanzadas para manejar la seguridad SSL/TLS, incluyendo la posibilidad de trabajar con certificados personalizados y la verificación de la conexión segura.
- **Tratamiento de Excepciones y Errores**: La biblioteca maneja de manera eficiente las excepciones y errores, permitiendo a los desarrolladores gestionar situaciones como tiempos de espera, conexiones fallidas y errores de protocolo.
## **Aplicaciones y Uso**

‘**urllib3**‘ se utiliza en una variedad de contextos, desde scraping web y automatización de tareas, hasta la construcción de clientes para interactuar con APIs complejas. Su capacidad para manejar conexiones de manera eficiente y segura la hace adecuada para aplicaciones que requieren un alto grado de interacción de red, así como para escenarios donde el rendimiento y la fiabilidad son cruciales.
### **Importancia en el Ecosistema de Python**

Si bien existen otras bibliotecas como ‘**requests**‘ que son más amigables para principiantes, ‘**urllib3**‘ se destaca por su control detallado y su rendimiento en situaciones que requieren un manejo más profundo de las conexiones de red. Es una biblioteca fundamental para desarrolladores que buscan un control más granular sobre sus operaciones HTTP/HTTPS en Python.

```python
#!/usr/bin/evn python3

import urllib3

http = urllib3.PoolManager() # Controlador e conexiones

response = http.request('GET', '<https://httpbin.org/get>') # Peticion get

print(response.data.decode()) # tenemos que saver que cuando llamamos la respuesta nos la da en formato bites, es por eso del decode
```

```python
#!/usr/bin/evn python3

import urllib3
import json

http = urllib3.PoolManager() # Controlador e conexiones

data = {'atributo': 'valor'} 
encoded_data = json.dumps(data).encode() # Tenoms que enviar la data en bitecode y para cosas declave valor usamos json.dumps para que nos lo ponga en formato json

response = http.request('POST', '<https://httpbin.org/post>', body=encoded_data, headers={'Content-Type': 'application/json'}) # tenemos que pasarle ese content tipe para que el servidor sepa la data que le pasamos
# de la misma forma podemos crear una nueva cabezera(headers)

print(response.data.decode()) # tenemos que saver que cuando llamamos la respuesta nos la da en formato bites, es por eso del decode
```

```python
#!/usr/bin/env python3

import urllib3

http = urllib3.PoolManager() # Controlador de conexión

response = http.request(
    'GET',
    '<https://httpbin.org/redirect/1>',
)

print(response.status) # Con esto logramos obtener el código de estado de respuesta web
print(f"El código de estado es: {response.status} y su redirector es: {response.get_redirect_location()}")
```

```python
#!/usr/bin/env python3

import urllib3
 
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning) # Nos ayuda a controlar el error por verificación ssl
http = urllib3.PoolManager(cert_reqs="CERT_NONE") # Controlador de conexión y con la ayuda de cert_reqs vamos a ignorar el certificado
response = http.request(
    'GET',
    '<https://45.4.174.194/>',
    redirect = False
)

print(response.data.decode())
```

# Librería Threading y multiprocessing

Las bibliotecas ‘**threading**‘ y ‘**multiprocessing**‘ en Python son herramientas esenciales para la programación concurrente y paralela. Proporcionan mecanismos para ejecutar múltiples tareas simultáneamente, aprovechando mejor los recursos del sistema. A continuación, se presenta una descripción detallada de ambas bibliotecas y sus diferencias.
## **Descripción Detallada de threading y multiprocessing**

### **Biblioteca threading**

‘**threading**‘ es una biblioteca para la programación concurrente que permite a los programas ejecutar múltiples ‘**hilos**‘ de ejecución al mismo tiempo. Los hilos son entidades más ligeras que los procesos, comparten el mismo espacio de memoria y son ideales para tareas que requieren poco procesamiento o que están limitadas por E/S.

- **Uso Principal**: Ideal para tareas que no son intensivas en CPU o que esperan recursos (como E/S de red o de archivos).
- **Ventajas**: Bajo costo de creación y cambio de contexto, compartición eficiente de memoria y recursos entre hilos.
- **Desventajas**: Limitada por el Global Interpreter Lock (GIL) en CPython, que previene la ejecución de múltiples hilos de Python al mismo tiempo en un solo proceso.
### **Biblioteca multiprocessing**

‘**multiprocessing**‘, por otro lado, se enfoca en la creación de procesos. Cada proceso en multiprocessing tiene su propio espacio de memoria. Esto significa que pueden ejecutarse en paralelo real en sistemas con múltiples núcleos de CPU, superando la limitación del GIL.

- **Uso Principal**: Ideal para tareas intensivas en CPU que requieren paralelismo real.
- **Ventajas**: Capacidad para realizar cálculos intensivos en paralelo, aprovechando múltiples núcleos de CPU.
- **Desventajas**: Mayor costo en recursos y complejidad en la comunicación entre procesos debido a espacios de memoria separados.
### **Diferencias Clave**

- **Modelo de Ejecución**: ‘**threading**‘ ejecuta hilos en un solo proceso compartiendo el mismo espacio de memoria, mientras ‘**multiprocessing**‘ ejecuta múltiples procesos con memoria independiente.
- **Uso de CPU**: ‘**multiprocessing**‘ es más adecuado para tareas que requieren mucho cálculo y pueden beneficiarse de múltiples núcleos de CPU, mientras que ‘**threading**‘ es mejor para tareas limitadas por E/S.
- **Global Interpreter Lock (GIL)**: ‘**threading**‘ está limitado por el GIL en CPython, lo que restringe la ejecución en paralelo de hilos, mientras que ‘**multiprocessing**‘ no tiene esta limitación.
- **Gestión de Recursos**: ‘**threading**‘ es más eficiente en términos de memoria y creación de hilos, pero ‘**multiprocessing**‘ es más eficaz para tareas aisladas y seguras en cuanto a datos.
### **Conclusión**

Entender y utilizar adecuadamente ‘**threading**‘ y ‘**multiprocessing**‘ es crucial para optimizar aplicaciones Python, especialmente en términos de rendimiento y eficiencia. La elección entre ambas depende de las necesidades específicas de la tarea, como el tipo de carga de trabajo (CPU-intensiva vs E/S-intensiva) y los requisitos de arquitectura de la aplicación.

```python
#!/usr/bin/env python3

import threading
import time

def tarea(num_tarea):
    print(f"[+] Tarea {num_tarea} inciando")
    time.sleep(2)
    print(f"[+] Tarea {num_tarea} finalizando")

thread1 = threading.Thread(target=tarea, args=(1,)) # esto nos permite crear un hilo, esto nos pemrite ejecuta funciones en paralelo
thread2 = threading.Thread(target=tarea, args=(2,))  # toemenos en cuenta que args se le pasa una tupla y cuando se le pasa tupla de un elmento hacemos (2,)

thread1.start()  # Esta es la forma en la cual iniciamos los hilos
thread2.start()

thread1.join() # Con esto podemos garantizar que los hilos ya terminaro y puede continuar con el coódigo
thread2.join()

print(f"[+] Los hilos han finalizando exitosamente")

```

```python
#!/usr/bin/env python3

import threading
import time

def tarea(num_tarea):
    print(f"[+] Tarea {num_tarea} inciando")
    time.sleep(2)
    print(f"[+] Tarea {num_tarea} finalizando")

thread1 = threading.Thread(target=tarea, args=(1,)) # esto nos permite crear un hilo, esto nos pemrite ejecuta funciones en paralelo
thread2 = threading.Thread(target=tarea, args=(2,))  # toemenos en cuenta que args se le pasa una tupla y cuando se le pasa tupla de un elmento hacemos (2,)

thread1.start()  # Esta es la forma en la cual iniciamos los hilos
thread2.start()

thread1.join() # Con esto podemos garantizar que los hilos ya terminaro y puede continuar con el coódigo
thread2.join()

print(f"[+] Los hilos han finalizando exitosamente")

```

```python
#!/usr/bin/env python3

import requests
import threading
import time

def realizar_peticion(url):
    response = requests.get(url)
    print(f"\\n[+] Dominio: {url}: {len(response.content)} bytes ")

dominios = [
    "<https://google.es>",
    "<https://yahoo.com>",
    "<https://wikipedia.org>"
]

start_time = time.time()

hilos = []
for url in dominios:
    hilo = threading.Thread(target=realizar_peticion, args=(url, )) # Estamos haciendo uso de los hilos por dominio
    hilo.start() # inidicamos cada hilo

    hilos.append(hilo) # Almacenamos cada hilo en una lista pra despues asegurarnos que estos termine

for hilo in hilos:
    hilo.join()  # nos asegramos que los hilos terminen 

end_time = time.time()

tiempo_total = end_time - start_time

print(f"\\n El tiempo total es de: {round(tiempo_total, 2)}")
```

```python
#!/usr/bin/env python3

import requests
import multiprocessing
import time

def realizar_peticion(url):
    response = requests.get(url)
    print(f"\\n[+] Dominio: {url}: {len(response.content)} bytes ")

dominios = [
    "<https://google.es>",
    "<https://yahoo.com>",
    "<https://wikipedia.org>"
]

start_time = time.time()

procesos = []
for url in dominios:
    proceso = multiprocessing.Process(target=realizar_peticion, args=(url, )) # Estamos haciendo uso de procesos
    proceso.start() # inidicamos cada proceso

    procesos.append(proceso) # Almacenamos cada prceso en una lista pra despues asegurarnos que estos termine

for proceso in procesos:
    proceso.join()  # nos asegramos que los procesos terminen

end_time = time.time()

tiempo_total = end_time - start_time

print(f"\\n El tiempo total es de: {round(tiempo_total, 2)}")
```