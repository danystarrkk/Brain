---
aliases: ["Operadores JavaScript", "Operators", "Coerción de tipos", "Precedencia de operadores"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Java Script]]"
  - "[[Booleanos]]"
  - "[[Números]]"
  - "[[Strings]]"
  - "[[Estructuras de Control]]"
  - "[[Sintaxis Moderna]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Operators"
---

# Operadores

Un **operador** es un símbolo que produce un valor a partir de uno (**unarios**), dos (**binarios**) o tres (**ternario**) operandos. JavaScript tiene ~50 operadores agrupados en familias: aritméticos, de asignación, comparación, lógicos, bitwise y de acceso. Más allá de la sintaxis, dominar JS es dominar la **coerción**: la conversión automática de tipos que muchos operadores ejecutan antes de operar. Todos los resultados citados aquí fueron verificados ejecutando el código con `node`.

```jsx
const num1 = 20;
const num2 = "20";
const num3 = 30;

// Operador Mayor que....
console.log(num1 > num3);   // false
console.log(num3 > num1);   // true

// Operador menor que..
console.log(num1 < num3);   // true
console.log(num3 < num1);   // false
```

## Mapa de familias

| Familia | Operadores | Naturaleza |
| ------- | ---------- | ---------- |
| Aritméticos | `+ - * / % ** ++ -- +unario -unario` | producen `number` (salvo `+` con strings) |
| De asignación | `= += -= *= /= %= **= &&= ||= ??=` | binarios, el izq. debe ser "locatable" |
| Comparación | `== != === !== < > <= >=` | producen `boolean` |
| Lógicos | `&& \|\| ! ??` | devuelven **operandos**, no booleanos |
| Ternario | `cond ? a : b` | expresión de 3 operandos |
| Bitwise | `& \| ^ ~ << >> >>>` | operan sobre enteros de 32 bits |
| Identidad/tipo | `typeof instanceof in` | predicados |
| Otros | `delete void comma (,) new await yield` | miscelánea |

## Aritméticos

| Operador | Nombre | Ejemplo | Resultado |
| -------- | ------ | ------- | --------- |
| `+` | suma / concatenación | `2 + 3` / `'a' + 'b'` | `5` / `'ab'` |
| `-` | resta | `5 - 2` | `3` |
| `*` | multiplicación | `4 * 3` | `12` |
| `/` | división | `7 / 2` | `3.5` |
| `%` | módulo (resto) | `7 % 2` | `1` |
| `**` | potencia | `2 ** 10` | `1024` |
| `++` / `--` | incremento / decremento | `x++` | `x + 1` |

### `+` es el operador sobrecargado

`+` es el único operador con dos semánticas: **suma numérica** o **concatenación de strings**, decididas por los tipos:

```js
console.log('20' - 5);   // 15   ← '-' SIEMPRE es numérico: "20" → 20
console.log('20' * 5);   // 100  ← ídem para '*'
console.log('20' + 5);   // '205' ← '+' ve un string y concatena
console.log(5 + '20');   // '520' ← ídem (concatenación commutativa, suma no)
console.log(1 + 2 + '3');   // '33' ← izq→der: (1+2)=3, luego '3'+'3'
console.log('1' + 2 + 3);   // '123' ← izq→der: '12'+'23'
```

> [!warning] El orden importa
> Los binarios asocian **izquierda a derecha**: `1 + 2 + '3'` agrupa `(1+2)` y concatena; `'1' + 2 + 3` empieza concatenando. Misma expresión con distinto orden, tres resultados distintos — el clásico bug de sumar precios concatenando strings. Ver [[Strings]] y [[Números]].

### Resta, multiplicación y división: siempre numéricas

Estos convierten **ambos** operandos a número (ToNumber) sin importar el otro tipo: `'5' * '2'` es `10` (strings numéricos válidos), `'hola' * 2` es `NaN`. La división **nunca** trunca: `7 / 2` es `3.5`.

```js
console.log(1 / 0);    // Infinity
console.log(-1 / 0);   // -Infinity
console.log(0 / 0);    // NaN — la "indeterminación" del lenguaje
```

