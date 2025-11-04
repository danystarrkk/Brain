---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Css Grid
Esto nos permite definir la ubicación y el tamaño de los contenidos de los elementos de la web. En css grid el contenido se agrupa dentro de un área definida. En cierta forma es como una tabla de html, con la ventaja de mayor flexibilidad y control sobre el diseño.

Una comparación entre css grid y Flexbox es que grid combina tanto row como column al diferencia de Flexbox el cual solo puede tener ya sea row o column:

![[Pasted image 20250124170628.png]]

Bueno tenemos que entender que ninguno es mejor que el otro ya que uno puede ser mas fácil de integrar que el otro y además se puede utilizar los dos juntos en un mismo diseño.


## Column Span y Row Span
Nosotros podemos hacer que un elemento tome mas de un espacio y para esto usamos el span, su sintaxis es la siguiente:
```css
.grid-4 .box:nth-child(1) {
  grid-column: 2/ span 2; /* comienza en la 2 y termina en la segunda, lo mismo que poner talves 2/4 */
}
```

## Grid Shorthand
Esto es una abreviación para la definición de nuestras filas y columnas, su sintaxis es:
```css
.grid-5 {
  display: grid;
  /* version larga */
  grid-template-rows: repeat(2, 300px);
  grid-template-columns: repeat(3, 300px);

  /* version corta */
  grid: repeat(2, 300px) / repeat(3, 300px);
  /* grid: valor row / valor column */
}

```


## Grid Autoflow
El propósito de esta opción es evitar los espacios libres al fijar un elemento, osea cuando tenemos algo parecido a lo siguiente:

![[Pasted image 20250731183018.png]]

como vemos se posiciono al bloque 2 en la columna de 1 a 2 pero esto deja un espacio y desordena todo, nosotros para solucionarlo vamos a definir el autoflow de la siguiente manera:

```css
.grid-6 {
  display: grid;
  grid: repeat(2, 300px) / repeat(3, 300px);
  grid-auto-flow: dense;
}

.grid-6 .box:nth-child(2) {
  grid-column: 1 / 2;
}
```

Tenemos 3 opciones, la mas optima para solucionar este tipo de problemas es `dense` pero es bueno saber que también tenemos `column` y `row` donde dependiendo de donde se de el salto podemos usar cualquiera pero de forma general se usa `dense`. Ahora nuestro panel queda de la siguiente manera:

![[Pasted image 20250731183309.png]]

## Grid gap
Este permite agregar una separación a nuestros elementos y su sintaxis corta es:
```css
.grid-6 {
  display: grid;
  grid: repeat(2, 1fr) / repeat(3, 1fr);
  gap: 4rem 3rem; /* gap: valor-row valor-column;
}
```

## Grid Areas
Esto es muy útil si queremos dar nombres y posicionar los elementos en lugar de las lineas, ademas de moldear de forma mas gráfica nuestros bloques. una configuración de esto seria:

```css
.grid-9 {
  display: grid;
  height: 120rem;
  /* definimos 3 columnas */
  grid-template-columns: repeat(3, 1fr);
  /* asignamos nombres a cada columan */
  grid-template-areas: "header header header"
    "nav nav nav"
    "contenido contenido sidebar"
    "footer footer footer";

  grid-template-rows: 2fr 1fr 4fr 2fr;
  gap: 2rem;
}

.grid-9 .box {
  margin: 0;
}

.grid-9 .box:nth-child(1) {
  grid-area: header;
}

.grid-9 .box:nth-child(2) {
  grid-area: nav;
}

.grid-9 .box:nth-child(3) {
  grid-area: contenido;
}

.grid-9 .box:nth-child(5) {
  grid-area: footer;
}
```


## Grid Template
Esta es una forma mucho mas resumida de usar grid areas y es de la siguiente manera:
```css
.grid-10 {
  display: grid;
  height: 120rem;
  /* definimos 3 columnas */
  grid-template-columns: repeat(3, 1fr);
  /* asignamos nombres a cada columan y el tamaño de la fila */
  grid-template: "header header header" 2fr
    "nav nav nav" 1fr
    "contenido contenido sidebar" 4fr
    "footer footer footer" 2fr / 1fr 1fr 1fr
    /*al final asignamos el tamaño de las columnas, no funciona con repeat()*/
  ;
  gap: 2rem;
}

.grid-10 .box {
  margin: 0;
}

.grid-10 .box:nth-child(1) {
  grid-area: header;
}

.grid-10 .box:nth-child(2) {
  grid-area: nav;
}

.grid-10 .box:nth-child(3) {
  grid-area: contenido;
}

.grid-10 .box:nth-child(5) {
  grid-area: footer;
}
```

## Alineación en Grid
Al igual que en flex vamos a podemos alinear los elementos con `align-items` la cual nos permite alinear las cosas de forma vertical y otra opción para lo mismo es `place-content` y otra mas seria `align-content`.

```css
.grid-11 {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  height: 30rem;
  align-items: center;
}
```

## Autofill y Autofit
Autofill nos permite generar columnas mientras aun tengamos espacio en nuestro contenedor así estas queden vacías.

Autofit lo que hace es generar las columnas del tamaño definido pero solo para los elementos que tengamos.

Su sintaxis sera la siguiente:
```css
.grid-12 {
  display: grid;
  grid-template-columns: repeat(auto-fill, 200px);
  grid-template-columns: repeat(auto-fit, 200px);
}
```

De por si no funciona con facciones pero podemos adaptarlo de la siguiente manera:
```css
.grid-12 {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
```

lo que pasa en esa función es que con ayuda de minmax(min, max) definimos como mínimo `200px` y como máximo `1fr`.