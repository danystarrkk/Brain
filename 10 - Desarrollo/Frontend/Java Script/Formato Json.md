---
aliases: ["JSON", "JavaScript Object Notation"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Qué es una API]]"
  - "[[Objetos]]"
  - "[[Acceder a Html]]"
  - "[[Node JS]]"
  - "[[Sintaxis Moderna]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/JSON"
---

# Formato JSON

**JSON** (*JavaScript Object Notation*) es un formato de **texto** para estructurar datos de forma que cualquier lenguaje pueda leerlos y escribirlos. Casi todas las [[Qué es una API|APIs]] modernas envían y reciben JSON, por eso es el formato de intercambio por excelencia de la web. Es un **estándar independiente** de JavaScript: RFC 8259, definido con una gramática propia y parsers en todos los lenguajes.

> [!info] JSON no es JavaScript
> Aunque la sintaxis parece la de un objeto JS, JSON es más restrictivo: solo acepta texto, números, booleanos, `null`, arreglos y objetos. No hay funciones, ni variables, ni comentarios, ni comas finales. `eval()` de JSON (la práctica pre-2005) era insegura y lenta; `JSON.parse` es la única forma correcta.

## Estructura básica

JSON se organiza en **pares clave/valor** dentro de `{}`:

```json
{
  "propiedad": "valor",
  "precio": 12,
  "estado": true,
  "arreglo": ["pera", "fresa"]
}
```

Las **claves van siempre entre comillas dobles** (y deben ser strings), los valores pueden ser string, número, booleano, `null`, arreglo u objeto. Un documento JSON completo es **un valor**: un objeto, un arreglo o incluso un escalar (`42`, `"hola"`, `true`, `null` son JSON válido en la raíz).

## Anidación

Los objetos y arreglos pueden anidarse sin límite, lo que permite modelar estructuras completas:

```json
{
  "propiedad": {
    "precio": 34
  },
  "perfil": [
    { "id": 3 },
    { "url": "https://ejemplo.com" }
  ]
}
```

## Reglas de la gramática (completo)

| Tipo | Reglas | Válido | Inválido |
| ---- | ------ | ------ | -------- |
| **Objeto** | `{}`; pares `"clave": valor` separados por `,` | `{"a": 1}` | `{a: 1}` (clave sin comillas) |
| **Arreglo** | `[]`; valores separados por `,` | `[1, "a", null]` | `['a']` (comillas simples) |
| **String** | **Solo comillas dobles**; escapes `\" \\ \/ \b \f \n \r \t \uXXXX` | `"hola\nmundo"`, `"A"` | `'hola'`, `"tab\there"` (escape inválido) |
| **Número** | Formato decimal: opcional `-`, sin `+`, sin octal/hex, sin sufijos, **sin `.5` ni `5.`** | `-1.5e10`, `0`, `300` | `.5`, `5.`, `0x1F`, `1_000`, `NaN`, `Infinity` |
| **Booleano** | Minúsculas exactas | `true`, `false` | `True`, `TRUE` |
| **Null** | Literal exacto | `null` | `NULL`, `Null`, `undefined` |
| **Comentarios** | **No existen** | — | `{"a": 1} // fin` |
| **Coma final** | Prohibida después del último elemento o par | `{"a": 1, "b": 2}` | `{"a": 1, "b": 2,}` |

### Reglas de oro del JSON

| Regla | Ejemplo incorrecto | Ejemplo correcto |
| ----- | ------------------ | ---------------- |
| Claves con comillas dobles | `{precio: 12}` | `{"precio": 12}` |
| Sin comas finales (*trailing*) | `{"a": 1, "b": 2,}` | `{"a": 1, "b": 2}` |
| Strings con comillas dobles | `{'hola'}` | `"hola"` |
| Sin comentarios | `{"a": 1} // no` | no existen comentarios |
| Solo tipos JSON | `undefined`, `funciones` | no están permitidos |

> [!warning] La coma final rompe el JSON
> El error más común al escribir JSON a mano es dejar la coma después del último elemento: `JSON.parse('{"a": 1,}')` lanza un `SyntaxError` (mensaje verificado: `Expected property name or '}' in JSON at position 1`). Valida tu JSON en la consola con `JSON.parse()` antes de consumirlo.

## JSON vs objeto literal de JavaScript

La confusión raíz de casi todos los bugs de JSON: parecen lo mismo, no lo son.

