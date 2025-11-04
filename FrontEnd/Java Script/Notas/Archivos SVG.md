---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Archivos SVG

Este tipo de archivo los cuales son imágenes en vectores escalares a diferencia de otro tipo de archivos de imágenes nos permite modificar el mismo, es decir que a este tipo de archivos podríamos hasta llegar a animarlos, por lo que vamos a ver algunos métodos importantes de JavaScript que nos permitan realizar esto, pero primero vamos a ver paso a paso el cómo se hace y recrea este tipo de código:

Lo que solemos hacer primero es capturar mediante el DOM los elementos que se van a modificar como si fueran un archivo HTML cualquiera de la siguiente manera:

```jsx
const eyes = document.querySelector('.eyes');
const head = document.querySelector('.head');
const ears = document.querySelector('.ears');
const nose = document.querySelector('.nose');
const snout = document.querySelector('.snout');
```

Una vez hecho esto, lo siguiente sería inicializar las valores tanto de la posición del cursor como el del tamaño de la ventana de la siguiente manera:

```jsx
// Cursor
let cursorPosition = { x: 0, y: 0 }; // definimos los valores inciales del cursor
// Ventana, los datos se capturan al iniciar el script
let windowWidth = window.innerWidth; // almacena del tamaño de la ventana del navegador (ancho)
let windowHeight = window.innerHeight; // almacena el  tamaño de la ventana del navegador (alto)
```

De esta forma ya tenemos los valores inicialízales de los datos que requerimos.

Ahora debemos crear una función encargada de manejar el movimiento del ratón de la siguiente manera:

```jsx

// Esta es la función que ejecutaremos al mover el ranto,
// donde la variable postion almacenara la posición del raton
function mouseMov(position) {
  cursorPosition = { x: e.clientX, y: e.clientY } // mediane los metodos clientX/Y obtenemos,
  // la posición actual del raton.
}

```

En el caso de que sé deva implementar la captura táctil, lo vamos a hacer de la siguiente manera:

```jsx
// Esta es la función que ejecutaremos al hacer click en el tactil de un dispositivo
function touchmode(e) {
  cursorPosition = { x: e.targetTouches[0].offsetX, y: e.targetTouches[0].offsetY };
  // mediante la función targetTouches toma la posición del primer toque y actualiza la posición
  // con las coordenas del toque.
}
```

Lo anterior sería lo básico, pero ahora veremos diferentes usos que se les dará a estas configuraciones anteriores:

**Mover Elementos según la posición del cursor o el toque(touch):**

Para esto debemos realizar los siguientes:

```jsx
//Esta sera un afunción que ejecutaremos para que un elemento de la web se mueva dependiendo de la
// posicion del cursor.
const seguirCursor = (element, valX, valY) => {
// elment es el elemento que vamos a mover, valX y valY son los valores de escala que nos permite 
// determinar que tan lejos se moverá el elmeto respecto del cursor
const elementOffset = element.getBoundingClientRect(); // El metodo getBoundignClientRect() nos 
// permite obtener las dimenciones del elmento como: la distancia desde el borde de la ventana
// izquierda hasta el borde del elmeto izquierdo (x), tambien desde el borde de arrba hasta el
// borde de arriba del elemto (y), tambien el ancho del elemento (width) y el alto del elemento
// (height)
  const centerX = elementOffset.x + (elementOffset.width / 2);// calculamos su ancho sumando la 
  // distancia en x mas la mitad del ancho del elemento para obtener el centro del elemento.
  const centerY = elementOffset.y + (elementOffset.height / 2);// de la misma forma sumamos la 
  // distancia en y mas la mitad del alto del elemento para obtener la del mismo.
  const distanceleft = Math.round(((cursorPosition.x - centerX) * 100) / window.innerWidth);
  const distanctop = Math.round(((cursorPosition.y - centerY) * 100) / window.innerHeight);

  element.style.transform = `translate(${distanceleft / valX}px,${distanctop / valY}px)`;
  // por ultimo estamos aplicando un style al elmento para que cambie continuamente la posición.
}

```

Ya con esto crearemos una función en la cual vamos a verificar cada una de las partes del elemento con el propósito de comprobar y ejecutar la función de seguir Cursor cuando movemos el cursor por la web de la siguiente manera:

```jsx
const seguir = () => {
  if (ears) seguirCursor(ears, 0, 0);
  if (head) seguirCursor(head, 6, 6);
  if (eyes) seguirCursor(eyes, 1.8, 1.8);
  if (nose) seguirCursor(nose, 3, 3);
  if (snout) seguirCursor(snout, 3, 3);
}
```

Por último vamos a asociar cada función creada con un elemento para así poder ejecutarlas continuamente:

```jsx
window.addEventListener('resize', defWindow); // cada que la pantalla cambie reasigna las dimencines
// de la web.
window.addEventListener('mousemove', mouseMov); // cada que el cursor se mueva este ejecuta la función
// que creamos.
window.addEventListener('touchmove', touchmode); // Lo mismo pero para el touch de la pantalla.
```