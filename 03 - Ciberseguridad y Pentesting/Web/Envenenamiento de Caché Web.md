---
aliases: ["Web Cache Poisoning"]
tags:
  - hacking/web
  - hacking/laboratorios
  - herramientas/burpsuite
estado: 🟢 Terminado
---
# Envenenamiento de Caché Web

El envenenamiento de caché web es un ciberataque en el que un atacante manipula una caché web para que almacene una respuesta HTTP maliciosa, que luego se distribuye a otros usuarios. Al recibir este contenido envenenado, los usuarios pueden sufrir diversos ataques, como el robo de información o el secuestro de sesiones, a través de vulnerabilidades como Cross-Site Scripting (XSS).

Otro concepto clave es la clave de caché (cache key), la cual, para comprenderla, podemos verla como el nombre que se usa tanto para guardar como para recuperar una respuesta en la caché.

> [!info] Clave de Caché (Cache Key)
> La clave de caché es un identificador único que utiliza un servidor de caché para almacenar y recuperar respuestas HTTP. Generalmente, se deriva de la URL de la solicitud, pero puede incluir otros componentes como cabeceras HTTP, cookies o parámetros de consulta, dependiendo de la configuración del servidor de caché. Si un atacante puede manipular un componente que no forma parte de la clave de caché (unkeyed input), pero que influye en la respuesta, puede envenenar la caché.

## PortSwigger

### Lab: Web Cache Poisoning con una Cabecera No Clave (Unkeyed Header)

Lo que tenemos que hacer en este laboratorio es, mediante una cabecera que la caché no interpreta para generar la clave de caché, inyectar un `alert(0)` para que se ejecute en otros usuarios que visiten la página principal.

Como es en la página principal, lo que vamos a hacer es directamente capturar la petición a la web principal:
![[Pasted image 20250926094028.png]]

Vemos en la respuesta algo interesante, que son las cabeceras `Cache-Control` y `Age`, donde una nos indica el tiempo máximo que permanece la petición en la caché y la otra nos indica el tiempo estimado que lleva esa respuesta en la caché.

> [!info] Cabeceras `Cache-Control` y `Age`
> *   **`Cache-Control`**: Esta cabecera HTTP define las directivas de caché para las solicitudes y respuestas. Por ejemplo, `max-age=3600` indica que el recurso es fresco durante 3600 segundos.
> *   **`Age`**: Esta cabecera indica el tiempo en segundos que el objeto ha estado en una caché de proxy. Un valor de `0` sugiere que la respuesta es fresca o que la caché se ha invalidado y se ha servido una nueva respuesta.

Podemos intentar poner parámetros para ver si la cabecera `Age` regresa a 0, ya que ese es un indicativo de que la clave de caché se está refrescando:

![[Pasted image 20250926094327.png]]

Vemos que la cabecera `Age` cambió a 0, por lo que se está tomando el parámetro como parte de la clave de caché. El objetivo era encontrar una cabecera que permita inyectar código malicioso y no sea detectada en la caché web para que se cargue en la página principal.

Para buscar la cabecera, como directamente nos dicen que es una cabecera, podemos usar una extensión de Burp Suite llamada `Param Miner`:

![[Pasted image 20250926094658.png]]

Vamos ahora al Repeater y podemos usar Param Miner para adivinar cabeceras (guess headers):
![[Pasted image 20250926094824.png]]

Vamos a ver el siguiente panel:
![[Pasted image 20250926094845.png]]

Y vamos a darle a 'OK', para luego irnos a `All Issues`:
![[Pasted image 20250926094940.png]]

![[Pasted image 20250926095023.png]]

Vemos que, al parecer, nos reporta un envenenamiento de caché donde, usando la cabecera `x-forwarded-host`, podemos envenenar la caché. Podemos intentarlo:

![[Pasted image 20250926095158.png]]

Vemos que esto funciona y se está poniendo en la respuesta. En este punto, cualquier usuario que haga una consulta a la raíz va a solicitar un script, al parecer, de la ruta que le pasemos. Con esto, podemos aprovechar el exploit server e intentar la alerta de la siguiente manera:

![[Pasted image 20250926095427.png]]

Usamos el dominio y modificamos para que sea igual al resto de la petición y enviamos el `alert(0)` en este caso. Vamos en este punto a intentar modificar la caché a ver qué sucede:

![[Pasted image 20250926095533.png]]

Funcionó, se cambió, y si recargamos la web desde el navegador, veremos la inyección en acción:
![[Pasted image 20250926095724.png]]

Perfecto, ahora vemos si con esto se resuelve el lab:

![[Pasted image 20250926095811.png]]

Lab Terminado.

### Lab: Web Cache Poisoning con una Cookie No Clave (Unkeyed Cookie)

En este laboratorio, el objetivo sigue siendo enviar un `alert()` envenenando la caché.

