--------

Los **Json Web Tokens** (**JWT**) son un tipo de token utilizado en la **autenticación** y **autorización** de usuarios en aplicaciones web. JWT es un estándar abierto (**RFC 7519**) que define un formato compacto y seguro para transmitir información entre diferentes partes de forma confiable.

En lo que respecta a la fase de enumeración y explotación de un JWT, esto se produce cuando un atacante es capaz de obtener información sobre los JWT que se utilizan en la aplicación, lo que podría permitir al atacante explotar las debilidades en la autenticación y autorización de la aplicación.

La enumeración de los JWT se produce cuando un atacante utiliza técnicas de fuerza bruta o cierta ingeniería inversa para obtener información sobre los JWT utilizados por la aplicación web. Por ejemplo, el atacante podría intentar adivinar los valores de los JWT mediante la construcción de **tokens falsos**, tratando de validar en todo momento si la aplicación web los acepta o no. Si el atacante tiene éxito en la enumeración de un JWT válido, podría obtener información confidencial, como nombres de usuario, contraseñas, roles de usuario y otros datos de autenticación y autorización.

La explotación de los JWT se produce cuando un atacante utiliza la información obtenida de la enumeración del JWT para explotar debilidades en la aplicación. Por ejemplo, si la aplicación web utiliza JWT para la autenticación, pero no valida adecuadamente la **firma** del JWT, un atacante podría falsificar el token y acceder a la aplicación web como si fuera un usuario legítimo.

Para prevenir la enumeración y explotación de los JWT, es importante utilizar prácticas seguras de desarrollo web, como la validación adecuada de las solicitudes de entrada, la gestión segura de las claves de firma JWT y la limitación del tiempo de expiración de los JWT.

## PortSwigger

### Lab: JWT authentication bypass via unverified signature

En este laboratorio al parecer se implementa un JWT para el manejo de sesiones que al parecer no esta implementado correctamente la verificación de la firma.

Vamos a usar la extensión de JWT Editor en Burp Suite.

Ingresamos con las credenciales que nos proporcionan y recargamos la web, esto pasándolo por el Burp Suite para ver como es que se esta tramitando:

![[Pasted image 20260223163658.png]]

Como observamos tenemos le JWT, este se compone por 3 partes donde la primera es el encabezado, el segundo los datos y el tercero la firma, estos apartados están separados por un punto.
El primer apartado contiene el tipo de codificación que se esta empleando, en el segundo son datos como el rol, usuario, etc, el tercer es la firma que se genera al combinarse el tipo de codificación con los datos esto da como resultado una firma única para cada conjunto.

El problema recae cuando no se valida correctamente la firma ya que si esto no esta correctamente validado podemos alterar los datos y usar la misma firma veamos el caso:

![[Pasted image 20260223164100.png]]

vemos que entre los datos tenemos el valor de `sub: wiener`, si esto no valida correctamente la firma podemos alterarlo por ejemplo con `adminitrator` y dejar la misma firma:

![[Pasted image 20260223164209.png]]

Al enviar esto o usar este nuevo JWT vamos a ver que es valido y entramos como administrador:

![[Pasted image 20260223164325.png]]

Como observamos entramos como administrador y podemos eliminar al usuario `carlos` como es de costumbre:

![[Pasted image 20260223164435.png]]

Lab terminado.

### Lab: JWT authentication bypass via flawed signature verification

En este caso tenemos un laboratorio similar el cual en realidad al parecer aceptar JWT sin firma.

Comenzamos entrado y capturando la petición con ayuda de Burp Suite:

![[Pasted image 20260223165017.png]]

Podemos observar que en el primer apartado tenemos el tipo de algoritmo implementado `RS256`, donde lo que vamos a hacer es cambiarlo a ninguno `none`, y luego quitamos el tercer apartado que es la firma, por último modificando los datos para ingresar como administrador, y como nos describe la máquina que acepta JWT sin algoritmo podemos intentarlo:

![[Pasted image 20260223165405.png]]

Podemos usar ese JWT resultante a ver si funciona:

![[Pasted image 20260223165710.png]]

Ya podemos eliminar el usuario `carlos` para completar el laboratorio:

![[Pasted image 20260223165743.png]]


### JWT authentication bypass via weak signing key

En este laboratorio al parecer vamos a tener que romper un json web token recordando que para poder firmar u obtener una firma valida es necesario una contraseña.

Nos comparten un diccionario que podemos usar el cual lo descargamos y tenemos listo.

Comenzamos capturando la petición para lograr identificar el JWT:

![[Pasted image 20260223170754.png]]

