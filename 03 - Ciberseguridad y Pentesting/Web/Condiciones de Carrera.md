---
aliases:
  - Race Conditions
  - Condiciones de Carrera
tags:
  - hacking/explotacion
estado: 🟢 Terminado
---
# Condiciones de Carrera (Race Conditions)

## ¿Qué son las Condiciones de Carrera?

Las **condiciones de carrera** (también conocidas como **Race Conditions**) son un tipo de vulnerabilidad que puede ocurrir en sistemas informáticos donde dos o más procesos o hilos de ejecución compiten por los mismos recursos sin un mecanismo adecuado de sincronización para controlar el acceso a estos.

Esto significa que si dos procesos intentan acceder a un mismo recurso compartido al mismo tiempo, puede resultar en una salida impredecible para uno o ambos procesos, o incluso en un comportamiento no deseado en el sistema.

> [!info] Secciones Críticas y Primitivas de Sincronización
> Una "sección crítica" es una parte del código que accede a un recurso compartido. Para prevenir condiciones de carrera, se utilizan primitivas de sincronización como **mutexes** (exclusión mutua), **semáforos**, **bloqueos (locks)** o **monitores**. Estas garantizan que solo un proceso o hilo pueda acceder a la sección crítica en un momento dado, asegurando la integridad de los datos y la predictibilidad del sistema.

## Impacto en la Seguridad y Explotación

Los atacantes pueden explotar las condiciones de carrera para llevar a cabo ataques de denegación de servicio (**DoS**), sobrescribir datos críticos, obtener acceso no autorizado a recursos, o incluso ejecutar código malicioso en el sistema.

El impacto de las condiciones de carrera en la seguridad depende de la naturaleza del recurso compartido y del tipo de ataque que se pueda llevar a cabo. En general, las condiciones de carrera pueden permitir a los atacantes acceder a recursos críticos, modificar datos importantes o incluso tomar el control completo del sistema.

> [!tip] Técnicas Comunes de Explotación
> La explotación de Race Conditions a menudo implica la manipulación del tiempo de ejecución para que el atacante pueda insertar una operación maliciosa entre dos operaciones legítimas. Esto puede lograrse mediante:
> *   **Time-of-check to time-of-use (TOCTOU)**: Una vulnerabilidad donde el estado de un recurso cambia entre el momento en que se verifica y el momento en que se usa.
> *   **Creación de archivos temporales**: Un atacante puede crear un archivo con el mismo nombre que un archivo temporal legítimo antes de que el programa lo haga, o manipular su contenido.
> *   **Bloqueos de archivos**: Intentar bloquear un archivo antes de que un proceso legítimo pueda hacerlo.
> Herramientas como `race-cli` o scripts personalizados pueden ser utilizados para automatizar la ejecución concurrente de peticiones y explotar estas vulnerabilidades en entornos web o de sistema.

## Ejemplo Práctico

Por ejemplo, supongamos que dos procesos intentan acceder a un archivo al mismo tiempo: uno para leer y el otro para escribir. Si no existe un mecanismo adecuado para sincronizar el acceso al archivo, el proceso de lectura podría leer datos incorrectos o el proceso de escritura podría sobrescribir datos importantes que deben ser preservados.

## Mitigación

Por lo tanto, es crucial que los desarrolladores y administradores de sistemas implementen medidas para prevenir y mitigar las condiciones de carrera en sus sistemas.

> [!info] Estrategias de Mitigación
> Las principales estrategias para mitigar las condiciones de carrera incluyen:
> *   **Sincronización de Hilos/Procesos**: Uso de mutexes, semáforos, locks, y monitores para controlar el acceso a recursos compartidos.
> *   **Diseño Inmutable**: Hacer que los datos sean inmutables siempre que sea posible, eliminando la necesidad de sincronización para esos datos.
> *   **Transacciones Atómicas**: Asegurar que las operaciones que modifican múltiples recursos se completen por completo o no se realicen en absoluto.
> *   **Bloqueo de Archivos (File Locking)**: Utilizar mecanismos de bloqueo a nivel de sistema operativo para archivos.
> *   **Revisión de Código y Pruebas de Concurrencia**: Realizar auditorías de código y pruebas exhaustivas bajo condiciones de alta concurrencia para identificar y corregir posibles Race Conditions.

