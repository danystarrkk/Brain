------
El **Server-Side Request Forgery** (**SSRF**) es una vulnerabilidad de seguridad en la que un atacante puede forzar a un servidor web para que realice solicitudes HTTP en su nombre.

En un ataque de SSRF, el atacante utiliza una entrada del usuario, como una URL o un campo de formulario, para enviar una solicitud HTTP a un servidor web. El atacante manipula la solicitud para que se dirija a un servidor vulnerable o a una red interna a la que el servidor web tiene acceso.

El ataque de SSRF puede permitir al atacante acceder a información confidencial, como contraseñas, claves de API y otros datos sensibles, y también puede llegar a permitir al atacante (en función del escenario) ejecutar comandos en el servidor web o en otros servidores en la red interna.

Una de las **diferencias** clave entre el **SSRF** y el **CSRF** es que el SSRF se ejecuta en el servidor web en lugar del navegador del usuario. El atacante **no necesita engañar a un usuario legítimo** para hacer clic en un enlace malicioso, ya que puede enviar la solicitud HTTP directamente al servidor web desde una fuente externa.

Para prevenir los ataques de SSRF, es importante que los desarrolladores de aplicaciones web validen y filtren adecuadamente la entrada del usuario y limiten el acceso del servidor web a los recursos de la red interna. Además, los servidores web deben ser configurados para limitar el acceso a los recursos sensibles y restringir el acceso de los usuarios no autorizados.

En esta clase, estaremos utilizando **Docker** para crear **redes personalizadas** en las que podremos simular un escenario de **red interna**. En esta red interna, intentaremos a través del SSRF apuntar a un recurso existente que no es accesible externamente, lo que nos permitirá explorar y comprender mejor la explotación de esta vulnerabilidad.

Para crear una nueva red en Docker, podemos utilizar el siguiente comando:

➜ `docker network create --subnet=<subnet> <nombre_de_red>`

Donde:

- **subnet**: es la dirección IP de la subred de la red que estamos creando. Es importante tener en cuenta que esta dirección IP debe ser única y no debe entrar en conflicto con otras redes o subredes existentes en nuestro sistema.
- **nombre_de_red**: es el nombre que le damos a la red que estamos creando.

Además de los campos mencionados anteriormente, también podemos utilizar la opción ‘**–driver**‘ en el comando ‘docker network create’ para especificar el controlador de red que deseamos utilizar.

Por ejemplo, si queremos crear una red de tipo “**bridge**“, podemos utilizar el siguiente comando:

➜ `docker network create --subnet=<subnet> --driver=bridge <nombre_de_red>`

En este caso, estamos utilizando la opción ‘**–driver=bridge**‘ para indicar que deseamos crear una red de tipo “**bridge**“. La opción –driver nos permite especificar el controlador de red que deseamos utilizar, que puede ser “**bridge**“, “**overlay**“, “**macvlan**“, “**ipvlan**” u otro controlador compatible con Docker.


## Notas de la practica

El primer escenario de ataque:

![[Pasted image 20250825200346.png]]

En este escenario vamos a tener 2 entorno 1 de producción (el cual podremos ver ) y otro de preproducción que esta esta interno y no podremos ver ya que será solo interno.
Entonces la idea es que veremos el puerto 80 abierto pero en el 8089 no podremos verlo de forma externa.

La idea es que el servicio web en el puerto 80 tenga una funcionalidad como de buscador que nos permita hacer búsquedas por ejemplo si le pasamos `google.com` nos muestre la pagina.

Esto lo vamos a configurar desde 0 nosotros por lo que solo vamos a ir escribiendo los comandos:

- `docker pull ubuntu:latest`
- `docker run -dit --name ssrf_fist_lab ubuntu`
- `docker exec -it ssrf_fist_lab bash` -> ingresamos para instalar las dependencias necesarias en el contenedor.
-  `apt install apache2 php nano python3 lsof`
- `service apache2 start` -> con la IP del docker ya podemos ver la web por defecto de apache pero lo vamos a cambiar.
- Eliminamos el index.html y el archivo utility.php lo creamos con el siguiente contenido:
```php
<?php
        if(isset($_GET['url'])){
                $url = $_GET['url'];
                echo "\n[+] listando el contenido de la web " . $url . ":\n\n";
                include($url);
        } else{
                echo "\n[!] No se puso la url\n";
        }
?>
```

