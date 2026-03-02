---
aliases:
  - OAuth Bypass
  - OAuth Vulnerabilidades
  - PortSwigger OAuth Labs
tags:
  - hacking/web
  - herramientas/burpsuite
estado: 🟢 Terminado
---
En esta clase se aborda un bypass de autenticación mediante el uso indebido del flujo implícito de OAuth, donde el error se encuentra en cómo la aplicación cliente valida la identidad del usuario tras completar el proceso de autorización.

Durante el flujo OAuth, el usuario inicia sesión a través de un proveedor externo (por ejemplo, una red social), que devuelve al cliente un token junto con información básica del usuario autenticado. La aplicación cliente luego usa estos datos para autenticar internamente al usuario en su sistema.

# PortSwigger

## Lab: Authentication bypass via OAuth implicit flow

Este laboratorio describe un servicio OAuth que permite a los usuarios acceder a sus redes sociales. La configuración incorrecta del lado del cliente permite el acceso a otras cuentas sin necesidad de contraseña. Se proporcionan las credenciales de 'wiener'.

Comenzaremos interceptando todo el tráfico para analizar las peticiones:

![[{23D5AA70-AA3F-4431-B87B-05D0FBA125E2}.png]]

Una vez dentro, analizamos las peticiones realizadas:

![[{E7AC5410-6E51-40D7-A573-6F8FE6DF623B}.png]]

Al revisar las peticiones, identificamos una clave en el proceso de autenticación. El problema radica en que, tras una autenticación previa, los datos del usuario (email, nombre de usuario y token de sesión) se envían directamente a la aplicación web de forma no segura. Esta transmisión en claro permite la manipulación de la información del email y el usuario. Al modificar el email a 'carlos' y mantener un token de sesión válido (que no está estrictamente vinculado a la cuenta de 'wiener' debido a una validación deficiente), la aplicación lo acepta como válido, permitiendo el acceso a la cuenta de 'carlos', como se muestra en la siguiente imagen:

> [!info] Flujo Implícito de OAuth y Riesgos
> El flujo implícito de OAuth (Implicit Flow) está diseñado para clientes que no pueden mantener un secreto de cliente de forma segura (como aplicaciones JavaScript en navegadores). El token de acceso se devuelve directamente al navegador del usuario en la URL del `redirect_uri` como un fragmento (`#`). El riesgo principal surge cuando la aplicación cliente no valida adecuadamente la identidad del usuario asociada con el token, permitiendo la suplantación si el token o los datos de usuario adjuntos pueden ser manipulados o interceptados.

![[{32466ABD-AA1F-416D-934F-2E4957E05D9F}.png]]
![[{1150F94F-B43E-489D-81C8-E04E14CAA034}.png]]

Laboratorio Terminado.

## Lab: SSRF via OpenID dynamic client registration

Primero, es fundamental entender OpenID Connect (OIDC). OIDC es una capa de identidad construida sobre el protocolo OAuth 2.0, que permite a las aplicaciones cliente verificar la identidad de un usuario basándose en la autenticación realizada por un servidor de autorización, así como obtener información básica del perfil del usuario de forma interoperable y RESTful.

> [!info] OpenID Connect (OIDC)
> OIDC es un protocolo de autenticación que permite a los clientes verificar la identidad de los usuarios finales basándose en la autenticación realizada por un servidor de autorización, y obtener información básica del perfil de los usuarios finales. A diferencia de OAuth 2.0, que es un protocolo de autorización, OIDC añade una capa de identidad para la autenticación. Utiliza tokens JWT (JSON Web Tokens) para transmitir la información de identidad (ID Token).

El objetivo es acceder a un recurso interno de la intranet de la máquina para obtener una 'flag' y completar el laboratorio.

Como es habitual, comenzaremos capturando todo el tráfico generado durante el inicio de sesión para analizar las interacciones una vez autenticados:

![[{73D8F282-8A28-4BF0-B314-BED7CD256583}.png]]

