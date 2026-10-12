---
tags:
  - desarrollo
  - frontend
  - sintaxis-sass
tipo: concepto
estado: terminado
---

# Sintaxis de Sass

Referencia completa del lenguaje: variables, anidación, placeholders, funciones, directivas de control, maps y catálogo de módulos incorporados. Sass existe en dos sintaxis (`.scss` y `.sass`), pero **el lenguaje es el mismo** — aquí se documenta en `.scss`, la recomendada (ver [[Qué es Sass]]).

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
```

- Las variables **se resuelven en compile-time**: el CSS de salida no las conoce (a diferencia de las custom properties del CSS — ver abajo).
- Convención: `$snake-case` para variables.
- Una variable puede referenciar a otra, y **los valores son expresiones completas** (aritmética, funciones):

```scss
$base: 1rem;
$grande: $base * 1.25;        // 1.25rem
$tema-activo: true;
```

- Las variables se **redefinen**: la última asignación "ganaba" desde el inicio del archivo (con `@use` cada módulo tiene su propio scope — ver [[Separar Sass en Archivos]]).

### Bandera `!default`

Asigna un valor **solo si la variable todavía no está definida**. Es la base de la configuración de librerías y partials reutilizables:

```scss
// _config.scss
$color-primario: #3498db !default;  // si el consumidor lo definió ANTES, se respeta
$spacing: 1rem !default;
```

```scss
// app.scss — sobreescribe ANTES de cargar el módulo
$color-primario: #e74c3c;
@use 'config';   // config.$color-primario === #e74c3c
```

Con el sistema de módulos, el patrón completo de configuración usa `@use ... with` (el consumidor declara variables y luego carga el módulo — ver [[Separar Sass en Archivos#Configuración con with]]).

### Bandera `!global`

Crea/modifica una variable **en el scope global** desde dentro de una función o mixin:

```scss
$contador: 0;
@function incrementar() {
  $contador: $contador + 1 !global;
  @return $contador;
}
```

Uso raro; si lo necesitás seguido, el diseño de la función casi siempre debe replantearse.

### Custom properties (CSS variables) vs variables Sass

```scss
$tema: dark;           // Sass: la decide el compilador, no existe en runtime
:root {
  --tema-fondo: #111;  // CSS: vive en el navegador, cambiable con JS
}
```

| | `$var Sass` | `--var` CSS |
|---|---|---|
| Se resuelve | En compile-time | En runtime (navegador) |
| Cambiar con JS | ❌ (hay que recompilar) | ✅ `element.style.setProperty()` |
| Herencia | No (scope Sass) | ✅ Sigue el árbol del DOM |
| Matemática/lógica | ✅ Completa | ⚠️ Solo `calc()`/`env()` |
| Existe en el CSS final | No (sustituida) | Sí |

Usá variables Sass para tokens de diseño fijos del sistema; custom properties cuando necesites **temas dinámicos o runtime**. Patrón híbrido habitual: compilar tokens Sass a custom properties.

## Anidación

```scss
.nav {
  display: flex;

  a {
    color: $color-primario;

    &:hover {          // & = el selector padre
      text-decoration: underline;
    }

    &.activo {         // compuesto: .nav a.activo (NO .nav a .activo)
      font-weight: bold;
    }

    .icono & {         // descendencia invertida: .icono .nav a
      margin-right: 4px;
    }
  }
}
```

### `&` avanzado

| Patrón | Salida | Caso de uso |
|---|---|---|
| `&:hover` | `.nav a:hover` | Pseudo-clases |
| `&--mod` | `.nav a--mod` | BEM: modificadores pegados al padre |
| `&__el` | `.nav a__el` | BEM: elementos |
| `.activo &` | `.activo .nav a` | Selector del "ambiente" (tema, estado de ancestro) |
| `&.activo` | `.nav a.activo` | Compuesto: mismo elemento con ambas clases |
| `&-sufijo` | `.nav a-sufijo` | Derivar variantes del nombre |
| `#{&}:not(:last-child)` | `.nav a:not(:last-child)` | Forzar el selector literal vía interpolación |
| `.a, .b { .c & }` | `.c .a, .c .b` | `&` duplica con cada selector de la lista |

> [!warning] Regla de los 3 niveles
> No nestees más de 3 niveles: la anidación profunda genera selectores largos, altamente específicos y difíciles de sobreescribir (ver [[Especificidad]] y [[Selectores en CSS]]). Si necesitás 4 niveles, casi siempre conviene una clase nueva plana.