| Característica | Objeto JS literal | JSON |
| -------------- | ----------------- | ---- |
| Claves | Sin comillas si son identificadores válidos | **Siempre** entre comillas dobles |
| Strings | `'o'` o `"o"` | **Solo** `"o"` |
| Comentarios | Sí | No |
| Funciones | Sí (como valor) | **No válido** |
| `undefined` | Sí (valor) | **No válido** |
| Comas finales | Permitidas en ES5+ | **No** |
| `NaN` / `Infinity` | Sí | **No** (número debe ser decimal) |
| Identificadores reservados como clave | Sí (`obj.class`) | Sí (`"class"` es clave válida) |
| Valores de referencia | Cualquier cosa | Solo los 6 tipos JSON |

```js
// ❌ Esto es JS válido pero NO es JSON — parsearlo revienta
const noJson = { fecha: new Date(), callback: () => {}, extra: undefined, };

// ✅ Equivalente JSON válido
const jsonTexto = '{"fecha": "2026-10-11", "extra": null}';
```

## `JSON.parse`: texto → objeto

```js
// Texto -> objeto (por ejemplo, la respuesta de un fetch)
const texto = '{"nombre": "Monitor", "precio": 500, "enStock": true}';
const producto = JSON.parse(texto);

console.log(producto.nombre);  // "Monitor"
console.log(producto.precio);  // 500
```

### El *reviver*: transformar durante el parseo

El segundo argumento es una función `(clave, valor)` invocada **por cada par**, de dentro hacia afuera (bottom-up): el valor de retorno reemplaza al par en el resultado; `undefined` lo **elimina**:

```js
// parsear fechas de vuelta a objetos Date
const crudo = '{"nombre": "Ana", "nacimiento": "1990-04-12"}';
const obj = JSON.parse(crudo, (clave, valor) => {
  if (clave === "nacimiento") return new Date(valor);
  return valor;
});
console.log(obj.nacimiento.getFullYear()); // 1990

// eliminar claves durante el parse (undefined las borra)
const limpio = JSON.parse('{"a":1,"borrar":2}', (k, v) => (k === "borrar" ? undefined : v));
console.log(limpio); // { a: 1 }
```

Errores de parseo (verificados):

| Entrada | Resultado |
| ------- | --------- |
| `'{"a":1,}'` | `SyntaxError: Expected property name or '}' ...` |
| `'{a:1}'` | `SyntaxError` (clave sin comillas) |
| `'undefined'` | `SyntaxError: "undefined" is not valid JSON` |
| `'"\\u0041"'` | `"A"` — los escapes `\uXXXX` se decodifican |

## `JSON.stringify`: objeto → texto

```js
// Objeto -> texto (por ejemplo, para enviarlo en un POST)
const envio = { titulo: 'Nuevo producto', activo: true };
const cuerpo = JSON.stringify(envio);
console.log(cuerpo); // '{"titulo":"Nuevo producto","activo":true}'
```

| Función | Entrada | Salida |
| ------- | ------- | ------ |
| `JSON.parse(texto)` | string JSON | objeto/arreglo JS |
| `JSON.stringify(obj)` | objeto JS | string JSON (o `undefined` — ver límites) |

### El tercer argumento `space`: formato legible

```js
console.log(JSON.stringify({ a: { b: 1 } }, null, 2));
// {
//   "a": {
//     "b": 1
//   }
// }
console.log(JSON.stringify({ a: 1 }, null, "¿?"));
// {
// ¿?"a": 1
// }
console.log(JSON.stringify({ a: 1 }, null, 10)); // hasta 10 espacios
```

`space` acepta un número (0–10 espacios, verificados con `2` y con el tope `10`) o un string (hasta 10 primeros caracteres — por eso `"¿?"` imprime literal). Es **solo cosmético**: no cambia el resultado al parsear. Útil para logs y archivos de configuración legibles.

### El *replacer*: filtrar o transformar

```js
// array de claves: serializar SOLO esas (recursivo para anidados)
console.log(JSON.stringify({ a: 1, b: 2, c: 3 }, ["a", "c"]));
// {"a":1,"c":3}

// función: transforma cada par (clave, valor); valores de retorno se aplican
console.log(JSON.stringify({ a: 1, b: 2 }, (k, v) => (typeof v === "number" ? v * 2 : v)));
// {"a":2,"b":4}

// excluir una propiedad concreta recursivamente
const usuario = {
  nombre: "Ana",
  password: "secreto",
  perfil: { nombre: "Ana", password: "también secreto" },
};
console.log(JSON.stringify(usuario, (k, v) => (k === "password" ? undefined : v)));
// {"nombre":"Ana","perfil":{"nombre":"Ana"}}
```

