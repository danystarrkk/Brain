---
aliases:
  - HTTP Smuggling
  - Request Smuggling
tags:
  - hacking/explotacion
  - redes/conceptos
estado: 🟢 Terminado
---
# Vulnerabilidades de HTTP Request Smuggling

Las vulnerabilidades de HTTP Request Smuggling se presentan cuando el inicio y fin de una solicitud HTTP son interpretados de manera diferente por el front-end y el back-end de una aplicación.

De forma más entendible, lo que sucede es que el front-end, al recibir una solicitud, contempla ciertos datos o atributos, pero al enviarla al back-end, este contempla cosas extras que, aunque el front-end no las revisa, sí puede enviarlas y el back-end las interpretaría.

> [!info] Fundamento del HTTP Request Smuggling
> Esta vulnerabilidad se basa en la ambigüedad en la forma en que los servidores front-end y back-end procesan los encabezados `Content-Length` y `Transfer-Encoding`. Cuando hay una discrepancia en la interpretación de estos encabezados, un atacante puede "smuggle" (contrabandear) una segunda solicitud HTTP dentro de la primera, lo que puede llevar a bypass de controles de seguridad, envenenamiento de caché, robo de sesiones, y más.

En este punto, tenemos que entender dos conceptos:

## Content-Length

Como tal, esto hace referencia al tamaño de los datos en la solicitud. Si enviamos un total de 3 datos, vamos a ver un `Content-Length: 3`:

![[Pasted image 20250918103256.png]]

> [!info] `Content-Length`
> El encabezado `Content-Length` especifica la longitud del cuerpo de la entidad de la solicitud en octetos (bytes). Es un mecanismo simple y directo para indicar dónde termina el cuerpo de la solicitud. Si este encabezado está presente, el servidor lo utiliza para determinar el final del mensaje.

## Transfer-Encoding

Esto funciona de una forma semejante a `Content-Length` pero con una representación distinta. Vamos a tener 3 factores: primero, el `Transfer-Encoding: chunked` por ejemplo; luego, debajo, el tamaño en bytes en formato `hexadecimal`; luego, debajo, los datos; y finalizamos el chunk con un `0`. Esto se vería de la siguiente manera:

![[Pasted image 20250918103844.png]]

> [!info] `Transfer-Encoding: chunked`
> Cuando se utiliza `Transfer-Encoding: chunked`, el cuerpo del mensaje se divide en una serie de "chunks" (fragmentos). Cada chunk comienza con su tamaño en formato hexadecimal, seguido de un CRLF, luego los datos del chunk, y finalmente otro CRLF. El final de la secuencia de chunks se indica con un chunk de tamaño cero (`0`), seguido de dos CRLF (`\r\n\r\n`). Este método es útil cuando el tamaño total del cuerpo de la solicitud no se conoce de antemano.

Entonces, lo que sucede es que desde HTTP/1.1 existe soporte generalizado para enviar múltiples solicitudes HTTP a través de un único socket TCP. El protocolo es extremadamente simple: las solicitudes HTTP se colocan una detrás de la otra, y el servidor analiza los encabezados (`Transfer-Encoding` o `Content-Length`) para determinar dónde termina cada una y dónde comienza la siguiente. Por lo tanto, el vector de ataque es que si logramos controlar o se permite el control del tamaño, lo que sucederá es que podremos apilar en esa data extra consultas donde, si el back-end las interpreta, podremos filtrar información. El objetivo principal es burlar las diferentes validaciones implementadas en el front-end, que es donde más se aplica, y en ocasiones el back-end.

## Tipos de HTTP Request Smuggling

Los ataques clásicos de HTTP Request Smuggling implican colocar tanto el encabezado `Content-Length` como el encabezado `Transfer-Encoding` en una única solicitud HTTP/1.1 y manipularlos para que los servidores front-end y back-end procesen la solicitud de manera diferente.

> [!tip] Mecanismo de Desincronización
> La clave para el HTTP Request Smuggling es la desincronización entre el front-end y el back-end. El front-end puede ser un balanceador de carga, un proxy inverso o un WAF. Si estos dos componentes interpretan la longitud de la solicitud de manera diferente, se crea una ventana para inyectar una segunda solicitud que el front-end no ve, pero el back-end sí.

La forma exacta en que se hace esto depende del comportamiento de los dos servidores:

-   **CL.TE**: El servidor de front-end usa el encabezado `Content-Length` y el servidor de back-end usa el encabezado `Transfer-Encoding`.
-   **TE.CL**: El servidor de front-end usa el encabezado `Transfer-Encoding` y el servidor de back-end usa el encabezado `Content-Length`.
-   **TE.TE**: Los servidores front-end y back-end admiten el encabezado `Transfer-Encoding`, pero se puede inducir a uno de los servidores a no procesarlo, ofuscando el encabezado de alguna manera.

> [!info] Impacto de los Tipos de Smuggling
> Cada tipo de smuggling (CL.TE, TE.CL, TE.TE) requiere una técnica de ataque ligeramente diferente para lograr la desincronización. Comprender cuál encabezado prioriza cada servidor es crucial para construir el payload correcto. Por ejemplo, en CL.TE, el front-end leerá `Content-Length` y pasará una cantidad fija de bytes al back-end, mientras que el back-end, al ver `Transfer-Encoding: chunked`, esperará un `0` para finalizar la solicitud. Si el `Content-Length` es menor que el cuerpo completo del chunked, el back-end interpretará los bytes restantes como el inicio de una nueva solicitud.

## Laboratorios PortSwigger

### Lab: HTTP request smuggling, confirming a CL.TE vulnerability via differential responses

Lo que queremos conseguir es que al momento de acceder a la raíz veamos un `404`.

Primero, interceptamos la solicitud:
![[Pasted image 20250918105524.png]]

Vamos a pasarla al método POST:
![[Pasted image 20250918105624.png]]

Perfecto, en este caso vemos que ya está cargando en `HTTP/1.1`.

Vamos a borrar todo lo que en este caso no nos importa y vamos a dejar lo que necesitamos, nos queda de la siguiente manera:
![[Pasted image 20250918105756.png]]

Lo que vamos a hacer es una pequeña prueba donde vamos a enviar datos:
![[Pasted image 20250918105828.png]]

Yo solo realicé cambios como el de `test=test`. El valor de `Content-Length` no lo modifiqué, ya que esto se adapta al tamaño de la data.

En este caso, no nos interesa que se actualice esto de forma automática, por lo que vamos a quitar en Burp Suite la opción. La cosa es dejarla desmarcada para que no lo realice:
![[Pasted image 20250918110104.png]]

> [!tip] Deshabilitar Actualización Automática de `Content-Length` en Burp Suite
> Para realizar ataques de HTTP Request Smuggling, es fundamental deshabilitar la actualización automática del encabezado `Content-Length` en Burp Suite. Esto se hace en `Proxy > Options > Intercept client requests` y desmarcando la opción "Update Content-Length". Si Burp actualiza automáticamente este encabezado, los payloads de smuggling no funcionarán correctamente, ya que la longitud del cuerpo se ajustaría y eliminaría la desincronización deseada.

Lo que vamos a hacer en este punto es primero activar la opción que nos deja ver los saltos de línea:
![[Pasted image 20250918110420.png]]

Ahora, algo importante a tener en cuenta es que cuando usamos el `Transfer-Encoding` al momento de cerrar el chunk con el `0`, tenemos que dar dos veces al enter, ya que si no, no estamos representando las cosas de forma correcta.

Ahora, en el laboratorio, el front-end está revisando el `Content-Length` pero el back-end admite o lee el `Transfer-Encoding`. Vamos a ver qué pasa si agregamos el `Transfer-Encoding` y enviamos la información en formato chunked. Como sabemos, por ahora estamos usando `HTTP/2` pero vamos a cambiarlo desde el inspector de la siguiente manera para que funcione:

![[Pasted image 20250918112828.png]]

Perfecto, vemos un error y es que esto se debe a que el front-end lo comprende, pero al llegar al back-end y ver el `Transfer-Encoding` pero no ver la estructura correcta, se produce un error interno. Ahora, creemos una estructura correcta y vemos qué sucede:

![[Pasted image 20250918113036.png]]

Excelente, ahora ya funciona, ya que es una estructura válida.

En este punto, lo que vamos a intentar es provocar un error de sincronización modificando el `Content-Length` para que no tome el cierre de la estructura `chunk` y se dé el error de sincronización:
![[Pasted image 20250918123217.png]]

En este punto, lo que queremos es tratar de efectuar un error del tipo `404`.

