---
aliases: ["GulpFile Completo", "gulpfile.js", "Gulpfile de Producción"]
tags:
  - devops/gulp
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Gulp]]"
  - "[[Instalar Gulp]]"
  - "[[Compilar Sass]]"
  - "[[Performance Web]]"
  - "[[Formatos Avif y Webp]]"
  - "[[Qué es Sass]]"
fuente: "https://gulpjs.com/docs/en/getting-started/quick-start"
---

# GulpFile Completo

El `gulpfile.js` **es** el proyecto de automatización: un módulo [[Node JS]] que exporta funciones-tarea y las compone con `series`/`parallel`. Esta nota es un recorrido comentado de un gulpfile de producción real — Sass, JS, imágenes multi-formato con sharp, limpieza, watch y modos dev/producción — seguido de la teoría (globs, error handling, sourcemaps, incremental) que explica cada decisión.

## El gulpfile completo

Requiere: `npm i -D gulp gulp-sass sass gulp-postcss autoprefixer cssnano gulp-terser gulp-concat del glob` y `npm i sharp`. Formato ESM (`"type": "module"` en `package.json`, ver [[Instalar Gulp]]).

```js
// gulpfile.js — ESM
import path from 'node:path';
import fs from 'node:fs';

import { src, dest, watch, series, parallel, lastRun } from 'gulp';
import { deleteAsync } from 'del';
import * as dartSass from 'sass';
import gulpSass from 'gulp-sass';
import postcss from 'gulp-postcss';
import autoprefixer from 'autoprefixer';
import cssnano from 'cssnano';
import concat from 'gulp-concat';
import terser from 'gulp-terser';
import sharp from 'sharp';
import { glob } from 'glob';

const sass = gulpSass(dartSass);          // gulp-sass necesita el compilador explícito
const esProd = process.env.NODE_ENV === 'production';

// ── Rutas centralizadas: cambiar el layout es tocar un objeto, no 10 strings ──
const paths = {
  scss:  { entry: 'src/scss/app.scss', watch: 'src/scss/**/*.scss', dest: 'dist/css' },
  js:    { src: 'src/js/**/*.js', dest: 'dist/js' },
  html:  { src: 'src/**/*.html', dest: 'dist' },
  img:   { src: 'src/img/**/*.{jpg,jpeg,png}', dest: 'dist/img' },
  thumb: { src: 'src/img/gallery/full', dest: 'src/img/gallery/thumb' },
  fonts: { src: 'src/fonts/**/*', dest: 'dist/fonts' },
};

// ── 1. Limpiar: SIEMPRE primero, nunca en paralelo con quien escribe en dist ──
export function limpiar() {
  return deleteAsync(['dist']);
}

// ── 2. Estilos: sass → postcss(autoprefixer [+ cssnano en prod]) ──
export function css() {
  const plugins = esProd ? [autoprefixer(), cssnano()] : [autoprefixer()];
  return src(paths.scss.entry, { sourcemaps: !esProd })
    .pipe(sass().on('error', sass.logError))   // error controlado: no mata el watch
    .pipe(postcss(plugins))
    .pipe(dest(paths.scss.dest, { sourcemaps: esProd ? false : '.' }));
}

// ── 3. Scripts: concat + terser condicional ──
export function js() {
  let stream = src(paths.js.src, { sourcemaps: !esProd }).pipe(concat('app.js'));
  if (esProd) stream = stream.pipe(terser());
  return stream.pipe(dest(paths.js.dest, { sourcemaps: esProd ? false : '.' }));
}

// ── 4. HTML: copia incremental (solo archivos tocados desde la última corrida) ──
export function html() {
  return src(paths.html.src, { since: lastRun(html) }).pipe(dest(paths.html.dest));
}

// ── 5. Fuentes binarias: encoding:false obligatorio en Gulp 5 ──
export function fuentes() {
  return src(paths.fonts.src, { encoding: false }).pipe(dest(paths.fonts.dest));
}

// ── 6. Imágenes: sharp → original recomprimido + webp + avif ──
export async function imagenes() {
  const files = await glob(paths.img.src);
  const tareas = files.map(async (file) => {
    const rel    = path.relative('src/img', file);          // conserva subcarpetas
    const outDir = path.join(paths.img.dest, path.dirname(rel));
    fs.mkdirSync(outDir, { recursive: true });
    const base = path.basename(file, path.extname(file));
    const opts = { quality: 80 };
    const esPng = path.extname(file) === '.png';

    await Promise.all([
      (esPng ? sharp(file).png(opts) : sharp(file).jpeg({ ...opts, progressive: true }))
        .toFile(path.join(outDir, path.basename(file))),   // original recomprimido
      sharp(file).webp(opts).toFile(path.join(outDir, `${base}.webp`)),
      sharp(file).avif({ quality: 60 }).toFile(path.join(outDir, `${base}.avif`)),
    ]);
  });
  await Promise.all(tareas);   // CRÍTICO: esperar todos los writes antes de cerrar
}

// ── 7. Thumbnails de galería: recorte centralizado 250×180 ──
export async function crop(done) {
  const width = 250, height = 180;
  fs.mkdirSync(paths.thumb.dest, { recursive: true });
  const fotos = (await glob(`${paths.thumb.src}/*.jpg`));     // solo jpg de la galería
  await Promise.all(fotos.map(async (file) => {
    const salida = path.join(paths.thumb.dest, path.basename(file));
    await sharp(file)
      .resize(width, height, { position: 'centre' })          // recorte centrado
      .jpeg({ quality: 70 })
      .toFile(salida);
  }));
  done();
}

// ── 8. Watch: una tarea por glob; gulp espera el stream devuelto ──
export function observar() {
  watch(paths.scss.watch, css);
  watch(paths.js.src, js);
  watch(paths.html.src, html);
  watch(paths.img.src, imagenes);
}

// ── 9. Composición del grafo de tareas ──
const build = series(limpiar, parallel(css, js, html, imagenes, fuentes));
export { build };
export const dev = series(build, observar);   // build y después watch (no termina)
export default dev;
```

```bash
gulp build            # producción (sin sourcemaps, minificado)
NODE_ENV=production gulp build
npm run dev           # vía scripts: build + watch
```

> [!info] El grafo de tareas en un diagrama
> ```
> default/dev ─► series ─┬─► limpiar ─┐
>                        └─► observar ◄┘ (tras el build)
> build ─► series ─► limpiar ─► parallel ─┬─► css   (sass→postcss→dist/css)
>                                         ├─► js    (concat→terser?→dist/js)
>                                         ├─► html  (copy incremental)
>                                         ├─► imgs  (sharp→dist/img + webp/avif)
>                                         └─► fonts (copy binario)
> ```
> `limpiar` va en `series` antes del `parallel` porque borrar `dist/` mientras otros escriben dentro es una condición de carrera. Todo lo demás corre en paralelo porque no se pisan.

## Decisiones explicadas

### `series` vs `parallel` en el build real

| Paso | Composición | Por qué |
|---|---|---|
| `limpiar` | Primero, solo | Borra `dist/`; si corre en paralelo con `css`, puede borrar el CSS recién escrito |
| `css`, `js`, `html`, `imagenes`, `fuentes` | `parallel` | Lean `src/`, escriben destinos distintos: independientes entre sí |
| `observar` | Último, en `series(build, observar)` | El watch nunca "termina" — si estuviera en paralelo con el build, arrancaría antes de tener los assets listos |

### Modos dev vs producción (`NODE_ENV` branching)

| | Dev (`gulp dev`) | Prod (`NODE_ENV=production gulp build`) |
|---|---|---|
| CSS | autoprefixer solo, expandido | + cssnano (minificado), comprimido |
| JS | concat sin terser (errores legibles) | terser (minificado) |
| Sourcemaps | `.map` al lado (`sourcemaps: '.'`) | desactivados |
| HTML | copia incremental (`since: lastRun`) | copia completa |
| Watch | activo | no |

La variable `NODE_ENV` es la misma que usa el ecosistema [[Node JS]]/React — un solo switch para todo el proyecto.

### Glob syntax en profundidad

| Glob | Matchea |
|---|---|
| `src/js/*.js` | Solo `src/js/` directo (sin subcarpetas) |
| `src/js/**/*.js` | Todo árbol de `src/js/` — el estándar |
| `src/img/**/*.{jpg,png}` | Brace expansion: dos extensiones |
| `src/**/*` | Todo bajo `src/` |
| `['dist/**/*', '!dist/legacy/**']` | Negación: todo salvo `dist/legacy/` |
| `'src/scss/**/*.scss'` | El watch típico: partials incluidos |

Regla del **base**: `src('src/js/**/*.js')` calcula `base = 'src/js/'` (el prefijo sin comodines) y `dest()` recrea solo la parte `**`. Por eso `dest('dist')` con `'src/**/*.html'` produce `dist/pages/...` si el HTML vive en `src/pages/`. Si querés otra estructura, forzá `base`: `src(glob, { base: 'src' })`. Cambiar el layout de salida = cambiar `base`, no renombrar archivos después.

> [!tip] `since: lastRun(tarea)` = builds incrementales gratis
> `lastRun` guarda el timestamp de la última ejecución exitosa de la tarea y `src(g, { since: lastRun(html) })` emite **solo archivos modificados desde entonces**. Ideal para watch: copiar 3 HTML tocados en vez de 200. Funciona bien en tareas de **copia/transformación por archivo** (html, fonts, js); en `css`/`imagenes` (entry points y orquestación propia con sharp) no aplica igual.

### Error handling en streams: por qué el watch se muere

Cuando un plugin lanza un error y nadie lo maneja, el stream se destruye, Gulp aborta **y el proceso sale** — el watch muere con él. Tres mecanismos, del más específico al más genérico:

```js
// 1. Manejo del propio plugin (Sass): loguea y cierra el stream limpio
.pipe(sass().on('error', sass.logError))

