---
aliases: ["Performance Web", "Rendimiento Web", "Optimización Web", "Web Performance"]
tags:
  - devops/gulp
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Gulp]]"
  - "[[GulpFile Completo]]"
  - "[[Compilar Sass]]"
  - "[[Lazy Loading]]"
  - "[[Preload]]"
  - "[[Prefetch]]"
  - "[[Formatos Avif y Webp]]"
  - "[[ReactJS - Vite]]"
  - "[[Html]]"
  - "[[CSS Introducción]]"
fuente: "https://web.dev/articles/vitals"
---

# Performance Web

**Performance web** es la velocidad y eficiencia con que una página se descarga, se renderiza y responde a la interacción del usuario. No es un "extra": es parte del producto — afecta conversión, retención, SEO (Core Web Vitals son factor de ranking) y accesibilidad de datos móviles. Esta nota cubre las métricas que lo miden (Core Web Vitals), cada técnica de optimización con su porqué, y **cómo se implementa cada una como tarea de Gulp** dentro del pipeline de [[Gulp]] / [[GulpFile Completo]].

## Core Web Vitals: las tres métricas que puntúan Google

| Vital | Qué mide | Bueno | Malo | Causas típicas | Arreglos principales |
|---|---|---|---|---|---|
| **LCP** (*Largest Contentful Paint*) | Cuándo aparece el elemento más grande (hero, título, imagen grande) | ≤ 2.5 s | > 4 s | Imagen LCP sin prioridad, CSS/JS bloqueantes, TTFB lento, render server lento | `fetchpriority="high"` + preload del LCP, [[Preload]] de CSS crítico, imágenes ligeras (AVIF/WebP), caching |
| **INP** (*Interaction to Next Paint*) | Latencia de **todas** las interacciones (reemplazó a FID en 2024) | ≤ 200 ms | > 500 ms | JS en el hilo principal, handlers pesados, layout thrashing, bundles grandes sin code splitting | Dividir JS (code splitting), `requestIdleCallback`, defer de scripts, [[ReactJS - Vite]] splitting |
| **CLS** (*Cumulative Layout Shift*) | Estabilidad visual: cuánto se mueven los elementos al cargar | ≤ 0.1 | > 0.25 | Imágenes/ads/iframes sin `width`/`height`, fuentes que reemplazan texto (FOUT/FOIT), contenido inyectado arriba | Dimensiones explícitas en [[Html]] `<img>`, `font-display: swap`, reservar espacio |

> [!info] INP reemplazó a FID (marzo 2024)
> FID solo medía el **primer** clic; INP mide **cada** interacción a lo largo de la sesión. Un botón que responde bien la primera vez pero se traba la décima ahora pierde puntos. Cualquier material viejo que hable de FID está desactualizado.

### Métricas de soporte

| Métrica | Defecto | Nota |
|---|---|---|
| **TTFB** (*Time to First Byte*) | ≤ 800 ms | Red + servidor; lo mejora el CDN/caching, no el build front |
| **FCP** (*First Contentful Paint*) | ≤ 1.8 s | Primer pintado de cualquier contenido — ancestro del LCP |
| **TBT** (*Total Blocking Time*) | — | Suma de bloqueos del hilo principal por JS — proxy del INP en lab |
| **Speed Index** | — | Velocidad de "llenado" visual de la página |

*Lab* (Lighthouse, controlado) vs *field* (datos reales de usuarios — Chrome UX Report, lo que puntúa Google): medí siempre las dos; una máquina de escritorio con fiber no representa a un usuario 4G.

## Camino crítico de render (*critical rendering path*)

El navegador encadena hasta poder pintar: **HTML → DOM**, **CSS → CSSOM**, juntos → árbol de render → pintado. JS **paraliza** el parser hasta que termina (a menos que lleve `defer`/`async`). De ahí las reglas derivadas:

| Recurso | Política | Por qué |
|---|---|---|
| CSS crítico | Inline o `preload`; el resto diferido | Todo el CSS es bloqueante por defecto: sin él no se pinta nada |
| JS de arranque | `defer` (respeta orden, post-parse) | Ejecutar antes no ayuda a pintar; solo retrasa |
| JS de terceros | `async` o cargado bajo demanda | Sin relación de orden, no debe bloquear |
| Fuentes | `font-display: swap` + preload selectivo | Evita el FOIT (texto invisible) y su CLS asociado |

Herramientas de carga diferida y priorización en profundidad: [[Preload]] (recursos que deben ir ya), [[Prefetch]] (siguiente navegación, bajo prioridad) y [[Lazy Loading]] (bajo demanda al entrar en viewport).

