-------------------

El envenenamiento de caché web es un ciberataque en el que un atacante manipula una caché web para que almacene una respuesta HTTP maliciosa, que luego se distribuye a otros usuarios. Al recibir este contenido envenenado, los usuarios pueden sufrir diversos ataques, como el robo de información o el secuestro de sesiones, a través de vulnerabilidades como Cross-Site Scripting (XSS)

Otro concepto clave es la clave caché (cache key) el cual para comprender podemos verlo como el nombre que se usa tanto para guardar como para recuperar una respuesta en la cache.
## PortSwigger

### Lab: Web cache poisoning with an unkeyed header

Lo que tenemos hacer en este laboratorio es mediante una cabecera la cual la cache no interpreta para generar el cache key, inyectar un alert(0) para que se ejecute en otros usuario que visiten el home.

Como es en la pagina principal lo que vamos a hacer es directamente capturar la petición a la web principal:
![[Pasted image 20250926094028.png]]

vemos en la respuesta algo interesante, que son las cabeceras de `Cache-Control` y `Age` donde la una nos indica el tiempo máximo que permanece la petición en la cache y la otra nos indica el tiempo estimado que va esa respuesta en la cache.

Podemos intentar poner parámetros para ver si la cabecera `Age` regresa a 0 ya que ese es un indicativo de que la cache key se esta refrescando:

![[Pasted image 20250926094327.png]]

Vemos que la cabecera `Age` cambio a 0 por lo que se esta tomando el parámetro como parte de el cache key, el objetivo era encontrar una cabecera la cual permita inyectar código malicioso y no sea detectada en el web cache para que se cargue en la pagina principal.

Para buscar la cabecera como directamente nos dicen que es una cabecera podemos usar una extension del BurpSuite llamada `Param Miner`:

![[Pasted image 20250926094658.png]]

vamos ahora al repeater y podemos hacer usar el Param Miner para adivinar cabeceras (guess headers):
![[Pasted image 20250926094824.png]]

vamos a ver el siguiente panel:
![[Pasted image 20250926094845.png]]

y vamos a darle al ok, para luego irnos a `All Issues`:
![[Pasted image 20250926094940.png]]

![[Pasted image 20250926095023.png]]

vemos que al parecer nos reporta un cache poisoning donde usando la cabecera `x-forwarded-host` podemos envenenar la cache, podemos intentarlo:

![[Pasted image 20250926095158.png]]

vemos que esto funciona y se esta poniendo en la respuesta, en este punto cualquier usuario que haga una consulta a la raíz va a solicitar un script al parecer de la ruta que le pasemos, con esto podemos aprovechar el exploit server e intentar la alerta de la siguiente manera:

![[Pasted image 20250926095427.png]]

usamos el dominio y modificamos para que sea igual al resto de la petición y enviamos el alert(0) en este caso. Vamos en este punto a intentar modificar la cache a ver que sucede:

![[Pasted image 20250926095533.png]]

funciono se cambio, y si recargamos la web desde el navegador veremos la inyección en acción:
![[Pasted image 20250926095724.png]]

perfecto ahora vemos si con esto se resuelve el lab:

![[Pasted image 20250926095811.png]]

Lab Terminado.

### Lab: Web cache poisoning with an unkeyed cookie

En este laboratorio el objetivo sigue siendo enviar un `alert()` envenenando el cache.

Vamos a capturar la petición de la pagina principal a ver que esta pasando:
![[Pasted image 20250926100042.png]]

Bueno al parecer tenemos que envenenar la cache mediante la cookie, en especifico el de `fehost`, vamos a ver si logramos ingresar algo allí:
![[Pasted image 20250926100208.png]]

vemos que si, al parecer estamos dentro de una tupla y podemos insertar cositas.

En este caso vamos a intentar cerrar alguna coas para escribir directamente dentro de la etiqueta script:
![[Pasted image 20250926100604.png]]

vemos que estamos cerrando la `"};` y luego insertamos el alert para al final comentar lo que queda colgado, veamos si funciono:

