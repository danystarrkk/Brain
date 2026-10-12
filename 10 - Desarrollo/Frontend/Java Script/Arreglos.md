---
tags:
  - desarrollo
  - frontend
  - arreglos
tipo: concepto
estado: terminado
---

# Arreglos

Un **arreglo** es una lista ordenada de valores bajo un mismo nombre. A diferencia de las variables normales (que guardan UN dato), el arreglo guarda **1 a n** elementos y permite acceder a cualquiera por su **índice** (posición, empezando en **0**).

> [!info] ¿"Mismo tipo"?
> Clásicamente se dice que un arreglo agrupa elementos del mismo tipo, pero en JavaScript un arreglo puede mezclar **todo**: strings, números, booleanos e incluso otros arreglos y objetos. Internamente es un objeto especializado: sus propiedades son índices numéricos + la propiedad `length`.

```js
const numeros = [10, 20, 30];
console.log(typeof numeros);        // "object" — no existe el tipo "array"
console.log(Array.isArray(numeros)); // true — la comprobación correcta
```

> [!warning] `typeof []` es `"object"`
> `[] instanceof Array` también funciona, pero falla con arreglos de otros realms (iframes, ventanas de prueba). `Array.isArray(arr)` es la única comprobación robusta.

## Crear arreglos

```js
const numeros = [10, 20, 30];            // literales: la forma común y recomendada
const meses = new Array('Enero', 'Febrero', 'Marzo'); // constructor (evitar)

const deTodo = ['Hola', 10, true, 'Sí', numeros]; // arreglos anidados permitidos
console.log(deTodo);
console.table(numeros); // imprime en forma de tabla en la consola
```

### El constructor y sus tres caras

```js
console.log(new Array(3));        // [ <3 empty items> ] ← tres HUECOS, no [3]
console.log(Array.of(3));         // [ 3 ] ← un elemento con valor 3
console.log(Array.from('abc'));   // [ 'a', 'b', 'c' ] ← desde cualquier iterable
console.log(Array.from({ 0: 'a', 1: 'b', length: 2 })); // array-like → ['a','b']
```

| Forma | Resultado | Uso |
|---|---|---|
| `[…]` literal | arreglo con los elementos dados | **siempre preferido** |
| `new Array(n)` con UN número | arreglo de `n` huecos | trampa clásica |
| `new Array(a, b, c)` | `[a, b, c]` | igual al literal; innecesario |
| `Array.of(x)` | `[x]` (nunca huecos) | equivalente literal seguro |
| `Array.from(iterable)` | copia plana desde iterable/array-like | mapear en el acto: `Array.from(s, f)` |

> [!warning] Evita `new Array()`
> `new Array(10)` no crea `[10]`: crea un arreglo de **10 posiciones vacías**. Los literales `[...]` no tienen esa trampa.

## Denso vs disperso: los huecos (*holes*)

Un arreglo **denso** (*dense*) tiene valor en cada índice de `0` a `length - 1`. Un arreglo **disperso** (*sparse*) tiene **huecos**: índices declarados en `length` pero sin valor. Los huecos se crean así:

```js
const conHueco = [1, , 3];        // literal con hueco (el ', ,' crea un hole)
console.log(conHueco.length);     // 3 (verificado)
console.log(1 in conHueco);       // false ← el índice 1 EXISTE en length pero no como propiedad
console.log(conHueco[1]);         // undefined (mismo resultado que un "undefined" guardado…)
console.log(conHueco.forEach((x) => console.log('forEach:', x))); // imprime 1 y 3 — SALTÓ el hueco
console.log(conHueco.map((x) => x * 2)); // [ 2, <1 empty item>, 6 ] — map NO toca los huecos
console.log([...conHueco]);       // [1, undefined, 3] — spread los convierte en undefined REAL
```

La semántica de los huecos, en una tabla:

| Operación | ¿Toca los huecos? | Ejemplo |
|---|---|---|
| `for...of`, spread, `Array.from` | Sí → `undefined` | `[...[1,,3]]` → `[1, undefined, 3]` |
| `forEach`, `map`, `filter`, `reduce` | **No** — los saltan | `map` devuelve hueco en esa posición |
| `join` | Sí → string vacío | `[1,,3].join('-')` → `'1--3'` |
| `in` / `hasOwnProperty` | distingue hueco de `undefined` | `1 in [1,,3]` → `false`; `1 in [1,undefined,3]` → `true` |
| `JSON.stringify` | Sí → `null` | `JSON.stringify([1,,3])` → `'[1,null,3]'` |