lo que observamos es que la firma es bastante pequeña, esto no significa que sea fácil de romper es solo una observación.

Lo que vamos a hacer es usar la herramienta de HashCat para intentar descubrir la contraseña con el cual se genera la firma de la siguiente manera:

- Almacenamos primero todo el JWT en un archivo llamado jwt
- Usamos hashcat  `hashcat -a 0 jwt jwt.secrets.list`

![[Pasted image 20260223171240.png]]

Como podemos observar ya tenemos la clave que es `secret1`.

Algo a tener en cuenta es que el algoritmo que se esta implementado tiene que ser simétrico, eso quiere decir que se usa para crear como para descifrar, donde si es simétrico lo vamos a poder usar para crear todo JWT sin problemas:

![[Pasted image 20260223171528.png]]

podemos buscar si el algoritmo es simétrico pero en este caso si lo es por lo que continuamos.

Vamos a hacer uso de la extensión de JWT editor de Burp Suite de la siguiente manera:

![[Pasted image 20260223171814.png]]

Como podemos observar tenemos que darle a `New Symmetric Key`:

![[Pasted image 20260223171854.png]]

damos a `Generate` y vemos lo siguiente:

![[Pasted image 20260223172019.png]]

A la estructura que vemos la conocemos como JWK (Json Web Key), lo que tenemos que cambiar es el valor de `k` donde tenemos que poner el secreto que es  `secret1` pero esto en base64 y luego damos ok:

![[Pasted image 20260223172143.png]]

listo se creo el identificador de Token y lo dejamos, regresamos a la captura que tenemos de la petición e ingresamos en JSON Web Token:

![[Pasted image 20260223172425.png]]

Podemos observar la cabecera y el payload (data) ya decodificados donde o que tenemos que cambiar es la firma o crear una nueva vinculada para cambios, y el cambio es de `wiener` a `administrator`:

![[Pasted image 20260223172707.png]]

Con el cambio echo tenemos que generar una nueva firma valida tenemos que ir a `Sign`:

![[Pasted image 20260223172728.png]]

vemos que nos da le mismo identificador de token que creamos anteriormente (que contiene el secreto) y damos en ok:

![[Pasted image 20260223172841.png]]

Con esto se genero una nueva firma y por lo tanto un JWT valido, vamos a copiarlo y a intentar usarlo a ver que conseguimos:

![[Pasted image 20260223173247.png]]

Solo queda eliminar el usuario `carlos` para completar el laboratorio.


### JWT authentication bypass via jwk header injection

En este laboratorio al parecer vamos a poder hacer una inyección en la cabecera, donde vamos poder realizar una manipulación del JWK embebido en el JWT para engañar al servidor en un contexto asimétrico.
Esta vulnerabilidad es de confianza en los datos, el servidor confía en los datos enviados lo que provoca la inyeccion.
La idea es engañar al servidor para que verifique el JWT empleando una cable publica que vamos a crear y que proporcionamos al servidor para poder crear un JWT valido a final de cuentas.

Primero capturamos la petición que lleva el  JWT:

![[Pasted image 20260223175434.png]]

Ya tenemos el JWT y si verificamos el `header` podemos observar que tenemos un algoritmo asimétrico, cuando vemos que es asimétrico cambia un poco el proceso de firma y verificación ya que el servidor se encarga de firmar el JWT mediante su clave privada y al momento de verificar este lo realiza mediante su clave publica.

El problema radica cuando el servidor admite un JWK embebido dentro del JWT sin verificar el origen ya que si yo le paso la clave publica con la cual verifique la firma que yo mismo cree con la correspondiente clave privada, nos lo toma como valido.

Entonces lo que vamos a hacer es volver a JWT Editor:

![[Pasted image 20260223180116.png]]

damos en `New RSA Key`:

![[Pasted image 20260223180132.png]]

Damos en `Generate`:

![[Pasted image 20260223180229.png]]

damos en ok y listo:

![[Pasted image 20260223180253.png]]

Ahora lo que tenemos que hacer es volver a donde tenemos capturado el token e ingresar en JWT y primero modificar de `wiener` a `administrator`:

![[Pasted image 20260223180812.png]]

Vamos a `attack` y a `Embedded JWK`:

![[Pasted image 20260223180831.png]]

![[Pasted image 20260223180555.png]]
Seleccionamos el algoritmo y la ID de la `key` que creamos y le damos en ok:

![[Pasted image 20260223180909.png]]

vemos como se agregaron campos que no es mas que el `JWK` donde se envía la clave publica que si el servidor permite usarla directamente nuestro JWT es valido.

Vamos a usar el JWT y veamos que sucede:

![[Pasted image 20260223181103.png]]

