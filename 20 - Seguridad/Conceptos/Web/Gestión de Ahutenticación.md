---
aliases: ["PortSwigger"]
tags: ["hacking/web", "herramientas/burpsuite"]
estado: 🟢 Terminado
---
# Gestión de Ahutenticación

## Lab: Username enumeration via different responses

Para completar este laboratorio, debemos encontrar usuarios válidos en el sistema y luego realizar un ataque de fuerza bruta para encontrar la contraseña.

En este laboratorio, se nos proporciona una lista de usuarios y contraseñas.

Si intentamos iniciar sesión con credenciales incorrectas, observaremos lo siguiente:
![[Pasted image 20250924132941.png]]

Vemos que solo nos dice `usuario inválido`. Podemos usar esta respuesta para identificar usuarios válidos. Vamos a capturar la petición:
![[Pasted image 20250924133123.png]]

Perfecto, enviaremos esta petición al Intruder para intentar identificar posibles usuarios válidos. Usaremos la lista proporcionada para facilitar el proceso:
![[Pasted image 20250924133239.png]]

Filtramos por la longitud de la respuesta y vemos que para el usuario `athena` obtenemos un mensaje diferente: `contraseña incorrecta`. Con un usuario válido, podemos repetir el proceso, pero esta vez con la lista de contraseñas:

![[Pasted image 20250924133440.png]]

Perfecto, con la contraseña `robert` logramos una redirección (follow redirect), lo que indica que es correcta. Vamos a iniciar sesión:
![[Pasted image 20250924133542.png]]

Lab Terminado.

> [!info] Enumeración de usuarios por respuestas diferentes
> Esta técnica se basa en analizar las respuestas del servidor a diferentes entradas. Si el servidor responde de manera distinta (por ejemplo, con un mensaje de error diferente, un código de estado HTTP distinto, o una longitud de respuesta variada) cuando un usuario es válido frente a cuando no lo es, se puede enumerar usuarios. Es una forma común de eludir protecciones básicas.

## Lab: 2FA simple bypass

En este laboratorio, tenemos un factor de doble verificación (2FA) que debemos eludir. Tenemos las credenciales de la víctima: `carlos:montoya`.

Comencemos analizando cómo está configurado el 2FA:
![[Pasted image 20250924134210.png]]

Después de ingresar las credenciales, nos lleva a este panel. Si revisamos la solicitud cuando ingresamos el código, esta realiza una redirección:
![[Pasted image 20250924134423.png]]

Esa es la ruta a la que la aplicación web redirige después de proporcionar el código correcto. Lo que podemos hacer es intentar ver si, una vez que estamos en el punto de introducir el código, al cambiar la ruta a `/my-account`, el código ya no sea necesario. Intentémoslo:
![[Pasted image 20250924134705.png]]

Nos permite pasar de forma correcta una vez que estamos en el punto de introducir el código.

Lab Terminado.

> [!tip] Bypass de 2FA por Path Traversal/Redirección
> Algunos sistemas 2FA solo verifican el código en una ruta específica. Si un atacante puede redirigir la sesión a una ruta de "cuenta" o "panel" directamente después de la autenticación inicial (antes de la verificación 2FA), y el servidor no valida que el 2FA se haya completado, se puede eludir la protección. Esto a menudo se logra manipulando la URL o las cabeceras de redirección.

## Lab: Password reset broken logic

En este laboratorio, buscamos acceder como el usuario `carlos`. Se menciona una lógica de restablecimiento de contraseña rota, por lo que primero vamos a ingresar a la web, iniciar sesión y buscar este restablecimiento defectuoso.

Parece ser, por cómo se describe, un problema que corrompe de alguna forma la manera en la que se recupera una contraseña. Podemos intentar realizar un cambio y ver cómo se procesa:
![[Pasted image 20250924135802.png]]

Nos solicitaron un correo y nos enviaron el siguiente enlace. Vamos a interceptar directamente la dirección de este enlace y ver qué se procesa.
![[Pasted image 20250924140223.png]]