> [!tip] Anidación de utilidad vs anidación de estructura
> Anidá **estados y variantes** (`&:hover`, `&.activo`, `&--mod`) — el `&` aporta contexto. Evitá anidar simplemente para replicar la jerarquía del HTML (`.nav ul li a`): eso duplica especificidad sin ganancia. Con Sass puro, el selector de contexto plano (`.tema-oscura .nav a`) suele ser mejor que el anidado.

## Interpolación `#{}`

Inserta expresiones Sass dentro de selectores, propiedades y valores — y **fuerza a Sass a tratar el entorno como texto crudo** (sin interpretar unidades ni comillas):

```scss
$columnas: 4;
$breakpoint: "screen and (min-width: 768px)";
$side: left;

.grid-#{$columnas} {                    // → .grid-4 (nombre dinámico)
  grid-template-columns: repeat($columnas, 1fr);
}

@media #{$breakpoint} { ... }           // media query dinámica

#{"." + $side}-icono { ... }            // → .left-icono (selector construido)

p {
  margin-#{$side}: 1rem;                // → margin-left (nombre de propiedad dinámico)
  content: "Ancho: #{$columnas}px";     // dentro de strings
}
```

Casos donde la interpolación es **obligatoria**:

- Selectores o nombres de propiedad dinámicos.
- Unir valor + unidad como texto: `$size: 12; width: #{$size}px;` (o `#{$size} * 1px`, ver matemática).
- Evitar que Sass evalúe aritmética dentro de un valor: `calc(#{$a} - #{$b})` deja la resta al navegador.
- Dentro de `@media`, `@supports`, `@keyframes`, comentarios `/*! */`.

## Operaciones y matemática

```scss
@use 'sass:math';

$gap: 1rem;
$padding: $gap * 2;        // 2rem
$ancho: 100% - 40px;       // 60% (mismo tipo de unidad compatible)
$area: math.div(10rem, 2); // 5rem
$mitad: math.percentage(0.5); // 50%
```

### La división `/` está deprecada

Desde Dart Sass 1.33 (2021), `$a / $b` **ya no es la forma correcta de dividir**: `/` se reservó como separador de la shorthand de fuentes (`font: 12px/1.5`, `border-radius: 4px/2px`) y genera deprecation warning. Estrategias de reemplazo:

| Quiero | Antes (❌ warning) | Ahora (✅) |
|---|---|---|
| Dividir valores Sass | `$mitad: $a / $b` | `math.div($a, $b)` |
| División que debe quedar en el CSS | (no existía) | `calc($a / $b)` — el navegador divide en runtime |
| Porcentaje rápida | — | `math.percentage(0.25)` → `25%` |

```scss
@use 'sass:math';
$columna: math.div(100%, 3);         // 33.333...%
.ancho { width: calc(100% - #{$gap}); }
```

### Precedencia de operadores

De mayor a menor:

| Nivel | Operadores |
|---|---|
| 1 | `()` agrupación |
| 2 | `*`, `/` (y `x` como alias de `*`) |
| 3 | `+`, `-` (también unarios) |
| 4 | `==`, `!=`, `<`, `>`, `<=`, `>=` |
| 5 | `and` |
| 6 | `or` |
| 7 | `...` rango/expansión |
| 8 | `?:` ternario (Dart Sass) e `if()` |

```scss
@use 'sass:math';
2 + 3 == 5        // true  (la suma se evalúa primero)
true or false and false  // true  (and > or)
$tema == dark and $layout == wide ? a : b;  // ternario estilo Dart
```

También existen `not`, `and`, `or` booleanos y las funciones `if(cond, a, b)`, `min()`, `max()`, `clamp()` de Sass (¡distintas de las nativas cuando se usan fuera de `calc()`!).

## Mapas

Colecciones **clave→valor**, ideales para temas, tokens y configuraciones:

```scss
@use 'sass:map';

$tema: (
  primario: #3498db,
  texto: #222,
  superficie: (
    base: #fff,
    elevada: #f5f5f5,     // maps anidados son válidos
  ),
);

.texto-primario { color: map.get($tema, primario); }
.card { background: map.get($tema, superficie, elevada); }  // get anidado con claves encadenadas
```

### API completa de `sass:map`

