------
El **Server-Side Template Injection** (**SSTI**) es una vulnerabilidad de seguridad en la que un atacante puede inyectar código malicioso en una **plantilla** de servidor.

Las plantillas de servidor son archivos que contienen código que se utiliza para generar **contenido dinámico** en una aplicación web. Los atacantes pueden aprovechar una vulnerabilidad de SSTI para inyectar código malicioso en una plantilla de servidor, lo que les permite ejecutar comandos en el servidor y obtener acceso no autorizado tanto a la aplicación web como a posibles datos sensibles.

Por ejemplo, imagina que una aplicación web utiliza plantillas de servidor para generar correos electrónicos personalizados. Un atacante podría aprovechar una vulnerabilidad de **SSTI** para inyectar código malicioso en la plantilla de correo electrónico, lo que permitiría al atacante ejecutar comandos en el servidor y obtener acceso no autorizado a los datos sensibles de la aplicación web.

En un caso práctico, los atacantes pueden detectar si una aplicación Flask está en uso, por ejemplo, utilizando herramientas como **WhatWeb**. Si un atacante detecta que una aplicación **Flask** está en uso, puede intentar explotar una vulnerabilidad de **SSTI**, ya que Flask utiliza el motor de plantillas **Jinja2**, que es vulnerable a este tipo de ataque.

Para los atacantes, detectar una aplicación Flask o Python puede ser un primer paso en el proceso de intentar explotar una vulnerabilidad de SSTI. Sin embargo, los atacantes también pueden intentar identificar vulnerabilidades de SSTI en otras aplicaciones web que utilicen diferentes frameworks de plantillas, como Django, Ruby on Rails, entre otros.

Para prevenir los ataques de SSTI, los desarrolladores de aplicaciones web deben validar y filtrar adecuadamente la entrada del usuario y utilizar herramientas y frameworks de plantillas seguros que implementen medidas de seguridad para prevenir la inyección de código malicioso.

## Notas de la practica

Vamos a desplegar la maquina de la siguiente manera:
```bash
docker run -p 8089:8089 -d filipkarc/ssti-flask-hacking-playground
```

ya con esto tenemos la maquina lista para practicar.

![[Pasted image 20250826081555.png]]

vemos que esa es la web donde vamos a ingresar algo aver que hace:

![[Pasted image 20250826081627.png]]

vemos que lo que escribamos lo representa en el titulo de la web pero también en la url lo que permite que cambiemos el dato desde allí.

Vamos a analizar la web con ayuda de [[Whatweb]] para ver que obtenemos:

```bash
whatweb "http://localhost:8089"
```

![[Pasted image 20250826081821.png]]
Vemos que nos da la versión de python, ahora siempre que veamos python o algún framework relacionado a este y que además nosotros podemos controlar algún tipo de output como en este caso la palabra `stark` podemos probar lo siguiente:

![[Pasted image 20250826082110.png]]

si se realiza la operación entonces ya estamos teniendo un problema:

![[Pasted image 20250826082140.png]]

como podemos observar si se ejecuta la operatoria lo que indica ya un **SSTI**.

Nosotros nos podemos ayudar de la siguiente web https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection donde nos dará para diferentes lenguajes una gran cantidad de y diferentes inyecciones para cada uno, en este caso usamos los de python para `jinja2`:

![[Pasted image 20250826082513.png]]

como podemos ver tenemos algunas inyecciones básicas por lo que podemos ir intentando.

Bueno si seguimos revisando nos vamos a encontrar con que podemos hacer multiples cosas como el leer de forma remota archivos por ejemplo:
```
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```

![[Pasted image 20250826082758.png]]

como vemos logramos cargar archivos de la maquina y ya esto es critico, no todos lo payloads que encontremos funcionaran por lo que siempre toca ir probando uno por uno esperando que uno funcione.

Esto puede llegar a ser critico ya que se puede llegar a ejecutar hasta comandos:

![[Pasted image 20250826083015.png]]

podemos ver que con otro payload logramos ejecutar comandos.




## PortSwigger

### Lab: Basic server-side template injection

En este laboratorio el objetivo es encontrar una inyección de comandos dentro de la web y que nos permita la eliminación del archivo `morale.txt` que se encuentra dentro del directorio home del usuario `carlos`.


Vamos a revisar un poco la web y ver como es que esta funciona un poco:
![[Pasted image 20250922135312.png]]

