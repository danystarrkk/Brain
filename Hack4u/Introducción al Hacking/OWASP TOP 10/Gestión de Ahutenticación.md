------------

## PortSwigger


### Lab: Username enumeration via different responses

Lo que tenemos que hacer para completar el laboratorio es encontrar usuarios validos en el sistema y luego hacer fuerza bruta para encontrar la contraseña.


Bueno en este laboratorio nos dan una lista de usuario y contraseñas.

En este punto si intentamos cualquier cosa para iniciar sesión vamos a ver lo siguiente:
![[Pasted image 20250924132941.png]]

vemos que solo nos dice `usuario invalido`, podemos usar esto para encontrar algún usuario valido, vamos a capturar la petición:
![[Pasted image 20250924133123.png]]

perfecto vamos a enviar esto al intruder para intentar identificar posibles usuario validos, vamos a usar la lista que nos dan para facilitar las cosas:
![[Pasted image 20250924133239.png]]

filtramos por la longitud de la respuesta y vemos que para el usuario `athena` nos dice otra cosa `password incorrecto`, con un usuario valido lo que podemos hacer es lo mismo pero con la lista de contraseñas:

![[Pasted image 20250924133440.png]]

perfecto con la contraseña `robert` logramos un follow redirect por lo que es correcto, vamos a iniciar sesión:
![[Pasted image 20250924133542.png]]

Lab Terminado.

### Lab: 2FA simple bypass

En este laboratorio tenemos una factor de doble verificación y tenemos que burlarlo, tenemos las credenciales de la victima que son `carlos:montoya`.

Vamos a comenzar viendo como es que esta configurado esto:
![[Pasted image 20250924134210.png]]

luego de ingresar las credenciales nos lleva a este panel, si revisamos la solicitud cuando ingresamos el código este realiza un redirect:
![[Pasted image 20250924134423.png]]

esa es la ruta a la que la web redirige luego de proporcionar el código correcto, lo que podemos hacer es intentar ver si una vez que estamos en el punto de introducir el código, al cambiar a `/my-account` puede ser que no sea ya necesario el código, intentemoslo:
![[Pasted image 20250924134705.png]]

Nos permite pasar de forma correcta una vez que estamos en el punto de introducir el código

Lab Terminado.

### Lab: Password reset broken logic

En este laboratorio buscamos acceder como el usuario carlos, nos hablan de una reset lógico roto por lo que primero vamos a ingresar a la web iniciar sesión y buscar este reset roto.


Parece ser por como se describe un problema que corrompe de alguna forma el manera en la que nosotros recuperamos una contraseña, podemos intentar realizar un cambio y ver como se tramita:
![[Pasted image 20250924135802.png]]

nos pidieron un correo y nos mandaron el siguiente link, vamos a interceptar directamente la dirección de este link y vemos que es lo que se tramita.
![[Pasted image 20250924140223.png]]

lo hemos usado pero si recargamos al parecer podemos reutilizar pero lo que estamos buscando es alguna forma en la que se este usando esto, podemos interceptar esto al enviar a ver como se esta tramitando:
![[Pasted image 20250924140359.png]]

vemos que de forma literal se esta tramitando para el usuario `wiener` podemos modificarlo para ver que pasa si cambiamos el username a `carlos`:
![[Pasted image 20250924140511.png]]

Pues no valida al parecer nada ya que vemos que nos deja avanzar:
![[Pasted image 20250924140553.png]]

Cambiamos correctamente las contraseñas.

Lab Terminado.



### Lab: Username enumeration via subtly different responses

En este punto volvemos a realizar ataques de fuerza bruta, lo que vamos a hacer es una enumeración de usuario y luego descubrir la contraseña, esto con las listas que nos dan, el como esto cambia dependiendo si es valido o no es sutil por lo que debemos descubrirlo.


El panel de login vemos lo siguiente:
![[Pasted image 20250924141225.png]]

ya no nos lo pone tan fácil debido a que nos dice usuario o contraseña invalida.

Bueno vamos a analizar un poco mas a fondo, intentemos capturar la petición y ver que obtenemos:
![[Pasted image 20250924141523.png]]

bueno vemos lo mismo pero como no tenemos un usuario valido no tenemos con que comparar. Vamos a intentar el mismo ataque que la vez anterior, vamos a ver si obtenemos algo por la longitud, código de estado algo que nos sirva.

Vamos a realizar en este caso nuevamente el ataque de fuerza bruta con el cual vamos a intentar encontrar alguna diferencia en la forma con la cual el usuario correcto responde:
![[Pasted image 20250924143001.png]]

