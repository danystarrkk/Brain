---
aliases: ["Objetos", "Objetos en JavaScript", "Object"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Java Script]]"
  - "[[Funciones]]"
  - "[[Sintaxis Moderna]]"
  - "[[Iteradores]]"
  - "[[Formato Json]]"
  - "[[Arreglos]]"
  - "[[Strings]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Object"
---

# Objetos

Un **objeto** es el tipo de referencia central de JavaScript: una colección de **pares clave/valor** donde los valores pueden ser de cualquier tipo (números, strings, [[Arreglos|arreglos]], funciones, otros objetos). A diferencia de los tipos primitivos ([[Números]], [[Strings]], [[Booleanos]]), los objetos se pasan **por referencia**: dos variables pueden apuntar al mismo objeto en memoria.

```js
const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true,
};

console.log(producto.nombre);        // "Monitor de 20 pulgadas" (sintaxis de punto)
console.log(producto["precio"]);     // 300 (sintaxis de corchetes)
```

> [!info] Por qué todo es objeto
> Incluso `new String("hola")` y `new Number(5)` son objetos con métodos (`length`, `toFixed`) — los primitivos se "auto-boxean" al acceder a una propiedad. Los [[Arreglos]] son objetos con claves numéricas, las funciones son objetos con una propiedad especial que las hace llamables, y la herencia se resuelve con el **encadenamiento de prototipos**, explicado más abajo.

## Formas de creación

| Forma | Ejemplo | Cuándo usarla |
| ----- | ------- | ------------- |
| Literal | `const p = { nombre: "X" }` | El 95% de los casos |
| Constructor | `function Producto(n) { this.nombre = n }` + `new Producto("X")` | Crear **muchas instancias** con la misma forma (métodos compartidos vía `prototype`) |
| `Object.create(proto)` | `const p = Object.create(proto)` | Herencia explícita sin constructor; objetos con prototipo custom |
| `new Object()` | `const p = new Object()` | Equivale al literal — no aporta nada, evítalo |
| Clase | `class Producto { constructor(n) {...} }` | Azúcar sobre constructor + `prototype` — ver [[Funciones]] |
| `Object.assign` / spread | `const copia = { ...otro }` | Copiar o combinar propiedades existentes |

### Object literal y accesos

```js
const nombre = "Monitor de 20 pulgadas";
const precio = 300;
const disponible = true;

// Un objeto agrupa todo en una sola variable
const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true,
};

console.log(producto.nombre);     // acceder por punto
console.log(producto["precio"]);  // acceder por corchetes

// Agregar nuevas propiedades al objeto
producto.imagen = "imagen.jpg";

// Eliminar una propiedad (devuelve true si la borró)
delete producto.disponible;

console.log(producto);
```

| Sintaxis | Acepta | Notas |
| -------- | ------ | ----- |
| `obj.propiedad` | identificadores válidos | Falla con guiones (`obj.min-height` es resta), espacios o claves dinámicas |
| `obj["min-height"]` | **cualquier string** | Única opción para claves con guiones, espacios o computadas |
| `obj[expr]` | cualquier expresión | Claves dinámicas: `obj[clave]`, `obj[3 + 1]` |

> [!tip] Claves computadas en el literal
> Desde ES6, el literal admite expresiones entre corchetes: `obj[clave] = v` se escribe `{ [clave]: v }`. Es la base de patrones como `{ ['get' + Nombre]: fn }`.

### Constructor clásico para múltiples instancias

```js
// Object Constructor: optimiza la creación de objetos múltiples
function Producto(nombre, precio) {
  this.nombre = nombre;
  this.precio = precio;
  this.disponible = true;
}

const producto2 = new Producto("Monitor de 24 Pulgadas", 500);
const producto3 = new Producto("Audífonos Logitech", 100);
```

Cada `new` crea un objeto cuyo `[[Prototype]]` es `Producto.prototype`; los métodos que agregues a `Producto.prototype` se comparten entre todas las instancias (una sola copia en memoria) — la razón de su "optimización". Ver "Cadena de prototipos" más abajo.

### `Object.create` y el patrón de prototipo