> [!warning] Hueco vs `undefined` guardado — casi nadie los distingue
> `[1, , 3]` y `[1, undefined, 3]` se ven iguales en consola y tienen el mismo `length`, pero no son el mismo arreglo: `map`/`forEach` saltan los huecos y no los undefined. Regla práctica: **no crees huecos a propósito** — inicializá con `undefined` explícito si hace falta. Los métodos modernos que retornan arreglos nuevos (`map`, `filter`, `concat`) generan arreglos **densos**, así que los huecos tienden a desaparecer solos al transformar.

## `length`: una propiedad viva

`length` no es "cuántos hay": es **la posición más alta + 1**, y se puede escribir:

```js
const numeros = [10, 20, 30];
console.log(numeros.length); // 3: cantidad de elementos

// Asignar un índice lejano NO rellena los huecos: crea el índice 7 directamente
const meses = ['Enero', 'Febrero', 'Marzo', 'Mayo', 'Junio'];
meses[7] = 'Último mes';
console.log(meses.length); // 8 posiciones, con huecos en medio
```

```js
// Acortar: asignar length menor TRUNCA (borra del final hacia atrás)
const t = [1, 2, 3, 4];
t.length = 2;
console.log(JSON.stringify(t)); // [1,2]  (verificado: borró 3 y 4 para siempre)

// Alargar: asignar length mayor crea huecos al final
t.length = 4;
console.log(JSON.stringify(t)); // [1,2,null,null] (JSON muestra null para undefined/huecos)
console.log(2 in t);            // false — huecos, no valores
```

El crecimiento dinámico es automático en la práctica: `arr[100] = x` fija `length` a 101 solo. Es la razón por la que las comprobaciones `i < arr.length` en bucles son siempre correctas aunque el arreglo crezca durante la iteración (los elementos nuevos quedan al final y los verá la siguiente pasada).

> [!tip] `const` no congela el contenido
> Un arreglo declarado con `const` **sí se puede modificar** (`meses[0] = ...`, `push`, `pop`). Lo que no se puede es **reasignar la variable** (`meses = []` daría error). Si quieres protegerlo del todo: `Object.freeze(meses)` — congelado, `push` lanza `TypeError` en modo estricto.

## Acceder y modificar por índice

```js
const numeros = [10, 20, 30];

console.log(numeros[0]);  // 10 (primer elemento)
console.log(numeros[2]);  // 30 (tercero)
console.log(numeros[20]); // undefined: el índice no existe (NO lanza error)

console.log(numeros.at(-1)); // 30 — índice negativo desde el final (ES2022)

// Reasignar un valor
const meses2 = ['Enero', 'Febrero', 'Marzo', 'Mayo', 'Junio'];
meses2[0] = 'Nuevo mes';
```

| Acceso | `arr[índice]` | `arr.at(índice)` |
|---|---|---|
| Positivos | ✅ | ✅ |
| Negativos | ❌ `undefined` (¡es propiedad nueva!) | ✅ `-1` = último |
| Índice inexistente | `undefined` | `undefined` |
| Variable dinámica | ✅ | ✅ |

Verificado: `[1,2,3][-1]` → `undefined` — `arr[-1]` no es un acceso al final, ¡es la **propiedad "-1"** del objeto! (`arr[-1] = 'x'` la crea y no toca `length`.) `at(-1)` es el acceso negativo correcto; también vale para strings — ver [[Strings]].

## Métodos que modifican el arreglo (in situ)

```js
const meses3 = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo'];

meses3.push('Junio');      // agrega al final
meses3.unshift('Diciembre'); // agrega al comienzo
meses3.pop();              // elimina el último
meses3.shift();            // elimina el primero
meses3.splice(1, 1);       // desde el índice 1, elimina 1 elemento
meses3.splice(1, 0, 'X'); // desde el índice 1, elimina 0 y agrega 'X'

console.table(meses3);
```

