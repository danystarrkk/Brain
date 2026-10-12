---
aliases: ["Compilar Sass", "Sass con Gulp", "Pipeline de Sass"]
tags:
  - devops/gulp
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Gulp]]"
  - "[[GulpFile Completo]]"
  - "[[Qué es Sass]]"
  - "[[Sintaxis de Sass]]"
  - "[[Separar Sass en Archivos]]"
  - "[[Mixins]]"
  - "[[CSS Introducción]]"
  - "[[ReactJS - Vite]]"
fuente: "https://github.com/doublefusion/gulp-sass"
---

# Compilar Sass

Compilar Sass dentro de Gulp es **el** pipeline de styles por excelencia: `src` lee el entry `.scss`, `gulp-sass` (con **dart-sass** como compilador) lo transforma a CSS, PostCSS agrega prefijos y minifica, y `dest` escribe el resultado con sourcemaps. Esta nota desarma cada eslabón, el watch, el manejo de errores sin matar el dev server, y las alternativas reales en 2026.

## Instalación

```bash
npm i -D gulp-sass sass                      # plugin + compilador (obligatorio el par)
npm i -D gulp-postcss autoprefixer cssnano   # opcional: prefijos + minificación
```

> [!warning] `gulp-sass` no compila solo — hay que pasarle el compilador
> `gulp-sass` es un **envoltorio sin motor**: desde v5 dejó de traer compilador propio y exige que le indiques cuál usar. Ese rol lo cumple **dart-sass** (el paquete `sass`, implementación oficial de referencia en JS). La línea `const sass = gulpSass(dartSass)` es obligatoria — sin ella la tarea falla. La opción histórica era **libsass** (`node-sass`): **EOL desde 2020**, sin soporte de `@use`/`@forward` y con binarios frágiles; dart-sass es la única vía mantenida. Esta distinción es también por qué los módulos de [[Sintaxis de Sass]] y [[Separar Sass en Archivos]] usan `@use`: funciona porque el compilador es dart-sass.

## La tarea `css` mínima

```js
import { src, dest } from 'gulp';
import * as dartSass from 'sass';
import gulpSass from 'gulp-sass';

const sass = gulpSass(dartSass);

export function css() {
  return src('src/scss/app.scss')                 // 1. un SOLO entry point
    .pipe(sass().on('error', sass.logError))      // 2. compilar + no morir con errores
    .pipe(dest('dist/css'));                      // 3. escribir
}
```

Anatomía paso a paso:

| Eslabón | Qué hace | Detalle |
|---|---|---|
| `src('src/scss/app.scss')` | Lee el entry | Dart-sass resuelve `@use` **por su cuenta** leyendo el disco: al gulp solo le llega el entry compilado |
| `sass(...)` | SCSS → CSS | Instanciado con dart-sass arriba; las opciones se pasan como objeto |
| `.on('error', sass.logError)` | Manejo de errores | Loguea el error de Sass formateado (archivo:línea) y cierra el stream sin tirar abajo el watch |
| `dest('dist/css')` | Escribe | Respetando el `base` del glob (ver [[GulpFile Completo]]) |

> [!info] Por qué compilás solo `app.scss` y no cada partial
> Los partials (`_variables.scss`, `_mixins.scss`, …) se `@use`an desde el entry: dart-sass los lee y los fusiona **en memoria**. Compilar un partial suelto generaría CSS fragmentado y duplicado. El watch, en cambio, sí observa **todos** los `.scss` — ver más abajo.

## Watch: el entry compila ante cualquier cambio

```js
import { watch } from 'gulp';

export function dev() {
  watch('src/scss/**/*.scss', css);   // cualquier partial dispara la compilación del entry
}
```

| Qué observes | Cuándo usarlo |
|---|---|
| `'src/scss/app.scss'` | Solo el entry: **rompe** si editás un partial (`_variables.scss`) — no se recompila nada |
| `'src/scss/**/*.scss'` | Correcto: editás cualquier archivo del árbol y se recompila el entry |
| `['src/scss/**/*.scss', 'src/tokens/*.json']` | Varias fuentes de datos (tokens JSON, etc.) |

El costo de recompilar todo el entry en cada keystroke es de milisegundos en proyectos normales — dart-sass es rápido y no hace falta incremental.

## Errores sin romper el watch

Sin manejo de errores, un typo en un partial **mata el proceso** y el watch desaparece: guardás el archivo y no pasa nada. Por eso el `.on('error', sass.logError)`:

```js
.pipe(sass().on('error', sass.logError))
// imprime algo tipo:
// ✖ [12:03:44] 'styles' errored after 6 ms
// ✖ [12:03:44] Error: Undefined variable.
//     ╷
// 12 │   color: $primario;
//     │         ^^^^^^^^^
//     ╷
// src/scss/_base.scss:12:10
```

El stream termina limpio, el watch sigue vivo, corregís y guardás de nuevo. Alternativa genérica: `gulp-plumber` en el task completo (ver tabla de herramientas de error en [[GulpFile Completo]]).

