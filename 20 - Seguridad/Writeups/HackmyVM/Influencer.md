# Enumeración

Comenzamos con un escaneo en red para determinar la IP de la máquina víctima mediante `arp-scan`:

```bash
sudo arp-scan -I vboxnet0 --localnet --ignoredups
```

![[Pasted image 20261009171650.png]]

La IP es la `192.168.56.3` en este caso, así que para intentar intuir el sistema vamos a utilizar el comando `ping`:

```bash
ping -c 1 192.168.56.3
```

![[Pasted image 20261009171854.png]]

Observamos que tenemos `ttl=64` eso quiere decir que el sistema posiblemente es base Unix.

Con esta básica información vamos a comenzar con el escaneo general mediante `nmap`:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.56.3 -oG allPorts
```

![[Pasted image 20261009172255.png]]

Observamos dos puertos abiertos que son el 80 y el 2121, que aparentemente son de http y ccproxy-ftp.

Vamos a realizar sobre estos dos puertos un escaneo más exhaustivo para identificar versiones y datos extras:

```bash
sudo nmap -p80,2121 -sVC 192.168.56.3 -oN target
```

![[Pasted image 20261009172700.png]]

Podemos observar las versiones y gracias a algunos scripts extras de reconocimiento que ejecutó `nmap` logramos observar que mediante el usuario `anonymous` logró listar los archivos contenidos dentro del servidor FTP, además tenemos las versiones de los dos servicios mediante las cuales podemos intuir que posiblemente es un sistema Linux.

Ahora vamos a conectarnos al servidor FTP para intentar extraer los archivos que vemos listados:

```bash
ftp 192.168.56.3 -p 2121
```

![[Pasted image 20261009173235.png]]

Ahora, ya que estamos dentro, vamos a descargar estos archivos:

```bash
mget *
```

![[Pasted image 20261009225047.png]]

Como podemos observar descargamos todos los archivos sin ningún problema y ahora, podemos comenzar a analizarlos.

Lo primero es observar el contenido del archivo `note.txt`:

![[Pasted image 20261011131147.png]]

Vemos que esta nos dice que tiene que cambiar la contraseña del WordPress, de aquí podemos sacar una conclusión y es que posiblemente tenemos alojado un WordPress.

Ahora en cuanto a los archivos, vamos a analizarlos mediante el comando `file` de la siguiente manera:

```bash
ls | grep "?*.jpg$" | while read file; do file $file; done
```

![[Pasted image 20261011131733.png]]
Si observamos todos están correctos, no vemos nada, pero qué tal si analizamos los metadatos de estos archivos a ver qué encontramos:

```bash
exiftool <nombre del archivo>
```

![[Pasted image 20261013173445.png]]
Podemos observar cómo en realidad no tenemos nada en el archivo y repetimos el proceso con los demás.

Ya como último recurso vamos a utilizar la herramienta de `stegseek` para ver si logramos extraer algo de estas imágenes y para esto procesaremos todas con el siguiente comando:

```bash
ls | grep '?*jpg$' | while read file; do stegseek $file; done
```

![[Pasted image 20261013181648.png]]

Podemos observar cómo en una de las imágenes logramos un resultado y en este caso es la de `snapchat.jpg` y el archivo lo extrajo como `snapchat.jpg.out` por lo tanto, vamos a ver su contenido:

![[Pasted image 20261013181749.png]]
Como observamos, tenemos un backup de un password el cual vamos a guardar para luego.

Ya no podemos analizar nada más así que vamos a verificar el servicio web que también tiene:

![[Pasted image 20261013181918.png]]
Como podemos observar tenemos la página por defecto de **Apache2**. Si recordamos anteriormente descubrimos mediante la nota alojada en el servidor FTP que hablaba de un WordPress, por lo tanto, es posible que se trate en específico de ese CMS.

Vamos a realizar Fuzzing con ayuda de `gobuster` con el objetivo de descubrir primero la ruta principal que nos permita ver si tiene contenido además del por defecto de Apache2:

```bash
gobuster dir -u http://192.168.56.3/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20261013182409.png]]
Encontramos ya directamente el directorio `wordpress`, miremos dentro a ver qué encontramos:
![[Pasted image 20261013182533.png]]

Podemos observar que se trata de un blog. Lo primero que voy a hacer es volver a realizar Fuzzing, pero con el propósito de identificar rutas por defecto de WordPress como `wp-admin`, de igual forma utilizaremos la herramienta de `gobuster`:

```bash
gobuster dir -u http://192.168.56.3/wordpress -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20261013183751.png]]
Bueno como observamos, sí tenemos las rutas usuales donde la que más nos interesa en este caso es la de `wp-admin`, así que visitémosla:

![[Pasted image 20261013183911.png]]

Listo, sí podemos ingresar al panel para intentar un inicio de sesión. Ahora no tenemos ni usuario ni contraseñas por lo que tenemos que comenzar a hacer reconocimiento para intentar identificar posibles usuarios, esto en el blog.

![[Pasted image 20261013184053.png]]

![[Pasted image 20261013184104.png]]

Como observamos, en las imágenes tenemos dos posibles usuarios que son `luna` y `Admin` vamos a intentar con estos dos usuarios y la contraseña de respaldo que conseguimos antes:

![[Pasted image 20261013184221.png]]
![[Pasted image 20261013184240.png]]
Vemos que ninguno funcionó, pero si nos fijamos un poco mejor en realidad nos detalla algo importante y es que el mensaje para el usuario `admin` dice que el usuario no está registrado, pero para `luna` el mensaje cambia especificando que la contraseña para el usuario es incorrecta.

Con esto llegamos a la conclusión de que el usuario `luna` existe.

# Explotación

Ahora vamos con el post de `luna` y lo que `admin` le comenta. Si ponemos atención, `luna` nos da una pequeña biografía **Datos Personales** y `admin` le comenta que no use datos personales en la contraseña y que la cambie. Esto es oro, porque nos dice que si `luna` aún no cambia su contraseña, claramente sea una combinación de datos personales.

Con esto en mente vamos a utilizar una herramienta llamada `cupp`, esta podemos descargarla desde el siguiente repositorio: [GitHub Cupp](https://github.com/mebus/cupp)

Ya con el repositorio en nuestra máquina vamos a ejecutarlo en modo interactivo de forma que nos realizará algunas preguntas y tendremos que rellenarlas:

```bash
python3 cupp.py -i
```

![[Pasted image 20261014210124.png]]
Como podemos observar tenemos varias preguntas a las cuales respondemos a medida que nos las solicita, además de que agregamos palabras de acuerdo a sus gustos y pasiones. Con esto logramos crear un diccionario personalizado que podemos utilizar para un ataque de fuerza bruta.

Ahora si revisamos tenemos un archivo nombrado como `luna.txt`:
![[Pasted image 20261014210409.png]]

Vamos a utilizar el diccionario que generamos con la herramienta cupp y el código del script lo pueden descargar de mi repositorio que es [Stark Tools](https://github.com/danystarrkk/Hacking-Tools/tree/main/Tools/Web%20Brute%20Force%20(Influencer)).

```bash
python3 bruteforce.py -u luna -w ../cupp/luna.txt -m http://192.168.56.3/wordpress/wp-login.php
```

![[Pasted image 20261017140628.png]]
Como podemos observar en la imagen, la contraseña es  `luna_1997` así que vamos a iniciar sesión con esas credenciales:

![[Pasted image 20261015182714.png]]
Como observamos, ya estamos dentro del panel administrativo de WordPress.

Ahora vamos a intentar ya desde aquí generar una **Reverse Shell** y para esto vamos a dirigirnos a Appearance:

![[Pasted image 20261015183901.png]]

Vemos que tenemos varios temas con los cuales viene por defecto WordPress, vamos a seleccionar la opción de **Customize** que vemos en el panel izquierdo abajo de `Theme file editor`:

![[Pasted image 20261015184347.png]]

Ya una vez dentro podremos observar cómo en el panel desplegable de la parte izquierda superior podemos seleccionar los temas así que podemos usar cualquiera yo usaré el de Twenty Twenty-One para esta ocasión:

![[Pasted image 20261015190402.png]]

Listo, como observamos, seleccionamos el `index.php` y modificamos su contenido para poder obtener una ejecución remota de comandos.

Ahora es cuestión de forzar la ejecución de ese código, en este caso ingresando a `http://192.168.56.3/wordpress/wp-content/themes/twentytwentyone/index.php` y vamos a agregar en este caso el parámetro que utilizamos para definir un comando con `?cmd=whoami` quedando entonces como `http://192.168.56.3/wordpress/wp-content/themes/twentytwentyone/index.php?cmd=whoami` así ya tenemos la ejecución de comandos remotos:

![[Pasted image 20261015191511.png]]
Listo, ahora si vamos a generar la Reverse Shell.

Dejamos un puerto abierto:

![[Pasted image 20261015210819.png]]
Luego entablamos la conexión de la siguiente manera:

![[Pasted image 20261015211116.png]]
Con esto ya tenemos conexión:

![[Pasted image 20261015211136.png]]

Ahora que ya contamos con conexión, le damos tratamiento para tener correctamente la Shell y continuamos.

# Escalada de Privilegios

## Escalada de Privilegios (luna)

Vamos a comenzar con un reconocimiento pequeño de usuario y directorios:

```bash
whoami
cat /etc/passwd | grep '?*sh$'
ls -la /home/
```
![[Pasted image 20261015211538.png]]
Como observamos, contamos con los usuarios `luna` y `juan`.

