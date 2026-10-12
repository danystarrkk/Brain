---
tags:
  - desarrollo
  - frontend
  - consola
tipo: concepto
estado: terminado
---

# La Consola en JavaScript

Todos los navegadores incluyen una **consola** integrada dentro de las Developer Tools. Cada navegador la abre en un lugar distinto, pero en la gran mayoría se llega a ella con clic derecho → **Inspeccionar** → pestaña **Console** (o la tecla `F12` / `Ctrl + Shift + I` / `Cmd + Option + I` en Mac).

La consola es una de las herramientas fundamentales, especialmente al empezar con JavaScript, porque permite:

- **Crear variables** y probar snippets de código de forma inmediata.
- Crear [[Arreglos|arreglos]], objetos y jugar con [[Funciones|funciones]].
- **Generar o detectar errores** del código (errores de sintaxis, `ReferenceError`, etc.).
- **Generar advertencias** (`console.warn`) para señalar problemas no críticos.
- Inspeccionar el estado del [[Document Object Model (DOM)|DOM]], la red y el rendimiento.

> [!info] La consola es un REPL
> La consola ejecuta lo que escribes **en vivo**, línea por línea, en el contexto de la página actual. Por eso sirve para experimentar: escribes `2 + 2` y obtienes `4` sin crear archivos. **REPL** = *Read-Eval-Print Loop*: lee la línea, la evalúa, imprime el resultado y espera la siguiente. El estado **persiste** entre líneas: si defines `const precio = 300` en una línea, la siguiente puede usar `precio`. También existe un REPL de JavaScript en la terminal con el comando `node` (ver más abajo).

> [!tip] Atajos de teclado de la consola (Chrome/Firefox)
> - `↑` / `↓` — historial de comandos.
> - `Ctrl + L` — limpiar la consola.
> - `Tab` — autocompletado (también completa propiedades: escribe `document.bod` y pulsa Tab).
> - `Ctrl + Shift + J` — abrir directamente la pestaña Console.
> - Escribir en la consola es **código real**: no hay permiso extra, podés hacer `fetch`, tocar el DOM, etc.

## Enviando datos a la consola

La forma más básica de enviar información es `console.log()`:

```js
const producto = 'Monitor de 20 pulgadas';
const precio = 300;

console.log(producto);
console.log(`El ${producto} cuesta ${precio} USD`);

// Acepta varias separadas por comas (no hace falta concatenar)
console.log('El producto', producto, 'cuesta', precio);
```

Pero `console.log` es solo el principio. El objeto `console` ofrece varios métodos de **nivel**, que se distinguen por color y por el filtro de la consola:

```js
console.log('Mensaje normal');
console.info('Información');          // en la práctica es igual a log
console.debug('Mensaje de depuración'); // se ocultan/muestran con el filtro de niveles
console.warn('Eso no se puede realizar'); // advertencia amarilla
console.error('Algo salió mal');        // mensaje en rojo
```

| Nivel | Color (Chrome) | Filtro | Uso |
| ----- | -------------- | ------ | --- |
| `debug` | gris | Verbose (desactivado por defecto) | ruido de depuración que no debe estorbar |
| `log` / `info` | blanco | Info | flujo normal |
| `warn` | amarillo + icono | Warnings | algo sospechoso, no fatal |
| `error` | rojo + icono + stack | Errors | fallos; también captura `uncaught` y `console.error` en red fallida |

> [!tip] `console.debug` para depuración desechable
> Es el nivel correcto para trazas que dejas en el código: por defecto el filtro de nivel no las muestra, así que no ensucian la consola de producción, pero están disponibles cuando las necesitás activando "Verbose".

## Medición de tiempo

```js
console.time('Tiempo');

console.warn('Eso no se puede realizar');
console.warn('Eso no se puede realizar');
console.warn('Eso no se puede realizar');

console.timeEnd('Tiempo'); // imprime "Tiempo: 1.234ms" (tu resultado será distinto)
```

- `console.time('etiqueta')` inicia un cronómetro con ese nombre.
- `console.timeEnd('etiqueta')` lo detiene e imprime cuánto tiempo pasó entre ambas llamadas. Sirve para **medir el rendimiento** de un bloque de código.
- `console.timeLog('etiqueta')` (Chrome/Firefox/Node) imprime el tiempo **sin detener** el cronómetro — permite marcas intermedias:

```js
console.time('proceso');
console.timeLog('proceso');   // "proceso: 0.3ms"
// ... más trabajo ...
console.timeEnd('proceso');   // "proceso: 12.8ms"
```

Etiquetas duplicadas reutilizan el mismo cronómetro. Para perfiles serios, la pestaña **Performance** (ver "rendimiento" abajo) da mucho más detalle que `time`.

## Catálogo completo de métodos

| Método | Para qué sirve |
| ------ | -------------- |
| `console.log(valor)` | Imprime un valor (el más usado) |
| `console.info(valor)` | Alias semántico de `log` |
| `console.debug(valor)` | Nivel verbose, oculto por defecto |
| `console.warn(valor)` | Imprime una advertencia |
| `console.error(valor)` | Imprime un error (con stack si lo tiene) |
| `console.time('x')` / `console.timeLog('x')` / `console.timeEnd('x')` | Mide el tiempo entre dos o más puntos |
| `console.count('x')` | Cuenta cuántas veces se ejecuta (con etiqueta); `console.countReset('x')` reinicia |
| `console.table(arreglo)` | Muestra arreglos u objetos como tabla; acepta columna extra: `console.table(list, ['nombre'])` |
| `console.group('x')` / `console.groupCollapsed('x')` / `console.groupEnd()` | Agrupa mensajes con sangría (colapsada o expandida) |
| `console.clear()` | Limpia la consola |
| `console.dir(objeto)` | Muestra la estructura real de un objeto/DOM (árbol expandible) |
| `console.dirxml(elemento)` | Representación XML/HTML del nodo (como `log` para elementos DOM) |
| `console.trace()` | Imprime el stack de llamadas desde ese punto |
| `console.assert(condición, msg)` | Si la condición es **falsy**, imprime el error; si es truthy, no imprime nada |
| `console.profile('x')` / `console.profileEnd('x')` | Inicia/finaliza un perfil (Chrome; ver pestaña Performance) |
| `console.memory` | (Chrome) heap usage: objetos, nodos DOM, listeners |
| `console.context(...)` | (Chrome) añade contexto estructurado a logs relacionados |

Ejemplos prácticos:

```js
// Contar ejecuciones: útil dentro de bucles
for (let i = 0; i < 3; i++) {
  console.count('iteraciones');
}
// iteraciones: 1
// iteraciones: 2
// iteraciones: 3

// Ver datos tabulares
const usuarios = [
  { nombre: 'Daniel', edad: 22 },
  { nombre: 'Danna', edad: 7 },
];
console.table(usuarios);
// ┌─────────┬──────────┬──────┐
// │ (index) │  nombre  │ edad │
// ├─────────┼──────────┼──────┤
// │    0    │ 'Daniel' │  22  │
// │    1    │  'Danna' │   7  │
// └─────────┴──────────┴──────┘
console.table(usuarios, ['nombre']); // solo columna nombre
```

### Agrupación y trazas

```js
console.group('Checkout');
console.log('Usuario: Ana');
console.group('Pago');
console.log('Tarjeta **** 1234');
console.log('Aprobado');
console.groupEnd();
console.groupEnd();

console.groupCollapsed('Detalle pesado'); // arranca colapsado
console.log(JSON.stringify(usuarios));     // visible al expandir
console.groupEnd();

function medio() { console.trace('¿quién me llamó?'); return 1; }
medio();
// Trace: ¿quién me llamó?
//     at medio (app.js:2)
//     at app.js:4    ← la pila revela el origen exacto
```

### `console.assert`: validación inline sin if

```js
function dividir(a, b) {
  console.assert(b !== 0, 'dividir: b no puede ser 0, se recibió', b);
  return b === 0 ? NaN : a / b;
}
dividir(10, 0);
// ✖ Assertion failed: dividir: b no puede ser 0, se recibió 0
```

> [!tip] Mostrar objetos
> `console.log(objeto)` a veces imprime `[object Object]` resumido. Con `console.table` ves los datos en filas, y con `console.dir` ves las propiedades y métodos reales del objeto (útil con elementos del DOM: `log` muestra el HTML renderizado, `dir` la instancia `HTMLElement` con todas sus propiedades).

## Formato de salida: especificadores y `%c`

