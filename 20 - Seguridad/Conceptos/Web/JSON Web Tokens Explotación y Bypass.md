---
tags:
  - ciberseguridad
  - web
  - json-bypass
tipo: concepto
estado: terminado
---

# JSON Web Tokens: Explotación y Bypass

Los **JSON Web Tokens** (**JWT**) son un formato compacto de tokens firmados para **autenticación y autorización** (estándar abierto **RFC 7519**, con JWS RFC 7515 y JWK RFC 7517). Permiten transmitir claims entre partes de forma confiable **sin estado en el servidor**. Esa misma naturaleza — *el token es autocontenido y el servidor solo verifica la firma* — es lo que convierte cada error de implementación (algoritmo mal confiado, clave expuesta, claim no validada) en bypass de autenticación o escalada de privilegios.

> [!info] Estructura de un JWT
> Un JWT se compone de tres partes separadas por puntos (`.`):
> 1.  **Header (Cabecera)**: Contiene metadatos sobre el token, como el tipo de token (`typ`) y el algoritmo de firma (`alg`).
> 2.  **Payload (Carga útil)**: Contiene las "claims" o afirmaciones sobre una entidad (normalmente el usuario) y metadatos adicionales.
> 3.  **Signature (Firma)**: Se utiliza para verificar que el remitente del JWT es quien dice ser y para asegurar que el mensaje no ha sido alterado.
>
> Cada parte es **base64url** (no está cifrada): `eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ3aWVuZXIifQ.<firma>`. Cualquiera puede decodificar header y payload; la firma solo garantiza **integridad**, nunca confidencialidad.

## Mecánica: cómo se firma y se verifica

| Algoritmo (`alg`) | Tipo | Clave de firma | Clave de verificación | Dónde se rompe típicamente |
|---|---|---|---|---|
| `HS256/384/512` | HMAC (simétrico) | secreto compartido | **mismo secreto** | secreto débil → crack; si el server usa la pública como secreto → confusión |
| `RS256/384/512` | RSA (asimétrico) | clave **privada** del server | clave **pública** del server | `alg:none`; jwk/jku inyectado; kid traversal |
| `ES256/384/512` | ECDSA | clave privada | clave pública | mismas familias que RS |
| `none` | sin firma | — | — | server que acepta tokens sin verificar |

Firma (JWS): `signature = HMAC/RSAlgo(base64url(header) + "." + base64url(payload), clave)`. **Cualquier cambio en payload invalida la firma**… salvo que el server no la verifique o la verifique con la clave equivocada.

Claims registradas importantes: `iss` (emisor), `sub` (sujeto/usuario), `aud` (audiencia), `exp` (expiración), `nbf` (no válido antes de), `iat` (emitido en), `jti` (id único). Claims custom: `role`, `admin`, `isAdmin`, etc.

> [!warning] Riesgos de una mala implementación de JWT
> Las vulnerabilidades en JWT a menudo surgen de una implementación incorrecta de la validación de la firma, la gestión de claves o la confianza excesiva en los datos proporcionados por el cliente. Esto puede llevar a:
> *   **Bypass de autenticación**: Acceso no autorizado a cuentas de usuario.
> *   **Escalada de privilegios**: Un usuario con privilegios bajos puede obtener privilegios de administrador.
> *   **Falsificación de identidad**: Suplantación de otros usuarios.

En lo que respecta a la fase de **enumeración**, el atacante observa el token (header al decodificar, endpoint `jwks.json`, tamaño/algoritmo de la firma) y prueba tokens falsificados contra la aplicación para validar hipótesis (¿acepta `none`? ¿la clave es `secret1`? ¿verifica `exp`?). La **explotación** usa esas señales para alterar claims (`sub`, `role`, `exp`) y obtener sesiones con privilegios elevados.

