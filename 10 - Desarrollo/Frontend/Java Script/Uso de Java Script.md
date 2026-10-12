---
tags:
  - desarrollo
  - frontend
  - java-script
tipo: concepto
estado: terminado
---

# Uso de Java Script

JavaScript es el **único lenguaje que el navegador ejecuta de forma nativa** (junto a CSS, que es declarativo). Esa exclusividad define todo lo demás: cualquier interacción — validar un formulario, actualizar un contador, animar una sección, decidir qué mostrar — pasa por JS. Esta nota ubica **dónde vive JS** en un proyecto: cómo se carga, qué tipo de apps permite, en qué stack encaja y, sobre todo, **cuándo NO hace falta**.

## ¿Qué podemos hacer con JavaScript?

Con JavaScript es posible hacer una gran cantidad de cosas como las siguientes:

- **Recolectar datos** del usuario:

```jsx
const nombre = prompt("Cual es tu nombre: ");
```

![[Pasted image 20250125075758.png]]

Podemos recolectar información y se almacenará dentro de la variable `nombre`. `prompt` es el ejemplo más crudo: en producción se usan `<form>` + `input` (semántica y accesibilidad gratis — ver [[Html]]), y el valor llega por `input.value` o `FormData`.

- **Seleccionar y modificar elementos en HTML** — el corazón de todo frontend (ver [[Document Object Model (DOM)]] y [[Acceder a Html]]):

```jsx
const nombre = prompt("Cual es tu nombre: ");
document.querySelector(".contenido").innerHTML = `${nombre} esta aprendiendo JavaScript Moderno`;
```

![[Pasted image 20250125075817.png]]

El objetivo es obtener información y luego insertarla en el HTML **sin modificar el archivo .html**: el contenido se decide en tiempo de ejecución.

> [!warning] `innerHTML` con datos de usuario
> En el ejemplo, si el nombre ingresado fuera `<img src=x onerror=alert(1)>`, el navegador lo ejecutaría: es **XSS**. Con datos dinámicos usá `textContent` (texto plano) o construí nodos con `createElement`. Regla completa en [[Recomendaciones]].

El mapa completo de capacidades: leer/escribir el DOM, reaccionar a [[Eventos|eventos]] (clics, teclado, scroll, sensores), almacenar datos (`localStorage`, cookies), hacer red (`fetch` → [[Qué es una API]]), dibujar (Canvas/WebGL), audio/video, trabajos en segundo plano (Web Workers) y hasta entrada/salida de archivos.

## Dónde se coloca el `<script>`

La posición y los atributos del `<script>` deciden **cuándo se descarga** y **cuándo se ejecuta** el código — y eso define si la página se bloquea o no. Detalle del modelo en [[Java Script]].

| Forma | Descarga | Ejecución | ¿Bloquea el parseo? | Uso |
|---|---|---|---|---|
| `<script src="a.js">` en `<head>` | Inmediata, en orden | Antes del HTML de abajo | **Sí** — página en blanco hasta que baja y corre | Casi nunca hoy |
| `<script src="a.js">` antes de `</body>` | Inmediata | Después del HTML (el DOM ya existe) | No (el parseo ya avanzó) | Forma clásica segura |
| `defer` (en `<head>`) | En paralelo | Al terminar el parseo, **en orden** | No | **Default recomendado** para scripts propios |
| `async` (en `<head>`) | En paralelo | En cuanto baja, **a destiempo** | No — pero rompe el orden | Scripts de terceros independientes (analytics) |
| `<script type="module">` | En paralelo (como `defer`) | Tras el parseo, como módulo | No | Código moderno con `import`/`export` |

```html
<!-- Orden correcto: GSAP primero, código propio después (ambos defer = en orden) -->
<script defer src="gsap.min.js"></script>
<script defer src="main.js"></script>

<!-- Módulos: import/export nativos, se cargan con prioridad diferida -->
<script type="module" src="app.js"></script>

<!-- Tercero independiente: no bloquea ni afecta a lo demás -->
<script async src="https://analytics.ejemplo.com/script.js"></script>
```

> [!info] La regla de oro del orden
> `defer` respeta el orden de aparición; `async` no. Por eso **nunca** se mezclan `async` y `defer` para scripts que dependen entre sí: si `main.js` usa `gsap` y baja tarde, explota con `gsap is not defined`.

### Módulos: `<script type="module">`

Además de diferir la carga, el atributo `type="module"` convierte el archivo en **módulo ES**: tiene su propio scope (sus variables no se cuelgan en `window`), se ejecuta en modo estricto por defecto y habilita `import`/`export`:

```js
// app.js — el navegador resuelve los paths relativos, sin bundler
import { calcularTotal } from './utils/precios.js';
import gsap from './vendor/gsap.min.js';
```