![[Pasted image 20250926100653.png]]

Lab Terminado.


### Lab: Web cache poisoning with multiple headers

En este caso el objetivo es el mimos, ejecutar un alert pero en esta ocasión lo que vamos a tener es multiples cabeceras.

Vamos a comenzar capturando la petición de la web principal a ver como se tramita:

![[Pasted image 20250926101231.png]]

Vamos a volver a emplear la extension de Param Miner, con el objetivo de encontrar cabeceras vulnerables:

![[Pasted image 20250926101335.png]]

Nos reporta la cabecera `x-forwarded-scheme`, de forma rápida esta cabecera nos permite definir el protocolo con el cual se realiza la conexión ya sea `HTTP o HTTPS` lo que vamos a hacer en este punto es intentar usarla con HTTP a ver que hace:

![[Pasted image 20250926101815.png]]

vemos que nos realiza un redirect a la raíz haciendo el cambio a https, ahora debemos buscar una forma la cual nos permita en este redirect alterarlo, podemos volver a emplear el Param Miner a ver que nos reporta:

![[Pasted image 20250926102002.png]]
vemos que ahora nos reporta el `x-forwarded-host` el cual puede que controle el dominio del redirect, veamos:

![[Pasted image 20250926102116.png]]

Perfecto esto controla ya el host, entonces lo que podemos hacer es buscar consultas alternas que se hagan al realizar la petición a la pagina principal:

![[Pasted image 20250926102346.png]]

encontramos que se realiza esa petición al tracking.js y si lo que hicimos anterior mente aplica a cualquier parte de la web podemos intentar que esto cambie y se redirija a una petición maliciosa:

![[Pasted image 20250926102502.png]]

Perfecto lo que hacemos es que en la cache se guarde ese redirect cada vez que se invoque al recurso y ese redirect ahora vamos a controlarlo para que llame a un recurso malicioso:

![[Pasted image 20250926102746.png]]

vemos el exploit malicioso lo configuramos perfectamente y ahora vamos con la petición:
![[Pasted image 20250926102818.png]]

En este caso configuramos y enviamos correctamente, vemos si se completa el laboratorio:

![[Pasted image 20250926103322.png]]

Perfecto, ahora podemos ver si se completa el lab:

![[Pasted image 20250926103549.png]]

Lab Terminado.


### Lab: Targeted web cache poisoning using an unknown header

En este laboratorio tenemos el mismo objetivo pero nos hablan de envenenar la cache web con una cabecera desconocida, vamos a ver que podemos hacer.

Vamos a comenzar capturando la petición de la web principal a ver que encontramos:

![[Pasted image 20250926105152.png]]

Bueno eso tenemos y lo que podemos hacer es volver a emplear la extension de Param Miner a ver si encontramos algo:

![[Pasted image 20250926105817.png]]

vemos que nos detecta la cabecera `X-host`, podemos intentar a ver que nos edita o nos inserta en la respuesta:

![[Pasted image 20250926110001.png]]

vemos que mediante esa cabecera logramos cambiar el dominio de una solicitud como ya lo hemos visto antes:

![[Pasted image 20250926110427.png]]]
![[Pasted image 20250926110444.png]]

Perfecto se queda en la cache y si revisamos:

![[Pasted image 20250926110558.png]]

perfecto veamos si con esto logramos completar la maquina.

Pues no se completa, si revisamos un poco mas de cerca la respuesta vemos una llamada `vary`:

![[Pasted image 20250926110821.png]]

esta cabecera en especifico lo que le dice a la web es que todo lo almacenado en la cache se lo cargue solo a los usuario con el user agent exacto que se envío como el nuestro:

![[Pasted image 20250926110928.png]]

lo que hace de forma general es que solo aplique para mi en este caso.

Lo optimo en este tipo de casos es tratar de por otros medios obtener el user agent con el cual la victima tramita sus peticiones, con esto en mente vemos en la descripción que los usuarios de la web siempre visitan los post, vamos a ver los post:

