-------------

## PortSwigger


### Lab: OS command injection, simple case

En este laboratorio tenemos que explotar una ejecución de comandos que por lo que nos dice la web esta en el apartado de stock checker y para completarlo tenemos que ejecutar el comandos `whoami`.

Vamos a comenzar viendo el apartado de stock checker:
![[Pasted image 20250922092642.png]]

Vamos a capturar la petición que se envía al momento de darle click:

![[Pasted image 20250922092749.png]]

Vemos como se esta realizando la petición, lo que nos viene a la mente es que en alguno de los dos parámetros tenemos alguna ejecución de comandos, por lo tanto al no conocer ni la lógica ni el como se esta empleando por detrás para obtener el valor, lo que tenemos que hacer es intentar diferentes tipos de cosas como:
```bash
;whoami
|whoami
||whoami
&&whoami
```

![[Pasted image 20250922093054.png]]

vemos un error cuando intentamos ejecutar el comando pero ya nos habla de una bash, en este punto como vemos de alguna forma es que corrompemos la petición ya que pasa de alguna forma como otro parámetro por lo tanto lo que vamos a hacer es intentar al final de la petición a ver si eso le gusta mas:
![[Pasted image 20250922093347.png]]

Perfecto eso si le gusto y nos devolvió el output del comando:
![[Pasted image 20250922093448.png]]

Lab terminado.

### Lab: Blind OS command injection with time delays

En este caso la inyección de comandos es a siegas y lo que tenemos que causar es que el servidor se demore en responder unos 10 segundos de espera.

En este caso no sabemos donde se acontece la posible inyección por lo tanto vamos a capturar un apartado primero donde tengamos entrada de datos como por ejemplo el Submit Feedback que tenemos en la web:

![[Pasted image 20250922094541.png]]

Podemos observar multiples campos los cuales pueden ser inyectables y para poder encontrarlo algo que tenemos que hacer es probar uno por uno:

![[Pasted image 20250922094741.png]]

En campos como el name, subject y message tenemos la misma respuesta al enviar un `;` incluido en su contenido, esto me da de alguna forma a entender que causamos algún error, podemos intentar enviar comandos pero no veremos una respuesta, recordemos que es a ciegas en esta ocasión.

Podemos intentar hacer comandos que demoren un tiempo con el cual identifiquemos que si se ejecutan algunos:
![[Pasted image 20250922095007.png]]

No funciona puede que se este pasando como parámetro pero si lo ejecutamos `$(sleep 5)` lo que hace es ejecutar el comando e ingresar su output:
![[Pasted image 20250922095129.png]]

De esta forma si nos permite la ejecución de comandos, otra forma fora usar en el caso de que no se permitan parámetros como `() o $` se puede hacer englobando la petición con los `;` de la siguiente manera:
![[Pasted image 20250922095312.png]]

En este caso en el parámetro name no funciono de esa manera pero en email si por lo tanto es muy importante tenerlo en cuenta a la hora de hacer pruebas.

Bueno completemos la maquina enviando un sleep de 10 segundos:

![[Pasted image 20250922095442.png]]

Lab terminado.


### Lab: Blind OS command injection with output redirection

En este laboratorio nos dice que tenemos una ruta `www/images` la cual nos permite almacenar imágenes, y que nosotros al ser una inyección a ciegas podemos usar esa ruta para almacenar en un archivo el output.

La inyección se encuentra de igual forma en el feedback de la anterior lab por lo tanto vamos a comenzar:

![[Pasted image 20250922102522.png]]

En este caso el objetivo seria identificar el punto de la inyección de comandos, pero como ya conocemos donde lo podemos realizar lo primero será ejecutar un sleep para identificar que es valido:
![[Pasted image 20250922102705.png]]

Si es valido, en este punto lo que vamos a hacer es ejecutar un comando y enviarlo a la ruta que nos especificaron con el nombre de `test.txt` para que se almacene:
![[Pasted image 20250922102831.png]]

Vemos que al parecer funciono y lo que podemos hacer en este punto es ver en la ruta que mediante una imagen descubrimos como se están cargando, en este punto vemos lo siguiente :
![[Pasted image 20250922102917.png]]

y con esto debería resolverse la maquina:
![[Pasted image 20250922102939.png]]

Lab Terminado.

### Lab: Blind OS command injection with out-of-band interaction

En este laboratorio lo que vamos a ver es como podemos ver los comandos ejecutados a nivel de sistema cuando no tenemos alguna forma de observarlos o almacenarlos como en el lab anterior.

Ya sabemos el punto de inyección por lo que procedemos a capturarlo para poder verlo:
![[Pasted image 20250922103346.png]]

Perfecto ya sabemos cual es el parámetro en el que vamos a inyectar por lo que solo vamos a comprobarlo por rutina:
![[Pasted image 20250922103703.png]]

al parecer funciona, en este punto lo único que nos pide el laboratorio es que realicemos una solicitud al Collaborator con nslookup de la siguiente manera:
![[Pasted image 20250922104107.png]]

si revisamos el collaborator vamos a ver lo siguiente:
![[Pasted image 20250922104138.png]]

Perfecto si nos llegaron las solicitudes y esto es todo:
![[Pasted image 20250922104153.png]]

Lab Terminado.


### Lab: Blind OS command injection with out-of-band data exfiltration

En este comando es prácticamente lo mismo que el anterior con la diferencia que en este punto tenemos que ya enviarnos a través de la petición información, en este caso nos pide el output del comando  `whoami`.


De igual forma tenemos el punto de inyección de comandos y vamos a ir directo a ejecutarlo ya que sabemos donde se están inyectando los comandos, en este punto veremos que para la solicitud hacemos:
![[Pasted image 20250922105008.png]]

En este caso hicimos uso del otro tipo de inyección mediante el cual obtenemos ya el nombre de usuario del sistema:
![[Pasted image 20250922105041.png]]

Perfecto podemos ver si se completo el lab:
![[Pasted image 20250922105132.png]]

Lab terminado.

