---
aliases: ["Strings", "Cadenas de texto en JavaScript", "Tipos de String"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Números]]"
  - "[[Booleanos]]"
  - "[[Funciones]]"
  - "[[Array Methods]]"
  - "[[Sintaxis Moderna]]"
  - "[[Operadores]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/String"
---

# Strings

Un **string** (*cadena de texto*) es una secuencia inmutable de caracteres que representa texto. Es el segundo tipo primitivo más usado después de `number`, y el que más métodos trae de serie: búsquedas, cortes, reemplazos, normalización Unicode y transformación de estructura. En JS, un string se almacena en **UTF-16**: cada posición guarda un *code unit* de 16 bits — ese detalle explica casi todos los traps con emojis y acentos de esta nota.

> [!info] Inmutable: el concepto que lo explica todo
> Ningún método de string "modifica" el string: todos devuelven **uno nuevo**. `str.trim()` sin guardar el retorno es un no-op silencioso (bug clásico). Mental model: los strings son valores, no contenedores — como los números: `const x = 5; x++;` no toca el 5 original, y con strings pasa igual.

```js
const producto = '          Monitor de 20 pulgadas               ';
producto.trim();     // no hace nada visible: el resultado se descarta
const limpio = producto.trim(); // así SÍ
```

## Definir strings

```js
// Definir Strings
const producto = "Monitor de 20 pulgadas";
const producto2 = String("Monitor de 24 Pulgadas");
const producto3 = new String("Monitor de 27 pulgadas");

console.log(producto);
console.log(producto2);
console.log(producto3);
```

| Forma | Ejemplo | Notas |
|---|---|---|
| Literal comillas dobles | `"hola"` | estándar; admite `'` dentro sin escapar |
| Literal comillas simples | `'hola'` | admite `"` dentro; estilo moderno preferido |
| Template literal | `` `hola ${nombre}` `` | interpolación + multilinea (ES6) |
| `String(x)` | `String(20)` | conversión explícita, seguro |
| `new String(x)` | `new String('20')` | **objeto**, no string — evitar siempre |

```js
console.log(typeof String(1));       // "string"
console.log(typeof new String(1));   // "object" ← la trampa de new: compara raro y es truthy siempre
```

### Escapes y caracteres especiales

```js
// Representar caracteres escapando caracteres
const productos = "\"Hola mundo entre comillas\"";
console.log(productos); // "Hola mundo entre comillas"

const salto = 'Primera\nsegunda línea';   // \n salto de línea
const tab = 'col1\tcol2';                 // \t tabulador
const ruta = 'C:\\Users\\docs';           // \\ barra invertida literal
```

| Escape | Produce | Escape | Produce |
|---|---|---|---|
| `\n` | salto de línea | `\\` | `\` literal |
| `\t` | tabulador | `\'` | `'` literal |
| `\r` | retorno de carro | `\"` | `"` literal |
| `\u{1F600}` | carácter Unicode | `` \` `` | `` ` `` en template |

Con comilla simple `'...'` ni hace falta escapar `"` (y viceversa) — una razón más para elegir un solo estilo de comillas por proyecto.

## Propiedades: `length`

```js
console.log(producto.length); // número de CODE UNITS, no de "letras"
```

> [!warning] `length` cuenta code units UTF-16, no caracteres
> `'ñ'.length` → `1` y `'👍'.length` → `2` (verificado): un emoji ocupa **dos** posiciones (*surrogate pair*, ver sección Unicode). `'👩‍💻'.length` → `5` (dos emojis + el joiner). Para contar "lo que el usuario ve", iterá por code points: `[...str].length`.

## Búsqueda: dónde está un texto

```js
// nos permite saber dónde comienza la palabra y si no la encuentra se -1
console.log(producto.indexOf('pulgadas'));

// Nos permite verificar la existencia de una palabra, devuelve un true o false
console.log(producto.includes('Tablet'));
console.log(producto.includes('Monitor'));
console.log(producto.includes('monitor'));
```

| Método | Retorna | Notas |
|---|---|---|
| `indexOf(búsqueda, desde)` | índice o `-1` | case-sensitive; acepta desdeIndex |
| `lastIndexOf(búsqueda, desde)` | índice del ÚLTIMO o `-1` | busca de derecha a izquierda |
| `includes(búsqueda, desde)` | `boolean` | ES2015; case-sensitive |
| `startsWith(prefijo, desde)` | `boolean` | |
| `endsWith(sufijo, largo)` | `boolean` | el 2º arg es **longitud total** a considerar, no índice inicial |
| `at(indice)` | carácter o `undefined` | índices negativos: `at(-1)` es el último (verificado) |