![[Pasted image 20250926111306.png]]

vemos que html es valido, podemos intentar cargar una imagen hacia nuestro servidor para ver el user agent:

![[Pasted image 20250926111412.png]]

![[Pasted image 20250926111440.png]]

perfecto, vemos nosotros el `user-agent` especifico para esta web, lo que podemos hacer es pasarle ese para que sea valido para la victima:

![[Pasted image 20250926111555.png]]

y si esperamos vamos a ver que ahora si:
![[Pasted image 20250926111607.png]]

Lab Terminado.


### Lab: Web cache poisoning via an unkeyed query string

Este laboratorio es vulnerable a un envenenamiento de la web cache porque tenemos una cadena como query  que excluye de la cache key y el objetivo es enviar la alerta.

Como siempre vamos a comenzar capturando la petición de la pagina principal:

![[Pasted image 20250926112233.png]]

perfecto, podemos intentar con algún parámetro de prueba a ver si conseguimos que se genera una nueva cache key o que no que seria lo ideal:

![[Pasted image 20250926112409.png]]

vemos que se inyecta el parámetro pero si intentamos enviar una nueva solicitud a la web principal:

![[Pasted image 20250926112618.png]]

vemos que la cache es la misma lo que quiere decir que no esta creando una nueva cache key usando el parámetro, sino que simplemente esta usando la misma raíz sin importar el parámetro que se le pase, en este punto podemos intentar una XSS:
![[Pasted image 20250926113113.png]]

vemos que estamos completando la solicitud saliendo del link y ingresando una etiqueta img para generar el alert(0):

![[Pasted image 20250926113153.png]]

perfecto, veamos si se completa el laboratorio:

![[Pasted image 20250926113217.png]]

Lab Terminado.


### Lab: Web cache poisoning via an unkeyed query parameter

Este laboratorio requiere los mismo que es ejecutar un XSS con un alert(1) en la victima y en este caso se excluyen de la cache key algunos parámetros.


Vamos a comenzar de forma habitual interceptando una petición de la web principal para probar:

![[Pasted image 20250926151142.png]]

si realizamos algo parecido que en el lab anterior vamos a ver como la cache key cambia debido a que a los parámetros también los interpreta:

![[Pasted image 20250926151258.png]]

vemos que pasa directo a 0 por lo que en este caso no podemos hacer lo mismo que en el anterior laboratorio.

En este caso podemos usar la extension de Param Miner para ver si puede encontrar algún parámetro que no se interprete:

![[Pasted image 20250926151502.png]]

Vamos a esperar a que nos responda y veremos lo siguiente:
![[Pasted image 20250926151724.png]]

vemos que nos detecta el parámetro `utm_content`, vamos a intentar a ver que nos responde:

![[Pasted image 20250926151940.png]]

perfecto esto funciona y como vemos en la respuesta también se inserta, vamos a intentar hacer la inyección XSS a ver que sucede:

![[Pasted image 20250926152217.png]]

perfecto el como se inserta el código, en este punto lo que vamos a hacer es ver que sucedió en la web y si se completo:

![[Pasted image 20250926152251.png]]

Lab Terminado.


### Lab: Parameter cloaking

Bueno en parameter cloaking es una técnica dentro del web cache donde ocultamos los parámetros maliciosos dentro de la solicitud donde el servidor los ignora pero el web cache los toma como parte única del recurso y almacena la respuesta generada.

Bueno en el lab tenemos parámetros en la web y tenemos un parámetro que genera esa inconsistencia entre servidor y el web cache, tenemos que ejecutar el alert.

Como ya es costumbre vamos a capturar la petición para ver como se tramita:
![[Pasted image 20250926152741.png]]

Vemos algo que llama algo la atención y es esa cookie de `country`, bueno dejando de lado eso lo que vamos a hacer es usar la extensión param Miner:

![[Pasted image 20250926153154.png]]

