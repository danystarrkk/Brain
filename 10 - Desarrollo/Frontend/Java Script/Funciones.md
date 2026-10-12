---
aliases: ["Functions", "Funciones en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Estructuras de Control]]"
  - "[[Iteradores]]"
  - "[[Sintaxis Moderna]]"
  - "[[Arreglos]]"
  - "[[Array Methods]]"
  - "[[Objetos]]"
  - "[[Operadores]]"
  - "[[ReactJS - Fundamentos]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Functions"
---

# Funciones

Una **función** es un bloque de código con nombre (o asignado a una variable) que se ejecuta **cuando se le llama**, no cuando se define. Sirven para dividir el programa en piezas pequeñas, **reutilizables** y con una sola responsabilidad. En JS, las funciones son **valores de primera clase**: se pueden guardar en variables, pasar como argumentos, devolver desde otras funciones y crear dinámicamente. Esa propiedad es la que habilita los callbacks, los higher-order functions y todo el estilo declarativo de [[Arreglos|los arreglos]].

```js
function sumar() {
  console.log(2 + 2);
}

sumar(); // llamada a la función: aquí se ejecuta
```

> [!info] Función como "receta"
> Definir una función es escribir la receta; llamarla es cocinar. La receta no se consume: puede ejecutarse 0, 1 o 1000 veces, y cada ejecución tiene su propio espacio de variables (su propio *frame* en el call stack).

## Las cinco formas de definir una función

JavaScript tiene cinco sintaxis distintas para funciones, y **no son intercambiables**: cada una tiene reglas propias de hoisting, `this`, `arguments` y si puede usarse como constructor.

| Forma | Ejemplo | ¿Hoisting? | Nombre | `arguments` | `this` dinámico | ¿Con `new`? | `prototype` |
|---|---|---|---|---|---|---|---|
| Declaración | `function f() {}` | Sí, completa | Obligatorio | Sí | Sí (según llamada) | Sí | Sí |
| Expresión | `const f = function () {}` | No (la variable está en TDZ) | Opcional (nombre interno) | Sí | Sí | Sí | Sí |
| Expresión nombrada | `const f = function g() {}` | No | Interno (`g`) | Sí | Sí | Sí | Sí |
| Arrow function | `const f = () => {}` | No | — | **No** | **No** (lexicoal, la del cierre) | **No** | **No** |
| Método abreviado | `obj = { f() {} }` | No (vive en el objeto) | Como la propiedad | Sí | Sí | Sí | Sí |

```js
// Function Declaration: puede llamarse ANTES de definirse (hoisting)
sumar(); // funciona

function sumar() {
  console.log(2 + 2);
}

// Function Expression: guardada en una variable, NO puede usarse antes
sumar2(); // ReferenceError: no se puede usar antes de su inicialización

const sumar2 = function () {
  console.log(3 + 3);
};
```

> [!tip] Expresión nombrada: nombre visible solo para la función
> `const fact = function facto(n) { return n <= 1 ? 1 : n * facto(n - 1); }` — el nombre `facto` existe **solo dentro** de la función (útil para recursión legible y stack traces legibles), pero desde afuera la variable se llama `fact`. No confundir con la declaración `function facto() {}`, que sí esnombrada para todo el scope.

> [!warning] `var` vs `let`/`const` en el hoisting
> Con `var`, la función expression **sí** se puede "llamar" antes (sin error), pero recibirá `undefined` → `TypeError: sumar2 is not a function`. Con `let`/`const` el error es más claro: `ReferenceError` por la zona temporal muerta (*TDZ*). Ver [[Sintaxis Moderna]] para `var`/`let`/`const`.

> [!info] Funciones vs métodos
> `parseInt("20")` es una **función** global: se llama suelta. `"20".toString()` es un **método**: pertenece a un objeto y se llama con punto. Misma idea, distinta forma de invocación — y esa diferencia decide cómo se resuelve `this` (ver "El objeto `this`" más abajo).

## Parámetros y argumentos

Los **parámetros** son los nombres de la entrada; los **argumentos**, los valores concretos que se pasan al llamar:

```js
function sumar(a, b) {
  console.log(b + a);
}

sumar(2, 4);               // 6
sumar(4534523, 23454325);  // reutilizable con otros valores
```

JS **no valida la cantidad de argumentos**: si faltan llegan como `undefined`, si sobran se ignoran (quedan accesibles en el objeto `arguments` de las funciones no-arrow):

```js
function ver(a, b) { return [a, b, arguments.length]; }
ver(1);        // [1, undefined, 1]
ver(1, 2, 3);  // [1, 2, 3]  ← el tercero se ignora… pero existe en arguments
```

Ejemplo práctico con `prompt` y `split`:

```js
function saludar(nombre, apellido) {
  console.log(`Hola ${nombre} ${apellido}, es un gusto saludarte`);
}

const nombreCompleto = prompt('Digita tu nombre y tu apellido');
const partes = nombreCompleto.split(' '); // separa por espacio -> arreglo

saludar(partes[0], partes[1]);
```

### Parámetros por defecto (*default parameters*)

Si el argumento no llega — o llega explícitamente como `undefined` — se usa el valor por defecto:

```js
function saludar(nombre = 'Invitado', apellido = '') {
  console.log(`Hola ${nombre} ${apellido}, es un gusto saludarte`);
}

saludar('Daniel');        // "Hola Daniel , es un gusto saludarte"
saludar();                // "Hola Invitado , es un gusto saludarte"
saludar('Daniel', 'Pérez');
```

Tres semánticas que casi nadie conoce:

```js
// 1. Solo UNDEFINED dispara el default; null NO
((a = 5) => a)(null);   // null  (null es un valor "presente")
((a = 5) => a)(undefined); // 5

// 2. El default se evalúa en cada llamada, en el scope de la función (puede usar parámetros anteriores y variables externas)
function crear(nombre, saludo = `Hola ${nombre}`) { return saludo; }
crear('Ana'); // "Hola Ana"

// 3. Los defaults tienen TDZ entre parámetros: no podés usar un parámetro antes de declararlo
// function mala(a = b, b = 1) {} → ReferenceError al llamar
```

> [!tip] Default vs comprobación manual
> `function f(options = {}) { const { x = 1 } = options; }` reemplaza el antiguo patrón `if (options === undefined) options = {}`. Combinado con desestructuración de parámetros (siguiente sección) es la forma moderna de firmas flexibles.

### Rest parameters (`...args`)

Recoge los argumentos **sobrantes** en un arreglo real:

```js
function sumar(...numeros) {          // numeros es un Array de verdad
  return numeros.reduce((a, n) => a + n, 0);
}
sumar(1, 2, 3); // 6
```

- Va **siempre al final** de la lista de parámetros (solo puede haber uno).
- Reemplaza al objeto `arguments` (que no es un arreglo: no tiene `map`, `reduce`…).
- A diferencia de `arguments`, se comporta como parámetro normal: no se sincroniza con los argumentos posicionales.

### Desestructuración de parámetros

Los parámetros pueden ser patrones de desestructuración — la firma declara **qué forma tiene** el dato que llega (muy usado en APIs de componentes, ver [[ReactJS - Fundamentos]] y [[Objetos]]):

```js
function crearProducto({ nombre, precio = 0, categoria = 'general' }) {
  return { nombre, precio, categoria };
}

crearProducto({ nombre: 'Monitor', precio: 300 });
// { nombre: 'Monitor', precio: 300, categoria: 'general' }

// Con arreglos: primer y segundo elemento
function primeros([a, b] = []) { return [a, b]; }
```

> [!warning] Desestructurar un objeto inexistente lanza error
> `function f({ x }) {}` llamada como `f()` lanza `TypeError: Cannot destructure…`. Solución: valor por defecto `({ x } = {})`.

### El objeto `arguments`

Solo existe en funciones **no-arrow**, y es un *array-like* (indexado, pero sin métodos de arreglo):

```js
function clásica() {
  console.log(arguments[0], arguments.length);
  console.log(Array.from(arguments)); // convertirlo si se necesitan métodos
}
```

Las arrow functions **no tienen `arguments` propio**: heredan el del scope exterior (casi siempre un comportamiento no deseado). Usá rest parameters.

## Comunicación entre funciones

Las funciones se llaman entre sí: así se arma el flujo del programa desde un punto de entrada, encadenando responsabilidades pequeñas:

```js
iniciarApp();

function iniciarApp() {
  console.log('Iniciando APP');
  segundaFuncion();
}

function segundaFuncion() {
  console.log('Hola desde la segunda función');
  login('Daniel');
}

function login(nombre) {
  console.log('Autenticando usuario.....');
  console.log(`Usuario ${nombre} correctamente autenticado`);
}
```

Cada función hace una cosa y delega la siguiente — el patrón de *orquestador* (`iniciarApp`) que llama a pasos especializados. Verificado arriba: la declaración permite que `iniciarApp` llame a `segundaFuncion` definida más abajo gracias al hoisting.

## Retorno de valores

`return` termina la función y entrega un resultado que quien la llamó puede guardar o usar:

```js
function sumar(a, b) {
  return a + b; // sin return, la función retornaría undefined
}

const resultado = sumar(2, 3);
console.log(resultado); // 5
```

Reglas de la mecánica de `return`:

| Situación | Qué retorna |
|---|---|
| Sin `return` (o `return;` solo) | `undefined` |
| `return expresion;` | el valor de la expresión |
| `return` a mitad de un `if` | corta la ejecución ahí (early return) |
| Varios `return` en la función | solo se ejecuta el que se alcanza; el resto es dead code |
| `return` dentro de un bucle | sale también del bucle |

> [!warning] Una sola sentencia con `return`
> En una función con llaves, si abrís un bloque `{` después del `return` y la llave de cierre va sola, el `return` ya no es alcanzable: ASI solo aplica hasta la siguiente `{`. Pasa mucho con callbacks: `carrito.forEach(p => { total += p.precio })` con el `{` en la misma línea del arrow sin `return` — el bloque es el cuerpo, y retorna `undefined` (inofensivo en `forEach`, un bug en `map`).

Ejemplo combinando funciones con estado compartido:

```js
let total = 0;

function agregarCarrito(precio) {
  return (total += precio); // suma al total global y lo retorna
}

function calcularImpuesto(total) {
  return total * 1.15; // 15% de impuesto
}

agregarCarrito(300);
agregarCarrito(100);
agregarCarrito(700);

const totalPagar = calcularImpuesto(total);

console.log(total); // 1100
console.log(`El total a pagar es de: ${totalPagar}`); // 1265
```

## Ámbito (*scope*) y cadena de ámbitos

Toda función crea su propio ámbito: las variables declaradas dentro (`let`, `const`, parámetros) **no existen fuera**. Cuando una función necesita un nombre, lo busca en su propio ámbito y, si no está, sube por la **cadena de ámbitos** (scope chain) hacia los ámbitos contenedores hasta llegar al ámbito global:

```js
const nivel1 = 'global';

function externa() {
  const nivel2 = 'de externa';

  function interna() {
    const nivel3 = 'de interna';
    console.log(nivel3, nivel2, nivel1); // busca hacia arriba: los tres se ven
  }

  interna();
  // console.log(nivel3); // ReferenceError: nivel3 no existe aquí
}
```

La regla clave: **un ámbito interior ve todo lo del exterior, pero el exterior no ve lo del interior**. Ese one-way es lo que hace posible los closures.

## Closures (*closures*)

Una closure es una función que **recuerda el ámbito donde fue creada**, incluso después de que ese ámbito haya terminado de ejecutarse. En la práctica: toda función anidada es una closure de su padre.

```js
function crearContador() {
  let cuenta = 0;               // variable privada de este ámbito
  return function () {
    cuenta += 1;                // accede a la variable del padre "muerto"
    return cuenta;
  };
}

const contadorA = crearContador();
contadorA(); // 1
contadorA(); // 2

const contadorB = crearContador(); // ámbito NUEVO: otro contador independiente
contadorB(); // 1
```

Por qué funciona: cuando `crearContador` termina, su variable `cuenta` **no se destruye** mientras la función devuelta siga viva, porque esa función mantiene una referencia al ámbito. Mental model: la closure "empaqueta" el entorno léxico junto con el código.

> [!warning] Implicación de memoria
> Mientras una closure viva mantenga referencias a un objeto grande (por ejemplo, un closure que captura un DOM node o un array gigante), **ese objeto no se puede recolectar**. Patrón clásico de leak: registrar closures en listeners globales y nunca removerlas. Regla práctica: cerrá el capture sobre **solo lo que necesitás**, no sobre todo el contexto.

### El bug clásico del closure en bucles

```js
const funciones = [];
for (var i = 0; i < 3; i++) {
  funciones.push(() => i);
}
console.log(funciones.map((f) => f())); // [3, 3, 3]  ← ¡no [0, 1, 2]!
```

No hay tres valores de `i`: hay **una sola variable** `var` compartida por todos los callbacks, y todos leen su valor final (3). Las tres arrow functions son closures del **mismo** ámbito. Tres soluciones verificadas:

```js
// 1. let en el bucle: crea una variable NUEVA en cada iteración (recomendada)
const f1 = [];
for (let i = 0; i < 3; i++) f1.push(() => i);
console.log(f1.map((f) => f())); // [0, 1, 2]

// 2. IIFE: congelar cada valor en un parámetro
const f2 = [];
for (var j = 0; j < 3; j++) f2.push((function (x) { return () => x; })(j));
console.log(f2.map((f) => f())); // [0, 1, 2]

// 3. forEach: el callback ya es un ámbito nuevo por iteración
const f3 = [];
[0, 1, 2].forEach((i) => f3.push(() => i));
console.log(f3.map((f) => f())); // [0, 1, 2]
```

Ver [[Iteradores]] y [[Estructuras de Control]] para los bucles en detalle.

## El objeto `this`

`this` **no se resuelve al definir la función sino al llamarla** (excepto en arrow functions). Las cuatro reglas de resolución, en orden de prioridad:

| Regla | Cuándo aplica | `this` resultante |
|---|---|---|
| 1. `new` | Llamada con `new f()` | objeto recién creado |
| 2. `call`/`apply`/`bind` | Llamada explícita con uno de ellos | el primer argumento (`thisArg`) |
| 3. Método (`obj.f()`) | La función es propiedad de un objeto en el momento de llamar | el objeto receptor (`obj`) |
| 4. Suelto / callback | `f()`, `setTimeout(f)`, `map(f)` | `undefined` en modo estricto; `globalThis` en modo sloppy |

```js
const reproductor = {
  canción: 'Bachata',
  normal() { return this.canción; },   // regla 3: this = el objeto
  flecha: () => this.canción,          // arrow: this NO es this del objeto
};

reproductor.normal();  // "Bachata"
reproductor.flecha();  // undefined: this es el del ámbito exterior
```

> [!warning] Arrow functions no tienen `this` propio
> La arrow "hereda" `this` **léxico** del scope donde fue escrita — no se reasigna con `call`, `apply`, `bind` ni con `new`. Por eso son ideales como callbacks (el `this` no se rompe al pasar la función), pero **malas como métodos de objeto**: ver ejemplo anterior. La tabla de la sección "cinco formas" resume la diferencia.

## Funciones dentro de objetos (métodos)

Cuando una función vive en una propiedad de un objeto, se llama **método**:

```js
const reproductor = {
  reproducir: function (id) {
    console.log(`Reproducir canción con el id ${id}`);
  },

  pausar() {
    console.log('Reproductor en pausa...');
  },

  agregarPlaylist(list) {
    console.log(`Agregando la playlist ${list}`);
  },
};

reproductor.reproducir(20);
reproductor.pausar();
reproductor.agregarPlaylist('Bachata');

// También se pueden agregar métodos después, como si fueran propiedades
reproductor.reproducirPlaylist = function (list) {
  console.log(`Reproducir la playlist ${list}`);
};

reproductor.borrar = (id) => {
  console.log(`Borrar canción con el id ${id}`); // cuidado: arrow + método = this heredado
};

reproductor.borrar(20);
reproductor.reproducirPlaylist('Vallenato');
```

Ver [[Objetos]] para el resto de operaciones con objetos.

## Arrow functions

La sintaxis moderna (ES6) más usada, especialmente como callbacks. Son más cortas y resuelven el problema del `this` (lo heredan del cierre):

```js
// Equivalentes: function expression vs arrow function
const aprendiendo = function () {
  console.log('Aprendiendo JavaScript');
};

const aprendiendo2 = () => console.log('Aprendiendo JavaScript');

// Con un paréntesis opcional si hay UNO solo
const aprendiendo3 = (tecnologia) => `Aprendiendo ${tecnologia}`;
console.log(aprendiendo3('JavaScript'));

const aprendiendo4 = tecnologia => `Aprendiendo ${tecnologia}`; // sin paréntesis

// Con dos o más parámetros SÍ se necesitan parárentesis
const aprendiendo5 = (tecnologia, tecnologia2) =>
  `Estamos aprendiendo ${tecnologia} y ${tecnologia2}`;
console.log(aprendiendo5('JavaScript', 'NextJS'));
```

> [!tip] Return implícito
> Si el cuerpo es **una sola expresión**, las llaves y el `return` son opcionales: `const doble = (n) => n * 2;`. Con varias líneas se usan llaves y `return` explícito. Detalle tramposo: `() => ({ x: 1 })` — las llaves del objeto obligan a parentesis, porque `{}` se interpreta como bloque.

### Qué NO se puede con una arrow function

| Característica | `function` normal | Arrow function |
|---|---|---|
| `new f()` como constructor | Sí (`TypeError` no) | **No** — lanza `TypeError: ... is not a constructor` (verificado) |
| `this` propio / `bind` | Sí | **No** |
| `arguments` | Sí | **No** (hereda el exterior) |
| `prototype` y `new.target` | Sí | **No** |
| Nombre en stack traces | Sí (la declara) | Solo el de la variable |

```js
try {
  new (() => {})();
} catch (e) {
  console.log(e.constructor.name); // TypeError
}
```

> [!info] ¿`typeof` y `instanceof`?
> `typeof (() => {})` es `"function"` igual que cualquier función — el runtime no distingue sintaxis. Donde sí se distinguen es en comportamiento (`new`, `this`, `arguments`).

## Arrow functions con `map` y `forEach`

Aquí las arrow functions brillan: los callbacks de [[Arreglos|los arreglos]] quedan en una sola línea — ver [[Iteradores]] y [[Array Methods]]:

```js
const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audífonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
];

// map: transforma y retorna un arreglo nuevo (return implícito)
const mensajes = carrito.map(
  (producto) => `${producto.nombre} con un precio de ${producto.precio}`
);

// forEach: ejecuta por cada elemento (no retorna nada)
carrito.forEach((producto) =>
  console.log(`${producto.nombre} tiene un precio de ${producto.precio}`)
);

console.log(mensajes);
```

Los parámetros del callback suelen ser `(valorActual, índice, arreglo)` — desclar el índice y el arreglo cuando no se usan: `carrito.map((p) => p.nombre)`, no `carrito.map((p, i, arr) => …)`.

## Higher-order functions (funciones de orden superior)

Una función es de orden superior si **recibe funciones como argumentos** y/o **devuelve una función**. Es el patrón que domina JavaScript moderno:

```js
// Recibe una función: el callback decide el comportamiento
function aplicarOperacion(a, b, operacion) {
  return operacion(a, b);
}
aplicarOperacion(10, 5, (x, y) => x + y); // 15
aplicarOperacion(10, 5, (x, y) => x * y); // 50

// Devuelve una función: "funciones que fabrican funciones"
function porFactor(factor) {
  return (numero) => numero * factor; // closure: recuerda factor
}
const doblar = porFactor(2);
const triple = porFactor(3);
doblar(5);  // 10
triple(5);  // 15
```

La receta de "devolver una función" es la base de los **middlewares**, del *currying* y del *partial application* — también del patrón de factory en [[Objetos]]. `map`, `filter`, `reduce`, `sort(cb)`, `setTimeout(cb)` y los handlers de [[Eventos]] son todos higher-order functions que ya usás a diario.

## IIFE: función que se ejecuta sola

Una *Immediately Invoked Function Expression* se define y ejecuta en el mismo instante. Su propósito histórico era crear un ámbito privado antes de que existieran `let`/`const`:

```js
const resultado = (function () {
  const secreto = 'no me veo desde afuera';
  return secreto;
})();
console.log(resultado); // "no me veo desde afuera"
// console.log(secreto); // ReferenceError

// Variante moderna: arrow IIFE
const otro = (() => 2 + 2)(); // 4
```

Hoy el 90% de los usos de IIFE se reemplaza con bloques `{ let ... }` o módulos ES (`import`/`export`). Se mantiene en librerías viejas para no contaminar el scope global (ver [[Sintaxis Antigua]]) y en algunos patrones de inicialización de módulos sin bundler.

## Funciones puras vs impuras

Una función **pura** cumple dos condiciones: (1) su resultado depende **solo** de sus argumentos (mismos argumentos → mismo resultado, sin importar cuándo ni cuántas veces se llame), y (2) no produce **efectos externos** (no muta estado global, no escribe en consola/red/DOM, no lanza efectos observables).

```js
// Pura
const iva = (precio, tasa = 0.15) => precio * (1 + tasa);

// Impura: depende de estado externo y lo muta
let total = 0;
const agregar = (precio) => (total += precio);

// Impura: mismo resultado "invisible", efecto en consola
const log = (x) => { console.log(x); return x; };
```

| Aspecto | Función pura | Función impura |
|---|---|---|
| Resultado | Determinista | Puede variar (estado, tiempo, azar) |
| Efectos | Ninguno | DOM, red, consola, variables globales |
| Testeabilidad | Trivial (assert puro) | Requiere mocks/entorno |
| Memoización | Segura (cache por argumentos) | Riesgosa |
| Ejemplos | `map`, `filter`, `reduce`, calculos | handlers de [[Eventos]], `fetch`, `prompt` |

> [!tip] La regla práctica
> No se trata de eliminar toda impureza: se trata de **aislar** las impuras (I/O, DOM, red) en el borde y mantener el núcleo de lógica puro. Es el mismo principio que hace que los componentes de [[ReactJS - Fundamentos]] sean fáciles de testear.

## Rendimiento: notas prácticas

- **Llamar a una función tiene costo mínimo** en JS moderno (call stack, no allocación pesada). No "inlines" manualmente por micro-optimización; la legibilidad gana.
- Lo que sí cuesta: **crear closures en bucles calientes** (cada closure aloja su entorno) y el patrón *function-in-loop* dentro de procesamiento de ~100k+ elementos. Para volúmenes grandes, bucles planos con índice (o `for...of`) superan a cadenas de `map/filter/reduce` intermedias — no porque `map` sea lento, sino porque cada paso crea un arreglo temporal (ver [[Array Methods]] → Rendimiento).
- **Recursión sin límite** (`function r(){ r() }`) revienta el stack (`RangeError: Maximum call stack size exceeded`, típicamente ~10k niveles). Para profundidades grandes, convertir a iteración o usar *tail-call* patterns manuales.
- `bind` crea una **nueva función** en cada llamada: en bucles calientes, preferir closures o `thisArg` explícito de `forEach`/`map`.
- Funciones **nombradas** mejoran los stack traces (el nombre aparece en errores) sin costo — nombrá funciones recurrentes en vez de asignar siempre anónimas.

## Errores comunes

1. **Llamar una function expression antes de definirla**: `TypeError: f is not a function` (con `var`: `undefined is not a function`; con `let/const`: `ReferenceError` por TDZ). Definí antes, o usá declaración.
2. **`return` con salto de línea tras `{`**: `() => { (objeto) }` o `function f() {\n objeto \n}` retorna `undefined` inesperadamente — la llave crea bloque.
3. **Usar arrow como método de objeto** y esperar `this`: `this` no es el objeto. Para métodos que necesitan `this`, usá `obj = { f() {} }` o `function`.
4. **`new` sobre una arrow**: `TypeError: ... is not a constructor`.
5. **Comparar funciones con `==` entre sí**: `f1 === f2` compara **referencias**, no comportamiento; dos funciones textualmente idénticas son distintas.
6. **Parámetros desestructurados sin default** llamados con `undefined`: `TypeError`. Usá `({ a } = {})`.
7. **Confundir `arguments` con array**: `arguments.map` no existe; convertí con `Array.from(arguments)` o usá rest (`...args`).
8. **Defaults que no disparan**: pasar `null` explícitamente **no** activa el default (solo `undefined` lo hace).

## Cheatsheet

| Concepto | Sintaxis | Nota clave |
|---|---|---|
| Declaración | `function f(a) {}` | Hoisting completo; anónima no |
| Expresión | `const f = function (a) {}` | Sin hoisting; TDZ con `let/const` |
| Arrow | `const f = (a) => a * 2` | Sin `this`, `arguments`, `new`, `prototype` |
| Método | `obj = { f() {} }` | `this` = objeto receptor |
| Default | `function f(a = 1) {}` | Solo con `undefined`; evaluado por llamada |
| Rest | `function f(...args) {}` | Arreglo real; siempre al final |
| Desestructurar params | `function f({ a, b } = {}) {}` | Patrón de entrada; default obligatorio |
| IIFE | `(function () {})()` / `(() => {})()` | Ámbito privado inmediato |
| Closure | función que captura su scope | Recuerda el entorno; cuidado con memoria |
| HOF | recibe o devuelve funciones | Base de callbacks y factories |
| Retorno | `return expr;` | Sin `return` → `undefined`; corta la ejecución |

## Relacionadas

- [[Sintaxis Moderna]] — donde viven las arrow functions y el resto del JS actual
- [[Estructuras de Control]] — las condiciones que las funciones suelen evaluar
- [[Iteradores]] — los bucles que las funciones recorren (y el patrón closure-en-bucle)
- [[Arreglos]] — callbacks con `map`, `forEach` y compañía
- [[Array Methods]] — `filter`, `reduce`... todo recibe funciones
- [[Objetos]] — métodos: funciones dentro de objetos
- [[Operadores]] — operador ternario y operadores lógicos que usan funciones como predicados
- [[Eventos]] — handlers: funciones pasadas al DOM
- [[ReactJS - Fundamentos]] — en React, los componentes son funciones
