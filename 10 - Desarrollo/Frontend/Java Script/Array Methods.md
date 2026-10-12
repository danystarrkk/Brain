---
tags:
  - desarrollo
  - frontend
  - array-methods
tipo: concepto
estado: terminado
---

# Array Methods

Los **métodos de arreglo** son funciones que ya vienen con cada arreglo y resuelven las operaciones más comunes: buscar, filtrar, transformar y combinar. Casi todos reciben una función callback — ver [[Funciones]] — y se usan con arrow functions para mantener el código corto. Esta nota es la **referencia completa**: ~45 de métodos de instancia + estáticos, con su semántica exacta, sus traps verificados y las recetas que se usan en producción.

> [!info] El modelo mental: mutadores vs transformadores
> Los métodos se dividen en dos familias. **Mutadores** (`push`, `sort`, `splice`…) modifican el arreglo original y suelen retornar length/elementos/referencia. **Transformadores** (`map`, `filter`, `slice`, `toSorted`…) devuelven un **arreglo nuevo** y dejan el original intacto. Elegir bien es la base de la inmutabilidad que usa [[ReactJS - Fundamentos]].

Datos de ejemplo para toda la nota:

```js
const meses = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio', 'Julio'];

const carrito = [
  { nombre: 'Monitor 27 Pulgadas', precio: 500 },
  { nombre: 'Televisión', precio: 100 },
  { nombre: 'Tablet', precio: 200 },
  { nombre: 'Audífonos', precio: 300 },
  { nombre: 'Teclado', precio: 400 },
  { nombre: 'Celular', precio: 700 },
];
```

## Mapa maestro: todos los métodos en una tabla

| Método | ¿Muta? | Retorna | Categoría |
|---|---|---|---|
| `push(...elems)` | ✅ | nueva `length` | mutador |
| `pop()` | ✅ | elemento quitado / `undefined` | mutador |
| `shift()` | ✅ | elemento quitado / `undefined` | mutador |
| `unshift(...elems)` | ✅ | nueva `length` | mutador |
| `splice(inicio, borra, ...elems)` | ✅ | arreglo de lo quitado | mutador |
| `sort(cb?)` | ✅ | **la misma referencia** | mutador |
| `reverse()` | ✅ | **la misma referencia** | mutador |
| `fill(valor, d, h)` | ✅ | la misma referencia | mutador |
| `copyWithin(d, f, h)` | ✅ | la misma referencia | mutador |
| `indexOf(v, d)` | ❌ | índice o `-1` | búsqueda |
| `lastIndexOf(v, d)` | ❌ | índice del último o `-1` | búsqueda |
| `includes(v, d)` | ❌ | `boolean` | búsqueda |
| `find(cb, thisArg)` | ❌ | elemento o `undefined` | búsqueda |
| `findIndex(cb, thisArg)` | ❌ | índice o `-1` | búsqueda |
| `findLast(cb, thisArg)` | ❌ | elemento o `undefined` | búsqueda (ES2023) |
| `findLastIndex(cb, thisArg)` | ❌ | índice o `-1` | búsqueda (ES2023) |
| `at(i)` | ❌ | elemento / `undefined` | acceso |
| `map(cb)` | ❌ | arreglo nuevo (mismo largo) | transformación |
| `filter(cb)` | ❌ | arreglo nuevo (⊆ original) | transformación |
| `flat(d)` | ❌ | arreglo nuevo aplanado | transformación |
| `flatMap(cb)` | ❌ | `map` + `flat(1)` | transformación |
| `reduce(cb, init)` | ❌ | **cualquier valor** | acumulación |
| `reduceRight(cb, init)` | ❌ | cualquier valor | acumulación |
| `forEach(cb, thisArg)` | ❌ | `undefined` | iteración |
| `some(cb)` | ❌ | `boolean` | predicado |
| `every(cb)` | ❌ | `boolean` | predicado |
| `keys()` / `values()` / `entries()` | ❌ | **iterador** (no arreglo) | iteración |
| `concat(...v)` | ❌ | arreglo nuevo | combinación |
| `slice(d, h)` | ❌ | subarreglo nuevo | combinación |
| `toSorted(cb)` | ❌ | arreglo ordenado nuevo | moderno (ES2023) |
| `toReversed()` | ❌ | arreglo invertido nuevo | moderno (ES2023) |
| `toSpliced(i, b, ...e)` | ❌ | copia con splice | moderno (ES2023) |
| `join(sep)` | ❌ | **string** | puente strings |
| `Array.from(iter, map)` | — | arreglo nuevo | estático |
| `Array.fromRight(iter, map)` | — | arreglo nuevo (de derecha) | estático (propuesta; verificar soporte) |
| `Array.of(...v)` | — | arreglo de argumentos | estático |
| `Array.isArray(x)` | — | `boolean` | estático |
| `Object.groupBy(iter, cb)` | — | objeto de grupos | estático (ES2024) |
| `Map.groupBy(iter, cb)` | — | `Map` de grupos | estático (ES2024) |