Vamos a capturar la petición de la página principal a ver qué está pasando:
![[Pasted image 20250926100042.png]]

Bueno, al parecer, tenemos que envenenar la caché mediante la cookie, en específico la de `fehost`. Vamos a ver si logramos ingresar algo allí:
![[Pasted image 20250926100208.png]]

Vemos que sí, al parecer estamos dentro de una tupla y podemos insertar cositas.

En este caso, vamos a intentar cerrar alguna cosa para escribir directamente dentro de la etiqueta `script`:
![[Pasted image 20250926100604.png]]

Vemos que estamos cerrando la `";` y luego insertamos el `alert` para al final comentar lo que queda colgado. Veamos si funcionó:

![[Pasted image 20250926100653.png]]

Lab Terminado.

### Lab: Web Cache Poisoning con Múltiples Cabeceras

En este caso, el objetivo es el mismo: ejecutar un `alert`, pero en esta ocasión lo que vamos a tener son múltiples cabeceras.

Vamos a comenzar capturando la petición de la web principal para ver cómo se tramita:

![[Pasted image 20250926101231.png]]

Vamos a volver a emplear la extensión de Param Miner con el objetivo de encontrar cabeceras vulnerables:

![[Pasted image 20250926101335.png]]

Nos reporta la cabecera `x-forwarded-scheme`. De forma rápida, esta cabecera nos permite definir el protocolo con el cual se realiza la conexión, ya sea `HTTP` o `HTTPS`. Lo que vamos a hacer en este punto es intentar usarla con `HTTP` a ver qué hace:

> [!info] Cabecera `X-Forwarded-Scheme`
> La cabecera `X-Forwarded-Scheme` es una cabecera HTTP no estándar (de facto) utilizada para identificar el protocolo (HTTP o HTTPS) que un cliente usó para conectarse a un proxy o balanceador de carga, cuando el proxy o balanceador de carga se conecta al servidor web usando un protocolo diferente. Es útil en entornos donde el tráfico se termina en un proxy HTTPS y luego se reenvía como HTTP al servidor de origen.

![[Pasted image 20250926101815.png]]

Vemos que nos realiza un redirect a la raíz haciendo el cambio a `https`. Ahora debemos buscar una forma que nos permita alterar este redirect. Podemos volver a emplear Param Miner a ver qué nos reporta:

![[Pasted image 20250926102002.png]]

Vemos que ahora nos reporta el `x-forwarded-host`, el cual puede que controle el dominio del redirect. Veamos:

![[Pasted image 20250926102116.png]]

Perfecto, esto ya controla el host. Entonces, lo que podemos hacer es buscar consultas alternas que se hagan al realizar la petición a la página principal:

![[Pasted image 20250926102346.png]]

Encontramos que se realiza esa petición a `tracking.js`, y si lo que hicimos anteriormente aplica a cualquier parte de la web, podemos intentar que esto cambie y se redirija a una petición maliciosa:

![[Pasted image 20250926102502.png]]

Perfecto, lo que hacemos es que en la caché se guarde ese redirect cada vez que se invoque al recurso, y ese redirect ahora vamos a controlarlo para que llame a un recurso malicioso:

![[Pasted image 20250926102746.png]]

Vemos el exploit malicioso, lo configuramos perfectamente y ahora vamos con la petición:
![[Pasted image 20250926102818.png]]

En este caso, configuramos y enviamos correctamente. Vemos si se completa el laboratorio:

![[Pasted image 20250926103322.png]]

Perfecto, ahora podemos ver si se completa el lab:

![[Pasted image 20250926103549.png]]

Lab Terminado.

### Lab: Web Cache Poisoning Dirigido Usando una Cabecera Desconocida

En este laboratorio, tenemos el mismo objetivo, pero nos hablan de envenenar la caché web con una cabecera desconocida. Vamos a ver qué podemos hacer.

Vamos a comenzar capturando la petición de la web principal a ver qué encontramos:

![[Pasted image 20250926105152.png]]

Bueno, eso tenemos, y lo que podemos hacer es volver a emplear la extensión de Param Miner a ver si encontramos algo:

![[Pasted image 20250926105817.png]]

Vemos que nos detecta la cabecera `X-host`. Podemos intentar a ver qué nos edita o nos inserta en la respuesta:

![[Pasted image 20250926110001.png]]

Vemos que, mediante esa cabecera, logramos cambiar el dominio de una solicitud, como ya lo hemos visto antes:

![[Pasted image 20250926110427.png]]
![[Pasted image 20250926110444.png]]

Perfecto, se queda en la caché y si revisamos:

![[Pasted image 20250926110558.png]]

Perfecto, veamos si con esto logramos completar la máquina.

Pues no se completa. Si revisamos un poco más de cerca la respuesta, vemos una llamada `vary`:

![[Pasted image 20250926110821.png]]

Esta cabecera en específico lo que le dice a la web es que todo lo almacenado en la caché se lo cargue solo a los usuarios con el `User-Agent` exacto que se envió, como el nuestro:

> [!info] Cabecera `Vary`
> La cabecera HTTP `Vary` le indica a los proxies y navegadores que la respuesta del servidor puede variar dependiendo de las cabeceras de la solicitud del cliente. Por ejemplo, `Vary: User-Agent` significa que la caché debe almacenar diferentes versiones del recurso para diferentes `User-Agent`s. Si un atacante envenena la caché con un `User-Agent` específico, solo los usuarios con ese mismo `User-Agent` recibirán la respuesta envenenada.

![[Pasted image 20250926110928.png]]

Lo que hace de forma general es que solo aplique para mí en este caso.

Lo óptimo en este tipo de casos es tratar de, por otros medios, obtener el `User-Agent` con el cual la víctima tramita sus peticiones. Con esto en mente, vemos en la descripción que los usuarios de la web siempre visitan los posts. Vamos a ver los posts:

![[Pasted image 20250926111306.png]]

Vemos que HTML es válido. Podemos intentar cargar una imagen hacia nuestro servidor para ver el `User-Agent`:

![[Pasted image 20250926111412.png]]

![[Pasted image 20250926111440.png]]

Perfecto, vemos nosotros el `User-Agent` específico para esta web. Lo que podemos hacer es pasárselo para que sea válido para la víctima:

![[Pasted image 20250926111555.png]]

Y si esperamos, vamos a ver que ahora sí:
![[Pasted image 20250926111607.png]]

Lab Terminado.

### Lab: Web Cache Poisoning a Través de una Cadena de Consulta No Clave (Unkeyed Query String)

Este laboratorio es vulnerable a un envenenamiento de la caché web porque tenemos una cadena como *query* que se excluye de la clave de caché, y el objetivo es enviar la alerta.

Como siempre, vamos a comenzar capturando la petición de la página principal:

![[Pasted image 20250926112233.png]]

Perfecto, podemos intentar con algún parámetro de prueba a ver si conseguimos que se genere una nueva clave de caché o que no, que sería lo ideal:

![[Pasted image 20250926112409.png]]

Vemos que se inyecta el parámetro, pero si intentamos enviar una nueva solicitud a la web principal:

![[Pasted image 20250926112618.png]]

Vemos que la caché es la misma, lo que quiere decir que no está creando una nueva clave de caché usando el parámetro, sino que simplemente está usando la misma raíz sin importar el parámetro que se le pase. En este punto, podemos intentar una XSS:
![[Pasted image 20250926113113.png]]

Vemos que estamos completando la solicitud, saliendo del link e ingresando una etiqueta `img` para generar el `alert(0)`:

![[Pasted image 20250926113153.png]]

Perfecto, veamos si se completa el laboratorio:

![[Pasted image 20250926113217.png]]

Lab Terminado.

### Lab: Web Cache Poisoning a Través de un Parámetro de Consulta No Clave (Unkeyed Query Parameter)

Este laboratorio requiere lo mismo: ejecutar un XSS con un `alert(1)` en la víctima, y en este caso se excluyen de la clave de caché algunos parámetros.

Vamos a comenzar de forma habitual interceptando una petición de la web principal para probar:

![[Pasted image 20250926151142.png]]

Si realizamos algo parecido que en el lab anterior, vamos a ver cómo la clave de caché cambia debido a que a los parámetros también los interpreta:

![[Pasted image 20250926151258.png]]

Vemos que pasa directo a 0, por lo que en este caso no podemos hacer lo mismo que en el laboratorio anterior.

En este caso, podemos usar la extensión de Param Miner para ver si puede encontrar algún parámetro que no se interprete:

![[Pasted image 20250926151502.png]]

Vamos a esperar a que nos responda y veremos lo siguiente:
![[Pasted image 20250926151724.png]]

Vemos que nos detecta el parámetro `utm_content`. Vamos a intentar a ver qué nos responde:

![[Pasted image 20250926151940.png]]

Perfecto, esto funciona y, como vemos en la respuesta, también se inserta. Vamos a intentar hacer la inyección XSS a ver qué sucede:

![[Pasted image 20250926152217.png]]

Perfecto, cómo se inserta el código. En este punto, lo que vamos a hacer es ver qué sucedió en la web y si se completó:

![[Pasted image 20250926152251.png]]

Lab Terminado.

### Lab: Ocultamiento de Parámetros (Parameter Cloaking)

Bueno, el *parameter cloaking* es una técnica dentro del envenenamiento de caché web donde ocultamos los parámetros maliciosos dentro de la solicitud, donde el servidor los ignora, pero la caché web los toma como parte única del recurso y almacena la respuesta generada.