Lo hemos usado, pero si recargamos, al parecer podemos reutilizarlo. Sin embargo, lo que estamos buscando es alguna forma en la que se esté utilizando esto. Podemos interceptar esto al enviar para ver cómo se está procesando:
![[Pasted image 20250924140359.png]]

Vemos que de forma literal se está procesando para el usuario `wiener`. Podemos modificarlo para ver qué pasa si cambiamos el nombre de usuario a `carlos`:
![[Pasted image 20250924140511.png]]

Al parecer no valida nada, ya que vemos que nos deja avanzar:
![[Pasted image 20250924140553.png]]

Cambiamos correctamente las contraseñas.

Lab Terminado.

> [!warning] Vulnerabilidad en el restablecimiento de contraseña por IDOR
> Este laboratorio demuestra una vulnerabilidad de IDOR (Insecure Direct Object Reference) en la función de restablecimiento de contraseña. Al manipular el parámetro `username` en la petición de cambio de contraseña, un atacante puede restablecer la contraseña de cualquier usuario sin autorización adicional. La aplicación no verifica si el token de restablecimiento pertenece al usuario especificado en la petición POST.

## Lab: Username enumeration via subtly different responses

En este punto, volvemos a realizar ataques de fuerza bruta. Lo que vamos a hacer es una enumeración de usuarios y luego descubrir la contraseña, utilizando las listas que nos dan. La forma en que esto cambia dependiendo de si es válido o no es sutil, por lo que debemos descubrirlo.

En el panel de inicio de sesión vemos lo siguiente:
![[Pasted image 20250924141225.png]]

Ya no lo pone tan fácil, debido a que nos dice `usuario o contraseña inválida`.

Vamos a analizar un poco más a fondo. Intentemos capturar la petición y ver qué obtenemos:
![[Pasted image 20250924141523.png]]

Bueno, vemos lo mismo, pero como no tenemos un usuario válido, no tenemos con qué comparar. Vamos a intentar el mismo ataque que la vez anterior. Veremos si obtenemos algo por la longitud, código de estado o cualquier otro indicador útil.

Realizaremos nuevamente el ataque de fuerza bruta con el cual intentaremos encontrar alguna diferencia en la forma en que el usuario correcto responde:
![[Pasted image 20250924143001.png]]

Usaremos Grep - Extract para ver el mensaje por el que vamos a filtrar y ver si sufre cambios:
![[Pasted image 20250924143103.png]]

Con esto, vamos a filtrar por esa cadena:
![[Pasted image 20250924143127.png]]

Vemos que al usuario `ae`, en comparación con otros, le falta el punto final. Ahora podemos intentar un ataque de fuerza bruta para este usuario para ver qué obtenemos:
![[Pasted image 20250924143255.png]]

Perfecto, vemos que para una de las contraseñas se aplica una redirección (follow redirect). Con esto, vamos a iniciar sesión:
![[Pasted image 20250924143348.png]]

Lab Terminado.

> [!info] Enumeración de usuarios por respuestas sutilmente diferentes
> A veces, la diferencia entre una respuesta para un usuario válido y uno inválido no es obvia (misma longitud, mismo código de estado). Sin embargo, puede haber variaciones sutiles en el contenido del cuerpo de la respuesta (por ejemplo, un carácter faltante, un mensaje ligeramente distinto). Herramientas como Burp Suite Intruder, con la función "Grep - Extract", son esenciales para identificar estas pequeñas diferencias y automatizar la enumeración.

## Lab: Username enumeration via response timing

En este laboratorio, por lo que se entiende, primero tenemos que enumerar ciertos usuarios y luego encontrar la contraseña del usuario válido, todo esto basado en el tiempo de respuesta.

Lo primero que podemos hacer es intentar iniciar sesión, pero veremos que después de algunos intentos se bloquea y envía otro tipo de mensaje:
![[Pasted image 20250924172657.png]]

Esto nos puede detener cuando hacemos un ataque de fuerza bruta, ya que en diferentes sitios se puede bloquear.