## Mutadores (modifican el arreglo original)

```js
const meses2 = ['Enero', 'Febrero', 'Marzo'];

meses2.push('Abril');        // al final → length 4
meses2.unshift('Diciembre'); // al inicio → length 5
const ultimo = meses2.pop();     // "Abril"
const primero = meses2.shift();  // "Diciembre"

// splice es el comodín: (índice inicial, cuántos borrar, ...a insertar)
const quitados = carrito.splice(1, 2);              // borra 2 desde el índice 1
carrito.splice(1, 0, { nombre: 'Mouse', precio: 50 }); // inserta sin borrar
carrito.splice(1, 1, { nombre: 'Dock', precio: 90 });  // reemplaza 1 por 1
```

| Método | Firma | Retorna | Costo | Notas |
|---|---|---|---|---|
| `push` | `(...elems)` | nueva length | O(1) | el más rápido; acepta varios |
| `pop` | `()` | último / `undefined` | O(1) | vacío → `undefined` |
| `unshift` | `(...elems)` | nueva length | O(n) | mueve todos los índices |
| `shift` | `()` | primero / `undefined` | O(n) | idem — malo en colas grandes |
| `splice` | `(inicio, borrar, ...elems)` | lo quitado | O(n) | borrar 0 = insertar; borrar sin elems = cortar |

### `sort`: el mutador con la trampa por defecto

`sort` ordena **in situ** y devuelve la misma referencia. Sin comparador, convierte cada elemento a **string** y ordena por código de carácter — no numéricamente (verificado):

```js
console.log([10, 9, 100].sort());        // [10, 100, 9] ← orden lexicográfico: "10" < "100" < "9"
console.log(['b', 'A', 'a'].sort());     // ['A', 'a', 'b'] ← mayúsculas antes (código menor)
console.log(meses.sort());               // alfabético español SIN ñ/acentos: orden por code point

// Correcto: comparador numérico
console.log([10, 9, 100].sort((a, b) => a - b)); // [9, 10, 100] — a-b ascendente
console.log([10, 9, 100].sort((a, b) => b - a)); // descendente

// Con objetos: comparar por propiedad
const porPrecio = [...carrito].sort((a, b) => a.precio - b.precio);

// Con texto: localeCompare para orden humano (ver [[Strings]])
const ordenLocal = [...meses].sort((a, b) => a.localeCompare(b, 'es'));
```

> [!warning] `sort` muta **y** devuelve el original
> `const ordenado = arr.sort()` no copia: `ordenado === arr` es `true`. Si el original no se puede tocar (estado de UI, datos compartidos), la alternativa moderna es `toSorted` (ver abajo) o `[...arr].sort(cb)`. Y el comparador por defecto es el bug silencioso #1 de JavaScript: **siempre** pasá comparador para números.