> [!info] Parameter Cloaking
> El *Parameter Cloaking* ocurre cuando un servidor web ignora ciertos parámetros de una solicitud HTTP (por ejemplo, debido a un *parser* de URL diferente o una lista blanca de parámetros), pero la caché web sí los considera al generar la clave de caché. Esto permite a un atacante inyectar un parámetro malicioso que el servidor de origen ignora, pero que la caché utiliza para almacenar una respuesta envenenada que luego se sirve a otros usuarios.

Bueno, en el lab tenemos parámetros en la web y tenemos un parámetro que genera esa inconsistencia entre el servidor y la caché web. Tenemos que ejecutar el `alert`.

Como ya es costumbre, vamos a capturar la petición para ver cómo se tramita:
![[Pasted image 20250926152741.png]]

Vemos algo que llama la atención, y es esa cookie de `country`. Bueno, dejando de lado eso, lo que vamos a hacer es usar la extensión Param Miner:

![[Pasted image 20250926153154.png]]

Vemos que tenemos el parámetro `utm_content`. Lo que podemos hacer es intentarlo a ver qué sucede:
![[Pasted image 20250926153248.png]]

Perfecto, vemos que esto no lo toma y recarga sin problemas. Al parecer, en este caso se puede aplicar parecido, pero vamos a centrarnos en la cookie que tanto llama la atención.

Si nos fijamos en la respuesta, se llama a un recurso externo de `geolocate`. Vamos a tratar de capturarlo y enviarlo al Repeater a ver de qué se trata:

![[Pasted image 20250926153502.png]]

Vemos que tenemos un parámetro en especial que se repite en la respuesta. Podemos intentar cambiarlo por el momento a ver si este cambia en la respuesta también:

![[Pasted image 20250926153547.png]]

Sí cambia, y esto es código JavaScript, por lo que podemos intentar un ataque. El problema radica en que por defecto llama a `setCountryCookie`, el cual es un parámetro por defecto, y para que funcione tenemos que inyectar el código sin que eso cambie.

Recordemos que tenemos un parámetro que encontramos inyectable: `utm_content`. Vamos a intentar ponerlo a ver qué sucede:
![[Pasted image 20250926153937.png]]

Bueno, vemos que sí nos lo toma como válido, ya que no se recarga, pero no vemos que se pueda cambiar el valor.

Un concepto clave es que en algunas ocasiones, cuando nosotros enviamos un `;`, el *parser* interpreta todo lo que está después de este `;` como un nuevo parámetro, donde podemos intentar cambiar el valor de `callback`:

> [!tip] Uso del punto y coma (`;`) en URLs
> En algunos contextos, especialmente en aplicaciones web antiguas o con configuraciones específicas de *parsing* de URL, el punto y coma (`;`) puede ser interpretado como un separador de parámetros, similar al ampersand (`&`). Esto puede llevar a que un servidor web ignore un parámetro después de un `;` si no lo espera, mientras que una caché intermedia podría considerarlo como parte de la URL o de la clave de caché, creando una discrepancia explotable.

![[Pasted image 20250926154141.png]]

Vemos que se cambia la información. Con esto validado, podemos intentar una inyección de la siguiente manera:
![[Pasted image 20250926154427.png]]

Perfecto, se inyectó. Podemos ver en la web qué pasa:

![[Pasted image 20250926154443.png]]

Perfecto, veamos si se completó el lab:

![[Pasted image 20250926154459.png]]

Lab Terminado.

### Lab: Web Cache Poisoning a Través de una Solicitud GET "Fat"

En esta máquina, vamos a tener que inyectar un `alert` donde vamos a tener que enviar una solicitud por GET inflada, donde tenga más información de la que debería.

> [!info] Solicitudes GET "Fat"
> Una solicitud GET "Fat" (o "GET con cuerpo") es una solicitud HTTP GET que incluye un cuerpo de mensaje, algo que no es estándar para las solicitudes GET según la especificación HTTP/1.1 (RFC 2616). Sin embargo, algunos servidores y proxies pueden procesar estas solicitudes. Si una caché web ignora el cuerpo de la solicitud GET al generar la clave de caché, pero el servidor de origen sí lo procesa para generar la respuesta, un atacante puede inyectar contenido malicioso en el cuerpo que se cacheará y servirá a otros usuarios.

Vamos a comenzar, como siempre, capturando la petición:
![[Pasted image 20250926155020.png]]

Perfecto, vemos que se tramita de forma usual. Podemos ver también que, al igual que en el lab anterior, tenemos el `geolocate` que se llama:
![[Pasted image 20250926155135.png]]

Ahora, recordando que la caché web no toma en cuenta el *body* para la clave de caché y que podemos por GET mandar *body*, podemos intentar lo siguiente:

![[Pasted image 20250926155254.png]]

