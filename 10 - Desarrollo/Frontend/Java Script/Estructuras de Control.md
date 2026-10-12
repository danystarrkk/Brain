---
tags:
  - desarrollo
  - frontend
  - estructuras-control
tipo: concepto
estado: terminado
---

# Estructuras de Control

Las **estructuras de control** deciden qué código se ejecuta y en qué orden. Las dos grandes familias son las **condicionales** (`if`, `switch`, ternario) y los **operadores lógicos** (`&&`, `||`, `!`) que combinan condiciones. Los bucles viven en detalle en [[Iteradores]]; aquí se cubre su anatomía completa, las etiquetas y las trampas.

## Comparadores: la base de toda condición

Antes del `if`, conviene dominar los comparadores, porque de ahí sale el booleano que la condición evalúa:

| Operador | Nombre | Ejemplo | Resultado |
| -------- | ------ | ------- | --------- |
| `==` | igual (con coerción de tipos) | `1000 == '1000'` | `true` |
| `===` | igual estricto (valor **y** tipo) | `1000 === '1000'` | `false` |
| `!=` | distinto (con coerción) | `1000 != 2000` | `true` |
| `!==` | distinto estricto | `1000 !== '1000'` | `true` |
| `>` `<` `>=` `<=` | mayor / menor | `300 >= 300` | `true` |

```js
const puntaje = 1000;

if (puntaje == '1000') {
  console.log('El puntaje sí es igual (con coerción)');
}

if (puntaje === '1000') {
  console.log('Nunca entra: number !== string');
} else {
  console.log('No es igual');
}
```

> [!warning] Usa `===` por defecto
> `==` convierte tipos antes de comparar y produce falsos positivos históricos (`0 == ''` es `true`). Con `===` lo que ves es lo que obtienes. Ver [[Booleanos]] y el algoritmo completo de coerción en [[Operadores]].

## La condición: todo se vuelve booleano

Cualquier expresión puede ir entre paréntesis del `if`; el motor aplica **ToBoolean** a su resultado. La tabla completa de valores *falsy* (verificada con node — todo lo demás es truthy):

| Falsy (8 valores) | Nota |
| ----------------- | ---- |
| `false` | el booleano mismo |
| `0` y `-0` | el único número falsy junto a NaN |
| `''` (string vacío) | `0`, `'0'` y `'false'` **sí** son truthy |
| `null` | ausencia explícita |
| `undefined` | sin valor |
| `NaN` | not-a-number |
| `0n` | BigInt cero |

Todo lo demás es truthy: `[]`, `{}`, `'0'`, `'false'`, `-1`, `Infinity`... Incluso `new Boolean(false)` (un objeto wrapper) es truthy — razón por la que los wrappers nunca se usan.

> [!tip] El truthy no es "verdadero"
> `if (lista)` solo pregunta "¿hay algo?"; `if (arr.length)` pregunta "¿hay **elementos**?". `[]` es truthy: `if (miArray)` **nunca** va a rama de vacío. Para arrays y strings usá `.length`; para objetos, `Object.keys(x).length > 0` o `x != null`.

## `if` / `else`

```js
const dinero = 300;
const totalApagar = 300;

if (dinero > totalApagar) {
  console.log('Sí podemos pagar');
} else if (dinero === totalApagar) {
  console.log('Pagamos exacto');
} else {
  console.log('Fondos insuficientes');
}
```

Sin llaves también funciona, pero solo para una línea: es válido y más corto, aunque las llaves explícitas evitan bugs famosos donde "se coló" una instrucción dentro del `if`.

```js
if (puntaje === 1000) console.log('¡Puntaje exacto!');
else console.log('Sigue intentando');
```

### La trampa del `if` sin llaves (dangling if)

El caso histórico — un `if` "absorbe" la línea siguiente aunque el autor creía que iba aparte:

```js
// Lo que el autor creía:
if (a > 0)
  console.log('positivo');
  console.log('esto también parece condicional… pero NO lo es');
// Solo la PRIMERA sentencia queda bajo el if. La segunda corre siempre.
```

Y la versión con `else` todavía peor (*dangling else*): el `else` se asocia al `if` **más cercano**, no al que "alineaste visualmente":

