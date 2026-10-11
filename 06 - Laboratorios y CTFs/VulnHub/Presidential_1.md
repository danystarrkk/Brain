## Enumeración

Vamos a comenzar con un escaneo de la red para identificar la máquina víctima. Para ello, utilizaremos [[Arp-Scan]]:

```
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250830163745.png]]

Como podemos observar, la dirección IP de la máquina víctima es `192.168.1.111`. Vamos a realizar una prueba con `ping` para comprobar si está activa y obtener una primera pista sobre el sistema operativo:

```
ping -c 1 192.168.1.111
```

![[Pasted image 20250830164009.png]]

Ahora, vamos a realizar un escaneo de puertos para identificar los servicios disponibles:

```
nmap -p- --open --min-rate 5000 -sS -v -n -Pn 192.168.1.111 -oG allPorts
```

![[Pasted image 20250830164214.png]]

Como podemos observar, tenemos abiertos los puertos `80` y `2082`. Por lo tanto, vamos a realizar un escaneo más detallado sobre ambos puertos para identificar los servicios y obtener información adicional:

```
nmap -p80,2082 -sCV 192.168.1.111 -oN target
```

![[Pasted image 20250830164418.png]]

Como podemos observar, tenemos una aplicación web en el puerto `80` y un servicio SSH aparentemente ejecutándose en el puerto `2082`. Esto último llama nuestra atención, ya que SSH suele utilizar el puerto `22` por defecto.

En este punto, vamos a investigar la versión de OpenSSH identificada durante el escaneo para comprobar si existen vulnerabilidades conocidas asociadas a ella.

![[Pasted image 20250830164636.png]]

Como referencia, encontramos que existen versiónes de OpenSSH más recientes. Ahora vamos a consultar las entradas disponibles en [[SearchSploit]]:

```
searchsploit openssh
```

![[Pasted image 20250830164848.png]]

Como podemos observar, aparecen distintas entradas relacionadas con OpenSSH. Algunas mencionan posibles escaladas de privilegios, pero todavía no podemos afirmar que sean aplicables a nuestra máquina sin comprobar la versión exacta del servicio y las condiciones necesarias para aprovecharlas.

Por lo tanto, vamos a dejar esta línea de investigación por el momento y centrarnos en la aplicación web, donde todavía tenemos bastante por enumerar.

Primero, vamos a utilizar Gobuster para identificar directorios y recursos disponibles:

```
gobuster dir -u http://192.168.1.111/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

![[Pasted image 20250830170207.png]]

Como podemos observar, encontramos un directorio llamado `cgi-bin`. Esto nos hace pensar en la posibilidad de investigar un [[Ataque Shell Shock]], ya que esta vulnerabilidad puede afectar a determinados entornos que utilizan scripts CGI.

Sin embargo, en este caso no conseguimos encontrar recursos útiles dentro del directorio, por lo que vamos a continuar con la enumeración.

Como todavía no encontramos una vía clara de entrada, se nos ocurre investigar si la aplicación utiliza algún subdominio. Antes de comenzar, vamos a identificar el dominio de la web a partir de la información que encontramos en los correos y de los resultados obtenidos con [[WhatWeb]].

![[Pasted image 20250830170801.png]]

Una vez identificado el dominio `votenow.local`, vamos a utilizar Gobuster para buscar posibles subdominios virtuales:

```
gobuster vhost -u http://votenow.local/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 20 | grep -v "400"
```

![[Pasted image 20250830171234.png]]

En este primer intento no conseguimos encontrar nada interesante. Por lo tanto, vamos a probar con otro diccionario, esta vez utilizando una lista de nombres de directorios y archivos web:

```
gobuster vhost -u http://votenow.local/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --append-domain | grep -v "400"
```

![[Pasted image 20250830171817.png]]

Como podemos observar, conseguimos identificar un subdominio. Vamos a acceder a él para comprobar qué contiene:

![[Pasted image 20250830171937.png]]

Como resultado, encontramos un panel administrativo de PHP. Podemos intentar acceder con nombres de usuario habituales, como `root`, o probar contraseñas por defecto, pero no conseguimos avanzar por esta vía.