Nos llama la atención la petición `oauth` y cómo se gestiona una respuesta a través de `auth`. Copiaremos ese enlace y lo probaremos directamente en el navegador para observar su comportamiento:
![[{D14F6A72-A1D3-41B1-A20D-FE218C3C0BD5}.png]]

Observamos un error de falta de autorización. Es común que los servicios OAuth/OpenID Connect expongan un endpoint de configuración estándar en `.well-known/openid-configuration`.

> [!tip] Descubrimiento de Endpoints OIDC
> El endpoint `.well-known/openid-configuration` es parte del mecanismo de "Discovery" de OpenID Connect. Permite a los clientes OIDC descubrir dinámicamente la información de configuración del proveedor de OpenID, incluyendo los URIs de los endpoints de autorización, token, información de usuario, registro de clientes, y las capacidades soportadas (algoritmos de firma, tipos de respuesta, etc.).

![[{52E216C6-F057-4CB4-BA05-2AEE7D2CB3DF}.png]]

Analizando la configuración, un punto que capta nuestra atención es:
![[{6E00E3E4-92DF-46F5-AFD5-F367C92BB915}.png]]

La exposición del endpoint de registro dinámico de clientes (`registration_endpoint`) es un riesgo significativo, ya que permite a un atacante registrar clientes maliciosos y potencialmente explotar otras vulnerabilidades.

> [!warning] Registro Dinámico de Clientes (Dynamic Client Registration)
> El registro dinámico de clientes permite a las aplicaciones registrarse automáticamente con un proveedor de OAuth/OIDC. Si este endpoint no está adecuadamente protegido o validado, un atacante puede registrar un cliente con `redirect_uri` maliciosos, `logo_uri` apuntando a recursos internos, o manipular otros parámetros para realizar ataques como SSRF, Open Redirect o phishing.

Intentaremos acceder a `/reg` utilizando una petición `GET`:
![[{18C018D3-CC39-4E77-AFA0-F5C50B3B3769}.png]]
La petición `GET` falla. Es probable que el endpoint requiera una petición `POST`. Capturaremos una petición y la manipularemos en Burp Suite:
![[{595EE279-83E0-4483-968C-0785736474BC}.png]]

La respuesta indica que se esperaba un cuerpo de petición en formato JSON. Modificaremos el `Content-Type` a `application/json` y el cuerpo de la petición:
![[{BA86E063-97C8-4736-896B-01B5F73E8FE7}.png]]

A pesar de cambiar el tipo de contenido, se produce un error al analizar el cuerpo de la petición. Intentaremos registrar un nuevo cliente con una estructura JSON válida, incluyendo un `redirect_uris` y un `client_name`:
![[{EE8E484B-C449-4BC5-8809-1C472A828A4E}.png]]

Se genera una respuesta con varios campos, entre los cuales destaca el `client_id`:
![[{6B940432-6114-4EBC-82D5-A0A277B7D239}.png]]

Recordamos que previamente observamos cómo se cargan los logos de los usuarios:

![[{617877CE-363B-48D6-94BE-27E4A34C40D4}.png]]

Intentaremos pasar este `client_id` para ver qué sucede:
![[{8D0FF98B-0470-4D00-B27F-23A278830EDE}.png]]

La respuesta indica que no hay contenido asignado. ¿Qué pasaría si, durante el registro, logramos asignar un `logo_uri` que apunte a un recurso interno? El objetivo es que el mecanismo de carga de logos, que procesa URLs, acceda a un recurso específico de la intranet, como el que nos pide el laboratorio (`/admin`):

> [!tip] Explotación de SSRF con `logo_uri`
> Muchos servicios OAuth/OIDC permiten especificar un `logo_uri` para mostrar el logo del cliente. Si el servidor procesa esta URL sin una validación adecuada, puede ser vulnerable a Server-Side Request Forgery (SSRF). Al proporcionar una URL interna (como `http://localhost/admin`), el servidor intentará acceder a ese recurso, revelando su contenido al atacante.

![[{862C5C74-C276-485C-9659-94BFFCE39A35}.png]]