| Función | Qué hace | Ejemplo |
|---|---|---|
| `map.get($map, $keys...)` | Valor por clave (claves encadenadas = acceso anidado) | `map.get($m, a, b)` |
| `map.merge($m1, $m2)` | Fusiona (m2 gana); admite claves encadenadas para merge profundo | `map.merge($m, (nuevo: 1))` |
| `map.remove($m, $keys...)` | Elimina claves (silencioso si no existen) | `map.remove($m, viejo)` |
| `map.keys($map)` | Lista de claves | para iterar |
| `map.values($map)` | Lista de valores | |
| `map.zip($l1, $l2, ...)` | Combina listas → mapa | `map.zip((a b), (1 2))` → `(a: 1, b: 2)` |
| `map.has-key($m, $keys...)` | ¿Existe la clave? | guardas en `@if` |
| `map.deep-get` / merge profundo | Acceso/merge encadenado vía claves múltiples | `map.get($m, a, b, c)` |
| `map.empty?($map)` | ¿Mapa vacío? | |
| Acceso profundo | No hay `deep-get`; usá claves encadenadas | `map.get($m, a, b, c)` |

```scss
@use 'sass:map';
@use 'sass:list';

$breakpoints: (movil: 480px, tablet: 768px, escritorio: 1024px);

// Iteración completa
@each $nombre, $ancho in $breakpoints {
  .is-#{$nombre} { max-width: $ancho; }
}

// Guardas
@if map.has-key($breakpoints, tv) { /* ... */ }

// Sintaxis alternativa: "clave valor" sin paréntesis anidados (map literal con pares)
$colores: primario: #3498db, error: #e74c3c;   // válido como map
```

## Directivas de control

### `@if` / `@else if` / `@else`

```scss
@use 'sass:color';

$tema: dark;

@if $tema == dark {
  body { background: #111; color: #eee; }
} @else if $tema == light {
  body { background: #fff; }
} @else {
  @error "Tema desconocido: #{$tema}.";
}
```

### `@for` — bucles con rango

```scss
@use 'sass:math';

@for $i from 1 through 4 {               // 1,2,3,4 (INCLUSIVO)
  .columna-#{$i} { width: math.div(100%, $i); }
}

@for $i from 1 to 4 {                    // 1,2,3 (EXCLUSIVO: "to" no incluye el final)
  .margen-#{$i} { margin: $i * 0.25rem; }
}
```

| Rango | Incluye el extremo | Salida para 1..4 |
|---|---|---|
| `through` | ✅ Sí | 1, 2, 3, 4 |
| `to` | ❌ No | 1, 2, 3 |

### `@each` — iterar listas y maps (con destructuring)

```scss
@use 'sass:color';

// Lista simple
@each $color in rojo, verde, azul {
  .texto-#{$color} { color: $color; }
}

// Mapa con destructuring clave, valor
$alertas: (info: #3498db, ok: #2ecc71, error: #e74c3c);

@each $nombre, $hex in $alertas {
  .alerta-#{$nombre} {
    color: $hex;
    background: color.mix($hex, #fff, 90%);
  }
}

// Destructuring anidado: pares de valores
@each $par in ((a, 1), (b, 2)) {
  // $par = (a, 1); extraer con list.nth($par, 1)
}

// Destructuring directo de listas anidadas
@each $prop, $size, $color in ((margin, 1rem, #333), (padding, 2rem, #666)) {
  .caja-#{$prop} { #{$prop}: $size; border-color: $color; }
}
```

### `@while` — con extrema precaución

```scss
@use 'sass:math';

$x: 100;
@while $x > 0 {
  .w-#{$x} { width: math.percentage(math.div($x, 100)); }
  $x: $x - 10;   // ← SIN ESTA LÍNEA: bucle infinito que cuelga la compilación
}
```

> [!warning] `@while` cuelga compilaciones
> Sin una condición de salida garantizada, Dart Sass queda en bucle infinito (hay que matar el proceso). `@for` (rango conocido) o `@each` (colección conocida) casi siempre son la opción correcta. Reservá `@while` para patrones matemáticos de convergencia (p. ej. generar escala hasta superar un umbral).

## Funciones

Devuelven un **valor**; los mixins devuelven **estilos** (ver [[Mixins]]):

```scss
@use 'sass:math';
@use 'sass:color';

@function ancho-col($cols, $gap: 1rem) {
  @return math.div(100% - ($cols - 1) * $gap, $cols);
}

.tres-col {
  width: ancho-col(3);        // cálculo reutilizable
}
```

### Modos de argumentos

```scss
@use 'sass:list';
@use 'sass:meta';

// 1. Posicional: f(1, 2) — en orden
// 2. Nombrado: f($b: 2) — en cualquier orden
// 3. Con default: f($b: 5) — opcional
// 4. Argumentos variables (...): resto en lista + keywords
@function escala($base, $factor: 1.5, $extras...) {
  $extra-lista: $extras;                 // lista de posicionales extra
  $extra-kw: meta.keywords($extras);     // mapa de nombrados extra
  @return $base * $factor;
}

// Múltiples valores de retorno (es una lista por coma)
@use 'sass:list';
@function xy($v) { @return $v, $v * 2; }
$res: xy(1rem);
$ancho: list.nth($res, 1);   // 1rem
$alto: list.nth($res, 2);    // 2rem
// (la destructuring con coma existe solo en @each, no en asignación)
```