> [!info] Semántica del comparador
> El comparador recibe `(a, b)` y debe retornar: **negativo** si `a` va antes, **positivo** si va después, **0** si son equivalentes. La fórmula `a - b` funciona con números; con strings, `localeCompare`. El orden final **no** depende del algoritmo de sort del motor (estable desde ES2019 en todos).

## Búsqueda: encontrar valores e índices

```js
// includes: comparación por valor, ideal para arreglos simples de primitivos
console.log(meses.includes('Enero')); // true

// some: "¿ALGÚN elemento cumple la condición?" — perfecto con objetos
const existe = carrito.some((producto) => producto.nombre === 'Celular'); // true

// some también sirve con primitivos
console.log(meses.some((mes) => mes === 'Mayo')); // true

// every: "¿TODOS los elementos cumplen la condición?"
const todosBaratos = carrito.every((producto) => producto.precio < 1000); // true
```

| Método | Pregunta que resuelve | Retorna | Comparación |
| ------ | --------------------- | ------- | ----------- |
| `includes(valor)` | ¿Este valor está en el arreglo? | `boolean` | `SameValueZero` (¡encuentra `NaN`!) |
| `indexOf(valor, d)` | ¿En qué índice está? | índice o `-1` | `===` (NaN nunca coincide) |
| `lastIndexOf(valor, d)` | ¿En el último? | índice o `-1` | `===` |
| `some(cb)` | ¿Algún elemento cumple? | `boolean` | lo decida el callback |
| `every(cb)` | ¿Todos cumplen? | `boolean` | lo decida el callback |
| `find(cb)` | Dame el PRIMERO que cumpla | elemento / `undefined` | callback |
| `findIndex(cb)` | ¿Qué índice cumple primero? | índice / `-1` | callback |
| `findLast(cb)` | El ÚLTIMO que cumpla (ES2023) | elemento / `undefined` | callback |
| `findLastIndex(cb)` | Último índice que cumple (ES2023) | índice / `-1` | callback |

Verificado — la diferencia `indexOf` vs `includes` con NaN (no usan el mismo algoritmo de igualdad):

```js
console.log([NaN].indexOf(NaN));   // -1 ← indexOf usa === y NaN !== NaN (ver [[Números]])
console.log([NaN].includes(NaN));  // true ← includes usa SameValueZero, que sí iguala NaN
```

```js
// Con forEach hay que guardar el índice a mano
meses.forEach((mes, i) => {
  if (mes === 'Mayo') console.log(i);
});

// findIndex es directo; retorna -1 si no encuentra
const indice = meses.findIndex((mes) => mes === 'Abril');
console.log(indice !== -1 ? `El índice de Abril es ${indice}` : 'Mes no encontrado');

const indiceCelular = carrito.findIndex((p) => p.nombre === 'Celular');
console.log(indiceCelular !== -1 ? `El índice de Celular es ${indiceCelular}` : 'Valor no encontrado');
```

> [!warning] El -1 rompe las comparaciones ingenuas
> Si el elemento está en la **posición 0**, una comprobación `indice > 0` daría `false` y reportaría "no encontrado" aunque sí exista. Compara siempre contra `-1`: `indice !== -1`.

### `find` vs `filter` vs `findIndex`: cuál cuándo

```js
// find retorna el PRIMER elemento que cumple la condición (o undefined)
const resultado = carrito.find((producto) => producto.nombre === 'Tablet');
console.log(resultado); // { nombre: 'Tablet', precio: 200 }

// Lo mismo con forEach obliga a esto (innecesariamente verboso):
let encontrado = null;
carrito.forEach((producto, indice) => {
  if (producto.nombre === 'Tablet') encontrado = carrito[indice];
});
```

