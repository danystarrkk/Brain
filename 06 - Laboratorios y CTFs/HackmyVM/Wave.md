-------------------------------------

# Enumeración

Comenzamos identificando la máquina vulnerable en la red con ayuda de`arp-scan`: 
```bash
sudo arp-scan -I vboxnet0 --localnet --ignoredups
```

![[Pasted image 20260906094118.png]]

Observamos que la IP de la máquina es la `192.168.56.102`.

Conociendo la IP vamos a realizar un ping a la misma con el objetivo de identificar el valor de su TTL:
```bash
ping -c 1 192.168.56.102
```

![[Pasted image 20260906094401.png]]

Cuenta con un `TTL=64`, donde si este no ha sido alterado claramente estamo frente a un sistema de base Unix.

Con esta información previa comenzamos con la enumeración de puertos mediante `nmap`:
```bash
sudo nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.56.102 -oG allPorts
```

![[Pasted image 20260906094728.png]]

Como resultado tenemos que el puerto 22 y el puerto 80 estan abiertos corriendo respectivamente los servicios de SSH y HTTP.

Ahora sobre estos puertos abiertos vamos a realizar un escaneo mas exsautivo para descubrir sus versiones he información extra:
```bash
nmap -p22,80 -sVC 192.168.56.102 -oN target
```

![[Pasted image 20260906095036.png]]

En cuanto al puerto 22 no tenemos mucho que hacer, pero, en cambio observamos que en el puerto 80 el servicio de HTTP esta realizado mediante un nginx 1.22.1, además de que gracias a algunos scripts básicos de reconocimiento que lanza nmap podemos observar que a detectado la ruta de `robots.txt`.

Con la detección de `robots.txt` procedemos a revizarlo:

![[Pasted image 20260906095535.png]]

Podemos ver que nos describe a `/backukp`. Recordemos que la ruta de `robots.txt` se utiliza para indicar a los rastreadores de motores de búsqueda lo que no se quiere indexar en la web, no es un estandar ni mucho menos una forma de ocultar rutas debido a que como vemos un atacante podria en realidad a través del mismo enumerar rutas que el desarrollador no quiere que sean del todo publicas.

Ahora procedemos a revizar `/backup`:

![[Pasted image 20260906095838.png]]

Se nos lista algunos backups de vários archivos los cuales, se intentara verificar su contenido pero para esto los descargamos de forma recursiva con ayuda de `wget`:

```bash
wget -r http://192.168.56.102/backup/
```

![[Pasted image 20260906100011.png]]

Al terminar como obsrvamos contamos con una carpeta donde dentro encontraremos todos los archivos:

![[Pasted image 20260906100048.png]]

En este punto voy a pasar cada uno de lo sarchivos por el comando file, esto hos ayuda a identificar el tipo de archivo en báse a su contenido:

```bash
ls | while read file; do file $file; done
```

![[Pasted image 20260906100214.png]]

Cómo podemos observar, contamos con varios archivos simplemente de texto done solo 2 parecen relevantes y son `phptest.bck` y `weevely.bck`, en este punto es lógico si se estan realizando backups de estos archivos posiblemente hablamos de que existen y estan expuestos.

Si revizamos a `phptest.bck` en su contenido encontramos lo siguiente:

![[Pasted image 20260906100436.png]]

Como observamos simplemente es un print y si verificamos su existencia como `phptest.php`en la web pues vemos lo siguiente:
![[Pasted image 20260906100515.png]]

Listo esto no tiene mayor misterio.

Siguiendo esta lódigica con el archivo `weevely.bck`, su contenido es el siguiente:

![[Pasted image 20260906100621.png]]

En este caso como observamos no es legible, pero aquí entra un concepto importante y es que este archivo mediante el comando `file` figuro como un archivo `phar` que es un formato para empaquetar en un solo archivo varios script PHP, por lo tanto vamos mediante el comando `phar` a desempaquetarlo:
```bash
mv weevely.bck weevely.phar
phar extract -f weevely.phar
```
![[Pasted image 20260906100951.png]]

# Explotación

Como se observamos ahora contamos con un nuevo archivos llamado `x` el cual tiene como contenido lo siguiente:

![[Pasted image 20260906101210.png]]

Comprender este código es algo complejo de primeras pero vamos a explicarlo lo mejor posible:

1. Contamos con un eval global y dentro varias variables aqui tenemos que entender que gracias a donde esta el eval y por como funciona es que en realidad combierte a cada uno de las declaraciones en variables globales y usable en el código.
2. Contamos con una funcion que recibe 2 parametros uno es un texto y el otro no es mas que la variable declarada `k` el porque es que dentro de la misma vamos a utilizarla para realizar una combinación XOR y así cifrar y descifrar el cóntenido, al final esta funcion retorna la cadena cifrada o descifrada, recordemos que la combinacióno o cifrado XOR se cifra de forma sencilla y aplicando la misma operatoria podemos realizar el descifrado.
3. Ahora tenemos una condicional esta verfica si dentro de la petición que se realiza al recurso, en este caso a `weevely` contine contenido entre dos cadenas que son las variables `kh` y `kf`, donde si detecta contenido simplemente ingresa.
4. Dentro del condicional contamos con `@ob_start()` y `@ob_end_clean()`, lo que sucede es que eso crea en realidad un nuevo buffer donde todo lo que se encuentre dentro así en este caso especifico ocultando la salidas de los comandos eval en este caso.
5. Tenemos dentro dun `@ob_get_contets()` esta funcion por defecto nos permitiria recuperar la salida del buffer y en este caso guardarla en una variable
6. Tenemos una operatoria dentro con `@eval` simplemente se esta relizando el proceso inverso al contenido que resive y aquí entra lo interezante devido a que se puede utilizar para la ejecucion remota de comandos y tenemos lo que es un **Backdoor**. Continuando pues al contenido se recive se decodifica el base64, luego se pasa a la funcion de XOR por ultimo se descomprime y se evalua.
7. Ahora esa ejecucion el resultado de esa ejecucion vive en el resultado del nuevo buffer se lo guarda en la variable `o` para posteriormente volver a enviarse con el mismo formato en la respuesta de la web donde se imprime y se realizando el mismo proceso del lado del atacante pues tenemos ya el resultado del comando.

Comprendiendo que tenemos un backdor ya en la web que podemos utilizar tenemos que encontrar y ver si el archivo se esta ejecutando. Conocemos que su nombre es `weevely` pero su extencion no así que con ayuda de `goobuster` vamos a realizar fuzzing para determiar la extencion de la siguiente manera:

```bash
ffuf -u "http://192.168.56.102/weevelyFUZZ" -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt
```

![[Pasted image 20260906111824.png]]

Como podemos observar con el resultado es que el archivo es `weevely.php7`.

Ahora conociendo todo el proceso me di a la tarea de crear un pequeño script en python que nos ayude a hacerlo de forma automatica y tener una ejecución de comandos directa, el script lo pueden encontrar en [Aquí](hola).

Con el script es cuestion de hacer lo siguiente:
```bash
python3 backConect.py whoami
```

![[Pasted image 20260906165458.png]]

Como se observa gracias al script podemos ejecutar commandos de forma remota en el servidor.

Con esto simplemente vamos a enviarnos una reverse shell:

![[Pasted image 20260906165923.png]]

 Ya nos encontramos dentro de la máquina.

# Escalando Privilegios 1

Algo que siempre hago es enumerar usuario en la máquina mediante el archivo `/etc/passwd` de la siguiente manera:
```bash
cat /etc/passwd | grep 'sh$'
```

![[Pasted image 20260906204218.png]]

Como podemos observar tenemos 3 usuarios que son `carla`, `angie`, y `root`.

En este caso se realiza el reconocimiento por permisos SUID y pero ejeucion de comandos configuradas por sudo pero no se encontro.

Se procedio a revizar puertos abiertos de forma interna de la siguiente manera:
```bash
ss -nltp
```

![[Pasted image 20260906204559.png]]

En el resultado vemos como el puerto `3923` se encuentra abierto, podemos intentar ver que proceseso se encuentra corriendo y sobre que usaurio lo esta haciendo de la siguiente manera:
```bash
ps aux | grep 127.0.0.1
```
![[Pasted image 20260906205507.png]]
En los resultados vemos dos que nos interesan, donde, se evidencia que se esta ejecutando como el usuario angie un servicio de python con el nombre de `copyparty-sfx.py`. Buscando de que se trata encuentro que es un servidor de archivos completo, rapido y ligero, el cual cuenta con servicio web y esto posiblemente es lo que corre por el puerto `3923`.

