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
  - "[[Funciones]]"
  - "[[Array Methods]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Number"
---

# Números

JavaScript tiene **un solo tipo numérico** (`number`): tanto enteros como decimales son el mismo tipo de dato. No existen tipos separados `int` y `float` como en otros lenguajes. Ese único tipo es, técnicamente, un **flotante de doble precisión IEEE-754** — por eso comparte con prácticamente todos los lenguajes la misma limitación de precisión decimal, y por eso existen reglas especiales para enteros grandes (`MAX_SAFE_INTEGER`, `BigInt`).

> [!info] El modelo mental correcto
> Todo número JS es una aproximación binaria de un valor real. Los enteros hasta 2^53 son exactos (se representan sin pérdida); los decimales como `0.1` **no tienen representación binaria exacta** y se redondean al vecino representable más cercano. De ahí salen el `0.1 + 0.2 ≠ 0.3` y todas las reglas de esta nota.

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

### Literales: todas las formas válidas

```js
const decimal = 1000;        // decimal
const negativo = -42;
const exponente = 5e3;       // 5000  (5 × 10^3) — también 5E3
const minuscula = 1.5e-4;    // 0.00015
const hex = 0xff;            // 255   (base 16)
const octal = 0o17;          // 15    (base 8, sintaxis moderna)
const binario = 0b1101;      // 13    (base 2)
const separadores = 2_000_000; // 2000000 — solo legibilidad (ES2021)
```

Los literales hex/oct/bin/binario son **números normales** una vez evaluados: `typeof 0xff === 'number'`. Los guiones bajos solo agrupan dígitos: no van al inicio (`_1` es error), ni dos seguidos (`1__0` es error), ni después del punto.

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

El **módulo** (`%`) devuelve el **residuo** de la división, con el signo del **dividendo**: `10 % 3` → `1`, pero `-10 % 3` → `-1`. Es muy útil para saber si un número es par o impar: `n % 2 === 0`. La potencia admite fracciones: `4 ** 0.5` → `2` (raíz cuadrada).

> [!warning] Precisión de punto flotante
> `0.1 + 0.2` no es `0.3`, sino `0.30000000000000004` (verificado). `(0.1 + 0.2) === 0.3` → `false`. Es una limitación del formato binario de coma flotante que comparten casi todos los lenguajes. Para dinero o comparaciones exactas se redondea con `toFixed` o se trabajan **centavos enteros**.

### Por qué ocurre (la explicación, no el dogma)

En binario, `0.1` es un decimal periódico — como `1/3` en decimal (`0.333…`), que tampoco se puede escribir exacto con 3 cifras. IEEE-754 guarda la fracción redondeada al valor representable más cercano; al sumar dos aproximaciones, el error se acumula y el resultado cae **un ulp** (`unit in the last place`) por encima de `0.3`:

```js
console.log(0.1 + 0.2);                 // 0.30000000000000004
console.log((0.1 + 0.2) === 0.3);       // false
console.log((0.1 + 0.2).toFixed(20));   // 0.30000000000000004441 ← el decimal exacto guardado
```

Las operaciones "seguras" son las que redondean al representable más cercano esperado: `0.1 * 3` es `0.30000000000000004`, pero `(0.1 * 3).toFixed(2)` → `"0.30"`, y `1.005` en binario es *menor* que 1.005 decimal, por eso `(1.005).toFixed(2)` → `"1.00"` (trampa clásica del banker-style rounding).

## Rangos y límites del tipo

| Constante | Valor | Significado |
|---|---|---|
| `Number.MAX_SAFE_INTEGER` | `9007199254740991` (2^53 − 1) | último entero representable sin ambigüedad |
| `Number.MIN_SAFE_INTEGER` | `-9007199254740991` | límite negativo espejo |
| `Number.MAX_VALUE` | ≈ `1.797…×10^308` | mayor flotante representable |
| `Number.MIN_VALUE` | ≈ `5×10^−324` | menor **positivo** distinto de cero (no es el menor número) |
| `Number.EPSILON` | `2.220446049250313×10^16` | diferencia entre 1 y el siguiente flotante; base de comparaciones con tolerancia |

Los enteros pierden precisión más allá de 2^53 (verificado):

