---
tags:
  - desarrollo
  - devops
  - instalar-gulp
tipo: concepto
estado: terminado
---

# Instalar Gulp

Instalar Gulp implica **dos piezas** con roles distintos: el **CLI** (`gulp-cli`, el comando `gulp` del terminal, una sola vez por máquina) y la **librería** (`gulp`, local a cada proyecto). Sumado al `gulpfile.js` donde se escriben las tareas, ese es todo el setup. Esta nota cubre el flujo completo, los errores de instalación reales y la migración desde Gulp 3.

## Requisitos previos

| Requisito | Versión mínima (Gulp 5) | Recomendado 2026 |
|---|---|---|
| Node.js | ≥ 10.13 | **LTS actual (18/20/22+)** — las librerías de plugins modernos (sharp, sass) lo exigen |
| npm | incluido con Node | último; o yarn/pnpm equivalentes |
| Sistema operativo | Win/macOS/Linux | — |

Verificar:

```bash
node --version    # v20.x.x
npm --version     # 10.x.x
```

> [!info] ¿Por qué dos piezas (CLI + librería)?
> El **CLI** es el binario global `gulp` que corre desde la terminal; la **librería** es la versión exacta de Gulp con la que trabaja tu proyecto. Separarlas permite que 20 proyectos usen 4 distintas versiones de la librería mientras todos se ejecutan con `gulp` desde el mismo comando global. Grunt, Make y el propio npm usan el mismo patrón.

## 1. Inicializar el proyecto

Gulp vive dentro de un proyecto Node, así que primero existe el `package.json`:

```bash
mkdir mi-sitio && cd mi-sitio
npm init -y
```

Esto genera un `package.json` mínimo. Para proyectos con gulpfile ESM (recomendado, ver sección final), agregá el flag de módulos:

```json
{
  "name": "mi-sitio",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "gulp dev",
    "build": "gulp build"
  }
}
```

Los `scripts` son la capa de conveniencia: en vez de recordar el nombre de la tarea, se ejecuta `npm run build` y eso invoca `gulp build`. Es además lo que usará cualquier CI (ver [[Comandos Git]] para el flujo típico).

## 2. Instalar el CLI global (una vez por máquina)

```bash
npm install --global gulp-cli
```

```bash
gulp --version
# [10:32:11] CLI version: 2.3.0
# [10:32:11] Local version: 5.0.0   ← una vez dentro de un proyecto con gulp instalado
```

El CLI detecta automáticamente el `gulp` **local** del proyecto en el que estés parado y ejecuta ese; el global solo aporta el binario.

> [!warning] No instales `gulp` global, solo `gulp-cli`
> `npm i -g gulp` instala la **librería** en global, que el CLI ignora: no aporta nada y además te ata a una versión global que choca con la de tus proyectos. Lo global es el CLI, lo local es la librería. Nunca al revés.

### Alternativa sin instalación global: `npx`

Si no querés (o no podés) instalar nada global, `npx` resuelve el CLI desde el `node_modules` o lo descarga bajo demanda:

```bash
npx gulp --tasks
npx gulp build
```

Con `gulp-cli` en las `devDependencies` (`npm i -D gulp-cli`) queda 100% local y determinista. El costo: escribir `npx` antes de cada `gulp`, por eso la mayoría agrega los `scripts` de `package.json` y olvida el tema (`npm run build` ya resuelve todo).

## 3. Instalar la librería local (por proyecto)

```bash
npm install --save-dev gulp
```

o en corto:

```bash
npm i -D gulp
```

Esto agrega `gulp` a las **devDependencies** del `package.json`:

```json
{
  "devDependencies": {
    "gulp": "^5.0.0"
  }
}
```

### ¿Por qué `devDependencies` y no `dependencies`?

