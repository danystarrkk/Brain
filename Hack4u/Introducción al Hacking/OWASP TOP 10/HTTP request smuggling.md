-------------------------
Las vulnerabilidades de HTTP smuggling se presentan cuando el inicio y fin de una solicitud HTTP son interpretados de manera diferente por el front-end y el back-end de una aplicación.
De forma mas entendible lo que pasa es que el el front-end al recibir una solicitud contempla ciertos datos o atributos pero al enviarle al back-end este contempla cosas extras que aunque el front-end no los revisa pero si puede enviar el back-end si las interpretaría.

En este punto tenemos que entender dos conceptos:

**Content-Length**: Como tal esto hace referencia al tamaño de los datos en la solicitud, si enviamos un total de 3 datos vamos a ver un `content-Length: 3`:

![[Pasted image 20250918103256.png]]

**Transfer-Encoding**: Esto funciona de una forma semejante a `Content-jLenght` pero con una representación distinta, vamos atener 3 factores, primero el `Transfer-Encoding: chunked` por ejemplo luego debajo el tamaño en bytes en formato `hexadecimal` luego debajo los datos y finalizamos el chunk con un `0`, esto se vería de la siguiente manera:

![[Pasted image 20250918103844.png]]

Entonces lo que sucede es que desde HTTP/1.1 existe soporte generalizado para enviar múltiples solicitudes HTTP a través de un único socket TCP. El protocolo es extremadamente simple: las solicitudes HTTP se colocan una detrás de la otra, y el servidor analiza los encabezados (Transfer-Encoding o Content-Length) para determinar dónde termina cada una y dónde comienza la siguiente. Por lo tanto el vector de ataque es  que nosotros si logramos controlar o se permite el control del tamaño lo que sucederá es que podremos apilar en esa data extra consultas donde si el back-end las interpreta podremos filtrar información. El objetivo principal es burlar las diferentes validaciones implementados en el front-end que es donde mas se aplica y en ocasiones el back-end.

## Tipos de HTTP Request Smuggling

Los ataques clásicos de HTTP Request smuggling implican colocar tanto el encabezado `Content-Length` como el encabezado `Transfer-Encoding` en una única solicitud HTTP/1.1 y manipularlos para que los servidores front-end y back-end procesen la solicitud de manera diferente.

La forma exacta en que se hace esto depende del comportamiento de los dos servidores:

- **CL.TE**: El servidor de front-end usa el encabezado `Content-Length` y el servidor de back-end usa el encabezado `Transfer-Encoding`.
- **TE.CL**: El servidor de front-end usa el encabezado de `Transfer-Encoding` y el servidor de back-end usa el encabezado `Content-Length`.
- **TE.TE**: Los servidores front-end y back-end admiten el encabezado `Transfer-Encoding`, pero se puede inducir a uno de los servidores a no procesarlo, ofuscando el encabezado de alguna manera.


## Laboratorios PortSwigger

### Lab: HTTP request smuggling, confirming a CL.TE vulnerability via differential responses

Lo que queremos conseguir es que al momento de acceder a la raíz veamos un `404`.

Primero interceptamos la solicitud:
![[Pasted image 20250918105524.png]]

Vamos a pasarla al método POST:
![[Pasted image 20250918105624.png]]

Perfecto en este caso vemos que ya esta cargando en `HTTP/1.1`.

Vamos a borrar todo lo que en este caso no nos importa y vamos a dejar lo que necesitamos, nos queda de la siguiente manera:
![[Pasted image 20250918105756.png]]

Lo que vamos a hacer es una pequeña prueba donde vamos a enviar datos:
![[Pasted image 20250918105828.png]]

Yo solo realize cambios como el de `test=test` el valor de `Content-Length` no lo modifique ya que esto se adapta al tamaño de la data.

En este caso no nos interesa que se actualize esto de forma automática por lo que vamos a quitar en el BurpSuite, la cosa es dejarla desmarcada para que no lo realize:
![[Pasted image 20250918110104.png]]

Lo que vamos a hacer en este punto es primero vamos a activar la opción que nos deja ver los saltos de línea:
![[Pasted image 20250918110420.png]]

Ahora algo importante a tener en cuenta es que cuando usamos el `Transfer-Encoding` al momento de cerrar el chunk con el `0` tenemos que dar dos veces al enter, ya que si no, no estamos representando las cosas de forma correcta.

Ahora en el laboratorio el front-end esta revisando el  `Content-Length` pero el back-end admite o lee el `Transfer-Encoding`, vamos a ver que pasa si agregamos el `Transfer-Encoding` y enviamos la información en formato chunked, com sabemos por ahora estamos usando `HTTP/2` pero vamos a cambiarlo desde el inspector de la siguiente manera para que funcione:

![[Pasted image 20250918112828.png]]

Perfecto vemos un error y es que esto se debe a que el front-end lo comprende pero llegar al back-end y ver el `Transfer-Encode` pero no ver la estructura correcta se produce un error interno, ahora creemos una estructúra correcta y vemos que sucede:

![[Pasted image 20250918113036.png]]

Excelente ahora ya funciona ya que es una estructura valida.

En este punto lo que vamos a intentar es provocar un error de sincronización esto modificando el el `Content-Length` para que no tome el cerrar de la estructura `chunck` y se de el error de sincronización:
![[Pasted image 20250918123217.png]]

En este punto lo que queremos es tratar de efectuar un error del tipo `404`.

Primero vamos a entender que nosotros cuando enviamos solicitudes estas entran en una cola dende se van procesando, ahora nosotros dentro de una solicitud podemos enviar otra solicitud, recordemos que en la imagen anterior estamos generando un error por no enviar la estructura completa del `chunked`, pero cuando lo hacemos en el back-end esto se procesa de tal manera que la solicitud termina donde esta el `0` que cierra al `chunked`. Ahora que pasa si nosotros luego de cerrar la estructura le pasamos otra petición de la siguiente manera:
![[Pasted image 20250918125221.png]]

Como podemos observar estamos agregando otra solicitud luego, esto se tomara en realidad como otra petición. Es muy importante tener en cuenta que para que una estructura de una petición sea valida por definición lo que se tiene que emplear siempre es una cabecera, en ocasiones podemos inventarnos una y poner como la la imagen `Test: A` lo que sucede al enviar esta petición es el concepto antes de la imagen se va a enviar una solicitud normal y el back-end al ver el fin del chunked va a cerrar la una pero en seguida a generar la otra, esta es una petición que en realidad no se suele ver ya que se desincroniza y queda colgada en la cola, esta le cargara a la persona que realize otra solicitud a la misma web puede que se entrelacen solicitudes y se arrastre esta a cualquier lado. Por lo tanto si la enviamos vamos a ver que nos responde con normalidad pero si recargamos en la web desde el navegador vamos a ver que se enlaza esta solicitud que se quedo colgada en espera y nos llega al navegador:
![[Pasted image 20250918125933.png]]

![[Pasted image 20250918125944.png]]

Como vemos esto es increíble ya logramos el objetivo. Podemos intentar cambiar para jugar un poco de error a ver algún post a ver si al recargar en el navegador nos sincroniza y nos manda a la web dentro de un post:
![[Pasted image 20250918130448.png]]

Perfecto dejamos una solicitud colgada para ir a un post, recarguemos a ver si en el Firefox nos envía la respuesta:

![[Pasted image 20250918130507.png]]

la url sale como la misma pero se cargo lo del post entonces es muy interesante el como se puede hacer estas cosas.
![[Pasted image 20250918130534.png]]
Lab Terminado.


### Lab: HTTP request smuggling, confirming a TE.CL vulnerability via differential responses

Bueno en el laboratorio anterior para la solicitud estándar necesitamos de solo definir el Método, la dirección y una cabecera, en esta maquina vamos a ver que con estos datos ya no son suficientes, lo que pasa es que se están revisando por un mayor numero de contenidos, lo que provoca que se tome a la segunda solicitud que estamos filtrando como parte de la primera mismo.

