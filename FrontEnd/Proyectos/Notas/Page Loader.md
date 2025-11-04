---
tags:
  - FrontEnd
Creado: 2025-01-27
Relacionado:
  - "[[SASS]]"
  - "[[Brain/FrontEnd/Html/HTML|HTML]]"
  - "[[Brain/FrontEnd/Gulp/GULP|GULP]]"
  - "[[JAVASCRIPT]]"
---
----------
# Page Loader
La idea es agregar una animación de carga antes de que cargue la web, no es necesario realizarlo siempre pero tenemos casos donde puede ser conveniente.

Lo primero será separar todo el contenido de nuestro cargador de página en un bloque aislado del contenido general de la web.

un ejemplo del html seria:

```html
<!DOCTYPE html>
<html lang="es">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Page Loader</title>
  <link rel="stylesheet" href="dist/css/app.css">

</head>

<body>

  <main class="content">

    <!-- aquí vamos a poner todo el contenido de nuestra web de forma normal -->
    <h1>Hola después del page Loader</h1>

  </main>

  <div class="page-loader">

    <div class="spinner"></div>
    <div class="txt">Cargando <br> espere...</div>

  </div>

  <script src="dist/js/app.js"></script>
</body>

</html>
```

eso seria todo en caso del html, ahora para crear nuestro page loader es cuestión de hacer uso de [[Animaciones]] por lo que no es nada que no se sepa ya, vamos en este punto a dar un ejemplo del código scss para que se sepa como se creo:

```scss
.page-loader {
  width: 100%;
  height: 100vh;
  position: absolute;
  background: #272727;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  gap: 4rem;
  z-index: 10;

  .txt {
    color: #666;
    text-align: center;
    text-transform: uppercase;
    letter-spacing: 0.3rem;
    font-weight: bold;
    line-height: 1.5;
  }

  .spinner {
    width: 8rem;
    height: 8rem;
    background: #fff;
    border-radius: 50%;
    -webkit-animation: sk-scaleout 1s infinite ease-in-out;
    animation: sk-scaleout 1s infinite ease-in-out;
  }
}

@-webkit-keyframes sk-scaleout {
  0% {
    -webkit-transform: scale(0);
    transform: scale(0);
  }

  100% {
    -webkit-transform: scale(1);
    transform: scale(1);
    opacity: 0;
  }
}
```

Como vemos no es nada complicado, en este punto lo que vamos a hacer es que con ayuda de [[JAVASCRIPT]] controlar el tiempo que dura el page loader visible.

```js
setTimeout(function () {
  document.querySelector('.page-loader').remove();
}, 3000)
```