> [!tip] Mejores prácticas de seguridad para JWT
> *   **Validación estricta de la firma**: Siempre verificar la firma con la clave correcta; nunca confiar en el header para elegir el algoritmo.
> *   **Allowlist de algoritmos**: Solo los que la app emite (p. ej. `['RS256']`), nunca `none`.
> *   **Gestión segura de claves**: Proteger secretos/privadas, **rotar regularmente**, usar JWKS con `kid` validado.
> *   **Expiración corta y revocación**: `exp` corto + refresh tokens; lista de revocación o `jti` cacheado para logout.
> *   **Validar todas las claims**: `iss`, `aud`, `exp`, `nbf` — no solo la firma.
> *   **No almacenar información sensible**: el payload es solo base64, no cifrado.

## Catálogo de ataques

| Ataque | Vector | Señal | Payload clave |
|---|---|---|---|
| Claim tampering sin verificar | server ignora firma | editar `sub`/`role` y sigue válido | `sub: administrator` con firma vieja |
| `alg: none` | header `alg` controlado | RS256 → none, sin signature | `{"alg":"none"}` (+ variantes `None`, `NONE`, `nOnE`) |
| Firma débil | secreto HS* crackeable | JWT corto, secreto default | hashcat `-m 16500` |
| Confusión de algoritmo | RS256→HS256 con clave pública como HMAC | `jwks.json` expuesto | firmar HS256 con la PEM pública |
| `jwk` injection | token embebe su clave pública | header `jwk` no visto antes | JWK propio en el header |
| `jku`/`x5u` injection | server descarga JWKS de URL | header `jku` controlable | URL a servidor atacante |
| `kid` path traversal | `kid` → ruta de archivo | kid = `../../../../dev/null` | firma con contenido conocido del archivo |
| Validación incompleta | no `exp`/`iss`/`aud` | token viejo o de otro issuer aceptado | replay con `exp` pasado |

## Labs PortSwigger: Bypass de Autenticación JWT

### Lab: JWT authentication bypass via unverified signature

En este laboratorio, al parecer, se implementa un JWT para el manejo de sesiones que no tiene correctamente implementada la verificación de la firma.

Vamos a usar la extensión de JWT Editor en Burp Suite.

Ingresamos con las credenciales que nos proporcionan y recargamos la web, pasándola por Burp Suite para ver cómo se está tramitando:

![[Pasted image 20260223163658.png]]

Como observamos, tenemos el JWT. Este se compone de 3 partes, donde la primera es el encabezado, la segunda los datos (payload) y la tercera la firma. Estos apartados están separados por un punto.
El primer apartado contiene el tipo de codificación que se está empleando. En el segundo, son datos como el rol, usuario, etc. El tercero es la firma que se genera al combinarse el tipo de codificación con los datos, lo que da como resultado una firma única para cada conjunto.

El problema recae cuando no se valida correctamente la firma, ya que si esto no está correctamente validado, podemos alterar los datos y usar la misma firma. Veamos el caso:

![[Pasted image 20260223164100.png]]

Vemos que entre los datos tenemos el valor de `sub: wiener`. Si la aplicación no valida correctamente la firma, podemos alterarlo, por ejemplo, a `administrator` y dejar la misma firma:

![[Pasted image 20260223164209.png]]

Al enviar esto o usar este nuevo JWT, vamos a ver que es válido y entramos como administrador:

![[Pasted image 20260223164325.png]]

Como observamos, entramos como administrador y podemos eliminar al usuario `carlos`, como es de costumbre:

![[Pasted image 20260223164435.png]]

Lab terminado.

### Lab: JWT authentication bypass via flawed signature verification

En este caso, tenemos un laboratorio similar que, en realidad, al parecer acepta JWT sin firma.

Comenzamos entrando y capturando la petición con ayuda de Burp Suite:

![[Pasted image 20260223165017.png]]

Podemos observar que en el primer apartado tenemos el tipo de algoritmo implementado `RS256`. Lo que vamos a hacer es cambiarlo a `none` (ninguno), y luego quitamos el tercer apartado, que es la firma. Por último, modificamos los datos para ingresar como administrador. Como nos describe la máquina que acepta JWT sin algoritmo, podemos intentarlo:

> [!info] Vulnerabilidad `alg: none`
> Esta vulnerabilidad ocurre cuando el servidor no verifica el algoritmo especificado en el encabezado del JWT y, si se establece en `none`, simplemente omite la verificación de la firma. Un atacante puede crear un JWT con `alg: none`, modificar el payload a su gusto y el servidor lo aceptará como válido sin verificar ninguna firma. Variantes para evadir filtros: `None`, `NONE`, `nOnE`, y el punto final (`eyJ...none.`) según la librería.

