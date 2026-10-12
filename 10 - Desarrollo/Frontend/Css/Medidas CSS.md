---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Medidas CSS

CSS tiene dos familias de medidas: **absolutas** (`px`, `pt`...) y **relativas** (`em`, `rem`, `%`, `vw`...). Las absolutas son fijas; las relativas se calculan en función de algo — por eso son las que permiten diseños adaptables y accesibles.

> [!info] El modelo mental: todo se resuelve a píxel
> Al final del layout, **todas** las medidas se convierten en píxeles de dispositivo CSS. La única pregunta que hay que hacerse ante cualquier unidad es: **¿relativa a QUÉ?** — la raíz (`rem`), la fuente del propio elemento (`em`), el contenedor (`%`), la ventana (unidades `v*`, ver [[Medidas Relativas]]) o un contenedor de consulta (`cqw`, ver [[Medidas Relativas]]). Con ese filtro, la tabla completa de unidades deja de ser memoria y pasa a ser lógica.

Mapa maestro de las unidades de longitud:

| Grupo | Unidades | Relativa a |
|---|---|---|
| Absolutas | `px` `pt` `pc` `in` `cm` `mm` `Q` | Nada (fijas por definición de CSS) |
| Resolución | `dppx` `dpi` `dpcm` | Densidad de píxeles del dispositivo (solo media queries / `image-set`) |
| Fuente (del elemento) | `em` `ex` `ch` `cap` `ic` `lh` | La fuente del propio elemento (o del padre para `font-size`) |
| Fuente (raíz) | `rem` `rlh` | La fuente del elemento raíz (`html`) |
| Viewport | `vw` `vh` `vmin` `vmax` `sv*` `lv*` `dv*` `vi` `vb` | La ventana — ver [[Medidas Relativas]] |
| Contenedor | `cqw` `cqh` `cqi` `cqb` `cqmin` `cqmax` | El contenedor de consulta — ver [[Medidas Relativas]] |
| Porcentaje | `%` | El eje correspondiente del contenedor |

## Unidades absolutas: `px`

El píxel es la unidad más fácil e intuitiva: 1px es siempre 1px (en CSS, un píxel lógico). Mide bien para lo **pequeño y fijo**: bordes, sombras, radios, iconos.

```css
.boton {
  border: 1px solid #333;   /* borde fino: px está bien */
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}
```

**El píxel CSS no es el píxel físico.** En una pantalla retina, 1 CSS px = 2×2 (o 3×3) píxeles físicos; la relación se mide en `dppx` (device pixel per CSS pixel), donde `1dppx = 96dpi`. CSS trabaja siempre en píxeles lógicos — por eso un borde de 1px se ve fino en todos los dispositivos. Las unidades de resolución (`dppx`, `dpi`, `dpcm`) solo aparecen en media queries (`resolution`) y en `image-set` (ver [[Imágenes CSS]]).

### Las otras absolutas: unidades de impresión

| Unidad | Equivalencia exacta | Dónde se usa de verdad |
|---|---|---|
| `in` | 1in = 96px | Print CSS (márgenes físicos) |
| `cm` | 1cm = 96/2.54 ≈ 37.8px | Print, documentos oficiales |
| `mm` | 1mm = 96/25.4 ≈ 3.78px | Print |
| `Q` | 1Q = 1/4 mm ≈ 0.945px | Print (cuartos de milímetro, ISO) |
| `pt` | 1pt = 1/72 in = 96/72 ≈ 1.333px | Tipografía de imprenta; `font-size` en `@media print` |
| `pc` | 1pc = 12pt = 16px | Herencia tipográfica antigua (picas) |

> [!warning] El problema de los px para tipografía y espaciado
> Si todo está en px, la web **no respeta la configuración de tamaño de fuente del usuario** ni escala con el zoom: mal para accesibilidad (WCAG) y mal para adaptar el diseño. Para tamaños de texto y espacios usá unidades relativas.

