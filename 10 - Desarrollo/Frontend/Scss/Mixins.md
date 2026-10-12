---
tags:
  - desarrollo
  - frontend
  - mixins
tipo: concepto
estado: terminado
---

# Mixins

Los **mixins** permiten definir bloques de estilos **reutilizables** que se insertan donde se necesiten con `@include`, con parámetros y con contenido variable (`@content`). Son la herramienta de Sass para **abstraer declaraciones** (no valores — para eso están las funciones, ver [[Sintaxis de Sass#Funciones]]).

> [!info] Mixin ≠ Función
> Un **mixin** emite **declaraciones CSS** (`display: grid`, media queries…). Una **función** devuelve un **valor** (`2rem`, `#3498db`) y no genera CSS por sí misma. Si el resultado es `property: value` → mixin; si es un valor que va dentro de otra declaración → función.

## Definición e inclusión

```scss
// Definición (en cualquier parte del archivo o de un módulo)
@mixin titulo-seccion {
  font-size: 2rem;
  font-weight: bold;
  margin-bottom: 1rem;
}

// Uso
.seccion h2 {
  @include titulo-seccion;
}
```

- `@include` **pega las declaraciones** del mixin en el punto de inclusión (es "copiar-pegar parametrizado", no una llamada en runtime).
- Los mixins pueden anidarse: un mixin puede incluir a otros.
- Conviven con la anidación normal y con `&`:

```scss
.boton {
  @include sombra-suave;
  &:hover { @include sombra-media; }
}
```

## Mixins con parámetros

```scss
@mixin grid($columnas, $gap: 5rem) {   // $gap tiene valor por defecto
  display: grid;
  grid-template-columns: repeat($columnas, 1fr);
  gap: $gap;
}

.layout-3 { @include grid(3); }         // gap: 5rem (default)
.layout-2 { @include grid(2, 2rem); }   // gap: 2rem
.layout-custom { @include grid($gap: 3rem, $columnas: 4); } // nombrado, cualquier orden
```

Los parámetros con valor por defecto hacen al mixin **progresivo**: funciona con lo mínimo y escala cuando necesitás control extra. Ver [[CSS Grid]] para el contexto de las propiedades usadas.

### Los mismos modos de argumentos que las funciones

| Modo | Ejemplo | Notas |
|---|---|---|
| Posicional | `@include grid(3, 2rem)` | En el orden de la firma |
| Nombrado | `@include grid($gap: 2rem)` | Orden libre; obligatorio para saltear defaults |
| Default | `@include grid(3)` | El resto usa su valor por defecto |
| Variables `...` | `@mixin util($props...) { #{$props}; }` | El resto de args se expande como declaraciones |

```scss
// Argumentos variables: reciben de más y los expanden
@mixin propiedades($declaraciones...) {
  #{$declaraciones};
}

.sombra {
  @include propiedades(box-shadow: 0 2px 4px rgba(0, 0, 0, .2), color: #333);
}
// → { box-shadow: ...; color: #333; }
```

El patrón `#{$args...}` con argumentos variables es la base de los "utility mixins" dinámicos: el mixin no conoce las propiedades, las recibe.

> [!warning] Argumentos sin default + sin nombre = error
> `@include grid($gap: 2rem)` falla si `$columnas` (sin default) no se pasa. Regla: o todos tienen default, o los obligatorios van primero en positional.

## `@content`: bloques de contenido

`@content` permite pasarle **CSS al mixin desde el `@include`** — el bloque `{}` del include se inyecta donde esté `@content`. Es lo que hace posible encapsular media queries reutilizables:

```scss
@use 'sass:map';

$breakpoints: (mobile: 480px, tablet: 768px, desktop: 1024px);

@mixin mobile  { @media (min-width: map.get($breakpoints, mobile))  { @content; } }
@mixin tablet  { @media (min-width: map.get($breakpoints, tablet))  { @content; } }
@mixin desktop { @media (min-width: map.get($breakpoints, desktop)) { @content; } }

// Uso: el bloque {} se inyecta dentro del @media
.card {
  width: 100%;

  @include tablet { width: 50%; }
  @include desktop { width: 25%; }
}
```

> [!tip] Patrón responsivo mobile-first
> Con `min-width` los estilos base aplican siempre y **suben** de tamaño — el enfoque recomendado de [[Responsive Web Design]]. El orden de los mixins en el CSS importa: de menor a mayor breakpoint.

### `@content` con argumentos (Dart Sass ≥ 1.38)

`@content` puede recibir valores que el bloque usa como si fueran variables locales:

```scss
@mixin hover-en-foco($estado: hover) {
  @content($estado);
}

.elemento {
  @include hover-en-foco(focus) {
    // $accento está disponible acá dentro
    outline: 2px solid currentColor;
  }
}
```

En la práctica el 95% de los casos no lo necesita; `@content` simple cubre casi todo. Donde sí aporta: mixins de tema que "inyectan" el token vigente al bloque.

> [!warning] `@content` y el scope de variables
> El bloque escrito en el `@include` ve las variables del **lugar donde se escribió** (el caller), no las del mixin. Las variables internas del mixin no son visibles en `@content` salvo que se pasen como argumentos.

## Mixins condicionales (`@if` dentro del mixin)

Un mixin puede decidir **qué declaraciones emitir** según sus argumentos — es la base de los mixins "variantes":

```scss
@use 'sass:color';

@mixin boton($fondo, $variante: sólido) {
  padding: 0.75rem 1.5rem;
  border-radius: 6px;

  @if $variante == sólido {
    background: $fondo;
    color: #fff;
  } @else if $variante == contorno {
    background: transparent;
    border: 2px solid $fondo;
    color: $fondo;
  } @else if $variante == fantasma {
    background: color.mix($fondo, transparent, 15%);
    color: $fondo;
  } @else {
    @error "Variante desconocida: #{$variante}.";
  }
}

.btn-ok { @include boton(#2ecc71); }
.btn-linea { @include boton(#3498db, contorno); }
```

`@error` (ver [[Sintaxis de Sass]]) convierte al mixin en una **API con contrato**: fallar rápido en compilación es mejor que emitir CSS roto.

## Mixin vs función vs placeholder: la tabla de decisión

| Criterio | `@mixin` + `@include` | `@function` | `%placeholder` + `@extend` |
|---|---|---|---|
| Emite | Declaraciones CSS | Un **valor** (nada de CSS por sí solo) | Declaraciones (agrupadas) |
| Acepta parámetros | ✅ | ✅ | ❌ |
| `@content` | ✅ | ❌ | ❌ |
| Duplica CSS en salida | Sí, una vez por `@include` | No | **No** — agrupa selectores |
| Funciona dentro de `@media` | ✅ | ✅ (valor) | ❌ Error si el placeholder está fuera |
| Uso típico | Bloques parametrizados, media queries | Cálculos, escalas, getters de map | Clases base compartidas sin parámetros |

> [!tip] La pregunta de control
> *¿El resultado es value que se usa dentro de otra declaración?* → función. *¿Son declaraciones que puedo describir de forma parametrizable?* → mixin. *¿Son declaraciones idénticas para N selectores y sin parámetros?* → placeholder.

## Mixin vs placeholder: salida compilada real

```scss
// ---------- Mixin ----------
@mixin card-base {
  padding: 1rem;
  border-radius: 8px;
}
.card-m { @include card-base; background: #fff; }
.card-m-dark { @include card-base; background: #222; }

// Salida: declaraciones DUPLICADAS en cada uso
.card-m { padding: 1rem; border-radius: 8px; background: #fff; }
.card-m-dark { padding: 1rem; border-radius: 8px; background: #222; }

// ---------- Placeholder ----------
%p-card-base {
  padding: 1rem;
  border-radius: 8px;
}
.card-p { @extend %p-card-base; background: #fff; }
.card-p-dark { @extend %p-card-base; background: #222; }

// Salida: selectores AGRUPADOS, sin duplicar
.card-p, .card-p-dark { padding: 1rem; border-radius: 8px; }
.card-p { background: #fff; }
.card-p-dark { background: #222; }
```

| | Mixin | Placeholder |
|---|---|---|
| CSS resultante | Lineal con los usos | Constante (agrupado) |
| Parámetros | ✅ | ❌ |
| `@media` | ✅ cualquier contexto | ⚠️ solo si el placeholder vive en el mismo contexto (error `You may not @extend an outer selector from within @media`) |
| Legibilidad del output | Prefrible (cada clase autocontenida) | Depende: a veces agrupa demasiado |

Cuándo preferir **cada uno**:

- **Mixin** por defecto: es explícito, funciona en cualquier contexto, y el CSS emitido es fácil de inspeccionar. Con media queries, **siempre** mixin.
- **Placeholder**: para resets/base styles idénticos aplicados a muchos selectores donde el ahorro de bytes es real (p. ej. utilidades de reset).

## Cuándo los mixins duelen (bloat y guardas)

| Problema | Síntoma | Guarda |
|---|---|---|
| **Bloat de CSS** | Cada `@include` duplica declaraciones; 50 usos de un mixin de 12 líneas = 600 líneas repetidas | Si el mixin es grande y sin parámetros → placeholder o clase simple; si los parámetros siempre son los mismos → convertir a clase |
| **API con 8 flags** | `@include card($shadow, $border, $hover, $radius, $pad, $cursor, $transition, $theme)` — ilegible | Dividir en mixins pequeños o sistema de variantes (`@each` sobre un mapa) |
| **Específidad engañosa** | El mixin no cambia especificidad (la de la regla que lo incluye), pero anidarlo dentro de selectores compuestos la eleva | Mantener selectores planos alrededor de `@include` |
| **Acoplamiento oculto** | El mixin usa variables globales del módulo sin declararlas como params | Params explícitos + config via `@use ... with` (ver [[Separar Sass en Archivos]]) |

## Receta: sistema responsivo completo con mapa

Un sistema de breakpoints real para un proyecto (configuración central, generación con `@each`, uso con `@content`):

```scss
// _breakpoints.scss
@use 'sass:map';

// Configurable por el consumidor (ver @use ... with)
$breakpoints: (
  movil: 0,
  tablet: 768px,
  escritorio: 1024px,
  wide: 1280px,
) !default;

// Mixin "desde": @include desde(tablet) { ... } → min-width
@mixin desde($nombre) {
  @if not map.has-key($breakpoints, $nombre) {
    @error "Breakpoint desconocido: #{$nombre}.";
  }
  @media (min-width: map.get($breakpoints, $nombre)) { @content; }
}

// Mixin "hasta": max-width (¡restar 0.02px para evitar solape!)
@mixin hasta($nombre) {
  @if not map.has-key($breakpoints, $nombre) {
    @error "Breakpoint desconocido: #{$nombre}.";
  }
  $ancho: map.get($breakpoints, $nombre);
  @media (max-width: $ancho - 0.02px) { @content; }
}

// Generar clases utilitarias por breakpoint (estilo Tailwind en miniatura)
@each $nombre, $ancho in $breakpoints {
  @if $ancho > 0 {
    @media (min-width: $ancho) {
      .md-oculto-#{$nombre} { display: none; }
      .md-texto-#{$nombre} { font-size: 1.125rem; }
    }
  }
}
```

```scss
// Uso en cualquier componente
.sidebar {
  @include desde(escritorio) {
    display: flex;
    flex-direction: column;
  }
}
```

Notas del diseño:

- Los nombres de breakpoint son **validados con `@error`** (typo = falla la compilación, no CSS silenciosamente roto).
- `hasta()` resta `0.02px` porque `max-width: 768px` y `min-width: 768px` **solapan en el píxel exacto**.
- La generación por `@each` deja en el CSS final solo las utilidades usadas… no: emite **todas** (Sass no hace purging). Eso es aceptable para conjuntos chicos; para utilities grandes, Tailwind es la herramienta correcta (ver [[TailwindCSS - Sintaxis Básica]]).

## Recetas: mixins de layout (flex y grid)

```scss
// ---------- Flex ----------
@mixin flex-centrado($dir: row) {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: $dir;
}

.centrado { @include flex-centrado; }              // centro perfecto
.centrado-col { @include flex-centrado(column); }

@mixin media-object($avatar: 48px) {
  display: flex;
  gap: 1rem;
  > :first-child { flex: 0 0 $avatar; }
  > :last-child { flex: 1; min-width: 0; }        // min-width 0: ver Flexbox
}

.tweet { @include media-object; }

// ---------- Grid ----------
@mixin grid-repetido($min: 250px, $gap: 1rem) {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax($min, 1fr));
  gap: $gap;
}

.cards { @include grid-repetido; }                 // grilla responsive sin media queries

@mixin col($span, $total: 12) {
  grid-column: span $span;
}
```

Los patrones visuales detrás de estas líneas (`minmax`, `auto-fit`, `min-width: 0`) están explicados en [[Flexbox]] y [[CSS Grid]] — el mixin solo los empaqueta.

## Mixin vs CSS nativo (anidación + custom properties)

Parte de lo que los mixins resolvían ya tiene contraparte nativa:

| Necesidad | Mixin Sass | Alternativa nativa (2026) |
|---|---|---|
| Valor parametrizado | `@include card(sombra)` | Clase compuesta o custom property (`--sombra: ...`) |
| Bloque repetido sin params | `@include reset` | Clase base (`.card--base`) |
| Tema | Mixin de tema por mapa | Custom properties + selector de tema (`.tema-dark { --fondo: ... }`) |
| Media query reutilizable | `@include tablet { }` | **No tiene equivalente** (hasta que existan mixins CSS reales) |
| Lógica condicional/loops | `@if`/`@each` dentro del mixin | **No tiene equivalente** |

Conclusión: si el comportamiento se puede expresar con **clases + custom properties**, eso es más liviano y runtime-flexible. El mixin gana cuando hay **lógica** (condiciones, parámetros, generación) o media queries centralizadas.

## Convenciones de nombres y organización

- **kebab-case**: `card-hover`, `flex-centrado`, `texto-truncado`.
- Verbo/estado claro para variantes: `boton--peligro` como nombre de clase resultante, mixin `boton($variante)`.
- Los mixins viven en partials propios, cargados por módulo (ver [[Separar Sass en Archivos]]): `abstracts/_mixins.scss`, o carpetas por dominio (`components/_buttons.scss` con sus mixins internos).
- **No generes mixins que solo envuelven una propiedad** (`@mixin color($c) { color: $c; }`): eso es ruido; usá una variable.
- Mantené la API chica: mixins con 1–3 parámetros son ideales; > 5 es señal de rediseño.

## Debugging de mixins

```scss
@mixin responsive-grid($cols, $gap) {
  @debug "responsive-grid: cols=#{$cols} gap=#{$gap} en #{&}";
  @if $cols < 1 {
    @error "responsive-grid: $cols debe ser ≥ 1 (recibí #{$cols})";
  }
  @if $gap > 3rem {
    @warn "responsive-grid: gap=#{$gap} es grande; ¿ok?";
  }
  display: grid;
  grid-template-columns: repeat($cols, 1fr);
  gap: $gap;
}
```

| Directiva | Cuándo usarla en mixins |
|---|---|
| `@debug` | Ver qué valores/selector recibe el mixin en cada inclusión (se imprime solo en compilación, no afecta el CSS) |
| `@warn` | Avisar uso de defaults sospechosos o parámetros legacy sin romper |
| `@error` | Contrato inválido: parámetro fuera de rango, nombre inexistente — falla ruidosamente |

## Errores comunes

1. **Olvidar el punto y coma o las llaves en la firma** — `@mixin grid($cols) { ... }` siempre con llaves aunque sea de una línea.
2. **Esperar variables del mixin dentro de `@content`** — el bloque ve el scope del caller; pasar valores como argumentos de `@content(...)`.
3. **`@extend` en media queries donde usabas mixin** — el placeholder no cruza contextos; mantener mixins para `@media` (tabla mixin vs placeholder).
4. **Mezclar mixin con lógica de valores** — `@include calcular-ancho(3)` que retorna un valor no emite nada útil en una declaración; para cálculos usar `@function`.
5. **Bloat silencioso**: contar el CSS final (`--style=expanded` y buscar el patrón) antes de crecer con includes masivos.

## Cheatsheet

| Concepto | Sintaxis |
|---|---|
| Definir | `@mixin nombre($a, $b: 1) { ... }` |
| Incluir | `@include nombre(2, $a: 1);` |
| Contenido variable | `@content;` dentro del mixin · bloque `{}` en el include |
| Contenido con args | `@content($x)` + bloque que recibe `($x)` |
| Args variables | `@mixin f($rest...) { #{$rest}; }` |
| Condición | `@if` / `@else if` / `@else` / `@error` dentro del mixin |
| Validar args | `@error` en compile-time (contratos) |
| Debug | `@debug` / `@warn` |
| Placeholder | `%base { }` + `@extend %base` (sin params, agrupa selectores) |
| Mixin vs fn | Declaraciones → mixin · valores → función |

## Relacionadas

- [[Sintaxis de Sass]] — funciones, placeholders y el lenguaje base de los mixins
- [[Separar Sass en Archivos]] — dónde viven los mixins en un proyecto real (`abstracts/_mixins.scss`)
- [[Responsive Web Design]] — patrones mobile-first con `@content`
- [[Flexbox]] — el layout detrás de los mixins flex-centrado/media-object
- [[CSS Grid]] — los mixins de grid construidos sobre estas bases
- [[TailwindCSS - Sintaxis Básica]] — la alternativa utility-first a los mixins de layout