```js
const proto = { saludar() { return `Soy ${this.nombre}`; } };
const obj = Object.create(proto);
obj.nombre = "Ana";
console.log(obj.saludar());                    // "Soy Ana"
console.log(Object.getPrototypeOf(obj) === proto); // true
```

## Destructuring de objetos

Extrae propiedades a variables de un solo golpe — el detalle completo está en [[Sintaxis Moderna]]; este es el repertorio:

```js
const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true,
  informacion: {
    medidas: { peso: "1kg", medida: "1m" },
    fabricacion: { pais: "china" },
  },
};

// Destructuring simple
const { nombre } = producto;
console.log(nombre); // "Monitor de 20 pulgadas"

// Múltiples valores
const { precio, disponible } = producto;
console.log(precio, disponible); // 300 true

// Anidado
console.log(producto.informacion.medidas);
console.log(producto.informacion.fabricacion.pais); // "china"

const { informacion, informacion: { fabricacion: { pais } } } = producto;
console.log(informacion, pais); // objeto completo, "china"

// Renombrar, defaults y resto en un solo patrón
const {
  nombre: nom,            // renombra la variable
  categoria = "general",  // default si la clave no existe
  informacion: { medidas: { peso } }, // destructuring profundo
  ...resto,               // resto: todo lo demás en un objeto nuevo
} = producto;
console.log(nom, categoria, peso, resto); // ...  "general" "1kg" { precio, disponible, informacion }

// Destructuring como parámetro de función (muy común en APIs y React)
function mostrarPrecio({ nombre, precio }) {
  console.log(`${nombre}: $${precio}`);
}
mostrarPrecio(producto);
```

> [!warning] `const` no inmuta el objeto
> Las **propiedades** de un objeto sí se pueden eliminar o modificar, incluso si la variable es `const`: `const` congela la **referencia** (no podés reasignar `producto = {...}`), no el contenido. `producto.disponible = false` funciona perfectamente. Para bloquear el contenido usá `Object.freeze` — ver la tabla de inmutabilidad.

## Cadena de prototipos (`[[Prototype]]`)

Todo objeto tiene un enlace interno a otro objeto — su **prototipo** — del cual *hereda* propiedades. La búsqueda de una propiedad **baja por la cadena** hasta encontrarla o llegar a `null`:

```
perro ──[[Prototype]]──▶ Animal.prototype ──[[Prototype]]──▶ Object.prototype ──▶ null
```

```js
function Animal(tipo) { this.tipo = tipo; }
Animal.prototype.saluda = function () { return `soy ${this.tipo}`; };

const perro = new Animal("perro");

console.log(Object.getPrototypeOf(perro) === Animal.prototype); // true
console.log(perro.__proto__ === Animal.prototype);              // true (azúcar)
console.log(Animal.prototype.__proto__ === Object.prototype);   // true
console.log(perro.saluda());        // "soy perro" — saluda vive en Animal.prototype
console.log(perro.hasOwnProperty("saluda"));         // false: es heredada
console.log(Animal.prototype.hasOwnProperty("saluda")); // true: vive ahí
console.log(perro instanceof Animal); // true: Animal.prototype está en la cadena
console.log(perro.constructor === Animal); // true: prototype guarda el constructor
```

Las tres caras de la misma moneda — no confundirlas:

| Concepto | Dónde vive | Qué es |
| -------- | ---------- | ------ |
| `[[Prototype]]` | enlace **interno** de cada objeto | La cadena real; se lee con `Object.getPrototypeOf(obj)` |
| `obj.__proto__` | getter/setter heredado de `Object.prototype` | Azúcar legacy para leer/escribir el enlace (uso moderno: `Object.getPrototypeOf`) |
| `Fn.prototype` | propiedad de la **función** | Objeto que servirá de prototipo de las instancias creadas con `new Fn()` |
| `Object.prototype` | raíz de casi todas las cadenas | Da a todos los métodos universales (`hasOwnProperty`, `toString`) |

> [!warning] Nunca mutar `Object.prototype`
> Agregar propiedades a `Object.prototype` (o a `Array.prototype`) las "filtra" a **todos** los objetos del programa — bug fuente de colisiones imposibles de rastrear. Extiende solo tus propios prototipos, o mejor: componé.