En este punto, lo que tenemos es que podemos cambiar el parámetro desde el cuerpo, y es más, esto se queda en caché, por lo que vamos a realizar de forma rápida la inyección:

![[Pasted image 20250926155419.png]]

![[Pasted image 20250926155428.png]]

Perfecto, veamos si el lab se completa:

![[Pasted image 20250926155451.png]]

Lab Terminado.

### Lab: Normalización de URL

Este laboratorio tiene una vulnerabilidad que no es directamente explotable con XSS debido a la codificación URL que se aplica en el navegador, pero vamos a aprovecharnos del proceso de normalización de la caché para lograr inyectar y lanzar un `alert(1)`.

> [!info] Normalización de URL
> La normalización de URL es el proceso de modificar y estandarizar una URL a una forma consistente. Esto puede incluir la eliminación de parámetros por defecto, la reordenación de parámetros, la conversión a minúsculas, la eliminación de barras finales, o la decodificación de caracteres codificados. Si un servidor de caché y un servidor de origen aplican diferentes reglas de normalización, un atacante puede explotar esta inconsistencia para envenenar la caché.

Como es de costumbre, vamos a analizar la petición de la web principal para ver:

![[Pasted image 20250926155924.png]]

Bueno, mucho no es que veamos que podamos hacer desde aquí, pero bueno, en este caso es enviarle un link a la víctima, por lo que podemos generar algún error de la siguiente forma:
![[Pasted image 20250926160223.png]]

Vemos esto y se ve algo peligroso. Podemos intentar cerrar el párrafo e insertar una inyección con etiquetas `script` a ver qué sucede:

![[Pasted image 20250926160325.png]]

Esto es algo bueno porque se inserta. Podemos intentar enviar esto en el navegador para probar primero con nosotros:

![[Pasted image 20250926160406.png]]

Vemos cómo se *URL-encode* la información y esto se almacena también en la caché, por lo que si recargamos el Burp, pasa igual:

![[Pasted image 20250926160551.png]]

Perfecto, esto nos quita el factor de inyección, pero ¿qué pasa si desde el Burp Suite, que no nos lo *URL-code*, lo ingresamos a la caché y luego enviamos el link a la víctima?

![[Pasted image 20250926160649.png]]

![[Pasted image 20250926160732.png]]
Esto funciona. Vamos a enviarlo a la caché y luego a enviar el link:

![[Pasted image 20250926160815.png]]

Lab Terminado.

### Lab: Web Cache Poisoning para Explotar una Vulnerabilidad DOM a Través de una Caché con Criterios de Cacheabilidad Estrictos

Bueno, en este laboratorio vamos a explotar una vulnerabilidad donde la caché que se emplea en el servidor tiene un criterio estricto de cara a ver qué respuesta cachea y cuál no. En este, al igual que en otros, tenemos que ejecutar un `alert`.

Como ya es costumbre, capturamos la petición de la web principal a ver con qué nos encontramos:

![[Pasted image 20250926161753.png]]

Vemos que tenemos las etiquetas `script` donde almacena el host y el path. Esto debe usarlo en algún lado, vamos a buscar:

![[Pasted image 20250926162102.png]]

Vemos que, en efecto, llama a una ruta donde usa el host y lo concatena con esa ruta. Podemos intentar con cabeceras que ya hemos usado, como `X-Forwarded-Host`, para intentar ver si modificamos el host:

![[Pasted image 20250926162455.png]]

Perfecto, vemos que sí logramos cambiar el host, y esto, combinado con la carga de información que hace luego, podemos configurar en nuestro exploit externo a ver qué podemos hacer. Algo a tener en cuenta es primero ver qué es lo que hace ese script, por lo que podemos intentar verlo:

![[Pasted image 20250926162623.png]]

Exactamente devuelve eso. Ahora, si revisamos nuestra web, esto aparece también, y aquí es donde se da el vector del ataque:

![[Pasted image 20250926162707.png]]

Perfecto, con eso claro, vamos a configurar nuestro exploit server:

![[Pasted image 20250926162820.png]]

Esto solo es una prueba, es por eso que no inyectamos nada aún. Podemos ya cambiar el host de la petición al de nuestro exploit server para ver cómo es que funciona:

![[Pasted image 20250926163003.png]]

Esto es perfecto, puede ser que simplemente no se esté interpretando al no tener el formato JSON. Vamos a intentar devolver un JSON primero para que se interprete:

![[Pasted image 20250926163328.png]]

![[Pasted image 20250926163512.png]]

Vemos que no carga nada, y si revisamos en la consola, vemos que nos dice que falta la cabecera de `Access-Control-Allow-Origin`, la cual es una cabecera de respuesta que determina los orígenes que permiten o pueden acceder a un recurso web.