```js
if (a)
  if (b) x = 1;
else x = 2;   // ¡el else pertenece a "if (b)", no a "if (a)"!
```

Regla inquebrantable: **llaves siempre**, incluso para una línea. Es la convención de todos los linters modernos.

## Cadena de `else if`

Cuando hay varios rangos, se encadenan condiciones. Un caso típico: calificar un puntaje.

```js
const puntaje = 400;

if (puntaje >= 300 && puntaje < 400) {
  console.log('Buen puntaje... felicidades');
} else if (puntaje >= 400 && puntaje < 500) {
  console.log('Excelente');
} else {
  console.log('Puntaje fuera de rango');
}
```

Detalles que importan:

- Las cadenas se evalúan **en orden** y el primer `true` gana: el resto **no se evalúa** (cortocircuito del `if`).
- Rangos sin intersección: un bug clásico es olvidar el `<` y escribir `puntaje >= 400` en vez de `>= 400 && < 500`, haciendo que 1000 también sea "Excelente".
- Si encadenás más de ~4 ramas comparando el mismo valor, `switch` es más legible; si las condiciones son arbitrarias, una **tabla lookup** (objeto de funciones) puede eliminar la cadena entera — ver "pattern matching" abajo.

### Guard clauses (retornos tempranos): la forma profesional

En vez de anidar, **invertí** las condiciones y salí temprano:

```js
// Pirámide de if (difícil de leer y de mantener)
function procesar(pedido) {
  if (pedido) {
    if (pedido.pagado) {
      if (pedido.stock > 0) {
        return enviar(pedido);
      } else { console.log('Sin stock'); }
    } else { console.log('Sin pagar'); }
  } else { console.log('Sin pedido'); }
}

// Guard clauses (misma lógica, plano)
function procesar(pedido) {
  if (!pedido) return console.log('Sin pedido');
  if (!pedido.pagado) return console.log('Sin pagar');
  if (pedido.stock <= 0) return console.log('Sin stock');
  return enviar(pedido);   // el camino feliz, sin anidar
}
```

Patrón relacionado: extraer el criterio a una **función con nombre** — `if (esPuntajeValido(puntaje))` comunica más que la expresión cruda y es testeable por separado (ver [[Funciones]]).

## `switch`

`switch` compara **igualdad estricta** (`===`) de una misma expresión contra varios valores. Es más legible que una cadena enorme de `else if` cuando se compara contra constantes.

```js
function pagar() {
  console.log('Pagando....');
}

const metodoPago = 'efectivo';

switch (metodoPago) {
  case 'efectivo':
    pagar(); // se pueden invocar funciones
    console.log(`Pagaste con ${metodoPago}`);
    break; // sin break, "caería" al siguiente case (fall-through)
  case 'cheque':
    console.log(`Pagaste con ${metodoPago}`);
    break;
  case 'tarjeta':
    console.log(`Pagaste con ${metodoPago}`);
    break;
  default:
    console.log('Aún no se soporta este método de pago');
    break;
}
```

### Anatomía y mecánica del *fall-through*

- El motor evalúa los `case` **en orden** con `===` hasta encontrar coincidencia; luego ejecuta **todas** las sentencias siguientes hasta un `break`, `return`, `throw` o el fin del bloque — incluidas las ramas de otros `case`. Verificado: `switch (2) { case 1: …; case 2: log('two'); case 3: log('three'); break; }` imprime **`two,three`**.
- `default` **no** necesita ir al final y no requiere coincidencia previa: si ningún `case` coincide, se ejecuta `default` donde esté.
- `case` acepta **expresiones**, no solo literales: `case a + b:`, `case 'x':` o `case someFn():` — se evalúan en cada comparación.

### Agrupación intencional: el único fall-through válido

```js
switch (dia) {
  case 'sab':
  case 'dom':
    console.log('Fin de semana');   // sin código entre cases = agrupar
    break;
  case 'lun':
  case 'mar':
  case 'mie':
  case 'vie':                       // 5 casos → 1 rama
    console.log('Laborable');
    break;
  default:
    console.log('Jueves… casi');
}
```

### Comparación estricta demostrada

```js
const v = '5';
switch (v) {
  case 5:  console.log('número 5'); break;   // nunca: '5' !== 5
  case '5': console.log('string 5'); break;  // ← gana este
}
```

