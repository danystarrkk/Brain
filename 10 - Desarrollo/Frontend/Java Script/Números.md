---
aliases: ["Números en JavaScript", "Objeto Math", "Tipos numéricos"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Operadores]]"
  - "[[Strings]]"
  - "[[Booleanos]]"
  - "[[Sintaxis Moderna]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Number"
---

# Números

JavaScript tiene **un solo tipo numérico** (`number`): tanto enteros como decimales son el mismo tipo de dato. No existen tipos separados `int` y `float` como en otros lenguajes.

## Definir números

```js
// Definir números
const num1 = 20; // un número NO lleva comillas
const num2 = 15;
const num3 = 18.5; // los decimales usan punto
const num4 = -14;  // también podemos crear números negativos

console.log(num1, num2, num3, num4);
console.log(typeof num1); // "number" para enteros y decimales por igual
```

> [!warning] Comillas cambian el tipo
> `"20"` (string) y `20` (number) no son lo mismo: `"20" + 5` da `"205"` (concatena), mientras que `20 + 5` da `25`. Ver [[Operadores]] para las conversiones.

## Operaciones aritméticas básicas

```js
let resultado;

resultado = num1 + num2; // suma -> 35
resultado = num1 - num2; // resta -> 5
resultado = num1 * num2; // multiplicación -> 300
resultado = num1 / num2; // división -> 1.3333...
resultado = num1 % num2; // módulo: el residuo de la división -> 5
resultado = num1 ** 2;   // potencia (ES2016): 20 al cuadrado -> 400
```

El **módulo** (`%`) devuelve el **residuo** de la división. Es muy útil para saber si un número es par o impar: `n % 2 === 0`.

> [!warning] Precisión de punto flotante
> `0.1 + 0.2` no es `0.3`, sino `0.30000000000000004`. Es una limitación del formato binario de coma flotante que comparten casi todos los lenguajes. Para dinero o comparaciones exactas se redondea con `toFixed` o se trabajan **centavos enteros**.

## El objeto `Math`

`Math` es un objeto con utilidades matemáticas **estáticas** (no se crea con `new`, se usa directo):

```js
console.log(Math.PI); // 3.141592653589793

// Redondeo
console.log(Math.floor(2.6));  // 2  redondea hacia abajo
console.log(Math.ceil(2.4));   // 3  redondea hacia arriba
console.log(Math.round(2.5));  // 3  redondea al entero más cercano (.5 hacia arriba)
console.log(Math.trunc(2.9));  // 2  corta los decimales sin redondear

// Raíz, valor absoluto y potencia
console.log(Math.sqrt(144)); // 12 raíz cuadrada
console.log(Math.abs(-500)); // 500 valor absoluto
console.log(Math.pow(4, 2)); // 16 elevar a una potencia (también: 4 ** 2)

// Mínimo y máximo: reciben valores como argumentos (para un arreglo usa spread)
console.log(Math.min(3, 2, 5, -5, 1, 9)); // -5
console.log(Math.max(3, 2, 5, -5, 1, 9)); // 9

const notas = [3, 2, 5, -5, 1, 9];
console.log(Math.min(...notas)); // -5 (spread: reparte los elementos como argumentos)

// Aleatorios
console.log(Math.random());       // decimal entre 0 (incluido) y 1 (excluido)
console.log(Math.random() * 20);  // decimal entre 0 y casi 20
```

> [!warning] `Math.random()` nunca da enteros
> Devuelve siempre un decimal. Y **no** usar `Math.round(Math.random() * 30)` como número aleatorio uniforme: los extremos (0 y 30) salen con la mitad de probabilidad que el resto. La fórmula correcta para un entero entre `min` y `max` **ambos incluidos** es:

```js
const randomEntero = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

console.log(randomEntero(0, 30)); // entero uniforme entre 0 y 30
console.log(randomEntero(5, 10)); // entero uniforme entre 5 y 10
```

## Incrementos, decrementos y asignación compuesta

```js
let puntaje = 5;

puntaje++;        // incrementa en 1 -> 6 (postfijo)
++puntaje;        // también incrementa en 1 -> 7 (prefijo; típico en bucles: for (let i = 0; i < n; i++))
puntaje += 2;     // compuesto: suma 2 -> 9
puntaje -= 2;     // compuesto: resta 2 -> 7
puntaje *= 2;     // compuesto: multiplica por 2 -> 14
puntaje /= 2;     // compuesto: divide entre 2 -> 7

console.log(puntaje);
```

> [!tip] Prefijo vs. posfijo
> `x++` devuelve el valor **antes** de incrementar; `++x` devuelve el valor **ya incrementado**. `console.log(x++)` imprime el valor viejo; `console.log(++x)` el nuevo. Fuera de casos puntuales, úsalos solo como "cambiar en 1".

## De strings a números

Los datos suelen llegar como texto (inputs, `prompt`, una API…). Para convertirlos existen dos rutas:

```js
const texto1 = '20';
const texto2 = '20.2';
const texto3 = 'Uno';
const num = 20;

console.log(typeof texto1); // "string"

// Number.parseInt: extrae un entero desde el inicio del texto
console.log(Number.parseInt(texto1)); // 20
console.log(Number.parseInt('45px')); // 45 (lee los dígitos y se detiene)

// Number.parseFloat: extrae un número con decimales
console.log(Number.parseFloat(texto2)); // 20.2

// Conversión con Number(): devuelve NaN cuando no puede convertir
console.log(Number(texto3)); // NaN (Not a Number)
console.log(Number(texto1)); // 20
```

Cuando la conversión falla, JS devuelve **`NaN`** (Not a Number) en lugar de lanzar un error. `NaN` es tramposo: no se compara bien consigo mismo y `typeof NaN` es `"number"`:

```js
console.log(NaN === NaN);        // false (siempre)
console.log(Number.isNaN(Number.parseInt('Uno'))); // true  forma correcta de detectarlo
console.log(isNaN('hola'));      // true (convierte antes de comparar: cuidado)
console.log(isNaN('20'));        // false: '20' se convierte a 20 y no es NaN
```

## Formato y verificaciones

```js
const precio = 300;

console.log(precio.toFixed(2)); // "300.00" -> ¡devuelve un STRING!
console.log(Number((3.999).toFixed(2))); // 4 -> conviértelo de nuevo si necesitas número

console.log(Number.isInteger(20)); // true
console.log(Number.isInteger(Number.parseFloat('20.2'))); // false: 20.2 tiene decimales
console.log(Number.isInteger(Number.parseInt('20.2')));   // true: parseInt ya devolvió 20
```

> [!tip] Convivencia con strings
> Muchos métodos numéricos viven en el objeto `Number` (`parseInt`, `parseFloat`, `isInteger`) porque el **mismo nombre existe como método de string** con otro comportamiento. Por eso se prefiere siempre la versión `Number.*` — ver [[Strings]].

## Relacionadas

- [[Operadores]] — comparaciones (`===`) y operadores aritméticos/compuestos
- [[Strings]] — conversión entre texto y números
- [[Booleanos]] — el otro tipo primitivo y sus valores truthy/falsy
- [[Sintaxis Moderna]] — `const`/`let` para declarar estas variables