> [!info] Cabecera `Access-Control-Allow-Origin` (CORS)
> La cabecera `Access-Control-Allow-Origin` es parte del mecanismo de Intercambio de Recursos de Origen Cruzado (CORS). Permite a un servidor indicar cualquier origen (dominio, esquema o puerto) distinto al suyo desde el que un navegador debe permitir la carga de recursos. Si esta cabecera no está configurada correctamente o falta, los navegadores bloquearán las solicitudes de origen cruzado por razones de seguridad, impidiendo que scripts de un dominio accedan a recursos de otro.

Esto se debe a que en nuestro exploit server, por configuración, puede que no pueda ser accedido, y para esto podemos hacer lo siguiente:

![[Pasted image 20250926163754.png]]

Como vemos, agregamos la cabecera y esto debería permitirle cargar desde otros dominios. Vemos si funciona:

![[Pasted image 20250926163905.png]]

Vemos que sí se inserta. Ya con esto, podemos configurar para que se realice una inyección de la siguiente manera:

![[Pasted image 20250926164003.png]]

![[Pasted image 20250926164328.png]]

Vemos si con esto completamos el lab:

![[Pasted image 20250926164354.png]]

Perfecto, vemos que se ejecuta:

![[Pasted image 20250926164408.png]]

Lab Terminado.

### Lab: Combinando Vulnerabilidades de Web Cache Poisoning

Bueno, en este laboratorio vamos a tener varios tipos de ataques donde vamos a tener que organizarnos correctamente para poder lograr completarlo. El objetivo, como siempre, es inyectar una alerta.

Vamos a capturar todo el tráfico de la web para ver qué obtenemos:
![[Pasted image 20250926192921.png]]
Vemos esta petición que parece ser de traducción. Vamos a ver qué nos retorna esto:

![[Pasted image 20250926193008.png]]

Vemos que tenemos la respuesta de todos los idiomas y diferente información. Vamos a irnos a la web y vamos a seleccionar un idioma a ver qué sucede:

![[Pasted image 20250926193105.png]]

Vemos que se tramitó:
![[Pasted image 20250926193157.png]]

Vemos que lo primero que se tramita es un `setlang/es`, donde observamos que define de alguna forma el idioma que elegimos. Esto realiza un redirect y listo.

Bueno, entonces con esto, la cadena de cómo vamos a tratar de atacar es que nosotros tengamos cacheada una inyección dentro del `translations.js`, donde luego intentemos hacer un redirect al idioma español y esto se inserte con el objetivo de lograr un XSS.

Bueno, vamos a tener todo listo, lo primero es la traducción:

![[Pasted image 20250926230211.png]]

Vamos a intentar, mediante nuestra extensión de Param Miner, descubrir alguna cabecera que nos permita inyectar información:

![[Pasted image 20250926230500.png]]

Vemos que tenemos una cabecera de `x-original-url`. Veamos qué generamos con esto:
![[Pasted image 20250926230732.png]]

Vemos que esto nos genera un *not found*, lo que quiere decir que esto nos puede estar intentando ingresar a `/test`.

Esto del todo no nos sirve, ya que queremos alterar la respuesta. Podemos tratar de volver a analizar la petición a la raíz:
![[Pasted image 20250926231116.png]]

La petición es igual a la del lab anterior. Podemos intentar la cabecera `X-Forwarded-Host` a ver si logramos cambiarlo:
![[Pasted image 20250926231404.png]]

Esto es perfecto, logramos cambiar esto y, como esto llama al script de `translation.json`, ya tenemos una forma de modificar el contenido si lo llamamos desde nuestro exploit. Vamos a configurar esto a ver qué sucede:

![[Pasted image 20250926231833.png]]

Ya tenemos el exploit server. Ahora vamos a cachear la respuesta para que esto suceda:

![[Pasted image 20250926231939.png]]

Bueno, el proceso es el siguiente: enviar esto a la caché, luego que se cachee lo que podemos hacer es intentar recargar la web en español a ver qué sucede. Para esto, necesitamos parámetros extras, como veremos en la URL:

![[Pasted image 20250926232846.png]]

De igual forma, vemos el problema en el exploit server a ver si ya funciona:

![[Pasted image 20250926232952.png]]

Perfecto, ahora vamos a ver si ya funciona:

![[Pasted image 20250926233158.png]]

Sí funciona, pero la inyección no se aplicó de forma correcta. Vamos a intentar de otra forma:

![[Pasted image 20250926233326.png]]

Vamos a ver ahora qué sucede:

![[Pasted image 20250926233408.png]]

Perfecto, ya funciona. Ahora nuestro objetivo es ver cómo podemos redireccionar la web hacia la web en español para que se aplique.

Si recordamos, tenemos una cabecera que realiza una redirección: `x-original-url`. Podemos intentar ver si esto funciona:

![[Pasted image 20250926235005.png]]

