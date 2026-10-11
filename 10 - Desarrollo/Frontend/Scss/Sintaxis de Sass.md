---
aliases: ["Sintaxis de SASS", "Sintaxis SCSS"]
tags:
  - dev/frontend/scss
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Qué es Sass]]"
  - "[[Mixins]]"
  - "[[Especificidad]]"
fuente: "https://sass-lang.com/documentation/syntax/"
---

# Sintaxis de Sass

Sass existe en dos sintaxis. Este cubre el lenguaje completo: desde variables hasta funciones y directrices de control.

## Las dos sintaxis: `.sass` vs `.scss`

**`.scss`** (llaves y punto y coma — la recomendada, 100% compatible con CSS):

```scss
.header {
  display: flex;
  .logo {
    margin-top: 10rem;
  }
}
```

**`.sass`** (indentación — sintaxis original, sin llaves ni punto y coma):

```sass
.header
  display: flex
  .logo
    margin-top: 10rem
```

> [!tip] Cuál usar
> **`.scss`** en todo proyecto nuevo: es CSS válido, permite migración incremental y es lo que usa el ecosistema (Vite, frameworks, la mayoría de plantillas). `.sass` solo aparece en proyectos heredados.

## Variables

```scss
$color-primario: #e1e1e1;
$separacion: 3rem;
$fuente: Arial, Helvetica, sans-serif;
$breakpoint-tablet: 768px;

.elemento {
  color: $color-primario;
  margin: $separacion;
  font-family: $fuente;
}
```

- Las variables **se resuelven en compile-time**: el CSS de salida no las conoce.
- Convención: `$snake-case` para variables Sass.

### `!default`

Asigna un valor **solo si la variable no está definida** — esencial para librerías y partials reutilizables:

```scss
// _config.scss
$color-primario: #3498db !default; // el consumidor puede sobreescribirlo antes del @use
```

### Custom properties (CSS variables) vs variables Sass

```scss
$tema: dark;          // Sass: la decide el compilador, no cambia en runtime
:root {
  --tema-fondo: #111; // CSS: vive en el navegador, se puede cambiar con JS
}
```

Usá variables Sass para tokens de diseño fijos; custom properties cuando necesites **temas dinámicos o runtime**.

## Anidación

```scss
.nav {
  display: flex;

  a {
    color: $color-primario;

    &:hover {          // & = selector padre
      text-decoration: underline;
    }

    &.activo {         // compuesto: .nav a.activo
      font-weight: bold;
    }

    .icono & {         // descendencia invertida: .icono .nav a
      margin-right: 4px;
    }
  }
}
```

> [!warning] Regla de los 3 niveles
> No nestees más de 3 niveles: la anidación profunda genera selectores largos, altamente específicos y difíciles de sobreescribir (ver [[Especificidad]]). Si necesitás 4 niveles, casi siempre conviene una clase nueva.

## Interpolación `#{}`

Inserta expresiones Sass dentro de selectores y valores:

```scss
$columnas: 4;
$-breakpoint: "screen and (min-width: 768px)";

.grid-#{$columnas} {         // → .grid-4
  grid-template-columns: repeat($columnas, 1fr);
}

@media #{$breakpoint} { ... }
```

## Operaciones y matemática

```scss
$gap: 1rem;
$padding: $gap * 2;        // 2rem
$ancho: 100% - 40px;

// Importante: desde Dart Sass, dividir se hace con math.div
@use 'sass:math';
$mitad: math.div($gap, 2); // $gap / 2 ya genera deprecación
```

> [!warning] Unidades y `/`
> El `/` dejó de significar división (ahora se lee como separador de fuentes: `font: 12px/1.5`). Usá siempre `math.div()` del módulo `@sass:math`.

## Mapas

Colecciones clave→valor, ideales para temas y configuraciones:

```scss
@use 'sass:map';

$colores: (
  primario: #3498db,
  exito: #2ecc71,
  error: #e74c3c,
);

.texto-exito {
  color: map.get($colores, exito); // → #2ecc71
}
```

## Directivas de control

```scss
@use 'sass:math';

@for $i from 1 through 4 {              // bucle
  .columna-#{$i} { width: math.div(100%, $i); }
}

@each $nombre, $hex in (primario: #3498db, error: #e74c3c) {
  .bg-#{$nombre} { background: $hex; }
}

$tema: dark;
@if $tema == dark {
  body { background: #111; }
} @else {
  body { background: #fff; }
}
```

## Funciones

Devuelven un **valor** (los mixins devuelven **estilos** — ver [[Mixins]]):

```scss
@use 'sass:math';
@use 'sass:color';

@function ancho-col($cols, $gap: 1rem) {
  @return math.div(100% - ($cols - 1) * $gap, $cols);
}

.tres-col {
  width: ancho-col(3);        // cálculo reutilizable
}

// Funciones oficiales (reemplazan a darken/lighten globales)
.tono-oscuro { color: color.adjust(#3498db, $lightness: -15%); }
```

> [!info] Nombres oficiales de módulos
> `@use 'sass:math'` · `sass:color` · `sass:map` · `sass:string` · `sass:selector` · `sass:list`. Reemplazan a las funciones globales deprecadas (`darken()`, `lighten()`, `map-get()`, `divide()`).

## Placeholders `%` con `@extend`

Definen estilos que **nunca se emiten solos** — solo existen cuando otra clase los extiende:

```scss
%card-base {
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.card {
  @extend %card-base;
  background: white;
}

.card-oscura {
  @extend %card-base;
  background: #222;
}
```

El CSS de salida agrupa ambos selectores sin duplicar declaraciones. Detalle y cuándo preferir mixins: ver [[Mixins]].

## Relacionadas

- [[Qué es Sass]] — qué es el preprocesador y cómo se compila
- [[Mixins]] — reutilización con parámetros y `@content`
- [[Separar Sass en Archivos]] — organización en módulos `@use`/`@forward`
- [[Especificidad]] — por qué la anidación profunda se sale de control
- [[CSS Grid]] — patrones de grid como los del mixin de [[Mixins]]