## PortSwigger

## Limit overrun race conditions

bueno el propósito es comprar cierto producto pagando un valor no esperado.

Para esto lo que vamos a hacer es comenzar ingresando en la web.

![[Pasted image 20260311190918.png]]

Como logramos observar tenemos 50 dólares y además un cupón de descuento del 20% por lo tanto veamos que tenemos en la tienda:

![[Pasted image 20260311191020.png]]

Mi objetivo es directamente comprar la chaqueta no me alcanza por lo tanto primero agreguemos al carrito:

![[Pasted image 20260311191102.png]]

Como podemos ver tenemos una opción para aplicar el cupón de forma única por lo tanto vamos a hacerlo:

![[Pasted image 20260311191154.png]]

Bueno no podemos aplicarlo mas de una vez.

Nuestro pensamiento nos dice que podemos tener 2 faces una donde se aplica el cupón y otra la validación donde se registra en el sistema, pero y que sucede si nosotros simplemente aplicamos el cupón muchas veces antes de llegar a la fase de verificación?, entonces como el sistema no registra aun el uso del cupón en la cuenta lo ingresamos muchas veces.

Vamos a hacerlo de dos formas y la primera es capturando la petición con la que se emplea la aplicación del cupón:

![[Pasted image 20260311191710.png]]

ya desde el repeater vamos a crear un grupo y vamos a duplicar dentro de este grupo la consulta:

![[Pasted image 20260311191907.png]]

Como vemos tenemos 50 peticiones y vamos a enviar todas en paralelo a ver cuantas ingresan porque no siempre todas serán válidas:

![[Pasted image 20260311192016.png]]

Como decimos algunas aplican pero otras no por lo que vamos a ver cuanto es el descuento:

![[Pasted image 20260311192054.png]]

Como vemos el descuento ya es mayor a solo el 20% a comparación del comienzo.

Como podemos observar esta es la primera forma y para seguir bajando el precio es cuestión de ir aumentando el numero de peticiones para intentar que un numero mayor de peticiones ingrese.