vemos que tenemos el parámetro `utm_content` lo que podemos hacer es intentarlo a ver que sucede:
![[Pasted image 20250926153248.png]]

perfecto vemos que esto no lo toma y recarga sin problemas, al parecer en este caso se puede aplicar parecido pero vamos a centrarnos en la cookie que tanto llama la atención.

Si nos fijamos en la respuesta se llama a un recurso externos de geolocate, vamos a tratar de capturarlo y enviarlo a repeater a ver de que se trata:

![[Pasted image 20250926153502.png]]

vemos que tenemos un parámetro en especial que se repite en la respuesta, podemos intentar cambiarlo por el momento a ver si este cambia en la respuesta también:

![[Pasted image 20250926153547.png]]

si cambia y esto es código Java Script por lo que podemos intentar un ataque, el problema radica que por defecto llama a `setCountryCookie`, el cual es un parámetro por defecto y para que funcione tenemos que inyectar el código sin que eso cambie.

Recordemos que tenemos un parámetro que encontramos inyectable `utm_content`, vamos a intentar ponerlo a ver que sucede:
![[Pasted image 20250926153937.png]]

Bueno vemos que si nos lo toma como valido ya que no se recarga pero no vemos que se pueda cambiar el valor.

Un concepto clave es que en algunas ocasiones cuando nosotros enviamos un ; el parser interpreta todo lo que esta después de este `;` como un nuevo parámetro donde podemos intentar cambiar el valor de `callback`:

![[Pasted image 20250926154141.png]]

vemos que se cambia la información, con esto valido podemos intentar una inyección de la siguiente manera:
![[Pasted image 20250926154427.png]]

Perfecto se inyecto, podemos ver en la web que pasa:

![[Pasted image 20250926154443.png]]

Perfecto, veamos si se completo el lab:

![[Pasted image 20250926154459.png]]

Lab Terminado.


### Lab: Web cache poisoning via a fat GET request

En esta maquina vamos a tener que inyectar un alert donde vamos a tener que enviar una solicitud por GET inflada donde tenga mas información que la que deberia.


Vamos a comenzar como siempre capturando la petición:
![[Pasted image 20250926155020.png]]

perfecto vemos que se tramita de forma usual. Podemos ver también que al igual que en el lab anterior tenemos el `geolocate` que se llama:
![[Pasted image 20250926155135.png]]

ahora recordando que la web cache no toma en cuenta el body para el cache key y que podemos por get mandar body podemos intentar lo siguiente:

![[Pasted image 20250926155254.png]]

En este punto lo que tenemos es que podemos cambiar el parámetro desde el cuerpo y es mas esto se queda en cache por lo que vamos a realizar de forma rápida la inyección:

![[Pasted image 20250926155419.png]]

![[Pasted image 20250926155428.png]]

perfecto veamos si el lab se completa:

![[Pasted image 20250926155451.png]]

Lab Terminado.

### Lab: URL normalization

Este laboratorio tiene una vulnerabilidad que no es directamente explotable XSS debido a la codificación url que se aplica en el navegador pero vamos a aprovecharnos del proceso de la normalización de la cache para lograr inyectar y lanzar un alert(1).


Como es de costumbre vamos a analizar la petición de la web principal para ver:

![[Pasted image 20250926155924.png]]

bueno mucho no es que veamos que podamos hacer desde aquí, pero bueno en este caso es enviarle un link a la victima por lo que podemos generar algún error de la siguiente forma:
![[Pasted image 20250926160223.png]]]

vemos esto y se ve algo peligroso podemos intentar cerrar el párrafo e insertar una inyección con etiquetas script a ver que sucede:

![[Pasted image 20250926160325.png]]

esto es algo bueno porque se inserta, podemos intentar enviar esto en el navegador para probar primero con nosotros:

![[Pasted image 20250926160406.png]]

vemos como se url-encode la información y esto se almacena también en el cache por lo que si recargamos el Burp pasa igual:

![[Pasted image 20250926160551.png]]