Primero, vamos a entender que nosotros, cuando enviamos solicitudes, estas entran en una cola donde se van procesando. Ahora, nosotros, dentro de una solicitud, podemos enviar otra solicitud. Recordemos que en la imagen anterior estamos generando un error por no enviar la estructura completa del `chunked`, pero cuando lo hacemos en el back-end, esto se procesa de tal manera que la solicitud termina donde está el `0` que cierra al `chunked`. Ahora, ¿qué pasa si nosotros, luego de cerrar la estructura, le pasamos otra petición de la siguiente manera?:
![[Pasted image 20250918125221.png]]

Como podemos observar, estamos agregando otra solicitud luego. Esto se tomará en realidad como otra petición. Es muy importante tener en cuenta que para que una estructura de una petición sea válida, por definición, lo que se tiene que emplear siempre es una cabecera. En ocasiones, podemos inventarnos una y poner como en la imagen `Test: A`. Lo que sucede al enviar esta petición es que el concepto antes de la imagen se va a enviar una solicitud normal, y el back-end, al ver el fin del chunked, va a cerrar la una, pero enseguida va a generar la otra. Esta es una petición que en realidad no se suele ver, ya que se desincroniza y queda colgada en la cola. Esta le cargará a la persona que realice otra solicitud a la misma web. Puede que se entrelacen solicitudes y se arrastre esta a cualquier lado. Por lo tanto, si la enviamos, vamos a ver que nos responde con normalidad, pero si recargamos en la web desde el navegador, vamos a ver que se enlaza esta solicitud que se quedó colgada en espera y nos llega al navegador:
![[Pasted image 20250918125933.png]]

![[Pasted image 20250918125944.png]]

Como vemos, esto es increíble, ya logramos el objetivo. Podemos intentar cambiar para jugar un poco de error a ver algún post, a ver si al recargar en el navegador nos sincroniza y nos manda a la web dentro de un post:
![[Pasted image 20250918130448.png]]

Perfecto, dejamos una solicitud colgada para ir a un post. Recarguemos a ver si en Firefox nos envía la respuesta:

![[Pasted image 20250918130507.png]]

La URL sale como la misma, pero se cargó lo del post, entonces es muy interesante cómo se pueden hacer estas cosas.
![[Pasted image 20250918130534.png]]
Lab Terminado.

### Lab: HTTP request smuggling, confirming a TE.CL vulnerability via differential responses

Bueno, en el laboratorio anterior, para la solicitud estándar, necesitamos solo definir el método, la dirección y una cabecera. En esta máquina, vamos a ver que con estos datos ya no son suficientes, lo que pasa es que se están revisando por un mayor número de contenidos, lo que provoca que se tome a la segunda solicitud que estamos filtrando como parte de la primera misma.

Entonces, lo que vamos a hacer en este caso es inflar la petición para que se tome como una segunda válida.

Primero, vamos a intentar causar una desincronización. Tomemos en cuenta que este laboratorio está invertido al anterior, donde ahora el front-end revisa por `Transfer-Encoding` y el back-end mira por `Content-Length`. Primero, comenzamos capturando la petición:
![[Pasted image 20250918162350.png]]
Perfecto, ahora tenemos cómo se tramita normalmente la solicitud. En este punto, vamos a hacer el procedimiento de cambiar el método, fijar la versión de HTTP:
![[Pasted image 20250918162445.png]]

Listo, esto carga normal. Ahora vamos a quitar lo que no nos sirve. Como el front-end de primeras no necesita el `Content-Length`, lo vamos a quitar también y a definir el `Transfer-Encoding`:
![[Pasted image 20250918162701.png]]

Vemos que esto sí es válido, así que perfecto. Podemos crear errores como no terminar el chunk para probar si se está comprobando el T.E:
![[Pasted image 20250918162746.png]]

Excelente, en este punto podemos ver que sí se revisa la información y nos arroja un error al enviar el chunk de forma incorrecta. Ahora, vamos a ver cómo ocasionar una desincronización para el `Content-Length`, el cual corre en el back-end:
![[Pasted image 20250918163134.png]]

La forma de causarlo es sencilla: le estamos diciendo al back-end que tenemos 20 bytes, pero solo enviamos 13. Esto provoca que se quede esperando por los bytes restantes y se genera la desincronización. Abusando de esto, lo que se va a hacer es:
![[Pasted image 20250918163727.png]]

A ver, lo que estamos haciendo dentro de esta nueva solicitud es traspasar primero el front-end con una estructura correcta, pero como vemos, tenemos dentro de nuestro chunk la solicitud que queremos enviar para que quede colgada. Entonces, aquí viene el truco con el `Content-Length`, donde al back-end le decimos que solo lea los primeros 4 bytes, hablamos del `1d\r\n`, y allí es donde a lo demás lo tomaría como una segunda petición:
![[Pasted image 20250918164052.png]]

Ahora vemos que no se efectúa problema al momento de tramitar esta petición y es debido a que la puede estar haciendo pasar como parte de la primera debido a que no tiene el tamaño o los datos suficientes para considerarla como otra. Lo que podemos hacer es lo siguiente:
![[Pasted image 20250918164543.png]]

Perfecto, ya tenemos una estructura más correcta, la cual, si entró correctamente, deberíamos ver la petición con el error 404:
![[Pasted image 20250918164649.png]]

Pues no, no logramos ver. Aquí es donde entra lo de inflar el `Content-Length`. Lo que hacemos es ponerle un dígito más al `Content-Length`. Esto lo que hace es dejarlo en formato "keep-alive" (en espera) de un dato adicional, y de donde lo saca es de una nueva petición entrante. Allí se sincroniza a ella, entonces la petición nos quedaría de la siguiente manera:
![[Pasted image 20250918165109.png]]

Vemos que al recargar la web principal se efectúa el error:
![[Pasted image 20250918165238.png]]