Entonces lo que vamos a hacer en este caso es inflar la petición para que se tome como una segunda valida.


Primero vamos a intentar causar una desincronización, tomemos en cuenta que este laboratorio esta invertido al anterior, en donde ahora el front-end revisa por `Transfer-Encoding` y el back-end mira por `Content-Length`, primero comenzamos capturando la petición:
![[Pasted image 20250918162350.png]]
Perfecto ahora tenemos el como se tramita normalmente la solicitud, en este punto vamos a hacer el procedimiento de cambiar el méotodo, fijar la version de HTTP:
![[Pasted image 20250918162445.png]]

Listo esto carga normal, ahora vamos a quitar lo que nos sirve, como el front-end de primeras no necesita el `Content-Length` lo vamos a quitar también y a definir el `Transfer-Encoding`:
![[Pasted image 20250918162701.png]]

vemos que esto si es valido a si que perfecto, podemos crear errores como no terminar el chuck para probar si se esta comprobando el T.E:
![[Pasted image 20250918162746.png]]

Excelente en este punto podemos ver que si se revisa la información y nos arroja un error al enviar el chunk de forma incorrecta, ahora vamos a ver como acontecer una desincronización para el `Content-Length` el cual corre en el back-end:
![[Pasted image 20250918163134.png]]

El como se causa es sencillo le estamos diciendo al back-end que tenemos 20 bytes pero solo enviamos 13, esto provoca que se quede esperando por los bytes restantes y se genera la desincronización, abuzando de esto lo que se va a hacer es:
![[Pasted image 20250918163727.png]]

A ver lo que estamos haciendo dentro de esta nueva solicitud es traspasar primero el front-end con una estructura correcta pero como vemos tenemos dentro de nuestro chunk tenemos la solicitud que que queremos enviar para que quede colgada, entonces aquí viene el truco con el `Content-Length` donde al back-end le decimos que solo lea los primeros 4bytes, hablamos del `1d\r\n` y allí es donde a lo demás lo tomaría como una segunda petición:
![[Pasted image 20250918164052.png]]

Ahora vemos que no se efectúa problema al momento de tramitar esta petición y es debido a que la puede estar haciendo pasar como arte de la primera debido a que no tiene el tamaño o los datos suficientes para considerarla como otra, lo que podemos hacer es lo siguiente:
![[Pasted image 20250918164543.png]]

Perfecto ya tenemos una estructura mas correcta la cual si entro correctamente deberíamos ver la petición con el error 404:
![[Pasted image 20250918164649.png]]

pues no no logramos ver, aquí es donde entra lo de inflar el `Content-Length` lo que hacemos es ponerle un digito mas al `content-length`, esto lo que hace es dejarlo en formato keep a life (en espera) de un dato adicional y de donde lo saca es de una nueva petición entrante, allí se sincroniza a ella, entonces la petición nos quedaría de la siguiente manera:
![[Pasted image 20250918165109.png]]

vemos que al recargar la web principal se efectúa el error:
![[Pasted image 20250918165238.png]]

Entonces lo que se hace en estos casos es inflar el `Content-length` con el objetivo de dejar a esa segunda solicitud como en escucha una una entrante la cual complete los bytes faltantes y se sincronice a la misma viendo el resultado.
![[Pasted image 20250918165350.png]]

Lab Terminado.

###  Lab: Exploiting HTTP request smuggling to bypass front-end security controls, CL.TE vulnerability

En este laboratorio tenemos que ingresar a `/admin` desde cual vamos a tener que eliminar al usuario carlos, el problema es que el front-end bloquea el acceso a este recurso por lo tanto tenemos que hacer un HTTP request smuggling para intentar acceder y borrar el usuario.

Empecemos capturando la petición:
![[Pasted image 20250918172624.png]]

Bueno vamos a comenzar, lo que sabemos es que el front-end interpreta lo que es  C.L y el back-end lo que es T.E.

En este punto vamos a tratar de enviar con un `Transfer-Encoding` valido para ver si esto se interpreta sin problemas:
![[Pasted image 20250918173230.png]]

Perfecto vemos que ya tenemos la estructura básica, en este punto lo que vamos a hacer es intentar primero el chunk dejando solo cerrado y agregar una estructura básica de petición dentro de esta misma de la siguiente manera:
![[Pasted image 20250918173519.png]]
Perfecto nos deja pasar y si revisamos una segunda solicitud veremos lo siguiente:
![[Pasted image 20250918173553.png]]

Intentemos con el panel admin a ver que nos responde:
![[Pasted image 20250918173659.png]]
![[Pasted image 20250918173711.png]]

Vemos que nos responde ya que esta segunda solicitud no pasa por el front-end como tal y solo se interpreta en el back-end vemos esta respuesta que es la de admin pero nos dice que solo esta disponible para usuario locales.

Lo que podemos intentar es definir la cabecera como `Host: localhost`:
![[Pasted image 20250918174209.png]]

Vemos que efectivamente la solicitud se envía pero si revisamos vamos a ver lo siguiente:
![[Pasted image 20250918174257.png]]
lo que nos dice es que estamos duplicando una cabecera, esto quiere decir que no nos esta interpretando la segunda como una petición sino que lo hace como parte de la misma lo que da el origen al error de la cabecera duplicada.

Para evitar esto tenemos que darle una mejor estructura para que se interprete como petición de la siguiente manera:
![[Pasted image 20250918174520.png]]

ya estamos enviando pero no sucede nada y para esto vamos a intentar inflar el el `content-length` de la siguiente manera:
![[Pasted image 20250918174623.png]]

Vemos que efectivamente ya se envía y si recargamos veremos lo siguiente:
![[Pasted image 20250918174648.png]]

Ya logramos cargar la pagina de admin, esto solo cambiando el `Host: localhost` para que piense que la solicitud va de forma local y mediante una inflación del `content-length` logramos enlazar a una nueva solicitud, con esto ya podemos eliminar al usuario carblos con la siguiente solicitud:

![[Pasted image 20250918175025.png]]

![[Pasted image 20250918175130.png]]

vemos eso pero si regresamos a la raíz vamos a observar que:
![[Pasted image 20250918175215.png]]
Lab Terminado.


### Lab: Exploiting HTTP request smuggling to bypass front-end security controls, TE.CL vulnerability

En este laboratorio vamos a realizar lo mismo con la diferencia de que en el front-end se esta empleando el T.E y en el back-end se esta empleando el C.L.

Vamos comenzar capturando la petición y configurándola de forma correcta:
![[Pasted image 20250918180353.png]]
Perfecto tenemos ya procesado correctamente esto ahora vamos a dentro del chunk ingresar la petición primero vamos a causar un error con el objetivo de probar si tenemos correctamente operativa la inyección:

![[Pasted image 20250918181428.png]]

Perfecto ya tenemos correctamente configurada la solicitud y esta nos retorna al panel de admin:
![[Pasted image 20250918181511.png]]

En este punto vamos a crear para que elimine el usuario carlos:
![[Pasted image 20250918181611.png]]

Perfecto vemos que nos retorna la siguiente petición, recordemos siempre inflar la petición para la segunda solicitud recordemos que para la segunda solicitud es desde la ultima línea no solo el `test=test` sino desde la ultima linea :
![[Pasted image 20250918181633.png]]

aunque aparente que no este mensaje nos esta configurando que si logro eliminar el mensaje, esto lo sabemos ya que solo es una maquina de practica:
![[Pasted image 20250918181736.png]]

Lab Terminado.


### Lab: Exploiting HTTP request smuggling to reveal front-end request rewriting


En este caso lo que tenemos un panel de administración en `admin` pero solo es disponible si usamos una cabecear que no conocemos pero que tiene que tener como valor `127.0.0.1`.

Vamos a comenzar capturando la petición:
![[Pasted image 20250918201237.png]]