### `toJSON`: el objeto decide cómo serializarse

Si el objeto (o una propiedad) tiene un método `toJSON`, se usa **en lugar** del objeto:

```js
const persona = { nombre: "Ana", edad: 30, toJSON() { return { iniciales: "A." }; } };
console.log(JSON.stringify(persona)); // {"iniciales":"A."} — verificado
```

`Date` lo usa internamente: `JSON.stringify(new Date("2026-10-11"))` → `"2026-10-11T00:00:00.000Z"`. También hay `toJSON` en `BigInt` (lanza error: BigInt no es serializable nativo).

## Límites de serialización (todo verificado con node)

| Situación en el objeto | Resultado de `stringify` |
| ---------------------- | ------------------------ |
| Propiedad con valor `undefined` o función o símbolo (en **objeto**) | **Se descarta** la propiedad: `{"a":undefined,b:()=>{}}` → `{}` |
| Elemento así en un **arreglo** | Se convierte en `null`: `[undefined, ()=>{}, 1]` → `[null,null,1]` |
| `NaN`, `Infinity`, `-Infinity` | `"null"` |
| `Date` | Su `toJSON`: string ISO-8601 |
| `Map` / `Set` vacíos | `{}` (¡no hay representación!) |
| Referencia **circular** | `TypeError: Converting circular structure to JSON` |
| `BigInt` | `TypeError: Do not know how to serialize a BigInt` |
| `stringify(undefined)` en la raíz | `undefined` (¡ni siquiera string!) |

> [!warning] El `stringify` mudo
> Las propiedades `undefined`/función/símbolo **desaparecen sin aviso**: puedes enviar un payload "incompleto" y no enterarte. Lo inverso también duele: `JSON.parse` no puede distinguir una propiedad borrada de una que nunca existió.

### Copia profunda con JSON — y sus trampas

```js
// Copia profunda de un objeto con JSON (truco clásico)
const copia = JSON.parse(JSON.stringify(producto));
```

Funciona porque parse/stringify reconstruye todo el árbol con valores nuevos. Pero el viaje pierde cosas:

| Pierde / transforma | Ejemplo |
| ------------------- | ------- |
| Funciones | Se descartan silenciosamente |
| `Date` → string ISO | La copia ya no es un `Date` |
| `undefined` en propiedades | Desaparecen |
| `Map`, `Set`, `Symbol`, `RegExp` | `{}` / símbolos borrados / `{}"` |
| Referencias circulares | Lanza `TypeError` (a diferencia de `structuredClone`) |

Alternativa moderna recomendada: `structuredClone(obj)` — copia profunda nativa que soporta `Date`, `Map`, `Set`, `ArrayBuffer` y ciclos, sin serializar a texto. JSON-copy solo cuando el dato ya viene de JSON y su forma lo garantiza.

## JSON y `fetch`: el flujo real de una API

El ciclo típico con una [[Qué es una API|API]]:

```js
const respuesta = await fetch("https://api.ejemplo.com/productos/1");
const producto = await respuesta.json(); // lee el body como texto y hace JSON.parse
console.log(producto.nombre);

// envío: stringify del body con Content-Type correcto
const resp = await fetch("/api/productos", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ nombre: "Monitor", precio: 300 }),
});

// errores de parseo: respuesta.json() rechaza la promesa si el body no es JSON válido
try {
  const datos = await respuesta.json();
} catch (e) {
  console.error("El servidor no devolvió JSON:", e.message);
}
```

`Response.json()` usa `JSON.parse` por dentro (y `Request`/`Response` usan `stringify` en bodies con objetos). Si la API devuelve HTML de error, el catch atrapa el `SyntaxError`.

## JSON en otros archivos

JSON también se usa como archivo con extensión `.json` para configuraciones: `package.json` y `tsconfig.json` en [[Node JS]], datos de ejemplo en `data/`, manifiestos de PWA. Todo respeta las mismas reglas de la tabla anterior.

### JSON Lines (`.jsonl`)

Variante para streams/logs: **un objeto JSON por línea**, sin arrays ni saltos entre documentos. Ventaja: se puede procesar incrementalmente (cada línea es un documento independiente) y sobrevive a cortes:

```jsonl
{"id": 1, "evento": "login"}
{"id": 2, "evento": "click", "x": 120}
{"id": 3, "evento": "logout"}
```