| Método | Retorna | Caso de uso |
| ------ | ------- | ----------- |
| `find(cb)` | el elemento o `undefined` | obtener UN objeto por su propiedad |
| `findIndex(cb)` | índice o `-1` | necesito la posición para modificar/eliminar |
| `filter(cb)` | arreglo (posiblemente vacío) | necesito TODOS los que cumplan |

`find` corta en el primer match (más rápido que `filter` si solo querés uno). `findLast`/`findLastIndex` resuelven "el último que cumpla" sin recorrer al revés a mano — útil con timestamps: ` eventos.findLast(e => e.ts > umbral)`.

## Transformación: `map`, `filter`, `flat`, `flatMap`

```js
// filter crea un ARREGLO NUEVO con los elementos que cumplen la condición
console.log(carrito.filter((p) => p.precio > 400));  // Monitor y Celular
console.log(carrito.filter((p) => p.precio < 600));  // todo menos Celular

// Muy útil para "quitar" algo de un carrito de compras
const sinAudifonos = carrito.filter((p) => p.nombre !== 'Audífonos');
```

```js
// map: transforma cada elemento a un arreglo nuevo del MISMO largo
const nombres = carrito.map((p) => p.nombre);
const conDescuento = carrito.map((p) => ({ ...p, precio: p.precio * 0.9 })); // copia nueva por objeto

// flat: aplana N niveles de arreglos anidados
console.log([1, [2, [3]]].flat(Infinity)); // [1, 2, 3] (verificado) — el nivel por defecto es 1

// flatMap: map + aplanar UN nivel (más rápido que map+flat y no aplana más)
console.log([1, [2, 3]].flatMap((x) => (Array.isArray(x) ? x : [x * 2]))); // [2, 2, 3] (verificado)
```

> [!tip] `flatMap` no "aplana todo"
> Solo nivela 1. Ejemplo verificado:
>
> ```js
> const conNido = [[1, [2]]];
> conNido.flatMap((x) => x); // [1, [2]] — el [2] interno NO se aplana
> ```
>
> Para profundidad arbitraria, `flat(Infinity)` (o `flat(99)`). Y ojo: `flatMap` **no** aplana el resultado si devolvés un arreglo anidado dentro de otro: devolvé elementos sueltos o aceptá un nivel.

> [!warning] `map` con huecos y con objetos
> `map` **salta los huecos** del arreglo (ver [[Arreglos]]): `[1,,3].map(x => x * 2)` → `[2, <hueco>, 6]`. Y en objetos, `map(p => p.precio * 0.9)` NO muta los objetos — crea solo un arreglo de números; si necesitás copias del objeto, `{ ...p, precio }`.

## Acumulación: `reduce` y `reduceRight`

```js
// Imperativo: declarar un acumulador y mutarlo
let total = 0;
carrito.forEach((producto) => (total += producto.precio));
console.log(total); // 2200

// Declarativo: reduce acumula SIN variable externa
// (acumulador, valor actual) => nuevo acumulador; el 0 es el valor inicial
const totalReduce = carrito.reduce((acumulador, producto) => acumulador + producto.precio, 0);
console.log(totalReduce); // 2200
```

La firma completa del callback: `(acum, valorActual, índice, arreglo) => nuevoAcum`. `reduceRight` es idéntico pero itera de derecha a izquierda.

> [!warning] El valor inicial de `reduce` — tres trampas verificadas
> 1. **Olvidarlo con objetos**: el acumulador arranca como el **primer elemento**, no `{}`: `[{v:1},{v:2}].reduce((acc,o)=>acc+o.v)` → `"[object Object]2"` (verificado: suma `objeto + 2` = concatenación).
> 2. **Arreglo vacío sin valor inicial**: lanza `TypeError: Reduce of empty array with no initial value` (verificado). Con valor inicial, retorna ese valor: `[].reduce((a,b)=>a+b, 0)` → `0`.
> 3. **Numerar cuándo el acumulador puede ser falsy**: `arr.reduce((a, b) => a + b)` sin init en `[0, 0]` funciona, pero el patrón seguro es siempre dar init — documenta el tipo de retorno del acumulador de entrada.

