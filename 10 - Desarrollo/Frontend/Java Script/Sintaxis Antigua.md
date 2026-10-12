---
tags:
  - desarrollo
  - frontend
  - sintaxis-antigua
tipo: concepto
estado: terminado
---

# Sintaxis Antigua

**Sintaxis antigua** es el JavaScript escrito **antes de ES2015** (ES6): `var` en vez de `let`/`const`, `function` en vez de arrow functions, IIFE en vez de módulos, callbacks en vez de `async/await`. Sigue corriendo en millones de páginas y librerías (jQuery, plugins, dashboards corporativos, código heredado en empresas), y **todo** navegador actual lo ejecuta sin problemas. Este nota es la guía para **leerlo, entenderlo y migrarlo**.

> [!info] Por qué estudiar sintaxis vieja
> 1. **Trabajo real**: la mayoría de bases de código en producción tienen años y fueron escritas así.
> 2. **Documentación vieja**: tutoriales pre-2015 usan `var` por todas partes.
> 3. **Dependencias**: leer el código fuente de librerías antiguas explica comportamientos que te sorprenden en producción.
> 4. **Entender el porqué**: cada convención rara de ES5 tiene una razón histórica — conocerla es conocer el modelo de scope real de JS.

## Variables con `var`

```jsx
var producto = "Monitor de 24 pulgadas"; // creando una variable
producto = "Monitor de 19 pulgadas"; // modificando una variable
var numero = 10; // Creando una variable numerica

console.log(producto);
console.log(numero);

// Inicializado variables sin valor
var numero2;
var saludo;

numero = 20;
saludo = "Hola como estan";

// Inicializando multiples variables
var categoria = "Computadoras",
  marca = "Marca famosa",
  calificacion = 5;

// Estilos para los nombres de variables
var nombreProducto; // camelcase
var nombre_producto; // snakecase
var NombreProducto; // Sintaxis utilizada en clases
```

> [!tip] Convención de nombres
> JS usa **camelCase** por convención (`nombreProducto`) desde siempre. `snake_case` se ve en código heredado de otros lenguajes; `PascalCase` (`NombreProducto`) se reserva para **constructores y clases** — usarlo en una variable simple engaña al lector pensando que es un constructor.

Las cuatro particularidades de `var` que definen su comportamiento:

### 1. Scope de **función**, no de bloque

`var` ignora las llaves `{ }`: solo crea límites dentro de **funciones**.

```js
if (true) {
  var dentro = 'visible en toda la función';
}
console.log(dentro); // 'visible en toda la función' — el bloque no aisló nada

for (var i = 0; i < 3; i++) { /* ... */ }
console.log(i); // 3 — i se escapó del bucle
```

Con `let`/`const` (ver [[Sintaxis Moderna]]) el mismo código lanzaría `ReferenceError`. Este es el **motivo principal** por el que `var` fue reemplazado.

### 2. *Hoisting*: la declaración sube, el valor no

El motor extrae todas las declaraciones `var` al **inicio de la función** (o del scope global). Leer la variable antes de la línea de asignación no da error — da `undefined`:

```js
console.log(precio); // undefined — sin error, la declaración ya "subió"
var precio = 100;
console.log(precio); // 100
```

El motor lo procesa en dos pasadas: primero registra `var precio` (inicializándolo a `undefined`), luego ejecuta las asignaciones en orden. Con `let`, esa misma línea lanza `ReferenceError` por la zona muerta (*TDZ*).

> [!warning] El hoisting de `var` es una trampa silenciosa
> No lanza error: produce `undefined` y el programa continúa con datos corruptos. Es el bug silencioso clásico del ES5: un typo en el nombre de la variable crea una **segunda** variable nueva en vez de fallar.

### 3. Redeclarar es legal

```js
var x = 1;
var x = 2;       // OK — no hay error
function f() { var x = 3; } // otra var x: OK
```

Dos `let` con el mismo nombre en el mismo scope lanzan `SyntaxError`. La redeclaración legal de `var` permitía que dos librerías sin módulos (era pre-módulos, todo colgaba del scope global) se pisaran variables sin que el navegador protestara — causa endémica de bugs en la era jQuery.