`cm`/`in` en pantalla **no** miden centímetros reales: el navegador los convierte a px con la misma regla de 96dpi. Usálos solo donde el medio sea físico (impresión) o el valor deba declararse en esa unidad (papeles, folletos).

## Unidades relativas: `em` y `rem`

| Unidad  | Relativa a                                        | Comportamiento                                            |
| ------- | ------------------------------------------------- | --------------------------------------------------------- |
| `em`    | El `font-size` **del propio elemento** (o del padre al heredarse) | Se acumula: anidado, 1em no es el mismo valor en dos niveles |
| `rem`   | El `font-size` del **elemento raíz** (`html`)    | Siempre el mismo valor en todo el documento               |

- **`em`**: ideal para componentes autocontenido (un padding que crece con el texto de su tarjeta), pero su acumulación lo hace impredecible en layouts grandes.
- **`rem`**: unidad recomendada para layout y tipografía: sin sorpresas anidadas y respeta el zoom/ tamaño de fuente del usuario (bueno para accesibilidad).

### Las dos caras de `em` (el matiz que rompe cabezas)

`em` significa cosas **distintas** según la propiedad a la que se aplica:

| Propiedad | `em` es relativa a |
|---|---|
| `font-size` | El `font-size` **calculado del padre** (por eso compone) |
| Todo lo demás (`padding`, `margin`, `border-radius`, `width`…) | El `font-size` **del propio elemento** (ya resuelto) |

```css
.tarjeta {
  font-size: 20px;
  padding: 1em;            /* 20px — el de la tarjeta */
}

.tarjeta > .titulo {
  font-size: 1.5em;        /* 30px = 1.5 × 20px (fuente del PADRE) */
  padding: 1em;            /* 30px — el propio ya-resuelto, no 20 */
}

.tarjeta > .titulo > .icono {
  font-size: 1.5em;        /* 45px = 1.5 × 30px — ¡se acumuló! */
}
```

`font-size` en `em` mira hacia **arriba** (el padre) y por eso se encadena; el resto de propiedades mira **hacia su propio resultado** y no acumula. Regla práctica: `em` en `font-size` = peligro de composición; `em` en todo lo demás = seguro y local.

> [!info] La excepción donde `em` gana
> Para bordes y radios dentro de componentes escalables, `em` es mejor que `rem`: si el usuario amplía el texto de la tarjeta, el borde crece proporcionalmente.

```css
.boton {
  font-size: 1rem;
  padding: 0.5em 1em;
  border-radius: 0.25em;
  border: 0.08em solid currentColor; /* el borde escala con el texto del botón */
}
```

### El truco del 62.5%: 1rem = 10px

Como `rem` es relativo al `font-size` de `html` (por defecto 16px), cambiarlo a `62.5%` (= 10px) hace que **1rem = 10px**, y los cálculos se vuelven triviales: un padding de 24px es `2.4rem`, un margen de 40px es `4rem`:

```css
html {
  box-sizing: border-box;
  font-size: 62.5%; /* 1rem = 10px */
}

*,
*::before,
*::after {
  box-sizing: inherit; /* hereda el border-box — ver [[Box Model]] */
}

body {
  font-size: 1.6rem; /* 16px: tamaño base de lectura "normal" */
}
```

Dos detalles de este setup:

- El `box-sizing: inherit` con `*` propaga el `border-box` a todo el árbol sin romper la herencia — patrón recomendado en [[Box Model]].
- `body` vuelve a `1.6rem` para que el texto de lectura siga midiendo los 16px reales.

> [!warning] Conocé el costo del 62.5%
> 1. `rem` deja de ser "el tamaño que eligió el usuario": si alguien configura fuentes grandes en el navegador, 1rem ya no sigue la escala original del navegador, solo la proporción. 2. Genera conversión mental constante entre px y rem. Es un trade-off válido y muy usado, pero no es gratis: muchos equipos hoy prefieren dejar `html` en 16px y usar valores "crudos" (`1.5rem`, `2.25rem`).

