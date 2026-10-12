---
tags:
  - desarrollo
  - frontend
  - java-script
tipo: concepto
estado: terminado
---

# Java Script

**JavaScript** (JS) es el lenguaje de programación de la web: mientras [[Html]] define la **estructura** y CSS el **estilo**, JavaScript define el **comportamiento**. Es dinámico, interpretado (o JIT-compilado, ver más abajo), multiparadigma (orientado a objetos basado en prototipos, con funciones de primera clase y programación funcional) y es el único lenguaje que ejecutan **todos** los navegadores sin instalación previa.

> [!info] JavaScript no es Java
> El nombre fue una decisión de marketing de Netscape en 1995 para surfear el éxito de Java. Comparten sintaxis de estilo C, y poco más. **Java** corre en máquinas virtuales con clases estáticas; **JavaScript** vive en el navegador con tipado dinámico y herencia por prototipos.

## Historia: ECMAScript vs JavaScript vs JScript

| Año | Evento |
| --- | ------ |
| 1995 | Brendan Eich crea **Mocha** (luego *LiveScript*, luego *JavaScript*) en Netscape en ~10 días |
| 1996 | Microsoft lanza **JScript** para Internet Explorer: una implementación compatible-pero-distinta de JS |
| 1997 | Netscape estandariza el lenguaje en **ECMA-262**; a partir de ahí la especificación se llama **ECMAScript** y la implementación de Netscape, *JavaScript* |
| 1999 | **ES3**: la base que corrió el "web 1.0" (IE6, jQuery aún no existía) |
| 2009 | **ES5**: estabiliza el lenguaje moderno temprano (`strict mode`, `JSON`, `Array.isArray`); dominó ~6 años |
| 2015 | **ES6 / ES2015**: la mayor revisión de la historia (`let`/`const`, clases, módulos, arrow functions, promesas, `for...of`) — ver [[Sintaxis Moderna]] |
| 2016+ | Cadencia **anual**: ES2016, ES2017… cada junio llega una versión pequeña y predecible |

La distinción práctica:

- **ECMAScript** = la especificación (el "qué debe hacer"). La mantiene el comité **TC39**.
- **JavaScript** = la implementación más usada (Netscape → Mozilla → todos). En la práctica, "JS" y "ECMAScript" se usan como sinónimos.
- **JScript** = la implementación histórica de Microsoft (IE). Código escrito para JScript es el JavaScript "raro" que se encuentra en sistemas legados — ver [[Sintaxis Antigua]].
- **ActionScript** (Flash) también era ECMAScript; murió con Flash en 2020.

## Motores: V8, SpiderMonkey y compañía

Cada navegador (y runtime de servidor) incluye un **motor de JavaScript**: el programa que convierte tu código en ejecución real.

| Motor | Creado por | Dónde corre | Dato |
| ----- | ---------- | ----------- | ---- |
| **V8** | Google (2008) | Chrome, Edge, **Node.js**, Deno, Bun | Escrito en C++; el más difundido del mundo |
| **SpiderMonkey** | Mozilla (1995) | Firefox | El **primer** motor de JS de la historia |
| **JavaScriptCore** (Nitro) | Apple | Safari, iOS | Optimizado para batería móvil |
| **Hermes** | Meta | React Native | Compila a bytecode en build time (arranque más rápido en móviles) |

### Cómo ejecuta un motor: parse → AST → bytecode → JIT

1. **Parsing**: el texto se convierte en tokens y luego en un **AST** (*Abstract Syntax Tree*, árbol de sintaxis). Un error de sintaxis (`Unexpected token`) aborta aquí — el archivo entero falla, no solo esa línea.
2. **Bytecode**: V8 (Ignition) genera bytecode compacto e inmediatamente lo empieza a ejecutar en un intérprete rápido. Arranque casi instantáneo.
3. **JIT**: las partes que se ejecutan muchas veces ("calientes") se envían al compilador optimizador (**TurboFan** en V8), que asume tipos a partir del historial de ejecución y genera código máquina nativo.

> [!tip] Por qué JS es "rápido pese a ser dinámico"
> El truco es el **JIT con especulación**: si una función siempre recibe `number`, el motor asume `number` y emite aritmética nativa. Si un día llega un `string`, **deoptimiza** y vuelve al intérprete. Por eso conviene mantener los tipos estables dentro de una función — funciones que alternan `string`/`number`/`object` en los mismos parámetros son notoriamente más lentas.