Los módulos exigen servidor HTTP (no funcionan con `file://` doble-cliqueado) y el navegador los trae con prioridad de red *diferida*. En proyectos grandes esta capa la automatiza el bundler (abajo).

## Arquitecturas de aplicación

La misma tecnología soporta modelos muy distintos — elegir bien evita el bug clásico de "usar un avión para ir a la esquina":

| Modelo | Qué renderiza | JS necesario | Ventaja | Costo | Ejemplo |
|---|---|---|---|---|---|
| **MPA clásico** (multi-page app) | El **servidor** (HTML completo por página) | Solo interacción (progressive enhancement) | SEO gratis, simple, robusto | Recargas completas entre páginas | Blogs, e-commerce tradicional |
| **MPA + islands/HTMX** | Servidor + islas de interactividad | Solo en las islas | HTML liviano, interactividad donde importa | Curva de las herramientas server | Sitios editoriales modernos |
| **SPA** (single-page app) | El **cliente** (JS pinta todo; el servidor solo manda datos) | Alto (todo el runtime vive en el navegador) | Transiciones fluidas, estado rico | SEO a gestionar, JS pesado, complejidad | Gmail, Trello, [[ReactJS - Fundamentos\|React]] apps |
| **SSR/híbrido** (Next/Nuxt) | Servidor **por request**, hidrata con JS | Medio-alto | Lo mejor de ambos | Complejidad de dos mundos | E-commerce moderno, portales |

**Progressive enhancement** (mejora progresiva) es la filosofía de partida: construí la versión funcional con HTML/CSS y agregá JS encima. Un formulario que funciona sin JS con `action`/`method` y gana enriquecimiento con JS es una app que **no se rompe** si el JS falla o tarda.

> [!tip] Preguntá "¿cuánto JS se justifica acá?"
> Una landing de 5 páginas estáticas no necesita un runtime de 300 KB. Un panel con drag & drop, gráficos en vivo y estado compartido entre vistas casi que sí. La arquitectura se elige por la **interacción**, no por moda.

## JS en el stack moderno

Un frontend actual combina capas, y JS es el pegamento:

| Capa | Herramienta típica | Rol del JS | Nota en el vault |
|---|---|---|---|
| Lenguaje | JavaScript (ES2024+) | Todo | [[Java Script]], [[Sintaxis Moderna]] |
| Bundler/dev server | **Vite**, esbuild, Rollup | Compila módulos, sirve en dev con HMR, empaqueta para prod | [[ReactJS - Vite]] |
| Framework de UI | **React**, Vue, Svelte | Declarar vistas que reaccionan a datos | [[ReactJS - Fundamentos]] |
| Estilos | **TailwindCSS**, CSS moderno | Tailwind se genera mirando el JS/HTML (JIT) | [[TailwindCSS - Sintaxis Básica]] |
| Datos | TanStack Query, SWR | Caché y estados de `fetch` | [[Qué es una API]] |
| Animación | CSS / **GSAP** | Ver cuándo vale la pena | [[Animaciones]], [[Librería GSAP]] |
| Tests | Vitest, Playwright | Confianza para refactorizar | |

**Vite** merece una mención: en desarrollo sirve los módulos ES **sin bundlear** (el navegador importa directo; arranque instantáneo) y en producción usa Rollup/esbuild para empaquetar. Es el estándar de facto para proyectos nuevos — ver [[ReactJS - Vite]].

## Más allá del navegador

JS no empezó en el navegador ni terminó ahí:

- **[[Node JS]]**: JS en el servidor (V8). Mismo lenguaje que el frontend → un equipo, dos lados del `fetch`. Ahí **no existe CORS** ni DOM: hay `fs`, `http`, bases de datos.
- **Deno** y **Bun**: runtimes alternativos con TS nativo, APIs modernas y arranque ultrarrápido; Bun además bundler y test runner.
- **Edge/worker**: lógica en el punto de entrega (Cloudflare Workers) — JS corriendo en cientos de ubicaciones geográficas.
- **Escritorio** (Electron: VS Code), **móvil** (React Native), **IoT**, **CLI**, **Bases de datos** (Stored procedures en CouchDB).

El hilo conductor: JS como **lenguaje universal de la capa de interacción y orquestación** — no necesariamente el mejor para cómputo pesado (ver abajo).

## Cuándo NO usar JavaScript

