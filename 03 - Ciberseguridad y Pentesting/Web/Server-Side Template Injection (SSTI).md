---
aliases: [SSTI, Inyección de Plantillas del Lado del Servidor]
tags: [hacking/explotacion, hacking/web, hacking/laboratorios, herramientas/burpsuite, hacking/herramientas, programacion/python, redes/conceptos]
estado: 🟢 Terminado
---
# Server-Side Template Injection (SSTI)

El **Server-Side Template Injection** (**SSTI**) es una vulnerabilidad de seguridad en la que un atacante puede inyectar código malicioso en una **plantilla** de servidor.

> [!info] ¿Qué es una plantilla de servidor?
> Las plantillas de servidor son archivos que contienen código que se utiliza para generar **contenido dinámico** en una aplicación web. Permiten separar la lógica de negocio de la presentación, facilitando el desarrollo y mantenimiento. Ejemplos comunes incluyen Jinja2 (Python), Twig (PHP), ERB (Ruby), FreeMarker (Java), y Handlebars (JavaScript).

Los atacantes pueden aprovechar una vulnerabilidad de SSTI para inyectar código malicioso en una plantilla de servidor, lo que les permite ejecutar comandos en el servidor y obtener acceso no autorizado tanto a la aplicación web como a posibles datos sensibles.

Por ejemplo, imagina que una aplicación web utiliza plantillas de servidor para generar correos electrónicos personalizados. Un atacante podría aprovechar una vulnerabilidad de **SSTI** para inyectar código malicioso en la plantilla de correo electrónico, lo que permitiría al atacante ejecutar comandos en el servidor y obtener acceso no autorizado a los datos sensibles de la aplicación web.

En un caso práctico, los atacantes pueden detectar si una aplicación Flask está en uso, por ejemplo, utilizando herramientas como **WhatWeb**. Si un atacante detecta que una aplicación **Flask** está en uso, puede intentar explotar una vulnerabilidad de **SSTI**, ya que Flask utiliza el motor de plantillas **Jinja2**, que es vulnerable a este tipo de ataque.

> [!tip] Jinja2 y su Sandbox
> Jinja2, como muchos motores de plantillas, implementa un *sandbox* para limitar las operaciones que se pueden realizar. Sin embargo, existen técnicas conocidas para evadir este sandbox, a menudo explotando la forma en que Python maneja los objetos y sus atributos (`__class__`, `__base__`, `__subclasses__`, `__globals__`).

Para los atacantes, detectar una aplicación Flask o Python puede ser un primer paso en el proceso de intentar explotar una vulnerabilidad de SSTI. Sin embargo, los atacantes también pueden intentar identificar vulnerabilidades de SSTI en otras aplicaciones web que utilicen diferentes *frameworks* de plantillas, como Django, Ruby on Rails, entre otros.

Para prevenir los ataques de SSTI, los desarrolladores de aplicaciones web deben validar y filtrar adecuadamente la entrada del usuario y utilizar herramientas y *frameworks* de plantillas seguros que implementen medidas de seguridad para prevenir la inyección de código malicioso.

## Notas de la práctica

Vamos a desplegar la máquina de la siguiente manera:

```bash
docker run -p 8089:8089 -d filipkarc/ssti-flask-hacking-playground
```

Ya con esto tenemos la máquina lista para practicar.

![[Pasted image 20250826081555.png]]

Vemos que esa es la web donde vamos a ingresar algo a ver qué hace:

![[Pasted image 20250826081627.png]]

Vemos que lo que escribamos lo representa en el título de la web, pero también en la URL, lo que permite que cambiemos el dato desde allí.

Vamos a analizar la web con ayuda de [[WhatWeb Herramienta de Fingerprinting|WhatWeb, Fingerprinting Web]] para ver qué obtenemos:

```bash
whatweb "http://localhost:8089"
```

![[Pasted image 20250826081821.png]]

> [!info] WhatWeb
> WhatWeb es una herramienta de código abierto que identifica sitios web. Detecta tecnologías web, incluyendo sistemas de gestión de contenido (CMS), plataformas de blogs, paquetes estadísticos/analíticos, librerías JavaScript, servidores web y dispositivos de red. Es útil para el reconocimiento inicial.