Ya es cuestión de eliminar el usuario `carlos` para terminar con el lab:

![[Pasted image 20260223181139.png]]

### JWT authentication bypass via jku header injection 

En este laboratorio se abusa de la confianza del servidor al igual que en el anterior sin embargo, a diferencia del anterior donde enviamos la clave publica directamente mediante el JWK lo que hacemos es almacenar esta clave en un servidor externo, donde el servidor victima mediante JKU obtiene las claves publicas maliciosas que permiten que el JWT auto firmado por un actor externo mediante la clave privada emparejada con la clave publica del servidor provoca que el JWT se vuelva valido, esto lo conocemos como vulnerabilidad de confianza en el origen debido a que estas claves vienen de forma externa.

comencemos interceptando el JWT:

![[Pasted image 20260223190520.png]]

Vemos que el algoritmo es asimétrico.

En el laboratorio también nos ayudan con un servidor y bueno comenzamos.

Lo primero es configurar nuestro JWT actual para agregar el parámetro de `jku` donde definimos la URL desde la cual se descargara el conjunto de claves publicas a usar y será nuestro servidor:

![[Pasted image 20260223190938.png]]

Por ahora solo modificamos y agregamos el parámetro `jku`, lo siguiente también será cambiar el `kid`, en nuestro server vamos a representar la siguiente estructura:

![[Pasted image 20260223191128.png]]

Ahora volvemos a hacer uso de la extensión JWT Editor y generamos una clave asimétrica como ya lo hemos hecho en el lab anterior:

![[Pasted image 20260223191308.png]]

ya con la clave vamos a darle clic derecho y seleccionamos la opción que vemos en pantalla:

![[Pasted image 20260223191408.png]]

y luego vamos a pegarlo en nuestro servidor:

![[Pasted image 20260223191455.png]]

Tenemos un `kid` el cual vamos a copiar para ponerlo en nuestro JWT:

![[Pasted image 20260223191739.png]]

ya con esto vamos a cambiar el valor de `wiener` a `administrator` y procedemos a firma este JWT para poder proceder:

![[Pasted image 20260223191859.png]]

Podemos proceder a copiar el JWT y ver si es valido:

![[Pasted image 20260223191948.png]]

Como podemos observar logramos entrar como administrador por lo que procedemos a eliminar al usuario `carlos` para terminar el laboratorio:

![[Pasted image 20260223192029.png]]

### JWT authentication bypass via kid header path traversal

En este laboratorio vamos a abusarnos del parámetro `kid` teniendo en cuanta que el servidor tiene una estructura parecida a:

![[Pasted image 20260223192846.png]]

en si lo que sucede es que las claves publicas con las cuales se realizara la verificación del token se encuentran almacenadas dentro de un directorio del sistema y aquí entra el problema es que podríamos efectuar un `path traversal` para salirnos del contexto del directorio actual y aunque no podríamos directamente leer archivos internos del sistema podríamos hacer cosas.

Comenzamos capturando el JWT:

![[Pasted image 20260223193534.png]]

vemos un algoritmo simétrico y vamos modificar de una vez de `wiener` a `administrator`:

![[Pasted image 20260223193626.png]]

Ahora si lo primero es crear una clave en la extensión de JWT Editor en este caso simétrica:

![[Pasted image 20260223193933.png]]

En este caso nosotros tendríamos que pasarle un secreto, el cual será un bite nulo en base 64 de la siguiente manera:

![[Pasted image 20260223194047.png]]

y quedaría de la siguiente manera:

![[Pasted image 20260223194115.png]]

y le damos ok y listo.

Ahora lo siguiente es en el parámetro de `kid` en el contexto del JWT que teníamos efectuamos el Path traversal pero hacia `../../../../dev/null`, esta ruta nos devuelve vacío como un byte nulo:

![[Pasted image 20260223194556.png]]

Y aquí esta el truco nosotros vamos a firmar con el secreto como byte nulo y la clave que el servidor va a cargar para verificar es un byte nulo, lo que teoricamente es correcto por lo tanto vamos a proceder a firmar el JWT:

![[Pasted image 20260223194728.png]]

Ya firmado vamos a copiar el token y a ver si funciona:

![[Pasted image 20260223194811.png]]

Ya es cuestión de eliminar el usuario `carlos` para terminar:

![[Pasted image 20260223194848.png]]

### JWT authentication bypass via algorithm confusion

En este laboratorio vamos a ver como podemos mediante una confucion del algoritmo cambiar el mismo de un simétrico a uno asimétrico de forma forzosa, combinando el uso indebido de la clave ya que esta expuesta en el servidor.