```js
const usuario = 'Juan';
console.log(usuario.startsWith('Ju'));  // true
console.log('archivo-final.md'.endsWith('.md')); // true
console.log('hola'.at(-2));            // "l" (verificado)
console.log('hola'.at(99));            // undefined
```

> [!tip] `indexOf !== -1` vs `includes`
> `str.includes()` es más legible y devuelve booleano directo; `indexOf` sigue siendo útil cuando necesitás **la posición** para cortar. Nunca compares el resultado de `includes` contra `-1` ni uses `indexOf` como booleano sin `!== -1`.

## Extracción: `slice`, `substring` y el legacy `substr`

```js
// Cortar palabras: el texto que se encuentra en el rango definido
console.log(producto.slice(0, 10));
console.log(producto.slice(8)); // Con solo ese valor muestra todo a partir del mismo
console.log(producto.slice(2, 1)); // Si el primer valor es mayor al segundo no muestra nada
```

| Método | `slice(2, 1)` | Negativos | Estado |
|---|---|---|---|
| `slice(inicio, fin)` | `""` (rango vacío) | sí: `-1` = desde el final | **preferido** — idéntico a `Array.prototype.slice` |
| `substring(a, b)` | `"b"` (¡invierte los args!) | no (ignora negativos) | legado — evitable |
| `substr(desde, largo)` | `""` | desde negativo sí | **deprecated** (annex B); nunca usar en código nuevo |

Verificado: `'abc'.substring(2, 1)` → `"b"` (ordena los argumentos) mientras `'abc'.slice(2, 1)` → `""` (rango invertido = vacío). `substring` además intercambia `''` con `0` y trata negativos como 0 — semántica heredada de Netscape. **Usá `slice` siempre**: es consistente con el `slice` de [[Arreglos|los arreglos]], acepta negativos y tiene la semántica predecible de rangos `[inicio, fin)`.

## Modificación de contenido

### Recorte de espacios: `trim*`

```js
const conEspacios = '          Monitor de 20 pulgadas               ';

console.log(conEspacios.trimStart());  // Eliminamos los espacios al comienzo del string.
console.log(conEspacios.trimEnd());    // Elimina los espacios al final del string.

console.log(conEspacios.trimStart().trimEnd()); // Elimina los espacios del inicio y final.

console.log(conEspacios.trim()); // Elimina los espacios del inicio y final.
```

`trim()` elimina whitespace según el spec: espacios, tabs (`\t`), saltos (`\n`, `\r`), y Unicode spaces (NBSP `\u00A0`, ideographic space…). Existen también `trimStart`/`trimEnd` (ES2019) — el antiguo `trimLeft`/`trimRight` son alias no estándar.

### Mayúsculas y minúsculas

```js
console.log(producto.toUpperCase()); // Nos permite transformar todo a mayúsculas
console.log(producto.toLowerCase()); // Nos permite transformar todo a minúsculas
```

Punto clave para búsquedas case-insensitive: ambos **no son Unicode-perfectos** para todos los scripts (Turco: `'I'.toLowerCase()` → `'i'` sin punto, no `'ı'`). Para comparar sin distinguir mayúsculas: `a.toLowerCase() === b.toLowerCase()`, o `localeCompare` con `sensitivity: 'accent'/'case'`.

### Repetir: `repeat`

```js
// Repetir una cadena de texto
const texto = ' en Promoción'.repeat(3); // si el valor no es entero se redondea (hacia abajo)

console.log(texto);
console.log(`${producto} ${texto}`);
```

Verificado: `'ab'.repeat(2.7)` → `'abab'` (trunca hacia abajo, no redondea). `repeat(-1)` lanza `RangeError`; `repeat(Infinity)` también.

### Reemplazar: `replace` y `replaceAll`

```js
console.log(producto.replace('Pulgadas', '"'));    // reemplaza SOLO la primera aparición
console.log(producto.replace('Monitor', 'Monitor Curvo'));
console.log('a-a-a'.replaceAll('-', '+'));         // "a+a+a" (ES2021, todas las apariciones)
```

> [!warning] `replace` con string reemplaza solo la **primera** ocurrencia
> Síntoma clásico: un `replace` de comillas o guiones "mitad del trabajo". Con `replaceAll` no hay sorpresas. Si necesitás regex, el patrón **debe** llevar bandera `g` (con `replaceAll(/x/g, y)`; sin `g` lanza `TypeError`).

```js
console.log('a-a'.replaceAll(/-/g, '+')); // regex + bandera g obligatoria
console.log('2025-01-25'.replace(/(\d{4})-(\d{2})-(\d{2})/, '$3/$2/$1')); // "25/01/2025" — grupos de captura
```

