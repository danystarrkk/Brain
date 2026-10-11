---
aliases: ["SVG con JavaScript", "Animar SVG"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Document Object Model (DOM)]]"
  - "[[Eventos]]"
fuente: "https://developer.mozilla.org/es/docs/Web/SVG/Tutorial"
---

# Archivos SVG con JavaScript

**SVG** (*Scalable Vector Graphics*) describe imágenes con **fórmulas** (líneas, círculos, paths) en vez de píxeles. Esa es su gran ventaja: al no tener resolución fija, **se pueden escalar y animar programáticamente** sin perder calidad. Como el SVG vive embebido en el HTML, es parte del [[Document Object Model (DOM)|DOM]] y se manipula con los mismos métodos de selección.

El ejemplo de esta nota es un avatar cuyas partes (orejas, cabeza, ojos, hocico) **siguen al cursor** del mouse o al dedo en pantallas táctiles.

## 1. Capturar las partes del SVG

Lo primero es seleccionar los elementos por clase, como cualquier HTML:

```js
const eyes = document.querySelector('.eyes');
const head = document.querySelector('.head');
const ears = document.querySelector('.ears');
const nose = document.querySelector('.nose');
const snout = document.querySelector('.snout');
```

## 2. Estado inicial: cursor y tamaño de ventana

```js
// Posición del cursor (se actualiza en cada movimiento)
let cursorPosition = { x: 0, y: 0 };

// Tamaño de la ventana, capturado al iniciar el script
let windowWidth = window.innerWidth;
let windowHeight = window.innerHeight;

// Al redimensionar hay que volver a medir, o los cálculos quedan desfasados
const defWindow = () => {
  windowWidth = window.innerWidth;
  windowHeight = window.innerHeight;
};
```

## 3. Funciones que actualizan la posición del cursor

```js
// Movimiento del mouse: e es el evento que dispara mousemove
const mouseMov = (e) => {
  // clientX/clientY: coordenadas del cursor relativas a la ventana
  cursorPosition = { x: e.clientX, y: e.clientY };
};

// Variante táctil: el primer dedo sobre la pantalla
const touchMode = (e) => {
  cursorPosition = {
    x: e.targetTouches[0].offsetX,
    y: e.targetTouches[0].offsetY,
  };
};
```

> [!warning] El parámetro del handler es el evento
> El error clásico es declarar `function mouseMov(position)` pero usar `e` por dentro: `e` no existe en ese scope y salta un `ReferenceError`. El parámetro y el nombre usado dentro deben coincidir.

## 4. Mover un elemento según el cursor

La idea: medir dónde está el elemento, comparar su centro con la posición del cursor y aplicar un `transform` de desplazamiento:

```js
const seguirCursor = (element, valX, valY) => {
  // Dimensiones y posición del elemento respecto de la ventana:
  // x, y (borde superior izquierdo), width y height
  const elementOffset = element.getBoundingClientRect();

  // Centro exacto del elemento
  const centerX = elementOffset.x + elementOffset.width / 2;
  const centerY = elementOffset.y + elementOffset.height / 2;

  // Distancia cursor-centro, normalizada al tamaño de la ventana (en %)
  const distanceLeft = Math.round(((cursorPosition.x - centerX) * 100) / window.innerWidth);
  const distanceTop = Math.round(((cursorPosition.y - centerY) * 100) / window.innerHeight);

  // valX/valY amortiguan el movimiento: cuanto más grande, menos se mueve
  element.style.transform = `translate(${distanceLeft / valX}px, ${distanceTop / valY}px)`;
};
```

Cada parte del rostro se mueve con su propio factor de amortiguación: las orejas apenas se mueven (`0` = casi fijas), los ojos casi siguen al cursor (`1.8`), y nariz y hocico quedan en el medio:

```js
const seguir = () => {
  if (ears) seguirCursor(ears, 0, 0);
  if (head) seguirCursor(head, 6, 6);
  if (eyes) seguirCursor(eyes, 1.8, 1.8);
  if (nose) seguirCursor(nose, 3, 3);
  if (snout) seguirCursor(snout, 3, 3);
};

seguir(); // una primera posición inicial
```

Los `if (ears)` protegen contra elementos ausentes: `querySelector` retorna `null` si la clase no existe, y llamar un método sobre `null` lanzaría un error.

## 5. Vincular los eventos

Por último se conectan las funciones a los eventos del [[Eventos|DOM]]:

```js
window.addEventListener('resize', defWindow);  // al cambiar el tamaño de la ventana
window.addEventListener('mousemove', mouseMov); // al mover el mouse
window.addEventListener('touchmove', touchMode); // al mover el dedo (móvil)
```

> [!tip] Rendimiento en `mousemove`
> Los handlers de `mousemove`/`scroll` se disparan decenas de veces por segundo. Si el cálculo es pesado, limita la frecuencia con `requestAnimationFrame` o con *throttling*, o la página se sentirá lenta en dispositivos modestos.

## Ideas para extender el ejemplo

- Cambiar la opacidad o el color de alguna parte según la distancia al cursor.
- Sumar `deviceorientation` para que el rostro siga la inclinación del celular.
- Animar con CSS las propiedades que no dependen del cursor (parpadeo, respiración), y dejar el `transform` de posición para JavaScript.

## Relacionadas

- [[Document Object Model (DOM)]] — el árbol que representa el SVG
- [[Eventos]] — `mousemove`, `touchmove` y compañía
- [[Acceder a Html]] — selección de elementos y modificación de estilos
