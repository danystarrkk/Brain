-------------

En esta clase se aborda un bypass de autenticación mediante el uso indebido del flujo implícito de OAuth, donde el error se encuentra en cómo la aplicación cliente valida la identidad del usuario tras completar el proceso de autorización.

Durante el flujo OAuth, el usuario inicia sesión a través de un proveedor externo (por ejemplo, una red social), que devuelve al cliente un token junto con información básica del usuario autenticado. La aplicación cliente luego usa estos datos para autenticar internamente al usuario en su sistema.

## PortSwigger

### Lab: Authentication bypass via OAuth implicit flow

Bueno en este laboratorio nos dice que emplea un servicio de OAuth que permite a los usuario acceder a sus redes sociales, al parecer de lado del cliente esta mal configurado por lo que podemos ingresar a otras cuentas sin ingresar contraseña, tenemos las credenciales de wiener.

Bueno vamos a comenzar interceptando absolutamente todo el trafico a ver que logramos obtener:

![[{23D5AA70-AA3F-4431-B87B-05D0FBA125E2}.png]]

ya logramos entrar y bueno vemos como se dieron las peticiones:

![[{E7AC5410-6E51-40D7-A573-6F8FE6DF623B}.png]]

Revisando un poco las peticiones encontramos la que vemos en imagen, al parecer es parte de lcomo se autentica y lo que esta pasando es que al pasarle la contraseña en un punto anterior los datos se están enviando directamente a a la web tanto el usuario como la contraseña y el token de sesión, es aquí el problema ya que le damos todo servido y lo enviamos en limpio lo que nos permite manipular la información del email y el usuario y provoca que al enviarlo como carlos nos lo de como valido ya que el correo es correcto, el usuario es correcto y el token también los es porque como no se esta validando del todo la información pues el token no esta enlazado de alguna forma a la cuenta de wiener, es por esto que nos deja pasar como veremos en la siguiente imagen:

![[{32466ABD-AA1F-416D-934F-2E4957E05D9F}.png]]
![[{1150F94F-B43E-489D-81C8-E04E14CAA034}.png]]

Lab Terminado.


### Lab: SSRF via OpenID dynamic client registration

Bueno primero necesitamos conocer lo que es open-ID facilita que una aplicación pueda confiar en que un usuario ha sido autenticado por un proveedor de identidad en lugar de tener que gestionar la autenticación de forma independiente.

Bueno la idea es acceder a un recurso que se tiene en la intranet de la maquina y obtener una flag para completar la maquina.

Vamos a comenzar como de costumbre en este caso capturando todo el trafico que se esta emitiendo al momento de iniciar sesión y una vez dentro vamos a analizar que fue lo que sucedio:

![[{73D8F282-8A28-4BF0-B314-BED7CD256583}.png]]

me llama la atención el `oauth` como este mediante `auth` se tramita una respuesta, vamos a copiar ese link y ver que sucede si intentamos ingresarlo dentro del navegador:
![[{D14F6A72-A1D3-41B1-A20D-FE218C3C0BD5}.png]]

Vemos una especie de error por falta de autorización ahora usualmente el `oauth` tiene ciertos archivos de configuración estándar están disponibles como puede ser el `.well-known/openid-configuration`

![[{52E216C6-F057-4CB4-BA05-2AEE7D2CB3DF}.png]]

Bueno vamos a analizar un poco esto y vemos algo que si llama nuestra atención que seria:
![[{6E00E3E4-92DF-46F5-AFD5-F367C92BB915}.png]]

es muy peligroso que se tenga conocimiento del punto de registro podemos hacer algunas cosas.

Vamos aver si podemos ingresar a `/reg` mediante `GET`:
![[{18C018D3-CC39-4E77-AFA0-F5C50B3B3769}.png]]
Vemos que en si no podemos así que pude ser mediante post, para esto capturamos la petición y la manipulamos:
![[{595EE279-83E0-4483-968C-0785736474BC}.png]]

vemos que esto no le gusta porque nos dice que esperaba data en JSON por lo que podemos cambiar el tipo de dato a ver que sucede:
![[{BA86E063-97C8-4736-896B-01B5F73E8FE7}.png]]

Vemos que nos dice que se tubo un error al analizar el cuerpo de la petición y vamos a tratar de registrar algo nuevo de la siguiente manera:
![[{EE8E484B-C449-4BC5-8809-1C472A828A4E}.png]]

Vemos una gran cantidad de campos de donde destacan algunos como el client_id:
![[{6B940432-6114-4EBC-82D5-A0A277B7D239}.png]]