perfecto esto nos quita el factor de inyección pero que pasa si desde el BurpSuite que no nos lo url code lo ingresamos a la cache y luego enviamos el link a la victima:

![[Pasted image 20250926160649.png]]

![[Pasted image 20250926160732.png]]
Esto funciona, vamos a enviarlo al cache y luego a enviar el link:

![[Pasted image 20250926160815.png]]

Lab Terminada.


### Lab: Web cache poisoning to exploit a DOM vulnerability via a cache with strict cacheability criteria

Bueno en este laboratorio vamos a explotar una vulnerabilidad donde la cache que se emplea en el servidor tiene de un criterio estricto de cara a ver que respuesta cachea y cual no, en este al igual que en otros tenemos que ejecutar un alert.

Como ya es costumbre capturamos la petición de la web principal a ver con que nos encontramos:

![[Pasted image 20250926161753.png]]

Vemos que tenemos las etiquetas script donde almacena el host y el path, esto debe usarlo en algún lado, vamos a buscar:

![[Pasted image 20250926162102.png]]

vemos que en efecto llama a una ruta donde usa el host para y lo concatena con esa ruta, podemos intentar con cabeceras que ya hemos usado como son `X-Forwarded-Host` para intentar ver si modificamos el host:

![[Pasted image 20250926162455.png]]

Perfecto vemos que si logramos cambiar el host y esto combinado con la carga de información que hace luego podemos configurar en nuestro exploit externo a ver que podemos hacer, algo a tener en cuenta es primero ver que es lo hace ese script por lo que podemos intentar verlo:

![[Pasted image 20250926162623.png]]

exactamente devuelve eso, ahora si revezamos nuestra web esto aparece también y aquí es donde se da el vector del ataque:

![[Pasted image 20250926162707.png]]

Perfecto con eso claro vamos a configurar nuestro exploit server:

![[Pasted image 20250926162820.png]]

esto solo es una prueba, es por eso que no inyectamos nada aun, podemos ya cambiar el host de la petición al de nuestro exploit server para ver como es que funciona:

![[Pasted image 20250926163003.png]]

esto es perfecto puede ser que simplemente no se este interpretando al no tener el formato JSON, vamos a intentar devolver un Json primero para que se interprete:

![[Pasted image 20250926163328.png]]

![[Pasted image 20250926163512.png]]

vemos que no carga nada y si revisamos en la consola vemos que nos dice que falta la cabecera de `Access-Control-Allow-Origin` la cual era una cabecera de respuesta que determinada los orígenes que permiten o pueden acceder a un recurso web.

Esto se debe a que en nuestro exploit server por configuración puede que no pueda ser accedido y para esto podemos hacer lo siguiente:

![[Pasted image 20250926163754.png]]

Como vemos agregamos la cabecera y esto debería permitirle cargar desde otros dominios, vemos si funciona:

![[Pasted image 20250926163905.png]]

vemos que si se inserta, ya con esto podemos configurar para que se realize una inyección de la siguiente manera:

![[Pasted image 20250926164003.png]]

![[Pasted image 20250926164328.png]]

Vemos si con esto completamos el lab:

![[Pasted image 20250926164354.png]]

perfecto vemos que se ejecuta:

![[Pasted image 20250926164408.png]]

Lab Terminado.

### Lab: Combining web cache poisoning vulnerabilities

Bueno este laboratorio vamos a tener varios tipos de ataques donde vamos a tener que organizarnos correctamente para poder lograr completarlo, el objetivo como siempre es inyectar una alerta.


Vamos a capturar por todo el trafico de la web para ver que obtenemos:
![[Pasted image 20250926192921.png]]
vemos esta petición que parece ser una un translate, vamos a ver que nos retorna esto:

![[Pasted image 20250926193008.png]]

vemos que tenemos la respuesta de todos los idiomas y diferente información, vamos a irnos a la web y vamos a seleccionar un idioma a ver que sucede:

![[Pasted image 20250926193105.png]]