Para evitar que esto suceda, lo que podemos hacer es arrastrar ciertas cabeceras para tratar de hacerle pensar a la aplicación web que las peticiones provienen de otro sitio o de distinto origen, lo que evitará este problema.

Usaremos la cabecera `X-Forwarded-For`, la cual contiene la dirección IP del cliente que realizó la solicitud:

![[Pasted image 20250924173117.png]]

Vemos que ya funciona; en pocas palabras, volvemos a tener 3 intentos.

Las respuestas, dependiendo de si son correctas o incorrectas, en ocasiones su tiempo de respuesta puede aumentar o disminuir. Lo que vamos a hacer es aumentar el tamaño de la contraseña de forma masiva y ver qué sucede:
![[Pasted image 20250924173402.png]]

Lo que descubrimos es que con usuarios válidos, la aplicación web tarda cierto tiempo en responder. Esto puede ser porque al ser un usuario válido y comenzar a comparar la contraseña, esta se demore. En este punto, vamos a enviar al Intruder para hacer el ataque:
![[Pasted image 20250924173651.png]]

En este caso, la configuración cambia, ya que tenemos que configurar dos *payloads* que cambien, esto para evitar el bloqueo. Vemos qué sucede:
![[Pasted image 20250924173951.png]]

Vemos que el usuario `adkit` parece ser válido, ya que el valor en "Response Completed" es el valor en milisegundos que demoró la petición. Para ver ese campo, que usualmente no es visible, vamos a los tres puntos y lo activamos. Ahora intentemos con ese usuario un ataque a la contraseña para ver si funciona:
![[Pasted image 20250924174436.png]]

Encontramos la contraseña. Vamos a ver si esto completa el laboratorio:
![[Pasted image 20250924174617.png]]

Lab Terminado.

> [!tip] Ataques de temporización (Timing Attacks)
> Los ataques de temporización explotan las diferencias en el tiempo de respuesta de una aplicación para inferir información sensible. En la autenticación, si la verificación de un usuario válido toma más tiempo que la de uno inválido (por ejemplo, porque el servidor intenta comparar la contraseña), se puede usar esta diferencia para enumerar usuarios. La cabecera `X-Forwarded-For` es crucial para eludir bloqueos basados en IP, ya que permite rotar la dirección IP percibida por el servidor.

## Lab: Broken brute-force protection, IP block

En este caso, tenemos un listado de contraseñas para el usuario `carlos`, pero al parecer lo que sucede es que después de varios intentos, la IP es bloqueada. Vamos a intentar eludir esto.

Vamos a ver qué tiene este inicio de sesión:
![[Pasted image 20250924174927.png]]

Vemos que sí es vulnerable, ya que nos lista. Sí podríamos listar usuarios, pero ese no es el objetivo. Después de unos intentos, sucede lo siguiente:

![[Pasted image 20250924175033.png]]

Nos bloquea cada minuto. En este caso, lo que vamos a intentar es hacer dos intentos incorrectos para `carlos`, y luego uno correcto para refrescar el contador. Vamos a intentarlo:

![[Pasted image 20250924175244.png]]

Esto parece funcionar. Vamos a seguir usando el Intruder, pero necesitamos crear un diccionario de contraseñas y usuarios que realice esta acción, y para crearlo, lo haremos con Python:

Generación de usuarios:
```python
#!/usr/bin/python3

def genUsers():
    for i in range(0, 200):
        if i % 3 == 0:
            print("wiener")
        else:
            print("peter")

```

![[Pasted image 20250924181035.png]]

Y eso lo metemos en un archivo `users`. Ahora vamos con las contraseñas, donde cada 2 contraseñas necesitamos usar `peter` para `wiener`:

```python
#!/usr/bin/python3

def genPassword():
    with open("passwords") as f:
        lines = f.readlines()

    i = 0

    for line in lines:
        if i%3 == 0:
            print("peter")
        else:
            print(line.strip())
        i+=1

genPassword()

```

