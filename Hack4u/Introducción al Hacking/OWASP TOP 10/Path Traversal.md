-----------------------

## PortSwigger 

### Lab: File path traversal, simple case

Este se un caso simple donde el objetivo es listar el archivo `/etc/passwd`.

![[Pasted image 20250923124439.png]]

los puntos inyectables usuales son cualquiera de los campos inyectables en los cuales le podemos pasar información, podemos usar el que se indica en la image y hacer la inyección.

En este caso vemos que no se puede en ese punto, otro que podemos usar es el parámetro mediante el cual se cargan las imágenes:
![[Pasted image 20250923124722.png]]

Vamos a intentar trasladar esto a la url para ver si nos permite la inyección:
![[Pasted image 20250923125644.png]]

No sale como no valido pero en realidad con eso si regresamos a la web principal:
![[Pasted image 20250923125709.png]]

Lab Terminada.

### Lab: File path traversal, traversal sequences blocked with absolute path bypass

Este laboratorio es similar al anterior pero dice que tenemos las secuencias bloqueadas, porque se usan rutas absolutas en este también es valido lo que hicimos en la maquina anterior:
![[Pasted image 20250923130005.png]]

vamos a intentarlo a ver que obtenemos:
![[Pasted image 20250923130423.png]]
vemos que en este caso el usar la ruta de forma absoluta es suficiente por lo que si revisamos:

![[Pasted image 20250923130518.png]]

Lab terminado.

### Lab: File path traversal, traversal sequences stripped non-recursively

Vemos que tenemos una mejor seguridad para esto donde ya no nos permiten el uso de `../` ya que el sistema de alguna forma lo borra o lo quita vamos a ver:
![[Pasted image 20250923131046.png]]

vemos que nos sale como no valido, en este punto una forma en ocasiones que logra engañar al sistema es usar `....//....//....//` porque se puede estar quitando y eliminando pero cuando esto no es recursivo lo que sucede es que me va a quitar solo un `../` en la primera pasada de todas las coincidencias pero al hacerlo queda aun otro `../`, vamos a ver si esto funciona:

![[Pasted image 20250923131226.png]]

Perfecto, esto si funciono, vamos a ver si con esto completamos el lab:

![[Pasted image 20250923131252.png]]

Lab Terminado.


### Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

al parecer ahora tenemos mas seguridad donde nos elimina los `...//...//` al usar URL-decode, vamos a intentar burlar esto:

![[Pasted image 20250923131748.png]]

vemos que no nos funciona y de forma directa tampoco y nada de lo que conocemos por el momento lo hace, ahora en ocasiones lo que hacemos es url-encode de las `/` con el objetivo de que el sistema al hacer un url-decode nos lo interprete, esto puede funcionar haciendo solo una vez, cuando hacemos el url-encode vamos a tener `%2f` per en ocasiones falla y necesitamos al `%` también hacerle url-encode donde seria `%25` quedando la cadena del `/` como `%252f`:

![[Pasted image 20250923132125.png]]

perfecto esto a funcionado.

![[Pasted image 20250923132143.png]]

Lab Terminado.

### Lab: File path traversal, validation of start of path

En esta ocasión vamos a ver como en la url se nos obliga a usar la ruta `/var/wwww/images` esta puede parecer una buena idea pero a la vez no ya que si no tenemos mas que eso podemos aprovecharnos aun de los `../` para modificarla:

![[Pasted image 20250923132549.png]]

Verifiquemos:

![[Pasted image 20250923132613.png]]

Lab Terminada.


### Lab: File path traversal, validation of file extension with null byte bypass

En esta ocasión se esta empleando una forma algo mas persistente y es que concadena siempre la extension de la imagen al final.

Lo que vamos a hacer en este caso es jugar con el conocido null byte que nos permite quitarnos esa concatenación de extension, algo a tener en cuenta es que esto funciona en webs antiguas de php por lo que ya en versiones nuevas no lo vamos a poder hacer.

![[Pasted image 20250923133121.png]]

vemos como esto ya es posible si revismos:
![[Pasted image 20250923133143.png]]

Lab Terminado.