### Módulo `%`: el resto con signo del dividendo

```js
console.log(7 % 2);    // 1
console.log(-7 % 3);   // -1  ← el resto conserva el signo del PRIMER operando
console.log(7 % -3);   // 1
console.log(10 % 0);   // NaN (no error)
```

Recetas clásicas del `%`: **par/impar** (`n % 2 === 0`), **ciclos** (`i % largo` para envolver índices), **cada N iteraciones** (`if (i % 100 === 0)`), y hash simple de strings.

### `**` potencia (ES2016)

```js
console.log(2 ** 10);   // 1024
console.log(2 ** 3 ** 2);   // 512  ← asocia DERECHA a izquierda: 2 ** (3 ** 2)
console.log(2 ** -1);   // 0.5  ← exponente negativo = fracción (1/2)
console.log((-2) ** 2); // 4
// -2 ** 2  es SyntaxError: el unario unido a ** necesita paréntesis
```

> [!info] `**` es el único operador binario **asociativo a la derecha**
> `a ** b ** c` = `a ** (b ** c)`, como en matemáticas. Todos los demás binarios agrupan de izquierda a derecha. Y el `-` unario no puede pegarse a `**` sin paréntesis — el parser lo rechaza (`SyntaxError`), no da el resultado que "esperarías".

## Incremento y decremento

```js
let n = 5;
console.log(n++); // 5  ← devuelve ANTES de incrementar
console.log(n);   // 6
console.log(++n); // 7  ← incrementa ANTES de devolver
console.log(n--); // 7
```

- **Postfijo** (`x++`): devuelve el valor viejo, luego incrementa.
- **Prefijo** (`++x`): incrementa primero, devuelve el nuevo.
- En una frase sola (`n++`), la diferencia solo se nota si usás el valor de la expresión.
- Coerción incluida: `let t = '5'; t++` → `t` es `6` (number). Con `NaN` se propaga: `let k = NaN; k++` → `NaN`.

## Asignación simple y compuesta

`=` asigna; las compuestas (`+=`, `-=`, …) son azúcar que expande la operación:

```js
let x = 10;
x += 5;    // x = x + 5   → 15
x -= 3;    // x = x - 3   → 12
x *= 2;    // x = x * 2   → 24
x /= 4;    // x = x / 4   → 6
x %= 4;    // x = x % 4   → 2
x **= 3;   // x = x ** 3  → 8

let s = 'a';
s += 1;    // s = 'a' + 1 → 'a1'  ← el += hereda la sobrecarga del +
```

### Asignación lógica (ES2021): `&&= ||= ??=`

```js
let cfg = { debug: false, nivel: 0, tema: null };
cfg.debug ||= true;    // falsy → true
cfg.nivel ||= 5;       // 0 es falsy → 5
cfg.tema  ??= 'claro'; // solo null/undefined → 'claro'
```

Equivalencia exacta: `x ||= v` ≡ `x || (x = v)` — la asignación **solo ocurre** si la condición del operador pasa. Ver el análisis completo en [[Sintaxis Moderna]].

### Asignación encadenada y múltiple

```js
let a, b;
a = b = 5;   // a=5 y b=5: la asignación retorna el valor y asocia a la derecha
```

## Comparación relacional: `< > <= >=`

Algoritmo: si **ambos** son strings → comparación **lexicográfica** (código Unicode, no numérico). En cualquier otro caso → convierte ambos a número y compara.

```js
console.log(10 < 9);        // false (numérico)
console.log('10' < '9');    // true  ← '1' < '9' lexicográfico: string gana
console.log('abc' < 'abd'); // true  ← orden de diccionario
console.log('A' < 'a');     // true  ← 'A'=65 < 'a'=97 (mayúsculas antes)

console.log(NaN < 1);   // false — TODA comparación con NaN es false
console.log(NaN >= 1);  // false
console.log(NaN === NaN); // false — NaN no es igual ni a sí mismo
```