> [!tip] Encadenamiento como fallback
> El prototipo es el **fallback**: `perro.tipo` se resuelve en la instancia; `perro.saluda` no existe ahí, así que se busca en `Animal.prototype`. Si nada la tiene, `undefined` (o `TypeError` al invocar). Este mismo mecanismo es el que usan los getters de eventos y el de los *web components*.

## Propiedades y descriptores: la verdad detrás de las claves

Cada propiedad no es "un par clave/valor": es un **descriptor** — un paquete de atributos que el motor respeta al pie de la letra:

```js
const o = {};
Object.defineProperty(o, "x", { value: 42 });
console.log(Object.getOwnPropertyDescriptor(o, "x"));
// { value: 42, writable: false, enumerable: false, configurable: false }
//          ↑ defaults de defineProperty: todo apagado salvo value
```

Defaults **verificados** al usar `defineProperty` (¡distintos de los del literal!):

| Atributo | Literal `{x: 1}` | `defineProperty` |
| -------- | ---------------- | ---------------- |
| `value` | el valor | el valor |
| `writable` | `true` | **`false`** |
| `enumerable` | `true` | **`false`** |
| `configurable` | `true` | **`false`** |

### Los tres atributos y qué gobiernan

| Atributo | Controla | `false` significa |
| -------- | -------- | ----------------- |
| `writable` | si el valor se puede **asignar** (`o.x = ...`) | asignación ignorada (silenciosa en sloppy; `TypeError` en `"use strict"`) |
| `enumerable` | si aparece en `Object.keys`, `for...in`, `JSON.stringify` | invisible a esos recorridos (pero accesible: `o.x` funciona) |
| `configurable` | si se puede **borrar** (`delete`) o **redefinir** el descriptor | `delete` falla; no se puede cambiar ningún atributo (ni `writable`, que solo puede pasar `true→false`) |

```js
const oculta = {};
Object.defineProperty(oculta, "interna", { value: 1, enumerable: false });
console.log(Object.keys(oculta));        // [] — invisible
console.log(oculta.interna);             // 1 — pero accesible
console.log(JSON.stringify(oculta));     // "{}" — stringify respeta enumerable
console.log(delete oculta.interna);      // false — configurable: false lo impide
```

### Getters y setters (descriptores de acceso)

Las propiedades pueden definirse como **acceso** (con funciones get/set) en vez de como **datos**:

```js
const persona = {
  nombre: "Ana",
  get mayus() { return this.nombre.toUpperCase(); },   // se lee como propiedad
  set mayus(v) { this.nombre = v.toLowerCase(); },     // se escribe como propiedad
};

console.log(persona.mayus);   // "ANA" — getter, sin paréntesis
persona.mayus = "CARLOS";     // setter
console.log(persona.nombre);  // "carlos"

// forma equivalente con defineProperty (permite defaults de descriptor)
Object.defineProperty(persona, "edad", {
  get() { return this._edad; },
  set(v) { if (v < 0) throw new RangeError("edad negativa"); this._edad = v; },
  enumerable: true,
  configurable: true,
});
```

Casos de uso: valores **calculados al vuelo** (precio con impuesto), **validación** al asignar, **sincronización** (actualizar una vista cuando cambia el dato). Advertencias: los getters se ejecutan en cada lectura (no caches si el fondo cambia) y `JSON.stringify` **invoca los getters** y serializa su resultado — un getter que lanza excepción rompe el stringify.

`Object.getOwnPropertyNames(obj)` lista **todas** las propias, incluidas las no enumerables y los getters (verificado: `['nombre', 'mayus']` para el objeto anterior); `Object.keys` solo las enumerables.

## Catálogo de métodos de objeto

### Propias vs heredadas: `hasOwn` vs `hasOwnProperty`

```js
const proto = { heredado: true };
const obj = Object.create(proto);
obj.propio = 1;

console.log(obj.hasOwnProperty("propio"));    // true
console.log(obj.hasOwnProperty("heredado"));  // false — es heredada
console.log(Object.hasOwn(obj, "propio"));    // true — API moderna (ES2022)
console.log("toString" in obj);               // true — `in` también ve heredadas
console.log(obj.hasOwnProperty("toString"));   // false
```

