-----------------------

## PortSwigger

### Lab: Unprotected admin functionality

El objetivo del lab es eliminar al usuario `carlos`.

La verdad es que no tenemos mayor información por lo que vamos a usar [[Nmap]] par enumerar la web usando el script de `http-enum`
```bash
nmap --script http-enum -p443 0a8b0015038a9503842a04ee00f400bd.web-security-academy.net
```

![[Pasted image 20250923195359.png]]

no vemos que el nmap funcione pero esto seri algo que podemos hacer en un principio.

En este punto podemos buscar por rutas que nosotros conocemos que seria algo como `robots.txt`:
![[Pasted image 20250923195447.png]]

Vemos que nos dice que no esta permitido que se indexe el panel de administración que es `/administrator-panel` pero nos lo esta diciendo en seco por lo que podemos intentar ingresar:
![[Pasted image 20250923195623.png]]

Perfecto, es cuestión de eliminar al usuario carlos y ya estaría:

![[Pasted image 20250923195658.png]]

Lab Terminado.


### Lab: Unprotected admin functionality with unpredictable URL


En este laboratorio tenemos una funcionalidad de admin no protegida donde tenemos una url no predecible (no fuerza bruta).

Bueno podemos intentar ver el código de la web a ver que encontramos:
![[Pasted image 20250923195912.png]]

vemos un código en JavaScript el cual nos dice ya la url que es del admin `admin-xj19s8`:

![[Pasted image 20250923200006.png]]

Pues esto a funcionando, vamos terminar el lab:
![[Pasted image 20250923200038.png]]

Lab Terminado.


### Lab: User role controlled by request parameter

al parecer nos habla de que el usuario tiene el rol asignado mediante un parámetro en la solicitud, se tiene el usuario `wiener:peater` y el panel `/admin`.

Bueno como nos habla de un parámetro en la solicitud lo que podemos hacer es intentar capturar la petición y analizarla:
![[Pasted image 20250923200336.png]]

vemos que en efecto tenemos un parámetro en la solicitud, pasémoslo a true y vemos que sucede al enviar la solicitud:

![[Pasted image 20250923200506.png]]

pasamos, ya con esto podemos terminar el lab:

![[Pasted image 20250923200546.png]]

![[Pasted image 20250923200554.png]]

Lab Terminado.


### Lab: User role can be modified in user profile

Al parecer en nuestro panel de usuario podemos modificar el rol, además en la maquina solo se puede usar el panel de admin si eres un usuario con `roleid:2`.

analizando la petición del campo de email vemos lo siguiente:
![[Pasted image 20250923201453.png]]

observamos que pasamos información que le enviamos como `email` pero nos responde con mas información de la que debería y vemos el campo `roleid` podemos intentar modificar esto de la siguiente manera para que el campo `roleid` cambie:

![[Pasted image 20250923201620.png]]

![[Pasted image 20250923201650.png]]

vemos que si nos cambio y ya podemos acceder al panel de admin, vamos a completar el lab:

![[Pasted image 20250923201716.png]]

Lab Terminado.

### Lab: User ID controlled by request parameter 

en este lab al parecer controlamos por un parámetro en la petición el id de usuario lo que podría realizar un salto a otro usuario:

![[Pasted image 20250923201934.png]]

vemos que en la web tenemos el parámetro id el cual apunta a wiener, podemos intentar cambiarlo a administrator a ver que sucede:
![[Pasted image 20250923202019.png]]

logramos cambiar el usuario con esto, vamos a completar el lab:

![[Pasted image 20250923202145.png]]

Lab Terminado.


### Lab: User ID controlled by request parameter, with unpredictable user IDs 

En este caso la web utiliza IDs no predecibles por lo que debemos encontrar la forma de encontrar el del usuario carlos.

Bueno en este caso nosotros tenemos que saber que cuando se publican cosas, en ocasiones logramos ver información de estos mismos filtrada, vamos a ver si encontramos algo de carlos:

![[Pasted image 20250923203143.png]]

parece que conseguimos el identificador de carlos.

Con esto podemos intentar saltar al usuario `carlos`:
![[Pasted image 20250923203252.png]]

Ya con esto podemos terminar el laboratorio:

![[Pasted image 20250923203318.png]]

Lab Terminado.


### Lab: User ID controlled by request parameter with data leakage in redirect 

Aquí nos habla de que tenemos información fugada en la data de la respuesta.

![[Pasted image 20250923203931.png]]

Vamos a ver la respuesta y el redirect de esta a ver que sucede, por contexto estamos mediante el parámetro de la url cambiando de `wiener` a `carlos` y eso capturamos:
![[Pasted image 20250923204132.png]]

Vemos un `302 found` de un redirect, el problema es que vemos mucha información para ser un redirect, vamos a ver que es lo que nos da:
![[Pasted image 20250923204248.png]]

vemos que ya tenemos el API-Key de `carlos` por lo que vamos a usarlos para terminar el lab:

![[Pasted image 20250923204404.png]]

Lab Terminado.


### 




En esta web si cambiamos el usuario de wiener a `carlos` vamos a ver lo siguiente:
![[Pasted image 20250923205014.png]]

vemos que nos pone la contraseña y aunque la veamos así modificando el html podríamos verla:
![[Pasted image 20250923205321.png]]

sacamos la password del usuario administrator y con esto podemos ingresar como ese usuario:

![[Pasted image 20250923205442.png]]

Perfecto terminemos el laboratorio.

![[Pasted image 20250923205505.png]]

Lab Terminado.


### Lab: Insecure direct object references

bueno vemos algo como parte del chat lo cual tramita diferentes opciones, vamos a ver una en especial que llama nuestra atención:

![[Pasted image 20250923211336.png]]

vamos a ver que tramita:

![[Pasted image 20250923211539.png]]

vemos información que se tramita en esta petición, vamos a intentar ver que pasa si pasamos valores de la siguiente menera:


![[Pasted image 20250923211735.png]]

nos movilizamos entre valores y vemos otra información donde encontramos una password que es del usuario carlos:

![[Pasted image 20250923212004.png]]

Lab Terminada.

### Lab: URL-based access control can be circumvented

Tenemos restricciones en el front-end que no permiten el acceso a `/admin`, por lo que leemos tenemos una cabecera `X-Original-URL` la cual se emplea para evitar acceso de forma externa a la web.


Si nos habla de una cabecera lo mas probable es que esta se pueda modificar, vamos a intentarlo:


![[Pasted image 20250923212838.png]]

vemos que tenemos el acceso denegado en un comienzo, podemos intentar agregar al cabecera ya que no esta a ver si nos la reescribe su valor:
![[Pasted image 20250923212958.png]]

logramos pasar, vamos a completar el lab:
![[Pasted image 20250923213152.png]]

al momento de eliminar modificamos un poco ya que en la cabecera no se puede ingresar parámetros pero ya funciono:
![[Pasted image 20250923213258.png]]

Lab Terminado.


### Lab: Method-based access control can be circumvented

Este laboratorio es sencillo lo que sucede es que tenemos un panel de administración `admin-roles` al cual no podemos acceder pero que se le pasa dos valores `username` y `action` esto se tramita por Post, podemos intentar cambiar esto a GET y que no necesitemos ser un usuario de grandes privilegios:
![[Pasted image 20250923214016.png]]

logramos darle permisos de administrador a carlos:
![[Pasted image 20250923214407.png]]

Lab Terminado.


### Lab: Multi-step process with no access control on one step 

En este laboratorio tenemos un proceso de multiples pasos sin control de acceso en uno de los pasos, el objetivo es de nuevo obtener privilegios de administrador.

de igual forma que en la anterior nosotros podemos lograr asignar privilegios de administrador de la siguiente manera:
![[Pasted image 20250923215120.png]]

Conociendo la ruta y algunos parámetros podemos ver como logramos asignar privilegios a wiener:
![[Pasted image 20250923215200.png]]

Lab Terminada.


### Lab: Referer-based access control 

En este laboratorio vamos a volver a ver algo del referrer, el objetivo sigue siendo el mismo conseguir permisos de administrador en el usuario wiener.

de igual forma configurando la cabecera `Referer` podemos realizar solicitudes como otro usuario:
![[Pasted image 20250923215953.png]]

![[Pasted image 20250923220011.png]]

Lab Terminado.