```js
console.log(Number.MAX_SAFE_INTEGER);                    // 9007199254740991
console.log(9007199254740992 === 9007199254740993);      // true ← ¡dos números distintos "iguales"!
console.log(Number.MAX_SAFE_INTEGER + 2 + 1);            // 9007199254740992 (redondeado)
```

Para IDs grandes, dinero en centavos grandes o cripto: **`BigInt`** (tipos de [[Sintaxis Moderna]]):

```js
console.log(typeof 10n);           // "bigint"
console.log(10n + 20n);            // 30n
console.log(Number(10n));          // 10 — conversión explícita permitida
try { 10n + 1; } catch (e) { console.log(e.constructor.name); } // TypeError: no se mezclan
```

> [!warning] No mezcles BigInt y Number
> `10n + 1` lanza `TypeError` (verificado): hay que convertir explícitamente (`Number(10n) + 1` o `10n + 1n`). Tampoco hay operaciones con decimales en BigInt: no existe `1.5n`.

## Divisiones imposibles: `Infinity` y `NaN`

JavaScript no lanza error por división entre cero — devuelve un valor especial (verificado):

```js
console.log(1 / 0);    // Infinity
console.log(-1 / 0);   // -Infinity
console.log(0 / 0);    // NaN (indeterminado)
console.log(JSON.stringify(Infinity)); // "null" ← cuidado al serializar con [[Formato Json]]
```

`Infinity` compara y ordena como se espera (`> 1e308`, `Math.max` lo respeta), pero no es un entero finito: `Number.isFinite(Infinity)` → `false`, y al entrar en aritmética finita contamina todo (`Infinity - Infinity` → `NaN`).

## `NaN`: el no-número

`NaN` es el resultado de operaciones imposibles (`Number('abc')`, `0/0`, `Math.sqrt(-1)`). Tiene dos propiedades contra-intuitivas (verificadas):

```js
console.log(NaN === NaN);        // false (siempre: NaN no es igual ni a sí mismo)
console.log(typeof NaN);         // "number" ← el nombre engaña
console.log(Object.is(NaN, NaN)); // true — la única comparación que NaN supera
```

## Detección: `Number.isNaN` vs `isNaN` (global)

Este par causa más bugs que cualquier otro. La versión **global** convierte el argumento a número **antes** de preguntar si es NaN — la versión de `Number` no:

```js
console.log(Number.isNaN(Number.parseInt('Uno'))); // true  forma correcta de detectarlo
console.log(isNaN('hola'));      // true (convierte antes comparar: 'hola' → NaN → true)
console.log(isNaN('20'));        // false: '20' se convierte a 20 y no es NaN
console.log(Number.isNaN('hola')); // false ← ¡diferente! 'hola' no ES NaN, es un string
console.log(isNaN(''));          // false: '' → 0 (verificado)
```

| Función | ¿Convierte el argumento? | Uso correcto |
|---|---|---|
| `isNaN(x)` (global) | **Sí** — primero `Number(x)` | Detectar "no convertible a número" con strings sucios |
| `Number.isNaN(x)` | No | Confirmar que un valor **ya numérico** es `NaN` |
| `Object.is(x, NaN)` | No | Igualdad exacta con NaN (funciona con cualquier tipo) |

> [!tip] La detección robusta en la práctica
> Para validar datos crudos (inputs, JSON): `const n = Number(valor); if (Number.isNaN(n)) …` — convierte una sola vez y pregunta con la versión exacta. `Number.isFinite(n)` es mejor aún si también descartás `Infinity`.

## Verificaciones numéricas

```js
const precio = 300;

console.log(Number.isInteger(20)); // true
console.log(Number.isInteger(20.5)); // false
console.log(Number.isInteger(Number.parseFloat('20.2'))); // false: 20.2 tiene decimales
console.log(Number.isInteger(Number.parseInt('20.2')));   // true: parseInt ya devolvió 20

console.log(Number.isFinite(300));            // true
console.log(Number.isFinite(Infinity));       // false
console.log(Number.isFinite('300'));          // false (no convierte — '300' no es número finito)
console.log(isFinite('300'));                 // true  (la global SÍ convierte)
```

Como con `isNaN`, las versiones `Number.*` **no convierten**: `Number.isInteger('20')` → `false`. Regla: convertí primero con `Number()`, verificá después.

## De strings a números