Receta: `reduce` como **cualquier estructura** (no solo números) — agrupar, contar, indexar:

```js
// Contar frecuencias
const frecuencia = carrito.reduce((acc, p) => {
  acc[p.nombre] = (acc[p.nombre] ?? 0) + 1;
  return acc;
}, {});

// Convertir arreglo → objeto (índice por id)
const porNombre = carrito.reduce((acc, p) => ({ ...acc, [p.nombre]: p }), {});
```

## Iteración: `forEach`, `some`, `every`, `at`, iteradores

```js
// forEach: ejecuta por cada elemento; retorna undefined (no es para transformar)
carrito.forEach((p, i) => console.log(`${i}: ${p.nombre} — ${p.precio}`));

// at: acceso con índice negativo (verificado)
console.log(carrito.at(-1));    // { nombre: 'Celular', precio: 700 }
console.log(carrito.at(99));    // undefined

// keys/values/entries: ITERADORES (no arreglos) — sirven en for...of
for (const [i, p] of carrito.entries()) console.log(i, p.nombre);
console.log([...meses.keys()]);    // [0, 1, 2, ...] índices
console.log([...carrito.values()]); // los elementos
```

| Método | Para qué | Nota |
|---|---|---|
| `forEach(cb)` | efectos secundarios por elemento | sin `break`; retorno ignorado |
| `some(cb)` | cortocircuito: para en el primer `true` | "¿existe alguno?" |
| `every(cb)` | cortocircuito: para en el primer `false` | "¿cumplen todos?" |
| `at(i)` | acceso directo con negativos | reemplaza `arr[arr.length + i]` |
| `keys()` | iterador de índices | funciona con huecos |
| `values()` | iterador de valores | |
| `entries()` | iterador `[índice, valor]` | el más versátil en `for...of` |

`some`/`every` son **perezosos**: no recorren todo el arreglo si la respuesta ya está decidida — mejor que `filter(...).length > 0` para preguntas de existencia.

## Creación y estáticos: `from`, `of`, `isArray`, `groupBy`

```js
console.log(Array.from('abc'));              // ['a','b','c'] — desde cualquier iterable
console.log(Array.from({ length: 3 }, (_, i) => i * 2)); // [0, 2, 4] — generar en el acto
console.log(Array.from('abc', (c) => c.toUpperCase())); // ['A','B','C'] — map en el acto
console.log(Array(3));                       // [ <3 empty items> ] ← TRAMPA: huecos
console.log(Array.of(3));                    // [ 3 ] ← siempre valores
console.log(Array.isArray([]));              // true — la única comprobación robusta
console.log(Array.isArray({ length: 0 }));   // false — un array-like NO es array

// groupBy (ES2024): agrupa por el retorno del callback (verificado)
console.log(Object.groupBy([1, 2, 3, 4], (n) => (n % 2 ? 'impar' : 'par')));
// { impar: [1, 3], par: [2, 4] } — antes: reduce a mano o lodash groupBy
```

- `Array.from` acepta **iterables** (string, Map, Set, generadores — ver [[Iteradores]]) y **array-like** (`{ length, 0… }`, `arguments`, NodeList del [[Document Object Model (DOM)|DOM]]): `Array.from(document.querySelectorAll('li'))` convierte la NodeList a arreglo para usar `map`/`filter`.
- `Array.fromRight` (propuesta Stage 3) haría lo mismo iterando desde el final; **aún no está en todos los runtimes** (no existe ni en Node 24) — donde haga falta, usar `[...iter].reverse()` o `Array.from(iter).reverse()`.
- `Object.groupBy` devuelve objeto (claves string); `Map.groupBy` devuelve `Map` con claves de cualquier tipo.

