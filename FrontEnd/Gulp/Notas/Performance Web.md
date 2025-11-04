---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[Brain/FrontEnd/Gulp/GULP|GULP]]"
---
----------
# Performance Web
El performance web se refiere a la velocidad y eficiencia con la que las páginas web son descargadas y mostradas en el navegador del usuario.
Un buen rendimiento web es crucial para proporcionar una experiencia de usuario óptima, mejorar la retención y conversión en sitios web y aplicaciones web.
Para esto vamos a realizar las siguiente recomendaciones y como se hace cada una de ellas:
## Minificar CSS y JS.

En el caso de css tenemos que realizar un aligera modificación, pero para JavaScript vamos a tener que instalar una dependencia:

```jsx
npm i --save-dev gulp-terser
```

ya con esto nuestro código para minificar quedaría de la siguiente manera:

```jsx
import { src, dest, watch } from 'gulp';
import * as dartSass from 'sass';
import gulpSass from 'gulp-sass';
import terser from 'gulp-terser';

const sass = gulpSass(dartSass);

function css(done) {
  src('src/scss/app.scss', { sourcemaps: true })
    .pipe(sass({ style: 'compressed' }).on('error', sass.logError))
    .pipe(dest('dist/css/', { sourcemaps: '.' }));

  done();
}

function js(done) {

  src('src/js/app.js')
    .pipe(terser())
    .pipe(dest('dist/js'));

  done();
}

export function dev() {
  watch('src/scss/**/*.scss', css);
  watch('src/js/app.js', js);
}
```

## Carga Diferida de Imágenes con Lazy Loading.

Mediante Lazy Loading vamos a evitar que las imágenes que tengamos en nuestra web se descarguen todas de golpe, esto para que estas se descarguen según se lo requiera en la web.

para esto vamos en el caso de las imágenes en nuestro html agregar el atributo de loading con lazy, a demás de darle una altura y una anchura:

```jsx
 <img width="300" height="300" loading="lazy" src="src/img/imagen_dj.jpg" alt="Sobre Festival">
```

En el caso de imágenes generadas o agregadas mediante Java Script vamos a tener que agregarlas desde el mismo Java Script.

En el caso de videos vamos a establecer un preload auto.

### Servir formatos modernos

```bash
npm i -D gulp-webp gulp-avif
```

```jsx
import webp from 'gulp-webp';
import avif from 'gulp-avif';

function ImageWebp() {
  const opt = {
    quality: 50,
  }
  return src('./src/img/**/*.{jpg,png}', { encoding: false })
    .pipe(webp(opt))
    .pipe(dest('./build/img/'))
}

function ImageAvif() {
  const opt = {
    quality: 50,
  }
  return src('./src/img/**/*.{jpg,png}', { encoding: false })
    .pipe(avif(opt))
    .pipe(dest('./build/img/'))
}

```


## Minificar Imágenes
Lo que nosotros logramos con la minificacion es tratar de hacerlas menos pesadas para tener optimizadas en la web.

las dependencias para la función es:

```sh
npm install --save-dev gulp-imagemin mozjpeg optipng gifsicle svgo
```

```js
import imagemin, { gifsicle, mozjpeg, optipng, svgo } from 'gulp-imagemin';

function imageMin() {
  return src('./src/img/**/*', { encoding: false })
    .pipe(imagemin([
      mozjpeg({
        quality: 80,
        progressive: true
      }),
      optipng({
        optimizationLevel: 5
      }),
      gifsicle({
        interlaced: true,
        optimizationLevel: 2
      }),
      svgo({
        plugins: [
          { name: 'removeViewBox', params: { active: false } },
          { name: 'removeDimensions', params: { active: true } }
        ]
      })
    ], {
      verbose: true
    }))
    .pipe(dest("./build/img"));
}

```


## Crear formatos WebP y Avif
Estos formatos son los mas modernos y optimizados para usar en la web por lo que vamos a utilizarlos y vamos a ver como crearlos:

```sh
npm i --save-dev gulp-webp gulp-avif
```

```js
import webp from 'gulp-webp';
import avif from 'gulp-avif'

// Crear webp
function versionWeb() {
  const opciones = {
    quality: 50,
  }
  return src('./src/img/**/*.{png,jpg}', { encoding: false })
    .pipe(webp(opciones))
    .pipe(dest('./builds/img'))
}
// Crear Avif
function versionAvif() {
  const opciones = {
    quality: 50,
  }
  return src('./src/img/**/*.{png,jpg}', { encoding: false })
    .pipe(avif(opciones))
    .pipe(dest('./builds/img'))
}

```