En este punto, empezamos a pensar que quizá estamos pasando algo por alto. Por lo tanto, vamos a volver a enumerar la web principal, esta vez teniendo en cuenta las extensiones de archivo que podrían revelar información sensible.

Para ello, utilizaremos Gobuster con varias extensiones:

```
gobuster dir -u http://192.168.1.111/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,zip,html
```

![[Pasted image 20250830172316.png]]

Como podemos observar, encontramos un archivo llamado `config.php`. Vamos a acceder a él para comprobar si contiene información útil:

![[Pasted image 20250830172447.png]]

El archivo parece cargarse, pero no conseguimos visualizar su contenido. Esto no significa necesariamente que esté vacío; es posible que el servidor interprete el código PHP y únicamente muestre el resultado de su ejecución.

Por lo tanto, vamos a ampliar la enumeración y probar extensiones de respaldo, como `bak`, que podrían permitirnos encontrar una copia del archivo original. También podemos buscar variantes como `config.php.bak`, ya que algunos desarrolladores guardan copias de seguridad sin retirarlas del servidor.

Vamos a comprobarlo con Gobuster:

```
gobuster dir -u http://192.168.1.111/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php.bak,bak
```

![[Pasted image 20250830173510.png]]

Como podemos observar, encontramos un archivo llamado `config.php.bak`. Ahora vamos a acceder a él para revisar su contenido:

![[Pasted image 20250830173649.png]]

En la respuesta no conseguimos identificar información útil a simple vista. Sin embargo, si revisamos el código fuente de la página, encontramos lo siguiente:

![[Pasted image 20250830173749.png]]

Como podemos observar, aparecen unas credenciales que llaman nuestra atención:

- **Usuario:** `votebox`

- **Contraseña:** `casoj3FFASPsbyoRP`


Con estas credenciales, vamos a intentar acceder a phpMyAdmin, ya que podrían corresponder a una cuenta de la base de datos:

![[Pasted image 20250830174024.png]]

Perfecto, conseguimos acceder a phpMyAdmin. Ahora vamos a comprobar la versión que está utilizando la aplicación para investigar si existen vulnerabilidades conocidas:

```
searchsploit phpmyadmin 4.8.1
```

![[Pasted image 20250830174441.png]]

Como podemos observar, encontramos una entrada relacionada con phpMyAdmin que parece permitir la inclusión de archivos locales. Vamos a revisar cómo funciona para determinar si podemos adaptarla al escenario que tenemos delante.

## Explotación

![[Pasted image 20250830180725.png]]

Al revisar el exploit, encontramos una ruta que parece estar relacionada con un ataque de tipo _Path Traversal_. La vulnerabilidad que nos interesa es CVE-2018-12613, que afecta a phpMyAdmin 4.8.0 y 4.8.1 y puede permitir la inclusión de archivos locales bajo determinadas condiciones.

Con esto en mente, vamos a comprobar si podemos aprovechar esta vulnerabilidad para realizar un [[Local File Inclusion (LFI)]] y comenzar a enumerar archivos del sistema.

![[Pasted image 20250830181114.png]]

Como podemos observar, conseguimos acceder al archivo `/etc/passwd`. Esto nos permite revisar las cuentas registradas en el sistema y buscar posibles usuarios que puedan ser interesantes para continuar con la explotación.

![[Pasted image 20250830181230.png]]

Entre los usuarios que encontramos, destaca `admin`. Por lo tanto, vamos a comprobar si podemos acceder a sus archivos personales o si dispone de claves SSH que podamos aprovechar.

![[Pasted image 20250830181424.png]]

Sin embargo, no conseguimos listar los archivos que nos interesan. Esto podría deberse a restricciones de permisos o a que la ruta que estamos intentando consultar no es accesible mediante esta técnica.

Como todavía tenemos una vulnerabilidad LFI, vamos a investigar si podemos acceder a los registros de Apache y comprobar si existe alguna vía para aprovecharlos mediante [[Log Poisoning (LFI - RCE)]].