> [!warning] Los huecos y `map`
> `Array.from` y los literales crean arreglos **densos**; los huecos solo existen si se declaran a mano o con `new Array(n)`. `map`/`forEach` los saltan (ver [[Arreglos]] → "Denso vs disperso").

## Combinación: `concat`, `slice` y los `to*` modernos

```js
const meses3 = ['Agosto', 'Septiembre'];
const meses4 = ['Octubre', 'Noviembre', 'Diciembre'];

// concat une arreglos (y acepta valores sueltos) — crea nuevo, no muta
const combinado = meses.concat(meses3, meses4, 'OtroMes');

// Spread: la forma moderna y más legible
const combinado2 = [...meses, ...meses3, ...meses4, 'OtroMes'];

// slice: subarreglo [desde, hasta) — copia superficial, el original intacto
const copia = meses.slice();          // copia completa
const tresAMenos = carrito.slice(1, 3); // índices 1 y 2
```

### La familia inmutable ES2023: `toSorted`, `toReversed`, `toSpliced`

Mismas operaciones que los mutadores, pero **sobre una copia** — nacidas exactamente para el código inmutable (estado de [[ReactJS - Fundamentos]], datos compartidos):

```js
const original = [3, 1, 2];
console.log(original.toSorted());      // [1, 2, 3] (verificado)
console.log(original);                 // [3, 1, 2] — intacto (verificado)
console.log([1, 2, 3].toReversed());   // [3, 2, 1]; el original no cambia
console.log([1, 2, 3].toSpliced(1, 1)); // [1, 3]; [1,2,3].toSpliced(1, 1, 9, 8) → [1, 9, 8, 3]
```

| Antes (muta) | Ahora (inmutable) | Nota |
|---|---|---|
| `arr.sort(cb)` | `arr.toSorted(cb)` | mismos semántica de comparador |
| `arr.reverse()` | `arr.toReversed()` | |
| `arr.splice(i, b, ...e)` | `arr.toSpliced(i, b, ...e)` | retorna la copia modificada |
| `[...arr].sort()` | `arr.toSorted()` | el spread-spread ya no hace falta |

`with` (ES2025) completa la familia: `arr.with(0, 'x')` retorna copia con el índice 0 reemplazado — la versión inmutable de `arr[0] = 'x'`.

> [!warning] `concat` vs spread con strings
> `[...meses, 'OtroMes']` agrega `'OtroMes'` como **un** elemento, pero `...'OtroMes'` sin comillas (o spreadyando un string directamente) separa el texto **en cada letra**. Spreadyando un objeto plano da error: los objetos no son iterables.

## Puente con strings: `join`

```js
console.log(['Enero', 'Febrero'].join(', '));  // "Enero, Febrero"
console.log([1, 2, 3].join(''));               // "123"
console.log([1, , 3].join('-'));               // "1--3" — los huecos → vacío
console.log(meses.join(undefined));            // separador por defecto: ","
```

`join` es el inverso de `String.prototype.split` — ver [[Strings]]. Nota: `join` **sí** recorre los huecos (los vuelve string vacío), a diferencia de `map`/`forEach`.

## Errores y trampas: la sección que evita los bugs

