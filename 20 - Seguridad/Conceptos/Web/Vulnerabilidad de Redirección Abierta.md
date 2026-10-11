---
aliases:
  - Open Redirect
tags:
  - hacking/web
  - hacking/explotacion
  - herramientas/burpsuite
estado: 🟢 Terminado
---
# Vulnerabilidad de Redirección Abierta (Open Redirect)

La vulnerabilidad de redirección abierta, también conocida como **Open Redirect**, es una vulnerabilidad común en aplicaciones web que puede ser explotada por los atacantes para dirigir a los usuarios a sitios web maliciosos. Esta vulnerabilidad se produce cuando una aplicación web permite a los atacantes manipular la URL de una página de redireccionamiento para redirigir al usuario a un sitio web malicioso.

> [!info] ¿Cómo funciona un Open Redirect?
> Un Open Redirect ocurre cuando una aplicación web acepta un parámetro de URL que especifica una URL de destino a la que el usuario debe ser redirigido, pero no valida adecuadamente este parámetro. Esto permite a un atacante insertar una URL arbitraria, redirigiendo a la víctima a un sitio controlado por el atacante.

Por ejemplo, supongamos que una aplicación web utiliza un parámetro de redireccionamiento en una URL para dirigir al usuario a una página externa después de que se haya autenticado. Si esta URL no valida adecuadamente el parámetro de redireccionamiento y permite a los atacantes modificarlo, los atacantes pueden dirigir al usuario a un sitio web malicioso, en lugar del sitio web legítimo.

Un ejemplo de cómo los atacantes pueden explotar la vulnerabilidad de redirección abierta es mediante la creación de correos electrónicos de **phishing** que parecen legítimos, pero que en realidad contienen enlaces manipulados que redirigen a los usuarios a un sitio web malicioso. Los atacantes pueden utilizar técnicas de **ingeniería social** para convencer al usuario de que haga clic en el enlace, como ofrecer una oferta atractiva o una oportunidad única.

> [!warning] Impacto de un Open Redirect
> El impacto principal de un Open Redirect es facilitar ataques de phishing y spear-phishing. Un atacante puede construir una URL que parezca legítima (usando el dominio de la aplicación vulnerable) pero que redirija a un sitio malicioso. Esto puede llevar a la divulgación de credenciales, descarga de malware o ejecución de ataques de Cross-Site Scripting (XSS) si se combina con otras vulnerabilidades.

Para prevenir la vulnerabilidad de redirección abierta, es importante que los desarrolladores implementen medidas de seguridad adecuadas en su código, como la validación de las URL de redireccionamiento y la limitación de las opciones de redireccionamiento a sitios web legítimos. Los desarrolladores también pueden utilizar técnicas de codificación segura para evitar la manipulación de URL, como la codificación de caracteres especiales y la eliminación de caracteres no válidos.

> [!tip] Medidas de Prevención
> *   **Lista Blanca (Whitelist)**: Permitir redirecciones solo a un conjunto predefinido y estricto de dominios o rutas internas.
> *   **Validación Estricta**: Asegurarse de que el parámetro de redirección apunte a una URL dentro del mismo dominio o a un subdominio controlado.
> *   **Advertencia al Usuario**: Si la redirección es a un dominio externo legítimo, mostrar una página de advertencia al usuario antes de redirigirlo.
> *   **No usar parámetros de URL para redirecciones**: Si es posible, evitar el uso de parámetros de URL para especificar destinos de redirección.

## Notas Prácticas

![[{781A3A4D-0329-4D81-8683-31B056D085DE}.png]]

El botón que estamos observando nos permite movilizarnos a otra web, aplica un redireccionamiento para mandarnos a otro sitio:

![[Pasted image 20260227200408.png]]

Vamos a interceptar esto para ver cómo se está tramitando:

![[{8B5F7609-8398-4983-86C5-889A9712BA74}.png]]

Observamos que nos permite modificar el redireccionamiento. Si cambiamos por `google.es`:

![[Pasted image 20260227200628.png]]

![[Pasted image 20260227200638.png]]

Como vemos, esto funciona. El problema de esto es que en las páginas que tienen esta vulnerabilidad se pueden realizar peticiones cruzadas donde se le indica a la página con el Open Redirect que haga la petición a otra máquina o servidor. Esto es muy usado para armar botnets y ejecutar ataques de fuerza bruta.

> [!info] Peticiones Cruzadas y Botnets
> Un Open Redirect puede ser encadenado con otras vulnerabilidades o técnicas para amplificar su impacto. Por ejemplo, un atacante podría usar un Open Redirect para forzar a los navegadores de las víctimas a realizar peticiones a un servidor controlado por el atacante, o incluso a participar en ataques DDoS o de fuerza bruta como parte de una botnet, sin que la víctima lo sepa.

Para evitar este tipo de vulnerabilidades se usan ciertas protecciones. Veamos en una máquina similar cómo burlar una de ellas.

### Bypassing Filtros de Caracteres

Tenemos otra web y vamos directo a la captura por parte de Burp Suite:

![[Pasted image 20260227201130.png]]

Podemos ver que es prácticamente igual, pero al momento de modificarla vamos a ver lo siguiente:

![[Pasted image 20260227201223.png]]

Vemos que no nos permite usar el `.` (punto). Por lo tanto, para burlar esto podríamos implementar URL-encoding de la siguiente manera:

![[Pasted image 20260227201347.png]]

Como vemos, el servidor sabe que eso es un punto y no permite pasar. Pero, ¿y si hacemos URL-encode al `%` que tenemos allí?

> [!tip] URL Encoding Doble
> El doble URL encoding (`%252E` para un punto) es una técnica común para evadir filtros que solo decodifican la URL una vez. Si el servidor decodifica el `%25` a `%` y luego el `%2E` a `.`, el filtro inicial podría ser eludido. Esto es efectivo cuando el filtro se aplica antes de la decodificación completa de la URL.

![[Pasted image 20260227201441.png]]

Como observamos, ya logramos burlar el filtro y pasamos, y en el navegador es funcional.

### Bypassing Filtros de Puntos y Barras

En este caso, tenemos algo un poco más complicado y es que no podemos usar ni puntos ni barras (`.` ni `/`). La forma de burlarla es la misma con URL-encoding:

![[Pasted image 20260227201758.png]]

En ocasiones puede funcionar, pero cuidado, no porque estemos ya en el `302` significa que logramos pasar. Si continuamos el flujo, veremos en realidad que no logramos llegar:

![[Pasted image 20260227201932.png]]

Lo que podemos hacer es que para dominios con `https` no es necesario el uso de las `//` (doble barra), por lo que podemos hacer:

> [!tip] Redirecciones sin `//`
> Algunos navegadores y servidores interpretan URLs que comienzan con `https:` (o `http:`) sin las dobles barras (`//`) como válidas, asumiendo que el resto es el dominio. Por ejemplo, `https:google.com` puede ser interpretado como `https://google.com`. Esta es una técnica de bypass útil cuando las barras están filtradas.

![[Pasted image 20260227202040.png]]

Y en este caso, sí logramos pasar.