---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Material Icons

Vamos a ver la forma en la cual podemos usar material icons, esto es esencial ya que un muchas webs tendremos que hacer uso de lo que son iconos y esta seria la forma correcta y gratis de hacerlo.

1. Lo primero es cargar la referencia en nuestro html en la parte del header, recordemos que los links tenemos que agregarlos siempre antes del css para no tener conflictos:

    ```html
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <link href="<https://fonts.googleapis.com/icon?family=Material+Icons>" rel="stylesheet"> <!-- LInk para los iconos-->
      <title>Material Icons</title>
    
    </head>
    ```

2. Ya con la referencia lista en nuestro html podemos continuar agregando el estilo que tendrán los iconos en la web dentro de nuestro css de la siguiente manera, tenemos que saber que esto es solo una base pre configurada para los iconos, por lo tanto podremos ajustarlos a nuestro gusto posteriormente:

    ```css
    .material-icons {
      font-family: 'Material Icons';
      font-weight: normal;
      font-style: normal;
      font-size: 24px;  /* Preferred icon size */
      display: inline-block;
      line-height: 1;
      text-transform: none;
      letter-spacing: normal;
      word-wrap: normal;
      white-space: nowrap;
      direction: ltr;
    
      /* Support for all WebKit browsers. */
      -webkit-font-smoothing: antialiased;
      /* Support for Safari and Chrome. */
      text-rendering: optimizeLegibility;
    
      /* Support for Firefox. */
      -moz-osx-font-smoothing: grayscale;
    
      /* Support for IE. */
      font-feature-settings: 'liga';
    }
    ```

3. Ya en este punto podemos agregar iconos en nuestro html, si revisamos lo iconos encontraremos los nombres, es cuestión de poner el nombre del icono y asignarle la clase de `material-icons` para que este adapte el estilo definido en esa clase.