Vemos que mediante un parámetro en la url se esta insertando información dentro de la web, podemos aquí intentar inyectar comandos, tenemos que saber que los commandos van a variar dependiendo completamente de la plantilla, si es python o ruby o cualquier otro tipo.

En este caso es ruby vamos a intentar lo siguiente:
![[Pasted image 20250922135534.png]]

la inyección ya que no se ve bien fue `<%=7*7%>` además de que como vemos no nos imprime el contenido de la inyección, sino que imprime el resultado de la operatoria.

Una recomendación es siempre buscar en https://github.com/swisskyrepo/PayloadsAllTheThings, siempre vamos a encontrar opciones de inyecciones para diferentes lenguajes:
![[Pasted image 20250922135832.png]]
Vemos de ejemplo la inyección que hicimos, pero en este caso vamos a buscar algo válido para que permite ejecutar comandos:
![[Pasted image 20250922140014.png]]

tenemos varias opciones pero la que vemos en azul es la que voy a usar:
![[Pasted image 20250922140038.png]]

aunque no se aprecia del todo bien, vemos directorios típicos de la raíz como lo son `mnt, opt, proc` y mas, vamos a intentar entonces eliminar el archivo que nos piden, que se encuentra en el `home/carlos`:
![[Pasted image 20250922140236.png]]

Perfecto ya tenemos el archivo que queremos eliminar, es cuestión de hacer un `rm morale.txt`:
![[Pasted image 20250922140335.png]]

Lab Terminado.

### Lab: Basic server-side template injection (code context)

En este laboratorio tenemos que encontrar la forma de ejecutar comando en plantillas de tornado y el objetivo es eliminar el archivo `morale.txt`.

Bueno comencemos haciendo login:
![[Pasted image 20250922200959.png]]

Vemos que podemos cambiar el nombre preferido, que no es mas que en un post cualquiera se refleje lo que se le defina, vamos a interceptar esta petición a ver como se esta tramitando:
![[Pasted image 20250922201342.png]]

Vemos un `user.nikname` el cual podemos de alguna forma asociarlo de alguna forma a objetos en programación, nosotros no conocemos el leguaje empleado ni la plantilla, por lo cual lo que vamos a hacer es identificar lo que se esta usando, para esto la forma mas fácil de hacerlo es provocando errores de la siguiente manera:
![[Pasted image 20250922221642.png]]

Intercepto la petición y la envío modificada donde `user.nickname` lo cambio o modifico por algo o como en el caso de la imagen es `user.nicknametest`, vemos que ocurre:
![[Pasted image 20250922222004.png]]

Con esto podemos intentar buscar algo util en payload all the things a ver que encontramos:
![[Pasted image 20250922222122.png]]''

vamos a intentar lo que tenemos allí:
![[Pasted image 20250922222325.png]]

![[Pasted image 20250922222348.png]]

vemos que causamos un error, tenemos que interpretarlo como cuando jugamos en los XXS donde primero tratamos de cerrar las etiquetas anteriores, aquí también vamos a intentar cerrar la placeholder anterior por decirlo de alguna forma de la siguiente manera:
![[Pasted image 20250922222631.png]]

Podemos observar como primero cerramos lo anterior y luego abrimos alguna nueva petición, ahora no cerramos la nueva porque por detrás lo que se tenia es `{{user.nickname}}` entonces cuando cerramos el primero obtenemos `{{user.nickname}}}}` vemos que tenemos 4 de cierre ya que se arrastran los dos que teníamos, ahora si agregamos la inyección tenemos `{{user.nickname}}{{7*7}}` y con esto tenemos dos peticiones que al momento de enviarla obtenemos:
![[Pasted image 20250922222843.png]]

Perfecto, ya vemos el 49, de la misma pagina que revisamos las inyecciones vamos a ver alguna forma de ejecutar comandos:

![[Pasted image 20250922223231.png]]

![[Pasted image 20250922223246.png]]

Perfecto, como podemos observar tenemos el usuario que es carlos, en este punto solo resta eliminar el archivo que se nos pidió:

![[Pasted image 20250922223410.png]]

![[Pasted image 20250922223427.png]]

Lab Terminado.

### Lab: Server-side template injection using documentation

En este caso el objetivo es el mismo y nos dan unas credenciales con las cuales iniciamos sesión y podemos intentar ver que encontramos:

![[Pasted image 20250922223724.png]]

Vemos que tenemos privilegios para modificar la información y vemos esa nomenclatura con las `{}` que nos da pistas sobre un SSTI.