## Cómo carga JS una página: `<script>`

Para que el navegador ejecute JavaScript, el código debe estar dentro de la página de alguna de estas dos formas: **inline** (dentro del HTML) o en un **archivo externo**.

### Dentro del HTML (inline)

```html
<script>
  alert('Hola Mundo');
</script>
```

La etiqueta `<script>` le dice al navegador que el contenido es JavaScript y debe interpretarse. `alert` es una función propia del navegador que muestra un cuadro de diálogo.

> [!warning] Con `alert` basta para aprender, no para producir
> `alert` **bloquea** el hilo de ejecución hasta que el usuario acepta. En código real se usa `console.log` (ver [[La Consola]]) o el DOM.

### Archivo externo (forma recomendada)

Lo recomendable es separar el JavaScript en su propio archivo, por ejemplo en una carpeta `js/` con un archivo llamado `app.js`:

![[Pasted image 20250125080131.png]]

El archivo contiene solo JavaScript, **sin** etiqueta `<script>`:

```js
// js/app.js
alert('Hola Mundo');
```

Y se enlaza desde el HTML:

```html
<script src="js/app.js"></script>
```

Ventajas de la forma externa:

- **Cacheo**: el navegador guarda `app.js` y no lo vuelve a descargar en cada página.
- **Mantenimiento**: el HTML queda limpio; el JS se edita en su propio archivo.
- **Reutilización**: el mismo archivo puede usarse en varias páginas.
- **Cache-busting**: para forzar re-descarga se versiona el nombre (`app.v2.js`) o se sirve con headers `Cache-Control`.

## ¿Dónde colocar el `<script>`?

El orden importa: el navegador descarga y ejecuta el JS mientras construye el HTML. Si el script usa `document.querySelector(...)` y corre antes de que ese elemento exista, obtiene `null` y falla. Por eso la regla histórica es **colocar el `<script>` justo antes del cierre de `</body>`**: todo el HTML ya está parseado.

```html
<body>
  ...contenido...
  <script src="js/app.js"></script>
</body>
```

### `defer` y `async`

El atributo `defer` moderniza ese consejo: permite poner el `<script>` en el `<head>` sin bloquear el parseo del HTML:

```html
<script src="js/app.js" defer></script>
```

| Atributo | Descarga | Ejecución |
| -------- | -------- | --------- |
| *(ninguno)* | bloquea el parseo | en cuanto baja, detiene el HTML |
| `defer` | en paralelo | **tras** terminar el parseo, en orden de aparición |
| `async` | en paralelo | en cuanto baja, interrumpa donde interrumpa el parseo |

> [!tip] Regla práctica
> Usa `defer` para tus scripts (se ejecutan con el DOM listo y en el orden correcto). Reserva `async` para scripts independientes que no dependen del DOM ni de otros scripts, como un widget de terceros. **Nunca** combines `defer` y `async` en el mismo archivo, y ten en cuenta que `document.readyState` puede ser `interactive` (DOM listo, recursos pendientes) antes de `complete`.

### Módulos: `type="module"`

La versión moderna del `<script>` — el scope ya no es global y habilita `import`/`export`:

```html
<script type="module" src="js/app.js"></script>
<!-- o inline -->
<script type="module">
  import { saludo } from './saludo.js';
  console.log(saludo);
</script>
```

Diferencias clave respecto al `<script>` clásico:

| Comportamiento | `<script>` | `<script type="module">` |
| -------------- | ---------- | ------------------------ |
| Scope de las variables | global (todo `var`/`function` cae en `window`) | **aislado** por archivo (como si todo estuviera en una IIFE) |
| Ejecución respecto al parseo | bloqueante salvo `defer`/`async` | **implícitamente `defer`** |
| `import`/`export` | no existen | sí (con rutas relativas explícitas, `.js` incluido) |
| Modo estricto | opt-in con `"use strict"` | **siempre** activo |

```js
// js/saludo.js — un módulo exporta
export const saludo = 'Hola';

// js/app.js — otro módulo importa
import { saludo } from './saludo.js';  // la extensión .js es obligatoria
console.log(saludo);
```

Ver módulos en el contexto completo de ES2015 en [[Sintaxis Moderna]].

## Entorno de ejecución: navegador vs Node

El lenguaje es el mismo; lo que cambia es el **entorno** que lo rodea — el conjunto de APIs disponibles alrededor del motor:

