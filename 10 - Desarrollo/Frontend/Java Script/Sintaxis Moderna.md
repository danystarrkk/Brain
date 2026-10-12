---
aliases: ["ES2015", "ES6", "JavaScript moderno", "Sintaxis ES6+"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Java Script]]"
  - "[[Sintaxis Antigua]]"
  - "[[Funciones]]"
  - "[[Iteradores]]"
  - "[[Objetos]]"
  - "[[Operadores]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment"
---

# Sintaxis Moderna

**Sintaxis moderna** es el JavaScript de **ES2015 (ES6)** en adelante: `let`/`const`, destructuring, spread/rest, template literals, arrow functions, módulos, promesas y todo lo que el lenguaje añadió desde la gran revisión de 2015. Desde entonces el comité **TC39** publica una versión **cada junio** (ES2016, ES2017… ES2025) con cambios pequeños y predecibles. Todo cubierto aquí corre nativamente en cualquier navegador actual sin transpilar (ver [[Java Script]] para el mapa completo).

> [!info] El salto de ES5 a ES2015
> ES5 (2009) describía un lenguaje con `var`, scope de función y pocas abstracciones. ES2015 reescribió el lenguaje para el software moderno: scope de bloque, módulos reales, asincronia con promesas, y azúcar sintáctico que elimina el código repetitivo del ES5. La guía de migración completa vive en [[Sintaxis Antigua]].

## Variables: `let` y `const`

### `let`: variable reasignable con scope de bloque

```jsx
let producto = "Tablet";

// Reasignar el valor

producto = "Monitor";
producto = 20;   // también cambia de tipo: JS es de tipado dinámico

console.log(producto);

// Creando variables sin valor
let precio;

precio = 40;
console.log(precio);
```

`let` se comporta como `var` en lo dinámico (re-asignación, tipado libre) pero con dos diferencias fundamentales:

| | `var` | `let` |
| --- | --- | --- |
| Scope | función | **bloque** `{ }` |
| Lectura antes de declarar | `undefined` (hoisting) | **`ReferenceError`** (TDZ) |
| Redeclarar en el mismo scope | legal | `SyntaxError` |
| Se crea en `for` por iteración | no (bug de closures) | sí |

```js
if (true) {
  let x = 1;
}
// console.log(x); // ReferenceError: x is not defined

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i)); // 0, 1, 2 — i nueva por iteración
}
```

### Zona muerta temporal (TDZ)

Entre el inicio del bloque y la línea de la declaración, `let`/`const` **existen pero lanzan error** si se leen:

```js
function f() {
  console.log(a); // ReferenceError — ¡aunque 'a' está declarada abajo!
  let a = 5;
}
```

El motor no "no conoce" la variable: la conoce (está en la tabla de scope) pero la marca como **no inicializada** hasta la asignación. Es un error **temprano y ruidoso** — lo opuesto al `undefined` silencioso de `var`.

### `const`: constante de binding

```jsx
// No podemos reasignar valores cuando usamos const.
const producto = 20;

// Las variables siempre inician con un valor.

const valor = 10;
```

Tres reglas exactas:

1. **Exige inicialización** — `const x;` es `SyntaxError` (no hay momento posterior para asignar).
2. **Prohíbe reasignar** — `producto = 30` lanza `TypeError: Assignment to constant variable` (error en tiempo de ejecución, no de sintaxis).
3. **NO congela el contenido** — el *binding* es const, el objeto puede mutar:

```js
const usuario = { nombre: 'Ana' };
usuario.nombre = 'Luis';   // OK: mutar propiedades no es reasignar
// usuario = {};           // TypeError: Assignment to constant variable

const arr = [1, 2];
arr.push(3);               // OK
// arr = [9];              // TypeError
```

> [!tip] Política de elección
> **`const` por defecto; `let` solo cuando hay reasignación real; `var` nunca.** Con `const` el lector sabe de un vistazo qué puede cambiar. `Object.freeze` es el que congela contenido (ver [[Objetos]]) — `const` no.

## Arrow functions (sintaxis de función)

```js
// Function expression (ES5) vs arrow (ES2015+)
const doble = function (n) { return n * 2; };
const doble2 = (n) => n * 2;          // una línea: return implícito
const suma = (a, b) => a + b;         // paréntesis cuando hay 2+ params
const sinParams = () => Date.now();   // sin params: paréntesis vacíos
const cuadrado = n => n * n;          // 1 param: paréntesis opcional
const hacer = n => ({ valor: n });    // retornar objeto: PARÉNTESIS obligatorios
```