```js
const user = 'Ana', edad = 28;

console.log('Hola %s, tienes %d años', user, edad); // %s string, %d/%i number, %f float
console.log('Objeto: %o', { a: 1 });   // %o: objeto expandible
console.log('Crudo: %O', { a: 1 });    // %O: expansión forzada (sin resumen del navegador)

// %c aplica CSS al texto restante del mensaje
console.log('%cImportante', 'color: red; font-size: 20px; font-weight: bold');
```

| Placeholders | Resultado |
| ------------ | --------- |
| `%s` | `String(valor)` |
| `%d` / `%i` | `Number(valor)` (entero) |
| `%f` | `Number(valor)` (decimal) |
| `%o` | Objeto expandible (el navegador decide la representación) |
| `%O` | Objeto expandible con representación completa |
| `%c` | Estilo CSS para el resto del mensaje |

> [!warning] `log` con argumentos separados ≠ concatenado
> `console.log('total:', precio)` guarda la **referencia** de `precio` y la inspecciona al expandir (ver "trampas" abajo); `console.log('total: ' + precio)` congela el valor como string en el momento de la llamada. Ambas son útiles según lo que quieras observar.

## El puente con errores y el stack trace

Cuando el código falla, el navegador imprime el error en rojo con el archivo y la línea exacta:

```js
// ReferenceError: saludo is not defined
console.log(saludo);

// TypeError: producto.noExiste.toUpperCase is not a function
const producto = {};
producto.noExiste.toUpperCase();
```

Anatomía de un error en la consola:

```
 TypeError: Cannot read properties of undefined (reading 'toUpperCase')
      at app.js:14:9     ← el frame exacto (archivo:línea:columna)
      at index.html:5    ← quién llamó a ese frame
```

Cada línea del stack es un **frame**: el primero es donde ocurrió el fallo; los siguientes, la cadena de llamadas que llegó hasta ahí. Clickear un frame abre el Sources en esa línea. Tres familias de errores se ven todo el tiempo:

| Error | Causa típica | Arreglo |
| ----- | ------------ | ------- |
| `ReferenceError: x is not defined` | variable inexistente (o en TDZ de `let`/`const`) | declarar / verificar nombre (ver [[Sintaxis Moderna]]) |
| `TypeError: x is not a function` | llamar a algo que no es función (`undefined`, typo) | revisar el valor con `console.log` antes de llamar |
| `SyntaxError: Unexpected token` | el archivo **entero** no parsea (llave sin cerrar, coma faltante) | revisar la línea del stack; el engine aborta todo el archivo |

También la consola **captura errores de red** (errores 404/500 de `fetch` o scripts que no se descargan) — aunque el detalle real está en la pestaña Network. Errores no capturados disparan además el evento `window.onerror` / `unhandledrejection` (ver [[Eventos]]): las librerías de monitoreo (Sentry, etc.) se enganchan ahí.

> [!warning] Errores silenciosos con tipos
> Operar con tipos mezclados no siempre lanza error: `"20" - 5` devuelve `15` (conversión implícita), mientras que `"20" * 5` también funciona pero `"20" + 5` concatena como texto. Un `NaN` propagado por una división rara **no** es un error — no hay mensaje, solo `NaN` viajando por tu programa. Ver [[Operadores]] y [[Números]] para las conversiones correctas.

## Trampas clásicas de la consola

### 1. Lazy evaluation: el objeto que "cambia después"

Los objetos e imágenes del DOM se muestran **como referencia**: la consola los inspecciona cuando los expandís, no cuando los imprimiste.

```js
const config = { debug: true };
console.log(config);              // al expandir: puede mostrar un estado MODIFICADO
config.debug = false;             // la referencia siguió viva
console.log(JSON.stringify(config)); // ← snapshot congelado: '{"debug":false}'
```

Regla: para congelar el estado en el momento exacto, serializá — `JSON.stringify(obj)` (ver [[Formato Json]]) o `structuredClone(obj)` (navegadores modernos). Para inspeccionar el estado **actual**, la referencia viva es justo lo que quieres.

### 2. Objetos DOM: `log` vs `dir`

```js
const h1 = document.querySelector('h1');
console.log(h1);  // imprime el HTML como se ve AHORA en la página
console.dir(h1);  // imprime la instancia HTMLElement: puedes expandir .classList, .dataset, etc.
```

### 3. La consola recuerda el contexto, no el closure