| | `dependencies` | `devDependencies` |
|---|---|---|
| Se instala con `npm install` (producción) | ✅ | ❌ (salvo `--include=dev`) |
| Se instala con `npm ci` / `npm install` en desarrollo | ✅ | ✅ |
| Instalación con `--omit=dev` (CI de deploy, imágenes Docker) | ✅ | ❌ — **lo que queremos** |
| Caso | lo que corre en producción (react, express) | lo que solo compila: gulp, sass, terser |

Gulp y sus plugins jamás llegan al servidor de producción: solo compilan los assets. Dejarlos en devDependencies reduce el árbol de dependencias del deploy (superficie de ataque, tiempo de `npm ci`, tamaño de imágenes Docker) — y es lo esperado por cualquier auditoría de un proyecto [[Node JS]].

## 4. Crear el `gulpfile.js`

El archivo de configuración **es código**: se definen funciones y se exportan. Un hola mundo real:

```js
// gulpfile.js
export function hola(done) {
  console.log('Hola desde gulp');
  done();                 // señal de "tarea terminada" (async completion)
}

export default hola;      // lo que corre al ejecutar `gulp` sin argumentos
```

```bash
npx gulp hola
# [11:04:22] Starting 'hola'...
# Hola desde gulp
# [11:04:22] Finished 'hola' after 3 ms
```

Detalles que importan de este archivo:

- **`export function`**: Gulp 4+ descubre las tareas como **exportaciones del módulo**. `exports.hola = hola` (CommonJS) funciona igual; lo que no existe más es el registro imperativo `gulp.task('hola', fn)` de Gulp 3 como forma principal.
- **`done()`**: toda función sin stream/promise debe invocar el callback o Gulp aborta con *"Did you forget to signal async completion?"*.
- **`export default`**: define la tarea sin nombre que corre `gulp` a secas.

## 5. Verificar la instalación

```bash
gulp --version      # CLI + versión local
gulp --tasks        # lista de tareas detectadas en el gulpfile
gulp --tasks-simple
gulp hola           # ejecuta una tarea puntual
gulp                # ejecuta la default
```

`gulp --tasks` es la verificación de salud definitiva: si lista tus exports, el CLI encontró el gulpfile y la librería local cargó bien. Salida esperada:

```text
Tasks for ~/mi-sitio/gulpfile.js
──┬── hola
  └── default
```

## Errores comunes de instalación

| Error | Causa | Arreglo |
|---|---|---|
| `gulp: command not found` | No hay CLI global ni se usa `npx` | `npm i -g gulp-cli` o ejecutar `npx gulp ...` |
| `Local gulp not found` | Hay CLI pero **no** `gulp` instalado en el proyecto | `npm i -D gulp` en la raíz del proyecto |
| `Local gulp not found — did you install it?` con gulp presente | Se ejecutó desde una subcarpeta sin `node_modules` arriba, o `node_modules` corrupto | Ejecutar desde la raíz (donde está el `package.json`); si persiste, reinstalar con `npm ci` |
| `CLI version != Local version` en `--version` | Inofensivo: es la separación CLI/librería | Ninguno; el proyecto usa su versión local |
| `EACCES: permission denied` al instalar global | npm global apunta a carpeta de sistema | **No usar `sudo npm`**. Reapuntar el prefijo: `mkdir ~/.npm-global && npm config set prefix ~/.npm-global` y agregarlo al PATH, o mejor: instalar solo local + `npx` |
| `ERESOLVE unable to resolve dependency tree` | Plugin con peerDependencies conflictivas | `npm i -D --legacy-peer-deps`, y de pasoevaluar si el plugin está abandonado |
| `Cannot find module 'gulp'` al ejecutar | `node_modules` no instalado o corrupto | `npm install` (o `npm ci` si hay lockfile) |
| Fallo compilando un plugin (`node-gyp`, `gyp ERR!`) | Plugin viejo con binarios nativos (fase pre-sharp de imagemin) | Reemplazar por alternativa moderna (sharp) o Node LTS menor si el binario es incompatible |
| Caracteres rotos / error de codificación en Gulp 5 | UTF-8 por defecto en streams | Para binarios: `src(..., { encoding: false })` — ver [[Gulp]] |