- **`this` léxico**: las arrow no tienen `this` propio — heredan el del scope envolvente. Elimina `var self = this` y `.bind(this)` del ES5.
- **No son hoisteadas** (son expresiones: la variable sube, la función no).
- **No tienen `arguments`** ni `new.target`; no pueden usarse como constructores (`new (() => {})` lanza `TypeError`).

Ver [[Funciones]] para el modelo completo de funciones (que incluye arrows en detalle).

## Template literals

```js
const nombre = 'Ana', edad = 28;
console.log(`Hola ${nombre}, tienes ${edad} años`);

// Multilínea real: los saltos de línea se conservan
const html = `
  <div class="card">
    <h2>${nombre}</h2>
  </div>
`;

// Aritmética dentro de la interpolación
const area = `El área es ${(10 * 5) / 2} m²`;
```

Backtick (`` ` ``) en vez de comillas: interpola con `${expresión}` (acepta **cualquier expresión**, incluidas llamadas y ternarios), respeta saltos de línea, y escapa automáticamente `\` y backticks al convertir a string.

### Template literals etiquetados

Una función "etiqueta" recibe las partes crudas del template:

```js
function resaltar(strings, ...vals) {
  // strings: ["Total: ", " USD"]  — strings.raw conserva los backslashes crudos
  // vals:    [300]
  return strings.reduce((acc, s, i) => acc + s + (vals[i - 1] ?? ''), '');
}
console.log(resaltar`Total: ${300} USD`); // "Total: 300 USD"
```

`strings.raw` da la versión **sin interpretar escapes** (`` String.raw`a\nb` `` produce literalmente `a\nb`, 4 caracteres). Uso real: i18n (traducciones con placeholders), SQL parametrizado (evita inyección), escaping seguro — el motivo de existir de `String.raw`.

## Destructuring: extraer como si fueran variables

Asignar propiedades/posiciones a variables con una expresión que "replica" la forma del dato.

### Objetos

```js
const usuario = { nombre: 'Ana', edad: 28, email: 'ana@x.com' };

const { nombre, edad } = usuario;          // nombre='Ana', edad=28
const { nombre: n, edad: e = 0 } = usuario; // renombrar (n) y valor por defecto (e)
const { password, ...seguro } = usuario;    // rest: resto de propiedades en un objeto nuevo
console.log(password); // undefined (no existía — sin error)
```

### Arreglos (posiciones)

```js
const nums = [10, 20, 30];
const [primero, segundo] = nums;      // primero=10, segundo=20
const [a = 1, , tercero = 99] = nums; // a=10; se salta el 2º; tercero=30
const [x, ...demas] = nums;           // x=10, demas=[20,30]
```

### Anidado y renombrado combinado

```js
const pedido = {
  id: 7,
  cliente: { nombre: 'Luis', direcciones: [{ ciudad: 'Madrid' }] }
};

const {
  id,
  cliente: { nombre: nomCliente, direcciones: [{ ciudad }] }
} = pedido;
// id=7, nomCliente='Luis', ciudad='Madrid' — la forma del template guía la extracción
```

### Swap sin variable temporal

```js
let m = 1, n = 2;
[m, n] = [n, m];   // m=2, n=1
```

### Destructuring en parámetros de función

```js
function render({ titulo, ancho = 100, alto = 50 }) {
  return `${titulo} ${ancho}x${alto}`;
}
render({ titulo: 'Hero' }); // "Hero 100x50"

const paleta = ([prim, seg = '#000']) => `${prim} / ${seg}`;
paleta(['red']);            // "red / #000"
```

> [!warning] Desestructurar algo que no existe lanza error
> `const { a } = null` o `= undefined` lanzan `TypeError`. Con datos de API que pueden faltar, encadená con optional chaining: `const { a } = respuesta?.data ?? {}` — ver más abajo.

## Spread y rest

Mismo `...` con dos lecturas según posición:

| Posición | Nombre | Qué hace | Ejemplo |
| -------- | ------ | -------- | ------- |
| Después de un operador/argumento | **spread** | expande | `[...arr1, ...arr2]`, `fn(...args)`, `{ ...obj }` |
| Al final de una lista de variables | **rest** | comprime el resto | `const [a, ...rest] = arr`, `(...params) => {}` |

```js
// Spread en arreglos: copiar y concatenar
const base = [1, 2];
const extendido = [...base, 3, ...[4, 5]]; // [1,2,3,4,5]

// Spread en objetos: clonar superficial y fusionar (el último gana)
const defaults = { tema: 'oscuro', lang: 'es' };
const config = { ...defaults, tema: 'claro' }; // { tema:'claro', lang:'es' }

// Spread en llamadas: Math.max sin apply
console.log(Math.max(...[3, 9, 2])); // 9

// Rest en funciones: recibir argumentos sueltos
function sumar(...nums) {            // nums = [1,2,3]
  return nums.reduce((a, b) => a + b, 0);
}
sumar(1, 2, 3); // 6
```

Ver [[Arreglos]] para el efecto en arreglos y [[Objetos]] para el de objetos (shallow: los objetos anidados quedan **compartidos**, igual que `Object.assign`).

## Propiedades shorthand y claves computadas

```js
const nombre = 'Ana', edad = 28;

// Shorthand: la clave igual que la variable
const persona = { nombre, edad };            // { nombre: 'Ana', edad: 28 }

// Método shorthand: sin function y sin dos puntos
const cuenta = {
  total: 0,
  agregar(v) { this.total += v; },           // equivalente a agregar: function(v) {...}
};

// Claves computadas: la clave es una expresión en corchetes
const campo = 'email';
const formulario = { [campo]: 'ana@x.com', ['x_' + 'y']: 1 };
console.log(formulario.email, formulario.x_y);
```

Shorthand y methods fueron el "desperdicio menos" del ES5; las claves computadas reemplazan hacks como `obj[campo] = valor` post-creación.

## Parámetros por defecto

```js
function crearUsuario(nombre, rol = 'viewer', activo = true) {
  return { nombre, rol, activo };
}
crearUsuario('Ana');            // { nombre:'Ana', rol:'viewer', activo:true }
crearUsuario('Ana', 'admin');   // { nombre:'Ana', rol:'admin',  activo:true }
crearUsuario('Ana', undefined); // el default se aplica SOLO ante undefined
crearUsuario('Ana', null);      // null NO es undefined: rol queda null
```

- El default se evalúa **en cada llamada**, no una vez.
- Solo `undefined` dispara el default; `null`, `0`, `''` y `false` **no** (esto es lo que distingue a los defaults de `||` — ver [[Operadores]]).
- Los defaults pueden referenciar parámetros anteriores: `(a, b = a * 2) => b`. Referenciar uno posterior lanza TDZ — mismo error ruidoso que `let`.

## `for...of`: el bucle de valores

Recorre **valores** de cualquier iterable (arreglos, strings, Map, Set, NodeLists — ver [[Iteradores]]):

```js
for (const item of [10, 20]) console.log(item);   // 10, 20

let palabra = '';
for (const ch of 'hola') palabra += ch.toUpperCase(); // HOLA

for (const [clave, valor] of new Map([['a', 1]])) console.log(clave, valor); // a 1
```

Contrasta con `for...in` (claves/enumerables, incluye heredadas — para objetos, no para arreglos). `for await...of` itera promesas secuencialmente: `for await (const res of respuestas)`. Ver [[Iteradores]] para el protocolo de iteración completo.

## Opcionalidad y valores nulos: `?.` y `??`

### Optional chaining `?.`

```js
const usuario = { perfil: { direccion: null }, contactar() { return 'mail'; } };

console.log(usuario?.perfil?.direccion);  // null (no lanza: corta y devuelve undefined)
console.log(usuario?.noExiste?.x);        // undefined
console.log(usuario.contactar?.());       // 'mail' — verifica que la función existe
console.log(usuario?.direccion?.calle);   // undefined — cadena entera se salta
console.log(primer?.[0]);                 // acceso opcional por índice
```

Si la parte izquierda es `null`/`undefined`, **toda la cadena** evalúa a `undefined` sin ejecutar nada más. Reemplaza patrones frágiles del ES5 como `obj && obj.a && obj.a.b`.

### Nullish coalescing `??`

```js
const config = { reintentos: 0, mensaje: '' };

console.log(config.reintentos ?? 3);   // 0  — 0 no es nulo, se respeta
console.log(config.reintentos || 3);   // 3  — || trata 0 como vacío (bug clásico)
console.log(config.mensaje ?? 'Hola'); // ''  — string vacío también se respeta
console.log(config.tema ?? 'oscuro');  // 'oscuro' — solo null/undefined caen al default
```

| Expresión | Devuelve el izquierdo si es… | Devuelve el derecho si es… |
| --------- | --------------------------- | -------------------------- |
| `\|\|` | truthy (todo lo que no esté en la lista falsy) | `0`, `''`, `null`, `undefined`, `NaN`, `false`, `0n` |
| `??` | **cualquier cosa salvo** `null`/`undefined` | solo `null` o `undefined` |

> [!warning] `??` no se puede mezclar con `&&`/`||` sin paréntesis
> `a ?? b || c` es `SyntaxError` (la precedencia sería ambigua). Obligatorio: `(a ?? b) || c`.

### Asignación lógica `&&=`, `||=`, `??=`

```js
let op = { debug: false, nivel: 0, tema: null };

op.debug ||= true;    // solo si es falsy:  false → true
op.nivel ||= 5;       // 0 es falsy → 5      (¡cuidado con !)
op.tema ??= 'claro';  // solo si es null/undefined: null → 'claro'
op.debug &&= console.log('debug activo'); // solo ejecuta si era truthy

// Equivalentes explícitos:
// x ||= v   ≡   x || (x = v)
// x ??= v   ≡   x ?? (x = v)
// x &&= v   ≡   x && (x = v)
```

`??=` es el de defaults seguro; `||=` respeta el atajo `x || (x = v)` con todas sus trampas de truthiness.

## BigInt: enteros sin límite

```js
const grande = 9007199254740993n;       // sufijo n (o BigInt(...))
console.log(2n ** 10n);                 // 1024n
console.log(typeof grande);             // "bigint"

const seguro = Number.MAX_SAFE_INTEGER; // 9007199254740991 — el techo de Number
console.log(Number(9007199254740993) === 9007199254740992); // true: Number perdió precisión
```

- `Number` es float64: pierde precisión más allá de 2⁵³−1. `BigInt` representa enteros **arbitrariamente grandes** exactos.
- **Prohibido mezclar** con `Number` en operaciones: `1n + 1` lanza `TypeError`. Convertí explícitamente: `BigInt(1) + 1n`.
- `typeof` devuelve `"bigint"`; `1n == 1` es `true` pero `1n === 1` es `false` (tipos distintos).
- No aplica a `Math` ni a `%` sobre floats; comparaciones y aritmética básica (`+ - * / % **`) sí, con `n` en ambos lados.
- Uso real: IDs de 64 bits, cripto, contadores que no pueden perderse.

## Símbolos: identificadores únicos

```js
const idUnico = Symbol('id');
const objeto = { [idUnico]: 123, nombre: 'x' };
console.log(objeto[idUnico]);       // 123
console.log(Object.keys(objeto));   // ['nombre'] — symbol no aparece en keys/JSON
```

Cada `Symbol()` es **único e intercambiable** — sirve como clave de propiedad que nadie puede chocar (tags internos, `Symbol.iterator` que alimenta a `for...of`, ver [[Iteradores]]). Dos `Symbol('id')` **no** son iguales; para claves compartidas se usa `Symbol.for('id')` (registro global).

## Cambios en objetos y arreglos (resumen con enlaces)

```js
const obj = { a: 1, b: 2 };
Object.keys(obj);        // ['a','b']         — claves propias
Object.values(obj);      // [1, 2]
Object.entries(obj);     // [['a',1],['b',2]] — ideal para for...of
Object.hasOwn(obj, 'a'); // true              — reemplaza hasOwnProperty (ver [[Objetos]])
Object.assign({}, obj, { c: 3 });             // fusión (o { ...obj, c: 3 })

[1, 2].includes(2);      // true              — indexOf-friendly para humanos (ver [[Array Methods]])
[3, 1, 2].find(n => n > 1);                   // 3 — ver [[Array Methods]]
```

## Clases y asincronía (puertas de entrada)

```js
class Animal {
  #secreto = 'privado real';                 // campos privados (ES2022)
  static total = 0;
  constructor(nombre) { this.nombre = nombre; Animal.total++; }
  describir() { return `Soy ${this.nombre}`; }
}
```

ES2015 introdujo `class` (azúcar sobre prototipos — ver [[Objetos]] para el modelo real) y ES2017 `async/await` sobre promesas. Ambos tienen nota propia: [[Funciones]] para el detalle funcional. Lo importante de sintaxis aquí: son parte del mismo salto moderno que destructuring y arrows.

## Proceso de propuestas TC39: cómo nace la sintaxis

Cada feature pasa por **5 etapas** antes de ser estándar:

| Etapa | Nombre | Qué significa |
| ----- | ------ | ------------- |
| 0 | **Strawman** | idea de un champion; puede no avanzar nunca |
| 1 | **Proposal** | el comité la considera viable; experimentable con polyfills |
| 2 | **Draft** | descripción inicial completa; **cambios breaking todavía probables** |
| 3 | **Candidate** | feedback de implementadores; solo bugs menores de por medio |
| 4 | **Finished** | aprobada; llegará a los navegadores (casi seguro en el siguiente release) |

Referencias vivas: el tracker oficial <https://github.com/tc39/proposals> lista cada propuesta con su etapa. Regla práctica: **solo usa en producción lo que está en Stage 4** (y con soporte en tus navegadores meta). Stage 1–2 en producción = código que puede cambiar o morir (el caso famoso: decorators, que giró por etapas durante años).

### ¿Cómo saber si un feature me sirve en mi navegador?

| Recurso | Para qué |
| ------- | -------- |
| <https://caniuse.com> | soporte por navegador de cada feature (compat de w3) |
| <https://compat-table.github.io/compat-table/es6/> | tabla clásica de ES6+ por engine |
| MDN — "Baseline" | badge integrado: "disponible desde hace X meses en todos los engines" |
| `node -e "console.log(…)"` | probar el comportamiento real del engine (ver [[La Consola]]) |
| `typeof feature !== 'undefined'` | feature detection manual (rara vez necesario) |

> [!tip] No hagas *browser sniffing*
> Detectar versiones de navegador (`navigator.userAgent`) es frágil. Lo correcto es **soportar lo universal** (todo ES2015+ salvo lo muy reciente) y, para lo cutting-edge, polyfills o feature detection puntual.

## Errores comunes

1. **Reasignar `const`** — `TypeError: Assignment to constant variable`. Decidir `let` desde el diseño de la variable.
2. **Leer una variable en su TDZ** — `ReferenceError` al usarla antes de la línea (clásico con defaults de parámetros que referencian otro parámetro posterior).
3. **Destructuring de `null`/`undefined`** — `const { a } = null` lanza `TypeError`. Proteger con `?? {}`.
4. **Olvidar los paréntesis del objeto en arrow** — `const f = () => { valor: 1 }` retorna... un bloque con label. Para retornar un objeto: `() => ({ valor: 1 })`.
5. **Confundir `||` y `??`** — `config.limite || 10` rompe cuando el límite es `0`. `??` para defaults, `||` solo si lo falsy realmente significa "sin valor".
6. **Mezclar `??` con `||`/`&&` sin paréntesis** — `SyntaxError` del parser.
7. **Mezclar `BigInt` y `Number`** — `TypeError` en la operación; convertir siempre explícitamente.
8. **Copiar objetos con spread y esperar deep-clone** — `const copia = { ...original }` comparte referencias anidadas; `structuredClone` para copia profunda.
9. **Usar `for...in` en arreglos** — itera claves como strings e incluye heredadas; `for...of` o `map` para valores (ver [[Iteradores]]).

## Cheatsheet

| Feature | Ejemplo mínimo | Reemplaza a (ES5) |
| ------- | -------------- | ----------------- |
| `let` | `let x = 1` | `var` |
| `const` | `const X = 1` | — |
| Arrow | `(a, b) => a + b` | `function (a, b) {}` |
| Template | `` `Hola ${n}` `` | `'Hola ' + n` |
| Destructuring | `const { a } = obj` | `obj.a` repetido |
| Rest | `(...args) => {}` | `arguments` |
| Spread | `[...arr, x]` | `concat` |
| Defaults | `f(a = 1) {}` | `a = a \|\| 1` |
| `for...of` | `for (const v of arr)` | `for (var i…)` |
| Optional chain | `a?.b?.c` | `a && a.b && a.b.c` |
| Nullish | `x ?? 10` | `x != null ? x : 10` |
| Logical assign | `x ??= 10` | `x = x \|\| 10` |
| BigInt | `10n` | — (no existía) |
| Symbol | `Symbol('id')` | strings como claves |
| Object.entries | `for (const [k,v] of …)` | `for…in` + `hasOwnProperty` |

## Relacionadas

- [[Sintaxis Antigua]] — el ES5 de donde venimos y su guía de migración
- [[Java Script]] — cómo corre este código en el navegador y en Node
- [[Funciones]] — arrows, closures y `this` en profundidad
- [[Iteradores]] — el protocolo detrás de `for...of` y `Symbol.iterator`
- [[Objetos]] — prototipos, clases, `Object.*` y congelamiento
- [[Operadores]] — precedencia de `?.`, `??`, `**` y el resto
- [[Estructuras de Control]] — dónde vive `for...of` entre las estructuras de flujo
- [[Arreglos]] — arreglos vistos a fondo
- [[Array Methods]] — `map`/`filter`/`reduce` como alternativa a bucles
- [[Formato Json]] — `JSON.stringify`/`parse` en el flujo moderno de datos