> [!warning] En producción el error **debe** romper
> `logError` es para dev. En CI/deploy, un Sass que no compila tiene que fallar el build con código de salida ≠ 0. No agregues `logError` ciego a la tarea de prod, o deployás sin CSS.

## Sourcemaps en flujo nativo

```js
export function css() {
  return src('src/scss/app.scss', { sourcemaps: true })   // mapas en memoria
    .pipe(sass().on('error', sass.logError))
    .pipe(dest('dist/css', { sourcemaps: '.' }));         // app.css + app.css.map
}
```

- `sourcemaps: true` en `src` → Gulp crea los mapas (sin `gulp-sourcemaps`: nativo desde Gulp 4).
- `sourcemaps: '.'` en `dest` → escribe el `.map` al lado del CSS y agrega el comentario `/*# sourceMappingURL=app.css.map */`.
- Los pasos intermedios compatibles (sass, postcss) actualizan los mapas: en DevTools ves **líneas del SCSS original**, no del CSS compilado.
- Producción: desactivar (`sourcemaps: false` o no pasar la opción) — el `.map` viaja aparte si hace falta.

## Autoprefixer + minificación con PostCSS

```js
import postcss from 'gulp-postcss';
import autoprefixer from 'autoprefixer';
import cssnano from 'cssnano';

const esProd = process.env.NODE_ENV === 'production';

export function css() {
  const plugins = esProd ? [autoprefixer(), cssnano()] : [autoprefixer()];
  return src('src/scss/app.scss', { sourcemaps: !esProd })
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss(plugins))                          // PostCSS procesa el CSS ya compilado
    .pipe(dest('dist/css', { sourcemaps: esProd ? false : '.' }));
}
```

El orden importa: **primero sass** (SCSS → CSS), **después PostCSS** (transformaciones sobre CSS). Al revés no funciona — PostCSS no entiende SCSS. El resultado de dev:

```css
/* expandido, legible, con prefijos según browserslist */
.nav {
  display: flex;
  gap: 1rem;
}
```

y de prod (cssnano): `.nav{display:flex;gap:1rem}` — ver [[Performance Web]] por qué minificar.

### Browserslist: qué prefijos pone

En `package.json`:

```json
{
  "browserslist": [
    "last 2 versions",
    "> 0.5%",
    "not dead"
  ]
}
```

| Query | Significado |
|---|---|
| `last 2 versions` | Últimas 2 versiones de cada navegador |
| `> 0.5%` | Navegadores con más de 0.5% de uso mundial |
| `not dead` | Excluye navegadores sin soporte oficial (IE, iOS Safari antiguo) |
| `defaults` | Atajo: `> 0.5%, last 2 versions, Firefox ESR, not dead` |

Con queries tan holgadas de 2026, autoprefixer casi no agrega nada (Flexbox, grid, custom properties ya son baseline) — sigue siendo una red de seguridad barata. Verificación puntual: `npx autoprefixer --info` lista qué prefijos se inyectan y para qué navegadores.

## Múltiples entries: `main.scss` + `admin.scss`

Los proyectos reales suelen tener más de un punto de entrada (layout público y panel admin, emails, etc.):

```js
export function css() {
  return src(['src/scss/app.scss', 'src/scss/admin.scss'])   // entries explícitos
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([autoprefixer()]))
    .pipe(dest('dist/css'));   // dist/css/app.css + dist/css/admin.css
}
```

> [!warning] Nunca uses el glob del árbol completo como `src` de compilación
> `src('src/scss/**/*.scss')` haría que gulp **compile cada partial suelto**: `_variables.scss` no produce CSS propio, pero sí genera archivos vacíos/raros en `dist/` y duplica reglas. Los partials son entradas de dart-sass **vía `@use`**, no archivos de gulp. Listá los entries a mano (o filtrá: el convención de nombre `_*.scss` no los salva solo — hay que excluirlos con `!` en el glob).

Cada entry genera su `.css` con el mismo nombre: `admin.scss` → `admin.css`. El watch sigue igual: `watch('src/scss/**/*.scss', css)` recompila **todos** los entries ante cualquier cambio — simple y correcto; la compilación diferencia es de milisegundos.

## Estructura de salida típica

```
src/scss/app.scss          →  dist/css/app.css
src/scss/_base.scss        →  (fusionado en app.css)
src/js/app.js              →  dist/js/app.js
src/img/…                  →  dist/img/…
dist/css/app.css.map       ←  sourcemaps de dev
```

Regla práctica: `dist/` refleja `src/` para que los `url()` relativos del CSS compilado (`../img/fondo.png`) resuelvan igual que en desarrollo — ver troubleshooting.

## Rendimiento