vemos que nomas se tramito:
![[Pasted image 20250926193157.png]]

vemos que lo primero que se tramita es un  `setlang/es`, donde observamos que define de alguna forma el idioma que elegimos, esto realiza un redirect y listo.

Bueno entonces con esto la cadena de como vamos a tratar de atacar es que nosotros tengamos cacheada una inyección dentro del `translations.js` donde luego intentemos hacer un redirect al idioma español y esto se inserte con el objetivo de lograr un XSS.

bueno vamos a tener todo listo, lo que es primero el translation:

![[Pasted image 20250926230211.png]]

Vamos a intentar mediante nuestra extension de Param Miner descubrir alguna cabecera que nos permita inyectar información:

![[Pasted image 20250926230500.png]]

vemos que tenemos una cabecera de `x-original-url`, veamos que generamos con esto:
![[Pasted image 20250926230732.png]]

vemos que esto nos genera un not found, lo que quiere decir que esto nos puede estar intentando ingresar a `/test`.

Esto del todo no nos sirve del todo ya que queremos alterar la respuesta, podemos tratar de hacer es volver a analizar la petición a la raíz:
![[Pasted image 20250926231116.png]]

la petición es igual a la del lab anterior, podemos intentar la cabecera del `X-Forwarded-Hos` a ver si logramos cambiarlo:
![[Pasted image 20250926231404.png]]

Esto es perfecto, logramos cambiar esto y como esto llama al script de `translation.json`, ya tenemos una forma de modificar el contenido si lo llamamos desde nuestro exploit, vamos a configurar esto a ver que sucede:

![[Pasted image 20250926231833.png]]

ya tenemos el exploit server, ahora vamos a cachear la respuesta para que esto suceda:

![[Pasted image 20250926231939.png]]

bueno el proceso es el siguiente, enviar esto a la cache, luego que se cachee lo que podemos hacer es intentar recargar la web en español a ver que sucede, para sto necesitamos de parámetros extras como eremos en la url:

![[Pasted image 20250926232846.png]]

de igual vemos el problema en el exploit server a ver si ya funciona:

![[Pasted image 20250926232952.png]]

perfecto ahora vamos a ver si ya funciona:

![[Pasted image 20250926233158.png]]

si funciona, pero la inyección no se aplico de forma correcta, vamos a intentar de otra forma:

![[Pasted image 20250926233326.png]]

vamos a ver ahora que sucede:

![[Pasted image 20250926233408.png]]

perfecto ya funciona, ahora nuestro objetivo es ver como podemos redireccionar la web hacia la web en español para que se aplique.

si recordamos tenemos una cabecera que realiza una redirección `x-origin-url`, podemos intentar ver si esto funciona:

![[Pasted image 20250926235005.png]]

vemos que si logramos redirigir la web, en este caso vamos a poner es para que vaya a español y vamos a cachear las dos de forma continua con el objetivo de que cuando la victima ingrese al home todo se ejecute:

![[Pasted image 20250927000041.png]]

Lab Terminado.


### Lab: Cache key injection

Al igual que en el laboratorio anterior se va a explotar una cadena de vulnerabilidades mediante las cuales vamos a llegar a ejecutar un `alert(1)` en el navegador de la victima.


![[Pasted image 20250928124331.png]]

Bueno comenzamos analizando las peticiones donde vamos a darnos cuenta que la primera a la raíz `/` es un redirect y la seleccionada en la imagen es otro redirect entonces por el momento donde nosotros podemos manipular es los puntos donde se generan estas redirecciones y para analizar podemos intentar ver la petición a `/login?lang=en` que es donde se aplica la segunda y ultima redirección.

Vamos ahora a ver como la pagina a la cual nos redirige ya carga el contenido de la web seria la siguiente petición al ultimo redirect.

![[Pasted image 20250928124731.png]]

bueno vemos que al hacerse la petición ese parámetro se esta aplicando directamente en la respuesta, podemos intentar varias cosas pero vamos a darnos cuenta que como se realiza un url-encode al insertarse, esto no nos sirve de absolutamente nada por el momento.