Ahora, intentaremos actualizar el registro del cliente con el `client_id` obtenido, especificando el `logo_uri` malicioso para que apunte al recurso interno `/admin`:

![[{38C9D8E0-28AA-4583-A8BD-743534634498}.png]]

¡Éxito! Hemos logrado que el servidor procese la URL interna, revelando el `secretAccessKey`. Con esta información, podemos completar el laboratorio:
![[{2D9BFC45-F619-43D6-8C98-26E8BA541BB7}.png]]

Laboratorio Terminado.

## Forced OAuth profile linking

En este laboratorio, se presenta la opción de vincular un perfil de red social mediante OAuth. Al parecer, es posible vulnerar este proceso para cambiar de usuario, y el objetivo final es eliminar al usuario `carlos`.

El proceso inicial implica iniciar sesión en la aplicación web mientras se intercepta todo el tráfico con Burp Suite.

Primero, iniciaremos sesión con las credenciales de `wiener` proporcionadas:
![[{5D34C5FD-AA6D-4CCA-B488-E32466040183}.png]]

Una vez dentro, observamos la opción `Attach a social profile` y procederemos a utilizarla:
![[{42B82022-97E0-49A3-A26E-2F46C728F49A}.png]]

Utilizaremos las credenciales proporcionadas para la red social y observaremos el flujo:
![[{F1D1A18C-80CA-442D-96BB-5F57A3DA5293}.png]]

Continuamos con el proceso:
![[{CAA2518C-8E14-4F44-B5B3-AC9BA2089436}.png]]

Hemos enlazado un perfil de red social a nuestra cuenta. Ahora, analizaremos el tráfico interceptado para identificar posibles vulnerabilidades:
![[{9D8A06A6-1D83-423B-A0AF-2CA395413D62}.png]]

Al analizar el tráfico, identificamos una petición crucial que es la última en el proceso de vinculación del perfil social. La hipótesis es que si logramos que el administrador (quien abre cualquier enlace que le enviemos) acceda a este enlace específico, podríamos forzar la vinculación de su cuenta con nuestro perfil social. Intentaremos enviar este enlace en el momento preciso:

> [!warning] Ataque de Vinculación Forzada de Perfiles (Forced OAuth Profile Linking)
> Este ataque explota la confianza en el `redirect_uri` y la falta de validación del estado de la sesión. Si un atacante puede engañar a la víctima para que haga clic en un enlace de autorización OAuth que ya ha sido iniciado por el atacante, la cuenta de la víctima podría vincularse al perfil social del atacante, o viceversa, permitiendo el secuestro de la cuenta.

![[{0971B2B8-5CFD-45FD-A5D3-9026A76EE103}.png]]

En este punto exacto, copiaremos la URL y denegaremos la petición en Burp Suite. Esto es crucial, ya que la solicitud de vinculación, aunque denegada por nosotros, aún podría ser válida si el administrador la procesa.
![[{DDB5C5A0-69CC-4822-91B2-F7918011DA6E}.png]]

Utilizaremos un `iframe` para incrustar esta URL y hacer que el usuario administrador la cargue. El objetivo es que su cuenta se vincule a nuestro perfil social:

Pendiente

## Lab: OAuth account hijacking via redirect_uri

En este laboratorio, explotaremos una vulnerabilidad en el proveedor OAuth que permite a los atacantes robar códigos de autorización asociados con otras cuentas. El objetivo es obtener el rol de administrador y eliminar al usuario `carlos`.

Como en los laboratorios anteriores, interceptaremos el tráfico con Burp Suite.

![[{FE1590FB-BF68-4BFB-B9D0-DC18D6FEBBC3}.png]]

Iniciamos sesión y analizamos el tráfico generado:

![[{CE3F6EC4-D4CE-40AA-999A-9F2D1AB7B766}.png]]

Identificamos la solicitud que define el `redirect_uri`. Realizaremos pruebas en vivo capturando esta petición y modificando el dominio del `redirect_uri` a un servidor bajo nuestro control (Exploit Server):

