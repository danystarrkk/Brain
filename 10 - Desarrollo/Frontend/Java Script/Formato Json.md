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
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/JSON"
---

# Formato JSON

**JSON** (*JavaScript Object Notation*) es un formato de **texto** para estructurar datos de forma que cualquier lenguaje pueda leerlos y escribirlos. Casi todas las [[Qué es una API|APIs]] modernas envían y reciben JSON, por eso es el formato de intercambio por excelencia de la web.

> [!info] JSON no es JavaScript
> Aunque la sintaxis parece la de un objeto JS, JSON es un **estándar independiente** (RFC 8259) y es más restrictivo: solo acepta texto, números, booleanos, `null`, arreglos y objetos.

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

Las **claves van siempre entre comillas** (y deben ser strings), los valores pueden ser string, número, booleano, `null`, arreglo u objeto.

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

## Reglas de oro del JSON

| Regla | Ejemplo incorrecto | Ejemplo correcto |
| ----- | ------------------ | ---------------- |
| Claves con comillas dobles | `{precio: 12}` | `{"precio": 12}` |
| Sin comas finales (*trailing*) | `{"a": 1, "b": 2,}` | `{"a": 1, "b": 2}` |
| Strings con comillas dobles | `{'hola'}` | `"hola"` |
| Sin comentarios | `{"a": 1} // no` | no existen comentarios |
| Solo tipos JSON | `undefined`, `funciones` | no están permitidos |

> [!warning] La coma final rompe el JSON
> El error más común al escribir JSON a mano es dejar la coma después del último elemento: `JSON.parse('{"a": 1,}')` lanza un `SyntaxError`. Valida tu JSON en la consola con `JSON.parse()` antes de consumirlo.

## JSON y JavaScript: `parse` y `stringify`

JavaScript trae dos funciones para traducir entre **texto JSON** y **objetos JS**:

```js
// Texto -> objeto (por ejemplo, la respuesta de un fetch)
const texto = '{"nombre": "Monitor", "precio": 500, "enStock": true}';
const producto = JSON.parse(texto);

console.log(producto.nombre);  // "Monitor"
console.log(producto.precio);  // 500

// Objeto -> texto (por ejemplo, para enviarlo en un POST)
const envio = { titulo: 'Nuevo producto', activo: true };
const cuerpo = JSON.stringify(envio);
console.log(cuerpo); // '{"titulo":"Nuevo producto","activo":true}'

// Copia profunda de un objeto con JSON (truco clásico)
const copia = JSON.parse(JSON.stringify(producto));
```

| Función | Entrada | Salida |
| ------- | ------- | ------ |
| `JSON.parse(texto)` | string JSON | objeto/arreglo JS |
| `JSON.stringify(objeto)` | objeto JS | string JSON |

El flujo típico con una [[Qué es una API|API]]: `fetch` → `respuesta.json()` (que usa `parse` por dentro) → objeto JS que puedes renderizar con [[Acceder a Html]] o [[ReactJS - Fundamentos]].

## JSON en otros archivos

JSON también se usa como archivo con extensión `.json` para configuraciones: `package.json`, `tsconfig.json`, datos de ejemplo en `data/`. Todo respeta las mismas reglas de la tabla anterior.

## Relacionadas

- [[Qué es una API]] — el formato con el que hablan las APIs
- [[Objetos]] — los objetos JS que JSON intenta reflejar
- [[Acceder a Html]] — mostrar los datos parseados en la página
- [[ReactJS - Fundamentos]] — consumir JSON en componentes React