Vemos que nos da la versión de Python. Ahora, siempre que veamos Python o algún *framework* relacionado a este y que además nosotros podemos controlar algún tipo de *output*, como en este caso la palabra `stark`, podemos probar lo siguiente:

![[Pasted image 20250826082110.png]]

Si se realiza la operación, entonces ya estamos teniendo un problema:

![[Pasted image 20250826082140.png]]

Como podemos observar, sí se ejecuta la operatoria, lo que indica ya un **SSTI**.

Nosotros nos podemos ayudar de la siguiente web: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection, donde nos dará para diferentes lenguajes una gran cantidad de y diferentes inyecciones para cada uno. En este caso, usamos los de Python para `jinja2`:

> [!tip] PayloadsAllTheThings
> Este repositorio es una referencia invaluable para *testers* de seguridad, ya que compila una vasta colección de *payloads* para diversas vulnerabilidades, incluyendo SSTI, XSS, SQLi, etc., organizados por tipo de vulnerabilidad y tecnología.

![[Pasted image 20250826082513.png]]

Como podemos ver, tenemos algunas inyecciones básicas, por lo que podemos ir intentando.

Bueno, si seguimos revisando, nos vamos a encontrar con que podemos hacer múltiples cosas, como leer de forma remota archivos, por ejemplo:

```
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```

![[Pasted image 20250826082758.png]]

Como vemos, logramos cargar archivos de la máquina y ya esto es crítico. No todos los *payloads* que encontremos funcionarán, por lo que siempre toca ir probando uno por uno esperando que uno funcione.

Esto puede llegar a ser crítico, ya que se pueden llegar a ejecutar hasta comandos:

![[Pasted image 20250826083015.png]]

Podemos ver que con otro *payload* logramos ejecutar comandos.

## PortSwigger

### Lab: Basic server-side template injection

En este laboratorio, el objetivo es encontrar una inyección de comandos dentro de la web que nos permita la eliminación del archivo `morale.txt` que se encuentra dentro del directorio *home* del usuario `carlos`.

Vamos a revisar un poco la web y ver cómo funciona:

![[Pasted image 20250922135312.png]]

Vemos que, mediante un parámetro en la URL, se está insertando información dentro de la web. Podemos aquí intentar inyectar comandos. Tenemos que saber que los comandos van a variar dependiendo completamente de la plantilla, si es Python o Ruby o cualquier otro tipo.

En este caso, es Ruby. Vamos a intentar lo siguiente:

![[Pasted image 20250922135534.png]]

La inyección, ya que no se ve bien, fue `<%=7*7%>`. Además, como vemos, no nos imprime el contenido de la inyección, sino que imprime el resultado de la operatoria.

Una recomendación es siempre buscar en https://github.com/swisskyrepo/PayloadsAllTheThings; siempre vamos a encontrar opciones de inyecciones para diferentes lenguajes:

![[Pasted image 20250922135832.png]]

Vemos de ejemplo la inyección que hicimos, pero en este caso vamos a buscar algo válido que permita ejecutar comandos:

![[Pasted image 20250922140014.png]]

Tenemos varias opciones, pero la que vemos en azul es la que voy a usar:

![[Pasted image 20250922140038.png]]

Aunque no se aprecia del todo bien, vemos directorios típicos de la raíz como `mnt`, `opt`, `proc` y más. Vamos a intentar entonces eliminar el archivo que nos piden, que se encuentra en `/home/carlos`:

![[Pasted image 20250922140236.png]]

Perfecto, ya tenemos el archivo que queremos eliminar. Es cuestión de hacer un `rm morale.txt`:

![[Pasted image 20250922140335.png]]

Lab Terminado.

### Lab: Basic server-side template injection (code context)

En este laboratorio, tenemos que encontrar la forma de ejecutar comandos en plantillas de Tornado y el objetivo es eliminar el archivo `morale.txt`.

Bueno, comencemos haciendo *login*:

![[Pasted image 20250922200959.png]]

Vemos que podemos cambiar el nombre preferido, que no es más que en un *post* cualquiera se refleje lo que se le defina. Vamos a interceptar esta petición a ver cómo se está tramitando:

![[Pasted image 20250922201342.png]]

Vemos un `user.nickname`, el cual podemos de alguna forma asociarlo a objetos en programación. Nosotros no conocemos el lenguaje empleado ni la plantilla, por lo cual lo que vamos a hacer es identificar lo que se está usando. Para esto, la forma más fácil de hacerlo es provocando errores de la siguiente manera:

![[Pasted image 20250922221642.png]]

Intercepto la petición y la envío modificada donde `user.nickname` lo cambio o modifico por algo o, como en el caso de la imagen, es `user.nicknametest`. Vemos qué ocurre:

![[Pasted image 20250922222004.png]]

> [!info] Identificación de Motores de Plantillas
> Provocar errores controlados es una técnica común para identificar el motor de plantillas subyacente. Los mensajes de error a menudo revelan el nombre del motor (ej. Jinja2, Twig, Tornado) o el lenguaje de programación, lo que permite al atacante buscar *payloads* específicos para esa tecnología.

Con esto podemos intentar buscar algo útil en *PayloadsAllTheThings* a ver qué encontramos:

![[Pasted image 20250922222122.png]]

Vamos a intentar lo que tenemos allí:

![[Pasted image 20250922222325.png]]

![[Pasted image 20250922222348.png]]

Vemos que causamos un error. Tenemos que interpretarlo como cuando jugamos en los XSS, donde primero tratamos de cerrar las etiquetas anteriores. Aquí también vamos a intentar cerrar el *placeholder* anterior, por decirlo de alguna forma, de la siguiente manera:

![[Pasted image 20250922222631.png]]

Podemos observar cómo primero cerramos lo anterior y luego abrimos alguna nueva petición. Ahora no cerramos la nueva porque por detrás lo que se tenía es `{{user.nickname}}`. Entonces, cuando cerramos el primero, obtenemos `{{user.nickname}}}}`. Vemos que tenemos 4 de cierre, ya que se arrastran los dos que teníamos. Ahora, si agregamos la inyección, tenemos `{{user.nickname}}{{7*7}}` y con esto tenemos dos peticiones que al momento de enviarla obtenemos:

![[Pasted image 20250922222843.png]]

Perfecto, ya vemos el 49. De la misma página que revisamos las inyecciones, vamos a ver alguna forma de ejecutar comandos:

![[Pasted image 20250922223231.png]]

![[Pasted image 20250922223246.png]]

Perfecto, como podemos observar, tenemos el usuario que es `carlos`. En este punto, solo resta eliminar el archivo que se nos pidió:

![[Pasted image 20250922223410.png]]

![[Pasted image 20250922223427.png]]

Lab Terminado.

### Lab: Server-side template injection using documentation

En este caso, el objetivo es el mismo y nos dan unas credenciales con las cuales iniciamos sesión y podemos intentar ver qué encontramos:

![[Pasted image 20250922223724.png]]

Vemos que tenemos privilegios para modificar la información y vemos esa nomenclatura con las `{}` que nos da pistas sobre un SSTI.

Lo que podemos hacer es generar algún error en la carga de estos atributos para ver qué se está implementando:

![[Pasted image 20250922223905.png]]

Vemos que al generar un error que vimos en el laboratorio anterior, que es cuestión de modificar el atributo de tal forma que no pueda cargarlo, vamos a buscar sobre `FreeMarker`, el cual, mediante una pequeña investigación, parece ser que pertenece a Java, por lo que podemos volver a hacer uso de *PayloadsAllTheThings* y buscar algo que nos sirva:

![[Pasted image 20250922224104.png]]

Vemos que tenemos esas formas de inyección, la cual podemos intentar a ver qué logramos:

![[Pasted image 20250922224140.png]]

Vemos que sí se logró la inyección:

![[Pasted image 20250922224203.png]]

Vamos ahora a ver alguna forma de ingresar comandos:

![[Pasted image 20250922224251.png]]

Intentamos con el comando `whoami`. Veamos qué sucede:

![[Pasted image 20250922224319.png]]

Perfecto, en este punto es cuestión de volver a eliminar el archivo que nos solicita el laboratorio:

![[Pasted image 20250922224427.png]]

Lab Terminado.

### Lab: Server-side template injection in an unknown language with a documented exploit

El objetivo sigue siendo el mismo, solo que ahora no nos dice nada, por lo que tenemos que identificar el lenguaje y el *exploit* por nuestra cuenta.

En este punto, comprendemos el concepto de reconocer generando errores, así que podemos intentarlo y ver qué nos dice:

![[Pasted image 20250922224749.png]]

