---
tags:
  - desarrollo
  - frontend
  - iteradores
tipo: concepto
estado: terminado
---

# Iteradores

Un **iterador** (o bucle) repite un bloque de código varias veces. JavaScript ofrece varios estilos: el `for` clásico para control total, `while` cuando la condición manda, y `for...of`/`forEach` para recorrer [[Arreglos|arreglos]] con menos ruido. Todos se apoyan en las mismas primitivas de [[Estructuras de Control|control de flujo]] (`break`, `continue`, etiquetas) y en el **protocolo de iteración**, el contrato que permite recorrer *cualquier* cosa: arreglos, strings, `Map`, `Set`, nodos del DOM y generadores.

> [!info] El modelo mental correcto
> No existen "bucles para arreglos" ni "bucles para objetos": existe **un protocolo** (`Symbol.iterator`) y **dos sintaxis de recorrido** (`for...of` consume el protocolo; `for...in` recorre claves enumerables). `for`, `while` y `do...while` no iteran nada: solo repiten código con una condición. Elegir bien empieza por saber cuál de los dos mundos estás usando.

## El `for` clásico: anatomía completa

```js
for (inicio; condición; incremento) { ... }
```

Las tres cláusulas están **separadas por `;`** y **cualquiera puede quedar vacía**:

| Cláusula | Cuándo se ejecuta | Puede vaciarse |
| -------- | ----------------- | -------------- |
| `inicio` | una sola vez, al arrancar | Sí (el contador se declara antes) |
| `condición` | antes de **cada** vuelta; si da falsa, el bucle termina | Sí (bucle infinito: solo sale con `break`) |
| `incremento` | después de **cada** vuelta | Sí (si no, el estado no avanza: bucle infinito) |

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
> Declarar el contador con `let` lo encierra en el scope del bucle: no se filtra al resto del código y **cada vuelta recibe su propio `i`** (una *lexical binding* nueva). Esto es clave al capturar el valor en closures dentro del bucle — con `var` todas las closures verían el último valor.

### Cláusulas vacías y variantes

```js
// inicio fuera del for (útil para reutilizar el contador o en dos contadores)
let i;
for (i = 0; i < 5; i++) { ... }

// sin incremento: se avanza manualmente (p. ej. cuando el paso es complejo)
for (let i = 0; i < 10; ) {
  i += 2;               // el "incremento" vive dentro del cuerpo
}

// solo condición ≈ while
for (; condicionSigueFalsa(); ) { ... }

// bucle infinito con break — patrón de "procesar hasta agotar"
for (;;) {
  const item = cola.shift();
  if (!item) break;
  procesar(item);
}

// contadores descendentes: i-- es más rápido de leer cuando se cuenta hacia atrás
for (let i = lista.length - 1; i >= 0; i--) { ... }

// paso personalizado
for (let i = 0; i <= 100; i += 5) { ... }   // múltiplos de 5

// contador doble (coma como operador en cada cláusula)
for (let i = 0, j = 10; i < j; i++, j--) { ... }
```

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

> [!warning] Cachear `length` — casi nunca hace falta hoy
> En bucles antiguos se escribía `for (let i = 0, len = arr.length; i < len; i++)` para no releer `.length` en cada vuelta. Los motores modernos optimizan esto; peor aún, si cacheas `len` y **modificas el arreglo dentro del bucle**, te quedas con un valor viejo. Regla actual: lee `carrito.length` directamente; solo cachea si perfilaste y el bucle es el hotspot real.

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

Ambos funcionan igual en `while`, `do...while`, `for...in` y `for...of`.

### Etiquetas (`labels`): salir de bucles anidados

Una etiqueta es un identificador seguido de `:` delante de un bucle; `break etiqueta` termina **ese bucle y todos los contenedores** hasta él, y `continue etiqueta` salta a la siguiente vuelta del bucle etiquetado:

```js
outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    if (j === 1) continue outer;   // salta a la siguiente i (imprime solo j=0)
    if (i === 2) break outer;      // termina TODO, ambos bucles
    console.log(i, j);             // 0 0 / 1 0
  }
}
```

`break` sin etiqueta rompe solo el bucle más interno. Las etiquetas son el único modo de `break`/`continue` cruzando más de un nivel — úsalas con mesura (son difíciles de seguir en código largo); a veces un `return` o una función extra comunica mejor la intención.