// 2. Patrón emitter clásico (genérico, pre-gulp-plumber)
.pipe(myPlugin().on('error', function () { this.emit('end'); }))

// 3. gulp-plumber: intercepta errores de TODOS los pipes del task
.pipe(plumber({ errorHandler: function (err) {
  console.error(err.message);
  this.emit('end');
}}))
```

| Mecanismo | Alcance | Cuándo |
|---|---|---|
| `sass.logError` | Solo errores de compilación Sass | Siempre en tasks de styles (ver [[Compilar Sass]]) |
| `.on('error', emit('end'))` | El pipe donde se engancha | Plugins sin soporte propio |
| `gulp-plumber` | Todos los pipes del task | Pipelines largos; evita repetir `.on` por paso |

> [!warning] Manejar el error ≠ ignorarlo
> `emit('end')` silencioso deja el watch vivo **pero** el build de producción debe **fallar**: si el CSS no compila, el deploy no puede seguir. Patrón estándar: handler que loguea y termina el stream en dev, y en prod dejar que el error se propague (o chequear el código de salida en CI).

### Flujo de sourcemaps

```js
src(entry, { sourcemaps: true })          // genera mapas en memoria
  .pipe(sass())                           // los plugins compatibles los actualizan
  .pipe(dest(out, { sourcemaps: '.' }));  // escribe app.css + app.css.map