![[Pasted image 20260223165405.png]]

Podemos usar ese JWT resultante a ver si funciona:

![[Pasted image 20260223165710.png]]

Ya podemos eliminar al usuario `carlos` para completar el laboratorio:

![[Pasted image 20260223165743.png]]

### Lab: JWT authentication bypass via weak signing key

En este laboratorio, al parecer, vamos a tener que romper un JSON Web Token, recordando que para poder firmar u obtener una firma válida es necesario una contraseña (clave secreta).

Nos comparten un diccionario que podemos usar, el cual descargamos y tenemos listo.

Comenzamos capturando la petición para lograr identificar el JWT:

![[Pasted image 20260223170754.png]]

Lo que observamos es que la firma es bastante pequeña; esto no significa que sea fácil de romper, es solo una observación.

Lo que vamos a hacer es usar la herramienta Hashcat para intentar descubrir la contraseña con la cual se genera la firma de la siguiente manera:

-   Almacenamos primero todo el JWT en un archivo llamado `jwt`.
-   Usamos Hashcat: `hashcat -a 0 -m 16500 jwt jwt.secrets.list`
> [!info] Hashcat y JWT
> El modo `-m 16500` en Hashcat está específicamente diseñado para crackear JWT con algoritmos `HS256`, `HS384` y `HS512`. El tipo de ataque `-a 0` indica un ataque de diccionario (`-a 3` para fuerza bruta, `-a 6/7` híbridos con mask: `hashcat -a 6 -m 16500 jwt wordlist?d?d?d?d`). La clave secreta (`secret1` en este caso) es crucial para la firma simétrica.

![[Pasted image 20260223171240.png]]

Como podemos observar, ya tenemos la clave que es `secret1`.

Algo a tener en cuenta es que el algoritmo que se está implementando tiene que ser simétrico. Eso quiere decir que se usa tanto para crear como para descifrar. Si es simétrico, lo vamos a poder usar para crear cualquier JWT sin problemas:

![[Pasted image 20260223171528.png]]

Podemos buscar si el algoritmo es simétrico, pero en este caso sí lo es, por lo que continuamos.

Vamos a hacer uso de la extensión JWT Editor de Burp Suite de la siguiente manera:

![[Pasted image 20260223171814.png]]

Como podemos observar, tenemos que darle a `New Symmetric Key`:

![[Pasted image 20260223171854.png]]

Damos a `Generate` y vemos lo siguiente:

![[Pasted image 20260223172019.png]]

A la estructura que vemos la conocemos como JWK (JSON Web Key). Lo que tenemos que cambiar es el valor de `k`, donde tenemos que poner el secreto que es `secret1` pero esto en Base64, y luego damos OK:

> [!tip] Codificación Base64 para JWK
> El valor de la clave secreta (`k`) en un JWK para algoritmos simétricos (como `HS256`) debe estar codificado en Base64 URL-safe. Esto asegura que la clave se transmita correctamente dentro de la estructura JSON.

![[Pasted image 20260223172143.png]]

Listo, se creó el identificador de Token y lo dejamos. Regresamos a la captura que tenemos de la petición e ingresamos en JSON Web Token:

![[Pasted image 20260223172425.png]]

Podemos observar la cabecera y el payload (data) ya decodificados. Lo que tenemos que cambiar es la firma o crear una nueva vinculada para los cambios, y el cambio es de `wiener` a `administrator`:

![[Pasted image 20260223172707.png]]

Con el cambio hecho, tenemos que generar una nueva firma válida. Tenemos que ir a `Sign`:

![[Pasted image 20260223172728.png]]

Vemos que nos da el mismo identificador de token que creamos anteriormente (que contiene el secreto) y damos en OK:

![[Pasted image 20260223172841.png]]

Con esto se generó una nueva firma y, por lo tanto, un JWT válido. Vamos a copiarlo y a intentar usarlo a ver qué conseguimos:

![[Pasted image 20260223173247.png]]