## Minificación y compresión

Dos capas distintas que suelen confundirse:

| Capa | Qué hace | Dónde | Herramienta |
|---|---|---|---|
| **Minificación** | Elimina espacios/comentarios/variables → menos bytes en el archivo | Build | `sass({ style: 'compressed' })`, `cssnano`, `gulp-terser` |
| **Compresión de transferencia** | Codifica el stream HTTP al vuelo | Servidor (o CDN) | gzip o brotli |

### gzip vs brotli

| | gzip | brotli |
|---|---|---|
| Compresión | ~15-25% mejor que gzip (brotli) | — |
| Navegadores | Todos | Todos modernos (2017+) |
| CPU en server | Baja | Más alta en compresión dinámica |
| Config | `gzip on;` en nginx | `brotli on;` (módulo) o precompresión estática |
| CDN | Automático (Netlify, Vercel, Cloudflare) | Automático en los mismos |

> [!tip] La combinación que conviene: minificar + brotli estático
> Brotli con `quality 11` precompresiendo los `.js/.css` del build y sirviendo los `.br` con `Content-Encoding: br` (o dejar que el CDN lo haga). La minificación reduce los *originales*; la compresión reduce lo que viaja. Las dos, siempre.

En [[Html]] el HTML se comprime igual (es texto — gzip/brotli dinámico del servidor cubre el 90% de los casos).

## Caching: la optimización más barata

| Header | Comportamiento | Uso |
|---|---|---|
| `Cache-Control: max-age=31536000, immutable` | 1 año, sin revalidar | Assets **con hash en el nombre** (`app.a1b2c3.css`) |
| `Cache-Control: max-age=3600` | 1 hora | Sin hash, actualización frecuente |
| `Cache-Control: no-cache` | Revalida siempre (ETag/304) | HTML shell, API |
| `ETag` / `Last-Modified` | 304 si no cambió | Cualquier recurso revalidable |

La estrategia canónica: **HTML sin cache + todo lo demás con hash y `immutable`**. El HTML (que referencia `app.a1b2c3.css`) se actualiza y arrastra los assets nuevos; los assets viejos quedan huérfanos y el cache nunca sirve stale. El hash lo pone el build:

```js
import rev from 'gulp-rev';

export function hashing() {
  return src('dist/**/*.{css,js,png,jpg}')
    .pipe(rev())                                    // app.css → app-a1b2c3d4.css
    .pipe(dest('dist'))
    .pipe(rev.manifest())                           // mapa: "app.css": "app-a1b2c3d4.css"
    .pipe(dest('dist'));
}
```

El manifest se usa para reescribir referencias en el HTML (`gulp-rev-replace` / `gulp-rev-all` lo automatizan). En [[ReactJS - Vite]] esto viene de serie (`[name]-[hash].js`).

## Optimización de imágenes

Las imágenes son el peso #1 en la mayoría de páginas. Pipeline completo:

| Formato | Mejor para | Compresión | Alpha | Nota |
|---|---|---|---|---|
| JPEG | Fotos | Buena, sin alpha | ❌ | Progressive |
| PNG | Alpha, capturas con texto nítido | Pobre | ✅ | Paleta PNG-8 reduce mucho |
| GIF | Animación simple | Mala | ✅ | Reemplazar por video/APNG |
| SVG | Iconos, logos, ilustraciones vectoriales | Excelente | ✅ | Minificar con SVGO |
| **WebP** | Fotos con alpha, reemplazo general | Mejor que JPEG | ✅ | Ver [[Formatos Avif y Webp]] |
| **AVIF** | Fotos — la mejor ratio calidad/peso | Superior a WebP | ✅ | Codificación más lenta |

### Pipeline Gulp de imágenes (absorbe el flujo completo)

Formatos modernos con plugins dedicados:

```bash
npm i -D gulp-webp gulp-avif
```

```js
import { src, dest } from 'gulp';
import webp from 'gulp-webp';
import avif from 'gulp-avif';

export function imageWebp() {
  return src('./src/img/**/*.{jpg,png}', { encoding: false })   // binarios: Gulp 5
    .pipe(webp({ quality: 50 }))
    .pipe(dest('./dist/img/'));
}

export function imageAvif() {
  return src('./src/img/**/*.{jpg,png}', { encoding: false })
    .pipe(avif({ quality: 50 }))
    .pipe(dest('./dist/img/'));
}
```

Minificación clásica con `gulp-imagemin` (los optimizadores ya vienen incluidos; no requiere instalarlos aparte en versiones actuales):

