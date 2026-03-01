-----
El **Client-Side Template Injection** (**CSTI**) es una vulnerabilidad de seguridad en la que un atacante puede inyectar código malicioso en una **plantilla de cliente**, que se ejecuta en el **navegador** del usuario en lugar del servidor.

A diferencia del **Server-Side Template Injection** (**SSTI**), en el que la plantilla de servidor se ejecuta en el servidor y es responsable de generar el contenido dinámico, en el CSTI, la plantilla de cliente se ejecuta en el navegador del usuario y se utiliza para generar contenido dinámico en el lado del cliente.

Los atacantes pueden aprovechar una vulnerabilidad de CSTI para inyectar código malicioso en una plantilla de cliente, lo que les permite ejecutar comandos en el navegador del usuario y obtener acceso no autorizado a la aplicación web y a los datos sensibles.

Por ejemplo, imagina que una aplicación web utiliza plantillas de cliente para generar contenido dinámico. Un atacante podría aprovechar una vulnerabilidad de CSTI para inyectar código malicioso en la plantilla de cliente, lo que permitiría al atacante ejecutar comandos en el navegador del usuario y obtener acceso no autorizado a los datos sensibles de la aplicación web.

Una derivación común en un ataque de Client-Side Template Injection (CSTI) es aprovecharlo para realizar un ataque de **Cross-Site Scripting** (**XSS**).

Una vez que un atacante ha inyectado código malicioso en la plantilla de cliente, puede manipular los datos que se muestran al usuario, lo que le permite ejecutar código JavaScript en el navegador del usuario. A través de este código malicioso, el atacante puede intentar robar la cookie de sesión del usuario, lo que le permitiría obtener acceso no autorizado a la cuenta del usuario y realizar acciones maliciosas en su nombre.

Para prevenir los ataques de CSTI, los desarrolladores de aplicaciones web deben validar y filtrar adecuadamente la entrada del usuario y utilizar herramientas y frameworks de plantillas seguros que implementen medidas de seguridad para prevenir la inyección de código malicioso.

## Pruebas

![[Pasted image 20260227102320.png]]

Veamos que hace la web:

![[Pasted image 20260227102420.png]]   

En si el problema nace de como la web realiza o se encarga de cargar el contenido, nosotros tenemos que suponer todo en pruebas de penetración pero en este caso al ser entornos controlados podemos ver como el index actual que es de la siguiente manera:

![[Pasted image 20260227102717.png]]

como podemos observar el principal problema es la forma en la que usa par representar el contenido, esta sintaxis es vulnerable a interpretar el contenido que se le paso, por ejemplo podemos intentar ejecutar operatorias para comprobarlo `{{3*4}}` y el resultado debe de ser `12` :
![[Pasted image 20260227102850.png]]

Observamos que en efecto esto es vulnerable pero eso no quiere decir que notros al igual que con [[Server-Side Template Injection (SSTI)]] podemos directamente cargar archivos internos, esto no es tan fácil.

En este caso tenemos que buscar y verificar diferentes payloads que nos permiten hacer diferentes cosas, pero para esto necesitamos saber las tecnologías que se están usando:
![[{02ADA119-9446-4023-AA0A-68918F4338D6}.png]]

vemos que se implementa angular por lo que tenemos que buscar payloads pero para angular:

![[Pasted image 20260227103815.png]]

en este caso usamos este, tomando en cuenta que el payload hace referencia de arriba hacia abajo el titulo.

![[Pasted image 20260227103850.png]]

a partir de allí podemos itentar representar lo que queramos pero no siempre es funcional como en este caso, si intentamos hacer un `Hacked` veremos que no podemos:
![[Pasted image 20260227104007.png]]

Lo enviamos y veremos que:

![[Pasted image 20260227104033.png]]

Como logramos observar no funciono.

Entonces lo que tenemos que hacer es representar los caracteres de forma alternativa y primero tenemos que probar si la web lo acepta usando la consola del navegador:

![[Pasted image 20260227104320.png]]

Como podemos observar mediante una función nosotros podemos representar primero los caracteres en numérico y pasarlos a posterior a letras, para generar nuestra cadena numérica juguemos con `python`:

![[Pasted image 20260227104616.png]]

como podemos observar ya tenemos cada uno de los caracteres en números, ahora si podemos intentar crear una cadena con ellos de la siguiente manera:

![[{86D1AB33-643E-4E6C-80BA-BD6DC25BB455}.png]]

ya esto podemos copiarlo directo en nuestra carga útil de la siguiente manera:
![[Pasted image 20260227104856.png]]

y esto ya es funcional:

![[Pasted image 20260227104905.png]]