Solo queda eliminar al usuario `carlos` para completar el laboratorio.

### Lab: JWT authentication bypass via jwk header injection

En este laboratorio, al parecer, vamos a poder hacer una inyección en la cabecera, donde vamos a poder realizar una manipulación del JWK embebido en el JWT para engañar al servidor en un contexto asimétrico.
Esta vulnerabilidad es de confianza en los datos; el servidor confía en los datos enviados, lo que provoca la inyección.
La idea es engañar al servidor para que verifique el JWT empleando una clave pública que vamos a crear y que proporcionamos al servidor para poder crear un JWT válido a final de cuentas.

Primero, capturamos la petición que lleva el JWT:

![[Pasted image 20260223175434.png]]

Ya tenemos el JWT y, si verificamos el `header`, podemos observar que tenemos un algoritmo asimétrico (`RS256`). Cuando vemos que es asimétrico, cambia un poco el proceso de firma y verificación, ya que el servidor se encarga de firmar el JWT mediante su clave privada y, al momento de verificar, lo realiza mediante su clave pública.

El problema radica cuando el servidor admite un JWK embebido dentro del JWT sin verificar el origen, ya que si yo le paso la clave pública con la cual verifique la firma que yo mismo creé con la correspondiente clave privada, nos lo toma como válido.

> [!warning] Inyección de JWK (JSON Web Key)
> La vulnerabilidad de inyección de JWK ocurre cuando el servidor procesa el encabezado `jwk` de un JWT y utiliza la clave pública proporcionada directamente en el token para verificar su firma. Un atacante puede generar un par de claves RSA, firmar un JWT con su clave privada y luego incrustar su clave pública en el encabezado `jwk` del token. Si el servidor confía en esta clave incrustada, validará el token malicioso.

Entonces, lo que vamos a hacer es volver a JWT Editor:

![[Pasted image 20260223180116.png]]

Damos en `New RSA Key`:

![[Pasted image 20260223180132.png]]

Damos en `Generate`:

![[Pasted image 20260223180229.png]]

Damos en OK y listo:

![[Pasted image 20260223180253.png]]

Ahora lo que tenemos que hacer es volver a donde tenemos capturado el token e ingresar en JWT y primero modificar de `wiener` a `administrator`:

![[Pasted image 20260223180812.png]]

Vamos a `Attack` y a `Embedded JWK`:

![[Pasted image 20260223180831.png]]

![[Pasted image 20260223180555.png]]
Seleccionamos el algoritmo y la ID de la `key` que creamos y le damos en OK:

![[Pasted image 20260223180909.png]]

Vemos cómo se agregaron campos, que no es más que el `JWK`, donde se envía la clave pública que, si el servidor permite usarla directamente, nuestro JWT es válido.

Vamos a usar el JWT y veamos qué sucede:

![[Pasted image 20260223181103.png]]

Ya es cuestión de eliminar al usuario `carlos` para terminar con el lab:

![[Pasted image 20260223181139.png]]

### Lab: JWT authentication bypass via jku header injection

En este laboratorio, se abusa de la confianza del servidor al igual que en el anterior. Sin embargo, a diferencia del anterior, donde enviamos la clave pública directamente mediante el JWK, lo que hacemos es almacenar esta clave en un servidor externo. El servidor víctima, mediante el encabezado `jku` (JWK Set URL), obtiene las claves públicas maliciosas que permiten que el JWT autofirmado por un actor externo, mediante la clave privada emparejada con la clave pública del servidor, provoque que el JWT se vuelva válido. Esto lo conocemos como vulnerabilidad de confianza en el origen, debido a que estas claves vienen de forma externa.

Comencemos interceptando el JWT:

![[Pasted image 20260223190520.png]]

Vemos que el algoritmo es asimétrico.

En el laboratorio también nos ayudan con un servidor, y bueno, comenzamos.

Lo primero es configurar nuestro JWT actual para agregar el parámetro de `jku`, donde definimos la URL desde la cual se descargará el conjunto de claves públicas a usar, y será nuestro servidor:

![[Pasted image 20260223190938.png]]