![[Pasted image 20250924181513.png]]

Perfecto, esto también vamos a ingresarlo en un archivo `password`.

Bueno, con estos dos diccionarios, ya vamos a intentar el ataque por medio del Intruder:
![[Pasted image 20250924181757.png]]

Listo, podemos intentar el ataque, pero antes tenemos que configurar el siguiente parámetro para asegurar que el orden de las peticiones sea correcto:
![[Pasted image 20250924182107.png]]

Perfecto, enviamos y vemos lo siguiente:
![[Pasted image 20250924182233.png]]

Vemos que la contraseña es `harley`. Perfecto, veamos si con esto terminamos el laboratorio:

![[Pasted image 20250924182329.png]]

Lab Terminado.

> [!warning] Bypass de bloqueo por IP con rotación de credenciales
> Este laboratorio ilustra cómo eludir un bloqueo de IP simple en ataques de fuerza bruta. La técnica consiste en intercalar intentos fallidos para el usuario objetivo con intentos exitosos (o al menos no bloqueados) para otro usuario, o incluso para el mismo usuario con una contraseña conocida, para "refrescar" el contador de intentos fallidos asociado a la IP. Esto requiere una lógica de ataque más compleja, a menudo implementada con *payloads* personalizados en herramientas como Burp Suite Intruder.

## Lab: Username enumeration via account lock

En este caso, al parecer vamos a tener que provocar algún bloqueo de cuenta para identificar cuándo un usuario es válido.

Comenzamos capturando una petición para ver por detrás cómo se está procesando la solicitud:
![[Pasted image 20250924183805.png]]

Bueno, vamos a enviarlo al Intruder y, mediante un ataque de tipo Cluster Bomb, vamos a intentar obtener el usuario válido:
![[Pasted image 20250924184112.png]]

Bueno, filtramos también en el Grep-Match por una cadena concreta y vemos que se bloquea el usuario `adsl`. Esto ya nos dice que este usuario es válido. Vamos a intentar un segundo ataque, pero solo para obtener su contraseña:

![[Pasted image 20250924184503.png]]

En este caso, podemos observar cómo el ataque no es del todo efectivo, ya que seguimos bloqueados, pero debido a un fallo lógico, supongo, lo que vemos es que la contraseña es válida, pero por el bloqueo no podemos entrar. Vamos a esperar el minuto y a intentarlo después para ver qué sucede:

![[Pasted image 20250924184639.png]]

Lab Terminado.

> [!info] Enumeración de usuarios por bloqueo de cuenta
> Algunos sistemas de autenticación bloquean una cuenta después de un número determinado de intentos fallidos. Si este bloqueo ocurre solo para usuarios válidos (y no para usuarios inexistentes), un atacante puede enumerar usuarios intentando iniciar sesión con una contraseña incorrecta repetidamente. El bloqueo de la cuenta es el indicador de que el nombre de usuario es válido.

## Lab: 2FA broken logic

En este punto, lo que vamos a intentar es romper la lógica del segundo factor de autenticación (2FA).

Bueno, mientras estamos en el proxy de Burp Suite, esto nos ayudará a capturar las diferentes solicitudes, incluyendo el factor de doble paso (2FA), para luego enviarlo al Repeater:
![[Pasted image 20250924191541.png]]

Bueno, ya podemos ver cómo las peticiones se están ejecutando. Se intentó justo en `login2` cambiar eso de `wiener` a `carlos`, pero no funciona, no nos envía a nosotros.

Ahora vamos a seguir viendo más peticiones:

![[Pasted image 20250924191922.png]]

Encontramos la petición con la cual se está intentando ingresar como `carlos` y, además, vemos que por alguna razón nos lo permite múltiples veces. Lo que podemos hacer es enviar una solicitud con la *request* anterior para que se envíe un código y, con la *request* actual, enviarla al Intruder e intentar todas las combinaciones posibles:

Vamos a crear primero esto con `seq` de la siguiente manera:
```bash
seq -w 1 9999 > password
```