> [!tip] Etiquetas y generadores
> En iteradores generadores, `outer:` sobre un `for...of` es el modo estándar de cortar dos niveles sin helpers. También funcionan sobre bloques etiquetados, no solo bucles: `bloque: { ... break bloque; }` sirve como "goto corto" para salir de un bloque con condición.

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

Casos de uso reales:

| Situación | Bucle |
| --------- | ----- |
| Repetir hasta que una condición externa cambie (hasta que una API responda, hasta que el usuario salga del `prompt`) | `while` |
| Mostrar un menú al menos una vez y repetir hasta que elija "salir" | `do...while` |
| Buscar coincidiendo con un criterio que no sabes si se cumple | `while` con índice creciente |

```js
// patrón de centinela: procesar hasta encontrar un valor especial
let entrada;
do {
  entrada = prompt("Ingresá un número (fin para salir)");
} while (entrada !== "fin");

// patrón de reintento con condición desconocida
let intento = 0;
while (!funciono && intento < MAX_INTENTOS) {
  funciono = intentarConexion();
  intento++;
}
```

> [!warning] Bucle infinito accidental
> El bug #1 de `while`: olvidar avanzar el estado (`i++`) dentro del cuerpo. A diferencia del `for`, aquí no hay cláusula de incremento que lo recuerde. Si el bucle puede no avanzar (p. ej. `while (!nodo.siguiente)` con un árbol mal formado), agregá un tope de vueltas de seguridad.

## `forEach`: recorrer arreglos con callback

`forEach` recibe una [[Funciones|función]] que se ejecuta por cada elemento. Con un segundo parámetro obtienes el índice, y con un tercero el arreglo completo:

```js
const pendientes = ['Tareas', 'Ejercicio', 'Programación', 'Trabajo'];

pendientes.forEach((pendiente, indice) => {
  console.log(`${pendiente} con índice ${indice}`);
});

// con el arreglo completo como tercer argumento
pendientes.forEach((pendiente, indice, arreglo) => {
  console.log(`${indice + 1}/${arreglo.length}: ${pendiente}`);
});
```

Semántica que convoce conocer:

| Comportamiento | Detalle |
| -------------- | ------- |
| Recorre **índices 0..length−1** | Los agujeros de arreglos *sparse* se **saltean** (verificado: `[1, , , 4].forEach` solo visita `1` y `4`) |
| No se puede cortar | `break`/`continue` no existen dentro; lanzar excepción tampoco "corta limpio" |
| Devuelve siempre `undefined` | No construye resultado; para transformar usá `map` |
| Segundo argumento opcional `thisArg` | Fija el `this` de la función (legacy; con arrow functions no aplica) |
| No muta el arreglo original | Pero el callback puede mutar los elementos |

> [!tip] Alternativas para "cortar" `forEach`
> `forEach` no admite `break`. Alternativas declarativas según la intención:
> - *¿Existe alguno?* → `arr.some((x) => cond(x))` (para en la primera coincidencia).
> - *¿Cuál es el primero?* → `arr.find((x) => cond(x))` o `findIndex` (retorna `undefined`/`-1`).
> - *¿Todos cumplen?* → `arr.every((x) => cond(x))`.
> - *¿Ninguno cumple?* → `!arr.some(...)`.
> Y si de verdad necesitás salir a mitad de una secuencia imperativa, cambiá a `for...of` con `break`.

## `for...of`: la forma moderna de recorrer

`for...of` itera sobre los **valores** de un **iterable** (arreglos, strings, `Map`, `Set`, nodos del DOM, generadores) sin índices ni callbacks:

```js
for (const pendiente of pendientes) {
  console.log(pendiente);
}

for (const producto of carrito) {
  console.log(producto.nombre, producto.precio);
}
```

### Qué es un iterable: el protocolo

Un objeto es iterable si expone un método `[Symbol.iterator]()` que devuelve un **iterador**: un objeto con `.next()` que retorna `{ value, done }`. `done: true` significa que la secuencia terminó:

```js
const it = [10, 20][Symbol.iterator]();
console.log(it.next()); // { value: 10, done: false }
console.log(it.next()); // { value: 20, done: false }
console.log(it.next()); // { value: undefined, done: true }
```

`for...of` simplemente llama a `[Symbol.iterator]()` y consume `next()` hasta `done`. Por eso funciona con todo lo que cumple el protocolo:

```js
// strings: itera code points, no code units UTF-16
for (const ch of "añb") console.log(ch); // "a", "ñ", "b" (un solo carácter)

// Map: en orden de inserción, cada [clave, valor]
const m = new Map([["b", 1], ["a", 2]]);
m.set("c", 3);
for (const [clave, valor] of m) console.log(clave, valor); // b 1 / a 2 / c 3

// Set: valores únicos, sin duplicados ni índices
for (const v of new Set(["x", "y", "x"])) console.log(v); // x, y

// NodeList del DOM (query NodeList es iterable directamente)
for (const li of document.querySelectorAll("li")) console.log(li.textContent);

// generadores: valores producidos "al vuelo"
function* fib() {
  let [a, b] = [0, 1];
  while (true) { yield a; [a, b] = [b, a + b]; }
}
const primeros = [];
for (const n of fib()) {          // secuencia infinita...
  primeros.push(n);
  if (primeros.length === 5) break; // ...cortada con break (verificado: [0,1,1,2,3])
}
```

> [!warning] Un objeto "array-like" NO es iterable
> `{ 0: 'a', 1: 'b', length: 2 }` tiene largo y claves numéricas, pero no tiene `[Symbol.iterator]`: `for (const v of arrayLike)` lanza `TypeError: arrayLike is not iterable` (verificado). Convierte primero: `[...arrayLike]` falla igual — usá `Array.from(arrayLike)`. Esto explica errores típicos al iterar `arguments` viejo o colecciones antiguas del DOM.

### `entries()`, `keys()`, `values()`: índices y claves

```js
// recorrer con índice SIN for clásico (entries devuelve [indice, valor])
for (const [indice, producto] of carrito.entries()) {
  console.log(`${indice}: ${producto.nombre}`);
}

// recorrer dos arreglos en paralelo por índice
for (const [a, b] of ['a', 'b'].map((v, i) => [v, nums[i]])) { ... } // mejor: zip manual o .map
```

## `for...in`: claves, no valores

`for...in` itera sobre las **propiedades enumerables** — incluidas las **heredadas** del prototipo. En objetos da las **claves** (como strings); en arreglos da los índices:

```js
// Con objeto: imprime las claves
const automovil = { modelo: 'Toyota', motor: 'Chevrolet', año: 2020 };

for (const propiedad in automovil) {
  console.log(propiedad);            // "modelo", "motor", "año"
  console.log(automovil[propiedad]); // los valores
}

// Con arreglo: imprime 0, 1, 2, 3 (los índices como strings, no los valores)
for (const indice in pendientes) {
  console.log(indice);
}
```

### Las tres reglas que explican por qué `for...in` está mal para arreglos

1. **Recorre índices, no valores.** Querías los productos del carrito y obtienes los números `0, 1, 2`.
2. **Salta los agujeros y también recoge propiedades agregadas.** Verificado: en `[1, , , 4]` recorre solo `'0'` y `'3'`; si alguien hizo `arr.miPropiedad = 'x'`, aparece en el recorrido junto a los índices.
3. **Incluye propiedades heredadas del prototipo.** Verificado: con `Object.create({heredado: 1})`, `for...in` imprime `propio` **e** `heredado`. El filtro clásico era `if (obj.hasOwnProperty(k))` — hoy `Object.hasOwn(obj, k)` (ver [[Objetos]]).

Orden de las claves: los índices enteros positivos van primero **en orden numérico ascendente**, después las claves de texto en orden de inserción, y al final las propiedades simbólicas (que `for...in` **no** recorre). Verificado: `{b, "10", a, "2", [Symbol]}` → `Object.keys` da `['1','2','10','b','a']`.

> [!warning] `for...in` no para arreglos
> Para **valores** de un arreglo usa siempre `for...of` o `forEach`. `for...in` sobre arreglos es un anti-patrón que solo sobrevive en código viejo — ver [Array Methods](https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Array) para las alternativas.

### Objeto con `for...of`: `Object.entries`

Para recorrer un objeto con `for...of`, conviértelo a pares clave/valor:

```js
for (const [clave, valor] of Object.entries(automovil)) {
  console.log(`${clave}: ${valor}`); // desestructuración del par
}

// equivalente con for...in + filtro de propias
for (const clave in automovil) {
  if (Object.hasOwn(automovil, clave)) console.log(clave, automovil[clave]);
}
```

`Object.entries` solo incluye propias y enumerables (sin heredadas) — ver [[Objetos]] para el catálogo completo.