| Método | Acción | ¿Retorna algo útil? |
| ------ | ------ | ------------------- |
| `push(...)` | agrega al final | nueva `length` |
| `unshift(...)` | agrega al inicio | nueva `length` |
| `pop()` | quita el final | el elemento quitado |
| `shift()` | quita el inicio | el elemento quitado |
| `splice(inicio, cant, ...)` | corta/reemplaza en medio | arreglo con lo quitado |
| `sort(cb)` | ordena **in situ** | la referencia al mismo arreglo ⚠️ |
| `reverse()` | invierte **in situ** | la referencia al mismo arreglo ⚠️ |
| `fill(v, d, h)` | rellena rango **in situ** | la referencia |
| `copyWithin(d, f, h)` | copia rango interno **in situ** | la referencia |

> [!warning] `sort` y `reverse` mutan y devuelven el mismo arreglo
> `const ordenado = arr.sort()` ordena `arr` y `ordenado === arr`. No hay copia. Ver [[Array Methods]] para el comparador (por defecto ordena como strings) y las alternativas inmutables `toSorted`/`toReversed`.

El patrón combinado de `splice` cubre insertar, borrar y reemplazar — ver [[Array Methods]] para el catálogo completo con mutadores vs transformaciones.

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
const sinProducto = carrito.filter((p) => p !== producto1); // quitar (sin mutar)
```

Hoy hasta se puede construir un objeto al vuelo sin `function` con `new`:

```js
const producto4 = { nombre: 'Mouse', precio: 20 };
```

## Copiar: spread, `slice` y la trampa de la copia poco profunda

```js
const copia1 = [...arr];   // copia superficial moderna
const copia2 = arr.slice(); // copia superficial legacy (mismo efecto)
const copia3 = structuredClone(arr); // copia PROFUNDA (nativa desde 2022)
```

> [!warning] Copiar es **poco profundo** (*shallow*)
> `[...arr]` copia el arreglo, pero los elementos **objetos/arreglos anidados quedan compartidos** por referencia. Verificado con node:

```js
const original = [1, [2, [3]]];
const copia = [...original];
copia[1][1].push(99);
console.log(original); // [1, [2, [3, 99]]] ← ¡el original también cambió!
console.log(copia[1] === original[1]); // true — mismo objeto anidado
```

Síntoma típico en estado de UI: "copié el array y el componente original se re-dibuja igual". Soluciones según profundidad: copia profunda con `structuredClone(arr)` (no copia funciones), JSON-roundtrip (`JSON.parse(JSON.stringify(x))`, pierde `undefined`/fechas/NaN), o copia manual de los niveles que importan (`arr.map((x) => [...x])`).

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

Usos adicionales verificados:

```js
// Intercambiar valores sin variable temporal (asignación por destructuring)
let x = 1, y = 2;
[x, y] = [y, x]; // x = 2, y = 1