![[Pasted image 20250924192333.png]]

Con esto ya tenemos la lista, vamos a intentar el ataque con el Intruder:

![[Pasted image 20250924192619.png]]

Vemos un `302` (Found), lo que indica que podría ser válido. Vamos a intentarlo:

![[Pasted image 20250924193126.png]]

Lab Terminado.

> [!warning] Bypass de 2FA por fuerza bruta en el token
> Si el token 2FA es un número predecible (como un PIN de 4 o 6 dígitos) y no hay un bloqueo de intentos o un límite de tiempo para su validación, es vulnerable a ataques de fuerza bruta. La clave es identificar la petición que valida el token y luego usar Burp Suite Intruder para probar todas las combinaciones posibles. La falta de un mecanismo de rate limiting es crítica aquí.

## Lab: Brute-forcing a stay-logged-in cookie

Por alguna razón, este laboratorio tiene una cookie persistente que mantiene a los usuarios en sesión incluso si cierran el navegador.

Bueno, tenemos que intentar ingresar como `carlos`.

Vamos a comenzar, como siempre, capturando todo el tráfico mientras probamos un poco la web con credenciales válidas:

![[Pasted image 20250924193954.png]]

Bueno, analizando, vemos que una vez iniciada la sesión, tenemos una cookie `stay-logged-in` que se mantiene. Podemos ver que Burp, al seleccionarlo, lo detecta como Base64, así que para maniobrar mejor, vamos a intentar desde la consola:

![[Pasted image 20250924194129.png]]

Vemos el `wiener:...`. Podemos seleccionar eso y ver su longitud:

![[Pasted image 20250924194220.png]]

Vemos que tiene una longitud de 32 bytes, lo que nos da indicios de ser MD5. Podemos usar algún software que nos permita ver el tipo de hash para confirmar:
![[Pasted image 20250924194339.png]]

Vemos que sí es MD5. Podemos tratar de ver si tal vez es la contraseña:
![[Pasted image 20250924194541.png]]

Perfecto, es la contraseña en MD5. Con esto y teniendo la lista de posibles contraseñas, automatizar el procedimiento de crear la cookie y luego que intente con esa desde el Intruder. Entonces, a esa petición, enviémosla al Intruder:
![[Pasted image 20250924194836.png]]

Podemos ver ya en el Intruder el "Payload Processing". Luego vamos a agregar uno:
![[Pasted image 20250924194952.png]]
Le decimos que primero transforme las contraseñas a MD5.

![[Pasted image 20250924195043.png]]
Ahora le agregamos antes de eso el `carlos:`.

![[Pasted image 20250924195156.png]]
Por último, todo en Base64 y vamos a ver qué tal funciona.

![[Pasted image 20250924200147.png]]

Perfecto, ya tenemos la cookie. Vamos a intentar ponerla y a ver si funciona:

![[Pasted image 20250924200224.png]]

Lab Terminado.

> [!tip] Fuerza bruta de cookies de sesión persistentes
> Las cookies de sesión persistentes (`stay-logged-in`) a menudo contienen información codificada o *hasheada* sobre el usuario y su contraseña. Si el algoritmo de *hashing* es débil (como MD5) y la contraseña es predecible, se puede realizar un ataque de fuerza bruta *offline* o *online* para descifrarla. Burp Suite Intruder, con sus opciones de "Payload Processing", es ideal para automatizar la codificación y el *hashing* de las contraseñas antes de enviarlas en la cookie.

## Lab: Offline password cracking

Esto es semejante al laboratorio anterior, pero la contraseña la tenemos que crackear offline.

Como en los laboratorios anteriores, vamos a comenzar capturando todo el tráfico desde que hacemos *login* y cómo investigamos para luego ver las peticiones y cómo se procesaron:

![[Pasted image 20250924201217.png]]

Vemos un tratamiento muy similar al del laboratorio anterior, pero si intentamos esto, veremos que no funciona.