## Rendimiento: cómo comparan los bucles

Mediciones variadas entre motores dan el mismo **orden relativo** estable (V8/JSC/SpiderMonkey):

| Bucle | Velocidad relativa | Notas |
| ----- | ------------------ | ----- |
| `for` clásico con índice | La más rápida (referencia) | El JIT lo convierte en código tan rápido como un `while` |
| `while` / `do...while` | ≈ igual que `for` | Diferencia solo perceptible en hotspots |
| `for...of` sobre arreglo | Muy cerca del `for` | Modernamente casi tan rápido; iteradores nativos están optimizados |
| `forEach` + callback | ~1.3–2× más lento que `for` | El costo es la llamada a función por elemento |
| `map` / `filter` / `reduce` | Igual que `forEach` + crea arreglo nuevo | La asignación del resultado añade GC pressure |
| `for...in` | **10×–100× más lento** | Construye la lista de claves en cada vuelta |

> [!tip] La regla práctica
> Elegí por **claridad**, no por velocidad: en la inmensa mayoría del código la diferencia es imperceptible. Optimizá solo cuando un perfillo señale el bucle como hotspot, y entonces la mejora real casi siempre viene de **no recorrer** (precalcular, indexar en un `Map`) antes que de cambiar `forEach` por `for`.

## Tabla de decisión: qué bucle usar

| Situación | Recomendado |
| --------- | ----------- |
| Recorrer arreglo solo con valores | `for...of` |
| Recorrer arreglo con índice | `for...of` + `.entries()` o `forEach((x, i) => ...)` |
| Necesitás `break`/`continue` | `for...of` o `for` clásico |
| Recorrer objeto (claves) | `for...in` + `Object.hasOwn`, o `Object.keys/entries` + `for...of` |
| Contar mientras haya condición | `while` / `do...while` |
| Transformar/filtrar arreglos | `map`, `filter`, `reduce` — ver [[Array Methods]] |
| Secuencia infinita o perezosa (generador) | `for...of` con `break` |
| Recorrer `Map`/`Set` | `for...of` (orden de inserción garantizado) |
| Colección del DOM (`HTMLCollection`) | `for (const el of [...coleccion])` o `querySelectorAll` |

## Iterar colecciones del DOM

Las colecciones viejas del DOM (`getElementsByClassName`, `getElementsByTagName`, `document.links`…) devuelven una `HTMLCollection` **viva**. Iterarlas bien:

```js
// NodeList de querySelectorAll: iterable directamente con for...of
for (const li of document.querySelectorAll('.menu li')) console.log(li);

// HTMLCollection: NO es iterable, hay que convertir
const enlaces = document.getElementsByClassName('enlace'); // HTMLCollection viva
for (const el of [...enlaces]) console.log(el);            // copia estática al instanciarse

// conversión en arreglo para map/filter (métodos de arreglo no existen en colecciones)
const visibles = [...enlaces].filter((el) => el.offsetParent !== null);
```

> [!warning] Copiar antes de recorrer y mutar
> Si dentro del bucle **eliminás nodos**, convertí primero a array (`[...coleccion]`): al ser vivas, la `HTMLCollection` cambia durante el recorrido y los índices se desplazan, saltándose elementos. Detalle completo en [[Acceder a Html]].

## Recetas prácticas

### FizzBuzz (clásico de entrevistas)

Múltiplos de 3 imprimen *Fizz*, de 5 *Buzz*, y de ambos *FizzBuzz*:

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

### Buscar el primer elemento que cumple (sin `forEach`)

```js
const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audífonos', precio: 50, descuento: true },
  { nombre: 'Teclado', precio: 100, descuento: true },
];

const enDescuento = carrito.find((p) => p.descuento);
const hayCaro = carrito.some((p) => p.precio > 250);
const todosBaratos = carrito.every((p) => p.precio < 500);
console.log(enDescuento?.nombre, hayCaro, todosBaratos); // Audífonos true true
```

### Recorrer un objeto como si fuera una lista

```js
const inventario = { manzanas: 12, peras: 5, uvas: 30 };

for (const [fruta, cantidad] of Object.entries(inventario)) {
  console.log(`${fruta}: ${cantidad}`);
}

// versión funcional con map
const etiquetas = Object.entries(inventario).map(([f, c]) => `${f} (${c})`);
```

### Generadores para secuencias perezosas