El `%` en `font-size` de `html` es relativo al **default del agente** (16px), no al del padre — por eso `62.5%` siempre significa 10px.

## Unidades de fuente: `ex`, `ch` y compañía

| Unidad | Mide | Uso real |
|---|---|---|
| `ex` | La altura de la "x" minúscula de la fuente actual | Ajustes finos de interlineado (legado) |
| `ch` | El ancho del carácter **"0"** de la fuente actual | Medidas de texto: `max-width: 65ch` |
| `cap` | La altura de mayúscula (cap height) | Alineación tipográfica fina (2023+) |
| `ic` | El ancho de un ideograma CJK (水) | Layouts de texto este-asiático |
| `lh` / `rlh` | Altura de línea del elemento / de la raíz | Espaciado basado en la caja de línea |

```css
.article p {
  max-width: 65ch; /* ~60-75 caracteres por línea: zona cómoda de lectura */
}

input {
  width: 20ch; /* cabe el texto de 20 caracteres "0" */
}
```

> [!tip] `ch` en fuentes proporcionales
> En una fuente normal, la "0" es más ancha que la letra promedio: `65ch` da líneas de ~70–75 caracteres reales — justo la zona cómoda de lectura. En monoespaciada, `ch` es exactamente el ancho de columna. `ch` cambia con la fuente y con `font-size`, ideal para measure tipográfico responsive.

## Porcentajes: qué es el "contenedor" en cada propiedad

`%` siempre resuelve contra algo **del contexto de la propiedad**, y ahí están las trampas:

| Propiedad | `%` relativo a | Trampa típica |
|---|---|---|
| `width` | Ancho del contenedor | — |
| `height` | Alto del contenedor | Padre sin `height` → el % no tiene referencia útil |
| `margin` / `padding` (cualquier eje) | **Siempre el ancho** del contenedor | Padding vertical en % mide horizontalmente |
| `top/right/bottom/left` | Contenedor de posicionamiento (ver [[Tipos de Posiciones]]) | — |
| `font-size` | `font-size` del **padre** | — |
| `line-height` | `font-size` **propio** | — |
| `border-radius` | lado horizontal / vertical respectivo | — |

```css
.hijo {
  width: 50%;        /* mitad del padre */
  padding: 5%;       /* 5% del ANCHO del padre en los 4 lados (¡incluso el vertical!) */
  margin-top: 10%;   /* también mide el ancho del padre — sorpresa clásica */
}

.padre-alto {
  height: 300px;
}
.hijo-50 {
  height: 50%;       /* 150px — solo porque el padre tiene altura definida */
}
```

El `%` de `height` hereda el problema del auto: si el contenedor no tiene altura explícita, el 50% no puede resolver — combinar con `height`/`min-height` del padre o usar `aspect-ratio` (ver recetas).

> [!warning] El margen en % es horizontal, siempre
> `margin-top: 10%` **nunca** es 10% del alto: es 10% del ancho del contenedor. Es la trampa con más víctimas de las medidas, directamente del comportamiento de las cajas (ver [[Box Model]]).

## Funciones matemáticas: `calc()`, `min()`, `max()`, `clamp()`

CSS puede calcular valores en tiempo de computación — la base del diseño fluido:

### `calc()`: mezclar unidades

```css
.contenido {
  width: calc(100% - 2rem);   /* ancho del contenedor menos padding */
  margin-inline: auto;
}

.titulo {
  font-size: calc(1rem + 0.5vw); /* crece lineal con la ventana */
}

.zona-texto {
  width: calc(100% - var(--sidebar) - 2 * var(--gap));
}
```

Reglas de `calc()`:

- `+` y `-` **requieren espacios** alrededor: `calc(100%-2rem)` es inválido (se lee como identificador).
- `*` y `/` operan contra números: `calc(100px * 2)` vale, `calc(100px * 2px)` no.
- Se anida: `calc(100% - calc(2rem + 10px))`, y admite `var()` completo (¡con la trampa de las unit-less custom properties: `--gap: 8` no sirve en `calc(100% - var(--gap))`; guardá `--gap: 8px`, o usá `calc(var(--gap) * 1px)`).
- Porcentajes dentro de `calc` se resuelven igual que fuera (contra el contexto de la propiedad).

### `min()` / `max()`: el mínimo o el máximo de un conjunto

```css
.texto {
  max-width: min(65ch, 100% - 2rem); /* 65ch, pero nunca más que el contenedor */
}

.gap-responsive {
  gap: max(1rem, 2vw);      /* respira con la pantalla, con piso de 1rem */
}

.titulo {
  font-size: max(1.5rem, 2vw); /* nunca menos de 1.5rem */
}
```

### `clamp()`: el rango con valor preferido

`clamp(MÍN, PREFERIDO, MÁX)` = `max(MÍN, min(PREFERIDO, MÁX))` — ni más pequeño que el mínimo, ni más grande que el máximo:

```css
h1 {
  font-size: clamp(2rem, 5vw + 1rem, 4rem);
  /*             mín   preferido (fluido)  máx */
}

.seccion {
  padding-block: clamp(2rem, 6vw, 6rem);
}
```

Las cuatro funciones son **intercambiables**: `clamp()` se escribe con `min()`/`max()` anidados y viceversa. Todo lo fluido que dependía de media queries ahora es una línea — ver [[Responsive Web Design]] y las recetas de pendiente en [[Medidas Relativas]].

> [!tip] Las tres funciones también comparan
> `min()`/`max()` no solo sirven para elegir valor: son el `if` del CSS. `width: min(100%, 600px)` es "600px, salvo que no quepa". Combinadas con `calc()` implícito (admiten matemática sin `calc`), reemplazan decenas de media queries.

## Recetas

### Caja que nunca desborda, con respiración

```css
.tarjeta {
  width: min(100%, 420px);   /* crece hasta 420px, nunca más que el padre */
  padding: clamp(1rem, 3vw, 2rem);
}
```

### Paddings fluidos sin breakpoints

```css
.pagina {
  --gap: clamp(1rem, 4vw, 3.5rem);
  display: grid;
  gap: var(--gap);
  padding: var(--gap);
}
```

### Altura proporcional sin media queries (pre-`aspect-ratio`)

```css
.medio {
  height: 0;
  padding-bottom: 56.25%;        /* 16:9 = 9/16 */
  position: relative;
}
.medio > * {
  position: absolute;
  inset: 0;
}
/* Hoy: aspect-ratio: 16 / 9 directamente */
```

### Setup 62.5% completo (ver sección arriba)

El patrón con `box-sizing` heredado de [[Box Model]] es el setup canónico de proyectos en Sass: tokens en `rem` derivados de px, `body` en 1.6rem.

## Accesibilidad

- **`font-size` en px rompe el ajuste de texto del usuario** (WCAG 1.4.4 Resize Text): en `rem`/`em` el navegador lo escala; en px no. Excepción legítima: `border: 1px` y sombras — allí px es correcto.
- El **zoom de página** (200–400%) siempre funciona; el **zoom de texto** solo con unidades relativas. Verificá con ambos.
- Medidas en `%`/`vh` dentro de alturas fijas pueden **cortar** contenido ampliado: preferir `min-height` a `height`.
- Unidades de fuente (`ch`, `em`) se adaptan cuando el usuario cambia la fuente — comportamiento deseable, no bug.

## Rendimiento

No hay diferencia de performance entre unidades: el navegador resuelve todas en layout time y el costo es despreciable. La única relación indirecta: las funciones `clamp/min/max` con unidades de viewport pueden cambiar el layout al redimensionar (una media query haría lo mismo). Nada que optimizar; lo que importa es no **animar** tamaños (layout cada frame — ver [[Animaciones]]).