y si recordamos encontramos el como se están cargando los logos por usuarios:

![[{617877CE-363B-48D6-94BE-27E4A34C40D4}.png]]

vamos a intentar pasarle ese id a ver que sucede:
![[{8D0FF98B-0470-4D00-B27F-23A278830EDE}.png]]

vemos que nos dice que no tenemos contenido lo que quiere decir que no tenemos nada asignado, pero y si mediante el registro logramos asignar algo?, lo que vamos a intentar es que este recurso que logra mediante el logo ingresar o cargar el link del logo cargue un recurso interno, especificamente el que nos pide la maquina:

![[{862C5C74-C276-485C-9659-94BFFCE39A35}.png]]

perfecto vamos a intentar ahora cargar el link del logo con el `client_id` que tenemos para ver si funciona:

![[{38C9D8E0-28AA-4583-A8BD-743534634498}.png]]

Pues logramos que suceda, vemos de forma interna el `secretAccessKey`, vamos a completar la maquina:
![[{2D9BFC45-F619-43D6-8C98-26E8BA541BB7}.png]]

Lab Terminada.


### Forced OAuth profile linking

En esta maquina vamos a tener la opción de vincular un perfil de una red social mediante el OAuth al parecer podemos cambiar de usuario vulnerándolo y el propósito es eliminar al usuario `carlos`.

El proceso puede parecer repetitivo pero lo que estamos haciendo es ingresar a la web mientras enviamos todo el trafico por detrás al burp.

Primero vamos a ingresar con las credenciales de `wiener` que nos dan:
![[{5D34C5FD-AA6D-4CCA-B488-E32466040183}.png]]

Ya dentro vemos ese `Attach a social profile` por lo que vamos a hacerlo:
![[{42B82022-97E0-49A3-A26E-2F46C728F49A}.png]]

usamos de igual forma las credenciales que nos dan y vamos a ver que sucede:
![[{F1D1A18C-80CA-442D-96BB-5F57A3DA5293}.png]]

vamos a continuar:
![[{CAA2518C-8E14-4F44-B5B3-AC9BA2089436}.png]]

Pues listo parece que enlazamos una especie de red social a nuestra cuenta, vamos a analizar todo lo que se tramito por detrás a ver que encontramos:
![[{9D8A06A6-1D83-423B-A0AF-2CA395413D62}.png]]

Analizando un poco lo que es el trafico de peticiones encontramos que esta en especial es la ultima petición que se realiza y permite la vinculación de la red social a la cuenta, algo que podemos hacer es ver si nosotros compartiendo este link sin abrirlo logramos vincular una cuenta externa, en este caso el usuario administrador abre todo link que le enviemos podemos intentar enviarlo justo en este punto a ver que sucede:

![[{0971B2B8-5CFD-45FD-A5D3-9026A76EE103}.png]]

estamos en ese punto exacto vamos a copiar la url y vamos a denegar esta petición por lo que en consecuencia hace que esta solicitud sea valida aun:
![[{DDB5C5A0-69CC-4822-91B2-F7918011DA6E}.png]]

como vemos mediante un `iframe` vamos a hacer que el usuario administrador la cargue, vamos aver si esto le gusta y se vincula nuestra cuenta a la de administrador:

Pendiente


### Lab: OAuth account hijacking via redirect_uri

En esta maquina vamos a tener una pequeña falla de OAuth provider que permite que los atacantes roben códigos de autorización asociados con otras cuentas, el objetivo es obtener el rol de administrador y vamos a eliminar al usuario `carlos`.

Al igual que en las otras maquinas vamos a estar capturando el trafico por el burp.

![[{FE1590FB-BF68-4BFB-B9D0-DC18D6FEBBC3}.png]]

Iniciamos sesión y pues bueno vemos que tenemos por detrás:

![[{CE3F6EC4-D4CE-40AA-999A-9F2D1AB7B766}.png]]

Encontramos la solicitud que define el `uri` y vemos todo el link, vamos a intentar hacer las pruebas en vivo por lo que vamos a capturar esta petición y vamos a modificar el dominio del `uri`:
![[{F163EE6D-0A4D-4A02-A87A-41676F486568}.png]]

enviemos y vemos que es lo que obtenemos:
![[{0AA38217-4FAE-49A7-A3DD-C5E6578DC571}.png]]

vemos que se envía el código pero a mas de esto podemos observar como el `URL` se cambio el dominio y se esta enviado el código hacia allá, esto no será valido pero encontramos el punto de entrada a como vamos a filtrar los códigos por que si nosotros enviamos esto a nuestro server exploit como solicitud podremos obtener los códigos.