El snippet que escribes en la consola corre en el **scope global** de la página. No puedes acceder a las `const` de un `<script type="module">` ni a variables dentro de closures — esos scopes están aislados. Si necesitás exponer algo: `window.miVar = miVar` temporalmente, o engancharte con `debugger`.

### 4. `console.log` de un nodo DOM que luego se elimina

Si el nodo se saca del árbol después del `log`, la consola puede mostrarlo tachado/disconnected — es el mismo mecanismo de referencia viva.

## Live expressions y Snippets

**Live expressions** (Chrome): escribís una expresión en la barra superior de la consola con el ícono de ojo — se **re-evalúa automáticamente** en cada actualización y muestra siempre el último valor. Ideal para vigilar un estado durante interacciones:

```
 document.querySelectorAll('.item').length
 performance.now()
 store.getState().user.name
```

**Snippets** (DevTools → pestaña Sources → Snippets): archivos de JS guardados que se ejecutan sobre la página actual con un clic o `Ctrl + Enter`. Sirven para:

- scripts one-off (llenar un form de prueba, scrapear el DOM visible, desactivar animaciones);
- "runner" de utilidades repetitivas sin crear archivos en el proyecto.

La pestaña **Console** también permite **seleccionar el contexto de ejecución**: el desplegable de la izquierda (por defecto "top") cambia a qué frame/página evalúa lo que escribís (útil con iframes y extensiones).

## Depuración con puntos de interrupción (más allá del log)

El flujo profesional combina `console.log` con el **depurador** de la pestaña Sources:

1. **Breakpoint**: clic en el número de línea — la ejecución se pausa ahí.
2. Al pausar: panel **Scope** (todas las variables del frame), **Watch** (expresiones siguiendo), **Call Stack** (quién llamó a quién) y ejecución paso a paso: `F8` (continuar), `F10` (step over), `F11` (step into), `Shift+F11` (step out).
3. **Conditional breakpoint**: clic derecho en el punto → condición; solo pausa si se cumple (p. ej. `i === 47` en un bucle de 1000 iteraciones).
4. **Logpoint**: punto de interrupción que **imprime** en vez de pausar — un `console.log` sin tocar el código.
5. **`debugger`** en el código: pausa automáticamente si las DevTools están abiertas; en producción se elimina en el build.

```js
for (let i = 0; i < 1000; i++) {
  const raro = procesar(i);
  if (raro === undefined) {
    debugger; // pausa aquí solo cuando algo sale mal — inspeccioná todo el estado
  }
}
```

> [!tip] Estrategia de debugging por capas
> 1. `console.log` para confirmar **qué** valor llega mal. 2. Breakpoint para ver **por qué** (estado completo + call stack). 3. Logpoints cuando el problema es intermitente y pausar lo rompe. El error stack de la consola ya te dice dónde; el breakpoint te dice el **camino**.

## Depuración consciente de la red

La consola se apoya en la pestaña **Network** para depurar el flujo de datos:

- Errores de `fetch` aparecen en rojo en ambas pestañas; el detalle (status, headers, body) está en Network.
- **Preserve log**: mantiene los mensajes entre navegaciones (sin esto, cada `location.href = ...` borra la consola).
- Filtros por nivel (Errors/Warnings/Info/Verbose) y por texto en el input de la consola.
- `console.table` sobre una lista de respuestas de API es la forma más rápida de inspeccionar un payload ([[Formato Json]] para la estructura, [[Qué es una API]] para el flujo).

En Node, el equivalente de red es inspeccionar respuestas con `console.log` directo o `console.dir(respuesta, { depth: null })` para no truncar objetos anidados.

## Performance: medir de verdad

Niveles de precisión, de menor a mayor:

1. `console.time` / `performance.now()` — micro-benchmarks de fragmentos. `performance.now()` (navegador) da ms con precisión de sub-milisegundo desde el arranque de la página:

```js
const t0 = performance.now();
// ... código a medir ...
console.log(`tomó ${(performance.now() - t0).toFixed(2)} ms`);
```

2. **Performance recording** (DevTools → pestaña Performance → botón Record → reproducir la interacción → Stop): graba un **timeline** con frame por frame: *Main* (hilo principal, tareas JS), *Rendering* (layout/paint), *Networking*, *Timings*. Permite encontrar:
   - funciones calientes (largas barras rojas = tareas largas);
   - *layout thrashing* (reflows repetidos por leer/escribir medidas alternadamente);
   - causas de jank (>16.6ms por frame = menos de 60fps).