// Extraer funciones de un objeto-arreglo de utilidades
const [map, filter] = [Array.prototype.map, Array.prototype.filter];
```

> [!tip] El default solo dispara con `undefined`
> `const [a = 5] = [null]` deja `a = null`. Igual que en los parámetros de [[Funciones]]: el default es para "no llegó valor", no para "llegó un valor raro".

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

Comparativa de patrones de iteración (profundidad en [[Iteradores]] y [[Array Methods]]):

| Patrón | Recorre | Salir / saltar | Crea arreglo |
|---|---|---|---|
| `for (let i…)` | por índice | `break` / `continue` | no |
| `for...of` | por **valor** (también strings, Map…) | `break` / `continue` | no |
| `forEach` | valor + índice | ❌ no hay `break` (usar `return`) | no |
| `for...in` | **propiedades** — no usar en arreglos | ✅ | no |
| `entries()` + `for...of` | `[índice, valor]` | ✅ | no |
| `map` / `filter` / `reduce` | valor + índice | — | sí (transformación) |

```js
// entries: índice y valor juntos, estilo moderno
for (const [i, producto] of carrito.entries()) {
  console.log(i, producto.nombre);
}
```

> [!warning] `for...in` sobre arreglos
> `for...in` enumera **propiedades string** (incluidas las heredadas y cualquier `arr.mifuncion = …`). No itera por orden de índice ni salta huecos. En arreglos usá `for...of` o `for` clásico — `for...in` es para objetos.

> [!warning] `map` no es para "hacer algo" con cada elemento
> Si no vas a usar el arreglo que retorna, usa `forEach`. `map` transforma; su resultado tiene el mismo largo que el original.

## Rendimiento: notas prácticas

- Los arreglos de JS son **densos por diseño**: los motores los almacenan como buffers lineales cuando son homogéneos, y como *dictionary mode* cuando se les agregan propiedades raras o huecos por todos lados. Mantené arreglos "limpios" (enteros indexados + length) y van rapidísimos.
- `push`/`pop` al final son O(1). `unshift`/`shift` son **O(n)** (mueven todos los índices) — en colas grandes preferir el patrón índice o `deque` manual.
- `includes`/`indexOf`/`find` son lineales: para pertenencia frecuente de primitivos, un `Set` es O(1) (`set.has(x)`).
- Concatenar en bucle con `push(...arr)` puede estallar el stack con arreglos gigantes (`push` con 100k+ argumentos): preferir `arr.push(...chunk)` en trozos, `concat` o `for...of` con `push` simple.
- Copiar un arreglo enorme: `[...arr]` es O(n) con memoria nueva; si el objetivo era mutar in-place, no copies.

## Errores comunes

1. **Confundir `length` con "último índice"**: `length` es último índice + 1; el último es `arr.at(-1)` o `arr[arr.length - 1]`.
2. **`arr[-1]` esperando el último elemento**: crea la propiedad `"-1"` y da `undefined`. Usar `at(-1)`.
3. **`new Array(n)` donde querías `[n]`**: crea `n` huecos. Usar literal o `Array.of`.
4. **Borrar con `delete arr[i]`**: deja un **hueco** (el length no cambia, `forEach` lo salta). Para quitar de verdad: `splice` o `filter`.
5. **Copia superficial sobre arreglos anidados**: `[...arr]` compone referencias internas — ver la trampa verificada arriba. `structuredClone` cuando haga falta profundidad.
6. **Comparar arreglos con `===`/`==`**: compara referencias, `[1] == [1]` es `false`. Comparar contenido: `a.length === b.length && a.every((v, i) => v === b[i])`, JSON para primitivos, o `structuredClone` + sort para casos generales.
7. **`for...in` en arreglos**: enumera strings de propiedad, incluidas funciones agregadas. `for...of` siempre.
8. **Esperar que `map`/`forEach` toquen los huecos**: los saltan — `[1,,3].map(x => x * 2)` tiene un hueco en medio. Inicializá con `undefined` explícito si importa.
9. **Confundir índice 0 con "no encontrado"**: `if (arr.indexOf(x))` descarta el índice 0. Comparar contra `-1`.

## Cheatsheet

| Concepto | Sintaxis | Nota clave |
|---|---|---|
| Literal | `[10, 20, 30]` | forma recomendada |
| Comprobar | `Array.isArray(x)` | `typeof` da `"object"` |
| Longitud | `length` | writable: trunca o alarga con huecos |
| Acceso | `arr[i]`, `arr.at(-i)` | `at` acepta negativos |
| Mutadores | `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin` | tocan el original |
| Copia superficial | `[...arr]`, `arr.slice()` | anidados compartidos por referencia |
| Copia profunda | `structuredClone(arr)` | no copia funciones |
| Huecos | `[1, , 3]` | `map`/`forEach` los saltan; spread los vuelve `undefined` |
| Iterar | `for`, `for...of`, `forEach`, `entries()` | `for...in` solo para objetos |
| Desestructurar | `const [a, b] = arr` | default solo con `undefined` |
| Intercambiar | `[x, y] = [y, x]` | sin variable temporal |
| Array-like | `Array.from({length, 0…})` | arguments, NodeList, DOM |

## Relacionadas

- [[Array Methods]] — el catálogo exhaustivo: buscar, transformar, reducir, combinar
- [[Iteradores]] — todas las formas de recorrer un arreglo
- [[Funciones]] — callbacks y arrow functions, la base de los métodos
- [[Objetos]] — arreglos de objetos, el patrón más común
- [[Strings]] — `split`/`join`: el puente string ↔ arreglo
- [[Números]] — `reduce` para sumas, máximos y estadísticas
- [[ReactJS - Fundamentos]] — inmutabilidad de arreglos en estado