> [!warning] OAuth Redirect URI Manipulation
> La manipulación del `redirect_uri` es una vulnerabilidad crítica en OAuth. Si el servidor de autorización no valida estrictamente el `redirect_uri` proporcionado por el cliente (por ejemplo, permitiendo subdominios arbitrarios o dominios no registrados), un atacante puede redirigir el código de autorización o el token de acceso a un servidor malicioso, secuestrando la sesión del usuario.

![[{F163EE6D-0A4D-4A02-A87A-41676F486568}.png]]

Enviamos la petición modificada y observamos la respuesta:
![[{0AA38217-4FAE-49A7-A3DD-C5E6578DC571}.png]]

Observamos que el código de autorización se envía, y lo más importante, el dominio de la URL ha sido modificado, redirigiendo el código. Aunque esta redirección directa no sea válida en este contexto, hemos identificado el vector para exfiltrar los códigos. Si enviamos esta solicitud a nuestro Exploit Server, podremos capturar los códigos de autorización.

Procederemos a obtener la URL completa para configurar nuestro Exploit Server de la siguiente manera:
![[{B997A393-F473-49C2-8EB7-BDC8C57D719B}.png]]

Guardamos la URL modificada y la enviaremos a la víctima. Luego, revisaremos los logs de nuestro Exploit Server para capturar el código de autorización:
![[{26AA8C9F-038E-4B42-88CE-B088E4B64688}.png]]

Hemos recibido el código. Ahora intentaremos usarlo en nuestra solicitud para obtener un token de acceso y, finalmente, el rol de administrador:

![[{4BD477D7-0FDB-4790-86EC-5B6FFD72EF75}.png]]

¡Perfecto! Hemos obtenido acceso al panel de administrador. Procederemos a eliminar al usuario `carlos` para completar el laboratorio:
![[{C952F692-AF84-44C2-B909-2B192445CDEC}.png]]

Laboratorio Terminado.

## Lab: Stealing OAuth access tokens via an open redirect

En este laboratorio, el objetivo es obtener la API key explotando una vulnerabilidad de Open Redirect, lo que nos permitirá robar un token de acceso y acceder como administrador.

Como es habitual, interceptaremos el tráfico con Burp Suite para su análisis.

Realizaremos el proceso de inicio de sesión:
![[{8F67C231-6417-49C7-A876-67654D6C5143}.png]]

Encontramos una petición similar a las anteriores, pero en este caso, el `redirect_uri` parece estar protegido y no permite modificaciones directas. Por lo tanto, debemos investigar otras vías. Observamos la funcionalidad de comentarios y cómo se procesan:


![[{41123E02-83A1-4271-B612-6F06E38E3A8A}.png]]

Observamos una petición `POST` pendiente. Al pasar el cursor sobre el botón `Next post`, notamos que la URL asociada podría ser vulnerable a un Open Redirect. Copiaremos el enlace e intentaremos modificar el parámetro de redirección para verificar la vulnerabilidad:

> [!info] Open Redirect
> Un Open Redirect ocurre cuando una aplicación web permite que un atacante especifique una URL arbitraria a la que el usuario será redirigido. Si esta redirección no se valida adecuadamente, un atacante puede dirigir a las víctimas a sitios maliciosos (phishing) o, en el contexto de OAuth, exfiltrar tokens de acceso o códigos de autorización si se combinan con otros parámetros vulnerables.

![[{368C8EF8-BE98-4A42-9964-9EAEB82811FF}.png]]


![[{7EB22968-4A9A-4907-856B-2566B4101C22}.png]]

¡Funciona! Hemos sido redirigidos a `google.com`, confirmando la vulnerabilidad de Open Redirect. Ahora, intentaremos aplicar este conocimiento a la petición OAuth que inicialmente parecía inmodificable:
![[{7986A360-6FED-419C-B73C-BF9DC4CA5340}.png]]