Bueno ya hemos echo algunas pruebas donde vemos que el front-end es CL y el back-end es TE es por eso de que al definir un CL que no tenga todo en este caso eliminamos el 0 del cierre del chunk se acontece el error.

Bueno ahora lo que podemos hacer es lo siguiente:
![[Pasted image 20250918201754.png]]
Realizamos una inyección común pero recordemos que como estamos inflando el `content-Length` el contenido de la siguiente solicitud se incrusta en el body de la petición y se usa para sincronizarla pero se incrusta en la petición, en este caso se pone después de test, vemos que nos da la siguiente solicitud:
![[Pasted image 20250918201944.png]]

Nos esta incrustando la información como podemos ver, entonces podemos aprovechar esto para intentar ver todo la información posible vamos a inflar mas el CL para ver toda la información posible:
![[Pasted image 20250918202054.png]]

Vemos al parecer una cabecera mas que es `x-YGwWNN-Ip`, en este caso podemos usar esta cabecera para darle el valor de `127.0.0.1` y esto en el mismo HTTP smuggling enviamos asía `/admin` vemos si logramos pasar:
![[Pasted image 20250918202420.png]]

Vemos si se logro en la web:
![[Pasted image 20250918202437.png]]
Ahora vamos a eliminar al usuario carlos:
![[Pasted image 20250918202758.png]]

Por temas de que salía cabecera duplicada se uso el método GET y listo:
![[Pasted image 20250918202829.png]]

Lab Terminada.


### Lab: Exploiting HTTP request smuggling to capture other users' requests


La idea es obtener la cookie de session de un usuario, cada que nosotros enviamos una petición hace que la victima también realize una petición.

La idea similar a la anterior pero en esta ocasión es mediante un comentario:

![[Pasted image 20250919115300.png]]

Perfecto sabiendo que el front-end interpreta `Content-length` y el back-end contempla el `Transfer-Encoding` vamos a jugar creando un chunk para que lo interprete el back-end.

Vamos a capturar el momento de enviar un comentario primero:
![[Pasted image 20250919115823.png]]

Ahora esto lo vamos a copiar para tramitarlo como una segunda petición:
![[Pasted image 20250919115855.png]]

Lo que vamos a hacer es eliminar cabeceras que por el momento no son útiles, quedando la siguiente petición:
![[Pasted image 20250919120026.png]]

Vamos a enviar a ver que sucede:
![[Pasted image 20250919120512.png]]

Perfecto enviamos la solicitud y es valida, veamos que nos retorna la solicitud enlazada:
![[Pasted image 20250919120547.png]]

Vemos que nos dice que tenemos un CSRF token no valido, esto puede ser por la cookie, podemos intentar poner la cookie a ver que sucede:
![[Pasted image 20250919121447.png]]

algo que nos acabamos de fijar es que cambia el valor del `CSRF token` por lo que también lo cambiamos por uno no usado y capturado, veamos si esto le gusta:
![[Pasted image 20250919121436.png]]

Vemos que nos responde como si tramitáramos el comentario, vamos a ver que obtenemos:
![[Pasted image 20250919121545.png]]

Perfecto, ahora el concepto es el mismo de la anterior maquina donde inflando el `Content-Length` vamos a intentar que se asigne como cuerpo de solicitud la información de la solicitud que esta haciendo el otro usuario por detrás, de esta forma vamos a intentar robarle la cookie, para esto inflamos mucho el `Content-Length`, además tomemos en cuenta que esto se esta insertando en el parámetro de website, por lo que luego lo que tenemos que revisar es el link, y esto nos queda de la siguiente manera:
![[Pasted image 20250919122516.png]]

![[Pasted image 20250919122500.png]]

Vemos que tenemos una nueva publicación por lo tanto veamos que contiene:
![[Pasted image 20250919124423.png]]

vemos que se captura pero por no vemos la cookie vamos a aumentar mas la inflación para ver si se logra ver el cookie:
![[Pasted image 20250919125557.png]]

perfecto, en este caso para verlo en el post lo que hacemos es cambiar el parámetro del comentario al final y lo inyectamos en el comentario, pero bueno ya tenemos lo que necesitamos, con esto vamos completar el laboratorio:

![[Pasted image 20250919125851.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to deliver reflected XSS

En este laboratorio nos vamos a encontrar con un XSS en la cabecera del `User-Agent`, tenemos que causar un `alert(1)`.

Vamos a comenzar identificando donde esta el XSS:
![[Pasted image 20250919150120.png]]

por alguna razón se declara el `User-Agent` como un input, lo que vamos a hacer en este punto es intentar modificar para cerrar la petición e inyectar código:
![[Pasted image 20250919150312.png]]

como podemos ver modificamos el `User-Agent` y esto se termina de reflejar en la respuesta, en este punto lo que vamos a hacer es intentar inyectar una alerta a ver si se ejecuta:
![[Pasted image 20250919150534.png]]

![[Pasted image 20250919150511.png]]

Perfecto en este punto lo que tenemos hacer que hacer es un HTTP smuggling para intentar colarle una petición a otro usuario con esto, vamos a hacer lo siguiente en una petición de la pagina home:
![[Pasted image 20250919150841.png]]

Lo que hago es comprobar como se esta tratando la petición y al parecer es para el front-end CL y para el back-end TE, por lo tanto vamos a intentar pasarle la nueva solicitud maliciosa a ver que sucede primero:
![[Pasted image 20250919151537.png]]

Perfecto nos quedaría de la siguiente manera ahora es cuestión de enviar y esperar a ver si se completa el lab:
![[Pasted image 20250919151655.png]]
Lab Terminada:

### Lab: Response queue poisoning via H2.TE request smuggling

Lo que sucede es que cuando nosotros enviamos alguna petición el back-end hace un downgrade de HTTP/2 a HTTP/1.1.

Dos conceptos rápidos el uso de HTTP2 ya no es necesario el especificar `Content-Length` y `Tranfer-Encoding` ya que se calculan de forma automática, ahora tenemos que comprender otra cosa y es que el objetivo es secuestrar una petición que queda en el aire me refiero a que nosotros vamos a crear como siempre una especie de petición doble donde veremos como la una nos responde y la otra corrompe la cola haciendo que esta se sincronice con una solicitud de cualquier otras pc, pero lo que se tiene que comprender es que cuando esto pasa la respuesta que debería de a ver visto por ejemplo otro usuario al iniciar session queda perdida, nosotros lo que podemos hacer es recargar multiples veces la web para recuperarla de nuestro lado, esto es lo que tenemos que lograr en este lab.

Vamos a comenzar capturando una petición, modificándola y haciendo una petición no de error sino una correcta que se quede colgada:
![[Pasted image 20250919163125.png]]

al aquí no necesitamos de inflar ni nada solo de enviar multiples veces las peticiones con el objetivo de que el usuario recoja nuestra petición y nosotros mediante el envío de múltiples peticiones logremos recuperar la de el:
![[Pasted image 20250919163301.png]]

Excelente logramos recuperar la petición de el y tenemos la cookie de session la cual podemos usar para entrar y eliminar al usuario carlos:
![[Pasted image 20250919170952.png]]

Listo Lab Terminado.

### Lab: H2.CL request smuggling

De igual forma de aplica un downgrade desde el front hacia el back donde el back-end interpreta content Lenght.

El objetivo es que a la victima le salga una alerta con la cookie.

Primero capturemos una petición usual:
![[Pasted image 20250919171516.png]]

Tenemos esto listo y vemos que funciona, vamos a pasarle algo con mas bytes:
![[Pasted image 20250919171612.png]]

Perfecto podemos comenzar a intentar inyectar una petición para ver primero si podemos tramitar algún error:
![[Pasted image 20250919171857.png]]

Perfecto con esto ya vemos que si tenemos la vulnerabilidad, ahora el punto seria intentar ver si nosotros de alguna forma logramos reenviar a una web externa para desde un exploit externo cargar la alerta:
![[Pasted image 20250919173032.png]]

Vemos que cada que cargamos la web principal se realiza una carga de archivos js como lo vemos alli, lo que quiero intentar es ver como se altera la web cuando yo cambio datos, por ejemplo:
![[Pasted image 20250919173328.png]]

Vemos que cuando realizamos esta búsqueda tenemos un estado de `302` para redirecciones y nos manda en la respuesta a /js/ , vemos que pasa si le quietamos el /js:
![[Pasted image 20250919173426.png]]

ya que vemos que esto aplica un redirect es intentar incrustar esto en nuestra solicitud maliciosa y ver si podemos alterar la cabecera HOST para que se inserte en un follow redirec:
![[Pasted image 20250919173837.png]]

vemos que funciona aunque debemos quitar el `http` ya que eso si lo incrusta, podemos usar el exploit del lab para crear un código malicioso:
![[Pasted image 20250919174015.png]]

Perfecto si llama la solicitud ahora lo que hacemos es introducir el código malicioso, guardarlo y esperar:
![[Pasted image 20250919174125.png]]

se envío, veamos si se completa el lab:
![[Pasted image 20250919174655.png]]

Lab Terminado.


### Lab: HTTP/2 request smuggling via CRLF injection

Primero tenemos que comprender que el `CRLF` es el `CR -> \r` y el `LF -> \n`.

Bueno comenzamos viendo la web:
![[Pasted image 20250919185025.png]]

Vemos un campo de búsqueda el cual vemos como funciona:
![[Pasted image 20250919185052.png]]

Vemos que tenemos un historial de búsquedas, si esto sucede puede que tengamos una cookie de session:
![[Pasted image 20250919185318.png]]

Entonces lo que podemos pensar es mediante el HTTP smuggling dejar colgada una solicitud la cual realize una búsqueda donde al usuario cargar la pagina de inicio esta se acople a la búsqueda e inflando el `Content-Length` lo suficiente llegar a obtener una cookie de sessión.

Entonces o que podemos hacer es capturar la petición de una búsqueda:
![[Pasted image 20250919185733.png]]

Bueno en este punto no sabemos que es lo que el servidor soporta, si esto es TE o CL, por lo que vamos a probar primero con TE:
![[Pasted image 20250919185946.png]]

Enviamos el `Transfer-Encoding` y el chunk completo con el objetivo de ver si generamos errores y pues no se general y luego hicimos lo de la imagen donde estamos enviando luego del chunk una solicitud con el objetivo de forzar un error, pero no logramos nada, podemos intentar enviar un `Content-Length` también que este un poco inflado para intentar forzar el sincronizado:
![[Pasted image 20250919190236.png]]

Seguimos sin genera el error.

Bueno en ocasiones el Front-end  esta configurado para que no puedas usar ciertas cabeceras, en este caso puede estar bloqueando la cabecera del TE con el objetivo de que no se le pueda colar una solicitud alternativa.
Pero ahora si no esta correctamente configurado esto de las cabeceras nosotros podemos hacer lo siguiente:
![[Pasted image 20250919190536.png]]

Nosotros en el apartado de Request headers podemos como observamos seleccionado agregar una nueva cabecera, aquí es donde vamos a jugar un poco y es que vamos a poner tal y como tenemos pero en su valor vamos a hacer lo siguiente:
![[Pasted image 20250919190745.png]]

Lo que estamos asiendo es agregar una nueva cabecera pero a esta le estamos aplicando un salto de línea y luego le agregamos lo que es el TE, esto provoca que se inyecten cabeceras, vamos a darle a add:
![[Pasted image 20250919190941.png]]

Vemos que nos corta la solicitud y nos dice que nos pueden ser representadas y mas, no le hagamos mucho caso vamos a enviar para ver si de esta forma logramos inyectar la cabecera y efectuar el error:
![[Pasted image 20250919191031.png]]

Nos lo esta haciendo correcto y estamos burlando si nos pone restricciones en la cabecera.

En este punto ya vamos a modificar de tal forma que contemple la cookie ya que sin esta no nos asocia el historial, luego modificamos el método ya que búsqueda se tramita por post en la raíz y agregamos el parámetro `search` con un valor que decidamos y procedemos a inflar lo suficiente el `content-length` para obtener los datos:
![[Pasted image 20250919191903.png]]

Bueno en este punto tenemos lista la petición lo que vamos a ocasionar es que al momento de enviar se quedara la segunda petición sincronizada en espera de los bytes faltantes, cuando otro usuario recargue la web principal lo que va a pasar es que se completaran los bytes faltantes con los datos de la petición del usuario y se enviara, esto provoca que como esta solicitud esta asociada a nuestra cookie se nos quedara en el historial, vemos si esto sucede:
![[Pasted image 20250919191951.png]]

Perfecto podemos ver como ya tenemos la cookie de session, vamos a iniciar sesión:
![[Pasted image 20250919192125.png]]

Lab Terminado.


### Lab: HTTP/2 request splitting via CRLF injection

En esta maquina ya entra en juego que en la propia cabecera al darle al enter `CRLF` se inyecte una nueva petición a diferencia de la anterior que mediante una inyección de cabecera dentro de otra logramos que no se detecte la cabecera que se estaba bloqueando.

La idea es que mediante un envenenado de respuestas con el concepto de que nuestra HTTP smuggling queda en cola y es que al usuario intentar iniciar sesión primero le retorna esa respuesta en cola y la respuesta al inicio de sesión del usuario queda ahora en cola lo que nos permite a nosotros luego recuperarla.

En este punto lo que vamos a hacer es capturar primero la petición del home:
![[Pasted image 20250919194044.png]]

Vemos que de por si logramos enviar la solicitud pero no sabemos como se esta tramitando esto, así que vamos a intentar primero probar con TE de la siguiente manera:
![[Pasted image 20250919194228.png]]

tratamos de generar un error en el TE cerrando el chunk de forma incorrecta con `X` y no con un `0` como es debido pero no logramos  el erro.

En este punto puede estar pasando es lo que sucedió en la anterior maquina donde vamos a intentar hacer bypass de la cabecera TE si es bloqueada igual que en la maquina anterior:
![[Pasted image 20250919194509.png]]
![[Pasted image 20250919194539.png]]

vemos que no, así que vamos a regresar y vamos a intentar lo siguiente:
![[Pasted image 20250919194648.png]]
Como vemos pasamos el método a GET y nos dice que por esto método no puede contener cuerpo por lo que si quitamos el body pero dejamos la cabecera del TE:
![[Pasted image 20250919194746.png]]

Vemos que si es valido, lo que vamos a intentar es colar una nueva cabecera de la siguiente manera:
![[Pasted image 20250919195116.png]]

Vemos que no le vamos a enviar contenido en el body pero mediante una cabecera y con el concepto similar al de la anterior maquina vamos a intentar colarle una solicitud como vemos en la image:
![[Pasted image 20250919195240.png]]

Enviemos y vemos que sucede:
![[Pasted image 20250919195309.png]]

Vemos que de esta forma logramos acontecer la segunda solicitud y logramos el error deseado

En este punto si nos sirve el error ya que lo único que deseamos es que la victima vea ese error y nosotros podamos ver la cookie de sesión así que vamos a ver si lo logramos:
![[Pasted image 20250919200608.png]]

Perfecto ingresemos y eliminemos a carlos para completar el lab:
![[Pasted image 20250919200738.png]]

Lab Terminado.


### Lab: CL.0 request smuggling

Lo que tenemos en este lab es que en el front-end se esta empleando lo que es el `Content-Length` y el back-end acepta todo.

Para este caso lo en donde vamos a hacer la inyección es en un recurso debido a que muchas veces den los recursos al cargarse el servidor no realiza en el back-end ninguna comprobación, en este caso vamos a capturar desde el historial el como se realizo la petición de una imagen y la vamos a enviar al repeater:
![[Pasted image 20250919205018.png]]

Vamos como ya estamos acostumbrados a darle el tratamiento habitual para lograr tener datos mínimos además de inyectar una nueva petición:
![[Pasted image 20250919205221.png]]

Perfecto como ya es de costumbre logramos realizar el error normal como siempre, lo que vamos a hacer en este punto es lo siguiente que vamos a hacer es realizar la consulta al `/admin` para cargar la información:
![[Pasted image 20250919205410.png]]

Perfecto si se recarga la web principal veremos lo siguiente:
![[Pasted image 20250919205434.png]]

Ya logramos ver la pagina de admin, en este punto vamos a sacar la solicitud con la cual vamos a eliminar el usuario `carlos` para terminar con el lab:
![[Pasted image 20250919205544.png]]

vemos la web principal a ver que logramos:
![[Pasted image 20250919205626.png]]
Lab Terminado.


### Lab: HTTP request smuggling, basic CL.TE vulnerability

En este laboratorio vamos a hacer que el back-end vea que estamos intentando emplear una petición GPOST.

Vamos a comenzar interceptando la solicitud y haciendo el típico tratamiento de la petición:

![[Pasted image 20250919213518.png]]

Perfecto, ahora vamos a modificarla para TE y vamos a hacer pruebas para ver si podemos generar algún error:
![[Pasted image 20250919213715.png]]

Vemos que si Podemos controlar los erroes, en este punto el back-end lee el TE pero vamos a hacer una petición solo con una G:
![[Pasted image 20250919213845.png]]

Lo que hacemos es control parcialmente la solicitud, osea en la primera solicitud el back-end lee el final de chunk y termina pero cuando llega una segunda petición por post lo que sucede es que se concatena con la G que estamos mandando y provoca el método GPOST, vamos a verlo:
![[Pasted image 20250919214056.png]]

Listo producimos el método GPOST.
![[Pasted image 20250919214141.png]]

 Lab Terminado.


### Lab: HTTP request smuggling, basic TE.CL vulnerability

Bueno en este caso el objetivo es el mismo pero en este caso se encuentra invertido, ahora el front-end interpreta TE y el back-end CL.

Vamos a comenzar tomando la captura y realizando el tratamiento básico:
![[Pasted image 20250919220145.png]]

En este caso directamente ya esta solucionando, no es algo que no extraño, le pasamos para el front-end la solicitud por TE y luego generamos una nueva dentro de esta que en el back-end al contemplar CL se interpretara por lo que provoca la segunda petición:
![[Pasted image 20250919220314.png]]

Lab Terminada.

### Lab: HTTP request smuggling, obfuscating the TE header

El objetivo es de por si el mismo que las anteriores maquinas donde tenemos que tramitar una cabecera en como `GPOST` ahora el problema es que tanto el front-end como el back-end están empleando diferentes formas para repeler el uso de cabeceras repetidas.

Vamos a tomar la captura y a realizar el tratamiento básico:
![[Pasted image 20250919220852.png]]

Bueno realizamos una prueba de una vez para ver que interpreta el front-end y si cerramos el chunk de forma invalida vamos a ver lo siguiente:
![[Pasted image 20250919220949.png]]

Vemos que nos da un error de petición lo que quiere decir que el front-end interpreta TE.

Ahora vamos a intentar comprobar si el back-end comprende CL y para esto vamos a causar una desincronización, esto inflando el el CL:
![[Pasted image 20250919221231.png]]

Vemos que no pasa, entonces el back-end también es TE, entonces en este caso vamos a intentar lo siguiente:
![[Pasted image 20250919221610.png]]

Bueno logramos ahora si la desincronización, lo que sucede es lo siguiente:
- Primero entra al front-end y por configuración digamos esto esta configurado para que cuando se tenga una cabecera invalida pero esta este duplicada coja la valida, entonces el front-end pasa al back-end.
- Ahora cuando llega al back-end este lo interpreta diferente y es que si ve la cabecera duplicada y mas que todo es invalida lo que hace en automático es leer por Content-Length.

El descubrir este tipo de cosas es un entorno real es algo complicado pero para eso estamos para pensar fuera de la caja y mediante prueba y error lograr encontrar una forma de encontrar una vuln.

Bueno ahora lo que vamos a hacer es pasarle una solicitud al de la siguiente manera dentro del TE:
![[Pasted image 20250919222217.png]]

Perfecto como dijimos incrustamos una solicitud dentro de TE y lo que hacemos es mediante el CL que se encuentra dentro y por el error que se produce en el back-end se interpreta y logramos inyectar la petición:

![[Pasted image 20250919222331.png]]

Lab Terminada.


### Lab: Exploiting HTTP request smuggling to perform web cache poisoning

En este laboratorio al parecer es envenenar la cache de la web donde mediante esto tenemos que hacer un `document.cookie` dentro de una alerta, tenemos a disposición al exploit-server así que seguramente tendremos que almacenar el código en javascript y vamos a tener que ejecutarlo desde una web maliciosa.

Viendo que tenemos que efectuar código malicioso lo primero que se nos ocurre es ver en el historial del Proxy al cargar la web que solicitudes se hacen y encontramos una que nos puede servir:
![[Pasted image 20250919223059.png]]

Esta solicitud lo que hace es de forma constante cargar el recurso de tracking.js cuando cargamos la web, además si nos fijamos que tiene un `cache-control` donde define como `max-age=30` lo que nos deja en claro que eso durara en cache almacenado por 30 segundos, esto también lo podemos comprobar en el repeater y enviando solicitudes continuas para ver como cambia y la llegar a 30 se reinicia.

Bueno vamos a mantener esta petición ya que vamos a tratar de emplear eso para cargar nuestro código malicioso:
![[Pasted image 20250919223657.png]]\

En la descripción de la maquina leímos que el front-end no contempla TE por lo que debe ser CL, es por eso que armamos la solicitud de esa manera, probemos a ver si logramos provocar el error que queremos:
![[Pasted image 20250919223854.png]]

Logramos el error controlado por lo que podemos continuar.

Bueno ahora vamos a intentar aprovecharnos del exploit-server para ejecutar el código JavaScript. revisando nos encontramos con el siguiente botón dentro de los post:
![[Pasted image 20250919224357.png]]

Vemos que apunta a lo mismo pero al hacerle click vamos a ver como nos lleva al siguiente post ósea al 8:
![[Pasted image 20250919224445.png]]

El problema es que no vemos que se este empleando alguna pagina con Next Post para efectuar esto sino es mas como un follow redirect por lo cual vamos a capturar el link para ver que hace:
![[Pasted image 20250919224633.png]]

Vemos que efectivamente se aplica el follow redirect pero esta arrastrando la cabecera Host, esto lo podemos controlar:
![[Pasted image 20250919224737.png]]

Vemos que nos dice cabecera Duplicada, esto es porque al incrustarse el resto de información de la petición a la que se sincroniza esta cuenta con la cabecera Host y lo duplicaría, para evitar esto lo que podemos hacer es terminar la petición de forma que lo que se incruste sea en el body y ya no se interprete como cabecera, nos quedaría de la siguiente manera:
![[Pasted image 20250919225019.png]]

Perfecto ya evitamos el problema y en este punto si nos fijamos en la respuesta el `location` ya cambio por lo que ya tenemos todo listo para reenviar a nuestra web maliciosa, un concepto clave es que no importa que se este usando un parámetro en este caso a donde nos lleva es a `post` por lo tanto a nuestro exploit-server lo llamamos post y listo, esto podemos hacerlo de la siguiente manera:

![[Pasted image 20250919225401.png]]

![[Pasted image 20250919225637.png]]

Perfecto logramos incrustar el follow redirect, ahora el entender esto es clave lo que sucede es que nosotros con esta segunda petición que queda colgada lo que sucede es que vamos al recurso de tranquink.js que tenemos capturado a cargar luego de que termine el tiempo de cache y luego de mandar nuestro smuggling y vamos a ver como se aplica para ese contenido el follow redirect ya que se sincroniza a ese recurso y nos carga en la cache el JavaScript malicioso:
![[Pasted image 20250919230138.png]]

Ahora si revisamos:
![[Pasted image 20250919230151.png]]

Lab Terminado.


### Lab: Exploiting HTTP request smuggling to perform web cache deception

El objetivo es obtener el apli key de otro usuario usando la cache.

El comienzo es el mismo que el del anterior lab, donde vamos a ver las solicitudes que se hacen al cargar la web como es la de traking.js:
![[Pasted image 20250919230938.png]]

De forma concreta vamos a aprovechar eso para cargar y ver el api key, dejémoslo en el repeater listo para usarlo a su tiempo.

Vamos ya con esto a capturar la petición en la web como siempre:
![[Pasted image 20250919231310.png]]

Esto es bueno ya que logramos el error controlado.

En este punto, primero vamos a entender que esta pasando al enviar esta petición y lo que sucede es que como no estamos completando la petición hasta el body todo lo que se inserte como cabeceras y eso se tomara como válido, entonces nosotros lo que queremos dejar en cache es la ruta `/my-admin` ya que si esto lo dejamos en la cola y se sincroniza con la petición del tacking.js cargada por el usuario al nosotros cargar esto se arrastra su cookie de sesión y pasamos como el usuario admin, vamos a intentarlo:

![[Pasted image 20250919232103.png]]

![[Pasted image 20250919232121.png]]

vemos que se cacheo en ese recurso y nos carga como admin y tenemos su api key:

![[Pasted image 20250919232211.png]]

Lab Terminado.

### Lab: Bypassing access controls via HTTP/2 request tunnelling

En este caso tenemos un front-end pero el servidor hace downgrade  HTTP/2 , además de que una de sus seguridades implementadas filtra por sus nombres. Para resolver ese laboratorio tenemos que eliminar al usuario `Carlos`.

Además todo lo que conocemos y hemos practicado por ahora no nos servirá de nada ya que no es vulnerable a una HTTP smuggling clásica. Pero si es vulnerable a request Tunneling.

Bueno como siempre comenzamos capturando una petición usual:
![[Pasted image 20250920182357.png]]


Perfecto, si recordamos lo que nos dice el lab es que no se validan correctamente los nombres de la cabeceras, con esto en mente podemos hacer lo de la siguiente forma:
![[Pasted image 20250920182837.png]]

Como vemos vamos a agregar una cabecera pero en la parte del mismo nombre le estamos asignando el valor, además de que estamos insertando una segunda cabecera Host lo que debería darnos un error de cabecera duplicada:
![[Pasted image 20250920182859.png]]

Vemos que es inyectable, esto por el `test.com` que si recordamos es el valor que le definimos a la cabecera Host, en este punto se nos pueden ocurrir algunas cosas como:
![[Pasted image 20250920183305.png]]

Vemos que estamos insertando una petición mediante la configuración de una nueva cabecera, podemos intentar esto a ver que sucede:
![[Pasted image 20250920184936.png]]

Insertamos pero no logramos ver la respuesta de la segunda petición.

Algo que podemos tratar de hacer es cambiar el método que se esta empleando de GET a HEAD ya que muchas veces este tipo de peticiones en ocasiones nos logra mostrar los bytes que estamos consultando y los bytes que se están esperando:
![[Pasted image 20250920185337.png]]

Podemos observar como lo cambiamos y enviemos la solicitud a ver que obtenemos:
![[Pasted image 20250920185904.png]]

Vemos que nos responde diciendo que se han recibido `3076` bytes, pero se esperaban `8866`, el por que no vemos información es porque queda en espera de los bytes faltantes pero como no llegan pues no nos carga la web.

En este punto podemos intentar apuntar a otros recursos como el `/admin` esto para intentar hacer que los bytes recibidos sean iguales o mayores a los esperados:
![[Pasted image 20250920190208.png]]

![[Pasted image 20250920190232.png]]

Vemos que nos responde a las dos solicitudes como no autorizadas, esto solo quiere decir que tenemos algo que queda pendiente. En ocasiones el front-end envía ciertas cabeceras ocultas las cuales pueden ser esenciales para que la consulta sea correcta, para esto podemos provecharnos de la barra de búsqueda ya que esta si nos fijamos inserta información:

![[Pasted image 20250920190446.png]]

Entonces vamos a intentar enviar la solicitud en POST y vamos a aprovechar la inyección en el nombre de cabecera para definir un nuevo CL, esto debido a que en ocasiones el back-end lo interpreta y nos permite filtrar por mas datos de los que debería:
![[Pasted image 20250920190939.png]]

![[Pasted image 20250920191003.png]]

Excelente ya estamos filtrando por mas información, esto es debido a que confunde al back-end ya que no es una forma común de hacer la solicitud.

Bueno vamos a aumentar el valor para lograr sacar toda la información posible:
![[Pasted image 20250920191338.png]]

Vemos algunas cosas, vamos a verlo en raw para ver la información correcta:
![[Pasted image 20250920191439.png]]

Aparte de la cookie vemos que tenemos 3 campos los cuales parecen ser esenciales que comienzan con `X` así que vamos a copiarlos y a definirlos en la smuggling que teniamos:
![[Pasted image 20250920191654.png]]

Vemos que sigue con la respuesta `401` , pero vamos a seguir jugando con las cabeceras, e investigando un poco damos con que en el de client se suele definir un cliente, podemos intentar con `administratror` que es el usuario administrador que conocemos:

![[Pasted image 20250920191830.png]]

Tampoco, vemos que la verificación tiene un `0` podemos pasarle a `1` a ver que sucede:
![[Pasted image 20250920191915.png]]

logramos de forma correcta en la smuggled un `200 OK`, podemos ver que pasa si pedimos la solicitud de nuevo a la raíz:
![[Pasted image 20250920192234.png]]

Vemos de nuevo el error, ahora tenemos que encontrar una web que cargue menos de 3608 bytes, para que nos permita incrustar la información, podemos intentar con el login:
![[Pasted image 20250920192347.png]]

Perfecto ya tenemos los dos en `200 OK` por lo  tanto si revisamos ya tenemos la url para eliminar el usuario `carlos` y la usamos en la smuggling para poder ejecutarlo:
![[Pasted image 20250920192605.png]]

Bueno tenemos de nuevo el problema ya que nos tocaría buscar una consulta con una cantidad menor a los 372 bytes o igual para que funcione correctamente, pero pese al error en el lab esto si funciona:
![[Pasted image 20250920192711.png]]

Lab Terminado:


### Lab: Web cache poisoning via HTTP/2 request tunnelling

En este laboratorio el propósito general es envenenar la cache de la web y tratar de que a otro usuario se le genere un `alert(1)`, el usuario victima recarga la web principal cada 15 segundos, además de que no vamos a poder ejecutar los smuggling attacks que hemos hecho hasta el momento.

Vamos primero a capturar una consulta normal y vemos que tenemos:
![[Pasted image 20250920195727.png]]

en este caso vemos que tenemos lo que es el valor que se mantiene almacenado en la cache que son 30 segundos, nosotros para que no tardar tanto tiempo es implementar un parámetro que nos permite actualizar de forma rápida la cache sin esperar los 30 segundos:

![[Pasted image 20250920200028.png]]

vemos que pasa el `age: 0` lo que quiere decir que se actualiza la cache cuando le pasamos un parámetro.

Bueno en este punto sabemos que hace un downgrade de HTTP/2 a HTTP/1.1, podemos intentar lo mismo de la maquina anterior que es inyectar las peticiones dentro del valor de la cabecera y luego dentro del nombre de la cabecera, pero esto no va a funcionar.

Entonces en este punto lo que vamos a hacer es comenzar a modificar el path de la siguiente manera:
![[Pasted image 20250920200805.png]]

lo que estamos haciendo es modificar el HTTP/2 y desplazarlo para abajo y para que esto no quede colgado le estamos ingresando antes el `Test:`, el `/post` es una ruta valida pero sin parámetro no lo es y nos debe indicar el parámetro que le hace falta, veamos si esto le gusta:
![[Pasted image 20250920200931.png]]

Pues al parecer esto si le gusta, esto seria un punto de entrada interesante a tomar en cuenta, en este punto lo que podemos hacer es intentar una inyección, nunca olvidarnos de tratar de aislar la nueva petición con ayuda del `\r\nj` quedando de la siguiente manera:
![[Pasted image 20250920201311.png]]

Un problema que se tubo es que la cache queda almacenada y agregamos el parámetro `testing=1` para actualizarla de forma inmediata, es cuestión de variar con eso para que se actualice pero ya envía la petición, el problema es que no vemos una segunda respuesta y esto puede que tenga que ver con los tamaños de nuevo, si nos fijamos en la respuesta esto tiene un total de 8689.

Para tener una mejor idea de lo que esta pasando con los tamaños podemos intentar jugar con el método HEAD:
![[Pasted image 20250920201557.png]]

En este caso con la cabecera `HEAD` podemos observar que en realidad nos retorna las dos peticiones, y si revisamos un poco la web vamos a ver como en esta nos refleja como contenido la petición:
![[Pasted image 20250920202651.png]]

Con esto podemos intentar enviar alguna cabecera extra a ver si esto se representa pero no va a funcionar, lo que se nos ocurre es buscar algún otro recurso que se llama de forma recurrente:

![[Pasted image 20250920202808.png]]

Encontramos este script en JS, el cual se llama de forma continua. En este punto podemos intentar ver que pasa si reducimos la url :
![[Pasted image 20250920202903.png]]

Vemos que la petición nos responde con un follow redirect pero mas importante lo que le pasamos en la url se refleja en la cabecera de `Location`, en este punto lo que se puede hacer es intentar ver si podemos incrustar etiquetas html y ver si esto nos valida en la smuggling:

![[Pasted image 20250920203106.png]]

Vemos que no tenemos ninguna limitación, en este caso vamos a intentar enviarlo en la smuggling a ver que sucede:
![[Pasted image 20250920203314.png]]

Lo enviamos pero vemos el error y algo igual de importante es que cambiamos a HTTP/2 ya que si no no se realiza la solicitud a ese recurso.

El problema ahora es encontrar algún recurso que nos retorne menos de `174` de tamaño va  a ser algo complicado pero nosotros podemos intentar rellenar esto con bytes, vemos que pasa si al final agregamos algunas `AAAAAA` el propósito es que aumente el `174` :
![[Pasted image 20250920203753.png]]

Vemos que nos da un error de comunicación en este caso y en otros casos no nos aumenta el valor se queda en `174` puede ser que no nos este interpretando, en este punto lo que podemos hacer es ir cambiando de paginas por ejemplo voy a intentar con los post pero ya voy a intentar con una carga útil de 8000 veces la A para ver si en alguna nos funciona:
![[Pasted image 20250920205348.png]]

Vemos que logramos inyectar en el post 2 y si ingresamos a este post vamos a ver el alert(1):
![[Pasted image 20250920205453.png]]

Bueno el problema ahora es que lo esta esperando en la pagina principal, para que nuestro chunk de `A` supere eso tendrían que ser algunas mas por lo que vamos a ingresar una `8600`:
![[Pasted image 20250920205654.png]]

Perfecto, logramos ingresarlo en la raíz, solo toca esperar y ver si se completa el lab:

![[Pasted image 20250920205727.png]]

Lab Terminado.


### Lab: Client-side desync

Tenemos que entender la diferencia entre el `Request Smuggling` y un `Client Side Desing`:

Usual mente en los Request Smuggling se acontece en una sola solicitud donde mediante e CL y el TE comenzamos a definir parámetros los cuales nos permite pasar por algún filtro donde llegamos a generar alguna discrepancia enter el back-end y el front-end donde vemos la desincronización y logramos de lado del back-end el ver cierto tipo de consultas que no son correctamente procesadas a final de cuentas y esto genera los riesgos, donde la victima principal es el servidor.

En el Client Side Desync como mínimo 2 o mas solicitudes parecido a cuando creamos grupos en BurpSuite que permite enviar dos solicitudes y en este caso la discrepancia se produce entre el back-end y el cliente que reutiliza la conexión, donde la victima principal es el cliente. El problema principal es que el Client Side Desync ignora por completo el Content Length y responde enseguida el back-end de forma inmediata pero eso si dejando la solicitud abierta a nuevos bytes donde la siguiente solicitud que se tramite se interpretara como parte del cuerpo de la solicitud anterior. Nosotros en este caso solo controlamos la primera solicitud y todo lo demás depende de la otra persona ya sea una victima o un navegador.

![[Pasted image 20250920211515.png]]


bueno vamos a comenzar interceptando una petición normal del home a ver como se esta tramitando y de paso haremos lo típico que es dejar la solicitud con las cabeceras mínimas:
![[Pasted image 20250920211815.png]]

Vamos a agregar una nueva petición y vamos a intentar colarla pero vamos a ver que no logramos nada ya que nos responde de forma inmediata la web, sin importar de que estemos inflando el content Length:

![[Pasted image 20250920212146.png]]

Bueno esto es una forma de ver que estamos frente a un Cliente Side Desync, lo que vamos a hacer en este punto ya que nos planteamos una hipótesis es buscar una petición normal por ejemplo al redirect que se estaba aplicando a `/en` :
![[Pasted image 20250920212437.png]]

Bueno usual mente o lo que se hiso en otras maquinas es enviar la primera solicitud maliciosa y luego enviar la otra esto provocaba que la primera solicitud que se quedaba colgada por falta de bytes se sincronizara a la segunda, en este caso eso no pasa.

Para nosotros podemos hacer esto de sincronizar la una solicitud con la otra lo que se tiene que hacer en el caso de la `Client Side Desync` es unificar todo en una misma conexión TCP (Esto es esencial) por lo tanto lo que vamos a hacer es jugar con el grupos ósea crear un grupo con las dos solicitudes lo que vamos a hacer es configurar el send como `Send group (Single Connection)`:

![[Pasted image 20250920212832.png]]

Las dos solicitudes están en el mismo grupo y si ahora enviamos esto el propósito es lograr ver el error en la otra solicitud:
![[Pasted image 20250920212919.png]]

Observamos que nos funciona.

Bueno el objetivo es robarle a una victima la cookie de session y para esto vamos a hacer uso del Exploit-Server.

En este punto lo que tenemos que hacer es automatizar esta petición que tiene que hacer la victima, ósea el recargar el `/en` para que funcione, vemos que podemos subir posts por lo que vamos a interceptar la solicitude subida de uno para anlalizarla:
![[Pasted image 20250920213308.png]]

Esto como tal nos crea un post y podemos quitar algunas cosas que no nos sirve:
![[Pasted image 20250920213448.png]]

vemos que se creo el post así que si funciona, el punto ahora es que esta solicitud minima se realice en nuestra solicitud de atacante de la siguiente manera:
![[Pasted image 20250920213753.png]]

Omitamos la respuesta que de la anterior solicitud pero veamos que la otra solicitud que le estamos pasando es para que se haga un post con mas información, en este punto lo que vamos a hacer es enviar la solicitud en grupo y ver que sucede:
![[Pasted image 20250920213937.png]]

![[Pasted image 20250920213952.png]]

Se va sin problema y vemos que se esta incrustando la información de la solicitud siguiente, podemos aumentar esto a 700 para que se vea lo mas posible, ahora aparte de eso tenemos que crear en nuestro exploit server un código en JavaScript que nos automatice esto el cual quedara de la siguiente forma:
```js
<script>
  smuggledRequest = [
  "POST /en/post/comment HTTP/1.1",
  "Host: 0a38005f045753e6804b034b007800b9.h1-web-security-academy.net",
  "Cookie: session=5yAGYNkbjLy9YmQ7aIzzZHOYnDFuI3Hm;",
  "Content-Type: application/x-www-form-urlencoded",
  "Content-Length: 800",
  "",
  "csrf=jDMLd5pw0ewbCikWBxA3BS0jZUPHtyBW&postId=8&name=testing&email=test@test.com&website=http://testing.com&comment=test"
  ].join("\r\n")

  fetch("https://0a38005f045753e6804b034b007800b9.h1-web-security-academy.net", {
    method: "POST",
  body: smuggledRequest,
  credentials: "include",
  mode: "no-cors"
});
</script>
```

listo una ves que enviamos esto a la victima veremos que en el post se sube uno nuevo con información:
![[Pasted image 20250920221354.png]]

Perfecto ya tenemos la cookie que necesitamos, vamos a intentar ingresar:

![[Pasted image 20250920221454.png]]

Lab Terminado.


### Lab: Server-side pause-based request smuggling

En este laboratorio nos vamos a encontrar que el servidor de frond-end transmite la solicitud a back-end y el back-end no cierra la conexión después de un tiempo en determinados endpoints.
Para solucionar esta maquina tenemos que identificar una pausa basada en el CL 0 desync vector. donde el objetivo es borrar al usuario carlos:

Lo que vamos a hacer es recargar la web mientras pasa por el BurpSuite para ver todas las solicitudes de que se tramitan. 

![[Pasted image 20250921103425.png]]

Vemos que tenemos multiples solicitudes que se tramitan en la web donde el endpoint esta bajo la ruta de resource, podemos seleccionar el de labHeader.js y enviarlo la repeater para analizarlo:
![[Pasted image 20250921103600.png]]

Lo que hacemos es pruebas donde quitando parte de la ruta sin dejar ninguna barra colgando vemos como se aplica un redirect, esto funciona siempre sobre todo el recurso de resources, vamos a dejarlo solo como resources y darle a la solicitud el tratamiento común que le hemos brindado en todos los demás labs:
![[Pasted image 20250921103911.png]]

Vemos que si podemos hacer la petición de esa manera, lo que podemos hacer intentar una inyección común de la siguiente manera:
![[Pasted image 20250921104236.png]]

La verdad es que no sucede nada.

Lo que vamos a hacer ahora es usar una extension del BurpSuite la cual es Turbo Intruder, el propósito es diseccionar las peticiones en tiempos de espera, para entenderlo bien vamos a enviar la primera parte, lo que esta marcado en la imagen:
![[Pasted image 20250921104552.png]]

Y luego de pasar unos segundos enviamos la segunda parte, esto claro en la misma solicitud.

Entonces lo que sucede es que nosotros enviamos lo primero y lo que pasa es que el back-end se queda esperando mas datos correspondientes al cuerpo de la solicitud pero no le enviamos esto pasados unos 61 segundos.

Entonces lo que sucede es que con lo primero el back-end espera por datos aproximadamente los 61 segundos y pasados este tiempo como no recibió el cuerpo de datos que esperaba en ocasiones el parser del back-end se refresca por lo que no interpreta los datos entrantes como parte de la solicitud anterior, sino que interpreta la información como una nueva solicitud.

Entonces para lograr este efecto vamos a enviar esto al turbo intruder, no es mas que click derecho -> extensiones -> Tubo Intruder -> Send to turbo Intruder, vamos a enviarlo y veremos la siguiente interfaz:
![[Pasted image 20250921105403.png]]

Vamos a cambiar el `Last code used` por el `examples/default.py` :
![[Pasted image 20250921105502.png]]

Bueno vamos a dejar el código que tenemos de la siguiente manera:
![[Pasted image 20250921105613.png]]

Ahora si vamos a ver como podemos diseccionar la petición por tiempos.

Lo que vamos a hacer son pequeñas divisiones de código de la siguiente manera:![[Pasted image 20250921105854.png]]

bueno eso seria de forma general el código, pero nosotros necesitamos manejar mejor esto y es que vamos a cambiar multiples veces la segunda petición y el content length que esta seleccionado, es por eso que vamos a mejor dejarlos representado como `%s` de string en python:
![[Pasted image 20250921110025.png]]

Bueno ya con esto vamos a crear nuestra smuggled request que es nuestra segunda solicitud:
![[Pasted image 20250921110207.png]]

También configuramos una solicitud normal debido a que tenemos que luego enviar una solicitud usual a la cual se sincronizara la smuggled request.

![[Pasted image 20250921110453.png]]

Se esta empelando un concepto de python mediante `engine.queue()` que nos permite unificar la request completa, donde le pasamos `attacker_request` la cual es la petición con los `%s` para ingresar los strings donde gracias al `engine.queue()` y pasándolo como una lista vamos a poder incrustar la información donde le pasamos primero `len(smuggled_request)` que es el tamaño de la cadena que lo primero que se incrusta es el valor de Content Length y luego le pasamos `smuggled_request` tal cual ya que tenemos que incrustar la smuggled.

Ahora para diseccionar esto por tiempo hacemos lo siguiente:
![[Pasted image 20250921111131.png]]

Definimos luego lo que es `pauseMarker` el cual nos permite definir un delimitador en donde tenemos que pasarle como lista el `\r\n\r\nGET` que es la ultima parte de nuestra petición principal:
![[Pasted image 20250921111050.png]]
Ahí entendemos el porque de la cadena, y luego definimos un `pauseTime` en el cual vamos a definir el tiempo en milisegundos donde recordemos que `1segundo = 1000milisengundos`, entonces como tenemos que esperar 61 segundos definimos 61000 milisegundos. Esto del tiempo ya se probo con anterioridad hasta dar con el correcto pero si no sabemos o no nos lo dicen o es en un entono real tenemos que probar hasta encontrar el tiempo exacto.

Entonces lo que esa línea nos dice es que vamos a enviar la petición pero al encontrar el `\r\n\r\nGET` vas a pausar y a esperar 61 segundos para enviar lo demás. El objetivo es que el back-end nos interprete estos datos como una nueva consulta y esto permite colarle una consulta nueva.

Entonces vamos a enviar la solicitud normal de la siguiente forma:
![[Pasted image 20250921115605.png]]

Vemos que es lo mismo `engine.queue()` y en este caso la normal_request, ahora le damos al botón d e attack  y veamos que es lo que sale:
![[Pasted image 20250921115625.png]]

Luego vemos lo siguiente:
![[Pasted image 20250921113857.png]]

eso es el contador donde nos indica que se envío la primera parte y al llegar a 61 se enviara la segunda y luego debería enviarse la normal, lo que obtenemos es lo siguiente:
![[Pasted image 20250921114618.png]]

Vemos la solicitud smuggled primero y se envío sin problemas, pero si revisamos la solicitud usual a la raíz vamos a ver lo siguiente:

![[Pasted image 20250921114719.png]]

Como podemos observar se realizo las solicitudes a la raíz pero nos responde lo de la solicitud smuggled por lo tanto esto esta funcionando.

Ahora con esto ya podemos probar que tenemos en `/admin`:

![[Pasted image 20250921115656.png]]

Veamos que obtenemos:
![[Pasted image 20250921121145.png]]

ahora vemos que llegamos a la ruta, pero nos dice que el panel de admin solo se puede acceder de forma local, esto ya lo hemos echo, es cuestión de modificar la cabecera Host para ver que sucede:

![[Pasted image 20250921121314.png]]

![[Pasted image 20250921121540.png]]

Perfecto ahora ya tenemos el panel y podemos eliminar un usuario pero lo que tenemos que hacer es automatizar esto, para eso me voy a ver el código y a copiar solo la parte del formulario:
![[Pasted image 20250921121702.png]]

Vemos un campo `csrf` donde podemos completar la petición de la siguiente manera:
![[Pasted image 20250921123622.png]]

vemos todo lo que se modifico esto enviamos y solo nos queda esperar para ver si el lab se completa:

![[Pasted image 20250921124344.png]]

Lab Terminado.
