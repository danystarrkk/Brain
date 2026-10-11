---
aliases: ["Bucles", "Loops", "Ciclos en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Arreglos]]"
  - "[[Funciones]]"
  - "[[Array Methods]]"
  - "[[Objetos]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Statements/for"
---

# Iteradores

Un **iterador** (o bucle) repite un bloque de código varias veces. JavaScript ofrece varios estilos: el `for` clásico para control total, `while` cuando la condición manda, y `for...of`/`forEach` para recorrer [[Arreglos|arreglos]] con menos ruido.

## El `for` clásico

Tiene tres partes separadas por punto y coma: **inicio**, **condición de continuación** e **incremento**:

```js
// imprime del 0 al 10
for (let i = 0; i <= 10; i++) {
  console.log(i);
}
```

- `let i = 0` — se ejecuta una vez al comenzar.
- `i <= 10` — mientras sea cierto, el bucle sigue.
- `i++` — se ejecuta tras cada vuelta.

> [!tip] `let` dentro del `for`
> Declarar el contador con `let` lo encierra en el scope del bucle: no se filtra al resto del código y cada vuelta tiene "su propio" `i` (clave si capturas el valor en closures).

### Pares e impares

```js
for (let i = 0; i <= 10; i++) {
  if (i === 0) {
    console.log(`El ${i} no es par ni impar`);
  } else if (i % 2 === 0) {
    console.log(`El ${i} es un número par`);
  } else {
    console.log(`El ${i} es un número impar`);
  }
}
```

El operador `%` (módulo) da el **residuo** de la división: si `i % 2 === 0`, el número es divisible entre 2.

### Recorrer un arreglo por índice

```js
const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audífonos', precio: 50 },
  { nombre: 'Teclado', precio: 100, descuento: true },
];

for (let i = 0; i < carrito.length; i++) {
  console.log(`El ${carrito[i].nombre} tiene un precio de ${carrito[i].precio}`);
}
```

## `break` y `continue`

```js
// break: termina el bucle COMPLETAMENTE
for (let i = 0; i < 10; i++) {
  console.log(`Es el número: ${i}`);
  if (i === 5) break; // imprime del 0 al 5 y se detiene
}

// continue: salta a la siguiente vuelta sin ejecutar el resto
for (let i = 0; i < 10; i++) {
  if (i === 5) continue; // no imprime el 5
  console.log(`Es el número: ${i}`);
}
```

Ejemplo práctico con el carrito: los productos con descuento toman un camino distinto.

```js
for (let i = 0; i < carrito.length; i++) {
  if (carrito[i].descuento) {
    console.log(`El ${carrito[i].nombre} está en descuento`);
    continue;
  }
  console.log(carrito[i].nombre);
}
```

### Reto: FizzBuzz

Clásico de entrevistas: múltiplos de 3 imprimen *Fizz*, de 5 *Buzz*, y de ambos *FizzBuzz*:

```js
for (let i = 1; i <= 100; i++) {
  if (i % 3 === 0 && i % 5 === 0) {
    console.log(`${i}: FizzBuzz`);
  } else if (i % 3 === 0) {
    console.log(`${i}: Fizz`);
  } else if (i % 5 === 0) {
    console.log(`${i}: Buzz`);
  } else {
    console.log(i);
  }
}
```

> [!warning] El orden de las condiciones importa
> Si chequearas `i % 3 === 0` antes del caso doble, los múltiplos de 15 caerían en "Fizz" y el "FizzBuzz" nunca se imprimiría. El caso más específico va **primero**.

## `while` y `do...while`

- **`while`**: evalúa la condición **antes** de cada vuelta; si es falsa desde el inicio, no ejecuta nada.
- **`do...while`**: ejecuta al menos **una vez** y evalúa la condición al final.

```js
let i = 0;
while (i < 10) {
  console.log(i);
  i++; // sin esto, bucle infinito
}

let j = 10;
do {
  console.log(j);
  j++;
} while (j < 10); // la condición es falsa, pero la vuelta ya se ejecutó: imprime 10
```

Úsalos cuando **no sabes de antemano** cuántas vueltas faltan (por ejemplo, leer hasta que un `prompt` devuelva "salir").

## Recorriendo arreglos: `forEach`

`forEach` recibe una [[Funciones|función]] que se ejecuta por cada elemento. Con un segundo parámetro obtienes el índice:

```js
const pendientes = ['Tareas', 'Ejercicio', 'Programación', 'Trabajo'];

pendientes.forEach((pendiente, indice) => {
  console.log(`${pendiente} con índice ${indice}`);
});
```

## `for...of`: la forma moderna de recorrer

`for...of` itera sobre los **valores** de un iterable (arreglos, strings, Sets, Maps) sin indices ni callbacks:

```js
for (const pendiente of pendientes) {
  console.log(pendiente);
}

for (const producto of carrito) {
  console.log(producto.nombre, producto.precio);
}
```

## `for...in`: índices y claves

`for...in` itera sobre las **propiedades enumerables**: en arreglos da los índices (como strings), en objetos da las **claves**:

```js
// Con arreglo: imprime 0, 1, 2, 3 (los índices, no los valores)
for (const indice in pendientes) {
  console.log(indice);
}

// Con objeto: imprime las claves
const automovil = { modelo: 'Toyota', motor: 'Chevrolet', año: 2020 };

for (const propiedad in automovil) {
  console.log(propiedad);          // "modelo", "motor", "año"
  console.log(automovil[propiedad]); // los valores
}
```

> [!warning] `for...in` no para arreglos
> En arreglos recorre índices (y puede recoger propiedades agregadas al arreglo). Para **valores** de un arreglo usa siempre `for...of` o `forEach`.

### Objeto con `for...of`: `Object.entries`

Para recorrer un objeto con `for...of`, conviértelo a pares clave/valor:

```js
for (const [clave, valor] of Object.entries(automovil)) {
  console.log(`${clave}: ${valor}`); // desestructuración del par
}
```

## ¿Cuál usar? Resumen

| Situación | Recomendado |
| --------- | ----------- |
| Recorrer arreglo solo con valores | `for...of` |
| Recorrer arreglo con índice | `forEach((x, i) => ...)` o `for clásico` |
| Necesitas `break`/`continue` | `for clásico` o `for...of` |
| Recorrer objeto | `for...in` o `Object.entries` + `for...of` |
| Contar mientras haya condición | `while` / `do...while` |
| Transformar/filtrar arreglos | `map`, `filter` — ver [[Array Methods]] |

## Relacionadas

- [[Arreglos]] — la estructura que más se itera
- [[Array Methods]] — alternativas declarativas: `map`, `filter`, `reduce`
- [[Funciones]] — callbacks de `forEach` y arrow functions
- [[Objetos]] — `for...in` y `Object.entries`
- [[Estructuras de Control]] — `if` dentro de los bucles
