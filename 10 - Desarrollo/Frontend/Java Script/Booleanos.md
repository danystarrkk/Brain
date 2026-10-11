---
aliases: ["Booleans", "Booleanos en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Estructuras de Control]]"
  - "[[Operadores]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Boolean"
---

# Booleanos

El tipo **booleano** solo tiene dos valores: `true` y `false`. Es la materia prima de las [[Estructuras de Control|estructuras de control]]: toda condición termina reduciéndose a un booleano.

## Definir booleanos

```js
const boolean1 = true;
const boolean2 = false;
const boolean3 = 'true';           // ¡esto es un string, no un booleano!
const boolean4 = new Boolean(true); // un OBJETO envolviendo un booleano

console.log(typeof boolean1); // "boolean"
console.log(typeof boolean3); // "string"
console.log(typeof boolean4); // "object"
```

> [!warning] Nunca uses `new Boolean()`
> Al ser un objeto, es siempre truthy... incluso `new Boolean(false)`. Crea situaciones absurdas como `if (new Boolean(false))` entrando al bloque. Solo se ve en APIs muy viejas.

## Comparaciones: `==` vs `===`

```js
console.log(boolean1 == boolean3);  // false: compara true con "true"
console.log(boolean1 === boolean3); // false: también, y además compara tipos
console.log(boolean1 == 'true');    // false
console.log(boolean1 == boolean2);  // false
console.log(boolean1 == true);      // true
console.log(boolean1 === true);     // true
```

El operador `==` intenta **convertir** los tipos antes de comparar (coerción), lo que produce resultados sorprendentes; `===` exige **mismo valor y mismo tipo**. La regla de oro del JavaScript moderno: **usa `===` y `!==` por defecto**, y reserva `==` para comparaciones donde la coerción es explícitamente deseada (por ejemplo `valor == null` para chequear `null` **o** `undefined` a la vez).

## Truthy y falsy

Un booleano no es lo único que puede ir en un `if`: cualquier valor se convierte a booleano. Solo unos pocos valores son **falsy** (falsos); todo lo demás es **truthy**:

| Falsy | Truthy (ejemplos) |
| ----- | ----------------- |
| `false` | `true` |
| `0`, `-0`, `NaN` | cualquier número distinto de cero: `1`, `-1`, `42` |
| `''` (string vacío) | `'hola'`, `'0'`, `'false'` |
| `null` | todos los objetos: `{}`, `[]` |
| `undefined` | funciones, `document.all` |
| `document.all` (legacy) | |

> [!warning] Los strings `"false"` y `"0"` son truthy
> `'false'` y `'0'` son strings **no vacíos**, así que pasan el `if`. Esto causa errores típicos al leer formularios: valida con conversión explícita (`Number(valor)`, `valor === 'true'`) antes de confiar en el booleano.

## Uso en condiciones

```js
const autenticado = true;

// if/else: la conversión a booleano es implícita
if (autenticado) {
  console.log('Sí puedes ver Netflix');
} else {
  console.log('No puedes verlo');
}

// Ternario: la forma compacta de if/else en una expresión
console.log(autenticado ? 'Sí está autenticado' : 'No está autenticado');
```

Con valores no booleanos se lee igual, gracias a truthy/falsy:

```js
const usuario = 'daniel';

if (usuario) {
  console.log('Hay usuario logueado'); // el string no vacío es truthy
}

const productos = [];
if (!productos.length) {
  console.log('El carrito está vacío'); // más explícito que !productos
}
```

## Operadores de comparación booleana

```js
const activo = true;
const premium = false;

console.log(activo && premium); // AND: ambos deben ser true  -> false
console.log(activo || premium); // OR: basta con uno          -> true
console.log(!activo);           // NOT: invierte el valor     -> false
```

Ver [[Estructuras de Control]] para cómo estos operadores combinan condiciones dentro de `if`, y [[Operadores]] para el resto de operadores de JavaScript.

## Relacionadas

- [[Estructuras de Control]] — `if`, `switch`, ternario y operadores lógicos
- [[Operadores]] — operadores de comparación y lógicos en detalle
- [[Funciones]] — funciones que retornan booleanos (predicados) para `filter`/`some`