En este punto vamos a hacer lo mismo y a obtener la url para poder configurar el Exploit Server de la siguiente manera:
![[{B997A393-F473-49C2-8EB7-BDC8C57D719B}.png]]

perfecto, en este punto lo que vamos a hacer es a guardarla y a enviarla a la victima, seguido revisamos los logs a ver que obtenemos:
![[{26AA8C9F-038E-4B42-88CE-B088E4B64688}.png]]

parece que nos a llegado vamos a intentar usarla en nuestra solicitud a ver que obtenemos:

![[{4BD477D7-0FDB-4790-86EC-5B6FFD72EF75}.png]]

Perfecto ya tenemos el panel de administrador, vamos a ingresar y a eliminar al usuario `carlos` a ver si se completa el lab:
![[{C952F692-AF84-44C2-B909-2B192445CDEC}.png]]

Lab Terminado.

### Lab: Stealing OAuth access tokens via an open redirect


Bueno en esta maquina tenemos que intentar obtener la api key mediante un open redirect y con esto vamos a tratar de obtener el token de acceso con el cual ingresaremos como admin.

Vamos como de costumbre a estar capturando el trafico con burp para poder analizarlo.

Como siempre vamos a hacer el login:
![[{8F67C231-6417-49C7-A876-67654D6C5143}.png]]

encontramos la misma petición pero en este caso ya no es valida ya que tiene algún tipo de seguridad que impide que cambiemos el `uri` por lo tanto solo nos queda seguir investigando, en este caso tenemos algunos post, vemos que pasa si comentamos en ellos y como se tramitan:


![[{41123E02-83A1-4271-B612-6F06E38E3A8A}.png]]

Bueno vemos que esta por hacer un post pero antes de eso hice hovering sobre el botón de `Next post` donde vemos que puede efectuarse un `open redirect` podemos copiar el link e intentar cambiar el redirect a ver si este si es vulnerable:

![[{368C8EF8-BE98-4A42-9964-9EAEB82811FF}.png]]


![[{7EB22968-4A9A-4907-856B-2566B4101C22}.png]]

funciona, esto nos acaba de enviar a `google` por lo que ya encontramos el open redirect, ahora la idea es intentar en nuestra petición que encontramos y dijimos que no es modificable hacer lo siguiente:
![[{7986A360-6FED-419C-B73C-BF9DC4CA5340}.png]]

como vemos modificamos pero no vamos a enviar si no a copiar el solo el contenido del `uri` e intentarlo en el navegador a ver que sucede:
![[{2378EB52-C42E-4E7F-B361-846284324C96}.png]]
![[{19142471-80E7-4CF2-A9BF-ABFCCF63425C}.png]]

Esto es excelente Funciona. bueno ahora si nosotros ahora alteramos este flujo, vemos que sucede:
![[{63593BCF-FBEC-4A54-91C0-0D0594561EB5}.png]]

![[{3E30A6C5-00A4-467D-B4DF-2043E134BA66}.png]]

pues vemos que en el fujo si se redirige a `google.com` por lo tanto si funciona y si vemos luego que sucede en la web observamos lo siguiente:
![[{AB3BB151-4947-431C-A77A-E1F0FD91875F}.png]]

com podemos ver se nos concatena a la petición datos como el `access_token` que es lo que queremos, pero por culpa de ese `#` que vemos lo mas posible es que no se tramite y solo nos de la petición en si y no los demás datos.

Bueno analizando un poco el trafico y el como se realizo todo vemos algo interesante:
![[{DB71C4DB-541F-409C-A939-1ECA706AEE38}.png]]

podemos observar que en esta petición antes de llegar al `oauth-callback` se tramita una petición la cual responde con un redirect que lleva un `bearer token`, si continuamos hasta el segundo `/me` en la peticiones veremos lo siguiente:

![[{C3002EE5-2F65-4A38-9108-D23B185F28F8}.png]]

Podemos observar en la petición que lleva el mismo `Bearer` dentro de la cabecera Authorization lo que nos lleva a pensar que si introducimos uno valido como vemos en la respuesta vamos a obtener información del usuario y esto incluye la `apikey`.

Con este flujo claro de como funciona podemos intentar la solicitud a ver que nos llega al exploit server:
![[{568A2461-3088-4E04-AEDD-BD7A16B88739}.png]]

perfecto como podemos ver modificamos con sentido al exploit server y a partir de allí nos dejamos llevar:
![[{9BEAB83E-C7BE-42F2-AAF8-5B162164490C}.png]]

