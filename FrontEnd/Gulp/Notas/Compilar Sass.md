---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[Brain/FrontEnd/Gulp/GULP|GULP]]"
---
----------
# Como compilar sass desde gulp

Primero vamos a instalar la dependencia:

```json
npm i -D gulp-sass
```

ya con esto es cuestión de tener la siguiente configuración en nuestro archivo de de guilfile.js:

```jsx
import { src, dest } from 'gulp'; // importando src (para identificar archivos) y dest (para colocar archivos)
import * as dartSass from 'sass'; // importando todos los módulos de sass
import gulpSass from 'gulp-sass'; // importando gulp-sass

const sass = gulpSass(dartSass); // función que nos ayuda a compilar

export function css(done) {

  src("src/scss/app.scss") // identifica el archivo
    .pipe(sass()) // compila el archivo
    .pipe(dest("build/css")) // ubica el archivo del código compilado.

  done();
}
```

En este punto vemos que no se compila por lo que también es necesario configurar el watch para que lo haga y para esto modificaremos el código de la siguiente manea:

```jsx
import { src, dest, watch } from 'gulp'; // importando src (para identificar archivos) y dest (para colocar archivos)
import * as dartSass from 'sass'; // importando todos los módulos de sass
import gulpSass from 'gulp-sass'; // importando gulp-sass

const sass = gulpSass(dartSass); // función que nos ayuda a compilar

export function css(done) {

  src("src/scss/app.scss") // identifica el archivo
    .pipe(sass().on('error', sass.logError)) // compila el archivo
    // el .on('error', sass.logError) nos permite ver errores en la compilación
    .pipe(dest("build/css")) // ubica el archivo del código compilado.

  done();
}

export function dev() {

  watch('src/scss/app.scss', css)

}
```

En el caso de tener haber separado en múltiples archivos nuestro SCSS vamos a modificar un poco el watch de la siguiente manera:

```jsx
import { src, dest, watch } from 'gulp'; // importando src (para identificar archivos) y dest (para colocar archivos)
import * as dartSass from 'sass'; // importando todos los módulos de sass
import gulpSass from 'gulp-sass'; // importando gulp-sass

const sass = gulpSass(dartSass); // función que nos ayuda a compilar

export function css(done) {

  src("src/scss/app.scss") // identifica el archivo
    .pipe(sass().on('error', sass.logError)) // compila el archivo
    // el .on('error', sass.logError) nos permite ver errores en la compilación
    .pipe(dest("build/css")) // ubica el archivo del código compilado.

  done();
}

export function dev() {

  watch('src/scss/**/*', css)

}
```

cuando compilamos vamos a ver que no se genera el archivo map de css, para esto es cuestión de activarlo en nuestra configuración de gulp:

```jsx
import { src, dest, watch } from 'gulp';
import * as dartSass from 'sass';
import gulpSass from 'gulp-sass';

const sass = gulpSass(dartSass);

function css(done) {
  src('src/scss/app.scss', { sourcemaps: true })
    .pipe(sass().on('error', sass.logError))
    .pipe(dest('dist/css/', { sourcemaps: '.' }));

  done();
}

export function dev() {
  watch('src/scss/**/*.scss', css);
}
```

## Agregar autoprefixer
esto nos permite hacer el código mucho mas compatible con todos los navegadores, para esto vamos a necesitar de dos dependencias que son:

```sh
npm i -D autoprefixer gulp-postcss
```

El como se implementa es de la siguiente manera:

```js
import { src, dest, watch } from 'gulp';
import * as darkSass from 'sass';
import gulpSass from 'gulp-sass';
import postcss from 'gulp-postcss';
import autoprefixer from 'autoprefixer';


const sass = gulpSass(darkSass);


function css(done) {
  //compilar sass
  src("./src/scss/app.scss")
    .pipe(sass({ style: 'compressed' }))
    .pipe(postcss([autoprefixer()]))
    .pipe(dest("./build/css"));
  done();
}

function dev() {
  watch("./src/scss/app.scss", css);
}

exports const cssex = css;
exports const devex = dev;
```

Para configurar con los navegadores a los que queremos hacer compatibles vamos a configurar nuestro `package.js` y vamos a agregar lo siguiente:

```json
  "browserslist": [
    "last 1 version",
    "> 1%"
  ]
```

esto especifica que vamos a adaptarlo a la última versión de los navegadores y ademas para el todos los navegadores que tengan un uso de mas de 1%.

