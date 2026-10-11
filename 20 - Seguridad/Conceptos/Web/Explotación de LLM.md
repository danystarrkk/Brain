---
aliases:
  - API
  - LLM
tags:
  - hacking/web
  - hacking/explotacion
estado: 🟢 Terminado
---

En esta clase interactuamos con una API LLM que tiene acceso al sistema de base de datos mediante una función de depuración. A través de la conversación con el modelo, identificamos que podemos ejecutar consultas SQL arbitrarias mediante la Debug SQL API. Primero enumeramos los usuarios con un **SELECT**, confirmamos la existencia del usuario carlos y luego emitimos un **DELETE** desde el propio modelo para eliminarlo.

Esto demuestra cómo una exposición excesiva de capacidades en modelos LLM puede llevar a vulnerabilidades críticas.

## Notas Practicas

### Exploiting LLM APIs with excessive agency

Bueno en este caso tenemos que vulnerar un modelo de inteligencia artificial y eliminar al usuario carlos.

![[Pasted image 20260316162315.png]]

Como podemos observar contamos con un Live chat en este laboratorio, vamos a interactuar con el a ver que hace:
![[Pasted image 20260316162548.png]]

Como podemos observar el nos lista las funciones de la API que tiene, esto ya puede ser malo ya que nos filtra información que no debería. Como vemos debug_sql vamos a intentar que ejecute alguna sentencia:

![[Pasted image 20260316162906.png]]

como vemos no obtenemos respuesta, intentemos listar todos los campos respecto y suponiendo la columna `users` existe:

![[Pasted image 20260316163036.png]]

Como podemos observar nos permite ver los datos de `carlos`, podemos intentar eliminarlo de la siguiente manera:

![[Pasted image 20260316163216.png]]

Como vemos ya logramos eliminar el usuario `carlos` de la base  de datos.

![[Pasted image 20260316163244.png]]

Lab Terminado.

### Exploiting vulnerabilities in LLM APIs

Este laboratorio al parecer tiene una inyección de comando que se puede explotar mediante la api.

Como ya vimos en otro laboratorio vamos a primero ver como esta funcionando el LLM y ver si logramos listar las APIs:

![[Pasted image 20260316171424.png]]

Podemos observar las funciones pero vamos a pedirle que nos describa el funcionamiento de cada una para entenderlas bien:

![[Pasted image 20260316171723.png]]

Bueno vamos a preguntar por parametros que usen dichas funcinciones:

![[Pasted image 20260316172057.png]]

como logramos observar nos lista los parámetros que usa cada uno y como tenemos en realidad una bandeja de correos vamos a intentar decirle que lo use y nos subscriba a un voletin con ese correo a ver que logramos:

![[Pasted image 20260316172302.png]]

como vemos lo logramos y si vemos el ciente de email vamos a encontrar lo siguiente:

![[Pasted image 20260316172330.png]]

Bueno vemos que funciona y si lo hace, ahora nosotros no sabemos como es que por detrás lo hace pero podemos intuir cosas como que si o si tiene que a ver consultado algo con la API, el como no lo sabemos pero podemos suponiendo que el sistema que lo hace es mediante la consola intentamos inyectar un comando en el correo:

![[Pasted image 20260316172633.png]]

como observamos ahora lo que intentamos es ejecutar un comando a nivel de consola a ver si esta nos responde y como lo hacemos es viendo que nos llegara al correo, tomemos en cuenta que nos llega cualquier correo asociado al dominio por lo que no importa si se modifica el usuario:

![[Pasted image 20260316172743.png]]

Observamos que efectivamente podemos ejecutar comandos y como tenemos que eliminar el archivo `/home/carlos/morale` vamos a inyectar el comando para que se efectue:

![[Pasted image 20260316172913.png]]

Bueno vemos que le enviamos y con esto completamos el laboratorio:

![[Pasted image 20260316172936.png]]


### Indirect prompt injection

Este laboratorio es vulnerable a una inyección del prompt de forma indirecta, el objetivo es eliminar al usuario carlos.

Comenzamos registrandonos con el correo que nos proporcionan:

![[Pasted image 20260316174849.png]]

ya registrados podemos activar la cuenta con el correo que nos llega y vamos iniciar sesión:

![[Pasted image 20260316174945.png]]

Ya con esto vamos a dirigirnos al Live Chat:

![[Pasted image 20260316175047.png]]

como siempre logramos que nos liste las APIs que puede usar, podemos preguntar por lo que necesita para ejecutar la primera que nos permitiría en un comienzo eliminar al usuario `carlos`  que es el objetivo, pero primero intentemos ver si puede hacerlo y para esto vamos a eliminar nuestra cuenta:
![[Pasted image 20260316175950.png]]

Como vemos logramos hacer que elimine nuestra cuenta, en este punto podemos intentar eliminar la cuenta de carlos de forma directa pero no lo puede hacer.