El mismo algoritmo que `===`: `'5' == 5` habría sido true, pero switch jamás usa coerción.

### Alcance de variables: un solo bloque para todo el switch

Todos los `case` comparten **un mismo scope de bloque** — declarar `let` con el mismo nombre en dos cases es `SyntaxError`, y una variable declarada en un case se ve desde otro (con TDZ si es después):

```js
switch (1) {
  case 1:
    let x = 'visible en todo el switch';
    break;
  case 2:
    console.log(x);   // funciona… pero es una bomba de legibilidad
    break;
}
// switch (1){case 1: let s=1; break; case 2: let s2=s;} → SyntaxError en case 2
```

Si necesitás aislar, envolveé el case en `{ }` propio: `case 2: { let s = 1; break; }`.

### `switch` sobre `typeof`, `instanceof` y rangos

```js
switch (typeof valor) {
  case 'string': /*…*/ break;
  case 'number': /*…*/ break;
  default: /*…*/
}

switch (true) {           // "switch(true)" con condiciones: funciona pero polariza
  case x > 100: /*…*/ break;
  case x > 10:  /*…*/ break;
}
```

`switch (true)` reutiliza la igualdad estricta contra `true`: los cases deben devolver booleanos. Es legítimo pero controvertido — con ramas arbitrarias un `if` encadenado se lee igual de bien.

## Bucles: anatomía completa

### `for` — el bucle controlado

```
for (inicio; condición; actualización) { cuerpo }
```

1. `inicio` se ejecuta **una sola vez** (puede declarar la variable del contador).
2. La `condición` se evalúa **antes** de cada vuelta; `false` termina el bucle.
3. El cuerpo se ejecuta.
4. La `actualización` corre, y vuelve al paso 2.

```js
for (let i = 0; i < 3; i++) console.log(i);   // 0, 1, 2
```

- Los tres segmentos son **opcionales**: `for (;;)` es un bucle infinito válido (sale con `break`).
- `let` en el `for` crea el contador **por iteración** (cierres/closures correctos); `var` comparte uno solo — ver [[Sintaxis Antigua]].
- `continue` salta a la actualización; `break` sale del todo.

### `while` — condición primero

```js
let n = 3;
while (n > 0) {
  n--;               // si olvidás esto: bucle infinito
}
```

Evalúa la condición **antes** de cada vuelta: con la condición falsa desde el inicio, el cuerpo **nunca** corre. Ideal cuando no sabés cuántas vueltas faltan (leer hasta fin de stream, esperar estado).

### `do...while` — cuerpo primero

```js
let i = 0;
do { i++; } while (i < 5);
console.log(i);   // 5
```

El cuerpo corre **siempre al menos una vez** (verificado), luego evalúa. Caso de uso real: menús y formularios "mostrar hasta que se confirme"; hoy menos común porque el mismo efecto se logra con una función + guard clause.

### `for...in` — claves enumerables

```js
const obj = { a: 1, b: 2 };
for (const clave in obj) console.log(clave, obj[clave]);
```

- Itera **claves** (como strings) del objeto **y de su prototipo** — verificado con `Object.create({heredado:1})`: imprime `propio` e `heredado`. Guard moderno: `Object.hasOwn(obj, k)` o `Object.keys(obj)`.
- Diseñado para objetos; **nunca** para arreglos (itera índices como strings, incluye heredados, y el orden no era garantizable en ES5).

### `for...of` — valores de cualquier iterable

```js
for (const v of [10, 20]) console.log(v);        // 10, 20
for (const ch of 'hola') console.log(ch);         // letras
for (const [k, v] of new Map([['a', 1]])) console.log(k, v);
```

Recorre **valores** siguiendo el protocolo de iteración: arreglos, strings, Map, Set, NodeList, generadores. El detalle del protocolo (`Symbol.iterator`, `next()`) vive en [[Iteradores]].

| Bucle | Itera | Recomendado para |
| ----- | ------ | ---------------- |
| `for` | lo que le digas | contador, índice, patrón especial |
| `while` | hasta condición | vueltas desconocidas |
| `do...while` | ≥1 vez | pocos casos (menús) |
| `for...in` | **claves** (con herencia) | solo objetos, con `hasOwn` |
| `for...of` | **valores** (iterables) | el bucle por defecto del día a día |