Ver [[Estructuras de Control|Regex]] se usa en `match`, `matchAll`, `replace`, `split` y `search`; la referencia completa de patrones vive en MDN (grupo `$1`, lookahead, banderas `g/i/m/u/y`).

### Alinear con relleno: `padStart` / `padEnd`

```js
console.log('5'.padStart(3, '0'));      // "005" (verificado) — números con ceros a la izquierda
console.log('ok'.padEnd(10, '.'));      // "ok........" — reportes alineados
console.log('abc'.padStart(10));        // "       abc" (relleno por defecto: espacio)
```

El primer argumento es la **longitud final** deseada, no cuántos agregar. Si el string ya es más largo, se devuelve sin cambios.

## Estructura: `split`, `join`, `concat`

```js
// Podemos dividir un string
const actividad = 'Estoy aprendiendo JavaScript Moderno';
console.log(actividad.split(' ')); // Crea un arreglo de palabras divididas por el espacio

const hobbies = 'Leer, Caminar, Escuchar Musica, Programar, Arreglar';
console.log(hobbies.split(','));

console.log('a b c d'.split(' ', 2)); // ['a','b'] — el 2º arg limita las partes
```

| Método | Dirección | Ejemplo |
|---|---|---|
| `split(sep, límite)` | string → **arreglo** | `'a,b'.split(',')` → `['a','b']`; `split('')` → letras |
| `join(sep)` | arreglo → **string** | `['a','b'].join('-')` → `'a-b'` (ver [[Array Methods]]) |
| `concat(...)` | une strings | `'a'.concat('b','c')` → `'abc'`; el `+` hace lo mismo |

```js
// Podemos unir dos strings
console.log(producto.concat(precio));
console.log(producto.concat('Descuentos'));
console.log(producto + precio);
console.log(producto + 'Con un precio de: ' + precio);
console.log('El producto ' + producto + ' tiene un precio de: ' + precio);
console.log('El producto', producto, 'Tiene un precio de ', precio); // console.log: espacios automáticos
```

Detalle: `join(undefined)` usa `,` como separador; `join()` sin argumentos también. `split('')` parte por code **units** — con emojis produce mitades rotas; preferir `[...str]`.

## Template literals: interpolación, multilinea y etiquetados

```js
// Template String, la más usada y recomendada
console.log(`${producto} tiene un precio de ${precio}`);
```

Capacidades completas del template literal (backtick):

```js
const nombre = 'Ana';
const multi = `línea 1
línea 2`;                       // multilinea real: los saltos se conservan
const expresion = `suma: ${2 + 3 * 2}`;   // cualquier expresión, no solo variables
const anidado = `Hola ${nombre ? nombre : 'Invitado'}`;
```

### Tagged templates: funciones que consumen el template

Una función "etiquetada" recibe el template separado en **partes literales** (congeladas) y los valores interpolados — el patrón por defecto para escaping seguro, i18n y estilos CSS-in-JS:

```js
function traducir(strings, ...valores) {
  // strings.raw une los literales tal cual
  return strings.reduce((acc, parte, i) =>
    acc + parte + (valores[i] ?? ''), '');
}
const nombre = 'Ana';
traducir`Hola ${nombre}, bienvenido`; // "Hola Ana, bienvenido"

// String.raw: literales sin interpretar escapes (raw = crudo)
console.log(String.raw`a\nb`);        // "a\nb" (literal: backslash + n, verificado)
console.log(String.raw`C:\nuevos\archivos`);
```

Usos reales: `String.raw` para rutas/regex literales; tagged templates para **escapar HTML** (no interpolar entrada de usuario cruda en el DOM — ver [[Document Object Model (DOM)|DOM]] y seguridad XSS), y para obtener offsets de los literales (highlighters, parsers).

## Iteración y Unicode: lo que importa con emojis

### Recorrer caracteres

```js
const emoji = '👍';

console.log([...emoji].length);   // 1 — SPREAD itera por CODE POINTS (verificado)
for (const c of emoji) console.log(c);  // 👍 completo — igual que spread
console.log(emoji.length);        // 2 — code UNITS (el trap)
console.log(emoji.codePointAt(0)); // 128513 — el punto de código real
```

| Forma | Itera por | Emoji seguro |
|---|---|---|
| `for (let i = 0; i < s.length; i++) s[i]` | code units | �️ parte los emoji |
| `for (const c of s)` / `[...s]` | code points | ✅ |
| `Array.from(s)` | code points | ✅ |

### Surrogate pairs: por qué `'👍'.length` es 2