vamos a usar Grep -Extract para ver el mensaje por el que vamos a filtrar y ver si sufre cambios:
![[Pasted image 20250924143103.png]]

con esto vamos a filtrar por esa cadena:
![[Pasted image 20250924143127.png]]

vemos que al usuario `ae` a comparación de otros le falta el punto final, podemos ahora intentar un ataque de fuerza bruta para este usuario a ver que obtenemos:
![[Pasted image 20250924143255.png]]

perfecto vemos que para una de las contraseñas se aplica un follow redirect, con esto vamos a iniciar sesión:
![[Pasted image 20250924143348.png]]

Lab Terminado.


### Lab: Username enumeration via response timing

En este laboratorio por lo que se entiende tenemos primero que enumerar ciertos usuarios y luego encontrar la contraseña del usuario valido y todo esto basado en el tiempo de respuesta.


Bueno lo primero que podemos hacer es intentar iniciar sesión pero veremos que después de algunos intentos se bloquea y envía otro tipo de mensaje:
![[Pasted image 20250924172657.png]]


Esto nos puede detener cuando hacemos un ataque de fuerza bruta ya que en diferentes sitios se puede bloquear.

Nosotros para evitar que esto suceda lo que podemos hacer es arrastrar ciertas cabeceras para tratar de hacerle pensar a la web que vienen de otro sitio o de distinto origen, lo que evitara este problema.

Vamos a usar la cabecera de `X-Forwarded-For` el cual es una cabecera que contiene la dirección IP del cliente que realizo la solicitud:

![[Pasted image 20250924173117.png]]

vemos que ya funciona, en pocas solo volvemos a tener 3 intentos.

Las respuestas dependiendo si esta bien o mal en ocasiones su tiempo de respuesta puede aumentar o disminuir, lo que vamos a hacer es aumentar el tamaño de la contraseña de forma masiva y ver que sucede:
![[Pasted image 20250924173402.png]]

lo que descubrimos es que con usuarios validos la web tarda cierto tiempo en responder, esto puede ser porque al ser un usuario valido y comenzar a compara la contraseña esta se demore, en este punto vamos a enviar al intruder para hacer el ataque:
![[Pasted image 20250924173651.png]]

en este caso la configuración cambia ya que tenemos que configurar dos payloads los cuales cambien, esto para evitar el bloqueo, vemos que sucede:
![[Pasted image 20250924173951.png]]

vemos que el usuario `adkit` parece ser valido, ya que el valor en Response Completed es el valor en milisegundos que demoro la petición,  para ver ese campo que usualmente no se ve vamos a los 3 puntitos y lo activamos. Ahora intentemos con ese usuario un ataque a la contraseña a ver si funciona:
![[Pasted image 20250924174436.png]]

encontramos la contraseña vamos a ver si esto completa el lab:
![[Pasted image 20250924174617.png]]

Lab Terminado.


### Lab: Broken brute-force protection, IP block

En este caso tenemos un listado de contraseñas para el usuario carlos, pero al parecer lo que sucede es que luego de varios intentos la IP sea boqueada, vamos a intentar eludir esto.

Vamos a ver que tiene este inicio de sesión:
![[Pasted image 20250924174927.png]]

vemos que si es vulnerable ya que vemos que nos lista, si prodríamos listar usuario pero no es punto, luego de unos intentes sucede lo siguiente:

![[Pasted image 20250924175033.png]]

nos bloquea cada un minuto. En este caso lo que vamos a intentar es que vamos a hacer dos intentos de carlos incorrectos, y luego uno correcto podemos refrescar el contador, vamos a intentarlo:

![[Pasted image 20250924175244.png]]

esto parece gustarle, vamos a seguir usando el intruder pero necesitamos crear un diccionario de contraseñas y usuario el cual realize esta acción y para crearlo vamos a hacerlo con python:

generate users:
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

y eso lo metemos en un archivo users, ahora vamos con las contraseñas donde cada 2 contraseñas necesitamos usar `peter` para wiener:

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

Perfecto, esto igual vamos a ingresarlo en un archivo `password`.

Bueno con estos dos diccionarios ya vamos a intentar el ataque por medio del intruder:
![[Pasted image 20250924181757.png]]

listo podemos intentar el ataque pero antes tenemos que configurar el siguiente parámetro para que el orden de las peticiones sea correcto:
![[Pasted image 20250924182107.png]]

Perfecto enviamos y vemos lo siguiente:
![[Pasted image 20250924182233.png]]

vemos que la contraseña es `harley`, perfecto veamos si con esto terminamos el lab:

![[Pasted image 20250924182329.png]]