Por ahora, solo modificamos y agregamos el parámetro `jku`. Lo siguiente también será cambiar el `kid`. En nuestro servidor, vamos a representar la siguiente estructura:

> [!info] Encabezado `jku` (JWK Set URL)
> El encabezado `jku` especifica una URL que apunta a un conjunto de JWKs (JSON Web Key Set) que contiene la clave pública necesaria para verificar la firma del JWT. La vulnerabilidad surge cuando el servidor confía en cualquier URL proporcionada en `jku`, permitiendo a un atacante alojar su propia clave pública maliciosa y hacer que el servidor la use para validar un JWT firmado por el atacante.

![[Pasted image 20260223191128.png]]

Ahora volvemos a hacer uso de la extensión JWT Editor y generamos una clave asimétrica, como ya lo hemos hecho en el lab anterior:

![[Pasted image 20260223191308.png]]

Ya con la clave, vamos a darle clic derecho y seleccionar la opción que vemos en pantalla:

![[Pasted image 20260223191408.png]]

Y luego vamos a pegarlo en nuestro servidor:

![[Pasted image 20260223191455.png]]

Tenemos un `kid` (Key ID), el cual vamos a copiar para ponerlo en nuestro JWT:

![[Pasted image 20260223191739.png]]

Ya con esto, vamos a cambiar el valor de `wiener` a `administrator` y proceder a firmar este JWT para poder proceder:

![[Pasted image 20260223191859.png]]

Podemos proceder a copiar el JWT y ver si es válido:

![[Pasted image 20260223191948.png]]

Como podemos observar, logramos entrar como administrador, por lo que procedemos a eliminar al usuario `carlos` para terminar el laboratorio:

![[Pasted image 20260223192029.png]]

### Lab: JWT authentication bypass via kid header path traversal

En este laboratorio, vamos a abusar del parámetro `kid` (Key ID), teniendo en cuenta que el servidor tiene una estructura parecida a:

![[Pasted image 20260223192846.png]]

En sí, lo que sucede es que las claves públicas con las cuales se realizará la verificación del token se encuentran almacenadas dentro de un directorio del sistema. Aquí entra el problema: podríamos efectuar un `[[Path Traversal|path traversal]]` para salirnos del contexto del directorio actual y, aunque no podríamos directamente leer archivos internos del sistema, podríamos hacer cosas.

Comenzamos capturando el JWT:

![[Pasted image 20260223193534.png]]

Vemos un algoritmo simétrico y vamos a modificar de una vez de `wiener` a `administrator`:

![[Pasted image 20260223193626.png]]

Ahora sí, lo primero es crear una clave en la extensión de JWT Editor, en este caso simétrica:

![[Pasted image 20260223193933.png]]

En este caso, nosotros tendríamos que pasarle un secreto, el cual será un byte nulo en Base64 de la siguiente manera:

> [!info] `kid` y Path Traversal con Null Byte
> La vulnerabilidad `kid` (Key ID) con Path Traversal ocurre cuando el servidor utiliza el valor del encabezado `kid` para construir una ruta de archivo desde la cual cargar la clave de verificación. Si el servidor no sanitiza adecuadamente este valor, un atacante puede inyectar secuencias de `../` para acceder a archivos fuera del directorio esperado. Combinado con un "null byte" (`%00` o `\x00`), se puede truncar la ruta y forzar al servidor a cargar un archivo predecible (como `/dev/null` en Linux) que contiene un valor conocido (o vacío), permitiendo al atacante firmar un JWT con ese valor conocido.

![[Pasted image 20260223194047.png]]

Y quedaría de la siguiente manera:

![[Pasted image 20260223194115.png]]

Y le damos OK y listo.

Ahora lo siguiente es en el parámetro de `kid` en el contexto del JWT que teníamos, efectuamos el Path Traversal, pero hacia `../../../../dev/null`. Esta ruta nos devuelve vacío como un byte nulo:

![[Pasted image 20260223194556.png]]

Y aquí está el truco: nosotros vamos a firmar con el secreto como byte nulo, y la clave que el servidor va a cargar para verificar es un byte nulo, lo que teóricamente es correcto. Por lo tanto, vamos a proceder a firmar el JWT:

![[Pasted image 20260223194728.png]]

Ya firmado, vamos a copiar el token y a ver si funciona:

![[Pasted image 20260223194811.png]]

Ya es cuestión de eliminar al usuario `carlos` para terminar:

![[Pasted image 20260223194848.png]]

### Lab: JWT authentication bypass via algorithm confusion

En este laboratorio, vamos a ver cómo podemos, mediante una confusión del algoritmo, cambiar el mismo de uno simétrico a uno asimétrico de forma forzosa, combinando el uso indebido de la clave, ya que esta está expuesta en el servidor.

Comenzamos capturando el JWT:

![[Pasted image 20260223200355.png]]

Bueno, recordemos que tenemos que obtener una clave pública expuesta. Esto lo podemos hacer mediante fuzzing, pero con un poco de investigación, vemos que el endpoint usual es:

![[Pasted image 20260223200648.png]]

La ruta es `/.well-known/jwks.json`. En nuestro caso, solo es directamente en `jwks.json`:
![[Pasted image 20260223200739.png]]

Bueno, para verlo mejor, podemos jugar desde consola con `curl`:

![[Pasted image 20260223200941.png]]

Como observamos, la salida es mucho más organizada y vemos mejor la clave pública. Recordemos que la clave pública la usamos para verificar.

Recordando esto, que es una clave asimétrica, vamos a intentar forzar una verificación simétrica, usando el algoritmo `HS256`.

El problema principal es que en servidores mal configurados se usa la clave pública tanto para el algoritmo `RS256` como para `HS256`.

> [!warning] Confusión de Algoritmos (RS256 a HS256)
> Esta vulnerabilidad ocurre cuando un servidor espera un JWT firmado con un algoritmo asimétrico (ej. `RS256`, que usa una clave privada para firmar y una clave pública para verificar), pero también acepta tokens firmados con un algoritmo simétrico (ej. `HS256`, que usa una única clave secreta para firmar y verificar). Si la clave pública del servidor está expuesta, un atacante puede tomar esa clave pública, usarla como clave secreta para firmar un JWT con `HS256`, y el servidor la aceptará porque intentará verificarla con la misma "clave pública" que ahora está usando como "clave secreta".

Lo primero que haremos es generar una clave asimétrica:

![[Pasted image 20260223201717.png]]

Como podemos observar, esto nos genera contenido, pero este contenido no es necesario porque tenemos el que está expuesto de la clave pública, por lo que es cuestión de representarlo allí en lugar de lo que se genera:

![[Pasted image 20260223201943.png]]

Con eso listo, le damos a OK.

Ahora vamos a dar clic derecho sobre la clave que generamos y vamos a copiarla:

![[Pasted image 20260223202127.png]]

Y en el mismo decoder de Burp Suite lo vamos a pegar:

![[Pasted image 20260223202953.png]]

Esto lo vamos a representar en Base64:

![[Pasted image 20260223203031.png]]

Ya con esto es lo que vamos a usar como secreto al crear el JWT simétrico:

![[Pasted image 20260223203202.png]]

Ya con esto listo, damos en OK y tenemos listo.

Por último, vamos a modificar 2 cosas: una es el algoritmo para que sea `HS256` y el otro es de `wiener` a `administrator`:

![[Pasted image 20260223203419.png]]

En este punto, lo que vamos a hacer es firmar el JWT con la clave simétrica y la usamos a ver si funciona:

![[Pasted image 20260223203526.png]]

Como podemos observar, solo es cuestión de eliminar al usuario `carlos` para terminar:

![[Pasted image 20260223203558.png]]

### Lab: JWT authentication bypass via algorithm confusion with no exposed key

El propósito es el mismo del anterior laboratorio, pero en este caso no vamos a contar con una clave expuesta, por lo tanto, vamos a ver cómo se puede resolver.

Comenzamos capturando el JWT:

![[Pasted image 20260223205100.png]]

Lo que vemos de primera es que el token está firmado por `RS256`, siendo un algoritmo asimétrico, y no tenemos acceso a la clave pública.

