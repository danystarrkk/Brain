---
tags:
  - desarrollo
  - frontend
  - booleanos
tipo: concepto
estado: terminado
---

# Booleanos

El tipo **booleano** solo tiene dos valores: `true` y `false`. Es la materia prima de las [[Estructuras de Control|estructuras de control]]: toda condición termina reduciéndose a un booleano mediante el algoritmo de **coerción a booleano** (*ToBoolean*). Entender ese algoritmo — y la coerción de comparación de `==` — es entender el 90% de los bugs de lógica en JavaScript.

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

No hay literales alternativos: `TRUE`/`True`/`1` no son booleanos (a diferencia de Python o SQL). Los booleanos son primitivos: se comparan por **valor**, no por referencia — `boolean1 === boolean1` siempre es `true`.

## El algoritmo de coerción a booleano (*ToBoolean*)

Cuando JS necesita un booleano — en `if`, `while`, `&&`, `||`, `!`, `??`-contextos, `Boolean()` — aplica una tabla **fija y completa**. No hay ambigüedad: cada valor cae en una de estas filas:

| Entrada | Resultado ToBoolean | Categoría |
|---|---|---|
| `false` | `false` | falsy explícito |
| `0`, `-0`, `0n` (BigInt cero) | `false` | número cero |
| `NaN` | `false` | "no número" |
| `''` (string vacío, solo length 0) | `false` | cadena vacía |
| `null` | `false` | nulo |
| `undefined` | `false` | sin valor |
| `document.all` | `false` ⚠️ (quirk, ver abajo) | excepción anómala |
| **todo lo demás** | `true` | truthy |

"Todo lo demás" incluye: cualquier número distinto de cero (incluidos `-1`, `42`, `Infinity`), **cualquier string no vacío** (incluidos `'false'`, `'0'`, `' '` — un espacio cuenta), todos los objetos (incluidos `[]`, `{}`, `[] == false`…), funciones, `new Boolean(false)`, `Symbol()`, `10n`.

> [!warning] `document.all`: el truthy que finge ser falsy
> `document.all` es un objeto del DOM legacy (solo existe en navegadores, en Internet Explorer/Edge viejo). Es truthy según el operador `if`, pero **`document.all == false` es `true`**: el spec lo marca como *falsy-throwing exotic object* para que `if (document.all)` siga funcionando en código pre-2000 escrito para Netscape/IE. Ningún otro objeto se comporta así. En la práctica: nunca lo uses; es el último vestigio de Internet Explorer.

> [!warning] `new Boolean(false)` — la trampa del objeto truthy
> Al ser un objeto, es siempre truthy... incluso `new Boolean(false)`. Crea situaciones absurdas como `if (new Boolean(false))` entrando al bloque. Lo mismo aplica a `Boolean(0)` vs `new Boolean(0)`: el primero es primitivo falsy, el segundo objeto truthy. Solo se ve en APIs muy viejas — **nunca uses `new Boolean()`**.

## Truthy y falsy

Un booleano no es lo único que puede ir en un `if`: cualquier valor se convierte a booleano con el algoritmo anterior. La tabla falsy **es exhaustiva** — ocho valores y punto:

| Falsy (lista completa) | Truthy (ejemplos) |
| ----- | ----------------- |
| `false` | `true` |
| `0`, `-0`, `0n` | cualquier número distinto de cero: `1`, `-1`, `42`, `Infinity` |
| `NaN` | `Infinity`, `Number.MIN_VALUE` (¡un decimal diminuto pero distinto de cero!) |
| `''` (string vacío) | `'hola'`, `'0'`, `'false'`, `' '` (un espacio) |
| `null` | todos los objetos: `{}`, `[]`, `[1,2]` |
| `undefined` | funciones, `Symbol()`, `10n` |
| `document.all` (legacy DOM) | |

> [!warning] Los strings `"false"` y `"0"` son truthy
> `'false'` y `'0'` son strings **no vacíos**, así que pasan el `if`. Esto causa errores típicos al leer formularios: valida con conversión explícita (`Number(valor)`, `valor === 'true'`) antes de confiar en el booleano.

```js
console.log(!!'false');  // true  (string no vacío)
console.log(!!'0');      // true
console.log(!![]);       // true  (objeto, aunque vacío)
console.log(!!{});       // truthy igual
console.log(!!' ');      // true  (un espacio NO es string vacío)
console.log(!!0n);       // false (BigInt cero)
```

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

