---
aliases: ["Métodos de arreglo", "Array methods"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Arreglos]]"
  - "[[Iteradores]]"
  - "[[Funciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Array"
---

# Array Methods

Los **métodos de arreglo** son funciones que ya vienen con cada arreglo y resuelven las operaciones más comunes: buscar, filtrar, transformar y combinar. Todos reciben una función callback — ver [[Funciones]] — y se usan con arrow functions para mantener el código corto.

Datos de ejemplo para toda la nota:

```js
const meses = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio', 'Julio'];

const carrito = [
  { nombre: 'Monitor 27 Pulgadas', precio: 500 },
  { nombre: 'Televisión', precio: 100 },
  { nombre: 'Tablet', precio: 200 },
  { nombre: 'Audífonos', precio: 300 },
  { nombre: 'Teclado', precio: 400 },
  { nombre: 'Celular', precio: 700 },
];
```

## ¿Existe un valor? `includes`, `some` y `every`

```js
// includes: comparación por valor, ideal para arreglos simples de primitivos
console.log(meses.includes('Enero')); // true

// some: "¿ALGÚN elemento cumple la condición?" — perfecto con objetos
const existe = carrito.some((producto) => producto.nombre === 'Celular'); // true

// some también sirve con primitivos
console.log(meses.some((mes) => mes === 'Mayo')); // true

// every: "¿TODOS los elementos cumplen la condición?"
const todosBaratos = carrito.every((producto) => producto.precio < 1000); // true
```

| Método | Pregunta que resuelve | Retorna |
| ------ | --------------------- | ------- |
| `includes(valor)` | ¿Este valor está en el arreglo? | `boolean` |
| `some(cb)` | ¿Algún elemento cumple? | `boolean` |
| `every(cb)` | ¿Todos cumplen? | `boolean` |

## Encontrar la posición: `findIndex`

```js
// Con forEach hay que guardar el índice a mano
meses.forEach((mes, i) => {
  if (mes === 'Mayo') console.log(i);
});

// Con findIndex es directo; retorna -1 si no encuentra
const indice = meses.findIndex((mes) => mes === 'Abril');
console.log(indice !== -1 ? `El índice de Abril es ${indice}` : 'Mes no encontrado');

const indiceCelular = carrito.findIndex((p) => p.nombre === 'Celular');
console.log(indiceCelular !== -1 ? `El índice de Celular es ${indiceCelular}` : 'Valor no encontrado');
```

> [!warning] El -1 rompe las comparaciones ingenuas
> Si el elemento está en la **posición 0**, una comprobación `indice > 0` daría `false` y reportaría "no encontrado" aunque sí exista. Compara siempre contra `-1`: `indice !== -1`.

## Sumar con acumulador: `forEach` vs `reduce`

```js
// Imperativo: declarar un acumulador y mutarlo
let total = 0;
carrito.forEach((producto) => (total += producto.precio));
console.log(total); // 2200

// Declarativo: reduce acumula SIN variable externa
// (acumulador, valor actual) => nuevo acumulador; el 0 es el valor inicial
const totalReduce = carrito.reduce((acumulador, producto) => acumulador + producto.precio, 0);
console.log(totalReduce); // 2200
```

El tercer argumento de `reduce` (el `0`) es el **valor inicial** del acumulador: olvidarlo con arreglos de objetos produce `NaN` o un `"[object Object]300"` rarísimo.

## Filtrar: `filter`

`filter` crea un **arreglo nuevo** con los elementos que cumplen la condición; el original queda intacto (inmutabilidad — ver [[Arreglos]]).

```js
console.log(carrito.filter((p) => p.precio > 400));  // Monitor y Celular
console.log(carrito.filter((p) => p.precio < 600));  // todo menos Celular

// Muy útil para "quitar" algo de un carrito de compras
const sinAudifonos = carrito.filter((p) => p.nombre !== 'Audífonos');
```

## Buscar el elemento: `find`

```js
// find retorna el PRIMER elemento que cumple la condición (o undefined)
const resultado = carrito.find((producto) => producto.nombre === 'Tablet');
console.log(resultado); // { nombre: 'Tablet', precio: 200 }

// Lo mismo con forEach obliga a esto (innecesariamente verboso):
let encontrado = null;
carrito.forEach((producto, indice) => {
  if (producto.nombre === 'Tablet') encontrado = carrito[indice];
});
```

| Método | Retorna | Caso de uso |
| ------ | ------- | ----------- |
| `find(cb)` | el elemento o `undefined` | obtener UN objeto por su propiedad |
| `findIndex(cb)` | índice o `-1` | necesito la posición para modificar/eliminar |
| `filter(cb)` | arreglo (posiblemente vacío) | necesito TODOS los que cumplan |

## Combinar arreglos: `concat` y spread

```js
const meses2 = ['Agosto', 'Septiembre'];
const meses3 = ['Octubre', 'Noviembre', 'Diciembre'];

// concat une arreglos (y acepta valores sueltos)
const resultado = meses.concat(meses2, meses3, 'OtroMes');

// Spread: la forma moderna y más legible
const resultado2 = [...meses, ...meses2, ...meses3, 'OtroMes'];

// Copiar sin mutar el original: programación funcional
const copia = [...meses, 'Julio'];

// Combinar arreglos de objetos también funciona
const producto = { nombre: 'Disco Duro', precio: 300 };
const carrito2 = [...carrito, producto];
```

> [!warning] `concat` vs spread con strings
> `[...meses, 'OtroMes']` agrega `'OtroMes'` como **un** elemento, pero `...'OtroMes'` sin comillas (o spreadyando un string directamente) separa el texto **en cada letra**. Spreadyando un objeto plano da error: los objetos no son iterables.

## Relacionadas

- [[Arreglos]] — crear, acceder y mutar arreglos
- [[Iteradores]] — `for`, `for...of`, `forEach` y los demás bucles
- [[Funciones]] — callbacks y arrow functions, la base de estos métodos
- [[Objetos]] — trabajar con los objetos dentro del arreglo