Hemos modificado la petición. En lugar de enviarla directamente, copiaremos solo el contenido del `redirect_uri` y lo probaremos en el navegador para observar el comportamiento:
![[{2378EB52-C42E-4E7F-B361-846284324C96}.png]]
![[{19142471-80E7-4CF2-A9BF-ABFCCF63425C}.png]]

¡Excelente, funciona! Ahora, si alteramos este flujo para redirigir a nuestro Exploit Server, ¿qué sucede?
![[{63593BCF-FBEC-4A54-91C0-0D0594561EB5}.png]]

![[{3E30A6C5-00A4-467D-B4DF-2043E134BA66}.png]]

Efectivamente, el flujo se redirige a `google.com`, lo que confirma la funcionalidad. Al observar lo que sucede en la web, notamos lo siguiente:
![[{AB3BB151-4947-431C-A77A-E1F0FD91875F}.png]]

Como podemos observar, el `access_token` se concatena a la petición, que es precisamente lo que buscamos. Sin embargo, la presencia del carácter `#` (fragmento de URL) significa que el `access_token` no se enviará al servidor en la petición HTTP, ya que los fragmentos son procesados únicamente por el navegador del cliente.

> [!info] Fragmentos de URL y Tokens de Acceso
> En el flujo implícito de OAuth, los tokens de acceso a menudo se devuelven como un fragmento de URL (`#access_token=...`). Los fragmentos de URL no se envían al servidor en la petición HTTP, sino que son manejados por el navegador del cliente. Esto es una medida de seguridad para evitar que los tokens se filtren a través de logs del servidor o proxies. Para exfiltrar un token de un fragmento, se requiere JavaScript en el lado del cliente para leer el fragmento y enviarlo al atacante.

Analizando el tráfico y el flujo completo, encontramos algo interesante:
![[{DB71C4DB-541F-409C-A939-1ECA706AEE38}.png]]

Observamos que, antes de llegar al `oauth-callback`, se procesa una petición que responde con una redirección que incluye un `bearer token`. Si seguimos el flujo hasta la segunda petición a `/me`, veremos lo siguiente:

![[{C3002EE5-2F65-4A38-9108-D23B185F28F8}.png]]

La petición a `/me` incluye el mismo `Bearer token` en la cabecera `Authorization`. Esto sugiere que si proporcionamos un `Bearer token` válido (como el que vemos en la respuesta), podremos obtener información del usuario, incluyendo la `apikey`.

Con este flujo de funcionamiento claro, intentaremos la solicitud para ver qué información llega a nuestro Exploit Server:
![[{568A2461-3088-4E04-AEDD-BD7A16B88739}.png]]

Perfecto, hemos modificado la URL para que apunte a nuestro Exploit Server. A partir de aquí, el flujo debería redirigir el token.
![[{9BEAB83E-C7BE-42F2-AAF8-5B162164490C}.png]]

Observamos que solo llegan peticiones a la raíz debido al carácter `#` en la URL. Para superar esto, implementaremos un reemplazo del `#` mediante JavaScript en nuestro Exploit Server.

> [!tip] Exfiltración de Fragmentos de URL con JavaScript
> Para exfiltrar datos contenidos en el fragmento de una URL (como `#access_token=...`), se puede usar JavaScript en una página controlada por el atacante. El script leería `window.location.hash` y luego enviaría ese valor a un servidor controlado por el atacante (por ejemplo, mediante una petición `XMLHttpRequest` o cargando una imagen con el token en la URL).

Con esto en mente, debemos comprender el flujo: al llegar a la URL por primera vez, no obtenemos el token debido al fragmento (`#`). Por lo tanto, si detectamos que el fragmento está presente, realizaremos una nueva redirección utilizando JavaScript para que el token se envíe al servidor de la siguiente manera:

![[{3963FDE1-4540-4458-A555-EE00BFC1616A}.png]]

Con la configuración lista, probaremos el exploit nosotros mismos para verificar que funciona y que nuestro token llega al Exploit Server:
![[{B65906A3-4E3D-4208-BA48-580EB2E89D88}.png]]

