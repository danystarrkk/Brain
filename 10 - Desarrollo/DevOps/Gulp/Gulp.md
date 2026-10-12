---
tags:
  - desarrollo
  - devops
  - gulp
tipo: concepto
estado: terminado
---

# Gulp

**Gulp** es un **task runner** (ejecutor de tareas) para Node.js que automatiza el flujo de trabajo de desarrollo web: compilar Sass, transpilar/minificar JavaScript, optimizar imágenes, copiar assets, refrescar el navegador, etc. Su filosofía es **code over configuration**: en vez de declarar un JSON de configuración como Grunt o Webpack, escribís **funciones JavaScript** en un `gulpfile.js` y las componés como cualquier otro programa.

Gulp requiere [[Node JS]] y npm (o yarn/pnpm) porque ejecuta JavaScript en Node: el mismo runtime que usa tu frontend. No instala nada mágico — es una librería más de tu `package.json`.

> [!info] Por qué existe Gulp
> Antes de Gulp (2013) había dos modelos: **Grunt** (2012) — declarativo, cada plugin lee y escribe archivos **temporales en disco**, lo que encadenar 5 pasos = 5 escrituras extra y builds lentos — y **npm scripts** crudos (`sass x.scss && postcss ... && uglify ...`), sin abstracción ni manejo de errores. Gulp tomó la *filosofía Unix* de Node: cada plugin es un programa chico que transforma un **stream**, y se ensamblan con `.pipe()`. Resultado: los archivos viven **en memoria** de principio a fin y los builds son 3-5× más rápidos que Grunt en pipelines multi-paso. Esa idea de stream **es** Gulp — todo lo demás es API de andamiaje.

## Modelo mental: streams de Vinyl

El dato que circula por los pipes no es "un archivo" sino un objeto **Vinyl** (`{ path, base, contents }`) donde `contents` es un stream o buffer de Node. El pipeline tiene tres tipos de piezas:

```
src/scss/app.scss
       │
       ▼
┌─────────────┐   emite Vinyl   ┌──────────┐   transforma   ┌───────────┐
│  gulp.src() │ ──────────────► │  sass()  │ ─────────────► │ postcss() │──┐
└─────────────┘   (lectura)     └──────────┘   scss → css   └───────────┘  │
                                                                           ▼
                                        ┌──────────┐  transforma   ┌──────────────┐
                                        │ dest()   │ ◄──────────── │ cssnano()    │
                                        └──────────┘   css → css   └──────────────┘
                                              │
                                              ▼
                                     dist/css/app.css  (escritura en disco)
```

- **`src()`** lee archivos y los convierte en un stream de Vinyl.
- **Cada plugin** consume el stream, transforma cada Vinyl y lo emite al siguiente (`transform stream` de Node, estándar de la plataforma — no una invención de Gulp).
- **`dest()`** es el *sink*: toma el stream final y **escribe** los archivos en disco.

> [!tip] Por qué los streams ganan
> Sin streams, cada plugin tendría que escribir su resultado a un archivo temporal, leerlo en el siguiente paso y borrarlo (así funciona Grunt). Con streams, el CSS compilado **nunca toca el disco** hasta que llega a `dest()`: menos I/O, menos errores intermedios y pipelines que crecen sin coste extra por paso. Es el mismo motivo por el que `cat a | grep x | wc -l` es el corazón de Unix.

Detalle práctico: los streams de Node son **buffers por defecto** (todo el archivo en memoria). Para archivos gigantes existe el modo streaming; en la práctica el modo buffer llega cómodo hasta archivos de decenas de MB.

## Anatomía de un gulpfile mínimo

```js
import { src, dest } from 'gulp';

export function css() {
  return src('src/scss/app.scss')      // 1. leo el archivo
    .pipe(sass())                       // 2. compilo (plugin)
    .pipe(dest('dist/css'));             // 3. escribo el resultado
}
```

Regla de oro del async: **toda tarea debe señalar cuándo termina**, devolviendo el stream/promise **o** llamando al callback `done()` — nunca las dos, nunca ninguna:

```js
// ✅ correcto: devuelve el stream (Gulp sabe cuándo termina)
export function copiar() {
  return src('src/assets/**/*').pipe(dest('dist/assets'));
}

// ✅ correcto: tarea sin stream, usa callback
export function limpiar(done) {
  console.log('listo');
  done();
}

// ❌ da el error clásico: "Did you forget to signal async completion?"
export function rota() {
  src('x.txt').pipe(dest('y'));   // sin return: Gulp no espera la escritura
}
```

Ver [[Instalar Gulp]] para montar el proyecto de cero y [[GulpFile Completo]] para un gulpfile de producción completo y comentado.

## Gulp 4 y 5: composición de tareas con `series` / `parallel`

Gulp 4 reemplazó por completo el modelo de dependencias de Gulp 3. Las tareas ya **no** declaran sus dependencias: se **componen** desde afuera como funciones puras:

```js
import { series, parallel, watch } from 'gulp';

const build = series(limpiar, parallel(css, js, imagenes));  // limpia, luego todo en paralelo
const dev   = series(build, observar);                        // build y luego watch (no termina nunca)

export { build, dev as default };
```

| Aspecto | Gulp 3 (2014) | Gulp 4/5 (actual) |
|---|---|---|
| Depender de otra tarea | `gulp.task('build', ['css', 'js'])` (array) | `series(css, js)` / `parallel(css, js)` |
| Orden garantizado | El orden del array **no** se garantizaba | `series` ejecuta en orden estricto |
| Ejecutar en paralelo | No (todo era "cuando pueda") | `parallel(t1, t2)` con cierre real |
| Arrancar desde CLI | `gulp.task('default', ...)` | `exports` de funciones (o `gulp.task` legado) |
| Secuenciador externo | Se necesitaba `run-sequence` | Innecesario, obsoleto |
| API de stream | `gulp.*` globales | Se importan: `import { src, dest } from 'gulp'` |

El grafo de tareas de un proyecto típico:

```
default/dev ──► build ──► limpiar
                    ├─► parallel: css, js, html, imagenes, fuentes ──► series: watch
build ──► parallel(limpiar? NO) ...
```

> [!warning] `parallel` no respeta dependencias
> `parallel(limpiar, css)` puede borrar `dist/` **mientras** `css` está escribiendo ahí dentro. La composición correcta siempre es `series(limpiar, parallel(css, js, ...))`: limpiar primero, todo lo demás en paralelo después.

## API core en una tabla

| API | Qué hace | Detalle clave |
|---|---|---|
| `src(globs, opts)` | Lee archivos → stream de Vinyl | Opciones: `base`, `sourcemaps`, `allowEmpty`, `since`, y en Gulp 5 `encoding` |
| `dest(carpeta, opts)` | Escribe el stream en disco | Conserva la ruta relativa a `base`; acepta `sourcemaps: '.'` |
| `pipe(transform)` | Encadena un plugin | Cualquier transform stream de Node sirve — no solo plugins `gulp-*` |
| `series(...tareas)` | Ejecuta en orden, una tras otra | Para pasos con dependencia (clean → build) |
| `parallel(...tareas)` | Ejecuta a la vez, cierra cuando todas cierran | Para pasos independientes (css ∥ js ∥ imgs) |
| `watch(globs, tarea)` | Reejecuta tarea ante cambios | Acepta glob, array de globs y función inline |
| `lastRun(tarea)` | Timestamp de la última ejecución | Base para builds incrementales: `src(globs, { since: lastRun(css) })` |

> [!warning] Gulp 5 y archivos binarios (cambio que rompe proyectos viejos)
> Gulp 5 (2023) pasa **utf8 por defecto** a todo lo que cruza los streams. Imágenes, fuentes o zip **se corrompen** al escribirse. Solución: declarar `src('src/img/**/*', { encoding: false })` en cada tarea con binarios. Los gulpfiles escritos para Gulp 4 necesitan este ajuste al migrar. Ver [[GulpFile Completo]] y [[Performance Web]] para ejemplos con `encoding: false`.

## Ecosistema de plugins

La convención es `gulp-<herramienta>` y hay ~2000 plugins publicados. Los esenciales:

| Plugin | Función | Notas 2026 |
|---|---|---|
| `gulp-sass` | Compila Sass (requiere pasar el compilador dart-sass) | Ver [[Compilar Sass]] |
| `gulp-postcss` + `autoprefixer` | Prefijos de navegador vía Browserslist | Se configura en `package.json` |
| `gulp-terser` | Minifica JS (sucesor de uglify, soporta ES2015+) | `gulp-uglify` quedó obsoleto para ES moderno |
| `gulp-concat` | Une archivos en uno solo | Útil sin bundler |
| `gulp-clean-css` | Minifica CSS (port de cssnano) | Alternativa: dejar que PostCSS haga cssnano |
| `gulp-rename` | Cambia nombre/extensión del Vinyl | `.min.css`, sufijos de versión |
| `gulp-rev` | Hash de contenido en nombres (`app.abc123.css`) | Para cache-busting, ver [[Performance Web]] |
| `gulp-imagemin` | Minifica imágenes (jpeg/png/gif/svg) | Mantenimiento laxo; ver tabla siguiente |
| `gulp-webp` / `gulp-avif` | Convierte imágenes a formatos modernos | Ver [[Formatos Avif y Webp]] |
| `del` | Borra archivos (promesa, no plugin de stream) | Se usa con `await deleteAsync(...)` |
| `gulp-plumber` | Manejo de errores sin romper el watch | Alternativa manual: `.on('error', ...)` |
| `browser-sync` | Live-reload + proxy del servidor | Se engancha como tarea o en `watch` |

> [!tip] Los plugins `gulp-*` son cada vez menos necesarios
> Muchos plugins no hacen más que envolver una librería Node en un stream. En Gulp 4/5 podés importar la librería directamente y llamarla dentro de una tarea `async` — el patrón ya era común: la tarea `imagenes` de [[GulpFile Completo]] usa **sharp** (librería pura, sin plugin) iterando archivos con `glob`. Regla: si existe un plugin `gulp-*` actualizado, usalo; si está abandonado (es el caso de buena parte del ecosistema pre-2020), envolvé la librería moderna vos mismo. **No** instales `run-sequence`, `gulp-util` (deprecados) ni plugins que dependan de `node-gyp` viejo.

## Gulp en 2026: cuándo tiene sentido

Tabla de decisión honesta — el bundler ganó la guerra de los SPAs, pero Gulp sigue siendo la mejor herramienta para **pipelines de assets que no son "empaquetar módulos"**:

| Situación | Herramienta | Por qué |
|---|---|---|
| SPA con React/Vue/Svelte | **[[ReactJS - Vite]]** (o esbuild) | Dev server, HMR, tree shaking, code splitting — Gulp no hace nada de esto |
| Solo transpilar/bundle JS moderno | **esbuild** / swc / tsc | Velocidad brutal, cero config |
| 2-3 pasos simples | **npm scripts** (`&&`, `npx`) | Sin mantenimiento extra, suficiente |
| Sitio estático / tema WordPress / email HTML / landing | **Gulp** | Sass + prefix + minify + imágenes + copia de fuentes + rev, sin bundler |
| Pipeline de assets custom multi-formato | **Gulp** | Imágenes a WebP/AVIF + responsive + hash + copia — orquestación de pasos heterogéneos |
| Proyecto legacy Gulp 3/4 | **Gulp** (migrar a 4/5) | Reescribir en Vite solo por el pipeline no rinde |
| CSS-first con utilidades | [[TailwindCSS - Instalación y Configuración]] | Tailwind trae su propio CLI/build |

La posición honesta: **Gulp ya no es la herramienta por defecto de un proyecto nuevo** — para eso está Vite. Pero "task runner" y "bundler" resuelven problemas distintos: Vite empaqueta **módulos JS**; Gulp orquesta **transformaciones de archivos**. Cuando el trabajo es *archivos* (Sass multipunto, imágenes, copias, hash de cache), Gulp sigue siendo la opción más directa, y es la razón por la que sobrevive en pipelines de WordPress, generadores estáticos y proyectos sin framework.

> [!info] Descargas ≠ adopción en proyectos nuevos
> Gulp sigue registrando cientos de millones de descargas mensuales en npm, pero buena parte son instalaciones transitivas de proyectos viejos. La señal real: el ecosistema de plugins casi no recibe innovación desde ~2019 (la innovación vive en Vite/esbuild), mientras que el núcleo se mantiene estable (Gulp 5 salió en 2023). Usá Gulp sabiendo que es una herramienta **madura y estable**, no en crecimiento.