## Errores comunes

1. **`em` acumulándose en `font-size` anidado**: 1.5em³ = 3.4× — para tipografía global, `rem`.
2. **`margin-top` en % esperando el alto**: mide el ancho del contenedor.
3. **`calc(100%-2rem)` sin espacios**: sintaxis inválida, la declaración se descarta.
4. **Custom property sin unidad** (`--gap: 8`) usada en `calc(100% - var(--gap))`: inválido — guardar `"8px"` o multiplicar.
5. **`height: 50%` sobre padre auto**: no resuelve; usar `min-height` o `aspect-ratio`.
6. **`pt` en pantalla**: 1pt = 1.33px fijo, no escala con nada — solo print.
7. **Todo en px = diseño inaccesible** con texto ampliado (ver accesibilidad).
8. **`clamp` sin unidad preferida acotada**: `clamp(2rem, 5vw, 4rem)` está bien, pero sin `+` fijo la pendiente puede ser demasiado empinada en móviles — ver la fórmula de pendiente en [[Medidas Relativas]].

## Cheatsheet — unidades y funciones

| Unidad | Relativa a               | Uso recomendado                          |
| ------ | ------------------------ | ---------------------------------------- |
| `px`   | Nada (absoluta)          | Bordes, sombras, radios, iconos          |
| `pt`/`in`/`cm`/`mm`/`Q`/`pc` | Nada (impresión) | Print CSS, tipografía de imprenta |
| `em`   | Fuente del elemento (padre en `font-size`) | Componentes escalables, tamaños locales  |
| `rem`  | Fuente raíz (`html`)     | Tipografía y espaciado de layout         |
| `ex`   | Altura de "x"            | Interlineado fino (legado)               |
| `ch`   | Ancho de "0"             | `max-width` de párrafo, anchos de input  |
| `cap`/`ic`/`lh`/`rlh` | métricas de fuente | Alineación tipográfica moderna    |
| `%`    | Eje del contenedor (ancho, en margins/paddings) | Anchos y alturas de caja              |
| `dppx`/`dpi` | Dispositivo        | Media queries de resolución, `image-set` |
| `vw/vh`… | Viewport              | Ver [[Medidas Relativas]]                |
| `cqw`…  | Contenedor de consulta | Ver [[Medidas Relativas]]               |

`calc()`, `min()`, `max()` y `clamp()` no son unidades: son **operadores** que combinan cualquier valor de la tabla — el lenguaje con el que las unidades relativas se vuelven diseño fluido. Para diseños fluidos modernos combiná estas unidades con `clamp()` — ver [[Responsive Web Design]].

## Compatibilidad

Todas las unidades de esta nota son viejas y universales. Excepciones modernas: `cap`/`ic`/`lh`/`rlh` (Chrome 123+, Firefox 133+, Safari 17.4+, 2024), `font-size-adjust` con sintaxis de número (2023–24). `calc/min/max/clamp` son universales desde 2020; `clamp` pide `@supports` solo si hay que aportar fallback explícito para navegadores pre-2020 (prácticamente inexistentes en 2026).

## Relacionadas

- [[Medidas Relativas]] — `%`, `vh`, `vw`, unidades de viewport y de contenedor
- [[Box Model]] — por qué `border-box` cambia cómo se calcula el tamaño
- [[Responsive Web Design]] — dónde entra cada unidad en un diseño adaptable
- [[Tipos de Posiciones]] — el contenedor de posicionamiento contra el que resuelven los `%` de `top/left`
- [[Fuentes Externas]] — métricas de fuente y por qué el fallback debe medir igual
- [[CSS Grid]] — `fr`, `minmax()` y `gap` en el sistema de unidades de layout
- [[Animaciones]] — por qué animar tamaños (cualquier unidad) es caro