![[Pasted image 20250830181527.png]]

En este caso, tampoco conseguimos acceder al archivo de registros que estamos buscando. Por lo tanto, vamos a cambiar de enfoque y comprobar qué servicios están disponibles internamente en la máquina.

Una forma de hacerlo es revisar el archivo `/proc/net/tcp`, que contiene información sobre las conexiones TCP y los puertos asociados. Esto nos permitirá investigar servicios que no necesariamente están expuestos desde el exterior.

![[Pasted image 20250830181656.png]]

Como podemos observar, obtenemos información sobre las conexiones y los puertos internos. Sin embargo, los valores de los puertos aparecen en hexadecimal, por lo que primero debemos convertirlos para interpretarlos correctamente.

Vamos a guardar la información obtenida en un archivo llamado `data` y utilizar el siguiente comando para extraer los puertos locales y convertirlos:

```
for port in $(cat data | awk '{print $2}' | awk '{print $2}' FS=":" | sort -u); do echo "[+] Puerto $port -> $((0x$port)) "; done
```

![[Pasted image 20250830182232.png]]

Como resultado, identificamos los puertos `80`, `2082` y `3306`. Los dos primeros coinciden con los servicios que habíamos detectado previamente, mientras que el puerto `3306` suele estar asociado a MySQL.

No obstante, debemos tener en cuenta que `/proc/net/tcp` puede mostrar conexiones en diferentes estados. Para confirmar que el puerto `3306` está escuchando localmente, debemos comprobar su dirección local y el estado correspondiente.

Con esto en mente, vamos a investigar si existe un servicio MySQL ejecutándose internamente. Para ello, revisaremos la información disponible en `/proc/sched_debug`, tal como se muestra en la captura:

![[Pasted image 20250830182441.png]]

Como podemos observar, encontramos una referencia al proceso `mysql`, lo que refuerza nuestra hipótesis de que el servicio de base de datos está ejecutándose en la máquina.

Ahora, vamos a utilizar la información obtenida de `/proc/net/tcp` para comprobar la dirección IP de la máquina y confirmar que estamos examinando el sistema esperado.

Debemos tener en cuenta que las direcciónes IPv4 que aparecen en `/proc/net/tcp` se representan en hexadecimal y con los bytes en orden inverso. En nuestro caso, el valor `6F01A8C0` corresponde a `192.168.1.111` después de invertir el orden de los bytes. Una vez reordenado, podemos convertir cada octeto con el siguiente comando:

```
echo "$((0xC0)).$((0xA8)).$((0x01)).$((0x6F))"
```

![[Pasted image 20250830182844.png]]

Como podemos observar, la dirección obtenida coincide con la IP de la máquina víctima. Esto nos permite confirmar que estamos investigando el sistema esperado.

También podemos revisar `/proc/net/fib_trie`, un archivo que contiene información sobre las rutas IPv4 del sistema y que puede ayudarnos a identificar las direcciónes IP configuradas:

![[Pasted image 20250830183039.png]]

Una vez que hemos enumerado los archivos y servicios internos, vamos a volver a la vulnerabilidad LFI para intentar conseguir ejecución remota de comandos.

![[Pasted image 20250830183507.png]]

En este punto, vamos a aprovechar la inclusión local para acceder a un archivo temporal asociado a la sesión de PHP. Para construir la ruta correcta, necesitamos identificar el ID de nuestra sesión, que podemos obtener de la cookie correspondiente.

Además, debemos recordar la ruta que identificamos al revisar el exploit de phpMyAdmin. La ruta de inclusión debe ajustarse al archivo temporal de sesión real; no basta con cambiar `sesións` por `sesión` sin comprobar que esa es la ruta correcta en el sistema.

![[Pasted image 20250830183645.png]]

Como podemos observar, al realizar la inclusión correctamente, conseguimos visualizar información relacionada con la sesión. Esto nos indica que estamos accediendo al archivo temporal que queríamos investigar.