```

Sourcemaps es **nativo de Gulp 4+** — `gulp-sourcemaps` ya no hace falta. `sourcemaps: true` (en memoria) + `sourcemaps: '.'` (disco, `.map` al lado) o `sourcemaps: '/maps'` (carpeta aparte). En producción se desactivan por tamaño de despliegue (o se suben aparte).

## Imágenes: dos estrategias comparadas

El gulpfile usa **sharp** directamente (tareas `imagenes` y `crop`). Alternativa histórica: **gulp-imagemin** como plugin de stream:

| | sharp (librería en tarea async) | gulp-imagemin (plugin de stream) |
|---|---|---|
| Velocidad | Muy rápida (libvips, multi-thread) | Depende del binario subyacente |
| WebP/AVIF | Nativo, un solo proceso | Requiere `gulp-webp` / `gulp-avif` aparte |
| Mantenimiento | Activo | Laxo (últimas releases esporádicas) |
| API | Imperativa, control total (condicionales, subcarpetas) | Declarativa, un pipe |
| Gulp 5 | Tarea `async` sin tema de streams | Hay que pasar `{ encoding: false }` en `src` |

Receta imagemin para comparar (verbatim del pipeline clásico):

```js
import imagemin, { gifsicle, mozjpeg, optipng, svgo } from 'gulp-imagemin';