vemos que no llegan peticiones mas que a la raiz debido a el `#` lo que vamos a intentar hacer es un remplazo donde quitemos ese `#` el como es mediante javascript desde nuestro `exploit server`.

Con esto en mente tenemos que comprender el flujo, recordemos que cuando llegamos a la url por primera vez no tenemos nada debido al hash por lo que si detectamos que no tenemos nada es cuando vamos a realizar un nuevo redirect mediante el cual vamos a implementar la redirección para que esto funcione de la siguiente manera:

![[{3963FDE1-4540-4458-A555-EE00BFC1616A}.png]]

ya con esto listo vamos a primero intentar ver el exploit por nosotros a ver si este funciona y nos llega primero nuestro token:
![[{B65906A3-4E3D-4208-BA48-580EB2E89D88}.png]]

perfecto ya nos llega la información de nuestra web, por lo tanto ahora podemos intentar enviarlo a la victima a ver si nos manda su access token:
![[{E7B8BEC9-AB69-4299-9E51-9E63C1A1A263}.png]]

ya tenemos el access token, ahora si recordamos lo que se emplea en la petición de `/me` es el access token por lo que podemos emplearlo a ver si nos permite ver el `apikey`:
![[{D1CFB898-4621-4F4B-AE72-BE9A5D89F391}.png]]

Perfecto, podemos completar el laboratorio:

![[{89C0A101-576F-458B-94AA-C38ECD3E201C}.png]]

Lab Terminada.


### Lab: Stealing OAuth access tokens via a proxy page

Este laboratorio vamos a tener que conseguir access token con los cuales vamos a poder obtener la información de usuario y para obtener el access token vamos a tener que investigar el flujo de OAuth.

Bueno como ya es costumbre vamos a pasar todo el fujo de la aplicación web por el burp del como iniciamos sesión y luego analizamos el como se realizaron las peticiones:
![[{6D0C1867-1782-4380-96C1-84CBC5BF7602}.png]]

a mas de esa que ya usamos y por el momento no tenemos otra forma de usarla pues no tenemos nada por lo tanto vamos a ver la parte de comentarios:
![[{B1EA0AEF-3EEC-450B-ACDA-8EF6D5C9640C}.png]]

realizamos un comentario, podemos intentar ver que es lo que se tramito por detrás:
![[{259FB960-7FFD-45E1-9BDA-67BB40E944F0}.png]]

vemos esta petición un poco rara donde al parecer estamos cargando mediante un  `script` con un `parent` donde usual mente se usa esta sintaxis para cargar algo `envevido` por decirlo de alguna forma como una especie de `iframe` podemos ver si revisando el código fuente de la web vemos algo:

![[{F34D3C58-C6FB-4349-89F8-42D346567C6E}.png]]

vemos que carga un `iframe` que es exactamente el formulario ya que es lo ultimo después de los comentarios, lo que podemos hacer en este punto es llevarnos el código en JavaScript para analizarlo un poco:

![[{59DCF4A9-F344-4AE5-9090-97B820A9ECF2}.png]] 

Leyendo un poco el código lo que esta haciendo es comunicarse con la pagina padre y enviarle la información en pocas palabras, entonces lo que primero vamos a hacer es veri si mediante la petición del comienzo logramos redirigir a esta petición del iframe:

![[{0F7CC423-A756-49E0-8F8A-96CB85749F3C}.png]]

pues lo logramos y si es valido en un principio, ahora lo que vamos a hacer es cargar esto dentro de un `iframe` este link de la siguiente manera:
![[{22119A5A-0CBF-4353-90DC-8599E9A45B95}.png]]

perfecto, ahora recordemos que nosotros lo que queremos hacer es pasarnos como la web padre de esta petición y para esto lo que vamos a hacer es recibir la información mediante un script de la siguiente manera:

![[{FC517A03-A82E-4185-AC0E-85E1B0CB6AB7}.png]]

perfecto esto ya nos permite enviar todo directamente al server exploit vamos a intentarlo:
![[{9BE4AF53-40D9-4DA1-B051-EBBA1D38312A}.png]]

llego de forma perfecta la información, en este punto o que vamos a hacer es quitar el url-encode y ya podremos ver el access token:
![[{8EABCCBC-AFAD-4028-83CD-3BBBB7BF5245}.png]]

perfecto ya tenemos el token y podemos aprovecharnos de la ruta `/me` que nos da información del usuario:

![[{A96C09C5-3661-4A86-AECF-E8D5AEC96F07}.png]]

listo usemos el `apikey` para ver si logramos completar la maquia:
![[{E0860CFE-A88E-4D63-B958-18CC4C28698C}.png]]

Lab Terminado.