```js
const lineas = textoJsonl.trim().split("\n");
const registros = lineas.map((l) => JSON.parse(l)); // parse por línea, sin array gigante
```

## JSON en otros formatos: comparación

| | **JSON** | **XML** | **YAML** |
| --- | --- | --- | --- |
| Sintaxis | Minimalista, sin tags | Verbose, etiquetas | Indentación tipo Python |
| Comentarios | No | Sí | Sí |
| Tipos nativos | 6 (string, número, bool, null, array, object) | Todo es texto (atributos tipados a mano) | Ricos (fechas, anchors...) |
| Referencias/IDs | No | `id`/`ref` | Anclas `&`/alias `*` |
| Peso | Mínimo | Pesado | Ligero |
| Ecosistema | El estándar de APIs web | SOAP, config enterprise, XHTML | CI/CD (K8s, GitHub Actions) |
| Errores comunes | Coma final, comillas simples | Tags sin cerrar | **Indentación inconsistente** (tan frágil como la coma en JSON) |

JSON ganó la web por simplicidad y parseo barato; XML sobrevive donde hay validación de esquemas (XSD) o documentos semánticos; YAML domina configuración humana-editable.

## Errores comunes

1. **Coma final**: `{"a":1,}` → `SyntaxError`. El error #1 al editar JSON a mano (configuraciones, fixtures).
2. **Clave sin comillas o comillas simples**: `{a: 1}` / `{'a': 1}` — el parser exige `"`. En JS ambos son válidos, por eso el cerebro "deja pasar" el error.
3. **Comentarios en `.json`**: no existen en el estándar. Si necesitas comentarios, usá JSONC (VSCode lo tolera en sus configs) o cambiá de formato.
4. **`undefined`/función esperando que se serialicen**: se descartan silenciosamente en objetos (y en `null` en arreglos). Verificado arriba.
5. **Copia profunda con JSON perdiendo tipos**: `Date` vuelve como string y las funciones desaparecen. Usá `structuredClone` para clonar objetos JS de verdad.
6. **`TypeError` con estructuras circulares**: padres que referencian hijos. Rompé la referencia, usá `replacer` que la omita, o `structuredClone` (soporta ciclos).
7. **Comparar `JSON.stringify` como igualdad profunda**: el orden de claves importa (`{a:1,b:2}` ≠ `{b:2,a:1}` como string) y pierde tipos. Solo sirve como heurística barata.
8. **Olvidar el `Content-Type: application/json`** al enviar: muchos backends leen el body como form-encoded y los campos llegan vacíos.
9. **Parsear números gigenteros**: JSON.parse usa IEEE-754 doble precisión — IDs de más de 15-16 dígitos pierden precisión (el `123456789012345678` se vuelve `...80`). En APIs serias los IDs grandes viajan como string.

## Cheatsheet

| Tema | API / regla |
| ----- | ----------- |
| Tipos válidos | string, número, booleano, `null`, array, object |
| Claves | Siempre `"comillas dobles"` |
| prohibido | comentarios, comas finales, `undefined`, funciones, `'simples'`, `NaN` |
| Texto → objeto | `JSON.parse(texto[, reviver])` |
| Objeto → texto | `JSON.stringify(obj[, replacer[, space]])` |
| Formato legible | `JSON.stringify(obj, null, 2)` |
| Filtrar | `stringify(obj, ["clave"])` o replacer-function |
| Transformar al parsear | reviver `(k, v) => ...` |
| Serialización propia | `obj.toJSON()` |
| Se descarta | `undefined`, funciones, símbolos (en objetos) |
| Se vuelve `null` | `NaN`, `±Infinity` (y huecos de arreglos) |
| Ciclos | `TypeError` en stringify; OK en `structuredClone` |
| Copia profunda | `structuredClone` > `JSON.parse(JSON.stringify(x))` |
| Fetch | `await res.json()` / `body: JSON.stringify(...)` + `Content-Type` |
| Stream/logs | JSON Lines: un objeto JSON por línea |

## Relacionadas

- [[Qué es una API]] — el formato con el que hablan las APIs
- [[Objetos]] — los objetos JS que JSON intenta reflejar (y sus descriptores/símbolos que JSON no ve)
- [[Acceder a Html]] — mostrar los datos parseados en la página
- [[ReactJS - Fundamentos]] — consumir JSON en componentes React
- [[Node JS]] — `package.json`, `fs.readFile` y JSON en el backend
- [[Sintaxis Moderna]] — destructuring de la respuesta: `const { data } = await res.json()`