| | Navegador | Node.js |
| --- | --------- | ------- |
| Objeto global | `window` (+ `globalThis`) | `global` (+ `globalThis`) |
| API principal | `document` (ver [[Document Object Model (DOM)]]), `fetch`, eventos | `fs`, `http`, `path`, `process` |
| Acceso a archivos/sistema | prohibido (sandbox) | total |
| Interfaz gráfica | sí (DOM/CSSOM, canvas) | no |
| Modelo de concurrencia | **event loop** + Web APIs (timers, red) | **event loop** + libuv (I/O de disco/red) |

```
  ┌────────────────────────────────────────────────┐
  │  Tu código JS        (hilo único, sincrónico)  │
  ├────────────────────┬───────────────────────────┤
  │  NAVEGADOR:        │  NODE:                    │
  │  DOM, fetch,       │  fs, http,                │
  │  setTimeout,       │  setTimeout,              │
  │  eventos de ratón… │  I/O de disco…            │
  └────────────────────┴───────────────────────────┘
```

JavaScript **corre en un solo hilo**: no hay dos funciones ejecutándose a la vez. Lo que sí existe son operaciones externas (descargar una imagen, leer un archivo) que el entorno hace **en paralelo** y notifica cuando terminan. Ese mecanismo de notificación es el **event loop**.

### Event loop: por qué `setTimeout(fn, 0)` no es inmediato

El modelo mental mínimo:

1. **Call stack**: el motor apila y desapila llamadas sincrónicas. Si el stack está ocupado, nada más avanza.
2. **Web APIs / libuv**: `setTimeout`, `fetch`, clicks — se procesan fuera del stack.
3. **Task queue** (y **microtask queue**, para promesas): los callbacks terminados esperan aquí.
4. **Event loop**: cuando el stack queda vacío, toma el siguiente callback de la cola y lo empuja al stack.

```js
console.log('1: sincrónico');

setTimeout(() => console.log('3: setTimeout (macrotask)'));

Promise.resolve().then(() => console.log('2: promesa (microtask)'));

// Orden REAL: 1 → 2 → 3
// Las microtareas (promesas) vacían la cola ANTES del siguiente macrotask (timers).
```

Ver el evento `DOMContentLoaded` y más del ciclo en [[Eventos]].

## Tipado dinámico: filosofía y consecuencias

JS no pide declares tipos: el tipo vive **en el valor**, no en la variable. La misma variable puede ser `string`, luego `number`, luego `object`:

```js
let x = '10';   // string
x = 10;         // number
x = [1, 2, 3];  // object (array)
x = () => 1;    // function
```

Esto tiene consecuencias profundas:

- **Coerción implícita**: los operadores convierten tipos automáticamente — `"20" - 5` es `15`, pero `"20" + 5` es `"205"`. Ver el algoritmo completo en [[Operadores]].
- **Errores en tiempo de ejecución**: un `undefined` no se detecta al escribir el código, sino al correrlo (`TypeError: ... is not a function`).
- **Truthiness**: cualquier condición acepta cualquier valor; `0`, `""`, `null`, `undefined`, `NaN` y `0n` son *falsy*. Ver [[Booleanos]].
- **Duck typing**: "si hace cuack como pato, es pato" — lo que importa es la forma del objeto, no su clase declarada.

**TypeScript** (Microsoft, 2012) añade tipos estáticos *por encima* de JS: se escribe TS, el compilador lo verifica y **emite JS**. No es un lenguaje distinto sino un sistema de anotaciones — por eso el runtime sigue siendo JS puro y todos los conceptos de esta nota aplican igual. Ver también [[Sintaxis Antigua]] para el contraste con el JS anterior a ES2015.

## Mapa del ecosistema: a dónde lleva JS desde aquí

Una vez domines el lenguaje puro, el ecosistema se divide en capas:

| Capa | Opciones típicas | Para qué |
| ---- | ---------------- | -------- |
| **Frameworks UI** | React, Vue, Angular, Svelte, Solid | Construir interfaces por componentes |
| **Meta-frameworks** | Next.js (React), Nuxt (Vue), Astro | Routing, SSR, build optimizado |
| **Build tools** | **Vite** (estándar de facto hoy), esbuild, webpack (legado), tsc | Bundlear, transpilar, minificar |
| **Gestores de paquetes** | npm, pnpm, yarn | Instalar dependencias de `node_modules` |
| **Tipos** | TypeScript, JSDoc + `tsc --checkJs` | Estática opcional |
| **Runtime de servidor** | **Node.js**, Deno, Bun | Backend, CLIs, scripts |
| **Testing** | Vitest, Jest, Playwright | Pruebas unitarias y E2E (ver [[Recomendaciones]]) |

