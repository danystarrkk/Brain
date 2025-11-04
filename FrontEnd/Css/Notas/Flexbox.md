---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# [[Flexbox]]
Esto fue diseñado como un modelo unidimensional para la creación de layouts, Flexbox consta de dos ejes por lo que solo podemos distribuirlos en fila o columna. El valor por defecto es row al definir display: flex;. Los otros valores son _row-reverse, column y column-reverse_.
## Row o Row-reverse
Si escogemos este tipo de flex o no definimos nada y se aplica directamente row vamos a ver que los elementos hijos se colocaran de manera izquierda a derecha uno junto al otro:

![[Pasted image 20250124163705.png]]
## Column o Column-reverse
En el caso de definir este flex los elementos se colocarán de arriba hacia abajo:

![[Pasted image 20250124163736.png]]

Bueno Flexbox esta diseñado para alinear elementos en nuestros diseños por lo que no añade efectos de animación ni textos.

## Flex basis
Esta opción nos permite definir el tamaño de nuestros bloques en el espacio designado.

```css
.d-flex-10 .box {
  flex-basis: 33.3%;
}
```

Podemos usarlo también para definir un valor base, osea desde el cual comenzara a crecer en el caso de combinarlo por ejemplo con flex grow.
## Flex Wrap
Permite que el contenido si no tiene el espacio suficiente de forma horizontal cree una línea mas para los demás elementos y que no se deforme el estilo.

```css
.d-flex-10 {
  display: flex;
  flex-wrap: wrap;
}
```
## Flex Grow
Esto nos permite controla el factor de crecimiento, esto nos permite repartir de la forma en la que se le especifique por ejemplo si el factor de crecimiento es de 1 y tenemos 3 cajas dentro una de una de 100px para cada caja va a repartir de un pixel en un pixel a cada uno.

El valor por defecto de flex grow es de 0, es por esto que el contenido toma por defecto el ancho de lo que tiene dentro sobreentendiendo texto, imágenes ,etc.

```css
.d-flex-11 .box {
  flex-grow: 1;
}
```

esto si esta aplicado a un solo elemento o a todos los elementos dentro de siempre se va a repartir de forma equitativa, su uso es mas cuando lo definimos a otros elementos.

```css
.d-flex-11 .box:nth-child(1) {
  flex-grow: 3;
}


.d-flex-11 .box:nth-child(2) {
  flex-grow: 2;
}


.d-flex-11 .box:nth-child(3) {
  flex-grow: 1;
}
```

esto visual mente se vería:

![[Pasted image 20250729213842.png]]

*Nota:* Siempre toma como referencia el tamaño de contenedor.

## Flex Shrink
Esto nos permite controlar el factor de contracción, osea cuando el elemento se hace mas pequeño.

```css
.d-flex-12 .box:nth-child(3) {
  flex-shrink: 2;
}
```

con esto lo que hacemos es lo opuesto a flex grow que seria para el 3 bloque va a decrecer de dos en dos por lo tanto se hace mas pequeño de forma mas acelerada que los otros 2.


## Flex Shortand
No es mas que la unión de Flex grow basis y shrink y sintaxis es:

```css
.d-flex-13 .box {

  /* version larga */
  flex-basis: 33.3%;
  flex-grow: 1;
  flex-shrink: 0;

  /* version corta */
  flex: 1 0 33.3%; /* flex: grow shrink basis*/

}
```