> [!warning] El gotcha definitivo de `null` en comparaciones
> `null` se convierte a **0** en la relacional pero tiene regla especial en la igualdad (ver abajo). Resultado verificado:
> `null >= 0` → `true` · `null <= 0` → `true` · `null > 0` → `false` · `null < 0` → `false` · `null == 0` → `false`.
> Relaciones que parecen contradictorias: `null` está "en el mismo punto" que `0` para `>=`/`<=`, pero ni es mayor ni menor. Nunca compares `null` con rangos — verificá nullish directamente.

Para detectar NaN hay que usar el dedicado: `Number.isNaN(x)` (ver [[Números]]) o `Object.is(x, NaN)`.

## Igualdad: `==` vs `===`

```jsx
console.log(num1 == num3);  // false
console.log(num1 == num2);  // true  — comparador NO estricto: solo "valor" tras coercionar
console.log(num1 === num2); // false — estricto: 20 (number) !== "20" (string)
console.log(num1 == parseInt(num2)); // true — parseInt("20") = 20, tipos iguales

// Diferente que...
const password1 = "admin";
const password2 = "Admin";
console.log(password1 != password2); // true — 'admin' y 'Admin' difieren en mayúsculas
console.log(num1 != num2);           // false — no estricto: coerciona y resultan iguales
console.log(num1 !== num2);          // true — estricto: types differ
```

### `===` (igualdad estricta)

Dos pasos, sin conversiones: **mismos tipos** → comparar valores. `20 === '20'` es `false` inmediatamente. Casos raros documentados:

```js
console.log(NaN === NaN);        // false (excepción del lenguaje)
console.log(0 === -0);           // true  (comparación de valores: son el mismo cero)
console.log(Object.is(NaN, NaN));// true  — la herramienta para NaN y -0
console.log(Symbol('a') === Symbol('a')); // false — symbols son únicos
```

### `==` (igualdad abstracta): el algoritmo completo

`==` **no** compara "solo el valor" — ejecuta un algoritmo de coerción de la especificación (Abstract Equality) con reglas exactas:

| # | Tipos | Regla |
| - | ----- | ----- |
| 1 | mismos tipos | como `===` (valor; NaN nunca igual) |
| 2 | `null` vs `undefined` | **`true`** (solo este cruce; `undefined == 5` → false) |
| 3 | `number` vs `string` | string → `Number`, luego regla 1 |
| 4 | `boolean` | el boolean → `Number` (`true`→1, `false`→0), luego comparar |
| 5 | `string`/`number` vs objeto | el objeto → primitivo vía `valueOf`/`toString`, luego reglas |

La tabla de casos célebres (todos verificados con node):

| Comparación | `==` | `===` | Por qué |
| ----------- | ---- | ---- | ------- |
| `0 == ''` | `true` | `false` | `''` → `0` (regla 3) |
| `'' == false` | `true` | `false` | ambos → `0` (reglas 3+4) |
| `0 == false` | `true` | `false` | `false` → `0` (regla 4) |
| `null == undefined` | `true` | `false` | regla 2 |
| `null == 0` | `false` | `false` | null solo es igual a undefined |
| `'1' == 1` | `true` | `false` | regla 3 |
| `[] == false` | `true` | `false` | `[]`→`''`→`0`, `false`→`0` |
| `[] == ![]` | `true` | `false` | `![]`→`false`→`0`; `[]`→`0` |
| `NaN == NaN` | `false` | `false` | regla 1 |
| `Symbol('a') == Symbol('a')` | `false` | `false` | únicos |

> [!warning] `==` no es transitivo
> `'' == 0` ✓, `0 == false` ✓, pero `'' == false`… también ✓ — la cadena funciona, pero al estilo ruso: con `['a']` y otros casos las cadenas de coerción dejan de ser transitivas. La moraleja: el resultado de `==` **no** es razonable por más que cada paso individual lo parezca. Con `===` lo que ves es lo que obtienes.

### Las dos excepciones donde `== null` es idiomático

```js
if (val == null) { /* val es null O undefined */ }
```

Este es el **único** uso aceptado de `==` en código profesional: captura los dos nulos de golpe sin escribir `val === null || val === undefined`. Todo lo demás: `===`.

### Relacional con `==` en la práctica