Ahora vamos a intentar identificar binarios con permisos `SUID`:

```bash
find / -perm -4000 2>/dev/null
```

![[Pasted image 20261015212210.png]]
De aquí no podemos observar mucho así que vamos a revisar los puertos abiertos:

```bash
ss -tuln
```

![[Pasted image 20261015212453.png]]
Aquí sí observamos cosas interesantes como el puerto 3306, que en realidad es el puerto usual de MySQL. En este punto comprendemos que si tenemos una base de datos, tiene que estar en uso por WordPress así que vamos a buscar a ver si encontramos algo de utilidad.

En las configuraciones del WordPress encontramos credenciales quemadas, exactamente en el archivo `wp-config.php`:

![[Pasted image 20261015213216.png]]

Con esto podemos conectarnos a la base de datos:

```bash
mysql -uwww-data -ps3cret 
```

![[Pasted image 20261015213431.png]]

Como observamos,ya estamos dentro de la base de datos así que vamos a investigar a ver qué encontramos.

![[Pasted image 20261015213705.png]]
Como observamos,encontramos una contraseña encriptada de luna, vamos a guardarla y a seguir haciendo reconocimiento.

Con respecto al hash no se logró el crackeo del mismo, así que lo dejamos.

Si recordamos teníamos otro puerto que no es usual y eso lo vuelve, en cambio, llamativo y es el `1212` y para identificar exactamente de qué se trata podemos intentar conectarnos:

```bash
nc 127.0.0.1 1212
```

![[Pasted image 20261015214237.png]]
Tenemos posibilidad de conexión SSH y podemos intentar primero con las contraseñas que tenemos, así vemos si reutilizan credenciales.

Primero se intentó con la misma que luna tenía en WordPress que es `luna_1997`, pero no fue válida. Así que intentamos con la que encontramos en los archivos que es `u3jkeg97gf`, y esta nos permitió conectarnos:

```bash
ssh luna@127.0.0.1 -p1212
``` 

![[Pasted image 20261016100933.png]]
## Escalada de Privilegios (juan)

Ya estamos como el usuario `luna`, vamos a realizar reconocimiento nuevamente y comenzamos con permisos de sudo:

![[Pasted image 20261016123503.png]]

Como observamos, tenemos permiso para ejecutar como usuario `juan` el binario de  `exiftool`.

Tomando en cuenta el permiso que tenemos vamos a buscar si podemos utilizarlo de alguna forma conveniente:

![[Pasted image 20261016124155.png]]

Como observamos, en realidad dice escritura, pero los archivos son copiados y con los permisos suficientes en realidad son movidos hacia donde queramos.

Tomando eso en cuenta y sabiendo que tenemos conexiones activas por qué no intentar crear dentro del usuario `juan` el respectivo archivo de `authorization_keys` con una clave pública para entablar una conexión sin necesidad de contraseñas.

Para esto primero vamos a crear el par de claves desde luna de la siguiente manera:

```bash
ssh-keygen -t ed25519 -C "test@test.com"
```

![[Pasted image 20261016133926.png]]

Listo, ahora podemos revisar en la ruta `/home/luna/.ssh` si contamos con el par de claves:

![[Pasted image 20261016134030.png]]Perfecto, ahora vamos a realizar una copia del archivo con extensión `.pub` y lo llamaremos `authorized_keys`:

```bash
cp id_ed25519.pub autorized_keys
```

![[Pasted image 20261016134236.png]]
Por último ya vamos a mover primero a un lugar donde el usuario juan sí pueda escribir en este caso a `/tmp`, cambiamos sus permisos con `chmod 766` y luego vamos ya a ejecutar el comando de GTFOBINS de la siguiente manera:

```bash
sudo -u juan /usr/bin/exiftool -filename=/home/juan/.ssh/authorized_keys autorized_keys
```

![[Pasted image 20261016135819.png]]

Ya con esto, vamos a conectarnos utilizando la clave privada en este caso:

```bash
ssh juan@127.0.0.1 -p1212 -i /home/luna/.ssh/id_ed25519
```

![[Pasted image 20261016135903.png]]
Listo, ya estamos como el usuario `juan`.

## Escalada de privilegios (root)

Ahora volvemos con un reconocimiento y comenzamos con permisos sudo:

![[Pasted image 20261016140231.png]]
Como observamos, tenemos los permisos para ejecutar un script que se encuentra en nuestro home, vamos a ver sus permisos y contenidos:

![[Pasted image 20261016140319.png]]

Como observamos, no lo podemos editar en realidad, pero al estar en nuestro directorio podemos moverlo y crear un nuevo archivo nuestro que haga lo que quiera, vamos a hacerlo:

```bash
mv check.sh check.sh.back
touch check.sh
chmod +x check.sh
echo 'chmod +s /bin/bash' > check.sh
```