Comenzamos capturando el JWT:

![[Pasted image 20260223200355.png]]

Bueno recordemos que tenemos que obtener una clave publica expuesta, esto lo podemos hacer mediante fuzzing pero con un poco de investigación vemos que el endpoint usual es:

![[Pasted image 20260223200648.png]]

la ruta es `/.well-known/jwks.json` en nuestro caso solo es directamente en `jwks.json`:
![[Pasted image 20260223200739.png]]

Bueno para verlo mejor podemos jugar desde consola con `curl`: 

![[Pasted image 20260223200941.png]]

como observamos la salida es mucho mas organizada y vemos mejor la clave publica, recordemos que la clave publica la usamos para verificar.

Recordando esto que es una clave asimétrica vamos a intentar forzar una verificación simétrica, usando el algoritmo `HS256`.

El problema principal es que en servidores mal configurados se usa la clave publica tanto para el algoritmo `RS256` como para `HS256`.

Lo primero que haremos es generar una clave asimétrica:

![[Pasted image 20260223201717.png]]

Como podemos observar esto nos genera contenido pero este contenido no es necesario porque tenemos el que esta expuesto de la clave publica por lo que es cuestión de representarlo allí en lugar de lo que se genera:

![[Pasted image 20260223201943.png]]

con eso listo le damos a ok.

Ahora vamos a dar clic derecho sobre la clave que generamos y vamos a copiarla:

![[Pasted image 20260223202127.png]]

y en el mismo decoder de Burp Suite lo vamos a pegar:

![[Pasted image 20260223202953.png]]

esto lo vamos a representar en base 64:

![[Pasted image 20260223203031.png]]

ya con esto es lo que vamos a usar como secret al crear el JWT simétrico:

![[Pasted image 20260223203202.png]]

ya con esto listo damos en ok y tenemos listo.

Por ultimo vamos a modificar 2 cosas, una es el algoritmo para que se `HS256` y el otro es de `wiener` a `administrator`:

![[Pasted image 20260223203419.png]]

En este punto lo que vamos a hacer es firmar el JWT con la clave simétrica y la usamos a ver si funciona:

![[Pasted image 20260223203526.png]]

Como podemos observar solo es cuestión de eliminar el usuario `carlos` para terminar:

![[Pasted image 20260223203558.png]]

### JWT authentication bypass via algorithm confusion with no exposed key

El propósito es el mismo del anterior laboratorio pero en este caso no vamos a contar una clave expuesta por lo tanto vamos a ver como se puede resolver.

Comenzamos capturando el JWT:

![[Pasted image 20260223205100.png]]

Lo que vemos de primera es que el token esta firmado por `RS256` siendo un algoritmo asimétrico y no tenemos acceso a la clave publica.

Ahora lo que sucede es que si tenemos ya una firma digital valida y podemos aprovecharnos de esto. Con ayuda de la herramienta que nos comparten, esta herramienta genera valores de n supuestamente validos los cuales de nuestro lado debemos validar, el objetivo es encontrar uno valido para mediante este aprovecharnos para obtener la clave publica y posteriormente vamos a poder generar la confunción de algoritmos igual que lo vimos anteriormente.

Para la herramienta copias dos tokens validos de dos sesiones distintas y procedemos a ejecutarla de la siguiente manera:

![[Pasted image 20260223205846.png]]

ya tenemos los dos JWT validos, por lo tanto vamos a ejecutar la herramienta:

![[Pasted image 20260223210550.png]]

La herramienta nos devuelve 2 cosas el valor de `n` y genera automáticamente un JWT el cual tenemos que probar a ver si es valido, si este es valido tendremos el valor correspondiente de n a usar para firmar un JWT malicioso.

Vemos si es valido:
![[Pasted image 20260223211021.png]]

Vemos que este es valido y pues ya tenemos el valor de `n` correcto.

Vamos a crear en JWT Editor la clave simétrica, recordemos que vamos a hacer una confusión de algoritmo por lo tanto al crear la clave simétrica le pasamos el valor de `n` correspondiente al token que usamos:

![[Pasted image 20260223211226.png]]

damos ok y listo.

Ahora cambiamos de `wiener` a `administrator` y pasamos de `RS256` a `HS256`:

![[Pasted image 20260223211330.png]]

Procedemos a firmar este JWT y a ponerlo a prueba:

![[Pasted image 20260223211413.png]]

Como vemos es valido y podemos eliminar al usuario `carlos` para terminar.

![[Pasted image 20260223211443.png]]

Algo a dejar claro es que el algoritmo no crackea el JWT sino que a partir de los tokens que se proporciona y de forma modular intenta generar valores de n validos.