```js
const puntaje = 1000;
if (puntaje == '1000') {          // true: coerciona '1000' → 1000
  console.log('igual con coerción');
}
if (puntaje === '1000') {
  console.log('nunca entra');
} else {
  console.log('No es igual');     // ← el camino correcto
}
```

## Lógicos: `&&`, `||`, `!`, `??`

### Devuelven **operandos**, no booleanos

```js
console.log(0 || 'default');        // 'default'
console.log('' || null || 'x');     // 'x'
console.log(1 && 2);                // 2  (devuelve el último operando)
console.log('a' && 'b' && 'c');     // 'c'
console.log(null && 1);             // null (el primero falsy)
console.log(0 ?? 'x');              // 0  — 0 no es nulo
```

- **`a && b`**: si `a` es falsy → devuelve `a`; si no → devuelve `b`. "Ejecuta/propaga solo si todo es truthy".
- **`a || b`**: si `a` es truthy → devuelve `a`; si no → `b`. "El primer truthy de la cadena".
- **`!a`**: el único lógico que **siempre** devuelve booleano: `!valor` = negación de truthiness. Doble negación `!!valor` = coerción a booleano puro (ver [[Booleanos]] para la tabla truthy/falsy).

### Cortocircuito: evaluación perezosa

En `a && b`, si `a` es falsy, **`b` nunca se evalúa**; en `a || b`, si `a` es truthy, tampoco. Patrones reales:

```js
// Guard: ejecutar solo si existe
usuario && usuario.saludo();          // ES5; hoy: usuario.saludo?.()

// Default con efecto secundario mínimo
const limite = config.limite || 10;   // trampa si limite===0; ver ?? abajo

// Cortocircuito defensivo
const nombre = entrada && entrada.trim();
```

### `??` nullish coalescing: el default correcto

`??` devuelve el izquierdo **salvo** que sea `null`/`undefined`; a diferencia de `||` **no** descarta `0`, `''`, `false` ni `NaN`:

```js
console.log(0 ?? 5);    // 0    (|| daría 5)
console.log('' ?? 5);   // ''   (|| daría 5)
console.log(false ?? 5);// false(|| daría 5)
console.log(null ?? 5); // 5
```

> [!warning] Sintaxis prohibida
> `a ?? b || c` es `SyntaxError` — el parser prohíbe mezclar `??` con `&&`/`||` sin paréntesis explícitos: `(a ?? b) || c`. Verificado: el motor rechaza el archivo entero.

## Ternario `? :`

El único operador de **tres** operandos y una expresión completa:

```js
const dinero = 300, total = 300;
const mensaje = dinero >= total ? 'Pagamos' : 'Fondos insuficientes';

// Equivale a:  condición ? valorSiTrue : valorSiFalse
console.log(dinero === total ? 'exacto' : 'no exacto'); // 'exacto'
```

Asocia de **derecha a izquierda**: `a ? b : c ? d : e` = `a ? b : (c ? d : e)`. Anidar más de un nivel arruina la legibilidad — ver [[Estructuras de Control]] para cuándo extraer a `if/else`.

## Coma `,`

Evalúa ambos operandos y devuelve el **último**:

```js
console.log((1, 2, 3));   // 3
let x = (1, 2, 3);        // x = 3
```

Usos reales: inicialización múltiple en el `for` clásico (`for (i = 0, j = 10; …)`), y en libs minificadas. En cualquier otro contexto confunde al lector — evitálo en código normal.

## `typeof`: detectar tipo en runtime

Unario, **no lanza error** aunque el nombre no exista:

```js
console.log(typeof 'texto');        // "string"
console.log(typeof 42);             // "number"
console.log(typeof 10n);            // "bigint"
console.log(typeof true);           // "boolean"
console.log(typeof undefined);      // "undefined"
console.log(typeof Symbol());       // "symbol"
console.log(typeof {});             // "object"
console.log(typeof []);             // "object"  ← no distingue arrays
console.log(typeof null);           // "object"  ← ¡el quirk!
console.log(typeof function () {}); // "function" ← único "tipo" que no es de datos
console.log(typeof totallyUndeclared); // "undefined" — sin ReferenceError
```