Entonces, lo que se hace en estos casos es inflar el `Content-Length` con el objetivo de dejar a esa segunda solicitud como en escucha de una entrante, la cual complete los bytes faltantes y se sincronice a la misma, viendo el resultado.
![[Pasted image 20250918165350.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to bypass front-end security controls, CL.TE vulnerability

En este laboratorio, tenemos que ingresar a `/admin` desde el cual vamos a tener que eliminar al usuario Carlos. El problema es que el front-end bloquea el acceso a este recurso, por lo tanto, tenemos que hacer un HTTP Request Smuggling para intentar acceder y borrar el usuario.

Empecemos capturando la petición:
![[Pasted image 20250918172624.png]]

Bueno, vamos a comenzar. Lo que sabemos es que el front-end interpreta lo que es C.L y el back-end lo que es T.E.

En este punto, vamos a tratar de enviar con un `Transfer-Encoding` válido para ver si esto se interpreta sin problemas:
![[Pasted image 20250918173230.png]]

Perfecto, vemos que ya tenemos la estructura básica. En este punto, lo que vamos a hacer es intentar primero el chunk dejándolo solo cerrado y agregar una estructura básica de petición dentro de esta misma de la siguiente manera:
![[Pasted image 20250918173519.png]]
Perfecto, nos deja pasar y si revisamos una segunda solicitud, veremos lo siguiente:
![[Pasted image 20250918173553.png]]

Intentemos con el panel admin a ver qué nos responde:
![[Pasted image 20250918173659.png]]
![[Pasted image 20250918173711.png]]

Vemos que nos responde, ya que esta segunda solicitud no pasa por el front-end como tal y solo se interpreta en el back-end. Vemos esta respuesta que es la de admin, pero nos dice que solo está disponible para usuarios locales.

Lo que podemos intentar es definir la cabecera como `Host: localhost`:
![[Pasted image 20250918174209.png]]

Vemos que efectivamente la solicitud se envía, pero si revisamos, vamos a ver lo siguiente:
![[Pasted image 20250918174257.png]]
Lo que nos dice es que estamos duplicando una cabecera. Esto quiere decir que no nos está interpretando la segunda como una petición, sino que lo hace como parte de la misma, lo que da origen al error de la cabecera duplicada.

Para evitar esto, tenemos que darle una mejor estructura para que se interprete como petición de la siguiente manera:
![[Pasted image 20250918174520.png]]

Ya estamos enviando, pero no sucede nada, y para esto vamos a intentar inflar el `Content-Length` de la siguiente manera:
![[Pasted image 20250918174623.png]]

Vemos que efectivamente ya se envía y si recargamos, veremos lo siguiente:
![[Pasted image 20250918174648.png]]

Ya logramos cargar la página de admin, esto solo cambiando el `Host: localhost` para que piense que la solicitud va de forma local y mediante una inflación del `Content-Length` logramos enlazar a una nueva solicitud. Con esto, ya podemos eliminar al usuario Carlos con la siguiente solicitud:

![[Pasted image 20250918175025.png]]

![[Pasted image 20250918175130.png]]

Vemos eso, pero si regresamos a la raíz, vamos a observar que:
![[Pasted image 20250918175215.png]]
Lab Terminado.

### Lab: Exploiting HTTP request smuggling to bypass front-end security controls, TE.CL vulnerability

En este laboratorio, vamos a realizar lo mismo con la diferencia de que en el front-end se está empleando el T.E y en el back-end se está empleando el C.L.

Vamos a comenzar capturando la petición y configurándola de forma correcta:
![[Pasted image 20250918180353.png]]
Perfecto, tenemos ya procesado correctamente esto. Ahora vamos a, dentro del chunk, ingresar la petición. Primero, vamos a causar un error con el objetivo de probar si tenemos correctamente operativa la inyección:

![[Pasted image 20250918181428.png]]

Perfecto, ya tenemos correctamente configurada la solicitud y esta nos retorna al panel de admin:
![[Pasted image 20250918181511.png]]

En este punto, vamos a crear para que elimine el usuario Carlos:
![[Pasted image 20250918181611.png]]

Perfecto, vemos que nos retorna la siguiente petición. Recordemos siempre inflar la petición para la segunda solicitud. Recordemos que para la segunda solicitud es desde la última línea, no solo el `test=test`, sino desde la última línea:
![[Pasted image 20250918181633.png]]

Aunque aparente que no, este mensaje nos está configurando que sí logró eliminar el mensaje. Esto lo sabemos, ya que solo es una máquina de práctica:
![[Pasted image 20250918181736.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to reveal front-end request rewriting

En este caso, lo que tenemos es un panel de administración en `/admin`, pero solo es disponible si usamos una cabecera que no conocemos, pero que tiene que tener como valor `127.0.0.1`.

Vamos a comenzar capturando la petición:
![[Pasted image 20250918201237.png]]

Bueno, ya hemos hecho algunas pruebas donde vemos que el front-end es CL y el back-end es TE. Es por eso que al definir un CL que no tenga todo, en este caso eliminamos el 0 del cierre del chunk, se ocasiona el error.

Bueno, ahora lo que podemos hacer es lo siguiente:
![[Pasted image 20250918201754.png]]
Realizamos una inyección común, pero recordemos que como estamos inflando el `Content-Length`, el contenido de la siguiente solicitud se incrusta en el body de la petición y se usa para sincronizarla, pero se incrusta en la petición, en este caso se pone después de `test`. Vemos que nos da la siguiente solicitud:
![[Pasted image 20250918201944.png]]

Nos está incrustando la información, como podemos ver. Entonces, podemos aprovechar esto para intentar ver toda la información posible. Vamos a inflar más el CL para ver toda la información posible:
![[Pasted image 20250918202054.png]]

Vemos al parecer una cabecera más que es `x-YGwWNN-Ip`. En este caso, podemos usar esta cabecera para darle el valor de `127.0.0.1` y esto en el mismo HTTP Smuggling enviamos hacia `/admin`. Vemos si logramos pasar:
![[Pasted image 20250918202420.png]]

Vemos si se logró en la web:
![[Pasted image 20250918202437.png]]
Ahora vamos a eliminar al usuario Carlos:
![[Pasted image 20250918202758.png]]

Por temas de que salía cabecera duplicada, se usó el método GET y listo:
![[Pasted image 20250918202829.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to capture other users' requests

La idea es obtener la cookie de sesión de un usuario. Cada vez que nosotros enviamos una petición, hace que la víctima también realice una petición.

La idea es similar a la anterior, pero en esta ocasión es mediante un comentario:

![[Pasted image 20250919115300.png]]

Perfecto, sabiendo que el front-end interpreta `Content-Length` y el back-end contempla el `Transfer-Encoding`, vamos a jugar creando un chunk para que lo interprete el back-end.

Vamos a capturar el momento de enviar un comentario primero:
![[Pasted image 20250919115823.png]]

Ahora, esto lo vamos a copiar para tramitarlo como una segunda petición:
![[Pasted image 20250919115855.png]]

Lo que vamos a hacer es eliminar cabeceras que por el momento no son útiles, quedando la siguiente petición:
![[Pasted image 20250919120026.png]]

Vamos a enviar a ver qué sucede:
![[Pasted image 20250919120512.png]]

Perfecto, enviamos la solicitud y es válida. Veamos qué nos retorna la solicitud enlazada:
![[Pasted image 20250919120547.png]]

Vemos que nos dice que tenemos un CSRF token no válido. Esto puede ser por la cookie. Podemos intentar poner la cookie a ver qué sucede:
![[Pasted image 20250919121447.png]]

Algo que nos acabamos de fijar es que cambia el valor del `CSRF token`, por lo que también lo cambiamos por uno no usado y capturado. Veamos si esto le gusta:
![[Pasted image 20250919121436.png]]

Vemos que nos responde como si tramitáramos el comentario. Vamos a ver qué obtenemos:
![[Pasted image 20250919121545.png]]

Perfecto, ahora el concepto es el mismo de la anterior máquina, donde inflando el `Content-Length`, vamos a intentar que se asigne como cuerpo de solicitud la información de la solicitud que está haciendo el otro usuario por detrás. De esta forma, vamos a intentar robarle la cookie. Para esto, inflamos mucho el `Content-Length`. Además, tomemos en cuenta que esto se está insertando en el parámetro de `website`, por lo que luego lo que tenemos que revisar es el link, y esto nos queda de la siguiente manera:
![[Pasted image 20250919122516.png]]

![[Pasted image 20250919122500.png]]

Vemos que tenemos una nueva publicación, por lo tanto, veamos qué contiene:
![[Pasted image 20250919124423.png]]

Vemos que se captura, pero no vemos la cookie. Vamos a aumentar más la inflación para ver si se logra ver la cookie:
![[Pasted image 20250919125557.png]]

Perfecto, en este caso para verlo en el post, lo que hacemos es cambiar el parámetro del comentario al final y lo inyectamos en el comentario, pero bueno, ya tenemos lo que necesitamos. Con esto, vamos a completar el laboratorio:

![[Pasted image 20250919125851.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to deliver reflected XSS

En este laboratorio, nos vamos a encontrar con un XSS en la cabecera del `User-Agent`. Tenemos que causar un `alert(1)`.

Vamos a comenzar identificando dónde está el XSS:
![[Pasted image 20250919150120.png]]

Por alguna razón, se declara el `User-Agent` como un input, lo que vamos a hacer en este punto es intentar modificar para cerrar la petición e inyectar código:
![[Pasted image 20250919150312.png]]

Como podemos ver, modificamos el `User-Agent` y esto se termina de reflejar en la respuesta. En este punto, lo que vamos a hacer es intentar inyectar una alerta a ver si se ejecuta:
![[Pasted image 20250919150534.png]]

![[Pasted image 20250919150511.png]]

Perfecto, en este punto lo que tenemos que hacer es un HTTP Smuggling para intentar colarle una petición a otro usuario con esto. Vamos a hacer lo siguiente en una petición de la página home:
![[Pasted image 20250919150841.png]]

Lo que hago es comprobar cómo se está tratando la petición y al parecer es para el front-end CL y para el back-end TE, por lo tanto, vamos a intentar pasarle la nueva solicitud maliciosa a ver qué sucede primero:
![[Pasted image 20250919151537.png]]

Perfecto, nos quedaría de la siguiente manera. Ahora es cuestión de enviar y esperar a ver si se completa el lab:
![[Pasted image 20250919151655.png]]
Lab Terminado.

### Lab: Response queue poisoning via H2.TE request smuggling

Lo que sucede es que cuando nosotros enviamos alguna petición, el back-end hace un downgrade de HTTP/2 a HTTP/1.1.

> [!info] Downgrade de HTTP/2 a HTTP/1.1
> Muchos servidores front-end modernos soportan HTTP/2 para optimizar la comunicación con los clientes. Sin embargo, es común que estos front-ends realicen un "downgrade" a HTTP/1.1 al comunicarse con los servidores back-end. Esta conversión puede reintroducir vulnerabilidades de HTTP Request Smuggling si la lógica de traducción de encabezados (`Content-Length` y `Transfer-Encoding`) no se maneja correctamente, permitiendo que un atacante explote la desincronización entre las dos versiones del protocolo.

Dos conceptos rápidos: el uso de HTTP/2 ya no es necesario especificar `Content-Length` y `Transfer-Encoding`, ya que se calculan de forma automática. Ahora, tenemos que comprender otra cosa, y es que el objetivo es secuestrar una petición que queda en el aire. Me refiero a que nosotros vamos a crear, como siempre, una especie de petición doble donde veremos cómo una nos responde y la otra corrompe la cola, haciendo que esta se sincronice con una solicitud de cualquier otra PC. Pero lo que se tiene que comprender es que cuando esto pasa, la respuesta que debería haber visto, por ejemplo, otro usuario al iniciar sesión, queda perdida. Nosotros lo que podemos hacer es recargar múltiples veces la web para recuperarla de nuestro lado. Esto es lo que tenemos que lograr en este lab.

Vamos a comenzar capturando una petición, modificándola y haciendo una petición no de error, sino una correcta que se quede colgada:
![[Pasted image 20250919163125.png]]

Aquí no necesitamos inflar ni nada, solo enviar múltiples veces las peticiones con el objetivo de que el usuario recoja nuestra petición y nosotros, mediante el envío de múltiples peticiones, logremos recuperar la de él:
![[Pasted image 20250919163301.png]]

Excelente, logramos recuperar la petición de él y tenemos la cookie de sesión, la cual podemos usar para entrar y eliminar al usuario Carlos:
![[Pasted image 20250919170952.png]]

Listo, Lab Terminado.

### Lab: H2.CL request smuggling

De igual forma, se aplica un downgrade desde el front hacia el back, donde el back-end interpreta `Content-Length`.

El objetivo es que a la víctima le salga una alerta con la cookie.

Primero, capturemos una petición usual:
![[Pasted image 20250919171516.png]]

Tenemos esto listo y vemos que funciona. Vamos a pasarle algo con más bytes:
![[Pasted image 20250919171612.png]]

Perfecto, podemos comenzar a intentar inyectar una petición para ver primero si podemos tramitar algún error:
![[Pasted image 20250919171857.png]]

Perfecto, con esto ya vemos que sí tenemos la vulnerabilidad. Ahora, el punto sería intentar ver si nosotros de alguna forma logramos reenviar a una web externa para desde un exploit externo cargar la alerta:
![[Pasted image 20250919173032.png]]

Vemos que cada vez que cargamos la web principal, se realiza una carga de archivos `.js`, como lo vemos allí. Lo que quiero intentar es ver cómo se altera la web cuando yo cambio datos, por ejemplo:
![[Pasted image 20250919173328.png]]

Vemos que cuando realizamos esta búsqueda, tenemos un estado de `302` para redirecciones y nos manda en la respuesta a `/js/`. Vemos qué pasa si le quitamos el `/js`:
![[Pasted image 20250919173426.png]]

Ya que vemos que esto aplica un redirect, es intentar incrustar esto en nuestra solicitud maliciosa y ver si podemos alterar la cabecera `Host` para que se inserte en un "follow redirect":
![[Pasted image 20250919173837.png]]

Vemos que funciona, aunque debemos quitar el `http` ya que eso sí lo incrusta. Podemos usar el exploit del lab para crear un código malicioso:
![[Pasted image 20250919174015.png]]

Perfecto, si llama la solicitud. Ahora lo que hacemos es introducir el código malicioso, guardarlo y esperar:
![[Pasted image 20250919174125.png]]

Se envió, veamos si se completa el lab:
![[Pasted image 20250919174655.png]]

Lab Terminado.

### Lab: HTTP/2 request smuggling via CRLF injection

Primero, tenemos que comprender que el `CRLF` es el `CR -> \r` y el `LF -> \n`.

> [!info] CRLF Injection
> `CRLF` (Carriage Return Line Feed) es la secuencia de caracteres `\r\n` que se utiliza para indicar el final de una línea en los protocolos de texto como HTTP. Una inyección de CRLF ocurre cuando un atacante puede insertar esta secuencia en una parte de la solicitud HTTP donde no debería estar, lo que puede llevar a la inyección de nuevos encabezados o incluso a la inyección de un cuerpo de solicitud completamente nuevo, facilitando ataques de HTTP Request Smuggling o Response Splitting.

Bueno, comenzamos viendo la web:
![[Pasted image 20250919185025.png]]

Vemos un campo de búsqueda, el cual vemos cómo funciona:
![[Pasted image 20250919185052.png]]

Vemos que tenemos un historial de búsquedas. Si esto sucede, puede que tengamos una cookie de sesión:
![[Pasted image 20250919185318.png]]

Entonces, lo que podemos pensar es, mediante el HTTP Smuggling, dejar colgada una solicitud la cual realice una búsqueda donde, al usuario cargar la página de inicio, esta se acople a la búsqueda e inflando el `Content-Length` lo suficiente, llegar a obtener una cookie de sesión.

Entonces, lo que podemos hacer es capturar la petición de una búsqueda:
![[Pasted image 20250919185733.png]]

Bueno, en este punto no sabemos qué es lo que el servidor soporta, si esto es TE o CL, por lo que vamos a probar primero con TE:
![[Pasted image 20250919185946.png]]

Enviamos el `Transfer-Encoding` y el chunk completo con el objetivo de ver si generamos errores, y pues no se generan. Luego hicimos lo de la imagen donde estamos enviando, luego del chunk, una solicitud con el objetivo de forzar un error, pero no logramos nada. Podemos intentar enviar un `Content-Length` también que esté un poco inflado para intentar forzar el sincronizado:
![[Pasted image 20250919190236.png]]

Seguimos sin generar el error.

Bueno, en ocasiones el front-end está configurado para que no puedas usar ciertas cabeceras. En este caso, puede estar bloqueando la cabecera del TE con el objetivo de que no se le pueda colar una solicitud alternativa.
Pero ahora, si no está correctamente configurado esto de las cabeceras, nosotros podemos hacer lo siguiente:
![[Pasted image 20250919190536.png]]

Nosotros, en el apartado de Request headers, podemos, como observamos, seleccionar agregar una nueva cabecera. Aquí es donde vamos a jugar un poco, y es que vamos a poner tal y como tenemos, pero en su valor vamos a hacer lo siguiente:
![[Pasted image 20250919190745.png]]

Lo que estamos haciendo es agregar una nueva cabecera, pero a esta le estamos aplicando un salto de línea y luego le agregamos lo que es el TE. Esto provoca que se inyecten cabeceras. Vamos a darle a "add":
![[Pasted image 20250919190941.png]]

Vemos que nos corta la solicitud y nos dice que no pueden ser representadas y más. No le hagamos mucho caso, vamos a enviar para ver si de esta forma logramos inyectar la cabecera y efectuar el error:
![[Pasted image 20250919191031.png]]

Nos lo está haciendo correcto y estamos burlando si nos pone restricciones en la cabecera.

En este punto, ya vamos a modificar de tal forma que contemple la cookie, ya que sin esta no nos asocia el historial. Luego, modificamos el método, ya que la búsqueda se tramita por POST en la raíz, y agregamos el parámetro `search` con un valor que decidamos y procedemos a inflar lo suficiente el `Content-Length` para obtener los datos:
![[Pasted image 20250919191903.png]]

Bueno, en este punto tenemos lista la petición. Lo que vamos a ocasionar es que al momento de enviar, se quedará la segunda petición sincronizada en espera de los bytes faltantes. Cuando otro usuario recargue la web principal, lo que va a pasar es que se completarán los bytes faltantes con los datos de la petición del usuario y se enviará. Esto provoca que, como esta solicitud está asociada a nuestra cookie, se nos quedará en el historial. Vemos si esto sucede:
![[Pasted image 20250919191951.png]]

Perfecto, podemos ver cómo ya tenemos la cookie de sesión. Vamos a iniciar sesión:
![[Pasted image 20250919192125.png]]

Lab Terminado.

### Lab: HTTP/2 request splitting via CRLF injection

En esta máquina, ya entra en juego que en la propia cabecera, al darle al enter `CRLF`, se inyecte una nueva petición, a diferencia de la anterior que, mediante una inyección de cabecera dentro de otra, logramos que no se detecte la cabecera que se estaba bloqueando.

La idea es que, mediante un envenenamiento de respuestas con el concepto de que nuestro HTTP Smuggling queda en cola, y es que al usuario intentar iniciar sesión, primero le retorna esa respuesta en cola y la respuesta al inicio de sesión del usuario queda ahora en cola, lo que nos permite a nosotros luego recuperarla.

En este punto, lo que vamos a hacer es capturar primero la petición del home:
![[Pasted image 20250919194044.png]]

Vemos que de por sí logramos enviar la solicitud, pero no sabemos cómo se está tramitando esto, así que vamos a intentar primero probar con TE de la siguiente manera:
![[Pasted image 20250919194228.png]]

Tratamos de generar un error en el TE cerrando el chunk de forma incorrecta con `X` y no con un `0` como es debido, pero no logramos el error.

En este punto, puede estar pasando lo que sucedió en la anterior máquina, donde vamos a intentar hacer bypass de la cabecera TE si es bloqueada, igual que en la máquina anterior:
![[Pasted image 20250919194509.png]]
![[Pasted image 20250919194539.png]]

Vemos que no, así que vamos a regresar y vamos a intentar lo siguiente:
![[Pasted image 20250919194648.png]]
Como vemos, pasamos el método a GET y nos dice que por este método no puede contener cuerpo, por lo que si quitamos el body pero dejamos la cabecera del TE:
![[Pasted image 20250919194746.png]]

Vemos que sí es válido. Lo que vamos a intentar es colar una nueva cabecera de la siguiente manera:
![[Pasted image 20250919195116.png]]

Vemos que no le vamos a enviar contenido en el body, pero mediante una cabecera y con el concepto similar al de la anterior máquina, vamos a intentar colarle una solicitud como vemos en la imagen:
![[Pasted image 20250919195240.png]]

Enviemos y vemos qué sucede:
![[Pasted image 20250919195309.png]]

Vemos que de esta forma logramos ocasionar la segunda solicitud y logramos el error deseado.

En este punto, sí nos sirve el error, ya que lo único que deseamos es que la víctima vea ese error y nosotros podamos ver la cookie de sesión, así que vamos a ver si lo logramos:
![[Pasted image 20250919200608.png]]

Perfecto, ingresemos y eliminemos a Carlos para completar el lab:
![[Pasted image 20250919200738.png]]

Lab Terminado.

### Lab: CL.0 request smuggling

Lo que tenemos en este lab es que en el front-end se está empleando lo que es el `Content-Length` y el back-end acepta todo.

Para este caso, donde vamos a hacer la inyección es en un recurso, debido a que muchas veces en los recursos, al cargarse, el servidor no realiza en el back-end ninguna comprobación. En este caso, vamos a capturar desde el historial cómo se realizó la petición de una imagen y la vamos a enviar al Repeater para analizarla:
![[Pasted image 20250919205018.png]]

Vamos, como ya estamos acostumbrados, a darle el tratamiento habitual para lograr tener datos mínimos, además de inyectar una nueva petición:
![[Pasted image 20250919205221.png]]

Perfecto, como ya es de costumbre, logramos realizar el error normal como siempre. Lo que vamos a hacer en este punto es lo siguiente: vamos a realizar la consulta al `/admin` para cargar la información:
![[Pasted image 20250919205410.png]]

Perfecto, si se recarga la web principal, veremos lo siguiente:
![[Pasted image 20250919205434.png]]

Ya logramos ver la página de admin. En este punto, vamos a sacar la solicitud con la cual vamos a eliminar el usuario `Carlos` para terminar con el lab:
![[Pasted image 20250919205544.png]]

Vemos la web principal a ver qué logramos:
![[Pasted image 20250919205626.png]]
Lab Terminado.

### Lab: HTTP request smuggling, basic CL.TE vulnerability

En este laboratorio, vamos a hacer que el back-end vea que estamos intentando emplear una petición GPOST.

Vamos a comenzar interceptando la solicitud y haciendo el típico tratamiento de la petición:

![[Pasted image 20250919213518.png]]

Perfecto, ahora vamos a modificarla para TE y vamos a hacer pruebas para ver si podemos generar algún error:
![[Pasted image 20250919213715.png]]

Vemos que sí podemos controlar los errores. En este punto, el back-end lee el TE, pero vamos a hacer una petición solo con una G:
![[Pasted image 20250919213845.png]]

Lo que hacemos es controlar parcialmente la solicitud. O sea, en la primera solicitud, el back-end lee el final de chunk y termina, pero cuando llega una segunda petición por POST, lo que sucede es que se concatena con la G que estamos mandando y provoca el método GPOST. Vamos a verlo:
![[Pasted image 20250919214056.png]]

Listo, producimos el método GPOST.
![[Pasted image 20250919214141.png]]

Lab Terminado.

### Lab: HTTP request smuggling, basic TE.CL vulnerability

Bueno, en este caso, el objetivo es el mismo, pero en este caso se encuentra invertido. Ahora el front-end interpreta TE y el back-end CL.

Vamos a comenzar tomando la captura y realizando el tratamiento básico:
![[Pasted image 20250919220145.png]]

En este caso, directamente ya está solucionado, no es algo que nos extrañe. Le pasamos para el front-end la solicitud por TE y luego generamos una nueva dentro de esta que, en el back-end, al contemplar CL, se interpretará, lo que provoca la segunda petición:
![[Pasted image 20250919220314.png]]

Lab Terminado.

### Lab: HTTP request smuggling, obfuscating the TE header

El objetivo es de por sí el mismo que las anteriores máquinas, donde tenemos que tramitar una cabecera como `GPOST`. Ahora el problema es que tanto el front-end como el back-end están empleando diferentes formas para repeler el uso de cabeceras repetidas.

Vamos a tomar la captura y a realizar el tratamiento básico:
![[Pasted image 20250919220852.png]]

Bueno, realizamos una prueba de una vez para ver qué interpreta el front-end, y si cerramos el chunk de forma inválida, vamos a ver lo siguiente:
![[Pasted image 20250919220949.png]]

Vemos que nos da un error de petición, lo que quiere decir que el front-end interpreta TE.

Ahora, vamos a intentar comprobar si el back-end comprende CL, y para esto vamos a causar una desincronización, esto inflando el CL:
![[Pasted image 20250919221231.png]]

Vemos que no pasa, entonces el back-end también es TE. Entonces, en este caso, vamos a intentar lo siguiente:
![[Pasted image 20250919221610.png]]

Bueno, logramos ahora sí la desincronización. Lo que sucede es lo siguiente:
-   Primero, entra al front-end y, por configuración, digamos, esto está configurado para que cuando se tenga una cabecera inválida, pero esta esté duplicada, coja la válida. Entonces, el front-end pasa al back-end.
-   Ahora, cuando llega al back-end, este lo interpreta diferente, y es que si ve la cabecera duplicada y, más que todo, es inválida, lo que hace en automático es leer por `Content-Length`.

El descubrir este tipo de cosas en un entorno real es algo complicado, pero para eso estamos, para pensar fuera de la caja y, mediante prueba y error, lograr encontrar una forma de encontrar una vulnerabilidad.

Bueno, ahora lo que vamos a hacer es pasarle una solicitud de la siguiente manera dentro del TE:
![[Pasted image 20250919222217.png]]

Perfecto, como dijimos, incrustamos una solicitud dentro de TE y lo que hacemos es, mediante el CL que se encuentra dentro y por el error que se produce en el back-end, se interpreta y logramos inyectar la petición:

![[Pasted image 20250919222331.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to perform web cache poisoning

En este laboratorio, al parecer, es envenenar la caché de la web, donde mediante esto tenemos que hacer un `document.cookie` dentro de una alerta. Tenemos a disposición el exploit-server, así que seguramente tendremos que almacenar el código en JavaScript y vamos a tener que ejecutarlo desde una web maliciosa.

Viendo que tenemos que efectuar código malicioso, lo primero que se nos ocurre es ver en el historial del Proxy al cargar la web qué solicitudes se hacen y encontramos una que nos puede servir:
![[Pasted image 20250919223059.png]]

Esta solicitud lo que hace es de forma constante cargar el recurso de `tracking.js` cuando cargamos la web. Además, si nos fijamos, tiene un `cache-control` donde define como `max-age=30`, lo que nos deja en claro que eso durará en caché almacenado por 30 segundos. Esto también lo podemos comprobar en el Repeater y enviando solicitudes continuas para ver cómo cambia y al llegar a 30 se reinicia.

Bueno, vamos a mantener esta petición, ya que vamos a tratar de emplear eso para cargar nuestro código malicioso:
![[Pasted image 20250919223657.png]]

En la descripción de la máquina, leímos que el front-end no contempla TE, por lo que debe ser CL. Es por eso que armamos la solicitud de esa manera. Probemos a ver si logramos provocar el error que queremos:
![[Pasted image 20250919223854.png]]

Logramos el error controlado, por lo que podemos continuar.

Bueno, ahora vamos a intentar aprovecharnos del exploit-server para ejecutar el código JavaScript. Revisando, nos encontramos con el siguiente botón dentro de los posts:
![[Pasted image 20250919224357.png]]

Vemos que apunta a lo mismo, pero al hacerle clic, vamos a ver cómo nos lleva al siguiente post, o sea, al 8:
![[Pasted image 20250919224445.png]]

El problema es que no vemos que se esté empleando alguna página con "Next Post" para efectuar esto, sino que es más como un "follow redirect", por lo cual vamos a capturar el link para ver qué hace:
![[Pasted image 20250919224633.png]]

Vemos que efectivamente se aplica el "follow redirect", pero está arrastrando la cabecera `Host`. Esto lo podemos controlar:
![[Pasted image 20250919224737.png]]

Vemos que nos dice "cabecera Duplicada". Esto es porque al incrustarse el resto de información de la petición a la que se sincroniza, esta cuenta con la cabecera `Host` y lo duplicaría. Para evitar esto, lo que podemos hacer es terminar la petición de forma que lo que se incruste sea en el body y ya no se interprete como cabecera. Nos quedaría de la siguiente manera:
![[Pasted image 20250919225019.png]]

Perfecto, ya evitamos el problema y en este punto, si nos fijamos en la respuesta, el `location` ya cambió, por lo que ya tenemos todo listo para reenviar a nuestra web maliciosa. Un concepto clave es que no importa que se esté usando un parámetro, en este caso a donde nos lleva es a `post`, por lo tanto, a nuestro exploit-server lo llamamos `post` y listo. Esto podemos hacerlo de la siguiente manera:

![[Pasted image 20250919225401.png]]

![[Pasted image 20250919225637.png]]

Perfecto, logramos incrustar el "follow redirect". Ahora, entender esto es clave: lo que sucede es que nosotros, con esta segunda petición que queda colgada, lo que sucede es que vamos al recurso de `tracking.js` que tenemos capturado a cargar luego de que termine el tiempo de caché, y luego de mandar nuestro smuggling, vamos a ver cómo se aplica para ese contenido el "follow redirect", ya que se sincroniza a ese recurso y nos carga en la caché el JavaScript malicioso:
![[Pasted image 20250919230138.png]]

Ahora, si revisamos:
![[Pasted image 20250919230151.png]]

Lab Terminado.

### Lab: Exploiting HTTP request smuggling to perform web cache deception

El objetivo es obtener la API key de otro usuario usando la caché.

El comienzo es el mismo que el del anterior lab, donde vamos a ver las solicitudes que se hacen al cargar la web, como es la de `tracking.js`:
![[Pasted image 20250919230938.png]]

De forma concreta, vamos a aprovechar eso para cargar y ver la API key. Dejémoslo en el Repeater listo para usarlo a su tiempo.

Vamos ya con esto a capturar la petición en la web como siempre:
![[Pasted image 20250919231310.png]]

Esto es bueno, ya que logramos el error controlado.

En este punto, primero vamos a entender qué está pasando al enviar esta petición, y lo que sucede es que como no estamos completando la petición hasta el body, todo lo que se inserte como cabeceras y eso se tomará como válido. Entonces, nosotros lo que queremos dejar en caché es la ruta `/my-admin`, ya que si esto lo dejamos en la cola y se sincroniza con la petición del `tracking.js` cargada por el usuario, al nosotros cargar esto, se arrastra su cookie de sesión y pasamos como el usuario admin. Vamos a intentarlo:

![[Pasted image 20250919232103.png]]

![[Pasted image 20250919232121.png]]

Vemos que se cacheó en ese recurso y nos carga como admin y tenemos su API key:

![[Pasted image 20250919232211.png]]

Lab Terminado.

### Lab: Bypassing access controls via HTTP/2 request tunnelling

En este caso, tenemos un front-end, pero el servidor hace downgrade a HTTP/2. Además, una de sus seguridades implementadas filtra por sus nombres. Para resolver ese laboratorio, tenemos que eliminar al usuario `Carlos`.

Además, todo lo que conocemos y hemos practicado por ahora no nos servirá de nada, ya que no es vulnerable a un HTTP Smuggling clásico. Pero sí es vulnerable a Request Tunneling.

Bueno, como siempre, comenzamos capturando una petición usual:
![[Pasted image 20250920182357.png]]

Perfecto, si recordamos lo que nos dice el lab es que no se validan correctamente los nombres de las cabeceras. Con esto en mente, podemos hacer lo de la siguiente forma:
![[Pasted image 20250920182837.png]]

Como vemos, vamos a agregar una cabecera, pero en la parte del mismo nombre le estamos asignando el valor. Además, estamos insertando una segunda cabecera `Host`, lo que debería darnos un error de cabecera duplicada:
![[Pasted image 20250920182859.png]]

Vemos que es inyectable, esto por el `test.com` que, si recordamos, es el valor que le definimos a la cabecera `Host`. En este punto, se nos pueden ocurrir algunas cosas como:
![[Pasted image 20250920183305.png]]

Vemos que estamos insertando una petición mediante la configuración de una nueva cabecera. Podemos intentar esto a ver qué sucede:
![[Pasted image 20250920184936.png]]

Insertamos, pero no logramos ver la respuesta de la segunda petición.

Algo que podemos tratar de hacer es cambiar el método que se está empleando de GET a HEAD, ya que muchas veces este tipo de peticiones en ocasiones nos logra mostrar los bytes que estamos consultando y los bytes que se están esperando:
![[Pasted image 20250920185337.png]]

Podemos observar cómo lo cambiamos y enviemos la solicitud a ver qué obtenemos:
![[Pasted image 20250920185904.png]]

Vemos que nos responde diciendo que se han recibido `3076` bytes, pero se esperaban `8866`. El porqué no vemos información es porque queda en espera de los bytes faltantes, pero como no llegan, pues no nos carga la web.

En este punto, podemos intentar apuntar a otros recursos como el `/admin`, esto para intentar hacer que los bytes recibidos sean iguales o mayores a los esperados:
![[Pasted image 20250920190208.png]]

![[Pasted image 20250920190232.png]]

Vemos que nos responde a las dos solicitudes como no autorizadas. Esto solo quiere decir que tenemos algo que queda pendiente. En ocasiones, el front-end envía ciertas cabeceras ocultas las cuales pueden ser esenciales para que la consulta sea correcta. Para esto, podemos aprovecharnos de la barra de búsqueda, ya que esta, si nos fijamos, inserta información:

![[Pasted image 20250920190446.png]]

Entonces, vamos a intentar enviar la solicitud en POST y vamos a aprovechar la inyección en el nombre de cabecera para definir un nuevo CL. Esto debido a que en ocasiones el back-end lo interpreta y nos permite filtrar por más datos de los que debería:
![[Pasted image 20250920190939.png]]

![[Pasted image 20250920191003.png]]

Excelente, ya estamos filtrando por más información. Esto es debido a que confunde al back-end, ya que no es una forma común de hacer la solicitud.

Bueno, vamos a aumentar el valor para lograr sacar toda la información posible:
![[Pasted image 20250920191338.png]]

Vemos algunas cosas. Vamos a verlo en raw para ver la información correcta:
![[Pasted image 20250920191439.png]]

Aparte de la cookie, vemos que tenemos 3 campos los cuales parecen ser esenciales que comienzan con `X`, así que vamos a copiarlos y a definirlos en el smuggling que teníamos:
![[Pasted image 20250920191654.png]]

Vemos que sigue con la respuesta `401`, pero vamos a seguir jugando con las cabeceras. E investigando un poco, damos con que en el de `client` se suele definir un cliente. Podemos intentar con `administrator`, que es el usuario administrador que conocemos:

![[Pasted image 20250920191830.png]]

Tampoco. Vemos que la verificación tiene un `0`. Podemos pasarle a `1` a ver qué sucede:
![[Pasted image 20250920191915.png]]

Logramos de forma correcta en la smuggled un `200 OK`. Podemos ver qué pasa si pedimos la solicitud de nuevo a la raíz:
![[Pasted image 20250920192234.png]]

Vemos de nuevo el error. Ahora tenemos que encontrar una web que cargue menos de 3608 bytes para que nos permita incrustar la información. Podemos intentar con el login:
![[Pasted image 20250920192347.png]]

Perfecto, ya tenemos los dos en `200 OK`, por lo tanto, si revisamos, ya tenemos la URL para eliminar el usuario `Carlos` y la usamos en el smuggling para poder ejecutarlo:
![[Pasted image 20250920192605.png]]

Bueno, tenemos de nuevo el problema, ya que nos tocaría buscar una consulta con una cantidad menor o igual a los 372 bytes para que funcione correctamente, pero pese al error en el lab, esto sí funciona:
![[Pasted image 20250920192711.png]]

Lab Terminado.

### Lab: Web cache poisoning via HTTP/2 request tunnelling

En este laboratorio, el propósito general es envenenar la caché de la web y tratar de que a otro usuario se le genere un `alert(1)`. El usuario víctima recarga la web principal cada 15 segundos. Además, no vamos a poder ejecutar los smuggling attacks que hemos hecho hasta el momento.

Vamos primero a capturar una consulta normal y vemos qué tenemos:
![[Pasted image 20250920195727.png]]

En este caso, vemos que tenemos lo que es el valor que se mantiene almacenado en la caché, que son 30 segundos. Nosotros, para no tardar tanto tiempo, vamos a implementar un parámetro que nos permite actualizar de forma rápida la caché sin esperar los 30 segundos:

![[Pasted image 20250920200028.png]]

Vemos que pasa el `age: 0`, lo que quiere decir que se actualiza la caché cuando le pasamos un parámetro.

Bueno, en este punto sabemos que hace un downgrade de HTTP/2 a HTTP/1.1. Podemos intentar lo mismo de la máquina anterior, que es inyectar las peticiones dentro del valor de la cabecera y luego dentro del nombre de la cabecera, pero esto no va a funcionar.

Entonces, en este punto, lo que vamos a hacer es comenzar a modificar el path de la siguiente manera:
![[Pasted image 20250920200805.png]]

Lo que estamos haciendo es modificar el HTTP/2 y desplazarlo para abajo, y para que esto no quede colgado, le estamos ingresando antes el `Test:`. El `/post` es una ruta válida, pero sin parámetro no lo es y nos debe indicar el parámetro que le hace falta. Veamos si esto le gusta:
![[Pasted image 20250920200931.png]]

Pues al parecer esto sí le gusta. Esto sería un punto de entrada interesante a tomar en cuenta. En este punto, lo que podemos hacer es intentar una inyección. Nunca olvidarnos de tratar de aislar la nueva petición con ayuda del `\r\nj`, quedando de la siguiente manera:
![[Pasted image 20250920201311.png]]

Un problema que se tuvo es que la caché queda almacenada y agregamos el parámetro `testing=1` para actualizarla de forma inmediata. Es cuestión de variar con eso para que se actualice, pero ya envía la petición. El problema es que no vemos una segunda respuesta, y esto puede que tenga que ver con los tamaños de nuevo. Si nos fijamos en la respuesta, esto tiene un total de 8689.

Para tener una mejor idea de lo que está pasando con los tamaños, podemos intentar jugar con el método HEAD:
![[Pasted image 20250920201557.png]]

En este caso, con la cabecera `HEAD`, podemos observar que en realidad nos retorna las dos peticiones, y si revisamos un poco la web, vamos a ver cómo en esta nos refleja como contenido la petición:
![[Pasted image 20250920202651.png]]

Con esto, podemos intentar enviar alguna cabecera extra a ver si esto se representa, pero no va a funcionar. Lo que se nos ocurre es buscar algún otro recurso que se llama de forma recurrente:

![[Pasted image 20250920202808.png]]

Encontramos este script en JS, el cual se llama de forma continua. En este punto, podemos intentar ver qué pasa si reducimos la URL:
![[Pasted image 20250920202903.png]]

Vemos que la petición nos responde con un "follow redirect", pero más importante, lo que le pasamos en la URL se refleja en la cabecera de `Location`. En este punto, lo que se puede hacer es intentar ver si podemos incrustar etiquetas HTML y ver si esto nos valida en el smuggling:

![[Pasted image 20250920203106.png]]

Vemos que no tenemos ninguna limitación. En este caso, vamos a intentar enviarlo en el smuggling a ver qué sucede:
![[Pasted image 20250920203314.png]]

Lo enviamos, pero vemos el error, y algo igual de importante es que cambiamos a HTTP/2, ya que si no, no se realiza la solicitud a ese recurso.

El problema ahora es encontrar algún recurso que nos retorne menos de `174` de tamaño. Va a ser algo complicado, pero nosotros podemos intentar rellenar esto con bytes. Vemos qué pasa si al final agregamos algunas `AAAAAA`. El propósito es que aumente el `174`:
![[Pasted image 20250920203753.png]]

Vemos que nos da un error de comunicación en este caso, y en otros casos no nos aumenta el valor, se queda en `174`. Puede ser que no nos esté interpretando. En este punto, lo que podemos hacer es ir cambiando de páginas. Por ejemplo, voy a intentar con los posts, pero ya voy a intentar con una carga útil de 8000 veces la A para ver si en alguna nos funciona:
![[Pasted image 20250920205348.png]]

Vemos que logramos inyectar en el post 2, y si ingresamos a este post, vamos a ver el `alert(1)`:
![[Pasted image 20250920205453.png]]

Bueno, el problema ahora es que lo está esperando en la página principal. Para que nuestro chunk de `A` supere eso, tendrían que ser algunas más, por lo que vamos a ingresar unas `8600`:
![[Pasted image 20250920205654.png]]

Perfecto, logramos ingresarlo en la raíz. Solo toca esperar y ver si se completa el lab:

![[Pasted image 20250920205727.png]]

Lab Terminado.

### Lab: Client-side desync

Tenemos que entender la diferencia entre el `Request Smuggling` y un `Client-Side Desync`:

> [!info] Request Smuggling vs. Client-Side Desync
> **HTTP Request Smuggling** ocurre cuando hay una desincronización en la interpretación de los límites de las solicitudes HTTP entre un front-end (proxy, balanceador de carga) y un back-end. El ataque se dirige al servidor, permitiendo al atacante manipular la cola de solicitudes del back-end.
>
> **Client-Side Desync** (o HTTP Desync Attacks) ocurre cuando la desincronización se produce entre el back-end y el cliente (navegador). El back-end puede responder prematuramente a una solicitud, dejando la conexión abierta y esperando más datos. La siguiente solicitud del cliente en esa misma conexión se interpreta como parte del cuerpo de la solicitud anterior, lo que puede llevar a la exposición de datos sensibles del cliente o a la manipulación de sus solicitudes. La clave es que el cliente reutiliza la conexión TCP, y el back-end ignora `Content-Length` o `Transfer-Encoding` en la primera parte de la solicitud.

Usualmente, en los Request Smuggling, se ocasiona en una sola solicitud donde, mediante el CL y el TE, comenzamos a definir parámetros los cuales nos permiten pasar por algún filtro donde llegamos a generar alguna discrepancia entre el back-end y el front-end, donde vemos la desincronización y logramos, del lado del back-end, ver cierto tipo de consultas que no son correctamente procesadas a final de cuentas, y esto genera los riesgos, donde la víctima principal es el servidor.

En el Client-Side Desync, como mínimo 2 o más solicitudes (parecido a cuando creamos grupos en Burp Suite que permite enviar dos solicitudes), y en este caso la discrepancia se produce entre el back-end y el cliente que reutiliza la conexión, donde la víctima principal es el cliente. El problema principal es que el Client-Side Desync ignora por completo el `Content-Length` y responde enseguida el back-end de forma inmediata, pero eso sí, dejando la solicitud abierta a nuevos bytes donde la siguiente solicitud que se tramite se interpretará como parte del cuerpo de la solicitud anterior. Nosotros, en este caso, solo controlamos la primera solicitud y todo lo demás depende de la otra persona, ya sea una víctima o un navegador.

![[Pasted image 20250920211515.png]]

Bueno, vamos a comenzar interceptando una petición normal del home a ver cómo se está tramitando y de paso haremos lo típico que es dejar la solicitud con las cabeceras mínimas:
![[Pasted image 20250920211815.png]]

Vamos a agregar una nueva petición y vamos a intentar colarla, pero vamos a ver que no logramos nada, ya que nos responde de forma inmediata la web, sin importar de que estemos inflando el `Content-Length`:

![[Pasted image 20250920212146.png]]

Bueno, esto es una forma de ver que estamos frente a un Client-Side Desync. Lo que vamos a hacer en este punto, ya que nos planteamos una hipótesis, es buscar una petición normal, por ejemplo, al redirect que se estaba aplicando a `/en`:
![[Pasted image 20250920212437.png]]

Bueno, usualmente, o lo que se hizo en otras máquinas, es enviar la primera solicitud maliciosa y luego enviar la otra. Esto provocaba que la primera solicitud que se quedaba colgada por falta de bytes se sincronizara a la segunda. En este caso, eso no pasa.

Para nosotros poder hacer esto de sincronizar una solicitud con la otra, lo que se tiene que hacer en el caso del `Client-Side Desync` es unificar todo en una misma conexión TCP (esto es esencial). Por lo tanto, lo que vamos a hacer es jugar con los grupos, o sea, crear un grupo con las dos solicitudes. Lo que vamos a hacer es configurar el envío como `Send group (Single Connection)`:

![[Pasted image 20250920212832.png]]

Las dos solicitudes están en el mismo grupo, y si ahora enviamos esto, el propósito es lograr ver el error en la otra solicitud:
![[Pasted image 20250920212919.png]]

Observamos que nos funciona.

Bueno, el objetivo es robarle a una víctima la cookie de sesión, y para esto vamos a hacer uso del Exploit-Server.

En este punto, lo que tenemos que hacer es automatizar esta petición que tiene que hacer la víctima, o sea, el recargar el `/en` para que funcione. Vemos que podemos subir posts, por lo que vamos a interceptar la solicitud de subida de uno para analizarla:
![[Pasted image 20250920213308.png]]

Esto como tal nos crea un post y podemos quitar algunas cosas que no nos sirven:
![[Pasted image 20250920213448.png]]

Vemos que se creó el post, así que sí funciona. El punto ahora es que esta solicitud mínima se realice en nuestra solicitud de atacante de la siguiente manera:
![[Pasted image 20250920213753.png]]

Omitamos la respuesta de la anterior solicitud, pero veamos que la otra solicitud que le estamos pasando es para que se haga un post con más información. En este punto, lo que vamos a hacer es enviar la solicitud en grupo y ver qué sucede:
![[Pasted image 20250920213937.png]]

![[Pasted image 20250920213952.png]]

Se va sin problema y vemos que se está incrustando la información de la solicitud siguiente. Podemos aumentar esto a 700 para que se vea lo más posible. Ahora, aparte de eso, tenemos que crear en nuestro exploit server un código en JavaScript que nos automatice esto, el cual quedará de la siguiente forma:
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

Listo, una vez que enviamos esto a la víctima, veremos que en el post se sube uno nuevo con información:
![[Pasted image 20250920221354.png]]

Perfecto, ya tenemos la cookie que necesitamos. Vamos a intentar ingresar:

![[Pasted image 20250920221454.png]]

Lab Terminado.

### Lab: Server-side pause-based request smuggling

En este laboratorio, nos vamos a encontrar que el servidor de front-end transmite la solicitud al back-end y el back-end no cierra la conexión después de un tiempo en determinados endpoints.
Para solucionar esta máquina, tenemos que identificar una pausa basada en el CL 0 desync vector, donde el objetivo es borrar al usuario Carlos:

Lo que vamos a hacer es recargar la web mientras pasa por Burp Suite para ver todas las solicitudes que se tramitan.

![[Pasted image 20250921103425.png]]

Vemos que tenemos múltiples solicitudes que se tramitan en la web donde el endpoint está bajo la ruta de `resource`. Podemos seleccionar el de `labHeader.js` y enviarlo al Repeater para analizarlo:
![[Pasted image 20250921103600.png]]

Lo que hacemos son pruebas donde, quitando parte de la ruta sin dejar ninguna barra colgando, vemos cómo se aplica un redirect. Esto funciona siempre sobre todo el recurso de `resources`. Vamos a dejarlo solo como `resources` y darle a la solicitud el tratamiento común que le hemos brindado en todos los demás labs:
![[Pasted image 20250921103911.png]]

Vemos que sí podemos hacer la petición de esa manera. Lo que podemos hacer es intentar una inyección común de la siguiente manera:
![[Pasted image 20250921104236.png]]

La verdad es que no sucede nada.

Lo que vamos a hacer ahora es usar una extensión de Burp Suite, la cual es Turbo Intruder. El propósito es segmentar las peticiones en tiempos de espera. Para entenderlo bien, vamos a enviar la primera parte, lo que está marcado en la imagen:
![[Pasted image 20250921104552.png]]

Y luego de pasar unos segundos, enviamos la segunda parte, esto claro en la misma solicitud.

Entonces, lo que sucede es que nosotros enviamos lo primero y lo que pasa es que el back-end se queda esperando más datos correspondientes al cuerpo de la solicitud, pero no le enviamos esto pasados unos 61 segundos.

Entonces, lo que sucede es que con lo primero, el back-end espera por datos aproximadamente los 61 segundos, y pasados este tiempo, como no recibió el cuerpo de datos que esperaba, en ocasiones el parser del back-end se refresca, por lo que no interpreta los datos entrantes como parte de la solicitud anterior, sino que interpreta la información como una nueva solicitud.

Entonces, para lograr este efecto, vamos a enviar esto al Turbo Intruder. No es más que clic derecho -> extensiones -> Turbo Intruder -> Send to Turbo Intruder. Vamos a enviarlo y veremos la siguiente interfaz:
![[Pasted image 20250921105403.png]]

Vamos a cambiar el `Last code used` por el `examples/default.py`:
![[Pasted image 20250921105502.png]]

Bueno, vamos a dejar el código que tenemos de la siguiente manera:
![[Pasted image 20250921105613.png]]

Ahora sí, vamos a ver cómo podemos segmentar la petición por tiempos.

Lo que vamos a hacer son pequeñas divisiones de código de la siguiente manera:
![[Pasted image 20250921105854.png]]

Bueno, eso sería de forma general el código, pero nosotros necesitamos manejar mejor esto, y es que vamos a cambiar múltiples veces la segunda petición y el `Content-Length` que está seleccionado. Es por eso que vamos a mejor dejarlos representados como `%s` de string en Python:
![[Pasted image 20250921110025.png]]

Bueno, ya con esto, vamos a crear nuestra smuggled request, que es nuestra segunda solicitud:
![[Pasted image 20250921110207.png]]

También configuramos una solicitud normal debido a que tenemos que luego enviar una solicitud usual a la cual se sincronizará la smuggled request.

![[Pasted image 20250921110453.png]]

Se está empleando un concepto de Python mediante `engine.queue()` que nos permite unificar la request completa, donde le pasamos `attacker_request`, la cual es la petición con los `%s` para ingresar los strings donde, gracias al `engine.queue()` y pasándolo como una lista, vamos a poder incrustar la información donde le pasamos primero `len(smuggled_request)` que es el tamaño de la cadena (lo primero que se incrusta es el valor de `Content-Length`) y luego le pasamos `smuggled_request` tal cual, ya que tenemos que incrustar la smuggled.

Ahora, para segmentar esto por tiempo, hacemos lo siguiente:
![[Pasted image 20250921111131.png]]

Definimos luego lo que es `pauseMarker`, el cual nos permite definir un delimitador en donde tenemos que pasarle como lista el `\r\n\r\nGET`, que es la última parte de nuestra petición principal:
![[Pasted image 20250921111050.png]]
Ahí entendemos el porqué de la cadena, y luego definimos un `pauseTime` en el cual vamos a definir el tiempo en milisegundos, donde recordemos que `1 segundo = 1000 milisegundos`. Entonces, como tenemos que esperar 61 segundos, definimos 61000 milisegundos. Esto del tiempo ya se probó con anterioridad hasta dar con el correcto, pero si no sabemos o no nos lo dicen o es en un entorno real, tenemos que probar hasta encontrar el tiempo exacto.

Entonces, lo que esa línea nos dice es que vamos a enviar la petición, pero al encontrar el `\r\n\r\nGET`, vas a pausar y a esperar 61 segundos para enviar lo demás. El objetivo es que el back-end nos interprete estos datos como una nueva consulta y esto permite colarle una consulta nueva.

Entonces, vamos a enviar la solicitud normal de la siguiente forma:
![[Pasted image 20250921115605.png]]

Vemos que es lo mismo `engine.queue()` y en este caso la `normal_request`. Ahora le damos al botón de "attack" y veamos qué es lo que sale:
![[Pasted image 20250921115625.png]]

Luego vemos lo siguiente:
![[Pasted image 20250921113857.png]]

Eso es el contador donde nos indica que se envió la primera parte y al llegar a 61 se enviará la segunda y luego debería enviarse la normal. Lo que obtenemos es lo siguiente:
![[Pasted image 20250921114618.png]]

Vemos la solicitud smuggled primero y se envió sin problemas, pero si revisamos la solicitud usual a la raíz, vamos a ver lo siguiente:

![[Pasted image 20250921114719.png]]

Como podemos observar, se realizaron las solicitudes a la raíz, pero nos responde lo de la solicitud smuggled, por lo tanto, esto está funcionando.

Ahora, con esto, ya podemos probar qué tenemos en `/admin`:

![[Pasted image 20250921115656.png]]

Veamos qué obtenemos:
![[Pasted image 20250921121145.png]]

Ahora vemos que llegamos a la ruta, pero nos dice que el panel de admin solo se puede acceder de forma local. Esto ya lo hemos hecho, es cuestión de modificar la cabecera `Host` para ver qué sucede:

![[Pasted image 20250921121314.png]]

![[Pasted image 20250921121540.png]]

Perfecto, ahora ya tenemos el panel y podemos eliminar un usuario, pero lo que tenemos que hacer es automatizar esto. Para eso, me voy a ver el código y a copiar solo la parte del formulario:
![[Pasted image 20250921121702.png]]

Vemos un campo `csrf` donde podemos completar la petición de la siguiente manera:
![[Pasted image 20250921123622.png]]

Vemos todo lo que se modificó. Esto enviamos y solo nos queda esperar para ver si el lab se completa:

![[Pasted image 20250921124344.png]]

Lab Terminado.