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