1. **Mutar dentro de `forEach`/`map` sobre el mismo arreglo**: `arr.forEach(x => arr.shift())` salta elementos porque cambia `length` durante la iteración. Para transformar, `map`/`filter` (arreglos nuevos); para quitar, `filter` o `splice` en bucle inverso.
2. **`sort` sin comparador**: orden numérico incorrecto verificado (`[10, 9, 100].sort()` → `[10, 100, 9]`). Comparador siempre: `(a, b) => a - b` o `localeCompare`.
3. **`sort`/`reverse` mutando estado compartido**: devuelven el mismo arreglo. `toSorted`/`toReversed` o copia con spread.
4. **`indexOf` con objetos**: compara **referencias**, `[{} ]indexOf({})` → `-1`. Usar `findIndex(p => p.id === x)`.
5. **`indexOf` con NaN**: `-1` eterno (usa `===`). `includes(NaN)` → `true` (verificado). Para objetos y NaN juntos: `find`.
6. **`reduce` sin valor inicial**: con objetos da `"[object Object]…"`; con arreglo vacío, `TypeError` (verificado). Init siempre.
7. **`map` sobre huecos** o esperando efectos: `map` no toca huecos y su retorno se descarta fácil — `forEach` para efectos, `map` para transformar.
8. **Comparar el resultado de `map`/`filter` con `==`**: compara referencias — `[1] == [1]` es `false`. Contenido: `JSON.stringify` (primitivos simples) o comparación elemento a elemento.
9. **`some`/`every` con lógica invertida**: `every(p => p.precio > 100)` con un solo `false` corta — revisar que la condición sea la pregunta correcta ("¿hay alguno caro?" → `some`).
10. **Usar `Array.from` en lo que ya es arreglo**: funciona (copia), pero `arr.slice()` o `[...arr]` es más claro.

## Recetas prácticas

```js
// 1. Únicos (primitivos) — Set elimina duplicados con SameValueZero
const unicos = [...new Set([1, 2, 2, 3, 3, 3])]; // [1, 2, 3]
// Únicos por propiedad de objeto
const vistos = new Set();
const unicosObj = carrito.filter((p) => (vistos.has(p.nombre) ? false : (vistos.add(p.nombre), true)));

// 2. Chunk (partir en grupos de n) — para paginar o rate-limits
const chunk = (arr, n) => Array.from({ length: Math.ceil(arr.length / n) }, (_, i) =>
  arr.slice(i * n, i * n + n));
chunk([1, 2, 3, 4, 5], 2); // [[1,2],[3,4],[5]]

// 3. Aplanar profundamente (sin stack overflow de recursión manual)
const aplanar = (arr) => arr.flat(Infinity);

// 4. Pipeline con reduce — encadenar transformaciones declarativamente
const pipeline = (datos, ...fns) => fns.reduce((acc, fn) => fn(acc), datos);
pipeline(carrito,
  (c) => c.filter((p) => p.precio >= 200),
  (c) => c.map((p) => p.nombre));
// ['Monitor 27 Pulgadas', 'Tablet', 'Teclado', 'Celular']

// 5. Conteo de frecuencia (reduce → objeto)
const frecuenciaPalabras = (texto) => texto.split(' ').reduce((acc, w) => {
  acc[w] = (acc[w] ?? 0) + 1;
  return acc;
}, {});

// 6. Agrupar con groupBy nativo (ES2024) — el de antes, sin dependencias
const porRango = Object.groupBy(carrito, (p) =>
  p.precio >= 500 ? 'caro' : 'barato');

// 7. Máximo por propiedad — max con reduce (sin Math.max(...spread) que explota con 100k+)
const masCaro = carrito.reduce((max, p) => (p.precio > max.precio ? p : max), carrito[0]);

// 8. Intersección de arreglos de primitivos
const interseccion = (a, b) => a.filter((x) => b.includes(x));

// 9. Diferencia / quitar todos los duplicados de B
const diferencia = (a, b) => a.filter((x) => !b.includes(x));

// 10. Zip: unir dos arreglos por índice
const zip = (a, b) => a.map((x, i) => [x, b[i]]);
```

> [!tip] Cuándo tocar el teclado para `reduce`
> Si la transformación se lee mejor con `filter` + `map` + `for`, usá eso: `reduce` gana cuando el resultado **no es un arreglo** (objeto, número, Map) o cuando hay que atravesar una sola vez con acumulador. Reduce "mágico" con callbacks de 20 líneas es anti-legibilidad.

## Rendimiento