| Resultado de `typeof` | Valores |
| --------------------- | ------- |
| `"undefined"` | `undefined`, variables no declaradas |
| `"boolean"` | `true`, `false` |
| `"number"` | todos los números (incluye `NaN`, `Infinity`, `-0`) |
| `"bigint"` | `10n` |
| `"string"` | `'a'`, `"a"` |
| `"symbol"` | `Symbol('x')` |
| `"function"` | cualquier función/constructor |
| `"object"` | `null`, objetos, arreglos, fechas, regexp, errores |

### El quirk `typeof null === "object"`

Bug heredado de 1995: en el primer diseño, los valores se representaban con un bit que marcaba "es referencia" (object) o "es primitivo"; `null` era el puntero nulo con la marca de referencia → el engine respondía `"object"`. Corregirlo rompería la web entera, así que quedó **para siempre** — y por eso `null` necesita comprobación explícita (`=== null`) además de `typeof`.

> [!tip] Para arrays y null, hay herramientas mejores
> - Array: `Array.isArray(x)` (funciona hasta con `x` de otro iframe/realm, que `instanceof` no).
> - Null: `x === null`.
> - `typeof` brilla para primitivos y para no romper con variables no declaradas.

## `instanceof`

Comprueba si `Constructor.prototype` está en la **cadena de prototipos** del valor:

```js
console.log([] instanceof Array);         // true
console.log([] instanceof Object);        // true (Array.prototype hereda de Object.prototype)
console.log({} instanceof Object);        // true
console.log(null instanceof Object);      // false (null no tiene prototipos)
console.log(new Date() instanceof Date);  // true

function Animal() {}
function Perro() {}
Perro.prototype = Object.create(Animal.prototype);
console.log(new Perro() instanceof Perro);  // true
console.log(new Perro() instanceof Animal); // true — la cadena entera se recorre
```

- **No sirve** para primitivos: `'texto' instanceof String` → `false` (el primitivo no tiene cadena de prototipos).
- **Falla entre realms** (mismo código en dos iframes/windows: `[] instanceof Array` puede ser `false` si el `Array` es de otro realm). Para eso: `Array.isArray`.
- `instanceof null` simplemente es `false`; no lanza.

Ver [[Objetos]] para el modelo de prototipos que hace posible todo esto.

## `in`: presencia de propiedad

```js
const p = { q: 1, r: undefined };
console.log('q' in p);    // true
console.log('r' in p);    // true — aunque su VALOR sea undefined
console.log('z' in p);    // false
console.log(0 in [10]);   // true — claves de array son strings de índice
console.log('length' in [10]); // true

// Protección contra heredados (para...in):
const hijo = Object.create({ heredado: 1 });
hijo.propio = 2;
for (const k in hijo) console.log(k); // imprime 'propio' Y 'heredado'
console.log(Object.hasOwn(hijo, 'heredado')); // false — la forma moderna del guard
```

`in` incluye el prototipo completo. Para "propiedad propia": `Object.hasOwn(obj, k)` (ES2022; antes `obj.hasOwnProperty(k)`).

## `delete`: eliminar propiedades

```js
const o = { a: 1, b: 2 };
console.log(delete o.a);    // true
console.log('a' in o);      // false

const arr = [1, 2, 3];
delete arr[1];              // true
console.log(arr);           // [1, empty, 3] — el índice DESAPARECE pero el largo queda 3
console.log(1 in arr);      // false — queda un "hueco" (hole), no un undefined
```

- `delete` **no reduce** `length` de arreglos: deja huecos que rompen `for...of`/`map` (el hueco se itera como `undefined` pero el slot no existe). Nunca borres índices de arreglos: usá `splice`.
- `delete` en variable declarada con `var/let/const` no borra nada: devuelve `false` (sloppy) o `SyntaxError` (const/estricto). Solo opera **propiedades**.
- En modo estricto, `delete` sobre una variable es error de sintaxis; sobre propiedad dinámica (`delete obj[clave]`) funciona.

## `void`: el undefined intencional

```js
console.log(void 0);        // undefined
console.log(void undefined);// undefined — evalúa y devuelve undefined siempre
```