Vemos eso de `handlebars` y, mediante una búsqueda rápida, vemos que se trata de un *template* de JavaScript. Vamos a buscar en la biblia del *hacker* a ver qué encontramos (*PayloadsAllTheThings*):

![[Pasted image 20250922224939.png]]

Vemos las inyecciones básicas y podemos intentarlo:

![[Pasted image 20250922225038.png]]

Vemos que le gusta esto, así que podemos intentar alguna forma de ejecución de comandos para ver si funciona:

![[Pasted image 20250922225305.png]]

> [!tip] URL Encoding en SSTI
> Es crucial recordar que los *payloads* de SSTI a menudo necesitan ser codificados en URL (`URL-encoded`) cuando se envían a través de parámetros de URL o cuerpos de petición que no manejan caracteres especiales directamente. Herramientas como Burp Suite facilitan esta tarea.

Es un comando de *PayloadsAllTheThings*, pero que se sepa que está *URL-encoded* y por eso se ve así. En este punto, lo que vemos es que se ejecutó el comando. Para esto, vamos a intentar eliminar el archivo y ver si la máquina se completa:

![[Pasted image 20250922225434.png]]

Se está modificando un *URL-encode* en el mismo Burp Suite. Ahora, si vemos si lo logramos:

![[Pasted image 20250922225727.png]]

Lab Terminado.

### Lab: Server-side template injection with information disclosure via user-supplied objects

Tenemos que obtener en este caso una clave secreta almacenada en el *framework*.

Vamos primero a identificar la plantilla, por lo que vamos a ver qué obtenemos:

![[Pasted image 20250922230215.png]]
![[Pasted image 20250922230131.png]]

A diferencia de otros laboratorios, en este, para que ocurra el error, no ponemos nada y listo. En la parte de la plantilla, vemos que usa `django`. En este punto, podemos utilizar de nuevo la biblia del *hacker* para ver diferentes inyecciones que podemos usar:

![[Pasted image 20250922230346.png]]

Podemos ver muchas inyecciones. En este caso, vamos a comenzar con la básica para comprobar que se puede y luego intentamos cosas como ver la *APP secret key*.

La inyección básica falla, pero intentamos con otras hasta llegar a la de *Debug information Leak*. En ese punto, funciona y vemos lo siguiente:

![[Pasted image 20250922230641.png]]

Eso y más información, pero no encontraremos nada. En este punto, y viendo que no tenemos más información allí, procedemos a buscar en otras fuentes y encontramos un comando que nos deja ver algo:

![[Pasted image 20250922231009.png]]

Vemos que filtramos por la *secret key*. Vamos a ver si con esto completamos el laboratorio:

![[Pasted image 20250922231043.png]]

Lab Terminado.

### Lab: Server-side template injection in a sandboxed environment

En este laboratorio, nos encontramos en un entorno limitado que no debería permitir muchas cosas. El objetivo es leer un archivo con la contraseña del usuario `carlos`.

Vamos primero a identificar el *template*:

![[Pasted image 20250922231355.png]]

Logramos producir un error y vemos que nos encontramos frente a un `FreeMarker Template`, por lo que podemos intentar con diferentes inyecciones de la biblia del *hacker*:

![[Pasted image 20250922231734.png]]

> [!warning] Evasión de Sandboxes
> Los entornos *sandboxed* están diseñados para restringir la ejecución de código. La evasión de estos *sandboxes* en SSTI a menudo implica el uso de funciones o métodos permitidos para acceder a objetos del sistema o a la cadena de prototipos/clases, permitiendo la ejecución de comandos o la lectura de archivos.

Mediante prueba y error, logré obtener una que funciona y nos permite listar, aunque no en texto claro. Como vemos, nos da en decimal y tenemos que pasarlo a ASCII, por lo que vamos a usar algún convertidor *online* para esto:

![[Pasted image 20250922231924.png]]

Tenemos la cadena que nos interesa. Podemos ver qué sucede si la usamos:

![[Pasted image 20250922232000.png]]

Lab Terminado.

### Lab: Server-side template injection with a custom exploit

En este caso, nos hablan de un *exploit* *custom* y que tenemos que eliminar la clave `ssh` del usuario `carlos` en este caso.

Vemos todos estos cambios:

![[Pasted image 20250923092517.png]]