Ahora, vamos a comprobar si podemos aprovechar este comportamiento para ejecutar código PHP. La idea consiste en introducir contenido controlado por nosotros en la sesión y después incluir ese archivo mediante la LFI. Esta técnica se conoce como _PHP sesión poisoning_ combinado con LFI, no como _Log Poisoning_, ya que el archivo que estamos manipulando es el de sesión y no un registro del servidor.

Para comprobarlo, vamos a acceder a la sección de SQL y ejecutar una consulta sencilla que nos permita introducir un texto de prueba:

![[Pasted image 20250830183905.png]]

Como podemos observar, la aplicación muestra el texto `Hola`. Ahora, si regresamos a la página anterior y la recargamos, podremos comprobar cómo se refleja ese contenido:

![[Pasted image 20250830184004.png]]

En efecto, vemos que se ha incluido el texto `Hola` y que más abajo aparece la consulta `SELECT "hola"`. Esto nos permite comprobar que el contenido de la consulta se está almacenando en la sesión que estamos incluyendo.

Con esto en mente, vamos a comprobar si podemos introducir código PHP en lugar del texto de prueba:

![[Pasted image 20250830184134.png]]

Ahora, vamos a comprobar si el servidor interpreta el código que hemos introducido:

![[Pasted image 20250830184218.png]]

Como podemos observar, el código se está interpretando correctamente. Por lo tanto, hemos conseguido aprovechar la combinación de la SQL Injection y la LFI para ejecutar comandos en el sistema.

En este punto, vamos a intentar obtener una reverse shell para trabajar de forma más cómoda. Para ello, primero prepararemos un listener en nuestra máquina atacante y después provocaremos la ejecución del código desde la aplicación web.

![[Pasted image 20250830184803.png]]

Antes de recargar la página que contiene el archivo de sesión, vamos a dejar el puerto `443` en escucha para recibir la conexión:

![[Pasted image 20250830184828.png]]

Perfecto, conseguimos recibir una shell. Sin embargo, hay algo importante que debemos tener en cuenta: después de establecer la conexión, la aplicación web deja de funcionar como esperamos. Por lo tanto, vamos a cerrar esa sesión y continuar investigando las posibles vías de acceso al sistema.

En este punto, nos interesa revisar la base de datos para comprobar si encontramos información que podamos utilizar para acceder a una cuenta del sistema.

![[Pasted image 20250830185506.png]]

Como podemos observar, encontramos un hash asociado al usuario `admin`. Recordemos que anteriormente identificamos una cuenta con ese nombre en el sistema, por lo que vamos a intentar recuperar la contraseña y comprobar si podemos utilizarla para acceder a ella.

Para ello, utilizaremos John the Ripper junto con la lista de contraseñas `rockyou.txt`:

```
john -w:/usr/share/wordlists/rockyou.txt hash
```

En este punto, debemos tener en cuenta que un ataque de fuerza bruta puede tardar bastante tiempo, dependiendo de la contraseña y de los recursos disponibles. Por lo tanto, vamos a esperar a que termine el proceso para comprobar si conseguimos recuperar la contraseña.

![[Pasted image 20250830190814.png]]

Como podemos observar, conseguimos recuperar la contraseña. Ahora vamos a generar nuevamente la reverse shell y, una vez dentro, intentaremos acceder al usuario `admin` utilizando las credenciales obtenidas.

![[Pasted image 20250830191246.png]]

Perfecto, ya estamos ejecutando comandos como el usuario `admin`. A partir de aquí, vamos a enumerar el sistema para identificar posibles vías de escalada de privilegios.

![[Pasted image 20250830191347.png]]

Como podemos observar, encontramos una nota que menciona que se utilizaron determinados comandos y que se comprimieron archivos sensibles. Esta información podría estar relacionada con alguna configuración especial del sistema, por lo que vamos a seguir investigando.

Primero, vamos a identificar la distribución y su versión:

```
cat /etc/os-release
```

![[Pasted image 20250830191603.png]]

Ahora, vamos a comprobar la versión del kernel:

![[Pasted image 20250830191650.png]]

Como podemos observar, el kernel es relativamente antiguo, aunque no encontramos una vía de explotación evidente únicamente a partir de su versión.