`void expr` evalúa `expr` y devuelve `undefined` **sin importar qué devuelva**. Uso clásico: `href="javascript:void 0"` (link que no navega) y la constante `undefined` segura del ES5 (donde `undefined` era una global writable). Hoy casi no se necesita; `undefined` es seguro.

## `new`: invocar constructores

```js
function Producto(nombre) { this.nombre = nombre; }
const p = new Producto('Monitor');
console.log(p.nombre);           // 'Monitor'
console.log(p instanceof Producto); // true
```

`new` ejecuta 4 pasos: crea un objeto **vacío** con `Producto.prototype` como prototipo → llama a la función con `this` = ese objeto → si la función devuelve un **objeto**, devuelve ese; si no, devuelve el objeto creado. Por eso `new (() => {})` lanza `TypeError` (las arrows no son constructores) y por qué `new` tiene **precedencia altísima**: `new A().x` es `(new A()).x`, y `new A.x` intenta `new (A.x)()` (verificado: lanza `TypeError` si `A.x` no es constructor).

> [!tip] `new.target`
> Dentro de una función, `new.target !== undefined` indica si se llamó con `new`. Es la guarda clásica para constructores "que también funcionan sin new".

## Bitwise: `& | ^ ~ << >> >>>`

Operan sobre la **representación de 32 bits** del número (conversión ToInt32) y devuelven `number`:

```js
console.log(5 & 3);     // 1    0101 & 0011 = 0001
console.log(5 | 3);     // 7    0101 | 0011 = 0111
console.log(5 ^ 3);     // 6    0101 ^ 0011 = 0110
console.log(~5);        // -6   ~(x) === -(x+1) en enteros con signo
console.log(1 << 30);   // 1073741824
console.log(-1 >>> 0);  // 4294967295 ← >>> "sin signo": recorre bits como unsigned
```

| Operador | Nombre | Significado |
| -------- | ------ | ----------- |
| `a & b` | AND | bits encendidos en ambos (máscara) |
| `a \| b` | OR | bits encendidos en cualquiera |
| `a ^ b` | XOR | bits distintos |
| `~a` | NOT | complemento: `~x = -(x+1)` |
| `a << n` | shift izquierda | multiplica por 2ⁿ |
| `a >> n` | shift derecha (con signo) | divide por 2ⁿ redondeando hacia −∞ |
| `a >>> n` | shift derecha **sin signo** | bits entran con ceros (máscara de 32 bits) |

Propiedades útiles y trampas verificadas:

```js
console.log(3.9 | 0);   // 3  ← truncamiento rápido de decimales
console.log('7' | 0);   // 7  ← convierte la string numérica
console.log(true | 0);  // 1  ← boolean → 0/1
console.log(NaN | 0);   // 0  ← NaN se convierte en 0 (¡sorpresa silenciosa!)
```

- **No** son operadores de conjuntos de bits "puros": convierten con ToInt32 — por eso truncan decimales y anulan `NaN` sin avisar.
- `>>` preserva signo (negativos llenan con 1); `>>>` no (`-1 >>> 0` = 4294967295 — el truco para convertir a unsigned 32 bits).
- Uso real: flags empaquetados (`PERM_READ | PERM_WRITE`), hashes, `| 0` para asegurar entero, álgebra de bits en algoritmos. El uso histórico como "entero rápido" ya no es más rápido en motores modernos — con `Math.trunc` o `Math.floor` se lee mejor.
- Los de **dos operandos** (`& | ^ << >> >>>`) devuelven siempre 32 bits con signo (salvo `>>>`); el de uno (`~`) también.

## Operadores de acceso y pertenencia (repaso cruzado)

| Operador | Sintaxis | Nota | Ver |
| -------- | -------- | ---- | --- |
| Miembro | `obj.prop`, `obj[exp]` | la clave se convierte a string/symbol | [[Objetos]] |
| Opcional | `obj?.prop`, `obj?.[exp]`, `fn?.()` | corta si null/undefined | [[Sintaxis Moderna]] |
| Llamada | `fn(a, b)` | `this` dinámico salvo arrow | [[Funciones]] |
| `in` | `'k' in obj` | incluye prototipos | arriba |
| `delete` | `delete obj.k` | deja `undefined` real | arriba |
| `typeof` | `typeof x` | 8 resultados posibles | arriba |
| `instanceof` | `x instanceof C` | recorre la cadena de prototipos | arriba |