> [!warning] El error `EACCES` nunca se arregla con `sudo`
> `sudo npm i -g` crea paquetes pertenecientes a root en tu home y provoca la segunda falla al mes siguiente. El arreglo correcto es cambiar el `prefix` de npm a tu home o, mejor, **evitar dependencias globales**: `gulp-cli` como devDependency + `npx` no toca permisos del sistema jamás.

## Gulpfile ESM vs CommonJS

Gulp 4.0.2+ y el CLI actual cargan gulpfiles **ESM** sin problemas. Tres formas de tener uno:

| Forma | Requisito | Nota |
|---|---|---|
| `gulpfile.js` + `"type": "module"` en package.json | Todo el proyecto es ESM | Estándar en 2026; permite `import { src } from 'gulp'` |
| `gulpfile.mjs` | Nada más | ESM explícito sin tocar el package.json |
| `gulpfile.js` CommonJS (sin `type: module`) | — | El formato histórico; los plugins ESM-only (del 7+, gulp-rev 11+, gulp-imagemin 8+) **no** se pueden `require` — hay que importarlos dinámicamente |

```js
// gulpfile.js ESM — el formato de todo este vault
import { src, dest, series, parallel, watch } from 'gulp';
```

> [!info] ¿Por qué el `"type": "module"`?
> Sin él, Node trata `.js` como CommonJS y sentencias `import/export` fallan con `SyntaxError: Cannot use import statement outside a module`. El flag convierte todos los `.js` del proyecto en ESM — el estándar de [[Node JS]] moderno y la única forma cómoda de usar los plugins ESM-only del ecosistema actual.

## Migración desde Gulp 3

Si el proyecto venía de Gulp 3 (2014-2019), la instalación es la misma pero el gulpfile necesita cambios:

| Gulp 3 | Gulp 4/5 |
|---|---|
| `var gulp = require('gulp')` | `import { src, dest, series, parallel, watch } from 'gulp'` |
| `gulp.task('css', fn)` como registro principal | `export function css(...)` |
| `gulp.task('build', ['css', 'js'], fn)` | `export const build = series(css, js)` o `parallel` |
| `gulp.start('css')` | Llamar la función directamente o `series` |
| `run-sequence` (plugin de orden) | `series(...)` nativo — desinstalarlo |
| Plugins `gulp-util` | Descontinuado → reemplazar por librerías puntuales (`fancy-log`, `ansi-colors`) |
| `gulp.src` de binarios sin más | Gulp 5: añadir `{ encoding: false }` |

Checklist de migración: 1) `npm i -D gulp@latest` (mayor), 2) convertir tareas a `export function`, 3) reemplazar arrays de dependencias por `series`/`parallel`, 4) quitar `run-sequence`/`gulp-util`, 5) `encoding: false` en imágenes/fuentes, 6) `gulp --tasks` para confirmar que el grafo se carga.

## Verificación final

```bash
gulp --version        # CLI y versión local correctas
gulp --tasks          # tareas del gulpfile listadas
node --check gulpfile.js   # sintaxis del gulpfile sin ejecutarlo
npm run build         # el pipeline completo (si hay scripts)
```

Si `--tasks` lista tus exports y `hola` imprime sin errores de async, la instalación está completa: el siguiente paso es construir tareas reales → [[GulpFile Completo]]; con Sass → [[Compilar Sass]].

## Relacionadas

- [[Gulp]] — qué es, modelo de streams y panorama 2026
- [[GulpFile Completo]] — gulpfile de producción task por task
- [[Compilar Sass]] — primer pipeline real (Sass dentro de Gulp)
- [[Node JS]] — runtime, ESM y `package.json`
- [[Comandos Git]] — flujo de trabajo con el código del proyecto