```js
function* range(desde, hasta, paso = 1) {
  for (let i = desde; i <= hasta; i += paso) yield i;
}

// consumible con for...of, con map(...), con [...range(...)]
for (const n of range(1, 10, 2)) console.log(n); // 1, 3, 5, 7, 9

function* infinite() {
  let i = 0;
  while (true) yield i++;
}
const pares = [...infinite()].slice(0, 5); // ⚠️ corta mal: infinite() no termina
// correcto: cortar con take() o break dentro de for...of
```

### Pausa entre iteraciones con `await` (async/await en bucles)

```js
for (const url of urls) {
  const resp = await fetch(url);          // espera cada respuesta en serie
  procesar(await resp.json());
}
// en paralelo: const datos = await Promise.all(urls.map((u) => fetch(u).then((r) => r.json())));
```

`await` solo es legal dentro de `async` funciones; el `for...of` es la sintaxis preferida porque respeta el orden de forma natural.

## Errores comunes

1. **Bucle infinito por olvidar el incremento** (`while` sin avanzar el estado, o `for` con cláusula de incremento vacía y sin `i++` en el cuerpo). Síntoma: el navegador se congela. Fix: garantizar que toda vuelta acerca la condición al fin.
2. **`for...of` sobre un objeto plano**: `TypeError: obj is not iterable` (verificado). Los objetos no son iterables; usá `Object.entries` o `for...in`.
3. **Esperar `break` dentro de `forEach`**: es un `SyntaxError` — `break` solo existe dentro de bucles, no de callbacks. Usá `some`/`find`/`every` o cambiá a `for...of`.
4. **Confundir `for...in` con `for...of`** en arreglos: `for...in` da índices *como strings* y recoge propiedades extra y heredadas; `for...of` da valores. Es el anti-patrón más clásico.
5. **Comparar con `==` dentro de la condición**: `while (i != "10")` puede no terminar nunca por coerción (ver [[Operadores]]). Usá `===`.
6. **Modificar la longitud mientras se itera por índice**: `for (let i = 0; i < arr.length; i++)` con `arr.pop()` dentro salta elementos; con `push` puede repetirlos. Iterá de atrás adelante o recorre una copia.
7. **Iterar `HTMLCollection` viva y mutar el DOM dentro**: los índices se desplazan y te saltás nodos — convertí a array primero (ver [[Acceder a Html]]).
8. **Esperar que `forEach`/`map` recorran agujeros**: en arreglos sparse, `forEach`, `map` y `filter` **saltean** los huecos, pero `for...of` los visita como `undefined`. Verificado: `[1, , , 4]` → `forEach` ve 2 elementos, `for...of` ve 4.

## Cheatsheet

| Constructo | Sintaxis | Itera | Cortable con `break` |
| ---------- | -------- | ----- | -------------------- |
| `for` | `for (init; cond; inc) {}` | nada (solo repite) | Sí |
| `while` | `while (cond) {}` | nada (solo repite) | Sí |
| `do...while` | `do {} while (cond);` | nada (mínimo 1 vuelta) | Sí |
| `for...of` | `for (const v of iterable) {}` | **valores** de cualquier iterable | Sí |
| `for...in` | `for (const k in obj) {}` | **claves** enumerables (propias + heredadas) | Sí |
| `forEach` | `arr.forEach((v, i, a) => {})` | valores de un arreglo | No |
| Etiqueta | `etq: for (...) { break etq; }` | — (controla bucles anidados) | Salta niveles |
| Generador | `function* () { yield x; }` | valores producidos perezosamente | Sí (vía `for...of`) |

## Relacionadas

- [[Arreglos]] — la estructura que más se itera
- [[Array Methods]] — alternativas declarativas: `map`, `filter`, `reduce`, `find`, `some`, `every`
- [[Funciones]] — callbacks de `forEach` y arrow functions
- [[Objetos]] — `for...in`, `Object.entries` y el orden de las claves
- [[Estructuras de Control]] — `if`, `switch` y `break`/`continue` dentro del bucle
- [[Operadores]] — `===`, módulo `%` y optional chaining dentro de condiciones
- [[Sintaxis Moderna]] — destructuring (`const [a, b] of ...`), spread y generadores
- [[Document Object Model (DOM)]] — las colecciones de nodos que se itera en la práctica
- [[Acceder a Html]] — `querySelectorAll` vs colecciones vivas