## Operadores asíncronos y de generación

- **`await`** (dentro de `async`): pausa la función hasta que la promesa se resuelva; el valor devuelto es la resolución. Precisa de `async` — fuera de ella es `SyntaxError`. Ver [[Funciones]] y [[Uso de Java Script]].
- **`yield`** (dentro de `function*`): entrega un valor al iterador y pausa el generador; `yield*` delega en otro iterable. Ver [[Iteradores]].

## Precedencia y asociatividad

La **precedencia** decide quién se agrupa primero; la **asociatividad** (izquierda/derecha) decide el orden entre operadores del mismo nivel. De mayor a menor (niveles clave):

| Precedencia | Operadores | Ejemplo / nota |
| ----------- | ---------- | -------------- |
| 19 | `()` agrupación, `[]`, `.` , `?.` | siempre lo primero |
| 18 | `new` (con argumentos), `?.()` | `new A().x` ≡ `(new A()).x` |
| 17 | `x++` `x--` (postfijo) | más alto que todo lo binario |
| 16 | `!` `~` `+x` `-x` `typeof` `void` `delete` `await` | unarios; **asocian a la derecha**: `- -x` = `-(-x)` |
| 15 | `**` | **asocia a la derecha**: `2 ** 3 ** 2` = 512 |
| 14 | `*` `/` `%` | izquierda |
| 13 | `+` `-` | izquierda |
| 12 | `<<` `>>` `>>>` | izquierda |
| 11 | `<` `<=` `>` `>=` `in` `instanceof` | izquierda |
| 10 | `==` `!=` `===` `!==` | izquierda |
| 9 | `&` | |
| 8 | `^` | |
| 7 | `\|` | |
| 6 | `&&` | |
| 5 | `\|\|` | |
| 4 | `??` | no mezclar con `\|\|`/`&&` sin paréntesis |
| 3 | `?:` | asocia a la derecha |
| 2 | `=` `+=` … `&&=` `\|\|=` `??=` | asocia a la derecha: `a = b = c` |
| 1 | `,` | lo más bajo |

Gotchas de precedencia en la práctica:

```js
// && agrupa antes que ||:
console.log(false || true && true);   // true  ≡ false || (true && true)

// Los unarios pegan fuerte:
console.log(-3 ** 2);                 // SyntaxError — equivale a (-3) ** 2
console.log(2 * -3);                  // OK: 2 * (-3)

// 'in' y 'instanceof' están en el nivel relacional:
console.log('x' in { x: 1 } === true); // true — ('x' in obj) === true

// Asignación retorna el valor, y asocia a la derecha:
let t; const r = (t = 5);            // r = 5
```

> [!tip] Regla de oro de la precedencia
> Si tenés que **pensar** cómo se agrupa, escribí los paréntesis. Un paréntesis de más es gratis; uno de menos es un bug en producción. La precedencia completa oficial: <https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Operators/Operator_precedence>.

## Recetas practicas

```js
// Valores por defecto según "sin valor" (null/undefined):
const limite = config.limite ?? 10;

// Valores por defecto según truthiness (0 cuenta como vacío):
const intentos = cfg.intentos || 3;

// Toggle seguro
elemento.classList.toggle('activo');
estado.activo = !estado.activo;             // negación = booleano garantizado

// Guard clause con cortocircuito (solo si la función existe)
user.saludo?.();

// Máscara de flags
const READ = 4, WRITE = 2;
let permisos = READ | WRITE;
console.log((permisos & READ) === READ);    // true: tiene permiso READ
permisos &= ~WRITE;                         // quitar el flag WRITE

// Truncar decimal entero (legible, no bitwise)
console.log(Math.trunc(3.9));               // 3

// Detectar NaN de verdad
console.log(Number.isNaN(NaN));             // true;  Number.isNaN('a') → false

// Par / impar, ciclo de índice
console.log(7 % 2 === 0 ? 'par' : 'impar');
```

