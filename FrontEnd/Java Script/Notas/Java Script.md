---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Java Script
Nosotros podemos escribir código html en dos formas, dentro del html y de forma externa en un archivo. En cualquiera de las dos formas en las que escribamos código html tenemos como una buena práctica implementarlo antes del cierre del body.
## En el HTML
Nosotros para crear código en html vamos a hacerlo de la siguiente manera:

```html
<script>
    alert("Hola Mundo");
</script>
```

La etiqueta script es la que nos permite que el html sepa que tiene que interpretarlo como código JavaScript. La función alert es una función ya implementada.

Es mucho más recomendable usar un archivo externo mediante el cual implementemos el código de JavaScript.

## En un archivo Externo
Esta es la forma más recomendada de crear o escribir código html, lo que vamos a hacer es crear una carpeta con el nombre de `js`  y dentro del mismo podemos darle cualquier nombre al archivo de JavaScript, pero por recomendación se usa `app.js`, nuestro árbol de directorios debería tener 

![[Pasted image 20250125080131.png]]

ya con esto listo en el archivo que creamos podemos comenzar a escribir el código JavaScript de la siguiente forma:

```jsx
alert("Hola Mundo");
```

Como podemos ver, cuando tenemos un archivo externo ya no es necesario el uso de las etiquetas scripts, lo que debemos hacer ahora que tenemos listo nuestro archivo es enlazarlo al html de la siguiente forma:

```html
<script src="js/app.js"></script>
```

Ya en este punto se ejecutará correctamente el código de JavaScript en nuestra web.