Vemos que sí logramos redirigir la web. En este caso, vamos a poner para que vaya a español y vamos a cachear las dos de forma continua con el objetivo de que cuando la víctima ingrese al *home*, todo se ejecute:

![[Pasted image 20250927000041.png]]

Lab Terminado.

### Lab: Inyección de Clave de Caché (Cache Key Injection)

Al igual que en el laboratorio anterior, se va a explotar una cadena de vulnerabilidades mediante las cuales vamos a llegar a ejecutar un `alert(1)` en el navegador de la víctima.


![[Pasted image 20250928124331.png]]

Bueno, comenzamos analizando las peticiones donde vamos a darnos cuenta de que la primera a la raíz `/` es un redirect y la seleccionada en la imagen es otro redirect. Entonces, por el momento, donde nosotros podemos manipular es en los puntos donde se generan estas redirecciones, y para analizar podemos intentar ver la petición a `/login?lang=en`, que es donde se aplica la segunda y última redirección.

Vamos ahora a ver cómo la página a la cual nos redirige ya carga el contenido de la web, sería la siguiente petición al último redirect.

![[Pasted image 20250928124731.png]]

Bueno, vemos que al hacerse la petición, ese parámetro se está aplicando directamente en la respuesta. Podemos intentar varias cosas, pero vamos a darnos cuenta de que, como se realiza un *URL-encode* al insertarse, esto no nos sirve de absolutamente nada por el momento.

![[Pasted image 20250928125111.png]]

Continuando con la búsqueda, vamos a encontrarnos con que carga un recurso dentro de un script. Podemos intentar ver cómo se tramita y el resultado de esa consulta:

![[Pasted image 20250928125230.png]]

Cosas a tomar en cuenta, como que esta petición se queda en caché. Vemos, además, que lo de `lang=en` se define en la petición y se inserta en la respuesta. Podemos intentar modificar esto a ver si podemos alterar de alguna forma la respuesta:

![[Pasted image 20250928125519.png]]

Vemos que sí se altera. Bueno, podemos intentar enviar la petición que se nos pasó en el lab para intentar ver qué podemos obtener de datos extras:

![[Pasted image 20250928125801.png]]

Podemos observar cómo esta cabecera de alguna forma nos acaba de permitir ver la clave de caché con la que se está almacenando, además de que si nos ponemos minuciosos, vamos a ver que al final de la misma se pasa un `$$` donde puede ser que sea alguna forma de dar un final a la cadena.

![[Pasted image 20250928130048.png]]
Bueno, regresamos a esta solicitud, ya que nuestro objetivo es encontrar una forma que nos permita inferir en este link.

Cosas que de primeras podemos intentar desde la misma URL es pasarle el parámetro `lang=test` para ver si logramos modificar su respuesta:

![[Pasted image 20250928130451.png]]

Veamos qué obtenemos:

![[Pasted image 20250928130746.png]]

Vemos que sí se modificó el contenido. Esto es interesante y un posible vector de ataque donde si nosotros logramos cargar el `lang=test`, esto también viajará a la otra petición como `lang-test`, donde se vería algo como lo siguiente:
![[Pasted image 20250928135317.png]]

Esto es más o menos lo que tendríamos. Ahora bien, en esta petición, si tratamos de enviar más parámetros, vamos a ver que no nos los incorpora. Bueno, como el objetivo es intentar alterar la respuesta, cosas que tenemos que tener en cuenta son el `vary: origin`, el cual se encarga de guardar y ejecutar la caché solo para objetivos del mismo origen con el mismo `User-Agent`.

Podemos intentar jugar con la cabecera `origin` a ver si logramos algún resultado:

![[Pasted image 20250928135720.png]]

Podemos observar que con esta cabecera pasa a ser parte de la clave de caché, y algo que vemos es que mediante los `$$` se está separando algo extraño. A más de eso, no logramos observar nada más. Algo que podemos intentar es jugar con los valores de ese parámetro de `cors` a ver qué logramos:

![[Pasted image 20250928140739.png]]

Esto es bueno, ya que logramos hacer que se considere. Ahora, algo que podemos hacer es una *header injection*. El propósito es intentar inyectar una nueva cabecera, y lo vamos a hacer de la siguiente manera:

![[Pasted image 20250928141240.png]]

Vemos que sí estamos inyectando cabeceras. Esto es mucho mejor, ya que podemos intentar a partir del `Content-Length` sobrescribir la información de la respuesta de la siguiente manera:

![[Pasted image 20250928141646.png]]

Perfecto, esto es bastante bueno, pero el problema que tenemos actualmente es cómo se tiene que consultar a nivel de clave de caché, ya que como vemos, esa clave es muy específica para un ataque, pero nosotros podemos intentar hacer que esto suceda, ya que recordemos que cómo se consulta a este documento lo conocemos, y el propósito será que se represente en la primera solicitud para que se consulte en forma de ataque. Ahora nuestro problema actual es que si intentamos ingresar parámetros extra en nuestra solicitud, no se tomarán en cuenta:

![[Pasted image 20250928142358.png]]

Vemos que no lo logramos. Podemos intentar con nuestra querida extensión encontrar algún parámetro inyectable:

![[Pasted image 20250928142553.png]]

Vemos que mediante el parámetro de `utm_content` ignora la caché, así que vamos a intentar:
![[Pasted image 20250928142833.png]]

Aunque efectivamente sí ignora la caché, no vemos que se esté implementando información.

Bueno, mediante la siguiente inyección, logramos redirigir y forzar el código en la web:

![[Pasted image 20250928151542.png]]

Ahora, si vemos cómo se está consultando por detrás la otra solicitud, vamos a ver lo siguiente:

![[Pasted image 20250928151629.png]]

Ahora tenemos un pequeño problema: cómo se tramita esto tiene un objetivo, y este es que nosotros almacenemos en la caché un código malicioso el cual contenga el mismo nombre de clave de caché. Por lo que el problema es que si vemos las dos claves de caché, vamos a observar que tienen cierta diferencia:

![[Pasted image 20250928152233.png]]

Nos centramos específicamente en que nos faltan los `$$`, pero esto podemos intentar solo agregarlo, y las dos solicitudes quedarían de la siguiente manera:

![[Pasted image 20250928152453.png]]

Y cómo se tramitaría mediante la URL:

![[Pasted image 20250928152522.png]]

Perfecto, vemos que las claves de caché ya son lo mismo y vería esto ejecutarse. Vamos a ver si lo logramos cacheando la solicitud URL y la petición:

![[Pasted image 20250928153001.png]]

Perfecto, revisemos si esto completó el lab:

![[Pasted image 20250928153031.png]]

A rasgos generales, para dejar todo claro, lo que se hizo fue generar una petición con valores exactos los cuales por detrás generan otra solicitud con valores exactos, generando allí una clave de caché la cual nosotros, mediante una petición alterna que genera la misma clave de caché pero es la inyectable, hacemos que se quede cacheada y cuando entra al *home* y genera esta segunda solicitud, lo que hace es agarrar la caché infectada y ejecutarla, generando el XSS con el `alert`.

Lab Terminado.

### Lab: Envenenamiento de Caché Interna

Tenemos que envenenar en esta máquina la caché interna. El objetivo será ejecutar una alerta.

Bueno, vamos a comenzar capturando una consulta de la raíz:

![[{F68BDB58-E8BB-4E96-9F77-9DFA375F4E2B}.png]]

Mediante algunas revisiones, vamos a ver que el link canónico no nos da paso a la ejecución de inyecciones maliciosas. Vamos a ver con nuestra extensión a ver qué encontramos:

![[{280C8418-096B-4353-9290-BC8FD792C0FC}.png]]

Vemos que en este caso podemos implementar la cabecera de `x-forwarded-host`. Vamos a intentarlo:

![[{BEEE3D35-2493-43EC-B64B-80BBA141D89F}.png]]

Vemos que en el link canónico cambia y también tenemos en otro link que el dominio se altera. Lo que vemos es que se está manipulando la caché externa, ya que cambia con respecto a la cabecera, por lo que si la víctima no envía dicha cabecera, no lo verá, ya que la clave de caché también se está alterando.

Lo que vamos a intentar es que, sin necesidad de la cabecera, nos genere un *match*. Por ahora, solo tenemos dos puntos los cuales se están inyectando. Lo que vamos a hacer es enviar de forma continua hasta ver si logramos obtener un tercer dato en la respuesta. Esto puede significar que estamos de alguna forma ingresando a la caché interna:

![[{FEF2D0BE-357B-48FE-82E8-CC3982CF0BC3}.png]]

Luego de un tiempo, vemos que una nueva cabecera cambia. Esto es muy bueno, ya que el cambio en esta nueva cabecera nos indica que logramos inyectar un valor de forma interna donde si intentamos una petición usual, el dominio de `test.com` permanecerá fijo. En este punto, vamos a dejar nuestro servidor exploit con el objetivo de que apunte a mi servidor malicioso la web y genere el `alert`:

![[{77061D41-DA9C-467D-B994-9A9B549BAA1A}.png]]

Ahora, el dominio de este server vamos a tratar de fijarlo en la caché para que el exploit funcione:

![[{E3A2DDC4-FD6F-4C8D-A484-ECA8B3A0215D}.png]]

Vemos que se fijó el tercer link, el cual es el que se cachea de forma interna. En este punto, lo que vamos a intentar es ver si el laboratorio se completa:

![[{97FCCC82-F1D2-41F0-983C-989C46413175}.png]]

Lab Terminado.