Lab Terminado.


### Lab: Username enumeration via account lock

En este caso al parecer vamos a tener que hacer algún bloqueo de cuenta para identificar cuando un usuario es valido.

Comenzamos capturando una petición para ver por detrás como es que se esta capturando la solicitud:
![[Pasted image 20250924183805.png]]

bueno, vamos a enviarlo al intruder y mediante un ataque de tipo cluster bomb vamos a intentar obtener el usuario válido:
![[Pasted image 20250924184112.png]]

bueno filtramos también en el Grep-Match por una cadena concreta y vemos que se bloquea el usuario `adsl`, esto ya nos dice que este usuario es valido, vamos a intentar un segundo ataque pero solo para obtener su contraseña:

![[Pasted image 20250924184503.png]]

en este caso podemos observar como el ataque no es del todo efectivo ya que seguimos bloqueados pero debido a un fallo lógico supongo lo que vemos es que la contraseña es valida pero por el bloqueo no podemos entrar, vamos a esperar el minuto y a intentarlo luego para ver que sucede:

![[Pasted image 20250924184639.png]]

Lab Terminado.


### Lab: 2FA broken logic

En este punto lo que vamos a intentar es romper la lógica del segundo factor de autentificación.

Bueno vamos mientras estamos en el proxy de BurpSuite, esto nos ayudara a capturar las diferentes solicitudes incluyendo el factor de doble paso, para luego extraerlo al repeater:
![[Pasted image 20250924191541.png]]

bueno ya podemos ver como las peticiones se están ejecutando, se intento justo en login2 cambiar eso de wiener a carlos pero no funciona, no nos envía a nostros.

Ahora vamos a seguir viendo mas peticiones:

![[Pasted image 20250924191922.png]]

encontramos la petición con la cual es esta intentando ingresar como carlos y además vemos que por alguna razón nos lo permite de multiples veces, lo que podemos hacer es enviar una solicitud con la request anterior para que se envíe un código y con esta que tenemos, enviarla al intruder e intentar todas las combinaciones posibles:

Vamos a crear primero esto con seq de la siguiente manera:
```bash
seq -w 1 9999 > password
```

![[Pasted image 20250924192333.png]]

con esto ya tenemos vamos a intentar el ataque con el intruder:

![[Pasted image 20250924192619.png]]

vemos un `302` así que puede que sea valido, vamos a intentarlo:

![[Pasted image 20250924193126.png]]

Lab Terminado.

### Lab: Brute-forcing a stay-logged-in cookie

Por alguna razón este laboratorio tiene una cookie persistente que mantiene a los usuario en sesión así ellos cierren el navegador.

Bueno tenemos que intentar ingresar somo `carlos`.

Vamos a comenzar como siempre capturando todo el trafico mientras probamos un poco la web con credenciales validas:

![[Pasted image 20250924193954.png]]

bueno analizando vemos que una vez tenemos iniciado sesión tenemos una cookie `stay-logged-in` que se arrastra, podemos ver que el burp al seleccionarlo lo detecta como base64 así que mejor para maniobrar vamos intentar desde consola:

![[Pasted image 20250924194129.png]]

vemos el `winer:...` podemos seleccionar eso y ver su longitud:

![[Pasted image 20250924194220.png]]

vemos que tiene una longitud de 32 bytes lo que nos da indicios de ser md5, podemos usar algún software que nos permite ver el tipo de hash para confirmar:
![[Pasted image 20250924194339.png]]

vemos que si es md5, podemos tratar de ver si talvez es la contraseña:
![[Pasted image 20250924194541.png]]

perfecto, es la contraseña en md5 con esto y teniendo la lista de posibles passwords, automatizar el procedimiento de crear la cookie y luego que intente con esa desde el intruder, entonces a esa petición mandémosla al intruder:
![[Pasted image 20250924194836.png]]

podemos ver ya en el intruder el `payload processing`, luego vamos a agregar uno:
![[Pasted image 20250924194952.png]]
le decimos que primero trasforme las contraseñas a md5.

![[Pasted image 20250924195043.png]]
ahora le agregamos antes antes de eso el `calrlos:`.

![[Pasted image 20250924195156.png]]
por ultimo todo en base64 y vamos a ver que tal funciona.

![[Pasted image 20250924200147.png]]

perfecto ya tenemos la cookie, vamos a intentar ponerla y a ver si funciona:

![[Pasted image 20250924200224.png]]

Lab Terminado.


### Lab: Offline password cracking


Esto es semejante al laboratorio anterior pero la contraseña la tenemos que hacer offline.