Algo que también comenta la máquina es que tenemos una vulnerabilidad de tipo XSS en el *post*, justamente en el apartado de comentarios. Con este XSS podemos intentar robar la cookie de sesión del usuario `carlos` si es que no está protegida. Esto podemos verlo en nuestra propia sesión:

![[Pasted image 20250924201548.png]]

Vemos que la cookie estática no está protegida, por lo que podemos intentar robarla mediante un XSS de la siguiente manera:

![[Pasted image 20250924201747.png]]

Al publicar el *post*, vemos que obtenemos correctamente la cookie del usuario `carlos`:
![[Pasted image 20250924202103.png]]

![[Pasted image 20250924202119.png]]

Perfecto, ahora lo que podemos hacer es extraer ese hash y vamos a intentar hacer el *cracking* con `John the Ripper`:


![[Pasted image 20250924204135.png]]

Vemos que ya obtenemos la contraseña. Vamos a intentar terminar el laboratorio:

![[Pasted image 20250924204410.png]]

Lab Terminado.

> [!info] Cracking de contraseñas offline y XSS para robo de cookies
> El *cracking* de contraseñas *offline* es posible cuando se puede obtener un *hash* de contraseña (o una cookie que contenga un *hash*) y se intenta descifrarlo sin interactuar directamente con el servidor. Las vulnerabilidades XSS (Cross-Site Scripting) son un vector común para robar cookies de sesión, especialmente si no están protegidas con las banderas `HttpOnly` y `Secure`. Una vez obtenida la cookie, si contiene un *hash* de contraseña, herramientas como `John the Ripper` pueden usarse para intentar descifrarla contra un diccionario o mediante fuerza bruta.

## Lab: Password reset poisoning via middleware

Un *middleware* es un software que funciona como un puente, facilitando la comunicación y la gestión de datos entre diferentes entidades. En este laboratorio, es vulnerable a un restablecimiento de contraseña y el usuario `carlos` hará clic en cualquier correo que se le envíe.

Bueno, al igual que en las otras máquinas, lo que vamos a hacer es capturar el tráfico de las peticiones subyacentes y, al analizarlo, lo que vamos a encontrar es lo siguiente:
![[Pasted image 20250925181259.png]]

Vemos la petición POST que envía, creemos, el correo. Vamos a enviarla al Repeater para ver de qué se trata.

Bueno, lo que recibimos es:
![[Pasted image 20250925181417.png]]
Como vemos, un correo al cual si entramos, vamos a observar lo siguiente:
![[Pasted image 20250925181513.png]]

Vemos el panel para cambiar la contraseña y en la URL un token. Esto ya es algo extraño, pero podemos suponer que es único para nuestra cuenta.

Vamos desde el Repeater a enviarnos nuevamente el correo para ver si este token cambia:
![[Pasted image 20250925181626.png]]

![[Pasted image 20250925181644.png]]

Vemos un segundo correo. Solo leyendo las URL, ya vemos que ambas procesan un token, pero es diferente para cada una de las URL. Podemos recargar la web con el token que teníamos para ver si sigue siendo válido:

![[Pasted image 20250925181807.png]]

Se bloqueó. Podemos intentar cambiar ese token por el de la nueva URL y debería funcionar:

![[Pasted image 20250925181905.png]]

Perfecto.

En este laboratorio, haremos uso de una nueva cabecera: `X-Forwarded-Host`. Esta cabecera se usa mucho cuando las aplicaciones web pasan por *proxies* que modifican el valor de la cabecera `Host`, pero con la ayuda de `X-Forwarded-Host` se puede mantener el host de origen.

Tomando lo anterior en cuenta, vemos si con la ayuda de esta cabecera logramos modificar el dominio de la URL del enlace que se envía al momento de usar esta cabecera y así redirigir a la víctima a un lugar malicioso:

![[Pasted image 20250925182658.png]]]

![[Pasted image 20250925182716.png]]

Vemos que logramos hacer que el dominio del enlace cambie. Esto es crítico y bueno para nosotros. Podemos cambiar el dominio por el de nuestro servidor *exploit* para que, mediante una petición, nos llegue el token de `carlos`:

![[Pasted image 20250925183008.png]]

Aquí enviamos el correo malicioso a `carlos`. Vemos si nos llega algo en los logs de nuestro servidor malicioso:
![[Pasted image 20250925183053.png]]

Perfecto, tenemos el token actual válido para hacer el cambio de contraseña. Vamos a copiarlo y, al igual que al comienzo, reemplazarlo para ver si nos permite hacer el cambio de contraseña:
![[Pasted image 20250925183206.png]]

Perfecto, sí es válido. Vamos a cambiar la contraseña y vemos si se completa el laboratorio al iniciar sesión:

![[Pasted image 20250925183252.png]]

Lab Terminado.

> [!warning] Envenenamiento de restablecimiento de contraseña por `X-Forwarded-Host`
> La cabecera `X-Forwarded-Host` es utilizada por *proxies* inversos para indicar el host original de la petición. Si una aplicación web confía ciegamente en esta cabecera para construir enlaces de restablecimiento de contraseña, un atacante puede inyectar un dominio malicioso. Cuando la víctima hace clic en el enlace, el token de restablecimiento se envía al servidor del atacante, permitiéndole tomar el control de la cuenta. Esta es una vulnerabilidad crítica de inyección de cabeceras.

## Lab: Password brute-force via password change

En este laboratorio, tenemos un apartado para cambiar la contraseña que es vulnerable a ataques de fuerza bruta. El usuario al que vamos a atacar es `carlos` y nos dan la lista de posibles contraseñas.

Vamos a comenzar pasando todo el tráfico por Burp Suite para ver cómo se están procesando las peticiones:

![[Pasted image 20250925184335.png]]
Vemos este panel donde tenemos la forma de cambiar la contraseña, ingresando la actual y luego las dos nuevas. Algo que llama la atención es que si tenemos las dos contraseñas nuevas incorrectas pero la actual correcta, nos da ese mensaje de que no son iguales. Pero si tenemos las dos nuevas correctas y la actual incorrecta, vamos a tener el mensaje de que la contraseña actual es incorrecta.

Vamos a ver cómo se está procesando esto:

![[Pasted image 20250925184638.png]]

Vemos que la petición es algo simple y no se bloquea al enviar de forma continua varias peticiones. Algo que podemos hacer es cambiar el usuario de `wiener` a `carlos` y con el Intruder intentar detectar la contraseña correcta:

![[Pasted image 20250925185014.png]]

Vemos la respuesta diferente para la contraseña `ashley`. Podemos intentar iniciar sesión con esta:

![[Pasted image 20250925185406.png]]

Lab Terminado.

> [!tip] Fuerza bruta en la función de cambio de contraseña
> Algunas aplicaciones implementan protecciones de fuerza bruta solo en la pantalla de inicio de sesión, pero no en otras funciones como el cambio de contraseña. Si la función de cambio de contraseña permite múltiples intentos sin bloqueo y proporciona mensajes de error diferenciados (por ejemplo, "contraseña actual incorrecta" vs. "las nuevas contraseñas no coinciden"), un atacante puede usarla para realizar un ataque de fuerza bruta contra la contraseña actual de un usuario.

## Lab: Broken brute-force protection, multiple credentials per request

En este laboratorio, si intentamos múltiples credenciales para un usuario, tenemos protección, pero nos da una pista: "múltiples credenciales por solicitud".

Vamos a ver el panel de inicio de sesión y cómo se procesa la solicitud:
![[Pasted image 20250925191035.png]]

Vemos que la data se está enviando por JSON. Lo que podemos intentar es enviar una lista de contraseñas, de la siguiente manera:

![[Pasted image 20250925191329.png]]

Perfecto, alguna de estas contraseñas fue aceptada, por lo que vemos si podemos obtener la cookie de sesión:
![[Pasted image 20250925191652.png]]

Lab Terminado.

