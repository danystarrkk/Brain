---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[Brain/FrontEnd/Html/HTML|HTML]]"
---
----------
# Modernizr
Para agregar es cuestión de buscar la pagina de modernizr y en la misma vamos buscar por webp, copiamos el código de webp y dentro de nuestro proyecto vamos crear una carpeta con una archivo js llamado `modernizr.js` ya con este archivo listo vamos a agregarlo a nuestro html, la recomendación es siempre agregar los archivos js antes de cerrar el body y lo agregaremos de la siguiente manera:

```html
  <script src="js/modernizr.js"></script>
```

Al cargar nuestra pagina web en un navegador e irnos al inspeccionar veremos en la cabecera si el navegador que usamos soporta o no webp agregando las clases de `webp` o `no-webp` y mediante estas podremos modificar el css para que cargue una imagen u otra dependiendo de la clase:

```css
.webp .header{
  background-image: url(../img/banner.webp);
}

.no-webp .header{
  background-image: url(../img/banner.jpg);
}
```