Como en los anteriores labs vamos a comenzar capturando todo el trafico desde que hacemos login y el como investigamos para luego ver las peticiones y el como se tramitaron:

![[Pasted image 20250924201217.png]]

vemos un tratamiento muy similar al de la anterior pero si intentamos esto vamos a ver que no funciona.

algo que también comenta la maquina es que tenemos una vulnerabilidad de tipo `XSS` en el post, justamente en el apartado de comentario, con este XSS podemos intentar robar la cookie de sesión del usuario carlos si es que no esta protegida, esto podemos verlo en nuestra propia sesión:

![[Pasted image 20250924201548.png]]

vemos que la cookie estática no  esta protegida por lo que podemos intentar robarla mediante un XSS de la siguiente manera:

![[Pasted image 20250924201747.png]]

al publicar el post vemos que obtenemos correctamente la cookie del usuario carlos:
![[Pasted image 20250924202103.png]]

![[Pasted image 20250924202119.png]]

perfecto ahora lo que podemos hacer es extraer ese hash y vamos a intentar hacer el cracking con `John`:


![[Pasted image 20250924204135.png]]

vemos que ya obtenemos la contraseña, vamos a intentar terminar el lab:

![[Pasted image 20250924204410.png]]

Lab Terminado.


### Lab: Password reset poisoning via middleware

Un middleware es un software que funciona como un puente done facilita la comunicación y la gestión de datos entre diferentes entidades, en este lab es vulnerable a un reseteo de contraseña y el usuario carlos va a hacer click en cualquier correo que se le envíe.


Bueno, al igual que en las otras maquinas lo que vamos a hacer es capturar el tráfico que das peticiones por detrás  y al anlalizarlo lo que vamos a encontrar es lo siguiente:
![[Pasted image 20250925181259.png]]

vemos la petición por post que envía queremos creer el correo, vamos a mandarla al repeater a ver de que se trata.

bueno lo que recibimos es:
![[Pasted image 20250925181417.png]]
como vemos un correo al cual si entramos vamos a observar lo siguiente:
![[Pasted image 20250925181513.png]]

vemos el panel para cambiar la contraseña y en la url un token, esto ya es algo extraño pero podemos suponer que es uno único para nuestra cuenta.

Vamos desde el repeater a enviarnos nuevamente el correo a ver si este token cambia:
![[Pasted image 20250925181626.png]]

![[Pasted image 20250925181644.png]]

vemos un segundo correo, solo leyendo las url ya vemos que tramitan un token las dos pero es diferente para cada una de las url, podemos recargar la web con el token que teníamos a ver si sigue siendo válido:

![[Pasted image 20250925181807.png]]

se bloqueo, podemos intentar cambiar ese token por el de la nueva url y debería funcionar:

![[Pasted image 20250925181905.png]]

Perfecto.

En este lab vamos a hacer uso de una nueva cabecera `X-Forwarded-Host`, esta cabecera se usa mucho cuando las webs pasan por proxys los cuales modifican el valor de la cabecera `Host`, pero donde con ayuda de `X-Forwarded-Host` se puede mantener el host de origen.

Tomando lo anterior en cuenta vemos si de con ayuda de esta cabecera logramos modificar el dominio de la url de link que se envía al momento de hacer uso de esta cabecera y así redirigir a la victima a una lugar malicioso:

![[Pasted image 20250925182658.png]]]

![[Pasted image 20250925182716.png]]

Vemos que logramos hacer que el dominio del link cambie, esto es critico y bueno para nosotros podemos cambiar el dominio por el de nuestro server exploit para que mediante una petición nos llegue el token de carlos:

![[Pasted image 20250925183008.png]]

aquí enviamos el correo malicioso a carlos, vemos si nos llega algo en nuestros logs del servidor malicioso:
![[Pasted image 20250925183053.png]]

Perfecto, tenemos el actual token valido para hacer el cambio de contraseña, vamos a copiarlo y al igual que al comienzo remplazarlo a ver si nos permite hacer el cambio de contraseña:
![[Pasted image 20250925183206.png]]

perfecto si es valido, vamos a cambiar la contraseña y vemos si se completa el lab al iniciar sesión:

![[Pasted image 20250925183252.png]]

Lab Terminado.


### Lab: Password brute-force via password change

Bueno en este lab tenemos un apartado par cambiar la contraseña donde es vulnerable a ataques de fuerza bruta, el usuario al que vamos a atacar es `carlos` y nos dan la lista de posibles contraseñas.

Vamos a comenzar pasando todo el trafico por el BurpSuite para ver como se están tramitando las peticiones:

![[Pasted image 20250925184335.png]]
vemos este panel donde tenemos la forma de cambiar la contraseña ingresando la actual y luego las dos nueva, pero algo que llama la atención es que si tenemos las dos contraseñas incorrectas pero la actual correcta nos da ese mensaje de que no son iguales, pero si tenemos las dos correctas y la actual incorrecta vamos a tener que las contraseñas actuales son incorrectas.

Vamos a ver como es que se esta tramitando esto:

![[Pasted image 20250925184638.png]]

vemos que la petición es algo simple y no se bloquea al enviar de forma continua varias peticiones, algo que podemos hacer es cambiar el usuario de `wiener` a `carlos` y con el intruder intentar detectar la contraseña correcta:

![[Pasted image 20250925185014.png]]

vemos la respuesta diferente para la contraseña `ashley`, podemos intentar iniciar sesión con esta:

![[Pasted image 20250925185406.png]]

Lab Terminado.


### Lab: Broken brute-force protection, multiple credentials per request

En este laboratorio si intentamos multiples credenciales para un usuario tenemos protección, pero nos da una pista y es que dice multiples credenciales por solicitud.

Vamos a ver el panel de login y vemos como se tramita la solicitud:
![[Pasted image 20250925191035.png]]

vemos que la data se esta enviando por JSON, podemos lo que podemos intentar es enviar una lista de contraseñas, de la siguiente manera:

![[Pasted image 20250925191329.png]]

Perfecto, alguna de estas contraseñas le gusto por lo que vemos si podemos obtener la cookie de sesión:
![[Pasted image 20250925191652.png]]

Lab Terminado.

### Lab: 2FA bypass using a brute-force attack

Tenemos acceso al usuario y contraseña pero no tenemos acceso al segundo factor por lo que a esto tenemos que hacerle fuerza bruta.

Vamos a ver como se encuentra esto:

![[Pasted image 20250925192108.png]]

vemos esto, vamos a intentar con algunos códigos a ver como se comporta:
![[Pasted image 20250925192139.png]]

al parecer tenemos un intento limpio y al segundo nos manda de nuevo al login.

Vamos a capturar la petición para ver como se tramita:

![[Pasted image 20250925192400.png]]

nosotros por pruebas vemos que tenemos un `token` el cual solo podemos usar 2 veces y luego al parecer tenemos que pasar por el proceso de login para conseguir otro que funcione solo 2 veces mas.

Bueno el objetivo es que para cada una de las solicitudes vamos a llamarlas de fuerza bruta lo que haga antes es pasar por el proceso de login enviando `usuario` y `contraseña`, luego cargar una vez el `login2` y con token ya valido enviar las dos solicitudes de fuerza bruta. Para automatizarla de forma mas precisa no vamos enviar dos veces si no solo uno donde seria la del login, carga el long2 (token) y luego envío un codigo.

Esto lo podemos hacer con ayuda del BurpSuite de la siguiente manera, primero vamos a settings -> sessions y en sessions handling rules, agregamos una nueva:
![[Pasted image 20250925193355.png]]

En el Scope seleccionamos all url y vamos a agregar en add una nueva macro `run macro`, le damos luego en el add y seleccionamos las tres peticiones que vamos a realizar:
![[Pasted image 20250925193631.png]]

Podemos luego de darle a ok hacer un test de la macro:
![[Pasted image 20250925193709.png]]

Perfecto vemos como ese va a ejecutar de manera correcta:
![[Pasted image 20250925194813.png]]

En este punto lo único que falta ya es enviar el código, entonces vamos a darle ok y ya tenemos la macro:

![[Pasted image 20250925194931.png]]

Bueno ahora vamos a interceptar el punto exacto donde se envía el código:

![[Pasted image 20250925195150.png]]

esta es la configuración de cara al payload que vamos a usar en el intruder, como vemos vamos del 0 hasta el 9999 pero al definir el Min integer digits a `4` lo que hacemos es que haga `0000` -> `0001` y así hasta el `9999`.

Ahora las macros entran en cuenta sin necesidad de nada pero lo que si tenemos que hacer es configurar e nel pool que vaya de 1 en una de la siguiente manera:

![[Pasted image 20250925195418.png]]

y ya con esto podemos intentar comenzar el ataque a ver que secede:

![[Pasted image 20250925195515.png]]

perfecto, en este punto es esperar hasta que obtengamos un código 302 que ha sido lo habitual en estos laboratorios, esto ya es cuestión de paciencia:

![[Pasted image 20250925204516.png]]

Listo, vemos si con esto logramos resolver el lab:

![[Pasted image 20250925204504.png]]

Lab Terminado.