## Recetas del día a día

### Limpiar con `del`

```js
import { deleteAsync } from 'del';   // del v7+: ESM con export nombrado

export function limpiar() {
  return deleteAsync(['dist', '!dist/keep.txt']);   // admite negación
}
```

`del` no es un plugin de stream sino una promesa: se usa con `await` dentro de tareas async o devolviendo la promesa directamente (como arriba). Nunca va dentro de un `.pipe()`.

### Copiar assets estáticos incrementalmente

```js
import { src, dest, lastRun } from 'gulp';

export function assets() {
  return src('src/assets/**/*', { since: lastRun(assets) })
    .pipe(dest('dist/assets'));      // en watch solo copia los tocados
}
```

### Ejecutar condicionalmente: `gulp build --prod`

```js
const esProd = process.argv.includes('--prod');   // gulp build --prod
// o por variable: NODE_ENV=production gulp build (recomendado, estándar del ecosistema)
```

### Live-reload con browser-sync

```js
import browserSync from 'browser-sync';
const server = browserSync.create();

export function servir() {
  server.init({ server: 'dist' });
}

function recargar(done) {
  server.reload();
  done();
}

export function observar() {
  watch('src/scss/**/*.scss', series(css, recargar));   // compilar y luego recargar
  watch('src/**/*.html', series(html, recargar));
}

export default series(build, parallel(servir, observar));
```

El patrón `series(tarea, recargar)` es la clave: recargar **después** de que el stream terminó, nunca en paralelo (o el navegador pide los archivos viejos).

## Errores comunes

| Sintoma | Causa | Arreglo |
|---|---|---|
| `Did you forget to signal async completion?` | La tarea no devuelve stream/promise ni llama `done()` | `return src(...).pipe(...)` o `done()` |
| `Local gulp not found` / `gulp: command not found` | Falta `gulp-cli` global o `gulp` local | Ver [[Instalar Gulp]] |
| El watch no dispara | Glob mal escapado (shell/Windows) o tarea sin `return` | Usar comillas simples en el glob y devolver el stream |
| Imágenes corruptas tras el build | Gulp 5 + binarios sin `encoding: false` | `src(..., { encoding: false })` |
| Tareas se pisan entre sí | `parallel` con tareas que escriben en la misma carpeta | Reordenar con `series` |
| `gulp.task` no define nada en el array de deps | Sintaxis Gulp 3 en Gulp 4/5 | Migrar a `series`/`parallel` |

Errores específicos de cada pipeline: [[Compilar Sass]] (Sass) y [[GulpFile Completo]] (streams y composición).

## Cheatsheet

| Concepto | Código |
|---|---|
| Leer → transformar → escribir | `src(g).pipe(p()).pipe(dest(d))` |
| Orden estricto | `series(tareaA, tareaB)` |
| A la vez | `parallel(tareaA, tareaB)` |
| Observar cambios | `watch('src/**/*.scss', css)` |
| Incremental | `src(g, { since: lastRun(css) })` |
| Binarios (Gulp 5) | `src(g, { encoding: false })` |
| Sourcemaps (nativo) | `src(g, { sourcemaps: true })` → `dest(d, { sourcemaps: '.' })` |
| Exponer tarea | `export function css() {...}` |
| Tarea por defecto | `export default build;` |
| Ejecutar | `npx gulp css` / `npx gulp` (default) |

## Relacionadas

- [[Instalar Gulp]] — instalación, gulp-cli vs gulp local, errores de setup
- [[GulpFile Completo]] — gulpfile de producción anotado task por task
- [[Compilar Sass]] — el pipeline de Sass dentro de Gulp en profundidad
- [[Performance Web]] — qué tareas de optimización viven dentro del pipeline (minify, imágenes, hash)
- [[Node JS]] — el runtime sobre el que corre todo
- [[ReactJS - Vite]] — cuándo un bundler moderno reemplaza a Gulp
- [[TailwindCSS - Instalación y Configuración]] — framework CSS con su propio build
- [[Formatos Avif y Webp]] — formatos de imagen modernos generados por el pipeline
- [[Comandos Git]] — flujo de trabajo alrededor del código del proyecto