![[Pasted image 20261016140616.png]]

Listo, ahora con esto vamos a ver si podemos ejecutar y veamos qué sucede:

```bash
sudo -u root /bin/bash /home/juan/check.sh
```

![[Pasted image 20261016140739.png]]
Perfecto le asignamos permisos SUID a la bash y con esto, solo es cuestión de ejecutar `bash -p` y listo:

![[Pasted image 20261016140822.png]]Tenemos privilegios de superusuario.

![[Pasted image 20261016140949.png]]

Máquina Terminada.

# Mitigaciones

En este apartado vamos a analizar cada uno de los fallos de configuración y seguridad que permitieron llegar a tomar el control completo del sistema, junto con sus respectivas soluciones.

## Panel y rutas de WordPress expuestas (y enumeración de usuarios)
Logramos encontrar el panel de administración de WordPress con mucha facilidad debido a que se mantenían las rutas por defecto (`/wp-admin`). Además, el formulario de inicio de sesión revelaba si un usuario existía o no mediante mensajes de error descriptivos ("contraseña incorrecta" vs "usuario no registrado"), lo que facilitó la enumeración de usuarios válidos.
Se propone el evitar dejar las configuraciones por defecto en el CMS. Es recomendable cambiar la ruta del panel de administración utilizando plugins de seguridad (como *WPS Hide Login*) o restringir el acceso a estas rutas mediante reglas de firewall o filtrado por IP. Adicionalmente, se deben unificar los mensajes de error del *login* (ej. "Credenciales inválidas") para evitar la enumeración de usuarios, y establecer políticas de bloqueo tras múltiples intentos fallidos para mitigar ataques de fuerza bruta.

## Acceso anónimo a FTP y exposición de información sensible
El servidor FTP (puerto 2121) tenía habilitado el inicio de sesión para el usuario `anonymous`. Esto permitió listar y descargar libremente archivos de la máquina. Aunque una de las contraseñas estaba enmascarada dentro de una imagen mediante técnicas de esteganografía, el simple hecho de exponer estos archivos públicamente es un riesgo crítico.
Se propone deshabilitar inmediatamente el inicio de sesión anónimo en la configuración del servidor FTP (por ejemplo, `anonymous_enable=NO`). Además, jamás se deben almacenar notas con información interna, backups ni archivos personales en directorios expuestos al público. Todo acceso debe requerir autenticación fuerte y estar cifrado.

## Reutilización de contraseñas (Password Reuse)
La contraseña extraída del archivo enmascarado (`u3jkeg97gf`) fue la misma que el usuario `luna` reutilizó para autenticarse en el servicio SSH expuesto en el puerto 1212.
Se propone implementar políticas de seguridad que fuercen la creación de contraseñas únicas y complejas para distintos servicios. Para el caso específico de SSH, la mejor práctica es deshabilitar por completo la autenticación por contraseña (`PasswordAuthentication no` en `sshd_config`) y forzar el acceso exclusivo mediante pares de claves criptográficas (claves públicas/privadas).

## Permisos Sudo inseguros y movimiento lateral (Exiftool)
El usuario `luna` contaba con permisos de sudo para ejecutar el binario `exiftool` haciéndose pasar por el usuario `juan`. Dado que esta herramienta permite la lectura y copiado de archivos, se abusó de este permiso para forzar la escritura de una clave pública SSH (`authorized_keys`) dentro del directorio personal de `juan`, logrando el movimiento lateral.
Se propone aplicar estrictamente el Principio de Menor Privilegio. Si un usuario necesita ejecutar una herramienta con privilegios de otro, se debe restringir sobre qué directorios o archivos específicos puede operar. Jamás se debe permitir que un comando ejecutado mediante `sudo` tenga capacidad de escritura sobre directorios críticos como `/home/usuario/.ssh`. 

## Mala configuración de permisos en scripts de Sudoers
Se configuró el archivo `sudoers` para que el usuario `juan` pudiera ejecutar el script `check.sh` como `root`. El administrador intentó asegurar la ejecución forzando la ruta absoluta del intérprete (`/bin/bash /home/juan/check.sh`) para evitar un secuestro de ruta (*PATH hijacking*), pero cometió el error de dejar el propio archivo `check.sh` con permisos de escritura y movimiento para el usuario `juan`. Esto permitió reemplazar el script legítimo por uno malicioso que asignaba permisos SUID a la bash.
Se propone que cualquier script o binario que esté autorizado para ejecutarse con privilegios de superusuario a través de `sudo` debe pertenecer única y exclusivamente a `root` (`chown root:root check.sh`), y sus permisos deben ser estrictos (ej. `chmod 755 check.sh`), impidiendo que usuarios sin privilegios puedan editarlo, moverlo o sobrescribirlo.