Lo que podemos es generar algún error en la carga de estos atributos para ver que se esta implementando:
![[Pasted image 20250922223905.png]]

vemos que al generar un error que vimos en la lab anterior que es cuestión de el atributo modificarlo de tal forma que no pueda cargarlo, vamos a buscar sobre `FreeMarker`, el cual mediante una pequeña investigación parece ser que pertenece a Java por lo que podemos volver a hacer uso de payload all the things y buscar algo que nos sirva:
![[Pasted image 20250922224104.png]]

Vemos que tenemos esas formas de inyección la cual podemos intentar a ver que logramos:
![[Pasted image 20250922224140.png]]

vemos que si se logro la inyección:
![[Pasted image 20250922224203.png]]

vamos ahora a ver alguna forma de ingresar comandos:
![[Pasted image 20250922224251.png]]

Intentamos con el comando `whoami` veamos que sucede:
![[Pasted image 20250922224319.png]]

Perfecto, en este punto es cuestión de volver a eliminar el archivo que nos solicita el lab:
![[Pasted image 20250922224427.png]]

Lab Terminado.


### Lab: Server-side template injection in an unknown language with a documented exploit

El objetivo sigue siendo el mismo solo que ahora lo que ahora no nos dice nada por lo que tenemos que identificar el lenguaje y el exploit por nuestra cuenta.

En este punto comprendemos el concepto de reconocer generando errores así que podemos intentarlo y ver que nos dice:
![[Pasted image 20250922224749.png]]

Vemos eso de `handlebars` y mediante una búsqueda rápida vemos que se tratar de un template de JavaScript, vamos a buscar en la biblia del hacker a ver que encontramos (payload all the things):
![[Pasted image 20250922224939.png]]

vemos las inyecciones básicas y podemos intentarlo:
![[Pasted image 20250922225038.png]]

vemos que le gusta esto, así que podemos intentar alguna forma de ejecución de comandos para ver si funciona:
![[Pasted image 20250922225305.png]]

es un comando de payload all the things pero que se sepa que esta url-encode y por eso que se ve así, en este punto lo que vemos es que se ejecuto el comando, para esto vamos a intentar eliminar el archivo y ver si la maquina se completa:
![[Pasted image 20250922225434.png]]

se esta modificando un url-encode en el mismo BurpSuite, ahora si vemos si lo logramos:
![[Pasted image 20250922225727.png]]

Lab Terminado.


### Lab: Server-side template injection with information disclosure via user-supplied objects

Tenemos que obtener en este caso una clave secreta almacenada en el framework.

Vamos primero a identificar la plantilla por lo que vamos a ver que obtenemos:

![[Pasted image 20250922230215.png]]
![[Pasted image 20250922230131.png]]

A diferencia de otros labs en este para acontecer el error no ponemos nada y listo en la parte de la plantilla vemos que usa  `django`, en este punto podemos utilizar de nuevo la biblia del hacker para ver diferentes inyecciones que podemos usar:
![[Pasted image 20250922230346.png]]

podemos ver muchas inyecciones, en este caso vamos a comenzar con la básica para comprobar que se puede y luego intentamos cosas como ver la APP secret key.

La inyección básica falla pero intentamos con otras hasta llegar a la de Debug information Leak, en ese punto funciona y vemos lo siguiente:
![[Pasted image 20250922230641.png]]

eso y mas información, pero no encontraremos nada, en este punto y viendo que no tenemos mas información allí procedemos a buscar en otras fuentes y encontramos un comando que nos deja ver algo:
![[Pasted image 20250922231009.png]]

Vemos que filtramos por la secret key, vamos a ver si con esto completamos el lab:
![[Pasted image 20250922231043.png]]

Lab Terminado.

### Lab: Server-side template injection in a sandboxed environment

En este laboratorio nos encontramos donde un entorno limitado que no debería permitir muchas cosas, el objetivo es leer un archivo con la contraseña del usuario carlos.

Vamos a primero a identificar el template:
![[Pasted image 20250922231355.png]]

Logramos producir un error y vemos que nos encontramos frente a un `FreeMarker Template` por lo que podemos intentar con diferentes inyecciones de la biblia del hacker:
![[Pasted image 20250922231734.png]]

Mediante prueba y error logre obtener una que funciona y nos permite listar aunque no en texto claro como vemos nos da en decimal y tenemos que pasarle ASCII por lo que vamos a usar algún convertidor online para esto:
![[Pasted image 20250922231924.png]]