| Pregunta | Herramienta |
| -------- | ----------- |
| ¿Es **propia** esta clave? | `Object.hasOwn(obj, k)` (moderno) / `obj.hasOwnProperty(k)` (legacy: falla si alguien hizo `obj = {hasOwnProperty: 1}` o si `k` es `"hasOwnProperty"`) |
| ¿Existe (propia o heredada)? | `k in obj` |
| ¿Existe con valor definido? | `obj[k] !== undefined` (con cuidado: la propiedad puede existir valiendo `undefined`) |
| ¿Fue borrada? | `!("k" in obj)` o `obj.k === undefined && !Object.hasOwn(obj, "k")` |

### Claves y valores

```js
const carro = { a: 1, b: 2 };
Object.keys(carro);    // ["a", "b"] — solo enumerables propias, strings
Object.values(carro);  // [1, 2]
Object.entries(carro); // [["a",1], ["b",2]] — ideal para for...of + destructuring
Object.getOwnPropertySymbols(carro); // símbolos: invisibles para keys/entries
```

**Orden garantizado** de `Object.keys` (verificado): primero los índices enteros positivos **en orden numérico ascendente** (no de inserción), después claves de texto en orden de inserución; las propiedades no enumerables y los símbolos quedan fuera.

### Combinar y copiar: `assign` vs spread

```js
const producto = { nombre: "Monitor", precio: 300, disponible: true };
const medidas = { peso: "1kg", medida: "1m" };

// Unir o combinar dos objetos (MUTA el primer argumento)
const resultado = Object.assign(producto, medidas);
console.log(resultado === producto); // true — es el mismo objeto, mutado

// Spread / rest: copia nueva, sin mutar el original
const resultado2 = { ...producto, ...medidas };
console.log(resultado2);
```

> [!warning] Ambas copias son **superficiales** — verificado
> `Object.assign({}, src)` y `{ ...src }` copian **referencias**: al modificar el objeto anidado de la copia, cambia el del original.
> ```js
> const src = { n: { v: 1 } };
> const copia = { ...src };
> copia.n.v = 111;
> console.log(src.n.v); // 111 — ¡compartido!
> ```
> Copia profunda real: `structuredClone(obj)` (nativo, copia Date/Map/Set y respeta referencias circulares) o `JSON.parse(JSON.stringify(obj))` con sus trampas (ver [[Formato Json]]).

Diferencias finas entre ambos:

| | `Object.assign(target, ...fuentes)` | `{ ...obj }` |
| --- | --- | --- |
| Mutación del destino | **Sí** (muta `target`) | No (objeto nuevo) |
| Setters del prototipo destino | Sí, dispara setters al copiar | No (copia propias enumerables como datos) |
| Sintaxis | Llamada; acepta cualquier número de fuentes | Literal; repite el spread |
| Propiedades no enumerables/símbolos | `assign` copia **símbolos** propios enumerables | spread también copia símbolos enumerables |

### Inmutabilidad: `freeze`, `seal` y compañía — semántica verificada

| Operación | `writable` | `configurable` (borrar/redefinir) | Agregar nuevas |
| --------- | ---------- | --------------------------------- | -------------- |
| `Object.freeze(o)` | → `false` | → `false` | Bloqueado |
| `Object.seal(o)` | **se conserva** | → `false` | Bloqueado |
| `Object.preventExtensions(o)` | intacto | intacto | Bloqueado |
| Estado puro | — | — | — |

```js
"use strict"; // en modo estricto las violaciones lanzan TypeError

const producto = { nombre: "Monitor", precio: 300, disponible: true };

Object.freeze(producto);
console.log(Object.isFrozen(producto)); // true
producto.precio = 1;                    // ❌ TypeError (strict) / silencioso (sloppy)
console.log(producto.precio);           // 300 — el valor no cambió

const sealado = Object.seal({ a: 1, b: 2 });
sealado.a = 5;            // ✅ writable sigue activo
delete sealado.b;         // ❌ false — configurable: false lo impide
sealado.c = 3;            // ❌ no se agrega (preventExtensions implícito)
console.log(sealado);                  // { a: 5, b: 2 }
console.log(Object.isSealed(sealado)); // true
console.log(Object.isFrozen(sealado)); // false — writable sigue true
```