Perfecto, la información de nuestra web llega correctamente. Ahora podemos intentar enviar el enlace a la víctima para que nos envíe su `access_token`:
![[{E7B8BEC9-AB69-4299-9E51-9E63C1A1A263}.png]]

Ya tenemos el `access_token`. Recordando que la petición a `/me` utiliza este token, lo emplearemos para ver si nos permite obtener la `apikey`:
![[{D1CFB898-4621-4F4B-AE72-BE9A5D89F391}.png]]

¡Perfecto! Podemos completar el laboratorio:

![[{89C0A101-576F-458B-94AA-C38ECD3E201C}.png]]

Laboratorio Terminado.

## Lab: Stealing OAuth access tokens via a proxy page

En este laboratorio, el objetivo es obtener tokens de acceso para acceder a la información de usuario. Para ello, investigaremos el flujo de OAuth.

Como es habitual, interceptaremos todo el flujo de la aplicación web con Burp Suite, desde el inicio de sesión, para analizar las peticiones realizadas:
![[{6D0C1867-1782-4380-96C1-84CBC5BF7602}.png]]

Además de la petición ya analizada, y al no encontrar otras vías inmediatas, nos centraremos en la funcionalidad de comentarios:
![[{B1EA0AEF-3EEC-450B-ACDA-8EF6D5C9640C}.png]]

Realizamos un comentario y analizamos el tráfico generado:
![[{259FB960-7FFD-45E1-9BDA-67BB40E944F0}.png]]

Observamos una petición peculiar que parece cargar contenido mediante un `script` con referencia a `parent`. Esta sintaxis se utiliza comúnmente para cargar contenido incrustado, similar a un `iframe`. Revisaremos el código fuente de la página para confirmarlo:

![[{F34D3C58-C6FB-4349-89F8-42D346567C6E}.png]]

Confirmamos que se carga un `iframe`, que corresponde al formulario de comentarios. En este punto, extraeremos el código JavaScript asociado para un análisis más detallado:

![[{59DCF4A9-F344-4AE5-9090-97B820A9ECF2}.png]] 

El código JavaScript se encarga de comunicarse con la página padre y enviarle información. Primero, intentaremos verificar si la petición inicial de OAuth puede redirigir a esta petición del `iframe`:

> [!info] Comunicación entre `iframe` y `parent` (postMessage)
> La comunicación entre un `iframe` y su ventana `parent` se realiza comúnmente a través de `window.postMessage()`. Esto permite enviar mensajes de forma segura entre ventanas de diferentes orígenes. Sin embargo, si la validación del origen del mensaje es débil o inexistente, un atacante puede simular ser la ventana `parent` o el `iframe` para interceptar o inyectar datos.

![[{0F7CC423-A756-49E0-8F8A-96CB85749F3C}.png]]

¡Lo logramos! La redirección es válida inicialmente. Ahora, incrustaremos este enlace dentro de un `iframe` de la siguiente manera:
![[{22119A5A-0CBF-4353-90DC-8599E9A45B95}.png]]

Perfecto. Nuestro objetivo es actuar como la página padre de esta petición. Para ello, implementaremos un script que reciba la información enviada por el `iframe` de la siguiente manera:

![[{FC517A03-A82E-4185-AC0E-85E1B0CB6AB7}.png]]

Con esto, ya podemos enviar toda la información directamente a nuestro Exploit Server. ¡Intentémoslo!
![[{9BE4AF53-40D9-4DA1-B051-EBBA1D38312A}.png]]

La información ha llegado perfectamente. Ahora, decodificaremos el `url-encode` para revelar el `access_token`:
![[{8EABCCBC-AFAD-4028-83CD-3BBBB7BF5245}.png]]

¡Perfecto! Ya tenemos el token de acceso. Podemos aprovechar la ruta `/me` para obtener información del usuario:

![[{A96C09C5-3661-4A86-AECF-E8D5AEC96F07}.png]]

Con la `apikey` obtenida, intentaremos completar el laboratorio:
![[{E0860CFE-A88E-4D63-B958-18CC4C28698C}.png]]

Laboratorio Terminado.