![[Pasted image 20260316180430.png]]

Podemos observar como si nos da información sobre un producto, ahora lo que podemos pensar es ver si podemos de alguna forma influir mediante los comentarios del producto en el resultado de la IA la prueba es poner en el producto que ya no quedan productos en stok he intentar que la IA nos responda al preguntarle que no tiene productos disponibles:

![[Pasted image 20260316180835.png]]

ya con el comentario listo veamos si se puede influir en la IA.

En este producto no se logro influir ya que la IA logra verificarlo por completo, ahora se intento con diferentes productos esperando obtener uno bueno:

![[Pasted image 20260316181909.png]]

Como logramos observar en esta ocasión si se logro influir para que la IA nos de como producto no disponible o agotado mediante las reseñas de la web.

Ahora tomando en cuenta eso vamos primero a influir primero para intentar que elimine nuestra cuenta pero desde la reseña, pero antes tenemos que comprender un concepto.

Muchos de los lenguajes implementados suelen implementar delimitadores de entrada en los prompts, esto les ayuda diferenciar la entrada, la respuesta generada el usuario que lo mando y mas. Su estructura podría ser:

![[Pasted image 20260316182247.png]]

Como observamos no es mas que bloques y tomando esto en cuenta podemos en el mismo producto realizar una prueba:

![[Pasted image 20260316182504.png]]

Podemos intentar enviar esta reseña y comenzar a ver si la IA nos interpreta esto:

![[Pasted image 20260316182745.png]]

logramos ver solo la reseña y no en si la inyección esto puede deberse a que si funciono y para comprobarlo es cuention de intentar movernos y si nos saca es porque la cuenta ya no existe:

![[Pasted image 20260316182907.png]]

como vemos no nos elimino la cuenta pero el echo de que no nos enviara el comentario completo quiere decir que posiblemente interpreto la estructura, en este caso vamos a comenzar a jugar con el comentario hasta lograr el objetivo:

![[Pasted image 20260316183127.png]]

Vemos si esto funciona pero no en realidad no lo hace.

Vamos a intentar otra cosa:

![[Pasted image 20260316183421.png]]

como podemos observar tenemos otra inyección vemos si esta funciona:

![[Pasted image 20260316184453.png]]

Luego de varias pruebas he intentos fallidos logramos ingresar por fin y si vemos o tratamos de movernos en la web nos saca porque el usuario no existe.

Ahora lo que vamos a hacer es usando el prompt que funciono a dejar un nuevo comentario en la review de la chaqueta :

![[Pasted image 20260316184903.png]]

con el comentario echo es cuestión de esperar a que el usuario converse con la IA y ver si es posible la inyección.

Al parecer no termina de gustarle la inyección porque se hace la petición y no funciona, por lo que podemos intentar ser mucho mas precisos:

![[Pasted image 20260316185254.png]]

Lab Terminado.

### Lab: Exploiting insecure output handling in LLMs

En este laboratorio vamos a tener que vulnerar el output del LLM para realizar un XSS y vamos a tener que intentar ingresar una inyección de prompt indirecta que elimine al usuario `carlos`.


Comenzamos registrando un usuario:

![[Pasted image 20260316195533.png]]

Vamos directo al chat a ver que podemos hacer.

![[Pasted image 20260316195657.png]]

Conociendo que tenemos un XSS vamos a intentar una inyección sencilla:

![[Pasted image 20260316195819.png]]

como podemos ver esto si se puede.


Bueno lo que vamos a hacer es probar con otros productos para intentar la inyección mediante la carga de la información de los mismos:

![[Pasted image 20260316200037.png]]

Con el comentario listo, vemos si logramos acontecer algo desde el chat:

![[Pasted image 20260316200210.png]]

Como podemos ver al parecer tiene alguna seguridad que no permite hacer la inyección del todo.

Bueno lo que podemos hacer es intentar llamar al botón de eliminar que tenemos en la home page del usuario mediante un `iframe`, esto nos ayudara a eliminar su cuenta, vamos a crear un comentario de la siguiente manera:

![[Pasted image 20260316200909.png]]

ya con esto vamos a probar si en el chat esto es valido:

![[Pasted image 20260316201023.png]]

Como podemos observar conservamos la reseña no valida por lo que vamos a intentar un variante donde nuestro objetivo sera aprovecharnos del contexto:

![[Pasted image 20260316201234.png]]

Bueno vamos a volver a intentarlo y no funciona vamos con otra variante:

![[Pasted image 20260316201710.png]]

Veremos lo siguiente:

![[Pasted image 20260316201759.png]]

El `iframe` carga y nos elimina en realidad ya el usuario por lo que vamos a registrarnos y vamos a ponerlo ahora en el de la chaqueta veamos si logramos eliminar su cuenta:

![[Pasted image 20260316202438.png]]

y si esperamos vamos a ver:

![[Pasted image 20260316202455.png]]

Lab Terminado.