### 4. Inicialización opcional

Sin asignación, `var` queda en `undefined` (ver ejemplo arriba). A diferencia de `const`, **no exige valor inicial**.

## El problema del cierre en bucles con `var`

La combinación fatal de scope de función + `var`:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Sale: 3, 3, 3 — los tres closures comparten la MISMA i

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0);
}
// Sale: 0, 1, 2 — let crea una j nueva por iteración
```

Los `setTimeout` se ejecutan **después** del bucle; con `var` solo existe una `i`, ya incrementada a 3. Con `let`, cada iteración tiene su propia copia (semántica que el comité TC39 diseñó **precisamente** para replicar los closures por iteración). Arreglos clásicos del ES5: `let` (si el runtime lo soporta), una IIFE que capture `i`, o `bind`.

## Declaraciones de función: la otra forma de hoisting

A diferencia de `var` (que sube a `undefined`), una **function declaration** sube **completa** — puede llamarse antes de su línea:

```js
saludar('Ana'); // funciona — la función entera subió al inicio del scope

function saludar(nombre) {
  return `Hola, ${nombre}`;
}
```

Pero una **function expression** (asignar una función a una variable) solo sube la `var` de la variable, no el valor:

```js
saludar2('Ana'); // TypeError: saludar2 is not a function

var saludar2 = function (nombre) {
  return `Hola, ${nombre}`;
};
```

Diferencia práctica entre las dos formas:

| | Function declaration | Function expression |
| --- | --- | --- |
| Hoisting | completa (llamable antes) | solo la variable (`undefined`) |
| Anónima | no (siempre tiene nombre implícito) | sí |
| Uso en condicionales | peligroso (varían entre engines) | seguro |
| Estilo moderno equivalente | — | arrow: `const f = (…) => …` (ver [[Sintaxis Moderna]]) |

## IIFE: el patrón rey del scope con `var`

Como `var` no tiene scope de bloque, el ES5 creaba funciones **auto-invocadas** para encapsular:

```js
// IIFE: Immediately Invoked Function Expression
(function () {
  var secreto = 'solo visible aquí';
  // todo el código del módulo vive dentro
})();

console.log(secreto); // ReferenceError — el secreto quedó aislado
```

Por qué existió: era **la única** forma de tener privacidad antes de los módulos ES2015. Patrón de módulo clásico (patrón módulo UMD/CommonJS que ves en librerías antiguas):

```js
var Utils = (function () {
  var contador = 0; // estado privado
  return {
    incrementar: function () { return ++contador; }
  };
})();
Utils.incrementar(); // 1
```

Hoy: `type="module"` en el HTML da scope aislado de gratis y `class` con campos privados (`#contador`) resuelve lo demás — ver [[Sintaxis Moderna]].

## `with`: la sentencia prohibida

```jsx
var ctx = { a: 1, b: 2 };
with (ctx) {
  console.log(a + b); // 3 — lee ctx.a y ctx.b como si fueran variables locales
}
```

`with` añade un objeto al **scope chain**: dentro del bloque, cualquier nombre se resuelve contra el objeto primero. Problemas:

- **Impredecible**: si el objeto no tiene la propiedad, el nombre cae al scope exterior — bugs silenciosos.
- **Optimización imposible**: los motores no pueden resolver nombres estáticamente; V8 y SpiderMonkey lo desactivan/penalizan.
- **Prohibido en modo estricto** (que es obligatorio en módulos, clases y TypeScript): lanzaría `SyntaxError` — verificable con `node -e 'with({}){}'` vs un módulo.

Nunca se usa en código nuevo. Aparece en librerías viejas (prototipos de `document.all` en IE) y en transpilaciones primitivas. Su espiritu vive en un patrón seguro moderno: destructuring en parámetros de función.

## Era de prototipos: constructores sin `class`

Antes de `class` (ES2015), la OO de JS era **funciones constructoras + `prototype`**:

```jsx
function Producto(nombre, precio) {
  this.nombre = nombre;   // atributos: asignados a "this"
  this.precio = precio;
}

// Métodos en el prototipo: compartidos por TODAS las instancias
Producto.prototype.describir = function () {
  return this.nombre + " cuesta $" + this.precio;
};

var monitor = new Producto("Monitor", 300);
console.log(monitor.describir()); // "Monitor cuesta $300"
console.log(monitor instanceof Producto); // true
```

Por qué los métodos iban al `prototype` y no al constructor: cada `new Producto(...)` crea un objeto nuevo; si el método se asignara dentro del constructor, cada instancia guardaría **su propia copia** de la función (memoria desperdiciada). En `prototype` vive una sola copia compartida. Ver [[Objetos]] para el modelo de prototipos completo.

> [!warning] `this` es dinámico en ES5
> `this` depende de **cómo se llama** la función, no de dónde está escrita: `var f = producto.describir; f()` pierde el `this` y rompe. Por eso el ES5 estaba lleno de `var self = this;` y `fn.bind(this)` — el "self-capture" es una de las señales inequívocas de código viejo. Las arrow functions modernas no tienen `this` propio y eliminan el problema (ver [[Sintaxis Moderna]]).

## Otras señales de código ES5 al leerlo

| Señal | Qué significa | Alternativa moderna |
| ----- | ------------- | ------------------- |
| `var` por todas partes | scope de función | `let`/`const` |
| `function` expressions | antes de las arrows | `const f = () => {}` |
| `var self = this` / `var that = this` | `this` dinámico | arrow functions |
| `.bind(this)` en cada método | idem | arrow en los callbacks |
| IIFE `(function(){})()` | módulo/privacidad | `type="module"`, `class` privada |
| `arguments` | acceso a todos los parámetros | rest: `(...args) =>` |
| `arr.concat(other)` | unir arreglos | `[...arr, ...other]` |
| `for (var i = 0; ...)` | iterar con índice | `for...of`, `.map`, `.filter` |
| `$.ajax({…})` | HTTP con jQuery | `fetch` (ver [[Uso de Java Script]]) |
| `$(document).ready(fn)` | DOM listo | `defer` / `DOMContentLoaded` (ver [[Eventos]]) |
| `new Promise` + callbacks anidados | asincronia | `async/await` |
| `JSON.parse(JSON.stringify(x))` | copiar objeto | `structuredClone(x)` |
| `Object.keys(obj).forEach(...)` | recorrer | `for...of` con `Object.entries` |
| Comas finales y `;` obsesivos | estilo de linters ES5 | correcto igual, solo cosmético |
| `"use strict";` arriba del archivo | opt-in de modo estricto | implícito en módulos |

## Leer JavaScript heredado de jQuery (era 2010–2015)

Un fragmento típico y su traducción:

```jsx
// --- Código heredado ---
$(document).ready(function () {
  var $boton = $("#enviar");
  $boton.on("click", function () {
    var dato = $("#campo").val();
    if (dato == "") {
      alert("Campo vacío");
      return;
    }
    $.ajax({
      url: "/api/guardar",
      method: "POST",
      data: { dato: dato },
      success: function (respuesta) {
        console.log("Guardado", respuesta);
      },
      error: function () {
        console.warn("Falló");
      }
    });
  });
});
```

```js
// --- Equivalente moderno (JS puro) ---
document.querySelector('#enviar').addEventListener('click', async () => {
  const dato = document.querySelector('#campo').value;
  if (dato === '') {          // === estricto siempre
    console.warn('Campo vacío');
    return;                    // guard clause: salir temprano
  }
  try {
    const respuesta = await fetch('/api/guardar', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ dato })
    });
    console.log('Guardado', await respuesta.json());
  } catch {
    console.warn('Falló');
  }
});
```

Guía de lectura del lado viejo: `$()` es *querySelector* con aliases; `.on('click', fn)` es `addEventListener`; `$.ajax` es `fetch`; el patrón `success/error` es `try/await/catch`; `if (dato == "")` con `==` es una bomba de coerción (ver [[Operadores]]) — en migración, reemplazar por `===` o por el idiom moderno `if (!dato)`.

## Tabla de migración: sintaxis antigua → moderna