Verificado además: `Object.isFrozen(Object.freeze({a:1}))` → descriptor `{value:1, writable:false, enumerable:true, configurable:false}` — `freeze` **no toca `enumerable`**.

> [!warning] `freeze`/`seal` son **superficiales** — verificado
> Congelan el objeto dado, **no los objetos anidados**:
> ```js
> const nest = Object.freeze({ inner: { v: 1 } });
> nest.inner.v = 99;
> console.log(nest.inner.v); // 99 — el hijo quedó modificado
> ```
> Y tampoco impiden mutar vía prototipo ni cambiar métodos heredados. `Object.freeze` sobre una instancia de clase no bloquea el comportamiento de sus métodos.

## Símbolos como claves

Un `Symbol` es un valor **único e inmutable** que puede usarse como clave sin colisionar — la base de `Symbol.iterator`, `Symbol.toPrimitive` y de las claves "ocultas" de librerías:

```js
const sym = Symbol("id");
const obj = { [sym]: 2, normal: 3, dinamica: 1 };
console.log(obj[sym]);                             // 2 — solo por símbolo, nunca por punto
console.log(Object.keys(obj));                     // ["dinamica", "normal"] — no aparece
console.log(JSON.stringify(obj));                  // {"dinamica":1,"normal":3} — invisible
console.log(Object.getOwnPropertySymbols(obj).length); // 1 — existe, pero aparte
```

Los símbolos **nunca aparecen** en `Object.keys`, `for...in`, `Object.entries` ni `JSON.stringify` (verificado): son el mecanismo de "propiedades privadas de hecho" pre-`#privadas` (ver [[Sintaxis Moderna]]).

## Optional chaining sobre objetos

`?.` corta la cadena y devuelve `undefined` si alguna referencia intermedia es `null`/`undefined` — en vez de lanzar `TypeError`:

```js
const user = { perfil: { direccion: null } };

console.log(user.perfil?.direccion?.calle); // undefined — sin excepción
console.log(user.inexistente?.profundo?.x); // undefined
user.perfil?.saluda?.();                    // método opcional: no existe → undefined
```

Con el operador `??` (nullish coalescing) combinas con un valor por defecto **solo** cuando el resultado es `null`/`undefined`:

```js
console.log(0 || "default");  // "default" — || mira falsiness
console.log(0 ?? "default");  // 0 — ?? solo respeta null/undefined
console.log("" ?? "default"); // ""
```

Detalle completo en [[Operadores]].

## JSON y objetos

`JSON.stringify`/`JSON.parse` traducen entre texto y objetos con reglas estrictas (claves entre comillas, sin funciones, sin `undefined`): ver [[Formato Json]] para el catálogo completo de qué se serializa, qué se descarta y por qué `JSON.stringify` lanza `TypeError` con referencias circulares.

## Rendimiento

- **Lectura de propiedad** en objetos "monomórficos" (siempre la misma forma) es ultrarrápida: los motores representan el objeto como *hidden class* y el acceso es un offset fijo. Saltar entre formas distintas de un mismo objeto (agregar/borrar claves dinámicamente, claves en orden cambiante) lo vuelve *polimórfico* y más lento. Patrón sano: **declarar todas las claves en el literal** (aunque valgan `undefined`).
- `delete o.prop` convierte la propiedad en *dictionary mode*: elimina la hidden class y degrada el acceso. Usá `undefined` o reformá si es hotspot.
- `Object.freeze`/`seal` recorren todas las propiedades: O(n) una vez; tras congelar, los motores pueden optimizar asignaciones (se descartan).
- Copiar objetos grandes con spread/`assign` es O(n); la copia profunda con JSON o `structuredClone` es mucho más cara — no la hagas en loops.

## Errores comunes