Ahora lo que sucede es que si tenemos ya una firma digital válida, podemos aprovecharnos de esto. Con ayuda de la herramienta que nos comparten, esta herramienta genera valores de `n` supuestamente válidos, los cuales de nuestro lado debemos validar. El objetivo es encontrar uno valido para, mediante este, aprovecharnos para obtener la clave pública y, posteriormente, vamos a poder generar la confusión de algoritmos igual que lo vimos anteriormente.

Para la herramienta, copiamos dos tokens válidos de dos sesiones distintas y procedemos a ejecutarla de la siguiente manera:

![[Pasted image 20260223205846.png]]

Ya tenemos los dos JWT válidos, por lo tanto, vamos a ejecutar la herramienta:

![[Pasted image 20260223210550.png]]

La herramienta nos devuelve 2 cosas: el valor de `n` y genera automáticamente un JWT el cual tenemos que probar a ver si es válido. Si este es válido, tendremos el valor correspondiente de `n` a usar para firmar un JWT malicioso.

> [!info] Ataque de Confusión de Algoritmos sin Clave Pública Expuesta
> Cuando la clave pública no está expuesta, se puede intentar reconstruir el módulo público (`n`) de la clave RSA a partir de dos JWT válidos firmados con `RS256` (ataque de Bleichenbacher sobre firmas mal implementadas, estilo `jwt-key-confusion` tools). Una vez obtenido `n`, este valor se puede usar como la clave secreta para firmar un nuevo JWT con `HS256`, explotando la confusión de algoritmos.

Vemos si es válido:
![[Pasted image 20260223211021.png]]

Vemos que este es válido y, pues, ya tenemos el valor de `n` correcto.

Vamos a crear en JWT Editor la clave simétrica. Recordemos que vamos a hacer una confusión de algoritmo, por lo tanto, al crear la clave simétrica le pasamos el valor de `n` correspondiente al token que usamos:

![[Pasted image 20260223211226.png]]

Damos OK y listo.

Ahora cambiamos de `wiener` a `administrator` y pasamos de `RS256` a `HS256`:

![[Pasted image 20260223211330.png]]

Procedemos a firmar este JWT y a ponerlo a prueba:

![[Pasted image 20260223211413.png]]

Como vemos, es válido y podemos eliminar al usuario `carlos` para terminar.

![[Pasted image 20260223211443.png]]

## Detección y reconocimiento

| Señal | Qué probar |
|---|---|
| `alg: none` en header aceptado | token sin firma con claims alteradas |
| `jwks.json` / `jku` accesible | confusión RS→HS, jku injection |
| HS256 + secreto de longitud corta | crack con hashcat 16500 |
| `kid` dinámico | traversal (`../../../../dev/null`) y SQLi (`kid=x'--`) |
| `exp` ausente o muy largo | replay de tokens viejos |
| Cookie/header alternativo | robar token de otro contexto (HTTP vs HTTPS) |

Herramientas: **Burp JWT Editor** (generar keys, embedded JWK, sign), `jwt_tool` (triage automatizado de todos los ataques), `hashcat -m 16500`, `john` con formato JWT, `c-jwt-cracker`.

## Defensas

**1. Allowlist de algoritmo + verificación estricta** (Node.js `jsonwebtoken`):

```js
const jwt = require('jsonwebtoken');

// VERIFICAR: algoritmo fijo y allowlist — nunca inferir desde el token
const payload = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
  algorithms: ['RS256'],          // jamás 'none' ni lista dinámica
  issuer: 'mi-app',
  audience: 'mi-app-web',
  maxAge: '5m',                   // exp corto
});
// además validar a mano las claims de negocio:
if (payload.role !== 'user') reject();
```

```python
# Python (PyJWT) — mismo principio
import jwt
payload = jwt.decode(token, key, algorithms=["RS256"], audience="mi-app", issuer="mi-app")
```

**2. Nunca confiar en el header para elegir clave/algoritmo**: seleccionar la clave por `kid` **validado contra un mapa interno** (o regex `^[\w-]{1,32}$` + lookup en JWKS local), y rechazar tokens con `jwk`/`jku`/`x5u` del cliente si no se usan deliberadamente con allowlist de hosts.

**3. Gestión y rotación de claves**:

```js
// Rotación: JWKS con kid y verificación por clave actual + anterior
const JWKS = jose.createLocalJWKS(fs.readFileSync('jwks.json')); // public keys
const { payload } = await jose.jwtVerify(token, JWKS, {
  algorithms: ['RS256'], issuer: 'mi-app',
  clockTolerance: 30,   // margen por desfase de reloj, no horas
});
// rotar: emitir clave nueva con kid nuevo; mantener la anterior para verificación
//         durante la ventana de vida de los tokens vigentes; revocar por jti si hace falta
```

- Secretos HS*: mínimo 256 bits de entropía, en vault/env, **rotados**; cada rotación invalida tokens viejos (por diseño).
- Privadas RS*: solo en el emisor; el verificador solo necesita la pública.

**4. Expiración + revocación**: `exp` ≤ minutos; refresh tokens con rotación; `jti` en denylist para logout; evitar datos sensibles en el payload (es solo base64 — cifrar si debe ser confidencial).

**5. Endurecer transporte**: `HttpOnly; Secure; SameSite` para cookies JWT; no meter JWT en URLs/`localStorage` si hay alternativa (XSS los roba).

## Errores comunes

**Atacante:**
1. Decodificar el JWT en base64 y creer que está "cifrado" — es legible siempre.
2. Probar solo `none` minúscula exacta (variantes de case y el punto final).
3. Firmar HS256 con la PEM **completa** cuando el server espera solo el `n`/clave raw (en el lab sin key expuesta se usa el módulo `n`, no el PEM).
4. Olvidar `Content-Type`/cookie correcta al replay el token alterado.
5. No verificar localmente con `jwt decode` qué claims quedaron tras editar.

**Defensor:**
1. `jwt.verify(token, secret)` sin `algorithms` → el header elige el algoritmo (confusión).
2. Confiar en `role`/`admin` del payload sin revalidar contra BD (token = dato viejo).
3. Secretos por defecto (`secret`, `changeme`, `your-256-bit-secret`) en librerías y plantillas.
4. Tokens sin `exp`, o `exp` de 30 días "para no molestar al usuario".
5. Aceptar `kid` como ruta de archivo sin sanitizar (traversal a `/dev/null`).

## Cheatsheet

| Paso | Acción |
|---|---|
| Decodificar | base64url de header/payload (jwt.io o Burp JWT Editor) |
| `alg:none` | poner `none`, vaciar firma (variantes `None/NONE/nOnE`, trailing dot) |
| Crack HS* | `hashcat -a 0 -m 16500 jwt.wordlist` |
| Confusión RS→HS | usar clave pública (o `n`) como secreto HS256 |
| jwk inyectada | `Attack → Embedded JWK` en JWT Editor |
| jku | alojar JWKS en servidor propio, header `jku`→tu URL |
| kid traversal | `kid=../../../../dev/null`, firma con byte nulo |
| Claims | `sub/role/admin/exp/iss/aud` — editar y re-firmar |
| Defensa | `algorithms:['RS256']` + iss/aud + exp corto + kid allowlist |

## Referencias

- PortSwigger Web Security Academy — JWT: https://portswigger.net/web-security/jwt
- RFC 7519 (JWT), RFC 7515 (JWS), RFC 7517 (JWK)
- Auth0 — JWT Handbook: https://auth0.com/resources/ebooks/jwt-handbook
- HackTricks — JWT: https://book.hacktricks.wiki/en/pentesting-web/jwt-python-rb.html
- Hashcat example hashes (mode 16500): https://hashcat.net/wiki/doku.php?id=example_hashes

## Relacionadas

- [[Gestión de Autenticación]] — el mapa completo de mecanismos y fallos de autenticación donde vive el JWT
- [[Inyección NoSQL]] — bypass de login previo a la emisión del token
- [[Path Traversal]] — el mismo vector aplicado al `kid` del JWT
- [[Vulnerabilidades OAuth en PortSwigger Labs]] — tokens de otro ecosistema con fallos de validación análogos
- [[Session Puzzling Session Fixation y Session Variable Overloading]] — alternativas de gestión de sesión y sus fallos