| Modo | Ejemplo de llamada | Notas |
|---|---|---|
| Posicional | `ancho-col(3, 2rem)` | Respeta el orden de la firma |
| Nombrado | `ancho-col($gap: 2rem, $cols: 3)` | Cualquier orden; legible en firmas largas |
| Default | `ancho-col(3)` | El `$gap` usa su default |
| Variables `...` | `escala(1rem, 2, 3, $x: 4)` | `$extras` = lista `(3)` + keywords `(x: 4)` |

> [!tip] Función devolviendo varios valores
> `@return $x, $y` devuelve una **lista por coma**. Se recibe con destructuring (`$a, $b = f()`) o `list.nth(f(), 1)`. Es el mecanismo de "múltiples return values" de Sass.

### Errores dentro de funciones: `@error`, `@warn`, `@debug`

```scss
@function tono($hex) {
  @if not $hex {
    @error "tono() requiere un color, recibí: #{$hex}.";
  }
  @return color.mix($hex, #000, 20%);
}

// @warn: aviso no fatal (deprecation notices de Sass usan esto)
@warn "El breakpoint #{$bp} es inusualmente chico.";

// @debug: imprime valor en la consola de compilación (no sale al CSS)
@debug map.get($tema, primario);
```

| Directiva | Severidad | Efecto |
|---|---|---|
| `@debug` | Info | Imprime en consola; nunca al CSS |
| `@warn` | Warning | Imprime con estilo de advertencia; compilación continúa |
| `@error` | Fatal | **Aborta la compilación** con el mensaje — para contratos inválidos |

## Catálogo de módulos incorporados (`sass:*`)

Desde `@use` (ver [[Separar Sass en Archivos]]), las funciones globales deprecadas viven en módulos oficiales:

### `sass:math`

| Función | Qué hace |
|---|---|
| `math.div($a, $b)` | División de compile-time (reemplaza `/`) |
| `math.percentage($n)` | Número → porcentaje (`0.5` → `50%`) |
| `math.round/ceil/floor/abs($n)` | Redondeos y valor absoluto |
| `math.min($a, $b...)` / `math.max(...)` | Mínimo/máximo (Sass, no las nativas) |
| `math.clamp($min, $n, $max)` | Apretar entre límites (Sass) |
| `math.pow($n, $exp)` / `math.log` / `math.sqrt` | Potencia y logaritmos |
| `math.random()` | Número pseudoaleatorio |
| `math.unit($n)` / `math.is-unitless($n)` | Inspección de unidades |
| `math.compatible($a, $b)` | ¿Se pueden operar sus unidades? |

### `sass:color`

| Función | Qué hace |
|---|---|
| `color.adjust($c, $lightness: -10%, ...)` | Ajuste **absolutivo** (suma/resta: -10% resta) — reemplaza a `darken`/`lighten` |
| `color.change($c, $lightness: 40%)` | **Fija** valores absolutos (40% = 40%, no +40) |
| `color.scale($c, $lightness: 10%)` | Escala **proporcional** entre 0 y 100% (para ramps) |
| `color.mix($c1, $c2, $weight: 50%)` | Mezcla de dos colores |
| `color.to-space($c, oklch)` | Convierte a un espacio de color (hsl, lab, oklch…) |
| `color.from-space(oklch, ...)` | Construye un color desde un espacio |
| `color.hue/saturation/lightness($c)` | Canales individuales |
| `color.invert`, `complement`, `grayscale` | Operaciones clásicas |

### `sass:map`, `sass:list`, `sass:string`, `sass:selector`, `sass:meta`

| Módulo | Funciones principales |
|---|---|
| `sass:map` | `get`, `merge`, `remove`, `keys`, `values`, `zip`, `has-key` (ver sección de maps) |
| `sass:list` | `length`, `nth`, `set-nth`, `join`, `append`, `zip`, `index`, `separator`, `is-bracketed` |
| `sass:string` | `length`, `insert`, `slice`, `index`, `to-upper-case`, `to-lower-case`, `unique-id` |
| `sass:selector` | `nest`, `append`, `unify`, `is-superselector`, `parse`, `replace`, `extend`, `simple-selectors` |
| `sass:meta` | `type-of`, `inspect`, `call`, `get-function`, `keywords`, `length` (args), `module-variables`, `module-functions`, `calc-name` |

