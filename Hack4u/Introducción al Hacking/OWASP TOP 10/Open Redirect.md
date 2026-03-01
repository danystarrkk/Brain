-------
La vulnerabilidad de redirección abierta, también conocida como **Open Redirect**, es una vulnerabilidad común en aplicaciones web que puede ser explotada por los atacantes para dirigir a los usuarios a sitios web maliciosos. Esta vulnerabilidad se produce cuando una aplicación web permite a los atacantes manipular la URL de una página de redireccionamiento para redirigir al usuario a un sitio web malicioso.

Por ejemplo, supongamos que una aplicación web utiliza un parámetro de redireccionamiento en una URL para dirigir al usuario a una página externa después de que se haya autenticado. Si esta URL no valida adecuadamente el parámetro de redireccionamiento y permite a los atacantes modificarlo, los atacantes pueden dirigir al usuario a un sitio web malicioso, en lugar del sitio web legítimo.

Un ejemplo de cómo los atacantes pueden explotar la vulnerabilidad de redirección abierta es mediante la creación de correos electrónicos de **phishing** que parecen legítimos, pero que en realidad contienen enlaces manipulados que redirigen a los usuarios a un sitio web malicioso. Los atacantes pueden utilizar técnicas de **ingeniería social** para convencer al usuario de que haga clic en el enlace, como ofrecer una oferta atractiva o una oportunidad única.

Para prevenir la vulnerabilidad de redirección abierta, es importante que los desarrolladores implementen medidas de seguridad adecuadas en su código, como la validación de las URL de redireccionamiento y la limitación de las opciones de redireccionamiento a sitios web legítimos. Los desarrolladores también pueden utilizar técnicas de codificación segura para evitar la manipulación de URL, como la codificación de caracteres especiales y la eliminación de caracteres no válidos.


## Notas Practica

![[{781A3A4D-0329-4D81-8683-31B056D085DE}.png]]

El botón que estamos observando nos permite movilizarnos a otra web, aplica un redirect para mandarnos a otro sitio:

![[Pasted image 20260227200408.png]]

Vamos a interceptar esto para ver como se esta tramitando:

![[{8B5F7609-8398-4983-86C5-889A9712BA74}.png]]


Observamos que nos permite modificar el redirect donde si cambiamos por google.es:

![[Pasted image 20260227200628.png]]

![[Pasted image 20260227200638.png]]

como vemos esto funciona, el problema de esto es que las paginas que tiene esto se puedo realizar peticiones cruzadas donde yo le digo a la pagina con el open redirect que haga la petición a otra maquina o servidor, esto es muy usado para armar bootnets y ejecutada en ataques de fuerza bruta. 

Para evitar este tipo de vulnerabilidades se usan ciertas protecciones, veamos en una máquina similar como burlar una de ellas.

Tenemos otra web y vamos directo a la captura por parte de Burp Suite:

![[Pasted image 20260227201130.png]]

Podemos ver que es prácticamente igual pero al momento de modificarla vamos a ver lo siguiente:

![[Pasted image 20260227201223.png]]

vemos que no nos permite usar el `.` por lo tanto para burlar esto podríamos implementar url-encode de la siguiente manera:

![[Pasted image 20260227201347.png]]

como vemos el servidor sabe que eso es un punto y pues no permite pasar pero y si hacemos url-encode al `%` que tenemos allí?

![[Pasted image 20260227201441.png]]

como observamos ya logramos burlar el filtro y pasamos, y en el navegador es funcional.

Bueno en este caso tenemos algo un poco mas complicado y es que no podemos usar ni puntos ni barras y la forma de burlarla es la misma con url-encode:

![[Pasted image 20260227201758.png]]

En ocasiones puede funcionar pero cuidado no porque estemos ya en el `302` significa que logramos pasar y es que si continuamos el flujo veremos en realidad que no logramos llegar:
![[Pasted image 20260227201932.png]]

Lo que podemos hacer es que para dominios con `https` no es necesario el uso de las `//` por lo que podemos hacer:

![[Pasted image 20260227202040.png]]

y en este caso si logramos pasar.