3. **Memory recording** + `console.memory`: detectar fugas — heap creciendo sin cedo, nodos DOM que nunca se liberan, listeners acumulados.

En Node el equivalente es `node --prof` / `node --inspect` + `console.profile` para CPU profiles.

## Consola en Node.js

Si instalas [Node.js](https://nodejs.org), la consola existe también en la terminal: ejecutas un archivo con `node app.js` y todo lo que imprimas con `console.log` aparece ahí. Útil para practicar lenguaje puro sin navegador.

- `node` solo abre un **REPL** con el mismo modelo (estado persistente, `.help`, `.exit`, `.break`).
- `node --watch app.js` re-ejecuta el archivo en cada guardado.
- `node --inspect app.js` habilita el **depurador** de Chrome DevTools sobre Node (`chrome://inspect`).
- En Node, `console.log(obj)` usa `util.inspect` con color y profundidad limitada: `console.dir(obj, { depth: null })` muestra todo, y `JSON.stringify(obj, null, 2)` da el JSON formateado ([[Formato Json]]).
- `console.table` existe desde Node 10 e imprime una tabla ASCII en la terminal, sin la paginación y expansión interactiva del DevTools.

> [!warning] `console.log` en producción
> Los logs masivos de objetos son costosos: serializar y renderizar millones de líneas degrada la consola y el rendimiento. En apps de producción se usan librerías de logging (pino, winston) y `console.log` queda solo para desarrollo — o se elimina del bundle con minificadores configurados.

## Errores comunes

1. **Confiar en el aspecto visual del log de un objeto** — es una referencia viva; el estado al expandir puede haber cambiado (ver trampa #1).
2. **`console.log(obj.prop)` lanza TypeError** — el objeto no tiene esa propiedad o es `undefined`; logueá el objeto entero antes.
3. **Perder el log entre navegaciones** — activar *Preserve log*.
4. **Medir rendimiento con `console.time` sobre código microscópico** — el overhead del propio `log` domina la medida; usá `performance.now()` o la pestaña Performance.
5. **Copiar un log y ver "[object Object]"** — copiar como **JSON** (menú contextual → Copy object / Save as) o loguear con `JSON.stringify` antes.
6. **Sobre-reliance en logs** — para estado profundo y condicional, un breakpoint condicional o un logpoint es más rápido y preciso que diez `console.log` temporales.

## Cheatsheet

| Necesidad | Herramienta |
| --------- | ----------- |
| Ver un valor | `console.log` / `%o` `%O` `%c` |
| Ver datos en filas | `console.table(lista, [cols])` |
| Ver estructura DOM/objeto | `console.dir` |
| Contar ejecuciones | `console.count` / `countReset` |
| Medir un fragmento | `console.time` / `timeLog` / `timeEnd`, `performance.now()` |
| Agrupar logs | `console.group` / `groupCollapsed` / `groupEnd` |
| Validar sin `if` | `console.assert(cond, msg)` |
| Ver la cadena de llamadas | `console.trace()` o el stack del error |
| Vigilar una expresión en vivo | Live expressions (ojo en la barra) |
| Código reutilizable one-off | Snippets (Sources → Snippets) |
| Pausar y inspeccionar | Breakpoints, condicionales, logpoints (`debugger`) |
| Grabar rendimiento | Pestaña Performance |
| REPL en terminal | `node` / `node --inspect` |

## Relacionadas

- [[Java Script]] — cómo se carga JS en una página
- [[Sintaxis Moderna]] — `const`/`let` para las variables que pruebes en consola
- [[Números]] — operaciones que puedes probar en el REPL (y el `NaN` silencioso)
- [[Operadores]] — comparaciones y qué devuelve cada una
- [[Estructuras de Control]] — bucles donde `console.count` es útil
- [[Document Object Model (DOM)]] — el árbol que inspeccionás con `dir`
- [[Eventos]] — `unhandledrejection` y el ciclo de vida donde pausan los breakpoints
- [[Formato Json]] — congelar estados con `JSON.stringify`
- [[Qué es una API]] — respuestas de red que inspeccionar con `table`
- [[Recomendaciones]] — buenas prácticas donde el logging entra