| ES5 (antiguo) | ES2015+ (moderno) | Nota |
| ------------- | ----------------- | ---- |
| `var x = 1` | `const x = 1` / `let x` | const salvo reasignación |
| `var x = 1; if (c) x = 2` | `let x = c ? 2 : 1` | expresión ternaria |
| `function f(a, b) {…}` | `const f = (a, b) => {…}` | o function declaration si se necesita hoisting |
| `function () { var self = this; … }` | arrow `() => { this.… }` | sin `this` propio |
| `f.bind(this)` | arrow en el callback | |
| `(function () {…})()` | módulo `type="module"` | scope gratis |
| `arguments` | rest `(...args)` | |
| `a.concat(b)` | `[...a, ...b]` | |
| `Object.assign({}, o, extra)` | `{ ...o, extra }` | literal de objeto |
| `for (var i = 0; i < n; i++)` | `for (let i = 0; i < n; i++)` | o `for...of` / `.map` |
| `for (var k in obj) if (obj.hasOwnProperty(k))` | `for (const [k, v] of Object.entries(obj))` | sin heredados |
| `$.ajax` / `XMLHttpRequest` | `fetch` + `async/await` | ver [[Uso de Java Script]] |
| `new Promise(function (res, rej) {…})` | `new Promise((res, rej) => {…})` | |
| callbacks anidados (pyramid) | `async/await` + `try/catch` | |
| `function P() {}; P.prototype.m = …` | `class P { m() {} }` | ver [[Objetos]] |
| `obj ? obj.prop : undefined` | `obj?.prop` | optional chaining |
| `x != null ? x : 0` | `x ?? 0` | nullish coalescing |
| `String(precio)` por comas | `` `…${precio}` `` | template literals |

## Errores comunes al tocar código viejo

1. **Copiar `var` a un bucle nuevo**: reintroducís el bug de closures (3,3,3). `let` siempre en bucles.
2. **Mezclar `var` y `let` con el mismo nombre**: `var x` + `let x` en el mismo scope = `SyntaxError` en engines modernos.
3. **Añadir `"use strict"` a un archivo ES5 entero**: puede romperlo — código que creaba variables globales por accidente (`x = 5` sin declarar) ahora lanza `ReferenceError`.
4. **Asumir que IIFE = módulo real**: no aísla imports ni exports; es solo un scope de variable.
5. **`==` heredado**: migrar a `===` puede cambiar comportamiento buscado (`dato == ""` aceptaba `0` y `null`; con `===` ya no). Revisar cada caso, no migrar a ciegas.
6. **IE quirks en copiado-pegado**: comentarios `<!--[if IE]>`, `attachEvent`, `ActiveXObject` — señales de código que asumía IE6–8; no tiene equivalente moderno directo, se reimplementa.

## Cheatsheet

| Concepto | Sintaxis antigua | Comportamiento clave |
| -------- | ---------------- | -------------------- |
| Variable | `var x` | scope de función, hoisting a `undefined`, redeclarable |
| Variable de bloque | — (no existía) | razón del `let` |
| Función | `function f() {}` | hoisting completo |
| Función anónima | `var f = function () {}` | sin hoisting del valor |
| Módulo | IIFE + objeto devuelto | privacidad manual |
| Constructor | `function P() {}` + `P.prototype` | `new P()` |
| Contexto `this` | `var self = this` / `.bind` | dinámico, se pierde al despegar |
| Iterador clásico | `for (var i = 0; …)` | i compartida: bug de closures |
| Scope extra | `with (obj) {…}` | prohibido en estricto |

## Relacionadas

- [[Sintaxis Moderna]] — el ES2015+ con el que se migra todo lo de esta nota
- [[Java Script]] — qué es JS y cómo corre hoy
- [[Funciones]] — el modelo de funciones completo (y `this`)
- [[Objetos]] — prototipos, `new`, herencia
- [[Iteradores]] — `for...of` y el reemplazo moderno de los bucles con índice
- [[Estructuras de Control]] — `if`/`switch` y guard clauses en código migrado
- [[Operadores]] — la coerción del `==` heredado explicada paso a paso
- [[Uso de Java Script]] — de `$.ajax` a `fetch`
- [[Eventos]] — de `$(document).ready` a `DOMContentLoaded`
