---
tags:
  - desarrollo
  - frontend
  - modernizr
tipo: concepto
estado: terminado
---

# Modernizr

**Modernizr** es una librería de JavaScript para **detección de compatibilidad** (*feature detection*): analiza el navegador en el que se está ejecutando la página y expone, tanto en CSS como en JavaScript, qué capacidades soporta (WebP, AVIF, `localStorage`, `intersectionobserver`, etc.) para poder mostrar una versión adecuada de la página a cada navegador.

> [!info] El principio por encima de la herramienta
> Lo importante no es Modernizr en sí, sino el patrón: **detectar capacidades, no identificar navegadores**. La librería es solo una forma conveniente de aplicar ese patrón — y en 2026, en muchos casos, la forma *nativa* la reemplaza (ver la tabla de alternativas al final).

## Detectar capacidades, no navegadores

La tentación clásica es adivinar el navegador por su *user agent*; eso es frágil. Por qué el UA sniffing falla siempre:

| Motivo | Ejemplo real |
|---|---|
| El string **miente** para arreglar bugs | Navegadores que se anuncian como Chrome/WebKit para pasar sniffing |
| Existen **forks y dispositivos raros** | navegadores de smart TV, kioscos, apps webviews, Opera (historicamente Blink disfrazado de Chrome) |
| Las versiones **cambian el string** | Chrome 100 rompió detecciones por regex de 2 dígitos |
| **Client Hints** lo reemplazan | UA-CH (`navigator.userAgentData.brands`) da marcas sin números completos por privacidad |
| El navegador **nuevo** no está en tu lista | Cada release obliga a actualizar la lógica |
| Soportar navegador ≠ soportar feature | Un Chromium viejo de TV puede tener todo salvo una API concreta |

Lo correcto es preguntar **"¿sabe el navegador hacer esto?"**:

```js
// ❌ Sniffing de navegador: frágil y siempre desactualizado
if (navigator.userAgent.includes('Trident')) { /* asumir IE viejo */ }

// ✅ Feature detection: responde a lo que importa
if ('IntersectionObserver' in window) { /* usar API moderna */ }
```

Modernizr lleva esa idea a escala: en lugar de escribir decenas de comprobaciones a mano, generás un build con las que te interesan.

## Qué hace Modernizr por dentro

Al cargar, la librería ejecuta cada test incluido en el build y:

1. **Agrega una clase al elemento `<html>`** por cada test: `.webp` si el navegador soporta WebP, `.no-webp` si no (el prefijo `no-` siempre negación).
2. **Reemplaza la clase `no-js`** del elemento raíz por `js` — la convención es escribir `class="no-js"` en la etiqueta `<html>` desde el HTML: sirve como red de seguridad para el caso de JavaScript deshabilitado (el CSS puede mostrar un banner de "necesitás JavaScript" o servir la versión sin efectos).
3. Expone cada resultado como booleano en el objeto global `Modernizr` para lógica en scripts.

Los tests se agrupan por categoría (~300 disponibles): CSS (flexbox, grid, custom properties…), HTML5 (canvas, video, geolocation…), Misceláneos (`touchevents`, `retina`), y formatos de imagen (webp, avif, jpegxl — algunos de estos últimos se ejecutan de forma **asíncrona**, ver más abajo).

## Crear un build personalizado