| Situación | Mejor camino | Por qué |
|---|---|---|
| Página estática informativa | HTML + CSS (o generador estático) | JS añade peso, fallos y retraso donde no aporta interacción |
| Contenido que debe existir **sin JS** | SSR / HTML del servidor | Un JS caído o lento deja pantallas en blanco; el SEO y la accesibilidad sufren |
| Cómputo pesado (ML, video, cifrado masivo) | Servidor o WASM | El hilo principal del navegador es único: un bucle grande **congela la UI** |
| Lógica sensible (pagos, permisos, reglas de negocio) | **Servidor** | El cliente se saltea con DevTools o `curl` (ver [[Qué es una API]]) |
| Estilos y layout | CSS | CSS es declarativo, más rápido y sin hilo de ejecución |

> [!info] El error de "hacer todo con JS"
> La tentación SPA-everything produce apps con 500 KB de JS para mostrar texto. La pregunta correcta no es "¿puedo?" sino "¿qué gano?" — a veces la respuesta es "una recarga de página", y está bien.

## Presupuesto de rendimiento: pensar en kilobytes

El navegador solo puede ejecutar JS **después** de descargarlo, parsearlo, compilarlo y correrlo — todo en **un solo hilo** compartido con el render y los eventos.

| Concepto | Regla práctica |
|---|---|
| **Presupuesto de JS** | Define un tope por página (p. ej. 170 KB gzipeados para landing) y medí contra él |
| **Ruta crítica** | Lo que el usuario ve primero no debe esperar JS: HTML + CSS primero, JS después (`defer`) |
| **Split por ruta** | Los bundlers (Vite) parten el código por página: `import()` dinámico carga solo lo necesario |
| **Tree-shaking** | Importar módulos, no el bundle entero (`import { gsap } from 'gsap'`, no un CDN global en todo) |
| **Costo de ejecución** | Medir no solo descarga: el `main thread blocking` de DevTools (Performance) muestra cuadro a cuadro qué congela la UI |
| **Core Web Vitals** | LCP (contenido principal rápido), INP (respuesta a interacción < 200 ms), CLS (sin saltos) |

Hábitos que sostienen el presupuesto: perezosidad (`defer`, rutas dinámicas), delegación de eventos y lotes de DOM (ver [[Recomendaciones]]), `requestAnimationFrame` para animación (ver [[Librería Parallax]]), y medir **antes** de optimizar.

## Del `<script>` a la pantalla: el flujo de carga

El ciclo completo de un script clásico en la página:

1. **Fetch**: el navegador descarga el archivo (paralelo si es `defer`/`async`/módulo).
2. **Parse/compile**: V8 parsea y compila el JS a bytecode/máquina.
3. **Execute**: corre el código (acá se leen/select DOM si el parser ya terminó).
4. **Render**: el navegador pinta el resultado de los cambios de DOM.

Los eventos que marcan cada hito:

| Evento | Cuándo dispara | Uso típico |
|---|---|---|
| `DOMContentLoaded` (`DOMContentLoaded`) | El HTML terminó de parsear; el DOM existe | Arrancar la app cuando no hay `defer` |
| `load` (`window.onload`) | DOM **+** todos los recursos (imágenes, CSS, fuentes) | Medición de performance, preloader |
| `defer` script | Automáticamente, justo **antes** del `DOMContentLoaded` | El porqué de no necesitar listeners |

```js
// Sin defer, el script en head corre ANTES del DOM: hay que esperar
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', iniciar);
} else {
  iniciar();   // el DOM ya existe (script diferido o al final del body)
}
```

> [!tip] `defer` elimina el boilerplate
> Un `<script defer>` corre después del parseo, así que adentro el DOM ya existe — `DOMContentLoaded` es innecesario. Es una de las razones prácticas para declararlo por defecto.

## La superficie de APIs del navegador

JS en el navegador no es "solo lenguaje": el runtime expone APIs listas para usar. Mapa de las principales familias:

| Familia | APIs | Nota en el vault |
|---|---|---|
| DOM | `document`, `querySelector`, `createElement` | [[Document Object Model (DOM)]], [[Acceder a Html]] |
| Eventos | `addEventListener`, Custom Events, `PointerEvent` | [[Eventos]] |
| Red | `fetch`, `WebSocket`, `XMLHttpRequest` | [[Qué es una API]] |
| Almacenamiento | `localStorage`, `sessionStorage`, IndexedDB, cookies | datos locales; no para secretos |
| Multimedia | `<canvas>`, Web Audio, WebRTC, MediaStream | cámara/micrófono con permiso |
| Gráficos | Canvas 2D/WebGL, **SVG** | [[Archivos SVG]] |
| Animación | CSSOM, Web Animations API, `requestAnimationFrame` | [[Animaciones]], [[Librería GSAP]] |
| Trabajo en segundo plano | Web Workers, Service Workers | no bloquean el hilo de UI |
| Sensores/entrada | `geolocation`, `deviceorientation`, `IntersectionObserver`, `ResizeObserver` | el parallax de [[Librería Parallax]] usa los de scroll/pointer |
| Utilidades | `Intl`, `Date`, `Math`, JSON, `crypto.subtle` | [[Números]], [[Strings]], [[Formato Json]] |