- **Un solo entry**: dart-sass compila app.scss completo en decenas de ms; no compiles carpetas de partials.
- **dart-sass moderno**: desde sass 1.45 existe la API moderna (`compileAsync`) más rápida que la legacy (`render`); gulp-sass la adopta progresivamente. Si consignas el warning *"legacy JS API is deprecated"* — es cosmético en proyectos normales (ver troubleshooting).
- **`loadPaths`**: para `@use 'tokens'` sin ruta relativa, pasá `{ loadPaths: ['src/scss'] }` a `sass()`; resuelve sin duplicar imports ni romper el watch.
- **Watch ≠ build**: en dev el sourcemap cuesta un poco; en prod desactivalo (ya está en el switch).

## Alternativas: gulp-sass vs las opciones nativas

| Pipeline | Cómo compila Sass | Ventaja | Coste |
|---|---|---|---|
| **gulp-sass** (esta nota) | Pipeline Gulp completo | Prefijos+minify+copy+imágenes en **un solo grafo de tareas** | Mantener gulpfile |
| **sass CLI nativo** (`npx sass src:dist --watch`) | Sin Gulp | Cero config, watch propio, sourcemaps `--source-map` | Sin autoprefixer/copy/imágenes: son pasos aparte |
| **[[ReactJS - Vite]]** | El bundler compila Sass integrado | Dev server + HMR instantáneo + todo el JS bundling | Solo si el proyecto es un bundler-app; no sirve para pipeline de assets estáticos |
| **Tailwind + PostCSS** | `tailwindcss` como plugin PostCSS | Si el CSS vive en [[TailwindCSS - Instalación y Configuración]] | Otro modelo de utilidades |

Criterio: si el proyecto ya es Vite, **no** dupliques el Sass en Gulp. Si es HTML/CSS/JS estático (tema WordPress, landing, sitio multipágina), gulp-sass da el pipeline cerrado que el CLI solo no ofrece.

## Troubleshooting

| Sintoma | Causa | Arreglo |
|---|---|---|
| `Cannot find module 'sass'` | Falta el compilador | `npm i -D sass` y el `gulpSass(dartSass)` |
| `File to import not found or unreadable` | `@use` con ruta relativa desde otro dir | Rutas relativas al archivo o `loadPaths` |
| Warning *"The legacy JS API is deprecated"* | dart-sass legacy API (render) | Inofensivo; con gulp-sass moderno se silencia vía opciones del compilador |
| Warnings *"Sass @import"* | `@import` deprecado en dart-sass | Migrar a `@use`/`@forward` (ver [[Separar Sass en Archivos]]) |
| Los `url()` del CSS 404 en el sitio | El `url()` se resuelve **contra la ubicación del CSS de salida**, no del SCSS | Mantener `dist/` espejando `src/` (css en `dist/css`, imgs en `dist/img`) o absolutos |
| El watch ignora cambios en partials | Glob del watch = solo el entry | `watch('src/scss/**/*.scss', css)` |
| DevTools muestra CSS compilado, no SCSS | Sourcemaps ausentes | `sourcemaps: true` en src **y** `sourcemaps: '.'` en dest (falta uno de los dos) |
| CSS vacío tras compilar | Entry sin reglas o glob mal escrito (0 archivos) | Verificar que el entry existe y tiene reglas; `gulp --tasks` + log de `src` |
| Errores de variables con `@use` | Estilo viejo de variables globales (`@import`) mezclado con `@use` | Sintaxis completa en [[Sintaxis de Sass]] |
| `exports const cssex = css` → SyntaxError | `exports` (CommonJS) y `const` no se combinan así | En ESM: `export const cssex = css;` o `export { css };` |

## Cheatsheet

| Paso | Código |
|---|---|
| Compilador | `const sass = gulpSass(dartSass)` |
| Compilar | `.pipe(sass())` |
| Minificado (dart-sass) | `.pipe(sass({ style: 'compressed' }))` (API moderna; legacy: `outputStyle`) |
| Rutas de @use | `.pipe(sass({ loadPaths: ['src/scss'] }))` |
| Errores dev | `.on('error', sass.logError)` |
| Prefijos | `.pipe(postcss([autoprefixer()]))` |
| Minify extra | `.pipe(postcss([cssnano()]))` |
| Sourcemaps | `src(g, { sourcemaps: true })` → `dest(d, { sourcemaps: '.' })` |
| Watch total | `watch('src/scss/**/*.scss', css)` |
| Probar queries | `npx autoprefixer --info` |

## Relacionadas

- [[Gulp]] — streams y composición de tareas
- [[GulpFile Completo]] — esta tarea `css` dentro del gulpfile de producción completo
- [[Qué es Sass]] — qué es el preprocesador y por qué compilarlo
- [[Sintaxis de Sass]] — `@use`, variables, control de flujo (lo que dart-sass soporta)
- [[Separar Sass en Archivos]] — estructura de partials que alimenta el watch
- [[Mixins]] — patrones de Sass que el pipeline compila
- [[CSS Introducción]] — el CSS compilado como destino final
- [[ReactJS - Vite]] — la alternativa cuando el proyecto es un bundler
- [[Performance Web]] — por qué minificar y sourcemaps en prod