1. Ir a [modernizr.com](https://modernizr.com/) → *Build*.
2. Marcar solo los tests necesarios (cargar todos es innecesario y engorda el bundle).
3. Descargar el `modernizr.js` generado y guardarlo en el proyecto, por ejemplo en `js/modernizr.js`.

También desde CLI (reproducible en CI):

```bash
npx modernizr -c modernizr-config.json -d js/modernizr.js
```

```json
// modernizr-config.json — build mínimo de ejemplo
{
  "classPrefix": "",
  "options": ["setClasses", "addTest"],
  "feature-detects": ["test/css/flexbox", "test/touchevents", "test/localstorage"]
}
```

Un build con 5–10 tests pesa unos pocos KB minificado+gzip; un build completo con los 300 es desperdicio puro.

Luego se carga en el HTML:

```html
<script src="js/modernizr.js"></script>
```

> [!warning] Va en el `<head>`, no antes de cerrar `body`
> La regla general de rendimiento dice que los scripts que no son urgentes van al final del `<body>`. Modernizr es la excepción: la librería **agrega clases al elemento `<html>` que el CSS usa para decidir estilos**. Si carga tarde, el usuario ve primero los estilos equivocados y luego el cambio (un *flash of unstyled content*). El render-blocking acá es intencional y vale la pena. Ver [[Html]] para la anatomía del documento.

### Tests asíncronos: el matiz de los formatos de imagen

Los tests de imagen (`.webp`, `.avif`, `.jpegxl`) requieren **decodificar una imagen de prueba**, así que Modernizr puede resolverlos *después* del parseo: la clase puede aparecer en `<html>` con retardo. Para ese caso existe el callback:

```js
Modernizr.on('webp', (result) => {
  // result: true/false — llega cuando el test asíncrono termina
  document.documentElement.classList.add(result ? 'webp-ready' : 'webp-fallback');
});
```

> [!tip] En la práctica: no detectes formatos con JS
> Todo lo que Modernizr hace para elegir imágenes, `<picture>` + `type` lo hace **sin JavaScript y antes del primer render** — ver [[Formatos Avif y Webp]]. La detección de formatos es el ejemplo clásico que hoy está *resuelto por el estándar*.

## Las clases en `<html>`: la API de CSS

Al cargar, Modernizr agrega al elemento raíz una clase por cada test: `.webp` si el navegador soporta WebP, `.no-webp` si no. Eso permite resolver la compatibilidad **enteramente desde el CSS**, sirviendo una imagen u otra:

```css
.webp .header {
  background-image: url(../img/banner.webp);
}

.no-webp .header {
  background-image: url(../img/banner.jpg);
}
```

- Los prefijos de estado son siempre el nombre del test y su negación (`audio`, `no-audio`, `canvas`, `no-canvas`).
- Tests "apropiados" (p. ej. `.touchevents`) llevan el prefijo del *approach* configurado; por defecto son clases planas.
- El patrón completo se llama **progressive enhancement**: todos reciben una versión funcional; quien soporte más, ve la mejor.
- **Los prefijos CSS convencionales** (`-webkit-`, `-moz-`) existieron por la misma razón histórica: detectar soporte antes de que hubiera detección de features. Hoy los prefijos están retirados en los motores evergreen; ver [[CSS Introducción]] para el contexto de cascada.

## La API de JavaScript

Los mismos tests quedan disponibles como booleanos en el objeto global `Modernizr`, para lógica condicional en scripts:

```js
if (Modernizr.webp) {
  cargarBanner('./img/banner.webp');
} else {
  cargarBanner('./img/banner.jpg');
}
```

API complementaria:

| Método | Qué hace | Ejemplo |
|---|---|---|
| `Modernizr.addTest(name, fn)` | Test propio que también genera su clase CSS | `Modernizr.addTest('gestures', () => 'ongesturestart' in window)` |
| `Modernizr.on(name, cb)` | Suscribirse a un test (especialmente los asíncronos) | `Modernizr.on('avif', cb)` |
| `Modernizr.prefixed('name')` | Resuelve el nombre con prefijo de un API | `Modernizr.prefixed('requestAnimationFrame')` |
| `Modernizr.mq('(min-width: 600px)')` | Evalúa una media query (wrapper de `matchMedia`) | redirección condicional |
| `Modernizr._prefixes` / `domPrefixes` | Lista de prefijos del navegador | tests propios |

> [!warning] `Modernizr.load` ya no existe
> En Modernizr 3 se eliminó el cargador de scripts condicionales (basado en yepnope). Para carga condicional moderna: `import()` dinámico, `<script>` inyectado bajo demanda, o simplemente incluir el módulo siempre y dejar que el bundler lo tree-shake.

## Estrategia de progressive enhancement por capas

El patrón de diseño que aprovecha al máximo el `no-js` + clases:

```css
/* Capa 0: todo el mundo recibe esto (incluso sin JS) */
.layout { display: block; }

/* Capa 1: quien tiene JS (o clases Modernizr) mejora */
.js .layout { display: flex; }

/* Capa 2: quien tiene la feature concreta */
.flexbox .layout { display: flex; }
.no-flexbox .layout { display: block; }
```

Reglas de la estrategia:

1. **El HTML base debe ser funcional sin ninguna mejora**: contenido legible, navegación por enlaces reales.
2. Cada capa añade comportamento, nunca lo *mueve*: si la capa falla, la anterior sostiene la página.
3. Los polyfills van **entre** capas y se cargan solo donde la detección dice que faltan — Modernizr **detecta**, no añade soporte.
4. El test debe medir lo que se va a usar: no detectes `canvas` si no hay nada en la página que lo use.

## Alternativas nativas hoy (2026)

Modernizr sigue vivo, pero gran parte de su terreno fue cubierto por estándares nativos:

| Necesidad | Alternativa nativa | Notas |
|---|---|---|
| Detectar soporte de una **propiedad CSS** | At-rule `@supports` | Sin JS; ver ejemplo abajo |
| Detectar **selectores/condiciones CSS** | `@supports selector(:has(*))` | También media queries anidadas |
| Detectar una **API de JS** | `'api' in window` / `typeof api === 'function'` | El patrón a mano, 1 línea |
| Evaluar una **media query** | `matchMedia` / `window.innerWidth` | Reemplaza `Modernizr.mq` |
| Servir imágenes en formatos modernos | `<picture>` + `type`, **sin JS** | Ver [[Formatos Avif y Webp]] |
| Detectar AVIF/WebP vía canvas | Test propio de canvas/fetch en ~10 líneas | Solo si `<picture>` no aplica |
| Detectar soporte de CSS con precisión | `CSS.supports('propiedad', 'valor')` | El mismo `@supports` expuesto a JS |

```css
/* Detección de features en CSS puro, sin librerías */
@supports (aspect-ratio: 16 / 9) {
  .banner { aspect-ratio: 16 / 9; }
}

@supports not selector(:has(*)) {
  /* fallback para navegadores sin :has() */
}
```

```js
// Equivalente JS del @supports
if (CSS.supports('display', 'grid')) { /* ... */ }
```

> [!tip] Cuándo sigue valiendo la pena Modernizr
> Cuando hay que mantener navegadores viejos muy extendidos (entornos corporativos, kioscos, dispositivos heredados) y conviene centralizar los tests en un solo archivo con un API uniforme, o cuando el equipo ya tiene la disciplina de build para generarlo. En un proyecto solo evergreen, `@supports` + `<picture>` + comprobaciones puntuales de JS cubren el 95% de los casos.

| Situación | ¿Modernizr? |
|---|---|
| Site evergreen, frameworks actuales | **No**: peso y complejidad por nada |
| Baseline corporativo con Edge viejo/IE11 | Sí (aunque para IE11 la vía real es polyfills + transpilar) |
| Muchos tests CSS propios centralizados | Sí, como catálogo único |
| Solo elegir formato de imagen | **No**: `<picture>` lo resuelve declarativo |
| Sin JS disponible como requisito | Casi: los tests en `<html>` funcionan con el JS de Modernizr, que es justamente el requisito |

## El patrón `no-js` en la práctica

La clase `no-js` es el contrato entre el HTML y el progressive enhancement. Forma canónica:

```html
<html lang="es" class="no-js">
<head>
  <script>document.documentElement.classList.replace('no-js', 'js');</script>
  <script src="js/modernizr.js"></script>
```

- El script **inline de una línea** reemplaza la clase antes de que cualquier CSS condicional evalúe: Modernizr hace lo mismo internamente, pero el inline garantiza el estado aunque el archivo falle en cargar.
- CSS base: `.no-js .solo-con-js { display: none; }` oculta efectos que no funcionarían; el contenido sigue legible.
- Ventaja del enfoque clase por clase: **sin JS no hay clases, sin clases no hay estilos «modernos» rotos** — la degradación es automática, sin detección alguna.

## Errores comunes

1. **Olvidar `class="no-js"` en `<html>`**: la red de seguridad del progressive enhancement no existe; sin JS el usuario ve exactamente los estilos "modernos" (rotos).
2. **Poner el script al final del `<body>`**: FOUC — los estilos condicionales aplican tarde. En el `<head>`, síncrono, es la excepción correcta a la regla de scripts diferidos.
3. **Build completo (~300 tests)**: decenas de KB y decenas de pruebas de canvas/imágenes ejecutándose en el hilo principal por nada. Build mínimo siempre.
4. **Usar `.touchevents` como "es móvil"**: hay laptops con pantalla táctil (usan mouse) y tablets con teclado. `touchevents` detecta *capacidad*, no *modo de uso*; para hover real usá `@media (hover: hover)`.
5. **Duplicar lo que el estándar ya hace**: escribir JS que elija imagen cuando `<picture>` lo haría gratis (y sin parpadeo).
6. **Mezclar detección con polyfills**: Modernizr **detecta**, no **agrega** soporte; la carga de polyfills es otra capa (hoy casi siempre innecesaria).
7. **Confiar en clases asíncronas antes de que existan**: el test `.webp` puede resolver tarde; si el CSS depende de la clase para el primer render, hay FOUC. Para formatos, `<picture>`; para el resto, tests síncronos.

## Accesibilidad y robustez

- La detección de features es la base del **graceful degradation**: si un navegador no entiende una interacción (p. ej. `draganddrop`), el contenido debe seguir siendo operable por teclado y formulario clásico — ver [[Formularios]].
- El `no-js` banner debe ser legible y no atrapar al usuario: si no hay JS, el sitio entero debe seguir siendo navegable (regla de progressive enhancement).
- Feature detection también evita *loops de error*: intentar APIs inexistentes produce excepciones que rompen scripts enteros; detectar antes = degradación controlada.

## Rendimiento

- Coste del build mínimo: ~1–3 KB gzip + unas pocas pruebas síncronas (microsegundos). Coste del build completo: decenas de KB y tests de canvas que sí cuestan frames.
- El script es render-blocking por diseño (las clases guían el CSS): es el precio de no parpadear estilos. Si el catálogo de tests crece, reconsiderá si Modernizr sigue justificando ese bloqueo.
- Alternativa cero-coste: `@supports`/`matchMedia` se evalúan dentro del motor CSS, sin JS ni clasificación en el hilo principal.

## Compatibilidad

- Modernizr funciona en cualquier navegador con ES5 (sostiene hasta IE6 si el build lo pide); por eso sobrevive en entornos legacy.
- Sus *alternativas nativas* (`@supports`, `matchMedia`, `'api' in window`, `<picture>`) tienen soporte universal en los navegadores evergreen desde ~2019–2022.

## Cheatsheet

| Pieza | Función | Ejemplo |
|---|---|---|
| `class="no-js"` en `<html>` | Red sin JS | Convención obligatoria |
| `.webp` / `.no-webp` | API de CSS | `.no-webp .hero { background: url(hero.jpg) }` |
| `Modernizr.webp` | API de JS | `if (Modernizr.webp) { … }` |
| `Modernizr.on(name, cb)` | Test asíncrono | formatos de imagen |
| `Modernizr.addTest(name, fn)` | Test propio | genera clase + booleano |
| `Modernizr.prefixed()` | Nombres con prefijo | APIs históricos |
| `@supports` (nativo) | Detección CSS sin librería | reemplazo moderno |
| `'api' in window` (nativo) | Detección JS sin librería | reemplazo moderno |
| `<picture type>` (nativo) | Formatos sin detección | reemplazo moderno |

## Relacionadas

- [[Html]] — dónde vive el `<script>` y las clases en `<html>`
- [[Formatos Avif y Webp]] — fallback de imágenes sin necesidad de detección con JS
- [[CSS Introducción]] — estilos alternativos resueltos con las clases de Modernizr
- [[Java Script]] — feature detection a mano con el objeto global
- [[Document Object Model (DOM)]] — `classList` para manipular las clases de detección
- [[Formularios]] — degradación de controles cuando faltan features