Los datos suelen llegar como texto (inputs, `prompt`, una API…). Para convertirlos existen dos rutas — más la ruta "total" de `Number()`:

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

Tabla completa de comportamiento (todo verificado):

| Entrada | `Number(x)` | `parseInt(x)` | `parseFloat(x)` |
|---|---|---|---|
| `'20'` | `20` | `20` | `20` |
| `'20.2'` | `20.2` | `20` (corta en el punto) | `20.2` |
| `'45px'` | `NaN` (debe ser entero completo) | `45` (lee y para) | `45` |
| `'3.14.15'` | `NaN` | `3` | `3.14` (para en el segundo punto) |
| `''` / `' '` | `0` ⚠️ | `NaN` | `NaN` |
| `'0b11'` | `3` (prefijos 0x/0o/0b) | `0` (no los entiende) | `0` |
| `true` | `1` | `NaN` | `NaN` |
| `null` | `0` | `NaN` | `NaN` |
| `undefined` | `NaN` | `NaN` | `NaN` |

El **radix** (base) de `parseInt`: conviene **pasarlo siempre** — antes de ES5, `'08'` se interpretaba como octal y devolvía `8`… o algo distinto según el motor. Hoy `parseInt('08')` → `8` en todos lados, pero `parseInt('08', 10)` documenta la intención y elimina la ambigüedad con prefijos:

```js
console.log(Number.parseInt('08'));      // 8
console.log(Number.parseInt('08', 10));  // 8 — siempre especificá la base
console.log(Number.parseInt('0b11'));    // 0 (el 0 no es dígito válido en base 10)
```

> [!tip] Convivencia con strings
> Muchos métodos numéricos viven en el objeto `Number` (`parseInt`, `parseFloat`, `isInteger`) porque el **mismo nombre existe como método de string** con otro comportamiento (`'20'.parseInt` no existe, pero `'20'.match`… la idea es la familia `Number.*` vs `String.*`). Por eso se prefiere siempre la versión `Number.*` — ver [[Strings]]. `Number.parseInt === parseInt` → `true`: son la misma función, solo que `Number.*` es la referencia moderna.

## Formato: `toFixed`, `toPrecision`, `toLocaleString`

```js
const precio = 300;

console.log(precio.toFixed(2)); // "300.00" -> ¡devuelve un STRING!
console.log(Number((3.999).toFixed(2))); // 4 -> conviértelo de nuevo si necesitas número
console.log((3.999).toFixed(2)); // "4.00"
console.log((2.675).toFixed(2)); // "2.67" ← el redondeo binario manda (2.675 real es 2.67499…)
```

| Método | Recibe | Devuelve | Cuándo |
|---|---|---|---|
| `toFixed(n)` | decimales (0–100) | **string** con redondeo | mostrar dinero/decimales fijos |
| `toPrecision(n)` | cifras significativas totales | **string** (o número si n es grande) | notación científica controlada |
| `toLocaleString([locales, opts])` | cultura y opciones | **string** localizado | UI multi-idioma: moneda, miles |

```js
console.log((1234.5678).toPrecision(3));   // "1.23e+3"
console.log((1234.5678).toLocaleString('es-ES'));             // "1234,568"
console.log((1234.5678).toLocaleString('en-US', { style: 'currency', currency: 'USD' })); // "$1,234.57"
```

> [!warning] `toFixed` devuelve string y redondea "a lo binario"
> `parseFloat(precio.toFixed(2))` o `Number(...)` si necesitás número. Y para **cálculos** de dinero no uses `toFixed` como aritmética: trabajá en centavos enteros (receta abajo).

### Receta: dinero sin errores de flotante

```js
// Mal: 0.1 + 0.2 = 0.30000000000000004
// Bien: trabajar en centavos enteros y formatear al mostrar
const sumaCentavos = (a, b) => a + b;                 // enteros: exactos
const aDolares = (centavos) => (centavos / 100).toFixed(2);

sumaCentavos(10, 20);        // 30 centavos
aDolares(30);                // "0.30"
```

Para montos grandes o contabilidad seria, la industria usa **decimales exactos** (`decimal.js`, o centavos/millones enteros hasta 2^53).

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

### Catálogo completo de `Math`