Por lo tanto, vamos a revisar los permisos del usuario, las reglas de `sudo` y los ejecutables que tienen asignado el bit SUID. Para ello, podemos utilizar comandos como `sudo -l` y `find` para buscar archivos con permisos especiales.

Sin embargo, en estas comprobaciones no encontramos nada que nos permita escalar privilegios de forma directa. Por lo tanto, vamos a investigar otra característica de Linux que puede resultar interesante: las _capabilities_.

## Escalada de privilegios

En Linux, las _capabilities_ permiten asignar permisos específicos a determinados procesos o ejecutables sin necesidad de otorgarles todos los privilegios de `root`. Algunas de ellas pueden resultar especialmente peligrosas si se asignan a programás que permiten acceder a archivos protegidos.

Vamos a enumerar las capabilities presentes en el sistema:

```
getcap -r / 2>/dev/null
```

![[Pasted image 20250830192344.png]]

Como podemos observar, encontramos un archivo llamado `tarS` que tiene asignada la capability `cap_dac_read_search`.

Esta capability permite omitir determinadas comprobaciones de permisos de lectura y búsqueda de directorios. Por lo tanto, un programa que la tenga asignada puede acceder a archivos que normalmente estarían fuera del alcance del usuario que lo ejecuta, siempre que el programa utilice esa capability.

En este punto, recordamos la nota que encontramos anteriormente, en la que se mencionaba que se habían comprimido archivos sensibles. Esto nos hace pensar que el ejecutable `tarS` podría ser precisamente la herramienta que necesitamos investigar.

Vamos a ejecutarlo para comprobar cómo funciona y qué opciones nos proporciona:

![[Pasted image 20250830193456.png]]

Como podemos observar, el programa nos muestra su ayuda y podemos comprobar las opciones disponibles. Con esto en mente, vamos a intentar aprovechar la capability para leer un archivo al que normalmente no tendríamos acceso, utilizando el programa para crear un archivo comprimido.

![[Pasted image 20250830193648.png]]

En efecto, conseguimos crear el archivo comprimido. Ahora vamos a descomprimirlo y comprobar si podemos acceder a su contenido:

![[Pasted image 20250830193745.png]]

Al intentar leer el archivo extraído, obtenemos un mensaje de `Permission denied`. Sin embargo, si revisamos los propietarios y permisos del archivo, encontramos lo siguiente:

![[Pasted image 20250830193830.png]]

Como podemos observar, somos los propietarios del archivo extraído. Por lo tanto, vamos a modificar sus permisos para poder leerlo y comprobar su contenido:

![[Pasted image 20250830193929.png]]

Perfecto, conseguimos acceder a su contenido. Esto demuestra que podemos aprovechar la capability del ejecutable para leer archivos que, en condiciones normales, estarían protegidos frente a nuestro usuario.

Ahora, recordemos que también tenemos un servicio SSH ejecutándose en el puerto `2082`. Como el usuario `admin` no dispone de privilegios suficientes para acceder directamente a los archivos de `root`, vamos a investigar si podemos utilizar `tarS` para obtener una copia de su clave privada SSH.

En concreto, vamos a intentar comprimir el archivo `/root/.ssh/id_rsa` y después extraerlo para comprobar si conseguimos recuperar la clave:

![[Pasted image 20250830194326.png]]

![[Pasted image 20250830194341.png]]

Perfecto, conseguimos obtener la clave privada SSH del usuario `root`. Ahora vamos a utilizarla para intentar conectarnos a la máquina como ese usuario. Recordemos que las claves privadas deben tener permisos restrictivos para que SSH las acepte.

Por lo tanto, vamos a establecer los permisos adecuados y realizar la conexión a través del puerto `2082`:

```
chmod 600 id_rsa
ssh -i id_rsa root@localhost -p 2082
```

![[Pasted image 20250830194552.png]]

Como podemos observar, conseguimos acceder como `root`. De esta manera, hemos completado la escalada de privilegios y obtenido el control total de la máquina.

![[Pasted image 20250830194645.png]]

**¡Máquina terminada!**