UTF-16 representa los ~1.1 millones de puntos de código de Unicode con espacios de 16 bits; los que quedan fuera del BMP (emojis, kanji raros, matemáticos) se codifican como **par de sustitución**: dos code units (0xD800–0xDBFF + 0xDC00–0xDFFF) que juntos forman un carácter. Un emoji ocupa 2 posiciones:

```js
console.log('👍'.length);           // 2 (verificado)
console.log('👍'.codePointAt(0));   // 128513 (el code point real del emoji)
console.log('👩‍💻'.length);          // 5 — dos emoji unidos por ZWJ: 2 + 1 + 2
console.log([...'👩‍💻'].length);     // 3 — la secuencia ZWJ "rompe" la regla: aun spread cuenta 3
```

> [!warning] El emoji "de familia/profesión" ni con spread se cuenta bien
> `'👩‍💻'.length` da 5 y `[...'👩‍💻'].length` da 3, pero **el usuario ve 1 símbolo**. Para contar unidades graphémicas reales se necesita `Intl.Segmenter` (ES2022): `new Intl.Segmenter('es', { granularity: 'grapheme' })`. Afecta a: contadores de caracteres con límite, truncar textos con emoji (cortar por índice puede dejar un emoji roto y mostrar �), y validaciones de longitud.

### Normalización: `normalize`

Dos Unicode distintos pueden **verse igual**: `'é'` como code point único (`U+00E9`) o como `e` + acento combinante (`U+0065 U+0301`). `normalize` los unifica a una forma estándar:

```js
console.log('é'.normalize('NFC') === 'é'); // true — ambas formas comparan iguales tras NFC
console.log('ñ'.normalize('NFD').length);  // 2 (verificado): n + tilde separados
```

| Forma | Qué hace | Cuándo |
|---|---|---|
| `NFC` (default) | compone: un solo code point | comparar, guardar, buscar |
| `NFD` | descompone: base + diacríticos | análisis lingüístico, búsquedas sin acentos |

Patrón de búsqueda "sin acentos": `str.normalize('NFD').replace(/\p{M}/gu, '')` quita los marcas combinantes.

### Comparar strings: `localeCompare`

```js
console.log('a'.localeCompare('B'));   // -1 (verificado): ordena por ORDEN LOCAL, no por código
console.log('á'.localeCompare('b'));   // -1 — en español 'á' va junto a 'a', no después de 'z'
console.log('a'.localeCompare('b', 'es', { sensitivity: 'accent' }));
```

El `<`/`>` de JS compara por code point (`'B'(66) < 'a'(97)`): sirve para sort técnico, pero **no para ordenar listas de texto humano**. `localeCompare` respeta la cultura (mayúsculas, acentos, ñ) y es el correcto para tablas, selects y reportes. Ver [[Array Methods]] para usarlo como comparador de `sort`.

## Coerción: de cualquier cosa a string

```js
const precio = 300;
console.log(precio.toString()); // Nos permite convertir una variable numérica a String
console.log(String(precio));    // "300" — forma segura (no falla con null/undefined)
console.log(`${precio}`);       // "300" — template literal: la forma más idiomática
console.log('precio: ' + precio); // concatenación con + (ver [[Operadores]])
```

| Forma | Comportamiento |
|---|---|
| `` `${x}` `` | llama a `toString`/`Symbol.toPrimitive`; seguro con todo |
| `String(x)` | lo mismo, pero invoca `toString` con **this = undefined** (funciona con números primitivos) |
| `x.toString()` | falla si `x` es `null`/`undefined` (no tienen método) |
| `'' + x` | coerción por `+`; con `Symbol` lanza TypeError |
| `new String(x)` | objeto — nunca (truthy incluso con `''`, compara por referencia) |

Para **JSON** ver [[Formato Json]]: `JSON.stringify` serializa strings con comillas y escapes propios.

## Rendimiento: `+` vs template vs `concat` vs `join`

- Los strings son inmutables: **cada concatenación crea un string nuevo**. En bucles grandes, el patrón clásico `s += x` era cuadrático en engines antiguos; los motores actuales (V8/SpiderMonkey) lo optimizan con *ropes* (árboles de concat perezosos), así que `+=` en bucle es hoy perfectamente aceptable.
- **`array.join('')`** era el consejo de la era pre-2015 para concatenar muchas piezas; sigue siendo útil cuando ya tenés las piezas en un arreglo (por ejemplo, tras un `map`).
- Los **template literals** compilan a concat bajo el hood: rendimiento idéntico al `+`. Elegí por legibilidad, no por velocidad.
- Donde sí se nota costo: `repeat` de strings enormes, `split('')` en textos largos (crea N strings), y `normalize` sobre textos grandes — úsalos al presentar/procesar, no en bucles calientes sin necesidad.