| Método | Qué hace | Ejemplo verificado | Detalle |
|---|---|---|---|
| `Math.round(x)` | al entero más cercano, `.5` hacia **arriba** | `Math.round(-2.5)` → `-2` ⚠️ | con negativos `-2.5` queda entre -3 y -2, y el `.5` sube |
| `Math.floor(x)` | hacia abajo (−∞) | `Math.floor(-2.9)` → `-3` | distinto de trunc con negativos |
| `Math.ceil(x)` | hacia arriba (+∞) | `Math.ceil(2.1)` → `3` | |
| `Math.trunc(x)` | corta decimales, ignora signo | `Math.trunc(-2.9)` → `-2` | el más "neutral" |
| `Math.sign(x)` | `-1`, `0`, `1` o `NaN` | `Math.sign(-5)` → `-1` | `Math.sign(0)` → `0` (y de `-0` → `-0`) |
| `Math.abs(x)` | valor absoluto | `Math.abs(-500)` → `500` | |
| `Math.sqrt(x)` | raíz cuadrada | `Math.sqrt(144)` → `12` | negativos → `NaN` |
| `Math.cbrt(x)` | raíz cúbica | `Math.cbrt(27)` → `3` | acepta negativos |
| `Math.hypot(...)` | raíz de la suma de cuadrados | `Math.hypot(3, 4)` → `5` | hipotenusa; evita desbordes intermedios |
| `Math.pow(b, e)` / `b ** e` | potencia | `2 ** 10` → `1024` | `**` es el operador preferido |
| `Math.min(...)` / `Math.max(...)` | menor/mayor entre argumentos | `Math.max()` → `-Infinity` ⚠️ | sin argumentos da extremos del vacío |
| `Math.clamp(x, min, max)` | acota (ES2025) | `Math.clamp(15, 0, 10)` → `10` | moderno; antes se hacía con min/max |
| `Math.random()` | decimal `[0, 1)` | | **no criptográfico** |

> [!tip] Clamp manual (patrón previo a `Math.clamp`)
> `const clamp = (x, min, max) => Math.min(Math.max(x, min), max);` — usado para sliders, volúmenes, valores de UI acotados. Con `Math.clamp(x, min, max)` desde 2025 se hace en una línea nativa.

> [!warning] `Math.random()` nunca da enteros
> Devuelve siempre un decimal. Y **no** usar `Math.round(Math.random() * 30)` como número aleatorio uniforme: los extremos (0 y 30) salen con la mitad de probabilidad que el resto. La fórmula correcta para un entero entre `min` y `max` **ambos incluidos** es:

```js
const randomEntero = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

console.log(randomEntero(0, 30)); // entero uniforme entre 0 y 30
console.log(randomEntero(5, 10)); // entero uniforme entre 5 y 10
```

Para tokens, claves o cualquier valor **seguro**, `Math.random()` no vale (es un PRNG predecible): usá `crypto.getRandomValues()` — ver [[Eventos|Web Crypto]] / APIs en [[Qué es una API]]:

```js
const idSeguro = () => crypto.randomUUID();               // UUID v4 nativo
const enteroSeguro = (max) =>
  crypto.getRandomValues(new Uint32Array(1))[0] % max;     // con módulo: leve sesgo si max no divide 2^32
```

## `0` negativo: el quirk de `-0`

JavaScript tiene dos ceros (`0` y `-0`) que casi siempre son intercambiables, pero se distinguen en comparaciones especiales (verificadas):

```js
console.log(-0 === 0);            // true (=== no los distingue)
console.log(Object.is(-0, 0));    // false (Object.is sí)
console.log((-5) ** 0);           // 1 — el -0 puede sobrevivir operaciones
console.log(1 / -0);              // -Infinity (¡el signo del cero afecta divisiones!)
```

Dónde aparece `-0` en la vida real: `Math.sign(-0)`, derivadas de funciones impares, y al parsear `'-0'`. Mitigación si importa: `const cero = x + 0;` normaliza `-0` → `0`.

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
> `x++` devuelve el valor **antes** de incrementar; `++x` devuelve el valor **ya incrementado**. `console.log(x++)` imprime el valor viejo; `console.log(++x)` el nuevo. Fuera de casos puntuales, úsalos solo como "cambiar en 1". Ver [[Estructuras de Control]] para el patrón en bucles `for`.

## Errores comunes