`IntersectionObserver` y `ResizeObserver` merecen mención: resuelven "¿está visible?" y "¿cambió de tamaño?" **sin** listeners de scroll/resize costosos — la forma moderna de lazy-load y reveals (el mismo rol que cumple ScrollTrigger en [[Librería GSAP]]).

## Receta: progressive enhancement de un formulario

La filosofía en la práctica — una suscripción a newsletter que funciona sin JS y gana UX con él:

```html
<!-- 1) Base: funciona con solo HTML (action a un backend real) -->
<form id="suscripcion" action="/api/newsletter" method="post">
  <label for="email">Email</label>
  <input id="email" name="email" type="email" required />
  <button>Suscribirme</button>
</form>
<p id="estado" role="status"></p>
```

```js
// 2) Enriquecimiento: si el JS corre, interceptamos y mejoramos
//    (script con defer → el DOM ya existe)
document.querySelector('#suscripcion').addEventListener('submit', async (e) => {
  e.preventDefault();                     // sin JS, el POST normal ocurre solo
  const form = e.target;
  const estado = document.querySelector('#estado');
  estado.textContent = 'Enviando…';
  try {
    const res = await fetch(form.action, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(Object.fromEntries(new FormData(form))),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    estado.textContent = '¡Listo! Revisá tu email.';   // sin recarga
    form.reset();
  } catch {
    estado.textContent = 'No se pudo enviar — probá de nuevo.';
  }
});
```

Sin JS: el navegador valida (`type="email"`, `required`) y hace el POST clásico. Con JS: validación instantánea, resultado sin recarga y estado accesible (`role="status"` announce a los lectores de pantalla). Ningún escenario deja al usuario en callejón sin salida.

## Errores comunes

1. **Script en `<head>` sin `defer`**: el parser se frena y la página aparece en blanco mientras baja el JS.
2. **Mezclar `async` y `defer` en scripts dependientes**: pierde el orden y explota `X is not defined`.
3. **Ejecutar JS antes del DOM**: `querySelector` devuelve `null` porque el HTML de abajo aún no existe → esperar a `defer`/`DOMContentLoaded` (ver [[Java Script]]).
4. **`file://` con módulos**: los `<script type="module">` no cargan con doble clic; hace falta servidor (`vite`, `python -m http.server`…).
5. **SPA donde bastaba HTML**: presupuesto de JS disparado, SEO a remediar, y una recarga de página habría resuelto el 90% de la UX.
6. **Lógica sensible en el cliente**: se saltea con DevTools — el servidor es la única frontera de seguridad.
7. **No medir**: optimizar de ojo kilobytes que no molestaban, ignorando los que sí.

## Cheatsheet

| Pregunta | Respuesta corta |
|---|---|
| ¿Dónde va el `<script>` propio? | `<head>` con `defer`, o antes de `</body>` |
| ¿Cuándo `async`? | Solo scripts de terceros independientes |
| ¿Cómo organizar código? | `<script type="module">` + `import`/`export` |
| ¿Cuándo JS puro (sin framework)? | Poco DOM, poca interacción, presupuesto ajustado |
| ¿Cuándo MPA? | Contenido primero, interacción moderada |
| ¿Cuándo SPA/híbrido? | Estado rico, muchas transiciones, app-like |
| ¿Dónde la lógica sensible? | **Servidor**, siempre |
| ¿Cuánto JS? | El mínimo que resuelva la interacción — y medido |
| ¿Fuera del navegador? | [[Node JS]], Deno, Bun, edge, escritorio/móvil |

## Relacionadas

- [[Java Script]] — el lenguaje, su sintaxis y dónde se carga el código
- [[Sintaxis Moderna]] — la JS que se escribe hoy
- [[Document Object Model (DOM)]] — el árbol que JS manipula en el navegador
- [[Acceder a Html]] — seleccionar y modificar elementos
- [[Eventos]] — cómo reacciona el JS al usuario
- [[Qué es una API]] — salir del navegador: red, `fetch` y backends
- [[Recomendaciones]] — seguridad (XSS), rendimiento DOM y buenas prácticas
- [[ReactJS - Fundamentos]] — el modelo SPA/islas con componentes
- [[ReactJS - Vite]] — el bundler/dev server del stack moderno
- [[Node JS]] — el mismo JS en el servidor
- [[TailwindCSS - Sintaxis Básica]] — la capa de estilos del stack
- [[Html]] — la base sobre la que todo esto se apoya