export function imageMin() {
  return src('./src/img/**/*', { encoding: false })   // binarios en Gulp 5
    .pipe(imagemin([
      mozjpeg({ quality: 80, progressive: true }),
      optipng({ optimizationLevel: 5 }),
      gifsicle({ interlaced: true, optimizationLevel: 2 }),
      svgo({ plugins: [{ name: 'removeViewBox', active: false }] }),
    ], { verbose: true }))
    .pipe(dest('./dist/img'));
}
```

La conversión a formatos modernos también puede ser plugin: `gulp-webp` / `gulp-avif` (ver [[Formatos Avif y Webp]] y [[Performance Web]]). Ver [[Gulp]] para cuándo un plugin `gulp-*` sigue valiendo la pena.

## Errores comunes del gulpfile

| Sintoma | Causa y arreglo |
|---|---|
| `Did you forget to signal async completion?` | Tarea sin `return` de stream y sin `done()` — añadir uno de los dos |
| El watch muere al primer error de Sass | Falta `.on('error', sass.logError)` o plumber (ver arriba) |
| `Task never defined: xyz` | El nombre exportado no coincide; revisar `gulp --tasks` |
| Salida en `dist/dist/...` o estructura inesperada | `base` calculado del glob: forzar `base` en `src()` |
| Imágenes corruptas (Gulp 5) | Falta `{ encoding: false }` en `src()` de binarios |
| `dist` se vacía a medias | `limpiar` dentro de un `parallel` — moverlo a `series(limpiar, ...)` |
| El build termina antes de escribir imágenes (sharp) | `done()` sin `await` — el código original cerraba la tarea antes de que sharp terminara; arreglado con `await Promise.all` en esta versión |
| Watch de Sass no detecta cambios en partials | Glob del watch solo apuntaba al entry; usar `src/scss/**/*.scss` |
| Errores raros con plugins viejos | Plugins CommonJS con proyectos ESM — ver [[Instalar Gulp]] |

> [!warning] Correcciones técnicas respecto de versiones anteriores de este gulpfile
> 1. **`done()` se llamaba sin esperar a sharp** → carrera: Gulp marcaba la tarea como terminada y el proceso podía salir con las imágenes a medio escribir. Ahora: `await Promise.all([...])`.
> 2. **Regex de crop** `/\\.(jpg)$/` no matcheaba `.jpg` (doble barra escapada); corregido a glob `'*.jpg'`.
> 3. **Glob de watch sobre `src/img/app.js`** puntuales → ahora `src/js/**/*.js` (subcarpetas incluidas).
> 4. **`sharp` lanzaba fuera del try real**: `forEach` no espera promesas, el `try/catch` no capturaba nada; ahora `map` + `Promise.all` con `await`.

## Cheatsheet del gulpfile

| Elemento | Código |
|---|---|
| Compilador Sass | `const sass = gulpSass(dartSass)` |
| Switch dev/prod | `process.env.NODE_ENV === 'production'` |
| Limpiar | `await deleteAsync(['dist'])` |
| Incremental | `{ since: lastRun(tarea) }` |
| Binarios (Gulp 5) | `{ encoding: false }` |
| Concat + minify | `.pipe(concat('app.js')).pipe(terser())` |
| Errores Sass | `.on('error', sass.logError)` |
| Build final | `series(limpiar, parallel(...))` |
| Watch final | `series(build, observar)` |
| Rutas | Objeto `paths` centralizado arriba del todo |

## Relacionadas

- [[Gulp]] — modelo de streams y composición de tareas
- [[Instalar Gulp]] — ESM, CLI y dependencias de este gulpfile
- [[Compilar Sass]] — el task `css` explicado en profundidad
- [[Performance Web]] — terser, imagemin, formatos modernos y hashing como tareas
- [[Formatos Avif y Webp]] — qué generan las tareas `imagenes`/`webp`/`avif`
- [[Qué es Sass]] — el origen de `src/scss/app.scss`
- [[Node JS]] — ESM, `package.json` y `NODE_ENV`