1. **`0.1 + 0.2 === 0.3` en comparaciones**: siempre `false`. Compará decimales con tolerancia: `Math.abs(a - b) < Number.EPSILON` (para valores ~1) o redondeá antes de comparar.
2. **`toFixed` usado como aritmética**: devuelve string y redondea al flotante vecino. Para dinero, centavos enteros.
3. **`isNaN('hola')` vs `Number.isNaN('hola')`**: la global convierte — da `true` por cualquier string no numérico; la de `Number` da `false` porque el string *no es* `NaN`. Convertí con `Number()` y verificá después.
4. **`parseInt` sin radix** y con strings con basura: `'12px'` → `12` (¡silencioso!), `'abc'` → `NaN`. Pasá siempre base 10 y validá la conversión completa si el input debe ser numérico puro.
5. **Dividir sin pensar en cero**: `1/0` no lanza error, da `Infinity` que contamina cálculos posteriores. Validá el denominador.
6. **Comparar con `===` sabiendo que hay `NaN` o `-0`**: `Object.is` resuelve ambos casos raros.
7. **IDs grandes en `Number`**: IDs de base de datos > 2^53 se redondean silenciosamente (`9007199254740992 === 9007199254740993` es `true`). Traélos como string o `BigInt` desde el backend.
8. **Mezclar `BigInt` y `Number`**: `10n + 1` lanza `TypeError`. Convertí un lado explícitamente.

## Rendimiento

- La aritmética numérica es lo más rápido del runtime: no hay micro-optimizaciones útiles en código normal.
- Donde sí importa: bucles que procesan **cientos de miles** de elementos con funciones por iteración — el boxing a objeto de `Number`/métodos (`toFixed`, `toLocaleString`) es caro en masa; calculá en crudo y formatea al presentar.
- `Math.random()` es un PRNG normal (mulberry32-ish): suficiente para juegos/UI, **no** para seguridad — `crypto.getRandomValues` tiene un costo mayor pero es el único aceptable para tokens.

## Cheatsheet

| Concepto | Ejemplo / valor | Nota clave |
|---|---|---|
| Un solo tipo | `typeof 20 === typeof 20.5` → `true` | ambos `number` (IEEE-754 doble) |
| Literales | `1e3`, `0xff`, `0o17`, `0b11`, `2_000` | guiones: solo legibilidad |
| Seguro hasta | `9007199254740991` | `Number.MAX_SAFE_INTEGER`; allende → `BigInt` |
| `0.1 + 0.2` | `0.30000000000000004` | error binario; comparar con tolerancia |
| `1/0`, `-1/0`, `0/0` | `Infinity`, `-Infinity`, `NaN` | sin excepción |
| `NaN === NaN` | `false` | usar `Number.isNaN` / `Object.is` |
| `isNaN` vs `Number.isNaN` | global convierte, estática no | regla de oro: `Number(x)` primero |
| `parseInt / parseFloat / Number` | parcial / parcial / total | radix siempre en `parseInt` |
| `toFixed(n)` | string con n decimales | no es aritmética |
| `Math.round(-2.5)` | `-2` | `.5` sube incluso con negativos |
| `Math.trunc` vs `floor` | `-2` vs `-3` (con `-2.9`) | trunc corta; floor va a −∞ |
| `Math.random()` | `[0, 1)` | no criptográfico; `crypto` para seguridad |
| Clamp | `Math.min(Math.max(x, min), max)` | o `Math.clamp` (ES2025) |
| `-0` | `Object.is(-0, 0)` → `false` | quirk; normaliza con `+ 0` |
| `BigInt` | `10n`, sin mezclar con `number` | `typeof` → `"bigint"` |

## Relacionadas

- [[Operadores]] — comparaciones (`===`) y operadores aritméticos/compuestos
- [[Strings]] — conversión entre texto y números (`parseInt`, `Number`)
- [[Booleanos]] — el otro tipo primitivo y sus valores truthy/falsy (`NaN` es falsy)
- [[Sintaxis Moderna]] — `const`/`let`, separadores numéricos, `**` y `BigInt`
- [[Funciones]] — predicados numéricos y callbacks matemáticos
- [[Array Methods]] — `reduce` como base de sumas, promedios y estadísticas
- [[Formato Json]] — `JSON.stringify` convierte `NaN`/`Infinity` en `null`