El camino de aprendizaje sugerido en este vault:

1. [[Sintaxis Antigua]] y [[Sintaxis Moderna]] — el lenguaje mismo: variables, scope, ES2015+.
2. [[Operadores]] y [[Estructuras de Control]] — expresiones y flujo.
3. [[Funciones]], [[Booleanos]], [[Números]], [[Strings]] — los tipos fundamentales.
4. [[Arreglos]] + [[Array Methods]] + [[Iteradores]] + [[Objetos]] — las estructuras de datos.
5. [[La Consola]] — cómo depurar todo lo anterior.
6. [[Document Object Model (DOM)|DOM]], [[Acceder a Html]], [[Eventos]] — el navegador.
7. [[Uso de Java Script]] y [[Formato Json]] — patrón asíncrono `fetch` + JSON.
8. Luego: [[ReactJS - Fundamentos]] o [[Node JS]] según el lado que te interese.
9. [[Qué es una API]] para entender cómo se comunica JS con servidores.

## Errores comunes al empezar

1. **Script antes de que el HTML exista**: `document.querySelector('h1')` devuelve `null` → `TypeError: Cannot read properties of null`. Solución: `defer`, fin de `</body>`, o `DOMContentLoaded` (ver [[Eventos]]).
2. **Olvidar `src` vs contenido**: `<script src="app.js">console.log('x')</script>` — el contenido inline se **ignora** si hay `src`.
3. **Rutas relativas rotas en módulos**: en `type="module"`, `import './app'` falla; debe ser `'./app.js'` con extensión.
4. **Tipos mezclados**: `if (valor == '0')` o sumas que concatenan en vez de sumar — ver la sección de coerción de [[Operadores]].
5. **Mezclar librerías con scope global**: dos scripts definiendo `function init()` se pisan; los módulos (`type="module"`) lo resuelven de raíz.
6. **Errores de tildes/encoding**: el HTML debe declarar `<meta charset="UTF-8">` o los acentos en cadenas JS se rompen.

## Cheatsheet

| Concepto | Forma | Nota |
| -------- | ----- | ---- |
| Script inline | `<script>…</script>` | Bloquea el parseo |
| Script externo | `<script src="js/app.js"></script>` | Cacheable, reutilizable |
| Carga diferida | `<script src="…" defer></script>` | Tras el parseo, en orden |
| Carga independiente | `<script src="…" async></script>` | En cuanto baja, sin orden |
| Módulo | `<script type="module" src="…">` | Scope aislado, `defer` implícito |
| Exportar | `export const x = 1` | En el módulo fuente |
| Importar | `import { x } from './m.js'` | Ruta relativa con extensión |
| Evento de DOM listo | `document.addEventListener('DOMContentLoaded', fn)` | Ver [[Eventos]] |
| Global canónico | `globalThis` | Navegador y Node |
| Orden de callbacks | sincrónico → microtasks (promesas) → macrotasks (timers) | El event loop |

## Compatibilidad

JS del lenguaje base (ES3/ES5) corre hasta en IE11. El JS moderno (ES2015+) **no requiere build** en navegadores actuales (2026): todo ES2022+ está disponible de forma nativa. El transpilar con Babel/tsc sigue existiendo por soporte a navegadores viejos en empresas, pero ya no es obligatorio para proyectos nuevos. Para verificar qué versión soporta cada navegador: <https://compat-table.github.io/compat-table/es6/> y <https://caniuse.com>.

## Relacionadas

- [[Html]] — el lenguaje donde se incrusta el script
- [[Document Object Model (DOM)]] — la estructura que el script manipula
- [[Acceder a Html]] — seleccionar y modificar elementos del DOM
- [[Eventos]] — hacer la página interactiva y el ciclo de vida del DOM
- [[La Consola]] — `console.log`, la herramienta de depuración básica
- [[Sintaxis Moderna]] — ES2015+ en detalle: `let`/`const`, módulos, destructuring
- [[Sintaxis Antigua]] — leer y migrar JS legado (var, IIFE, jQuery)
- [[Operadores]] — cada operador con su algoritmo de coerción
- [[Estructuras de Control]] — `if`, `switch`, bucles, guard clauses
- [[Node JS]] — el mismo lenguaje en el servidor
- [[Qué es una API]] — cómo JS se comunica con el exterior