Tenemos la cadena que nos interesa, podemos ver que sucede si la usamos:
![[Pasted image 20250922232000.png]]

Lab Terminado.

### Lab: Server-side template injection with a custom exploit

En este caso nos hablan de un exploit custom y que tenemos que eliminar la clave `ssh` del usuario carlos en este caso.

Vemos todos estos cambios:
![[Pasted image 20250923092517.png]]

Nosotros ya realizamos una inyección donde jugamos con el valor de Preferred Name veamos si este punto sigue siendo inyectable:
![[Pasted image 20250923092745.png]]

cerramos el primer campo `}}` vemos que nos aparece:
![[Pasted image 20250923092825.png]]

Observamos que queda colgando el cerrado por defecto así que vamos a aprovecharnos de esto para inyectar algo sencillo:
![[Pasted image 20250923092926.png]]

vemos que obtenemos:
![[Pasted image 20250923092951.png]]

bueno tenemos el punto de la inyección. Claro en este punto seguimos sin saber de que template se esta empleando por lo tanto vamos a intentar causar algún error:
![[Pasted image 20250923093103.png]]

Mediante un `;` logramos provocar el error:
![[Pasted image 20250923093224.png]]

mediante una búsqueda rápida lo de `twig` es una plantilla de php por lo tanto ya tenemos el lenguaje, así que vamos a ver si logramos obtener algo:
![[Pasted image 20250923093435.png]]

encontramos cositas, vamos a ver alguna forma de ejecutar código.

Buscamos con todas las que encontramos y pues no ninguna funciona.

En este punto vemos otra funcionalidad la cual nos permite cambiar el avatar de nuestro perfil:
![[Pasted image 20250923093955.png]]

Mirando esto lo que podemos hacer es tratar de subir una imagen y ver como se esta tramitando:
![[Pasted image 20250923094131.png]]

cosas que podemos probar es corromper alguna cosa como el `name` cambiando por cualquier cosa:
![[Pasted image 20250923094239.png]]

tenemos fuja de información ya que vemos rutas como`/home/carlos/avatar_upload.php` también un archivo el cual es el que esta fallando, por cosa obvia la ruta también nos lista al usuario carlos.

Bueno podemos intentar:
![[Pasted image 20250923094747.png]]

vemos que cambiamos el contenido, el nombre y la extensión y funciona entonces puede que nos interprete otro tipo de archivo:
![[Pasted image 20250923094944.png]]

vemos que si abrimos la imagen nos carga el contenido, solo talvez un y conociendo que la web interpreta php podemos enviar código malicioso en php:
![[Pasted image 20250923095120.png]]

No funciono, vamos a seguir intentando cosas como quitar el content type a ver que nos dice la web:
![[Pasted image 20250923095636.png]]

generamos otro error pero vemos que nos indica algo de `User->setAvatar()` esto es extraño ya que no se esta empleando esto en esta petición podemos revisar si en la otra esto puede ser valido:
![[Pasted image 20250923095834.png]]

![[Pasted image 20250923095900.png]]

vemos un error distinto, que nos habla de que la función tiene pocos argumentos, podemos intentar pasarle la ruta que nos dio de ejemplo de `tmp/fondo.jpg` tal y como tenemos en la otra:
![[Pasted image 20250923100947.png]]

perfecto enviamos  y vemos si funciona:
![[Pasted image 20250923101019.png]]

No vemos errores, no se haber la imagen sino que se descarga, podemos intentar hacer una petición con curl:

![[Pasted image 20250923101141.png]]


Logramos filtrar información, ver diferentes archivos en el sistema y nosotros vimos algunas rutas de archivos php que se estaban empleando como `/home/carlos/User.php` y `/home/carlos/avatar_upload.php`, vamos aver uno por uno a ver com están programados:
![[Pasted image 20250923101423.png]]

![[Pasted image 20250923101448.png]]

Perfecto, ya vemos el código vamos a entender y a buscar formas de vulnerarlo.

![[Pasted image 20250923102009.png]]

vemos esta función allí la cual parece ser una que permite eliminar el avatar, podemos igual el valor del avatar al de `/.ssh/ssh.pub` a ver si luego al invocar a esta función se elimina:
![[Pasted image 20250923102859.png]]

![[Pasted image 20250923102908.png]]

al parecer ese es el contenido, vemos si al invocar a la función que encontramos logramos eliminar ese archivo:
![[Pasted image 20250923103016.png]]

![[Pasted image 20250923103034.png]]

Lab Terminada.