- por útimo vamos a tener un login.html:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login Form</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }

        .login-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
            width: 300px;
            text-align: center;
        }

        h2 {
            margin-bottom: 20px;
            color: #333;
        }

        .form-group {
            margin-bottom: 15px;
            text-align: left;
        }

        label {
            display: block;
            margin-bottom: 5px;
            color: #555;
        }

        input[type="text"],
        input[type="password"] {
            width: calc(100% - 20px);
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }

        button {
            background-color: #007bff;
            color: white;
            padding: 10px 15px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
            font-size: 16px;
        }

        button:hover {
            background-color: #0056b3;
        }

        #message {
            margin-top: 15px;
            color: red;
        }
    </style>
</head>
<body>
    <div class="login-container">
        <h2>Login</h2>
        <form id="loginForm">
            <div class="form-group">
                <label for="username">Username:</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="form-group">
                <label for="password">Password:</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
        <div id="message"></div>
    </div>

    <script>
        document.getElementById('loginForm').addEventListener('submit', function(event) {
            event.preventDefault(); // Prevent default form submission

            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;
            const messageDiv = document.getElementById('message');

            // Simple client-side validation (replace with server-side authentication)
            if (username === 'user' && password === 'pass') {
                messageDiv.style.color = 'green';
                messageDiv.textContent = 'Login successful!';
                // In a real application, you would redirect to another page or set a session
            } else {
                messageDiv.style.color = 'red';
                messageDiv.textContent = 'Invalid username or password.';
            }
        });
    </script>
</body>
</html>
```

- Dentro del directorio `tmp` nos copiamos al html y lo ponemos de igual manera con algo representativo y ya esta, para levantar el servidor vamos a usar: `python3 -m http.server 4646 --bind 127.0.0.1` con esto ya tenemos los dos servicios listos.

Comenzamos.

Lo primero que hacemos es ver si podemos referenciar a la propia maquina enviando una solicitud al local hosts:

![[Pasted image 20250825204150.png]]

como vemos si nos lo permite, ahora esto nos abre una posibilidad de ver puertos abiertos en el sistema de la siguiente manera:

![[Pasted image 20250825204421.png]]

como vemos en la url seria cuestión de ir cambiando los puertos, pero el como podemos darnos cuenta es mediante un fuzzing de la siguiente manera:

```bash
wfuzz -c --hl=3 -t 200 -z range,1-65535 "http://172.17.0.2/utility.php?url=http://127.0.0.1:FUZZ"
```

![[Pasted image 20250825204653.png]]

miramos que en realidad logramos el cometido la maquina es esta haciendo una consulta a si misma y nos figura el puerto 4646 que no esta expuesto como abierto.
por lo tanto vamos a tratar de entrar mediante este buscador que tiene la propia maquina:

![[Pasted image 20250825204843.png]]

como podemos ver ya logramos ver información privilegiada mediante un SSRF.

Ahora procedemos a complicarlo un poco donde el diagrama será de la siguiente manera:
![[Pasted image 20250825205000.png]]

En este caso vamos a tener dos maquina una la cual expone la pagina principal al publico y otra maquina la cual contiene la pagina de pre producción pero las dos maquina tanto la de producción y la de pre producción se encuentran en le mismo segmento de red por lo que las dos maquinas se ven.

- Primero creamos una red con `docker network create --driver=bridge network1 --subnet=10.10.0.0/24`
- Luego creamos el primer contenedor `docker run -dit --name produccion ubuntu`
- Ahora vamos a configurar la red que creamos `docker network connect network1 produccion`
![[Pasted image 20250825210234.png]]
ya tenemos las dos redes en la primera maquina, la cual va a tener la pagina de producción.

Ahora vamos a configurar la maquina de pre-producción: `docker run -dit --name preProduccion --network=network1 ubuntu`

![[Pasted image 20250825210529.png]]

como podemos observar ya tenemos la segunda maquina lista.

Por ultimo vamos a desplegar un docker de atacante  `docker run -dit --name atacante ubuntu`.
esta maquina no tendrá acceso a la subred tampoco y solo podrá acceder a la maquina de producción y no a la de pre-producción.

![[Pasted image 20250825210842.png]]

ya tenemos listas todas las maquinas ahora solo vamos a realizar la configuración primero de la maquina de producción:
`apt install apache2 php nano`
`service apache2 start`
El código de utility.py va a ser mucho mas directo:
```php
<?php
	include($_GET['url']);
