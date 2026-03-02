---
aliases: ["CSSI", "CSS Injection"]
tags: ["hacking/web"]
estado: 🟢 Terminado
---
# Inyecciones CSS (CSSI)

Las **Inyecciones CSS** (**CSSI**) son un tipo de vulnerabilidad web que permite a un atacante inyectar código CSS malicioso en una página web. Esto ocurre cuando una aplicación web confía en entradas no confiables del usuario y las utiliza directamente en su código CSS, sin realizar una validación adecuada.

> [!info] Mecanismo de Inyección CSS
> Una Inyección CSS se produce cuando la entrada del usuario no se sanea o escapa correctamente antes de ser incrustada en una hoja de estilo CSS o en un atributo `style` dentro del HTML. Esto permite al atacante cerrar la declaración CSS existente e introducir sus propias reglas CSS arbitrarias.

El código CSS malicioso inyectado puede alterar el estilo y diseño de la página, permitiendo a los atacantes realizar acciones como la **suplantación de identidad** (phishing) o el **robo de información confidencial**.

Las Inyecciones CSS (CSSI) pueden ser utilizadas por los atacantes como un vector de ataque para explotar vulnerabilidades de **Cross-Site Scripting** (**XSS**). Imagina que una aplicación web permite a los usuarios introducir texto en un campo de entrada que se muestra en una página web. Si el desarrollador de la aplicación no valida y filtra adecuadamente el texto introducido por el usuario, un atacante podría inyectar código malicioso en el campo de entrada, incluyendo código JavaScript.

> [!warning] CSS-Induced JavaScript Injection
> Aunque CSS no es un lenguaje de programación Turing-completo, ciertas propiedades CSS y selectores pueden ser manipulados para desencadenar la ejecución de JavaScript. Por ejemplo, el uso de `url()` en propiedades como `background-image` o `list-style-image` puede apuntar a recursos con esquemas `javascript:`, o el abuso de `expression()` en versiones antiguas de IE. Más modernamente, la interacción con `input` y `label` junto con selectores avanzados (`:checked`, `::before`, `::after`) puede exfiltrar datos o manipular el DOM de formas que faciliten un XSS.

Si el código CSS inyectado es lo "suficientemente complejo", puede hacer que el navegador web interprete el código como si fuera código JavaScript. Esto significa que el código CSS malicioso puede ser utilizado para inyectar código JavaScript en la página web, lo que se conoce como una inyección de JavaScript inducida por CSS (CSS-Induced JavaScript Injection).

Una vez que el código JavaScript ha sido inyectado en la página, este puede ser utilizado por el atacante para realizar un ataque de Cross-Site Scripting (XSS). Una vez en este punto, el atacante podría ser capaz de inyectar un script malicioso que robe las credenciales del usuario o que los redirija a una página web falsa, entre otros muchos posibles vectores.

## Notas Prácticas

![[Pasted image 20260227172723.png]]

Tenemos esta web, donde si ponemos el color nos lo representa:

![[Pasted image 20260227172746.png]]

Si revisamos el código fuente:

![[Pasted image 20260227172824.png]]

Nosotros controlamos cómo funciona el estilo. Podemos derivar esto a un [[Cross-Site Scripting XSS]] de la siguiente manera:

![[Pasted image 20260227172928.png]]

![[Pasted image 20260227172933.png]]