## Errores comunes

1. **`==` donde va `===`** — coerciones sorprendentes (`0 == ''` → `true`). `===` por defecto; `== null` solo para null/undefined.
2. **`||` para defaults con valores legítimos falsy** — `limite || 10` rompe con `limite = 0`. Usar `??`.
3. **Comparar con `NaN`** — `x === NaN` siempre `false`. `Number.isNaN(x)` o `Object.is`.
4. **Sumar en vez de concatenar** — `precio + precio` con strings duplica texto. `Number(precio)` o `+precio` antes de sumar (ver [[Números]]).
5. **Mezclar `??` con `||`/`&&` sin paréntesis** — `SyntaxError` real del parser.
6. **`-x ** 2` sin paréntesis** — `SyntaxError`; el unario no se pega a `**`.
7. **Asumir que bitwise es gratis y puro** — convierte a int32 (trunca decimales, `NaN` → 0, pierde bits más allá de 2³¹).
8. **Borrar índices de arreglos con `delete`** — deja huecos invisibles que rompen iteraciones; `splice` o `filter`.
9. **`if (x = 5)` en vez de `===`** — asigna y evalúa truthy (ver [[Estructuras de Control]]).
10. **`typeof null` esperando "null"** — nunca lo fue; comprobación explícita `=== null`.
11. **Confundir postfijo y prefijo en la misma expresión** — `n++ + ++n` es legal y enrevesado; separalo en dos sentencias.

## Cheatsheet

| Operador | Qué hace | Devuelve | Gotcha |
| -------- | -------- | -------- | ------ |
| `+` | suma o concatena | number/string | orden izq→der define el tipo final |
| `- * /` | aritmética | number | coerción numérica; `/` no trunca |
| `%` | resto | number | signo del primer operando |
| `**` | potencia | number | asociativo a la derecha; unario requiere `()` |
| `++ --` | incremento | number | postfijo devuelve el valor viejo |
| `= += … ??=` | asignar | valor asignado | compuestas heredan coerción del operador base |
| `=== !==` | igualdad estricta | boolean | sin coerción; `NaN !== NaN` |
| `== !=` | igualdad abstracta | boolean | algoritmo de coerción; solo `== null` es aceptable |
| `< > <= >=` | relacional | boolean | string-string = lexicográfico; con NaN → todo false |
| `&& \|\|` | lógicos | **operandos** | cortocircuitan |
| `!` | negación | boolean | `!!` = coerción a booleano |
| `??` | default nullish | operandos | solo null/undefined; no mezclar con `\|\|` |
| `?:` | ternario | expresión | asocia a la derecha |
| `,` | secuencia | último operando | precedencia mínima |
| `typeof` | tipo | string | 8 resultados; `null`→"object"; no lanza |
| `instanceof` | cadena prototípica | boolean | falla entre realms; inútil con primitivos |
| `in` | propiedad existe | boolean | incluye heredadas |
| `delete` | borra propiedad | boolean | `delete arr[i]` deja hueco |
| `void` | evalúa y… | `undefined` | `void 0` = undefined seguro |
| `new` | constructor | objeto | precedencia altísima; no con arrows |
| `& \| ^ ~ << >> >>>` | bitwise int32 | number | trunca a 32 bits; `NaN`\|0 = 0 |
| `await` | espera promesa | resolución | solo dentro de `async` |
| `yield` | cede a iterador | valor recibido | solo dentro de `function*` |

## Relacionadas

- [[Booleanos]] — truthy/falsy y cómo `!`/`&&`/`||` deciden
- [[Números]] — `NaN`, `Number`, precisión y conversiones
- [[Strings]] — la cara `+` de la concatenación
- [[Estructuras de Control]] — operadores lógicos dentro de `if`/ternario
- [[Funciones]] — `this`, `new.target` y la naturaleza de las funciones
- [[Objetos]] — prototipos detrás de `instanceof` e `in`
- [[Sintaxis Moderna]] — `?.`, `??`, asignación lógica y `**` en contexto
- [[Iteradores]] — `yield` y el protocolo iterable
- [[Uso de Java Script]] — `await` en el flujo real de `fetch`