```bash
npm i -D gulp-imagemin
```

```js
import imagemin, { gifsicle, mozjpeg, optipng, svgo } from 'gulp-imagemin';

export function imageMin() {
  return src('./src/img/**/*', { encoding: false })
    .pipe(imagemin([
      mozjpeg({ quality: 80, progressive: true }),
      optipng({ optimizationLevel: 5 }),
      gifsicle({ interlaced: true, optimizationLevel: 2 }),
      svgo({ plugins: [{ name: 'removeViewBox', active: false }] }),
    ], { verbose: true }))
    .pipe(dest('./dist/img'));
}
```

Alternativa moderna: **sharp** directamente en tareas `async` (recetas de conversión y recorte en [[GulpFile Completo]]) — más rápida y con WebP/AVIF en un solo proceso. Ver [[Formatos Avif y Webp]] para el `<picture>` que sirve cada formato.

### HTML sin CLS y con lazy

```html
<img width="300" height="300" loading="lazy" src="img/foto.jpg" alt="Sobre festival">
```

- **`width`/`height`** siempre: el navegador reserva la caja → CLS = 0 aunque la imagen tarde.
- **`loading="lazy"`** en todo lo que esté bajo el fold. **Nunca** en el LCP (la imagen del hero con lazy **empeora** el LCP — es el error más común de esta técnica).
- Videos: `preload="none"` (o `metadata`) por defecto; `preload="auto"` solo si el video arranca solo.
- Imágenes insertadas con JS: pasar `width`/`height` también en el template JS.
- Detalle de lazy loading y comportamiento por navegador: [[Lazy Loading]].

### Responsive: `srcset` + `sizes`

```html
<img src="img/foto-800.jpg"
     srcset="img/foto-400.jpg 400w, img/foto-800.jpg 800w, img/foto-1200.jpg 1200w"
     sizes="(max-width: 600px) 100vw, 50vw"
     width="800" height="600" loading="lazy" alt="...">
```

El navegador elige la variante según ancho real del slot y densidad de pantalla — servir un JPEG de 2000px a un móvil es el mayor despilfarro típico. Generar las variantes: `sharp(...).resize(400)` en tarea Gulp (ver [[GulpFile Completo]]) o plugins `gulp-responsive`. Ver también [[Responsive Web Design]].

## Herramientas de auditoría

| Herramienta | Tipo | Para qué |
|---|---|---|
| **Lighthouse** (DevTools / CLI) | Lab | Score 0-100 + diagnóstico accionable por métrica |
| **PageSpeed Insights** | Lab + **field** (CrUX) | Lo que puntúa Google de verdad |
| **WebPageTest** | Lab | Waterfall real multi-ubicación/red; matiza el "por qué" |
| **Chrome DevTools → Performance** | Lab interactivo | Grabar interacciones, ver tareas largas (INP), layout shifts |
| **DevTools → Network** | Lab | Waterfall, tamaño transferido, caché |
| **`npm i -D lighthouse-ci`** o budgets JSON | CI | Fallar el build si se exceden los presupuestos |

### Presupuestos (*performance budgets*)

Definí topes duros y hacé fallar el CI al pasarlos (Lighthouse budgets acepta bytes y tiempos):

```json
[
  {
    "path": "/*",
    "resourceSizes": [
      { "resourceType": "script", "budget": 150 },
      { "resourceType": "css", "budget": 60 },
      { "resourceType": "image", "budget": 300 },
      { "resourceType": "total", "budget": 500 }
    ],
    "timings": [{ "metric": "first-contentful-paint", "budget": 1800 }]
  }
]
```

## Optimizaciones en tiempo de build

| Técnica | Qué hace | Herramienta |
|---|---|---|
| **Tree shaking** | Elimina código muerto de módulos ESM | Bundlers: [[ReactJS - Vite]] / esbuild (Gulp **no** lo hace: no entiende grafos de módulos) |
| **Code splitting** | Divide el JS en chunks cargados bajo demanda | Vite (`import()` dinámico) |
| **Minify** | Comprime sintaxis | `gulp-terser` (JS), cssnano / sass compressed (CSS) |
| **Concat** | Une archivos, reduce requests | `gulp-concat` — útil solo sin bundler HTTP/2 |
| **Hashing** | Cache-busting | `gulp-rev` |