### Bucle infinito y control de salida

```js
while (true) {
  const dato = obtener();
  if (!dato) break;            // salir
  if (dato.raro) continue;     // saltar a la siguiente vuelta
  procesar(dato);
}
```

`break` y `continue` afectan al **bucle más interno**. Para salir de bucles anidados: etiquetas.

## Etiquetas: `break` y `continue` con label

Una etiqueta es un identificador seguido de `:` delante de una sentencia (típicamente un bucle); `break etiqueta` sale de **ese** bucle y `continue etiqueta` salta a su **siguiente iteración**, atravesando anidamientos:

```js
// Buscar en matriz 3x3 y salir del bucle EXTERNO
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) break outer;   // termina los dos bucles
  }
}

// continue con etiqueta: saltar la vuelta completa del bucle externo
outer2: for (let a = 0; a < 3; a++) {
  for (let b = 0; b < 3; b++) {
    if (b === 1) continue outer2;   // no procesa el resto de la fila
    console.log(a, b);              // imprime solo (0,0) (1,0) (2,0)
  }
}
```

Verificado: el `continue` etiquetado con la condición `b === 1` ejecuta el cuerpo exterior exactamente 3 veces. Alternativas a menudo más legibles: extraer a función (el `return` sale de todo) o `Array.flat()` + `find` sobre la matriz.

> [!warning] Etiquetas con estilo de goto
> Son el único mecanismo "goto" de JS. Regla: úsalas de forma **escasa y con nombre descriptivo** (`buscarProducto:`), nunca anides etiquetas ni las uses para reordenar lógica normal — si la lógica necesita dos etiquetas, casi siempre conviene una función.

## Sin pattern matching nativo: alternativas modernas

JavaScript **no** tiene *pattern matching* (la propuesta de TC39 sigue en etapa temprana, ver [[Sintaxis Moderna]]). Las alternativas que se usan hoy:

### 1. Tabla lookup: el switch convertido en datos

```js
const acciones = {
  efectivo: () => console.log('Pagando en efectivo'),
  cheque:   () => console.log('Depositando cheque'),
  tarjeta:  () => console.log('Cobrando a tarjeta'),
};
(acciones[metodoPago] ?? acciones.desconocido)();
```

### 2. Switch sobre discriminante (el "pattern matching" del día a día)

```js
function manejar(msg) {
  switch (msg.type) {
    case 'USER_LOGIN':  return Login(msg.payload);
    case 'USER_LOGOUT': return Logout();
    case 'ITEM_ADD':    return Add(msg.payload);
    default:            return msg;   // devolver tal cual: ignorar desconocidos
  }
}
```

### 3. Destructuring en parámetros: "desestructurar el patrón"

```js
function dibujar(figura) {
  switch (figura.tipo) {
    case 'circulo':  const { r } = figura; /*…*/ break;
    case 'rectangulo': { const { w, h } = figura; /*…*/ break; }
  }
}
```

### 4. Clases con polimorfismo: cuando el "patrón" es un tipo real

```js
// Cada tipo define su comportamiento; no hace falta switch:
figura.dibujar();
```

Criterio rápido: **pocos valores fijos** → switch; **pares clave→acción** → objeto lookup; **tipos con comportamiento** → polimorfismo; **condiciones arbitrarias** → if con guard clauses.

## Rendimiento de los patrones de rama

- **Nunca midas ramas sin perfiles**: los motores JIT (ver [[Java Script]]) convierten cadenas de `if` y `switch` pequeños en *jump tables* o comparaciones inline — las diferencias entre `if` y `switch` con pocas ramas son **ruido** en aplicaciones reales.
- **Ramas predecibles vs impredecibles**: un `if` cuyo resultado es ~siempre igual se predice sin costo; una condición 50/50 obliga al JIT a *deoptimizar* ramas especulativas. Esto solo importa en bucles calientes (millones de iteraciones) medidos con la pestaña Performance (ver [[La Consola]]).
- **El costo real está en el cuerpo, no en la condición**: `if` con DOM (`querySelector` dentro del loop) dispara reflows; medí con `performance.now()` antes de micro-optimizar.
- **Orden de condiciones**: poné las más frecuentes/ baratas primero en cadenas largas (se evalúan hasta el primer `true`).
- **Bucles**: evitá trabajo invariante dentro del cuerpo — calcular `arr.length` o `Math.sqrt` constante afuera solo importa en bucles muy grandes, pero es gratis hacerlo. Para DOM masivo: `DocumentFragment`, delegación de eventos y `requestAnimationFrame` (ver [[Document Object Model (DOM)|DOM]]).