?>
```

por ultimo modificamos el archivo php.ini para que nos perta cargar contenido igual que en el primer entorno.
el Código del login.html va a ser el mismo y ya esta listo.

Ahora la maquina de preproducción no es mas que en `tmp` crear un `index.html` con cualquier contenido o el mismo login pero con algo que lo diferencie para saber cual pagina es cual y luego levantamos el servidor python por un puerto `python3 -m http.server 7878`.

Por ultimo en la maquina de atacante vamos a instalar curl y comenzamos.

En general con curl y usando la ruta del utility de la primera maquina podemos conectarnos a la tercera lo que quiere decir que podemos ver contenido dela maquina interna gracias a la maquina de producción.
![[Pasted image 20250825212402.png]]

en la imagen vemos como usamos el comando curl y la `utility.php` para enviar una petición como si fuera la maquina de producción y por esto la maquina que esta en el otro segmento de red responde con normalidad.


## PortSwigger

### Lab: Basic SSRF against the local server

Para este laboratorio tenemos que tratar de explotar la vulnerabilidad que al parecer se encuentra al hacer check en el stock de los productos:
![[Pasted image 20250916203858.png]]

Vemos como se esta efectuando la petición pero tenemos un parámetro algo curioso el cual es `stockApi`, vamos a quitarle el url-encode aver que contiene aunque por lo que se logra observar un `http` podemos pensar que es un link o una url:
![[Pasted image 20250916204038.png]]

Vemos que si efectivamente es una url, y en este punto vamos a ver ya que puede ser inyectable a un SSRF:
![[Pasted image 20250916204147.png]]

Efectivamente vemos los usuarios y el panel de administrador que tenemos allí. Esto no debería ser accesible pero al ejecutar la petición la misma maquina esto si es valido.

El objetivo es eliminar el usuario carlos, es complicado el hacerlo por la interfaz entonces toca buscar el vinculo de como se tramita la petición y enviarlo así para lograr eliminarlo:
![[Pasted image 20250916204904.png]]

Perfecto vamos a tramitar de esa forma la petición para ver que sucede:
![[Pasted image 20250916204956.png]]

al enviar la petición sucedió un follow redirect y es por eso que no vemos directamente la petición pero con esto ya estamos:
![[Pasted image 20250916205059.png]]

Lab Terminado.


### Lab: Basic SSRF against another back-end system

En este lab el objetivo es encontrar ya directamente el panel admin no sabemos la IP, entonces vamos a buscarlo:
![[Pasted image 20250916205733.png]]

en este punto lo que tenemos que hacer es hacer con el burp o con otra herramienta un escaneo para los 254 dispositivos y encontrar el panel administrativo `admin`:
![[Pasted image 20250916210545.png]]

Logramos encontrar la IP `192.168.0.179` donde si tenemos el admin igual forma, ahora vamos a enviar el url para eliminar el usuario `carlos`:
![[Pasted image 20250916210818.png]]
ya se envío de igual forma:

![[Pasted image 20250916210841.png]]

Lab Terminado.

### Lab: Blind SSRF with out-of-band detection

Este lab es relativamente sencillo lo que vamos a hacer es enviarnos una petición a nuestro burp Collaborator.

Vamos primero a capturar la petición principal:
![[Pasted image 20250916212536.png]]

como vemos el lugar inyectable en este punto es el `Referer`, recordemos que tenemos que probar de todo y en todo ya que nosotros no sabemos como se están construyendo las webs, en este caso el `Referer` fue el vulnerable y si vemos el collaborator vamos a ver las peticiones:
![[Pasted image 20250916212802.png]]

![[Pasted image 20250916212812.png]]
Con esto tenemos el Lab Resuelto.


### Lab: SSRF with blacklist-based input filter

En este laboratorio vamos atener un problema y es que tenemos que lograr obtener la información de la web `localhost/admin` pero tienen empleada una blacklist la cual no permite el `localhost`.

![[Pasted image 20250917213416.png]]

Vemos como esta conformada la petición y lo que vamos a hacer es intentar ver dentro de `localhos/admin`:
![[Pasted image 20250917213501.png]]

Vemos que nos lo detecta, es aquí donde tenemos que tratar de burlar la protección:
![[Pasted image 20250917213738.png]]

Tampoco funciona pero nosotros podemos intentar otro tipo de representación como representarlo de forma hexadecimal:
![[Pasted image 20250917213900.png]]

lo que vamos a representar de la siguiente manera: `0x7f000001`:
![[Pasted image 20250917213959.png]]

Tampoco funciona, vamos a intentar a representarlo en decimal;
![[Pasted image 20250917214642.png]]

lo que hacemos es el poner el valor multiplicado por 256 elevado a su posición, contando desde el 0 y desde la derecha, vamos a intentarlo:
![[Pasted image 20250917214843.png]]

Tampoco funciona, en este punto podemos pensar que mas que ser localhost es la cadena admin, podemos probar sin ella a ver si logramos algo:
![[Pasted image 20250917214943.png]]

Efectivamente es la cadena admin, en este punto lo que vamos a hacer es comenzar a probar un doble url-encode haciendo lo siguiente:
![[Pasted image 20250917215212.png]]

En la primera tenemos la primera letra en `url-encode` pero vamos a hacer lo mismo para el `%` el propósito es lograr que en la primera pasada lo inteprete el `%` y luego cambie a la `a` para que se entienda como admin:
![[Pasted image 20250917215415.png]]

Ahora si enviemos a ver si logramos pasar:
![[Pasted image 20250917215450.png]]

Perfecto procedemos a eliminar el usuario Carlos:
![[Pasted image 20250917215527.png]]

Perfecto.
![[Pasted image 20250917215551.png]]

Lab Terminado.


### Lab: SSRF with filter bypass via open redirection vulnerability

En este punto lo que vemos es que tenemos la misma vulnerabilidad pero mediante un open redirect.

![[Pasted image 20250917220012.png]]

bueno en este caso vemos que porque tambien intente que no se puede hacer http y ir a otras paginas así que tenemos que buscar otro factor de ataque:
![[Pasted image 20250917220513.png]]

En uno de los botones encontramos esa forma de ir al siguiente producto mediante un parámetro, en este punto lo que vamos a hacer es intentar hacer uso de ese parámetro en el stock a ver si funciona:
![[Pasted image 20250917221410.png]]

logramos intercambiando la url ver como funciona, procedemos a eliminar la cuenta de Carlos:
![[Pasted image 20250917221508.png]]

![[Pasted image 20250917221530.png]]

Lab Terminado.


### Lab: Blind SSRF with Shellshock exploitation

En esta ocasión lo que vamos a explotar es un `Shellshock` mediante un `SSRF` dentro del sistema, esto es por pasos ya que primero lo que vamos a buscar es la ip correspondiente y luego intentar el Shellshock para obtener el nombre de usuario.

Vamos a comenzar capturando la petición vulnerable, y en realidad es en el Referer:
![[Pasted image 20250917222210.png]]

en este punto vamos a intentar encontrar el valor de posibles host:
![[Pasted image 20250917222737.png]]

para el ataque ponemos nuestra carga util con el objetivo de enviar el nombre de usuario a un servidor del collaborator mediante nslookup, pero como no sabemos cual es la IP con el server vulnerable vamos con el intruder a enviar a todas las IP para que si le atinamos a la correcta podemos obtener el nombre de usuario como nos lo piden:

![[Pasted image 20250917223612.png]]

Bueno vemos que si nos retorna con esto ya tenemos el Lab Resuelto.

### Lab: SSRF with whitelist-based input filter

Este lab es algo extraño, donde tenemos en el stock check el cual lanzan datos al sistema interno y vamos a testearlo:
![[Pasted image 20250917223853.png]]

El objetivo es que se pueda realizar la petición a `http://localhost/admin`, en este punto lo que vamos a hacer es intentar de forma directa a ver que sucede:
![[Pasted image 20250917224042.png]]

Vemos que en este punto nos pide de forma clara el dominio ese, nada de lo anterior nos permite pasar.

En este punto tenemos que recordar que nosotros en las url podemos en ocaciones podemos poner credenciales, vamos a ver si la estructura nos lo permite:
![[Pasted image 20250917224427.png]]

Vemos que el error es diferente  y que carga algo de contenido, esto quiere decir que si estamos ya pasando sobre esa verificación, ahora como vamos hacer para irnos a `/admin` es de la siguiente manera:

![[Pasted image 20250917224750.png]]

Nos aprovechamos el `#` para que el `@stock.weli...` mas que credenciales sea algo a lo que hacer focus, ahora el  `#` dio problemas por lo que se realizo un doble url al `#` y al `%` por eso de los valores. Vemos que carga y podemos ya  intentar entrar a admin:
![[Pasted image 20250917225015.png]]

perfecto ya podemos eliminar el usuario carlos:
![[Pasted image 20250917225052.png]]

![[Pasted image 20250917225103.png]]

Lab Terminado.