> [!info] Dónde entra Gulp en un pipeline de optimización
> Gulp orquesta: **minify** (sass compressed + `gulp-terser`), **imágenes** (`sharp`/`gulp-imagemin`/`gulp-webp`/`gulp-avif`), **hashing** (`gulp-rev`), **copia de assets**, **fuentes subconjunto** (`gulp-subset-font`). Lo que **no** aporta: tree shaking y code splitting — eso es del bundler. En un proyecto Vite, el rol de optimizador lo asume Vite; Gulp queda para las tareas no-JS (imágenes, rev de estáticos), o se abandona.

### Minificación con `gulp-terser` (JS)

```bash
npm i -D gulp-terser
```

```js
import { src, dest } from 'gulp';
import terser from 'gulp-terser';

export function js() {
  return src('src/js/app.js')
    .pipe(terser())                     // minifica; options: mangle, compress, format.comments
    .pipe(dest('dist/js'));
}
```

CSS minificado en el propio pipeline de Sass (`style: 'compressed'` o cssnano) — ver [[Compilar Sass]]. `gulp-uglify` (port de UglifyJS) quedó obsoleto para ES moderno: **terser** es el estándar.

## Metodología de medición (orden recomendado)

1. **Baseline**: Lighthouse en local (`npx lighthouse http://localhost:3000`) + PageSpeed Insights (field) → anotar métricas.
2. **Presupuesto**: definir topes (tabla de budgets) antes de optimizar, no después.
3. **Atacar por impacto**: peso de imágenes → CSS/JS bloqueantes → fuentes → terceros.
4. **Medir de nuevo** tras cada cambio; comparar lab **y** field (el field tarda días en reflejarse).
5. **CI**: lighthouse-ci o budgets para no regresar. Repetir en cada release relevante.

## Errores comunes

1. **`loading="lazy"` en la imagen del hero** — el LCP se difiere a propósito: empeora la métrica que priorizaste.
2. **Optimizar imágenes de dev y olvidar el pipeline**: a mano funciona una vez; el build debe regenerar todo (`gulp build` reproducible).
3. **Minificar sin sourcemaps en QA**: depurar producción se vuelve imposible — sourcemaps al CDN o archivo aparte.
4. **Subir WebP/AVIF sin fallback**: navegadores antiguos reciben imagen rota — `<picture>` con `type="image/webp"` + `<img>` JPEG de respaldo ([[Formatos Avif y Webp]]).
5. **`width`/`height` ausentes** → CLS gratis al cargar imágenes. La causa #1 de CLS.
6. **Medir solo en localhost con throttling**: simula, pero no reemplaza los datos field.
7. **Concatenar todo en un bundle monolítico**: baja requests pero sube INP (todo se descarga y parsea) — splitting > concat.
8. **Cachear el HTML un año**: los usuarios ven la app vieja; el HTML debe ser lo único sin `immutable`.
9. **Confundir minificación con compresión**: sirves el `.js` minificado pero el server no tiene gzip/brotli — perdiste la capa gratis.

## Cheatsheet de performance

| Técnica | Implementación | Capa |
|---|---|---|
| CSS comprimido | `sass({ style: 'compressed' })` / cssnano | Gulp build |
| JS minificado | `gulp-terser` | Gulp build |
| Imágenes ligeras | `sharp` / `gulp-imagemin` | Gulp build |
| WebP/AVIF | `gulp-webp` / `gulp-avif` / sharp | Gulp build |
| Variantes responsive | `sharp().resize()` + `srcset` | Gulp build + HTML |
| Lazy loading | `loading="lazy"` + `width`/`height` | HTML |
| Prioridad LCP | `fetchpriority="high"` + [[Preload]] | HTML |
| Precarga siguiente | [[Prefetch]] | HTML |
| Hash + immutable | `gulp-rev` + `Cache-Control` | Build + servidor |
| gzip/brotli | nginx/CDN | Servidor |
| Tree shaking/splitting | [[ReactJS - Vite]] | Bundler |
| Auditoría | Lighthouse / PSI / WebPageTest | Medición |
| Topes automáticos | lighthouse-ci budgets | CI |

## Relacionadas

- [[Gulp]] — el task runner que orquesta estas optimizaciones
- [[GulpFile Completo]] — pipeline de producción completo (terser, sharp, rev)
- [[Compilar Sass]] — CSS minificado y sourcemaps
- [[Lazy Loading]] — `loading="lazy"` en detalle
- [[Preload]] — priorizar recursos del camino crítico
- [[Prefetch]] — preparar la siguiente navegación
- [[Formatos Avif y Webp]] — formatos modernos y `<picture>`
- [[ReactJS - Vite]] — tree shaking y code splitting (lo que Gulp no hace)
- [[Html]] — atributos de rendimiento en el marcado
- [[CSS Introducción]] — base sobre la que se apoya el CSS compilado