## Recetas prácticas

```js
// Capitalizar la primera letra
const capitalizar = (s) => s.charAt(0).toUpperCase() + s.slice(1).toLowerCase();
capitalizar('dANIEL'); // "Daniel"

// Contar ocurrencias de una subcadena (sin regex)
const contar = (s, sub) => s.split(sub).length - 1;
contar('a-b-c', '-'); // 2

// "Slug" para URLs
const slug = (s) => s.toLowerCase().normalize('NFD').replace(/\p{M}/gu, '').replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
slug('¡Hola, Café!'); // "hola-cafe"

// Truncar con elipsis sin cortar emoji a la mitad
const truncar = (s, max) => [...s].length <= max ? s : [...s].slice(0, max).join('') + '…';

// Invertir un string (respetando code points)
const invertir = (s) => [...s].reverse().join('');
invertir('hola'); // "aloh"
```

## Errores comunes

1. **Descartar el resultado de un método**: `str.trim()` sin asignar no cambia `str` (inmutabilidad). Siempre: `const limpio = str.trim()`.
2. **`indexOf`/`includes` case-sensitivity**: `'Monitor'.includes('monitor')` → `false`. Normalizá con `toLowerCase()` en ambos lados antes de comparar.
3. **Confundir `endsWith(s, largo)` con un índice inicial**: el segundo parámetro es **longitud total**, no "empezar desde".
4. **`substring` vs `slice` con args invertidos**: `substring(2,1)` invierte; `slice(2,1)` da vacío. Usá `slice` y no hay sorpresas.
5. **`length` con emoji**: contar o truncar por índice parte los pares surrogate y deja caracteres rotos (`�`). Usá `[...str]` o `Intl.Segmenter`.
6. **`split('')` en texto con emoji**: idéntico problema; `[...str]` es el split seguro.
7. **`replace` que "no reemplaza todo"**: solo cambia la primera ocurrencia — `replaceAll` o regex con `/g`.
8. **`replaceAll` con regex sin `/g`**: lanza `TypeError: replaceAll must be called with a global RegExp`.
9. **Comparar con `==` un string numérico**: `'1' == 1` pasa por coerción; en entradas de formulario compará con `===` o convertí con `Number` (ver [[Booleanos]]).
10. **`new String()`**: crea objeto truthy y compara por referencia — `new String('a') === 'a'` es `false`. Nunca usar.

## Cheatsheet

| Categoría | Métodos | Nota clave |
|---|---|---|
| Longitud | `length` | code units, no caracteres (emoji = 2) |
| Búsqueda | `indexOf`, `lastIndexOf`, `includes`, `startsWith`, `endsWith`, `at` | todos case-sensitive; `at(-1)` = último |
| Extracción | `slice`, `substring` (legacy) | usar `slice`: negativos y rangos consistentes |
| Recorte | `trim`, `trimStart`, `trimEnd` | whitespace Unicode incluido |
| Case | `toUpperCase`, `toLowerCase` | no perfectos para todos los locales |
| Relleno | `padStart`, `padLongitud` | 1er arg = longitud final deseada |
| Repetir | `repeat(n)` | n fraccionario trunca; negativo lanza RangeError |
| Reemplazo | `replace` (1ª), `replaceAll` (todas) | regex con `replaceAll` exige `/g` |
| Estructura | `split`, `join`, `concat` | `split` → array; `join` es su inverso |
| Unicode | `[...s]`, `for...of`, `codePointAt`, `normalize` | spread itera por code points |
| Comparación | `localeCompare` | orden local real; `<` ordena por código |
| Interpolación | `` `text ${expr}` `` | multilinea + expresiones + tagged templates |
| Tagged | `fn\`...\``, `String.raw` | partes crudas; escaping seguro |
| Coerción | `String(x)`, `` `${x}` ``, `x.toString()` | `String()` seguro con null/undefined |

## Relacionadas

- [[Números]] — conversión entre texto y números (`parseInt`, `Number`, `toFixed`)
- [[Booleanos]] — por qué `'0'` es truthy y el string vacío no
- [[Sintaxis Moderna]] — template literals, `String.raw`, `replaceAll`
- [[Operadores]] — `+` como concatenación y coerción de tipos
- [[Array Methods]] — `split`/`join` como puente entre string y arreglo
- [[Funciones]] — tagged templates como funciones de orden superior
- [[Formato Json]] — serializar strings con escaping propio de JSON
- [[Document Object Model (DOM)|DOM]] — insertar texto seguro (evitar `innerHTML` con datos crudos)