## Errores comunes

1. **`==` en condiciones** — `if (valor == '0')` acepta `0`, `''`, `false`, `'0'`… con coerciones distintas. `===` siempre; la única excepción es el idiom `x == null` (null + undefined).
2. **Asignación en la condición** — `if (x = 5)` **asigna** 5 y evalúa truthy: entra siempre. Verificado: la expresión vale `5`. Un `=` donde va `==`/`===` compila y no avisa. Lint: `no-cond-assign`.
3. **Sin llaves** — el *dangling if* se coló (ver arriba). Llaves siempre.
4. **Comparar con `NaN`** — `if (x === NaN)` nunca entra. `Number.isNaN(x)`.
5. **Confiar en `if (objeto)` o `if (array)`** — `[]` y `{}` son truthy; para "está vacío" usá `.length` / `Object.keys().length`.
6. **`switch` sin `break`** — fall-through accidental ejecuta ramas extra; el error no lanza, solo hace cosas raras. Agrupación vacía solo cuando es intencional.
7. **`continue` que no continúa** — en `for` salta a la actualización del contador; olvidarlo provoca bucles infinitos con `continue` como primera línea de `while`.
8. **`for...in` sobre arreglos** — índices como strings + herencia; `for...of` o `map`.
9. **Bucle infinito con `while (cond)` y `cond` que nunca cambia** — depurar con `console.count` dentro del cuerpo o pausa condicional en DevTools (ver [[La Consola]]).
10. **Rangos con solapamiento** — `if (x > 10) … else if (x > 5) …` donde la segunda rama asumía `<= 10`; los else-if solo alcanzan si no entraron antes, pero el orden inverso (`> 5` primero) las rompe. Verificá ambos extremos.
11. **Etiquetas duplicadas** — dos bucles con la misma etiqueta en el mismo scope: `SyntaxError`.

## Cheatsheet

| Estructura | Forma | Detalle clave |
| ---------- | ----- | ------------- |
| `if` | `if (c) {} else if (c2) {} else {}` | primer true gana; llaves siempre |
| Guard clause | `if (!x) return;` | planifica el flujo, elimina anidación |
| Ternario | `c ? a : b` | expresión; asociatividad derecha |
| `switch` | `switch (v) { case a: … }` | `===`, fall-through, un scope |
| `for` | `for (ini; cond; act)` | condición **antes** de cada vuelta |
| `while` | `while (c) {}` | 0+ iteraciones |
| `do...while` | `do {} while (c)` | ≥1 iteración |
| `for...in` | `for (const k in obj)` | claves, con heredadas: `hasOwn` |
| `for...of` | `for (const v of iterable)` | valores, iterable protocol |
| `break` | sale del bucle/switch actual | con etiqueta: del bucle etiquetado |
| `continue` | salta a la siguiente vuelta | con etiqueta: del bucle etiquetado |
| Etiqueta | `bucle: for(…){ break bucle; }` | única salida de bucles anidados |
| Lookup table | `{ caso: fn }` | alternativa a switch largo |

## Relacionadas

- [[Booleanos]] — truthy/falsy y el tipo booleano
- [[Iteradores]] — bucles en profundidad: `for...of`, generadores, protocolo
- [[Funciones]] — extraer condiciones complejas a funciones con nombre
- [[Operadores]] — operadores de JavaScript en detalle (coerción de las comparaciones)
- [[Sintaxis Moderna]] — `?.`/`??` como control de flujo sin `if`, optional chaining
- [[Sintaxis Antigua]] — cómo se escribían estas estructuras en ES5
- [[La Consola]] — depurar ramas con breakpoints condicionales y `console.count`
- [[Java Script]] — el motor JIT detrás de las decisiones de rendimiento
- [[Document Object Model (DOM)]] — por qué el DOM dentro de bucles es caro
