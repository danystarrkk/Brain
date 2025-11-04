---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Medidas en CSS

Bueno tenemos diferentes tipos de medidas en css como son: px, em y rem. En la actualidad ya no es recomendado el uso de px ni em por lo que nuestra medida sera el rem. Con esto claro vamos a comprender que aunque los px son la medida mas fácil e intuitiva no tenemos una relación con los rem ya que no son lo mismo, con esto claro para nosotros poder tener una relación de rem y px vamos a agregar el siguiente código a nuestra hoja de estilos en cascada:

```css
html {
  box-sizing: border-box;
  font-size: 62.5%;
}

*,
*:before,
*:after {
  box-sizing: inherit;
}

body {
  font-size: 1.6rem;
}

```

Con estos dos valores hacemos que los rem sean super sencillos de usar ya que forzamos una relación la cual es 1rem = 10px.