> [!warning] Nada de funciones globales en código nuevo
> `darken()`, `lighten()`, `map-get()`, `map-merge()`, `percentage()`, `unquote()`… están **deprecadas** (warnings desde 1.79–1.80; eliminadas en Sass 3.0). Equivalentes: `color.adjust`, `color.change`, `map.get`, `math.percentage`, `string.unquote`. Tabla de mapeo completo en [[Qué es Sass]].

## Placeholders `%` con `@extend`

Definen estilos que **nunca se emiten solos** — solo existen cuando otra clase los extiende (detalle y comparación con mixins en [[Mixins]]):

```scss
%card-base {
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.card { @extend %card-base; background: white; }
.card-oscura { @extend %card-base; background: #222; }
// CSS: .card, .card-oscura { padding: 1rem; ... }  (sin duplicar)
```

### Mecánica de `@extend` (y por qué revienta con media queries)

`@extend` no "pega" declaraciones: le pide al compilador que **unifique selectores** — el placeholder pasa a listar también a los selectores que lo extenden, en el lugar donde el placeholder fue definido. Consecuencias:

1. **No funciona entre contextos de media query distintos**: definir `%base` fuera de un `@media` e `@extend`arlo dentro lanza el error `You may not @extend an outer selector from within @media`. El motivo es estructural: Sass no puede mover declaraciones a un contexto de medios que no coincide con el del placeholder (produciría CSS inválido o duplicado). Solución: definir el placeholder **dentro del mismo media query**, o usar mixin:

```scss
// ❌ Error de compilación
%base { color: red; }
@media (min-width: 768px) { .nav { @extend %base; } }

// ✅ Placeholder duplicado en cada contexto, o mejor:
@mixin base { color: red; }
@media (min-width: 768px) { .nav { @include base; } }
```

2. **Extender selectores compuestos es frágil**: `@extend .a.b` unifica contra el par completo — con pocos usos produce selectores monstruosos. Regla: extendé **placeholders** o clases simples.
3. **El orden final depende de dónde esté el placeholder**: el CSS agrupado aparece en la posición de la definición, no del `@extend`.

## Errores comunes

1. **`$a / $b` con warning** → deprecación de división; `math.div()` o `calc()`.
2. **Usar funciones globales** (`darken`, `map-get`) → warnings y futuro rompimiento; módulos `sass:*`.
3. **`You may not @extend an outer selector from within @media`** → placeholder fuera del contexto de medios; usar mixin o redefinir dentro del `@media`.
4. **Mapa "vacío" por get con clave inexistente** → `map.get` retorna `null` en silencio; guardá con `map.has-key`.
5. **Ternario/condicional con asignación** → en Sass no hay `=` de asignación dentro de expresiones: la asignación es su propia declaración `$x: valor;`.
6. **`@while` sin decrecimiento** → compilación colgada.
7. **Anidamiento > 3 niveles** → selectores gigantes con especificidad alta (ver [[Especificidad]]).

## Cheatsheet

| Tema | Referencia rápida |
|---|---|
| Variable + default | `$x: 1rem !default;` |
| Variable global | `$x: 1 !global;` |
| Referencia padre | `&` (compuesto, invertido, `&--mod`) |
| Interpolación | `#{expr}` — selectores, props, media queries |
| División | `math.div(a, b)` (nunca `/`) |
| If | `@if` / `@else if` / `@else` |
| For inclusivo | `@for $i from 1 through N` |
| For exclusivo | `@for $i from 1 to N` |
| Each + map | `@each $k, $v in $map` |
| While | `@while cond` (¡con salida garantizada!) |
| Map get/merge | `map.get`, `map.merge`, `map.remove`, `map.keys`, `map.values` |
| Function | `@function f($a, $b: 1, $rest...) { @return ...; }` |
| Errores | `@debug` / `@warn` / `@error` |
| Módulos | `@use 'sass:math'`, `sass:color`, `sass:map`, `sass:list`, `sass:string`, `sass:selector`, `sass:meta` |

## Relacionadas

- [[Qué es Sass]] — qué es el preprocesador y cómo se compila
- [[Mixins]] — reutilización con parámetros y `@content`
- [[Separar Sass en Archivos]] — organización en módulos `@use`/`@forward`
- [[Especificidad]] — por qué la anidación profunda se sale de control
- [[Selectores en CSS]] — los selectores que la anidación de Sass termina generando
- [[CSS Grid]] — patrones de grid construidos con mixins (ver [[Mixins]])