1. **Esperar que `const` inmutable el objeto**: `const` solo bloquea reasignar la referencia; las propiedades siguen libres. Para inmutabilidad real: `Object.freeze` (y recordar que es superficial).
2. **Copiar con JSON "por costumbre"**: pierde funciones, convierte `Date` a string y revienta con circulares — usá `structuredClone` o copia explícita (ver [[Formato Json]]).
3. **Modificar la copia y sorprenderse de que cambió el original** (o al revés): `assign`/spread copian **referencias** de valores anidados — verificado con `{v: 1} → 111`.
4. **`for...in` sin filtrar heredadas**: recoge claves del prototipo. Guard con `Object.hasOwn`, o mejor `Object.keys/entries` (solo propias) — ver [[Iteradores]].
5. **Acceder a anidados sin optional chaining**: `usuario.datos.direccion.calle` revienta si `datos` es `undefined`. Con `?.` devuelve `undefined` y lo resolvés con `??`.
6. **Comparar objetos con `==`/`===`**: comparan **referencias**, no contenido — `{} === {}` es `false`. Para contenido: `JSON.stringify(a) === JSON.stringify(b)` (limitado) o librerías/deep-equal estructural.
7. **Clave con guiones por punto**: `obj.min-height` es interpretado como resta. Usá `obj["min-height"]`.
8. **`defineProperty` sobre propiedad existente sin especificar todo**: los atributos no dados usan los defaults estrictos (`writable/enumerable/configurable: false`) y pueden "apagar" una propiedad que funcionaba.
9. **Esperar que `Object.keys` ordene por inserción siempre**: las claves enteras se reordenan numéricamente (`{b, 10, a, 2}` → `['1','2','10','b','a']` — verificado). Nunca dependas del orden salvo el garantizado por especificación.

## Cheatsheet

| Operación | API | Nota |
| --------- | --- | ---- |
| Crear | `{...}`, `new Fn()`, `Object.create`, `class` | Literal por defecto |
| Leer | `obj.k`, `obj[k]`, `obj?.k` | `?.` evita TypeError |
| Escribir | `obj.k = v`, `obj[k] = v` | Falla si `writable: false` o `freeze` |
| Borrar | `delete obj.k` | Devuelve booleano; falla si `configurable: false` |
| ¿Existe? | `"k" in obj`, `Object.hasOwn(obj, k)`, `obj.k !== undefined` | `in` ve heredadas |
| Claves | `Object.keys` | Solo propias enumerables; orden: enteros → inserción |
| Valores | `Object.values` / `Object.entries` | Para `for...of` |
| Copiar | `{ ...obj }`, `Object.assign({}, obj)` | **Superficiales** |
| Copia profunda | `structuredClone(obj)` | Nativo; `JSON` tiene trampas |
| Combinar | `{ ...a, ...b }` | Ganador: `b` |
| Congelar | `Object.freeze(o)` | `writable`+`configurable` → false; superficial |
| Sellado | `Object.seal(o)` | Solo bloquea agregar/borrar; valores editables |
| Estado | `Object.isFrozen/isSealed/isExtensible` | Comprobaciones de estado |
| Descriptor | `defineProperty`, `getOwnPropertyDescriptor` | Control total: writable/enumerable/configurable |
| Prototipo | `Object.getPrototypeOf`, `instanceof`, `Fn.prototype` | Cadena de herencia |
| Símbolos | `Symbol()`, `getOwnPropertySymbols` | Claves invisibles a JSON/keys |

## Relacionadas

- [[Java Script]] — el lenguaje donde vive todo esto
- [[Funciones]] — constructores, `this` y clases sobre prototipos
- [[Sintaxis Moderna]] — destructuring, spread/rest, clases, campos privados `#`
- [[Iteradores]] — `for...in`/`for...of` y el protocolo de iteración de objetos
- [[Formato Json]] — serializar y parsear objetos; qué se pierde en el camino
- [[Operadores]] — optional chaining `?.` y nullish `??`
- [[Arreglos]] — objetos con claves numéricas y su propio prototipo
- [[Strings]] — primitivo que se auto-boxea a objeto al acceder a `length`
- [[La Consola]] — `console.table(obj)` para inspeccionar objetos
