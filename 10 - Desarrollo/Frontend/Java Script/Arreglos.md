---
aliases: ["Arrays", "Arreglos en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Array Methods]]"
  - "[[Iteradores]]"
  - "[[Objetos]]"
  - "[[Strings]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Array"
---

# Arreglos

Un **arreglo** es una lista ordenada de valores bajo un mismo nombre. A diferencia de las variables normales (que guardan UN dato), el arreglo guarda **1 a n** elementos y permite acceder a cualquiera por su **índice** (posición, empezando en **0**).

> [!info] ¿"Mismo tipo"?
> Clásicamente se dice que un arreglo agrupa elementos del mismo tipo, pero en JavaScript un arreglo puede mezclar **todo**: strings, números, booleanos e incluso otros arreglos y objetos.

## Crear arreglos

```js
const numeros = [10, 20, 30];            // literales: la forma común y recomendada
const meses = new Array('Enero', 'Febrero', 'Marzo'); // constructor (evitar)

const deTodo = ['Hola', 10, true, 'Sí', numeros]; // arreglos anidados permitidos
console.log(deTodo);
console.table(numeros); // imprime en forma de tabla en la consola
```

> [!warning] Evita `new Array()`
> `new Array(10)` no crea `[10]`: crea un arreglo de **10 posiciones vacías**. Los literales `[...]` no tienen esa trampa.

## Acceder y modificar por índice

```js
const numeros = [10, 20, 30];

console.log(numeros[0]);  // 10 (primer elemento)
console.log(numeros[2]);  // 30 (tercero)
console.log(numeros[20]); // undefined: el índice no existe

console.log(numeros.length); // 3: cantidad de elementos

// Reasignar un valor
const meses = ['Enero', 'Febrero', 'Marzo', 'Mayo', 'Junio'];
meses[0] = 'Nuevo mes';

// Asignar un índice lejano NO rellena los huecos: crea el índice 7 directamente
meses[7] = 'Último mes';
console.log(meses); // 8 posiciones, con huecos en medio (undefined)
```

> [!tip] `const` no congela el contenido
> Un arreglo declarado con `const` **sí se puede modificar** (`meses[0] = ...`, `push`, `pop`). Lo que no se puede es **reasignar la variable** (`meses = []` daría error). Si quieres protegerlo del todo: `Object.freeze(meses)`.

## Métodos que modifican el arreglo (in situ)

```js
const meses2 = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo'];

meses2.push('Junio');      // agrega al final
meses2.unshift('Diciembre'); // agrega al comienzo
meses2.pop();              // elimina el último
meses2.shift();            // elimina el primero
meses2.splice(1, 1);       // desde el índice 1, elimina 1 elemento
meses2.splice(1, 0, 'X'); // desde el índice 1, elimina 0 y agrega 'X'

console.table(meses2);
```

| Método | Acción | ¿Retorna algo útil? |
| ------ | ------ | ------------------- |
| `push(...)` | agrega al final | nueva `length` |
| `unshift(...)` | agrega al inicio | nueva `length` |
| `pop()` | quita el final | el elemento quitado |
| `shift()` | quita el inicio | el elemento quitado |
| `splice(inicio, cant, ...)` | corta/reemplaza en medio | arreglo con lo quitado |

## Forma imperativa vs declarativa

La forma **imperativa** muta el arreglo original directamente. La **declarativa** crea arreglos nuevos con spread — sin tocar el original, lo que facilita depurar y es la base de la inmutabilidad que usa [[ReactJS - Fundamentos]]:

```js
const carrito = [];

function Producto(nombre, precio) {
  this.nombre = nombre;
  this.precio = precio;
}

const producto1 = new Producto('Celular', 500);
const producto2 = new Producto('Monitor', 110);
const producto3 = new Producto('Teclado', 50);

// Imperativo: muta el carrito
carrito.push(producto1, producto2);
carrito.unshift(producto3);

// Declarativo: produces copias con cambios, el original queda intacto
const conProducto4 = [...carrito, producto2];              // agregar al final
const conProducto5 = [producto1, ...carrito];               // agregar al comienzo
```

Hoy hasta se puede construir un objeto al vuelo sin `function` con `new`:

```js
const producto4 = { nombre: 'Mouse', precio: 20 };
```

## Desestructuración

Permite extraer valores del arreglo a variables con nombre:

```js
const numeros = [10, 20, 30, 40, 50];

const [segundo, tercero] = numeros;      // segundo = 10, tercero = 20
console.log(segundo, tercero);

const [primero, , , cuarto] = numeros;   // saltos con posiciones vacías
console.log(primero, cuarto);            // 10, 40

const [, , ...lista] = numeros;          // el resto desde la posición 2
console.log(lista);                      // [30, 40, 50]

// Valores por defecto
const [a = 1, b = 2] = [];
console.log(a, b); // 1, 2
```

## Recorrer arreglos

```js
const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audífonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
];

// for clásico: control total del índice
for (let i = 0; i < carrito.length; i++) {
  console.log(`${carrito[i].nombre} tiene un precio de ${carrito[i].precio}`);
}

// forEach: más limpio para recorrer sin necesitar el índice
carrito.forEach((producto) => {
  console.log(`${producto.nombre} tiene un precio de: ${producto.precio}`);
});

// forEach con índice cuando hace falta
carrito.forEach((producto, i) => console.log(`${i}: ${producto.nombre}`));

// map: recorre Y crea un arreglo nuevo con el resultado
const mensajes = carrito.map(
  (producto) => `${producto.nombre} tiene un precio de ${producto.precio}`
);
console.log(mensajes); // el carrito original no cambia
```

> [!warning] `map` no es para "hacer algo" con cada elemento
> Si no vas a usar el arreglo que retorna, usa `forEach`. `map` transforma; su resultado tiene el mismo largo que el original.

Ver [[Iteradores]] para `for...of`, `while` y el resto de bucles, y [[Array Methods]] para buscar, filtrar y reducir arreglos.

## Relacionadas

- [[Array Methods]] — `filter`, `find`, `reduce`, `includes` y compañía
- [[Iteradores]] — todas las formas de recorrer un arreglo
- [[Funciones]] — callbacks que reciben estos métodos
- [[Objetos]] — arreglos de objetos, el patrón más común
- [[Strings]] — métodos de la otra estructura secuencial