> [!info] Bypass de protección de fuerza bruta con múltiples credenciales por petición
> Algunas protecciones de fuerza bruta se activan por el número de peticiones fallidas. Sin embargo, si la aplicación permite enviar múltiples pares de credenciales (usuario/contraseña) dentro de una única petición (por ejemplo, en un array JSON), se puede eludir esta protección. El atacante puede probar muchas contraseñas en un solo intento, reduciendo el número de peticiones y evitando los umbrales de bloqueo.

## Lab: 2FA bypass using a brute-force attack

Tenemos acceso al usuario y contraseña, pero no tenemos acceso al segundo factor, por lo que a esto tenemos que hacerle fuerza bruta.

Vamos a ver cómo se encuentra esto:

![[Pasted image 20250925192108.png]]

Vemos esto. Vamos a intentar con algunos códigos para ver cómo se comporta:
![[Pasted image 20250925192139.png]]

Al parecer, tenemos un intento limpio y al segundo nos redirige de nuevo al inicio de sesión.

Vamos a capturar la petición para ver cómo se procesa:

![[Pasted image 20250925192400.png]]


Nosotros, por pruebas, vemos que tenemos un `token` el cual solo podemos usar 2 veces y luego, al parecer, tenemos que pasar por el proceso de inicio de sesión para conseguir otro que funcione solo 2 veces más.

Para automatizarlo de forma más precisa, no vamos a enviar dos veces, sino solo una, donde sería la del inicio de sesión, carga el `login2` (token) y luego envío un código.

Esto lo podemos hacer con ayuda de Burp Suite de la siguiente manera: primero vamos a `Settings` -> `Sessions` y en `Session handling rules`, agregamos una nueva:
![[Pasted image 20250925193355.png]]

En el *Scope* seleccionamos todas las URL y vamos a agregar en *Add* una nueva macro `Run macro`. Le damos luego en el *Add* y seleccionamos las tres peticiones que vamos a realizar:
![[Pasted image 20250925193631.png]]

Podemos, luego de darle a OK, hacer un test de la macro:
![[Pasted image 20250925193709.png]]

Perfecto, vemos cómo se va a ejecutar de manera correcta:
![[Pasted image 20250925194813.png]]

En este punto, lo único que falta es enviar el código. Entonces, vamos a darle OK y ya tenemos la macro:

![[Pasted image 20250925194931.png]]

Bueno, ahora vamos a interceptar el punto exacto donde se envía el código:

![[Pasted image 20250925195150.png]]

Esta es la configuración de cara al *payload* que vamos a usar en el Intruder. Como vemos, vamos del 0 hasta el 9999, pero al definir el "Min integer digits" a `4`, lo que hace que genere `0000` -> `0001` y así hasta el `9999`.

Ahora las macros entran en juego sin necesidad de nada, pero lo que sí tenemos que hacer es configurar en el *pool* que vaya de 1 en 1 de la siguiente manera:

![[Pasted image 20250925195418.png]]

Y ya con esto podemos intentar comenzar el ataque para ver qué sucede:

![[Pasted image 20250925195515.png]]

Perfecto, en este punto, es esperar hasta que obtengamos un código `302` (Found), que ha sido lo habitual en estos laboratorios. Esto ya es cuestión de paciencia:

![[Pasted image 20250925204516.png]]

Listo, vemos si con esto logramos resolver el laboratorio:

![[Pasted image 20250925204504.png]]

Lab Terminado.

> [!warning] Bypass de 2FA con fuerza bruta y macros de Burp Suite
> Este laboratorio demuestra un escenario avanzado de fuerza bruta contra un token 2FA con un límite de intentos por token. La solución implica el uso de macros de Burp Suite para automatizar el proceso de obtener un nuevo token 2FA (iniciando sesión y obteniendo el `login2` *request*) antes de cada par de intentos de fuerza bruta del código. Esto permite eludir la protección de intentos limitados por token, ya que se genera un token fresco para cada ciclo de ataque. La configuración del *payload* numérico y el *pool* de hilos en Intruder son clave para la eficiencia del ataque.