Bueno la segunda forma es cargar un código que se puede cargar en Burp Suite que esta en el siguiente [link](https://github.com/PortSwigger/bambdas/blob/main/CustomAction/ProbeForRaceCondition.bambda) , con eso vamos a igual mente capturar la petición y enviarla a repeater:
![[Pasted image 20260311192717.png]]

como vemos entramos en Custom actions y luego load para luego seleccionar la opción de create new template:

![[{840B214E-273C-48CD-9284-4A0E109D1D2D}.png]]

vemos que se nos despliega la siguiente ventana:

![[Pasted image 20260311193000.png]]

como vemos solo nos desplazamos a Trigger race condition ya que actualmente ya se implemento el código y listo le damos en create using this template, si necesitamos cambiar el código o se quiere pegar el código de la web pues no hay problema:

![[Pasted image 20260311193301.png]]

ahora en este caso yo implemente el código de la web vamos a cambiar el numero de peticiones al deseado en mi caso usare 50 y le daré en ok:

![[Pasted image 20260311193434.png]]

es cuestión de darle a boto de play y veamos que sucede:

![[Pasted image 20260311193558.png]]

Como logramos observar tenemos ya un descuento enorme y esto ya lo podemos pagar:

![[Pasted image 20260311193638.png]]

Lab Terminado.

### Bypassing rate limits via race conditions

En este laboratorio tenemos que sobrepasar los limites de intentos en un inicio de sesión para realizar un ataque de fuerza bruta.

Bueno vamos directamente en el Burp Suite ya capturada la petición de inicio de sesión:

![[Pasted image 20260311194147.png]]

Bueno como podemos acontecer una condición de carrera aquí es primero utilizando la extención llamada Turbo Intruder, ya instalada clic derecho, extenciones y la de Turbo Intruder todo desde la petición y veremos la siguiente ventana:

![[Pasted image 20260311194502.png]]

Para ejecutar la condición de carrera vamos a modificar la parte de Last code used por la de race-single-packet-attack.py:

![[Pasted image 20260311194557.png]]

Podemos observar el código y tendremos que modificar ciertas cosas en la petición donde el valor de password lo representamos como una cadena en python como  `%s` :

![[Pasted image 20260311194724.png]]

ahora en el código vamos a agregar nuestra lista de contraseñas:

![[Pasted image 20260311195125.png]]

tenemos que copiar las contraseñas y luego modificar el bucle y agregar el valor y lo enviamos:

![[Pasted image 20260311200939.png]]

como podemos observar solo tenemos un código de respuesta `302` lo que nos quiere indicar que es valida vamos a verlo:

![[Pasted image 20260311201050.png]]

Lab Terminado.


### Multi-endpoint race conditions

Al parecer tenemos un mecanismo de compra que nos permite comprar productos por un precio no intencionado y el objetivo es pagar un producto para el cual no tenemos suficiente dinero.

![[Pasted image 20260311202219.png]]

![[Pasted image 20260311202232.png]]

Como podemos observar tenemos dos peticiones las cuales la primera es cuando agregamos un producto al carrito y la segunda cuando pagamos un producto.

tenemos que pensar en faces las segunda petición que nos permite comprar un producto lo que hace en realidad es validar si tenemos suficiente dinero y luego comprar pero y si luego de validar logramos incorporar otro producto para que ya la ser valido solo se pague lo de un comienzo y la compra del otro sea gratis por decirlo de alguna manear, nosotros para esto tenemos que verificar los tiempos en los que estos responden y si una tarda mas que otra en este caso si lo hacen la de agregar un producto tarda mas que al validar y compara y si enviamos en paralelo funcionara:

![[Pasted image 20260311214400.png]]

Como observamos tenemos en un grupo las dos peticiones y si enviamos en paralelo:

![[Pasted image 20260311214335.png]]

Lab Terminado.

### Single-endpoint race conditions

 Al parecer alguien con una dirección de correo tiene una invitación pendiente para ser administrador de la web, donde si alguien logra crear la cuenta de forma exitosa va a heredar las propiedades de administrador.

Bueno entonces tenemos un cliente de email y vamos a comenzar:

![[Pasted image 20260312092810.png]]

tenemos el apartado que parece ser el cambio de email, podemos hacer una prueba cambiando el email y registrando el flujo en Burp Suite para analizarlo:

![[Pasted image 20260312092920.png]]

habla de un link de confirmación y si vemos en el cliente de email vamos a encontrar dicho email:

![[Pasted image 20260312093000.png]]

Confirmamos y ya veremos en nuestro dashboard que se cambia el correo:

![[Pasted image 20260312093027.png]]

Bueno esto lo registramos en Burp Suite a sí que vamos a ver la petición con la que se cambio el email:

![[Pasted image 20260312093122.png]]

Vemos la petición y lo que vamos a hacer es crear un grupo y vamos a duplicar la petición unas 10 veces y cambiar en cada una su correo para intentar enviar por individual cada uno de la siguiente manera:


![[Pasted image 20260312093414.png]]

ya con esto enviamos todas pero por conexiones separadas a ver que sucede:

![[Pasted image 20260312093442.png]]

Bueno si observamos el cliente vamos a ver el cliente donde deberíamos ver todas:

![[Pasted image 20260312093546.png]]

Bueno no sucede nada inusual pero y si mandamos ahora en paralelo?, veamos que sucede:

![[Pasted image 20260312093633.png]]

y si verificamos nuevamente el correo tendremos:

![[Pasted image 20260312093732.png]]

si ponemos atención envía a test10 pero en el contenido la confirmación es para test2, aquí ya detectamos la condición de carrera porque al parecer se realizan dos operaciones donde al momento de solicitar el cambio de email al parecer redacta el cuerpo del correo y pues suponemos que el correo se inserta en pendiente y cuando ya tenemos el mensaje pues lo revisa y envía al correo pendiente pero aquí es donde se desincroniza ya que al momento de redactar el correo y verificar por el correo al que tiene que enviar ya tenemos otro en la lista por lo que lo envía a ese y no al propietario original.

Entonces lo que vamos a tratar es intentar registrar un correo que si me llegue primero:

![[Pasted image 20260312094338.png]]

ahora vamos a duplicar esta consulta para crear una que contenga el correo victima:

![[Pasted image 20260312094526.png]]

ya con las dos peticiones vamos a crear un grupo con las dos para enviarlas en paralelo intentando que el correo se genere para carlos pero nos llegue a nosotros:

![[Pasted image 20260312094620.png]]

ahora si lo enviamos vamos a ver que sucede:

![[Pasted image 20260312094920.png]]

Como podemos observar esto no sucede a la primera ya que se necesito de varias repeticiones así que paciencia.

Cuando ya confirmamos el correo se vinculo a la cuenta:

![[Pasted image 20260312100113.png]]

y si nos fijamos tenemos ya el panel de administrador por lo tanto ya podemos terminar el laboratorio borrando al usuario Carlos:

![[Pasted image 20260312100157.png]]

Lab Termiando.

### Exploiting time-sensitive vulnerabilities

En el laboratorio tenemos un mecanismo para resetear contraseña roto al parecer que tenemos que verificar debido a que tenemos una rotura en la criptografía para la creación de tokens.

![[Pasted image 20260312100648.png]]

vamos a enviar esto y si nos fijamos en el cliente de email vamos a encontrar una ruta la cual contiene un token:

![[Pasted image 20260312100822.png]]

Podemos copiarlo y mediante alguna herramienta de identificación verificar que encriptación es:

![[Pasted image 20260312101242.png]]

Vemos un posible SHA1, lo que podemos pensar es puede que este creado de diferentes formas como por ejemplo el tiempo, la fecha, un valor que envía, tenemos muchas posibilidades pero podemos intentar ver como se tramita al entrar a la url:

![[Pasted image 20260312101545.png]]

como observamos si se tramita un `csrf` puede estar como no puede estar incluido.

Entonces lo que podemos hacer es tratar de enviar dos consultas de forma paralela a ver es lo que obtenemos:

![[Pasted image 20260312101811.png]]

enviamos y si revisamos los tiempos vemos que difieren:

![[Pasted image 20260312101853.png]]
![[Pasted image 20260312101909.png]]

uno mayor que otro y si nos fijamos en los correos:

![[Pasted image 20260312101956.png]]

vemos que no cambian y es que al no tener los mismo tiempos quiere decir que no se envían en paralelo sino secuencial y puede ser alguna restricción dependiendo de la cookie de sesión para evitar problemas.

Tomando lo anterior en cuenta vamos a intentar refrescar esa cookie para que se envíen dos peticiones en paralelo pero desde diferente sesión, donde podríamos tener lo siguiente en el Burp Suite:

![[Pasted image 20260312102427.png]]

![[Pasted image 20260312102437.png]]

ya con esto tenemos dos sesiones por lo tanto con diferentes vamos a intentarlo a ver si es funcional:

![[Pasted image 20260312102829.png]]

Ahora si las peticiones se enviaron en paralelo, veamos el panel de email a ver que sucedió:

![[Pasted image 20260312102938.png]]

Podemos observar que el token de los dos es igual, bueno esto quiere decir que para generar el token si esta implementado el tiempo actual y posiblemente un valor secreto. Ahora no sabemos este valor pero podemos seguir haciendo pruebas como ahora enviar las dos en paralelo pero un para `carlos` y otra para `wiener` a ver si podemos usarlo para el usuario carlos:

![[Pasted image 20260312104048.png]]

ahora vemos la URL en el email:

![[Pasted image 20260312104203.png]]

a esta la vamos a intentar abrir pero utilizando `user=carlos` :

![[Pasted image 20260312104243.png]]

vemos que es valido por lo tanto vamos a cambiar la contraseña:

![[Pasted image 20260312104324.png]]

Pues vemos como logramos cambiar la contraseña e ingresar a su cuenta por lo que ahora podemos eliminar al usuario `carlos`:

![[Pasted image 20260312104411.png]]

Lab Terminado.

### Partial construction race conditions

Bueno al parecer nosotros podemos sobrepasar el registro del email mediante una condición de carrera y un email el cual no poseemos.

Entonces en este laboratorio no tenemos credenciales e intentar conectarnos como uno con permisos administrativos por lo tanto vamos a registrarnos primero:

![[Pasted image 20260312105215.png]]

![[{C210CB6B-DBDD-4D0C-B6EA-4D3E438E49F9}.png]]

vemos que en realidad tenemos que registrarnos vamos el correo `@ginanjuice.shop` pero nosotros no tenemos acceso a tal dominio, ahora vamos a intentar registrar algo de forma valida mientras todo se esta capturando para ver que sucede:

![[Pasted image 20260312105553.png]]

si verificamos la petición para ver como se tramita veremos lo siguiente:

![[Pasted image 20260312105644.png]]

ya tenemos la petición y como prueba intentamos volver a enviarlo pero vemos que no nos deja debido a que esta en espera de confirmación.

![[Pasted image 20260312105752.png]]

vemos que si cambiamos el nombre de usuario tampoco es valido.

Navegando por la web y mirando como esta funciona encontramos un script de JavaScript interesante:

![[Pasted image 20260312110150.png]]

y si revisamos encontramos en el código de la respuesta la función de confirmEmail:

![[Pasted image 20260312110227.png]]

al parecer la confirmación es una petición por `POST` a la ruta `/confirm? + action` donde `action` es un token y su valor de token.

Vamos a representar esto en Burp Suite:

![[Pasted image 20260312110510.png]]

vemos que si nos responde como token invalido y donde logramos ver que a posterior de enviarlo lo que tenemos es la creación de dicho token el cual se envía al correo en un en lase donde se verifica.

Bueno lo que podemos tratar de hacer es pasaría si al mismo tiempo en el que creamos una cuenta enviamos la confirmación del token?, queremos suponer que al estar echo por faces si intentamos verificar una cuenta que acaba de ser creada y que aun no se le asigna un token podríamos llegar a identificar a su token como vacío es decir `token=`, vemos primero que sucede con eso:

![[Pasted image 20260312111816.png]]

como vemos es algo prohibido esto debe ser porque se identifico esa vulnerabilidad y se soluciono pero, esta no es la única forma de representar algo como vacío, también podemos hacer `token[]=` o `token[]`:
![[Pasted image 20260312111946.png]]

como vemos si nos deja pasar y en realidad es valido si pasa por lo tanto tomando en cuenta esto vamos a crear un grupo ya tener lista la petición que registra el usuario para intentar esto:

![[Pasted image 20260312112152.png]]

![[Pasted image 20260312112204.png]]

vemos que no es valido pero si nos fijamos mas allá en el tiempo podemos ver que no fue del todo en paralelo y además al responder de alguna forma el servidor mas rápido a la verificación que a al registro al no tener registrado el usuario aun no confirma nada por lo tanto sigue siendo invalido, entonces lo que vamos a hacer es jugar con el Turbo Intruder.

El objetivo es enviar para un registro una tanda de 50 consultas al token al tiempo que se realiza el registro el objetivo es que alguna de estas logre se valida al tiempo que se registra su usuario y no se le asigna token provocando la confirmación de token vacío.

Para lograr esto vamos a primero ir a petición de registro y mandarlo al Turbo Intruder:

![[Pasted image 20260312113038.png]]

Bueno ya con esto lo que vamos a hacer es generar valores de nombre de usuario como `test1,test2` y así consecutivamente modificando el código de la siguiente manera:

![[Pasted image 20260312113714.png]]

lo que estamos haciendo es para la petición de registro armamos un `gate` y luego con un bucle anidado en ese mismo `gate` almacenamos las 50 peticiones de confirmación al token para posteriormente enviarlas al abrir el `gate` con `openGate`, vemos si funciona:

![[Pasted image 20260312114127.png]]

bueno lo que sucede es que no funciono solo vemos que se registraron correctamente y esto es cuestión de ir probando aumentamos a `60` y cambiamos el usuario y también el correo y así continuamente cambiamos el valor a 50, 30 y vamos probando y esperando encontrar algo:

![[Pasted image 20260312114946.png]]

vemos que funciono y activamos la cuenta de `beater9` así que vamos intentar iniciar sesión:

![[Pasted image 20260312115056.png]]

como vemos ya esta el laboratorio Terminado.