![[Pasted image 20250928125111.png]]

Continuando con la búsqueda vamos a encontrarnos que carga un recurso dentro de un script, podemos intentar ver como se tramita y el resultado de esa consulta:

![[Pasted image 20250928125230.png]]

cosas a tomar en cuenta como que esta petición se queda en cache, vemos a demás que lo de `lang=en` se define en la petición y se inserta en la respuesta, podemos intentar modificar esto a ver si podemos alterar de alguna forma la respuesta:

![[Pasted image 20250928125519.png]]

Vemos que si se altera, bueno podemos intentar enviar la petición que se nos paso en la lab para intentar ver que podemos obtener de datos extras:

![[Pasted image 20250928125801.png]]

Podemos observar como esta cabecera de alguna forma nos acaba de permitir el ver la clave cache con la que se esta almacenando, además de que si nos ponemos minuciosos vamos a ver que al final de la misma se pasa un `$$` donde puede ser que sea alguna forma de dar un final a la cadena.

![[Pasted image 20250928130048.png]]
Bueno regresamos a esta solicitud ya que nuestro objetivo es encontrar una forma que nos permita inferir en este link.

Cosas que de primeras podemos intentar desde la misma url es pasarle el parámetro `lang=test` para ver si logramos modificar su respuesta:

![[Pasted image 20250928130451.png]]

veamos que obtenemos:

![[Pasted image 20250928130746.png]]

vemos que si se modifico el contenido, esto es interesante y un posible vector de ataque donde si nosotros logramos cargar el `lang=test` esto también viajara a la otra petición como `lang-test` donde se vería algo como lo siguiente:
![[Pasted image 20250928135317.png]]

esto es mas o menos lo que tendríamos, ahora bien en esta petición si tratamos de enviar mas parámetros, vamos a ver que no nos lo incorpora. Bueno como el objetivo es intentar alterar la respuesta, cosas que tenemos que tener en cuenta son el `vary: origin` el cual se encarga de guardar y ejecutar la cache solo para objetivos del mismo origen con el mismo `User-Agent`.

Podemos intentar jugar con la cabecera `origin` a ver si logramos algún resultado:

![[Pasted image 20250928135720.png]]

Podemos observar que con esta cabecera pasa a ser parte del cache key y algo que vemos es que mediante los `$$` se esta separando algo extraño. A mas de eso no logramos observar nada mas. Algo que podemos intentar es jugar con los valores de ese parámetro de `cors` a ver que logramos:

![[Pasted image 20250928140739.png]]

esto es bueno ya que logramos hacer que se considere, ahora algo que podemos hacer es un `header inyection`, el propósito es intentar inyectar una nueva cabecera, y lo vamos a hacer de la siguiente manera:

![[Pasted image 20250928141240.png]]

vemos que si estamos inyectando cabeceras, esto es mucho mejor, ya que podemos intentar a partir del `content-length` sobre escribir la información de la respuesta de la siguiente manera:

![[Pasted image 20250928141646.png]]

Perfecto, esto es bastante bueno pero el problema que tenemos actualmente es el como se tiene que consultar a nivel de clave cache ya que como vemos esa clave es muy especifica para un ataque pero nosotros podemos intentar hacer que esto suceda ya que recordemos que el como se consulta a este documento lo conocemos y el propósito será que se represente en la primera solicitud para que se consulte en forma de ataque, ahora nuestro problema actual es que si intentamos ingresar parámetros extra en nuestra solicitud no se tomaran en cuenta:

![[Pasted image 20250928142358.png]]

vemos que no lo logramos, podemos intentar con nuestra querida extensión encontrar algún parámetro inyectable:

![[Pasted image 20250928142553.png]]

vemos que mediante el parámetro de `utm_content` ignora la cache así que vamos a intentar:
![[Pasted image 20250928142833.png]]

aunque efectivamente si ignora la cache no vemos que se esta implementando información.

