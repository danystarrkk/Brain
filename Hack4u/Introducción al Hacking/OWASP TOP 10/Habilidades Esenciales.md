-------------

## Discovering vulnerabilities quickly with targeted scanning


En realidad no es mas que una funcionalidad de Burp Suite para intentar encontrar vulnerabilidades mucho mas rapido.

Primero ingresamos al laboratorio y capturamos la web:

![[Pasted image 20260223212852.png]]

damos clic derecho y en `scan` :

![[Pasted image 20260223212958.png]]

seleccionamos tal y como en la imagen y damos scan:

![[Pasted image 20260223213113.png]]

seleccionamos eso y scan.
![[Pasted image 20260223213156.png]]

comienza a probar diferentes cosas y es cuestión de esperar.

![[Pasted image 20260223213542.png]]

observamos una vulnerabilidad de tipo `External service interaction` un XXE:

![[Pasted image 20260223213557.png]]

Vemos que se están jugando con entidades externas, recordando los conceptos y con ayuda de algún payload podemos hacer que se impriman recursos internos de la siguiente manera:

![[Pasted image 20260223214102.png]]

## Scanning non-standard data structures

vamos a escanear estructuras de datos no estándar.

Comenzamos capturando la petición:

![[Pasted image 20260223214732.png]]

como observamos tenemos `wiener` y algo que no sabemos que es exactamente.

Vamos a seleccionar lo siguiente que veremos en las imágenes, teniendo seleccionado lo que es `wiener`:

![[Pasted image 20260223215209.png]]

![[Pasted image 20260223215222.png]]

le damos a `scan` y simplemente toca, esperar.

![[Pasted image 20260223215741.png]]

Enviando la petición al repeater vemos:
![[Pasted image 20260223215904.png]]

al parecer esta intentado escapar de algún tipo de contexto pero podemos hacer peticiones externas.

![[Pasted image 20260223220137.png]]

lo que hacemos es modificar la petición para que llegue a nuestro collaborator y que incluya lo que es en base64 la cookie del usuario al que llega la inyeccion:

![[Pasted image 20260223220356.png]]

nos a llegado y al parecer tenemos el identificador que queríamos donde el usuario administrador a nivel de sesión se inyecto algo en algún panel administrativo posiblemente algún panel de lectura de logs donde se interpreto los comandos y pues nos llega a nosotros la información de sus cookies.

Iniciemos sesión como `administrator` para terminar el lab:

![[Pasted image 20260223220713.png]]

Pues ya esta, pero en realidad vemos que si tenemos un apartado de Logs donde se interpretan, podemos hacer una prueba integrando una alerta:

![[Pasted image 20260223220852.png]]

enviamos y veamos si al entrar se genera la alerta:

![[Pasted image 20260223220924.png]]

Pues si, aquí es el punto vulnerable.

Eliminamos al usuario `carlos` y terminamos el lab:

![[Pasted image 20260223220957.png]]