Nosotros ya realizamos una inyección donde jugamos con el valor de *Preferred Name*. Veamos si este punto sigue siendo inyectable:

![[Pasted image 20250923092745.png]]

Cerramos el primer campo `}}`. Vemos qué nos aparece:

![[Pasted image 20250923092825.png]]

Observamos que queda colgando el cerrado por defecto, así que vamos a aprovecharnos de esto para inyectar algo sencillo:

![[Pasted image 20250923092926.png]]

Vemos que obtenemos:

![[Pasted image 20250923092951.png]]

Bueno, tenemos el punto de la inyección. Claro, en este punto seguimos sin saber qué *template* se está empleando, por lo tanto, vamos a intentar causar algún error:

![[Pasted image 20250923093103.png]]

Mediante un `;` logramos provocar el error:

![[Pasted image 20250923093224.png]]

Mediante una búsqueda rápida, lo de `twig` es una plantilla de PHP, por lo tanto, ya tenemos el lenguaje, así que vamos a ver si logramos obtener algo:

![[Pasted image 20250923093435.png]]

Encontramos cositas. Vamos a ver alguna forma de ejecutar código.

Buscamos con todas las que encontramos y, pues, ninguna funciona.

En este punto, vemos otra funcionalidad, la cual nos permite cambiar el avatar de nuestro perfil:

![[Pasted image 20250923093955.png]]

Mirando esto, lo que podemos hacer es tratar de subir una imagen y ver cómo se está tramitando:

![[Pasted image 20250923094131.png]]

Cosas que podemos probar es corromper alguna cosa como el `name` cambiando por cualquier cosa:

![[Pasted image 20250923094239.png]]

Tenemos fuga de información, ya que vemos rutas como `/home/carlos/avatar_upload.php`, también un archivo el cual es el que está fallando. Por cosa obvia, la ruta también nos lista al usuario `carlos`.

Bueno, podemos intentar:

![[Pasted image 20250923094747.png]]

Vemos que cambiamos el contenido, el nombre y la extensión y funciona. Entonces, puede que nos interprete otro tipo de archivo:

![[Pasted image 20250923094944.png]]

Vemos que si abrimos la imagen, nos carga el contenido. Solo tal vez, y conociendo que la web interpreta PHP, podemos enviar código malicioso en PHP:

![[Pasted image 20250923095120.png]]

No funcionó. Vamos a seguir intentando cosas como quitar el *content type* a ver qué nos dice la web:

![[Pasted image 20250923095636.png]]

Generamos otro error, pero vemos que nos indica algo de `User->setAvatar()`. Esto es extraño, ya que no se está empleando esto en esta petición. Podemos revisar si en la otra esto puede ser válido:

![[Pasted image 20250923095834.png]]

![[Pasted image 20250923095900.png]]

Vemos un error distinto, que nos habla de que la función tiene pocos argumentos. Podemos intentar pasarle la ruta que nos dio de ejemplo de `tmp/fondo.jpg` tal y como tenemos en la otra:

![[Pasted image 20250923100947.png]]

Perfecto, enviamos y vemos si funciona:

![[Pasted image 20250923101019.png]]

No vemos errores, no se abre la imagen, sino que se descarga. Podemos intentar hacer una petición con `curl`:

![[Pasted image 20250923101141.png]]


Logramos filtrar información, ver diferentes archivos en el sistema y nosotros vimos algunas rutas de archivos PHP que se estaban empleando como `/home/carlos/User.php` y `/home/carlos/avatar_upload.php`. Vamos a ver uno por uno a ver cómo están programados:

![[Pasted image 20250923101423.png]]

![[Pasted image 20250923101448.png]]

Perfecto, ya vemos el código. Vamos a entender y a buscar formas de vulnerarlo.

![[Pasted image 20250923102009.png]]

Vemos esta función allí, la cual parece ser una que permite eliminar el avatar. Podemos igualar el valor del avatar al de `/.ssh/ssh.pub` a ver si luego, al invocar a esta función, se elimina:

![[Pasted image 20250923102859.png]]

![[Pasted image 20250923102908.png]]

Al parecer, ese es el contenido. Vemos si al invocar a la función que encontramos logramos eliminar ese archivo:

![[Pasted image 20250923103016.png]]

![[Pasted image 20250923103034.png]]

Lab Terminada.