- Los métodos de transformación (`map`, `filter`, `flat`) crean **arreglos nuevos**: O(n) de memoria cada paso. Cadenas largas `map().filter().map()` hacen 3 pasadas y 3 arreglos temporales — para volúmenes grandes (100k+), un solo `reduce` o bucle `for...of` con push es más rápido y contamina menos el GC (ver [[Funciones]] → Rendimiento).
- `push` en `for...of` con acumulador es el patrón de máximo rendimiento conocido (sin dependencia de callbacks).
- `includes`/`indexOf`/`filter(...).includes(...)` son O(n) con comparación por elemento: para pertenencia frecuente, preconvertir a `Set` (O(1) por consulta).
- `some`/`every`/`find` son **perezosos** — cortan en el primer resultado; `filter`/`map` siempre recorren todo.
- `concat` y spread de muchos arreglos: O(n) total, ok. La vieja recomendación de `apply` para `push.apply(dst, src)` con arreglos de ~100k+ **revienta el stack** (límite de argumentos); con arreglos enormes, `for...of` con `push` o `dst = dst.concat(src)`.
- `sort` es O(n log n) con comparador por elemento: comparadores caros (con `localeCompare` sobre miles de strings) se notan; para datos estáticos ordená una vez.

## Cheatsheet de uso rápido

| Quiero… | Método | Firma mínima |
|---|---|---|
| agregar/quitar | `push/pop/shift/unshift` | `arr.push(x)` |
| insertar/borrar en medio | `splice` / `toSpliced` | `arr.splice(i, b, ...e)` |
| ordenar (inmutable) | `toSorted` | `arr.toSorted((a,b) => a-b)` |
| invertir (inmutable) | `toReversed` | `arr.toReversed()` |
| ¿existe el valor? | `includes` (primitivos) | `arr.includes(v)` |
| ¿existe el objeto? | `some` / `find` | `arr.some(p => p.id === x)` |
| elemento por condición | `find` / `findLast` | `arr.find(p => p.id === x)` |
| posición | `findIndex` / `findLastIndex` | `arr.findIndex(p => p.id === x)` |
| transformar | `map` | `arr.map(p => p.x)` |
| dejar solo algunos | `filter` | `arr.filter(p => p.x > 0)` |
| sumar/acumular | `reduce` | `arr.reduce((a, p) => a + p.x, 0)` |
| aplanar | `flat` / `flatMap` | `arr.flat(Infinity)` |
| agrupar | `Object.groupBy` | `Object.groupBy(arr, p => p.cat)` |
| unir | `concat` / spread | `[...a, ...b]` |
| copiar | `slice` / spread / `structuredClone` | `[...arr]` |
| a texto | `join` | `arr.join(', ')` |
| preguntas globales | `some` / `every` | `arr.every(p => p.ok)` |
| recorrer con índice | `entries()` + `for...of` | `for (const [i, v] of arr.entries())` |
| convertir iterable | `Array.from` / `Array.of` | `Array.from(nodelist)` |
| hacer Set | `new Set(arr)` | `[...new Set(arr)]` (únicos) |

## Relacionadas

- [[Arreglos]] — crear, acceder, mutar y la semántica de huecos y `length`
- [[Iteradores]] — `for`, `for...of`, `forEach` y los bucles + protocolo iterable (`keys`/`values`/`entries`)
- [[Funciones]] — callbacks y arrow functions, la base de estos métodos
- [[Objetos]] — trabajar con los objetos dentro del arreglo; `Object.groupBy`
- [[Strings]] — `split`/`join` como puente string ↔ arreglo
- [[Números]] — `reduce` para sumas, promedios, mínimos y máximos
- [[Sintaxis Moderna]] — spread, desestructuración, `Set`, sintaxis de los métodos nuevos
- [[ReactJS - Fundamentos]] — por qué el estado pide `toSorted`/`filter` y no `sort`/`splice`
- [[Document Object Model (DOM)|DOM]] — `Array.from` sobre NodeList