> [!tip] Preferí comprobar la **propiedad**, no el truthiness
> `if (productos.length)`, `if (usuario !== null)`, `if (fecha)` son más precisos que `if (productos)`. El truthiness es un atajo cómodo, pero cuando el valor puede ser `0`, `''` o `null` según el caso, la comprobación explícita documenta **qué** esperás. `!x` es correcto cuando el `0`/vacío también significa "falsy a propósito" (arrays vacíos, strings vacíos).

> [!warning] `switch` y truthiness
> `switch (valor)` usa **`===`**, no coerción: `switch ('5')` no entra en `case 5`. Ver [[Estructuras de Control]].

## `==` vs `===`: la decisión

```js
console.log(boolean1 == boolean3);  // false: compara true con "true"
console.log(boolean1 === boolean3); // false: también, y además compara tipos
console.log(boolean1 == 'true');    // false
console.log(boolean1 == boolean2);  // false
console.log(boolean1 == true);      // true
console.log(boolean1 === true);     // true
```

Con booleanos, `==` tiene una particularidad poco conocida: **convierte el booleano a número** antes de comparar. Por eso sobreviven trampas como estas (todas verificadas):

```js
console.log(1 == '1');        // true  (string → número)
console.log(true == 1);       // true  (booleano → número)
console.log(true == 'true');  // false ('true' → NaN)
console.log([] == false);     // true  ([] → '' → 0; false → 0)
console.log(null == undefined); // true (la ÚNICA pareja que == iguala)
console.log(NaN === NaN);     // false (NaN no es igual ni a sí mismo; usar Object.is o Number.isNaN)
```

La regla de oro del JavaScript moderno: **usa `===` y `!==` por defecto**. La única coerción de `==` que se justifica:

| Comparación | Equivalente estricto | Por qué |
|---|---|---|
| `valor == null` | `valor === null \|\| valor === undefined` | chequea null **y** undefined a la vez — el único uso recomendado |
| Todo lo demás | usar `===` | `==` produce resultados impredecibles sin ahorro real |

> [!info] Por qué `[] == false` es true
> Ambos pasan por ToPrimitive → ToNumber: `[]` (objeto vacío) se convierte en `''` y luego en `0`; `false` se convierte en `0`. `0 == 0` → `true`. Cadenas de coerción de hasta 4 pasos como esta son exactamente lo que `===` evita.

## `Boolean()` vs `!!`

Dos formas de convertir **explícitamente** a booleano:

```js
console.log(Boolean('hola'));   // true
console.log(Boolean(''));       // false
console.log(Boolean(0));        // false
console.log(!!'hola');          // true — doble NOT: ! true → false → true
console.log(!!'');              // false
```

| Forma | Mecanismo | Legibilidad |
|---|---|---|
| `Boolean(x)` | Llamada al constructor como función (no `new`) = ToBoolean directo | Explícita: se lee como conversión |
| `!!x` | `!` invierte a booleano; el segundo `!` lo vuelve al valor original | Compacta; leída como "cast a booleano" |

Ambas hacen **exactamente lo mismo** (ToBoolean, sin coerciones extra). En métodos que piden booleano puro son útiles:

```js
const existe = Boolean(coleccion.find((x) => x.id === 5));
// Nota: find ya retorna undefined → !! es suficiente aquí
```

> [!warning] `!!` no "arregla" tipos
> `!!'0'` es `true`. El doble NOT aplica ToBoolean, que no convierte strings numéricos. Si venís de un input y necesitás el booleano real del texto, convertí primero: `valor === 'true'` o `Number(valor) === 1`.

## Operadores lógicos como control de flujo

Los operadores de [[Operadores|lógicos]] **retornan operandos, no booleanos** — y eso los convierte en herramientas de flujo, no solo de comparación:

```js
const activo = true;
const premium = false;

console.log(activo && premium); // AND: ambos deben ser true  -> false
console.log(activo || premium); // OR: basta con uno          -> true
console.log(!activo);           // NOT: invierte el valor     -> false
```

### `&&` — cortocircuito: "si el primero es falsy, devuélvelo; si no, devuélve el segundo"

```js
// Guard clause: ejecuta solo si el objeto existe
usuario && console.log(`Hola ${usuario.nombre}`);

// Encadenar defaults encadenados (objetos anidados)
const ciudad = usuario && usuario.direccion && usuario.direccion.ciudad;
```