Para poder analizarlo mucho mejor vamos a realizar port forwarding mediante la herramienta de `chisel`, recordar que necesitamos el binario tanto en la máquina atacante como en la máquina victima:

![[Pasted image 20260906211110.png]]

Con esto  tenemos listo el tunel, así que vamos a verificarlo entrando desde el navegador a ver que encontramos:

![[Pasted image 20260906211204.png]]

Pefecto ya vemos el portal y es aquí donde tenemos que pensarlo bien, si recordamos el archivo que ejecuta para levantar esto esta en el **Home** por lo tanto como vemos el script en realidad queremos suponer que todo lo que subamos llega a home, además de que claro ya tenemos la flag de usuario.

Con lo  anterior en mente lo que vamos a hacer es subir una `Authorizedkey` esto con el objetivo de poder simplemente conectarnos medite ssh, y para esto la vamos a crear de la siguiente manera:

```bash
ssh-keygen
```
![[Pasted image 20260906211621.png]]
En este caso no es necesario especificar mucho podemos dejarlo tal y como lo genera y no es necesario establecer una contraseña, con la clave creada lo que vamos a hacer es copiar el contenido de la clave publica dentro de un archivo llamado `authorized_keys` y le asignamos los permisos 600.

```bash
touch authorized_keys
echo "contenido de la clave publica" > authorized_keys
chmod 600 authorized_keys
cat autorized_keys
```
![[Pasted image 20260906211931.png]]
Ya con esto vamos a crear una carpeta `.ssh` en el directorio actual y moveremos el archivo dentro:

```bash
mkdir .ssh
mv authorized_keys .ssh
```

![[Pasted image 20260906212122.png]]

Listo ya con esto tenemos todo listo, algo que tenemos que tener en cuenta es que para que esto sea válido es que el directorio que contenga la clave de autorizacion tenga solo permisos de manipulacion para el usuario en pocas permisos 700 así que vamos a darle esso permiso a la carpeta `.ssh`:

```bash
chmod 700 .ssh
```

![[Pasted image 20260906212325.png]]

En este punto creo que el objetivo es claro subir esto mediante el servicio que contamos, el servidor al almacenarlo directamente en el **home** crearemos directamente la carpeta .ssh y su clave autorizada lo que nos permitira una conexión legitima por ssh:

![[Pasted image 20260906212511.png]]
![[Pasted image 20260906212526.png]]

Ya con esto podemos intentar conectarnos como el usuario `angie` a ver si logramos establecer la conexión:

```bash
ssh angie@192.168.56.102
```

![[Pasted image 20260906212642.png]]

Ya estamos dentro como el usuario angie.

# Escalada de Privilegios 2

En este punto realizamos el mismo reconocimiento y en este caso ahora vemos algo al revizar las capacidades de `sudo`:
```bash
sudo -l
```

![[Pasted image 20260906212819.png]]
Listo, podemos ejecutar el comando `/usr/bin/less -F /opt/secret.txt` sin necesidad de contraseña.

Para esto al ser el binario less podemos buscar en GTFOBINS por vulnerabilidades coneocidas:

![[Pasted image 20260906213234.png]]

Encontramos y esta vulnerabilidad se basa en el echo de que archivos extremeadamente largos entra en algo parecido a vim donde dentro del mismo podemos llamara una terminal.

Ahora si lo intentamos ejecutando nuestro archivo veremos que no funciona:

![[Pasted image 20260906213351.png]]

esto se debe a que las dimeciones de la terminal son tan grandes que avarca todo el archivo sin problema por lo tanto vamos a forzar a que la TTY solo pueda imprimir maximo 2 lineas:

```bash
stty rows 2 columns 100
```

![[Pasted image 20260906213507.png]]

Con esto podemos volver a intentarlo y veremos lo siguiente:
```bash
sudo /usr/bin/less -F /opt/secret.txt
```

![[Pasted image 20260906213528.png]]

entramos donde necesitavamos por lo tanto podemos ejecutar la siguiente linea para invocar una bash:
```bash
!/bin/bash
```

![[Pasted image 20260906213649.png]]

Listo como observamos tenemos una terminamos como el usuario `root` y podemos obtener ya la ultima flag.

![[Pasted image 20260906213916.png]]

Máquina Terminada.

# Mitigaciónes