Bueno mediante la siguiente inyección logramos redirigir y forzar el código en la web:

![[Pasted image 20250928151542.png]]

Ahora si vemos como se esta consultando por detrás la otra solicitud vamos a ver lo siguiente:

![[Pasted image 20250928151629.png]]

Ahora tenemos un pequeño problema el como se tramita esto tiene un objetivo y este es que nosotros almacenemos en la cache un código maliciosos el cual contenga el mismo nombre de cache key por lo que problema es que si vemos los dos cache key vamos a observar que tiene cierta diferencia:

![[Pasted image 20250928152233.png]]

nos centramos específicamente en que nos faltan los  `$$` pero esto podemos intentar solo agregarlo y las dos solicitudes quedarían de la siguiente manera:

![[Pasted image 20250928152453.png]]

y el como se tramitaría mediante la url:

![[Pasted image 20250928152522.png]]

Perfecto vemos que el cache ya son lo mismo y vería esto ejecutarse, vamos a que si lo logramos cacheando la solicitud url y la petición:

![[Pasted image 20250928153001.png]]

perfecto revisemos si esto completo el lab:

![[Pasted image 20250928153031.png]]

A rasgos generales para dejar todo claro lo que se hiso fue generar una petición con valores exactos los cuales por detrás generan otra solicitud con valores exactos generando allí una key cache la cual nosotros mediante una petición alterna que genera el mismo key cache pero es la inyectable hacemos que se quede cacheada y cuando entra al home y genera esta segunda solicitud lo que hace es agarrar el cache infectado y ejecutarlo generando el XSS con el alert.

Lab Terminado.


### Lab: Internal cache poisoning

Tenemos que envenenar en esta maquina la cache interna, el objetivo será ejecutar una alerta.

Bueno vamos a comenzar capturando una consulta de la raíz:

![[{F68BDB58-E8BB-4E96-9F77-9DFA375F4E2B}.png]]

mediante algunas revisiones vamos a ver que el link canonico no nos da paso a la ejecución de inyecciones maliciosas, vamos a ver con nuestra extensión a ver que encontramos:

![[{280C8418-096B-4353-9290-BC8FD792C0FC}.png]]

vemos que en este caso que podemos implementar la cabecera de `x-forwarded-host`, vamos a intentarlo:

![[{BEEE3D35-2493-43EC-B64B-80BBA141D89F}.png]]

vemos en el link canonico cambia y tambien tenemos en otro link el dominio se altera, lo que vemos es que se esta manipulando la cache externa ya que cambia con respecto a la cabecera por lo que si la victima no envía dicha cabecera no lo vera ya que la clave cache también se esta alterando.

lo que vamos a intentar es que sin necesidad de la cabecera nos genere un match, por ahora solo tenemos dos puntos los cuales se están inyectando, lo que vamos a hacer es enviar de forma continua hasta ver si logramos obtener uno tercer dato en la respuesta, esto puede significar que estamos de laguna forma ingresando a la cache interna:

![[{FEF2D0BE-357B-48FE-82E8-CC3982CF0BC3}.png]]

luego de un vemos que una nueva cabecera cambia, esto es muy bueno ya que el cambio en esta nueva cabecera nos indica de que logramos inyectar un valor de forma interna donde si intentamos una petición usual el dominio de test.com permanecerá fijo, en este punto vamos a dejar nuestro servidor exploit con el objetivo de que apunte a mi servidor malicioso la web y genere el alert:

![[{77061D41-DA9C-467D-B994-9A9B549BAA1A}.png]]

ahora el dominio de este server vamos a tratar de fijarlo en la cache para que el exploit funcione:

![[{E3A2DDC4-FD6F-4C8D-A484-ECA8B3A0215D}.png]]

vemos que se fijo el tercer link el cual es el que se cachea de forma interna, en este punto lo que vamos a intentar es si el laboratorio se completa:

![[{97FCCC82-F1D2-41F0-983C-989C46413175}.png]]

Lab Terminado.