### `||` — cortocircuito: "si el primero es truthy, devuélvelo; si no, el segundo"

```js
// Patrón de default (¡con la trampa del 0 y el vacío!)
const nombre = ingreso || 'Invitado'; // 'Invitado' también si ingreso es '' o 0
```

> [!warning] La trampa de `||` con 0 y strings vacíos
> `0 || 100` da `100`, y `'' || 'defecto'` da `'defecto'` — correcto si el 0/vacío es "sin valor", **bug** si es un valor legítimo (un contador en 0, un input vacío). Para defaults donde `0` y `''` son válidos, usá **`??`** (nullish coalescing): `0 ?? 100` → `0`, porque `??` solo cae al segundo con `null`/`undefined`. Ver [[Operadores]] y [[Sintaxis Moderna]].

### De operador lógico a booleano

Cuando sí necesitás un booleano de verdad (guardar en estado, devolver de un predicado), el `!` simple basta: `const haySesion = !!sesion;` o mejor `sesion !== undefined`.

## Predicados: funciones que retornan booleanos

Una **función predicado** retorna `true`/`false` y es el pegamento de todo el estilo declarativo de [[Array Methods]]:

```js
const esMayorDeEdad = (edad) => edad >= 18;

[15, 22, 9].filter(esMayorDeEdad);     // [22]
[15, 22, 9].some(esMayorDeEdad);       // true
[22, 30].every(esMayorDeEdad);         // true
```

Los callbacks de `filter`/`some`/`every` aplican ToBoolean **al retorno** del callback: `filter(arr => arr.precio)` funciona porque el precio es truthy, pero con precio `0` filtraría el elemento — devolvé siempre booleanos explícitos en predicados (`> 0`, `!== null`).

## Errores comunes

1. **Confundir `'true'`/`'false'` (strings) con booleanos**: llegan así de inputs y APIs; comparar con `===` contra los literales o convertir con `Number`/`JSON.parse`.
2. **`new Boolean(false)` en un `if`**: siempre truthy (es objeto). No usar el constructor.
3. **Defaults con `||`** cuando `0` o `''` son valores válidos: usar `??`.
4. **`==` contra `true`/`false`**: `1 == true` pasa, pero `2 == true` falla (2 → 2, true → 1). Compará contra booleanos con `===` o usá el truthiness directo.
5. **`if (indice)` para índices de arreglos**: el índice **0 es falsy** — `findIndex` retorna `0` y `if (indice)` lo descarta como "no encontrado". Comparar contra `-1`: `if (indice !== -1)`.
6. **NaN como "falsy raro"**: `NaN` es falsy, pero `if (valor === valor)` es la única comprobación fiable de "no NaN" (o `Number.isNaN` / `Object.is`).
7. **Usar `switch` esperando coerción**: `switch` usa `===`; no convierte.

## Cheatsheet

| Concepto | Sintaxis / valor | Nota |
|---|---|---|
| Literales | `true`, `false` | Únicos valores del tipo `boolean` |
| Falsy (completo) | `false`, `0`, `-0`, `0n`, `NaN`, `''`, `null`, `undefined`, `document.all` | Todo lo demás es truthy |
| ToBoolean | algoritmo fijo | Sin conversión de strings numéricos |
| `Boolean(x)` | conversión explícita | = `!!x`, más legible |
| `!!x` | doble NOT | Cast a booleano compacto |
| `x == null` | null o undefined | Único `==` recomendado |
| `===` | valor + tipo | Por defecto en todo |
| `&&` | devuelve op. falsy o el segundo | guard clauses, encadenados |
| `\|\|` | devuelve op. truthy o el segundo | defaults (trampa con 0/`''`) |
| `??` | solo null/undefined cae | defaults seguros con 0/`''` |
| `!x` | invierte con coerción | predicados, negaciones |
| `new Boolean(x)` | objeto truthy siempre | prohibido en código moderno |

## Relacionadas

- [[Estructuras de Control]] — `if`, `switch`, ternario y operadores lógicos
- [[Operadores]] — operadores de comparación, lógicos, `??` y coerción en detalle
- [[Funciones]] — funciones que retornan booleanos (predicados) para `filter`/`some`
- [[Números]] — `NaN`, el cero y la detección de números válidos
- [[Strings]] — por qué `'0'` es truthy y el string vacío no
- [[Objetos]] — todos los objetos son truthy (incluidos los vacíos)
