---
aliases: ["SASS", "SCSS", "Syntactically Awesome Style Sheets"]
tags:
  - dev/frontend/scss
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Sintaxis de Sass]]"
  - "[[Mixins]]"
  - "[[Separar Sass en Archivos]]"
  - "[[Compilar Sass]]"
  - "[[CSS Introducción]]"
fuente: "https://sass-lang.com/documentation/"
---

# ¿Qué es Sass?

**SASS** son las siglas de ***S**yntactically **A**wesome **S**tyle**s**heets*: es un **preprocesador de CSS** — un programa que toma archivos con sintaxis extendida (variables, anidación, mixins, funciones, maps, loops) y los **compila** a CSS estándar que el navegador entiende.

> [!info] Preprocesador = CSS con superpoderes **compilados**
> Los navegadores no entienden Sass. Un compilador traduce `.scss` → `.css` antes de que el navegador vea nada. Por eso Sass no es "mejor CSS": es una **capa de productividad en tiempo de compilación** que produce CSS puro. Todo lo que Sass resuelve con lógica propia, el CSS moderno lo ha ido copiando parcialmente (variables, anidación) — pero no del todo (ver "Sass vs CSS moderno").

## Linaje: Ruby Sass → LibSass → Dart Sass

Tres implementaciones distintas del mismo lenguaje. Saber cuál usó cada proyecto viejo explica comportamientos raros al leer código heredado:

| Implementación | Lenguaje | Vigencia | Notas |
|---|---|---|---|
| **Ruby Sass** (2006–2019) | Ruby | **Descontinuada** (1.32 EOL) | La original; lenta; ya no recibe features ni security fixes |
| **LibSass** (2013–2020+) | C++ | **Abandonada** (de-facto; 4.0 cancelada) | Rápida para su época; se usaba vía `node-sass` — **queda atrás del lenguaje** (sin `@use`, sin módulos modernos) |
| **Dart Sass** (2016–) | Dart | **✅ Implementación oficial y activa** | La que mantienen los creadores; CLI `sass`; primera en recibir features; compila a JS si hace falta |

> [!warning] `node-sass` es historia
> Proyectos antiguos con `node-sass` en `devDependencies` usan LibSass: sintaxis más vieja (puede fallar con `@use`, `math.div`, maps anidados) y builds frágiles (binarios nativos por versión de Node). Migralo a `sass` (Dart Sass puro, JS): `npm uninstall node-sass && npm install -D sass`. Los comandos de la CLI son casi idénticos.

## Las dos sintaxis: `.scss` vs `.sass`

| | `.scss` | `.sass` |
|---|---|---|
| Delimitadores | Llaves `{}` y `;` | **Indentación** (sin llaves ni punto y coma) |
| Compatibilidad con CSS | **Total** — todo CSS válido es SCSS válido | No directa |
| Comentarios `//` | Comentario de compilación (no sale) | Igual |
| Uso actual | **Estándar de facto** (95%+ del ecosistema) | Legado; sin uso en proyectos nuevos |

```scss
// .scss
.header {
  display: flex;
  .logo { margin-top: 10rem; }
}
```

```sass
// .sass — mismo resultado
.header
  display: flex
  .logo
    margin-top: 10rem
```

> [!tip] Cuál usar
> **`.scss`** en todo proyecto nuevo: es CSS válido, permite migración incremental y es lo que usa el ecosistema (Vite, frameworks, plantillas). `.sass` solo aparece en proyectos heredados. El lenguaje (variables, mixins, maps…) es **idéntico** en ambos; cambia solo la carátula.

## Qué aporta Sass (y qué no)

**Ventajas**:

- **Orden y estructura**: proyectos de CSS grandes se mantienen modularizados en partials con `@use`/`@forward` (ver [[Separar Sass en Archivos]]).
- **Variables de compile-time**: tokens centralizados — cambiar uno actualiza todo el proyecto (ver [[Sintaxis de Sass#Variables]]).
- **Anidación**: la estructura del CSS refleja la del HTML.
- **Lógica real**: `@if`, `@for`, `@each`, `@while` — CSS no tiene nada equivalente en generalidad.
- **Funciones y matemática**: `math.div`, `color.adjust`, maps como sistemas de diseño programables (ver [[Sintaxis de Sass]]).
- **Mixins parametrizados**: reutilización con argumentos y `@content` (ver [[Mixins]]).
- **Compatible**: todo CSS válido es SCSS válido — la migración es incremental.

**Desventajas**:

- **Otra sintaxis** que aprender y una capa de build más en el pipeline.
- **No es nativo**: requiere compilación (CLI o integración con el build — ver [[Compilar Sass]]).
- **La anidación abusa fácil**: selectores anidados profundos generan selectores frágiles y más específicos (ver [[Especificidad]]) y más código del necesario.

## Cómo se compila: modos y APIs

| Vía | Cuándo | Ejemplo |
|---|---|---|
| **CLI global** | Aprendizaje, scripts sueltos | `npm i -g sass` → `sass src/app.scss dist/app.css` |
| **Watch** | Desarrollo sin bundler | `sass --watch src:dist` |
| **npm scripts** | Proyectos Node sin bundler | `"sass": "sass src/scss:dist/css --watch"` |
| **JS API** | Integración en herramientas Node | `sass.compile('app.scss', { style: 'compressed' })` |
| **Vite (nativo)** | Cualquier proyecto Vite | `npm i -D sass` + `import './app.scss'` — sin config extra |
| **PostCSS + postcss-sass** | Pipelines PostCSS existentes | Compila SCSS como paso del pipeline (menos común) |
| **Gulp** | Proyectos con Gulp | `gulp-sass` — ver [[Compilar Sass]] |

Comandos CLI esenciales:

```bash
npm install -g sass                    # instalar una vez
sass src/app.scss dist/app.css         # compilar una vez
sass src/scss:dist/css --watch         # modo desarrollo (watch por carpeta)
sass --style=compressed src/app.scss dist/app.min.css   # minificado
sass --no-source-map src/app.scss dist/app.css          # sin mapa de fuentes
```

> [!tip] En Vite la integración es transparente
> Vite detecta `sass` en devDependencies y compila `.scss` en el dev server y el build, con HMR incluido. No hay plugin ni config: solo `import './estilos.scss'` en tu punto de entrada. Detalle en [[ReactJS - Vite]].

### Cómo funciona la compilación por dentro

El compilador Dart Sass no "reemplaza texto": construye y evalúa el programa completo:

1. **Parseo**: el `.scss` se convierte en un AST (árbol sintáctico) — reglas, at-rules, expresiones.
2. **Evaluación**: se resuelven variables, funciones, maps y directivas de control; el resultado es un **árbol de CSS plano** (sin `$`, sin `@mixin`).
3. **Emisión**: se serializa a CSS, en el **estilo** elegido, generando opcionalmente **source map** (`.css.map`) que liga cada línea del CSS a su archivo Sass origen — indispensable para depurar en DevTools.
4. **Códigos de caracteres**: si el CSS contiene caracteres no-ASCII (tildes en `content:`), Sass agrega `@charset "UTF-8"` automáticamente.

| Estilo de salida | CLI | Resultado |
|---|---|---|
| `expanded` (default) | `--style=expanded` | Legible, 1 regla por bloque — desarrollo |
| `compressed` | `--style=compressed` | Minificado — producción (minificación final suele hacer el bundler) |
| Source maps | on por defecto / `--no-source-map` | `.css.map` para DevTools |

> [!info] Compile-time = no hay Sass en el navegador
> El CSS de salida no lleva "runtime" de Sass: es idéntico en calidad al CSS escrito a mano. Eso también significa que bugs de Sass se manifiestan **en compilación** (errores con stack trace de archivo:línea), no en runtime — leer ese error es la primera herramienta de debugging.

## Sass vs CSS moderno: la tabla de paridad (2026)

El CSS nativo avanzó mucho desde 2020. Comparación honesta feature por feature:

| Capacidad | CSS nativo (2026) | Sass | Veredicto |
|---|---|---|---|
| Variables | ✅ Custom properties `--x` (**runtime**, tematizables) | ✅ `$x` (**compile-time**, no cambian sin recompilar) | **Complementarias**: Sass para tokens fijos del sistema, custom properties para temas dinámicos |
| Anidación | ✅ Nativa (2023+) con reglas de `&` | ✅ Más flexible (selectores de referencia, `@at-root`) | Paridad práctica; Sass permite más |
| Condicionales | ⚠️ Solo `@container style()` (limitado) | ✅ `@if`/`@else if`/`@else` completos | **Gana Sass** |
| Bucles | ❌ No hay | ✅ `@for`, `@each`, `@while` | **Gana Sass** |
| Matemática | ⚠️ `calc()` en runtime, sin lógica de unidades | ✅ `math.div`, `math.round`, aritmética de unidades en compile-time | **Gana Sass** |
| Funciones propias | ❌ No | ✅ `@function` + `@return` | **Gana Sass** |
| Datos/estructuras | ❌ No | ✅ Maps y listas programables | **Gana Sass** |
| Reutilización de bloques | ⚠️ Custom properties + clases/mixins CSS (props + `@property`) | ✅ `@mixin`/`@include` con args + `@content` | Gana Sass para parametrizar |
| Estilos condicionales por estado | ✅ `:has()`, `:is()`, `:where()`, container queries | ⚠️ Sass los compila como texto, sin entenderlos | **Gana CSS** |
| Selección contextual | ✅ `:has()` (selector "parent") | ❌ No tiene equivalente | **Gana CSS** |
| At-rules dinámicas | ✅ `@property`, `@scope`, `@layer` | ❌ No modelados | **Gana CSS** |

> [!info] ¿Para qué sirve Sass en 2026 entonces?
> El nicho que el CSS todavía no cubre: **lógica de diseño programable** (generar clases con `@each`, sistemas de breakpoints como mapa, matemática de unidades, temas como maps compilados, funciones de color como helpers). Todo lo demás (variables runtime, anidación, capas, `:has`) es mejor nativo. La decisión real no es "Sass o CSS": es **qué capa resuelve qué problema** — muchos proyectos usan ambas (Sass para el sistema de diseño, custom properties para la theming en runtime). Y junto a Tailwind conviven sin conflicto (ver más abajo).

### Prueba de fuego: lo que el CSS nativo NO puede hacer

Worked example — un sistema de escalas de espaciado y utilidades generado con lógica que no tiene equivalente nativo:

```scss
@use 'sass:math';
@use 'sass:map';

// 1. Datos: el sistema como MAPA (el CSS no tiene datos)
$escala: (
  xs: 0.25rem,
  sm: 0.5rem,
  md: 1rem,
  lg: 2rem,
  xl: 4rem,
);

// 2. Generación con @each + interpolación (el CSS no tiene loops)
@each $nombre, $valor in $escala {
  .m-#{$nombre} { margin: $valor; }
  .p-#{$nombre} { padding: $valor; }
  .gap-#{$nombre} { gap: $valor; }
}

// 3. Función con lógica de unidades (solo compile-time la resuelve)
@function escalado($valor, $factor: 1.5) {
  @return $valor * $factor;
}

.texto-lg { font-size: escalado(map.get($escala, md)); }  // 1.5rem
```

Ese archivo compila a 16 reglas que escribirías a mano en 40 líneas — y cambiar el mapa los actualiza todos. Ninguna combinación de custom properties, `:has()` ni container queries produce esto hoy. (Con Tailwind esto mismo se logra con `@theme` — ver [[TailwindCSS - Instalación y Configuración]]; Sass es la alternativa sin framework.)

### Guía de decisión (2026)

| Situación | Herramienta | Por qué |
|---|---|---|
| SPAs React/Vue modernas | **Tailwind** (o CSS Modules) + custom properties | Utilities + tokens runtime; el ecosistema ya vive ahí |
| Design system con lógica (escalas, temas, generación) | **Sass** parcial | Maps + loops + funciones son únicos |
| Proyecto legacy con SCSS (Bootstrap, etc.) | **Sass** (migrar `@use`) | Reescribirlo todo sale más caro que mantenerlo |
| Sitio estático pequeño, 2026 | **CSS nativo** (anidación + custom props) | Cero tooling; nesting + `:has()` cubren casi todo |
| Blog / contenido largo | Tailwind + `@typography` o CSS propio | El CSS de contenido no escala con utilities |
| Equipo que no quiere clases en HTML | Sass/BEM + CSS Modules | Adopción y convención del equipo pesan más que la herramienta |

## Ecosistema construido con Sass

 frameworks y herramientas históricas escritos (o históricamente disponibles) en Sass:

| Proyecto | Rol de Sass |
|---|---|
| Bootstrap 4/5 | Todo el sistema (variables, mixins, functions) en SCSS — customizarlo es editar sus partials |
| Bulma | Grid + componentes como mixins SCSS |
| Foundation | Sass desde su origen |
| Skeleton, Milligram | Versiones Sass de resets/estilos base |
| Sistemas de diseño corporativos | Tokens como maps Sass compilados a custom properties |

Nota: Tailwind **no** está construido con Sass — es utility-first con su propio motor. Convivencia: se usa Sass para lo que Tailwind/CSS no resuelve (ver [[TailwindCSS - Sintaxis Básica]]).

## Roadmap: `@import` fuera, Dart Sass 2.0

Cambios de lenguaje que todo desarrollador debe conocer:

| Evento | Fecha | Impacto |
|---|---|---|
| `@use`/`@forward` (sistema de módulos) | 2019 (1.23) | Namespaces, encapsulado, config — la forma moderna (ver [[Separar Sass en Archivos]]) |
| División `/` deprecada | 1.33 (2021) | `/` ahora es separador (`font: 12px/1.5`); usar `math.div()` |
| Funciones globales deprecadas | 1.79–1.80 (2024) | `darken()`, `lighten()`, `map-get()`, `percentage()`… → módulos `sass:color`, `sass:map`, `sass:math` |
| **`@import` deprecado** | **1.80.0 (oct 2024)** | Warning en toda compilación; eliminado en **Sass 3.0** |
| Dart Sass 2.0 | Futuro | Remoción de `@import` y APIs globales deprecadas; posibles cambios de sintaxis `.sass` |

Práctica recomendada hoy: todo `@import` heredado se migra a `@use`/`@forward` (pasos en [[Separar Sass en Archivos]]) — es cuestión de tiempo que las herramientas nuevas dejen de soportar `@import`.

## Errores comunes

1. **Usar `node-sass`** → LibSass vieja, sin `@use` ni `math.div`, builds rotos al actualizar Node. Migrar a `sass`.
2. **`Undefined function` al usar `darken()`/`map-get()` con warnings** → son APIs globales deprecadas; importar los módulos (`@use 'sass:color'` → `color.adjust`).
3. **Compilar partials a CSS** → los `_*.scss` no deben compilarse sueltos; solo se consumen (ver [[Separar Sass en Archivos]]).
4. **Esperar variables Sass en runtime** → `$x` no existe en el CSS final; para cambiar estilos desde JS hay que usar custom properties.
5. **Anidación profunda "porque puedo"** → selectores gigantes y específicos; regla de los 3 niveles (ver [[Sintaxis de Sass]] y [[Especificidad]]).

## Cheatsheet

| Tema | Referencia |
|---|---|
| Instalar | `npm install -D sass` (o `-g` para CLI global) |
| Compilar | `sass entrada.scss salida.css` |
| Watch | `sass --watch src:dist` |
| Minificar | `--style=compressed` |
| Implementación oficial | Dart Sass (Ruby y LibSass → legado) |
| Sintaxis a usar | `.scss` (`.sass` = legado) |
| Módulos | `@use` / `@forward` (nunca `@import` en código nuevo) |
| División | `math.div(a, b)` — nunca `/` |
| Color | `color.adjust/change/mix` — nunca `darken/lighten` globales |

## Relacionadas

- [[Sintaxis de Sass]] — el lenguaje completo: variables, maps, funciones, control
- [[Mixins]] — reutilización con parámetros y `@content`
- [[Separar Sass en Archivos]] — módulos `@use`/`@forward` y partials
- [[Compilar Sass]] — compilación del proyecto con Gulp
- [[CSS Introducción]] — la base sobre la que Sass extiende
- [[Especificidad]] — por qué la anidación profunda es peligrosa
- [[TailwindCSS - Sintaxis Básica]] — la otra filosofía (utility-first) con la que Sass convive
- [[ReactJS - Vite]] — cómo Vite compila SCSS sin configuración
