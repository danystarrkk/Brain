---
tags:
  - desarrollo
  - frontend
  - css-grid
tipo: concepto
estado: terminado
---

# CSS Grid

**CSS Grid** (*Grid Layout*) es el modelo de layout **bidimensional** de CSS: define a la vez filas **y** columnas, y coloca el contenido dentro de esa retícula. Mientras [[Flexbox]] organiza en **una dimensión** (fila *o* columna), Grid decide el **esqueleto completo**: dónde va cada zona de la página y cuánto ocupa en ambos ejes a la vez.

En cierta forma es como una tabla de HTML, pero sin sus restricciones: las pistas pueden medir cualquier cosa (`fr`, `minmax()`, `min-content`…), los elementos pueden superponerse, cruzar varias pistas y reordenarse, y nada obliga a que exista una celda por item.

![[Pasted image 20250124170628.png]]

> [!info] Por qué existe Grid
> Antes de 2017 los layouts de página se hackeaban con `float` + porcentajes + media queries, o con tablas. Grid nació para un caso concreto que Flexbox no resuelve bien: **alinear elementos entre sí en los dos ejes** (que la columna 2 de la fila 1 empiece exactamente donde la columna 2 de la fila 3). Ninguno de los dos modelos es "mejor": se complementan — ver [[Cuando usar GRID o Flexbox]].

## Modelo conceptual: contenedores, pistas, líneas y áreas

| Término | Significado |
|---|---|
| **Grid container** | Elemento con `display: grid` o `display: inline-grid` |
| **Grid items** | Hijos **directos** del contenedor |
| **Pista** (*track*) | Cada columna o fila de la retícula |
| **Línea** (*line*) | Borde de una pista, **numerada desde 1**; la última línea tiene índice `-1` |
| **Área** (*area*) | Rectángulo formado por una o más pistas contiguas |
| **Gutter** | Hueco entre pistas — lo llenan `row-gap` / `column-gap` |
| **Grid explícito** | Lo que definís con `grid-template-columns/rows` |
| **Grid implícito** | Filas/columnas que el navegador **crea solo** cuando el contenido no cabe en el explícito |
| **`fr`** | Unidad de "fracción del espacio libre" del contenedor |

> [!tip] La pregunta que separa Grid de Flexbox
> Con Flexbox preguntás *"¿cómo se reparte el espacio entre estos items?"*. Con Grid preguntás *"¿cuáles son las columnas y filas del diseño?"*. En Grid **el contenedor define la retícula y los items se colocan en ella**; en Flexbox el contenedor define la dirección y los items negocian su tamaño.

## Activar el modelo

```css
.grid {
  display: grid;          /* bloque, ocupa su línea */
  /* display: inline-grid;  mismo modelo, fluye como inline */
  grid-template-columns: repeat(3, 1fr); /* 3 columnas iguales */
  grid-template-rows: 200px 1fr;         /* 2 filas: una fija, una flexible */
  gap: 2rem;
}
```

- Las columnas se crean con `grid-template-columns`, las filas con `grid-template-rows`.
- Las **líneas** se numeran desde 1: la columna 2 vive entre la línea 2 y la 3. La línea `-1` es la última (el borde final del contenedor).
- `1fr` es una fracción del **espacio libre** — ver [[Medidas Relativas]] para unidades relativas.

### Contenedor vs item

| Propiedades de **contenedor** | Propiedades de **item** |
|---|---|
| `display`, `grid-template-columns/rows/areas`, `grid-auto-rows/columns/flow` | `grid-column`, `grid-row`, `grid-area` |
| `justify-items`, `align-items`, `justify-content`, `align-content` | `justify-self`, `align-self` |
| `gap`, `row-gap`, `column-gap` | `min-width/height`, `max-width/height` (tamaño mínimo de contenido) |

## Pistas: cómo se dimensionan las columnas y filas

`grid-template-columns` acepta cualquier longitud, pero el arsenal propio de Grid es este:

| Valor | Significado | Uso típico |
|---|---|---|
| `200px` / `12rem` / `30%` | Tamaño fijo o relativo al contenedor | Sidebars, headers |
| `1fr` | Reparte el **espacio libre** proporcionalmente | Columnas fluidas |
| `auto` | Según el contenido (min-content a max-content) | Columnas de texto |
| `min-content` | La mínima anchura sin desbordar (la palabra más larga) | Trucar contenido |
| `max-content` | La máxima anchura deseada (una sola línea larga) | Medir sin wrap |
| `minmax(min, max)` | Combinación: crece entre dos límites | `minmax(200px, 1fr)` |
| `fit-content(300px)` | Como `auto` pero techo de 300px | Títulos que no estiran |

### La mecánica de `fr` (y su trampa de min-content)

`fr` = *fraction of the free space*. El navegador:

1. Suma las pistas **fijas** y los **gaps**.
2. Lo que sobra del contenedor es el "espacio libre".
3. Reparte ese espacio entre las pistas `fr` en proporción a su factor: `1fr 2fr` → 1/3 y 2/3.

```css
.plantilla {
  grid-template-columns: 240px 1fr 1fr;  /* sidebar fijo + 2 columnas iguales */
}
```

> [!warning] `1fr` no se encoge debajo de su contenido
> Una pista `fr` tiene mínimo **automático = min-content**: si el texto largo de una celda no cabe, la pista **se agranda** y fuerza scroll horizontal en vez de cortar. Fórmula de emergencia: `minmax(0, 1fr)` para permitir que la pista se encoja hasta 0 (y el contenido haga scroll interno con `overflow: auto` + `min-width: 0` en el item, igual que en [[Flexbox]]).

```css
.columna-scroll {
  grid-template-columns: 240px minmax(0, 1fr); /* la columna scrollea, no estira */
}
```

- Con gaps incluidos, `1fr 1fr 1fr` **sí** funciona (a diferencia de los porcentajes en flex): los `fr` se reparten *después* de restar gaps. Nada de `(100% - gaps)/3`.
- `fr` con otros valores: `minmax(150px, 1fr) 2fr` — primero se garantizan los mínimos, el sobrante se reparte 1:2.

## `repeat()` y las preguntas frecuentes

```css
grid-template-columns: repeat(3, 1fr);        /* 3 columnas iguales */
grid-template-columns: repeat(2, 200px) 1fr;  /* repite y completa: mixto permitido */
grid-template-columns: repeat(auto-fill, 250px);
grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
```

| Forma | Cuándo genera pistas |
|---|---|
| `repeat(n, …)` | Siempre `n` pistas |
| `repeat(auto-fill, X)` | Tantas pistas de tamaño X como **caben** en el contenedor, **aunque queden vacías** |
| `repeat(auto-fit, X)` | Igual, pero las pistas **vacías colapsan a 0** y su espacio lo absorben las que tienen contenido |

- `auto-fill` es correcto cuando el contenido llega después y no querés que las columnas se estiren (feeds, dashboards).
- `auto-fit` es correcto para galerías: las tarjetas existentes se estiran para llenar la fila.
- **Por sí solas con píxeles fijos no se estiran**: el patrón que reemplaza breakpoints en [[Responsive Web Design]] es `repeat(auto-fit, minmax(250px, 1fr))`.

## Colocación de items: `grid-column`, `grid-row`, `grid-area`

La sintaxis es `grid-column: línea-inicio / línea-fin`:

```css
.box-1  { grid-column: 1 / 3; }       /* de la línea 1 a la 3 = ocupa 2 columnas */
.box-2  { grid-column: 2 / span 2; }  /* empieza en 2 y abarca 2 (= 2 / 4) */
.box-3  { grid-column: span 3; }      /* auto-start, ocupa 3 */
.box-4  { grid-column: 1 / -1; }      /* primera a última línea: ancho completo */
```

- `grid-row` funciona igual sobre las filas.
- `span` combinado con `-1`: `grid-column: 2 / -1` — desde la línea 2 hasta el final.
- Sin colocación explícita, el **auto-placement** ordena los items en el orden del DOM (ver "flujo automático" más abajo).

### Column span y row span en acción

```css
.grid-4 .box:nth-child(1) {
  grid-column: 2 / span 2; /* empieza en la línea 2 y abarca 2 columnas (equivale a 2 / 4) */
}

.box-alfila {
  grid-row: span 2; /* ocupa 2 filas */
}
```

## Flujo automático y `dense`

Cuando un elemento ocupa más pistas de las que le corresponden por orden, el auto-placement deja **huecos vacíos**:

![[Pasted image 20250731183018.png]]

`grid-auto-flow: dense` le permite al navegador **volver hacia atrás** y rellenar esos huecos con elementos posteriores:

```css
.grid-6 {
  display: grid;
  grid: repeat(2, 300px) / repeat(3, 300px);
  grid-auto-flow: row dense;
}

.grid-6 .box:nth-child(2) {
  grid-column: 1 / 2;
}
```

El panel queda así:

![[Pasted image 20250731183309.png]]

| Valor de `grid-auto-flow` | Comportamiento |
|---|---|
| `row` (defecto) | Llena fila a fila, con un cursor que **solo avanza** (*sparse*) |
| `column` | Llena columna a columna |
| `row dense` / `column dense` | Igual pero el cursor **reinicia** al principio para rellenar huecos |

> [!warning] `dense` reordena el orden visual
> Un item posterior puede aparecer **antes** visualmente. Para lectores de pantalla y tabulado el orden sigue siendo el del DOM (igual que `order` en [[Flexbox]]): si la lectura secuencial importa, evitá `dense`.

## Grid implícito: filas que nacen solas

Si escribís 10 items en un grid de 2×2, el navegador **crea** filas implícitas para acomodarlos. Esas pistas implícitas se dimensionan con `grid-auto-rows` / `grid-auto-columns` (por defecto `auto`):

```css
.feed {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: minmax(180px, auto); /* filas implícitas: mín. 180px */
  grid-auto-columns: 200px;            /* columnas implícitas (cuando el flujo es por columna) */
}
```

Saber distinguir **explícito vs implícito** explica el 80% de los "¿por qué apareció una fila gigante?": la fila implícita nace con `auto` y `auto` mide el contenido — un item con `grid-row: span 5` hila 5 filas implícitas enormes.

## Nombrar líneas (avanzado, muy útil)

Los nombres de línea se declaran entre corchetes dentro de las pistas y permiten colocarse por nombre en vez de número:

```css
.page {
  grid-template-columns:
    [full-start] minmax(1em, 1fr)
    [content-start] minmax(0, 1200px) [content-end]
    minmax(1em, 1fr) [full-end];
}

.full-bleed { grid-column: full-start / full-end; } /* de borde a borde */
.contenido   { grid-column: content-start / content-end; }
```

Este es el patrón **full-bleed**: el contenido centrado con tope de ancho, pero los fondos llegan a los bordes de la ventana — sin wrappers extra en el HTML (ver recetas).

## Grid areas: el layout como diagrama

Nombrar áreas deja el layout **legible como un mapa**: `grid-template-areas` dibuja literalmente la página con strings, donde **cada string es una fila** y **cada celda es una columna**:

```css
.grid-9 {
  display: grid;
  height: 120rem;
  grid-template-columns: repeat(3, 1fr);
  grid-template-areas:
    "header   header   header"
    "nav      nav      nav"
    "contenido contenido sidebar"
    "footer   footer   footer";
  grid-template-rows: 2fr 1fr 4fr 2fr;
  gap: 2rem;
}

.grid-9 .box { margin: 0; }
.grid-9 .box:nth-child(1) { grid-area: header; }
.grid-9 .box:nth-child(2) { grid-area: nav; }
.grid-9 .box:nth-child(3) { grid-area: contenido; }
.grid-9 .box:nth-child(5) { grid-area: footer; }
```

Reglas del "mapa":

- El nombre repetido en celdas contiguas **une** esas celdas en un solo rectángulo.
- **Todas** las filas deben tener la misma cantidad de celdas (mismo número de columnas).
- Cada área debe ser **rectangular** — `L` en forma de L invalida toda la declaración.
- El nombre no usado por ningún item queda como área vacía (útil para reservar espacio).

`grid-area` también acepta la forma larga `grid-area: row-start / column-start / row-end / column-end` (acepta nombres de línea, números y `span`).

## Shorthands: `grid`, `grid-template` y sus trampas

`grid` abrevia **todo**, filas primero y columnas después de la barra:

```css
.grid-5 {
  display: grid;

  /* versión larga */
  grid-template-rows: repeat(2, 300px);
  grid-template-columns: repeat(3, 300px);

  /* versión corta: grid: rows / columns */
  grid: repeat(2, 300px) / repeat(3, 300px);
}
```

`grid-template` combina áreas + tamaños de fila/columna:

```css
.grid-10 {
  display: grid;
  height: 120rem;
  grid-template-columns: repeat(3, 1fr);
  grid-template:
    "header   header   header" 2fr
    "nav      nav      nav" 1fr
    "contenido contenido sidebar" 4fr
    "footer   footer   footer" 2fr
    / 1fr 1fr 1fr; /* tras la barra: tamaños de las columnas */
  gap: 2rem;
}
```

> [!warning] Los shorthands reinician todo
> `grid` y `grid-template` **borran** cualquier valor previo de sus sub-propiedades (`grid-template-areas`, `grid-auto-flow`…). Además, en `grid-template` las columnas tras la barra **no aceptan `repeat()`**. Por eso en equipos muchos prefieren la forma explícita (`grid-template-*`): legibilidad y cero sorpresas — misma lógica que con `background` (ver [[Background]]).

## Gap

`gap` separa las pistas **sin afectar los bordes** de las cajas (ni siquiera existen gaps en los extremos):

```css
.grid-6 {
  display: grid;
  grid: repeat(2, 1fr) / repeat(3, 1fr);
  gap: 4rem 3rem; /* gap: filas columnas — también existen row-gap y column-gap */
}
```

> [!tip] `gap` también funciona en Flexbox
> Desde 2021 es compatible en flex containers en todos los navegadores modernos — ver [[Flexbox]]. En Grid llevaba desde 2017 (`grid-gap` es el alias viejo, ya solo-deprecado).

## Alineación: dos niveles (items y retícula)

Grid expone los mismos ejes que Flexbox pero en **dos capas**: dentro de cada celda, y de la retícula entera dentro del contenedor:

```css
.grid-11 {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  height: 30rem;
  align-items: center;    /* alinea cada ítem dentro de su celda (eje vertical) */
  justify-items: start;   /* eje horizontal */
  place-content: center;  /* centra toda la retícula si tiene espacio sobrante */
}
```

| Propiedad | Alinea | Valores típicos |
|---|---|---|
| `align-items` / `justify-items` | Cada ítem dentro de su celda | `stretch` (defecto), `start`, `end`, `center`, `baseline` |
| `align-self` / `justify-self` | Lo anterior, para **un** item | mismos + `auto` |
| `align-content` / `justify-content` | Toda la retícula dentro del contenedor | `start`, `end`, `center`, `space-between`, `stretch`… |
| `place-items` / `place-content` / `place-self` | Shorthands de ambos ejes | — |

Diferencias clave con [[Flexbox]]: en Grid el defecto de los items es `stretch` (la celda se llena); el `justify-*` acá no es "eje principal de flujo" sino eje horizontal, y el eje no depende de `grid-auto-flow` (aunque el flujo por columna sí cambia qué es "principal"). Cuando la retícula es más chica que el contenedor y sobra espacio, `place-content: center` la centra como bloque; con pistas `auto` y `stretch`, el espacio sobrante se reparte en vez de sobrar.

### Superponer items

Dos items colocados en la **misma área** se apilan; el orden del DOM (o `z-index`) decide cuál queda arriba:

```css
.hero       { grid-area: header; }
.hero-texto { grid-area: header; align-self: end; z-index: 1; }
```

Los items de Grid aceptan `z-index` **sin necesidad de `position`** — crean contexto de apilamiento propio (ver [[Box Model]] para el modelo de apilamiento).

## Subgrid: alinear hijos con el padre

Desde 2023 con soporte en los tres motores, un grid hijo puede **heredar las pistas del padre** con `subgrid` — la solución definitiva para filas de tarjetas cuyo contenido debe alinearse *entre* tarjetas:

```css
.grilla {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}

.tarjeta {
  display: grid;
  grid-row: span 2;
  grid-template-rows: subgrid;   /* hereda las filas del padre */
}

/* dentro de cada tarjeta: imagen en la fila 1, texto en la fila 2 —
   todas las imágenes comparten línea base, igual que los textos */
.tarjeta img { grid-row: 1; }
.tarjeta p   { grid-row: 2; }
```

Alternativa sin subgrid: `display: contents` en la tarjeta (sus hijos pasan a ser items del grid padre) — pero ahí se pierden el fondo y el padding de la tarjeta.

## Ajuste al contenedor: container queries

El grid puede adaptarse al **ancho de su contenedor** y no solo al de la pantalla: `container-type: inline-size` en el padre y `@container` en las reglas hijas. Ideal para una card reutilizable que vive en un sidebar angosto y en un main ancho a la vez (ver [[Responsive Web Design]]).

```css
.panel { container-type: inline-size; }

@container (min-width: 400px) {
  .panel .cards { grid-template-columns: repeat(2, 1fr); }
}
```

## Recetas prácticas

### 1. Holy grail (header / nav / main / aside / footer)

```css
.layout {
  display: grid;
  min-height: 100vh;                     /* o 100dvh en móvil, ver Medidas Relativas */
  grid-template-areas:
    "header header header"
    "nav    main   aside"
    "footer footer footer";
  grid-template-columns: 220px minmax(0, 1fr) 280px;
  grid-template-rows: auto 1fr auto;
  gap: 1rem;
}

.header { grid-area: header; }
.nav    { grid-area: nav; }
.main   { grid-area: main; min-width: 0; }
.aside  { grid-area: aside; }
.footer { grid-area: footer; }

/* responsive sin cambiar el HTML */
@media (max-width: 720px) {
  .layout {
    grid-template-areas:
      "header"
      "nav"
      "main"
      "aside"
      "footer";
    grid-template-columns: 1fr;
  }
}
```

### 2. Galería fluida sin media queries

```css
.galeria {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}
```

Una, dos, tres… columnas según el ancho disponible — sin contar items ni escribir un solo breakpoint.

### 3. Full-bleed con líneas nombradas

```css
.page {
  display: grid;
  grid-template-columns:
    [full-start] minmax(1em, 1fr)
    [main-start] minmax(0, 1200px) [main-end]
    minmax(1em, 1fr) [full-end];
}

.page > *   { grid-column: main-start / main-end; }
.banner     { grid-column: full-start / full-end; } /* fondo de borde a borde */
```

### 4. Artículo tipo revista (texto + imágenes cruzadas)

```css
.articulo {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1.5rem;
}

.articulo p       { grid-column: 1 / 3; }
.articulo .cita   { grid-column: 3 / 5; grid-row: span 2; }
.articulo .ancho  { grid-column: 1 / -1; }
```

### 5. Sidebar con ancho fijo + contenido scrolleable

```css
.split {
  display: grid;
  grid-template-columns: 300px minmax(0, 1fr);
  height: 100vh;
}
.split__sidebar { overflow-y: auto; }
.split__main    { overflow-y: auto; min-width: 0; }
```

## Accesibilidad

- **El orden del DOM manda**: tabuladores y lectores de pantalla recorren el DOM, no la retícula. `dense` puede reordenar lo visual — revisar con Tab.
- Grid permite **saltar filas/columnas visuales** (`grid-column: 3` en el primer item): el salto no se "siente" al navegar con teclado. Condiciones visuales complejas deben mantener una lectura secuencial coherente.
- `gap` genera separación táctil real (mínimo ~8px entre targets).
- Los layouts que usan `100vh` en móvil cortan contenido tras la barra del navegador: usar `svh`/`dvh` (ver [[Medidas Relativas]]).

## Rendimiento

- Grid es un modelo de **layout normal**: calcular pistas es barato para layouts de página; el costo aparece al **animar** tamaños de pista. Los motores modernos ya interpolan `grid-template-columns/rows` (Chrome 107+, Firefox 129+, Safari 16.4+), pero sigue siendo *layout* en cada frame: para animaciones fluidas preferir `transform`/`opacity` (ver [[Animaciones]]).
- Grid **reduce** el DOM: los patrones flex que requerían wrappers extra (centinelas `::after`, columnas fantasma) desapren — menos nodos, menos estilos, más rápido.
- `repeat(auto-fit, minmax(…))` elimina reflows de breakpoint: el navegador recalcula la retícula con el resize, sin JS.

## Errores comunes

1. **`1fr` y el scroll fantasma**: la pista `fr` tiene mínimo `min-content`; contenido largo la estira y aparece scroll horizontal. Solución: `minmax(0, 1fr)` + `overflow` en el item.
2. **Porcentajes + gaps**: `repeat(3, 33.33%)` con `gap` desborda el 100%. Con Grid usá `1fr` (los fr restan los gaps solos) — la fórmula `(100% - gaps)/n` solo si insistís con %.
3. **Esperar que el grid implícito mida como el explícito**: las filas nacidas solo tienen `auto` (tamaño de contenido). Definí `grid-auto-rows` o cambiá a `span` sobre pistas explícitas.
4. **Áreas no rectangulares o filas de distinto largo**: `grid-template-areas` invalida **toda** la declaración y el layout se cae al default — inspeccionar el mapa carácter por carácter.
5. **`dense` y el orden de lectura** — ver la advertencia de flujo automático.
6. **Mezclar shorthands**: escribir `grid: …` y luego `grid-template-areas` en otro bloque (o al revés) puede pisarse el shorthand con valores vacíos. Preferir siempre la forma explícita en CSS no trivial.
7. **Items con `margin` automático**: en Grid, `margin: auto` centra el item en su celda y absorbe el espacio sobrante — útil a propósito, confuso si es accidental.
8. **Olvidar `min-width: 0`** en items con contenido largo (código, tablas): mismo bug #1 de [[Flexbox]], mismo arreglo.

## Cheatsheet — todas las propiedades

| Propiedad | Dónde | Valores | Defecto |
|---|---|---|---|
| `display` | contenedor | `grid`, `inline-grid` | `block` |
| `grid-template-columns` / `grid-template-rows` | contenedor | pistas: longitudes, `fr`, `auto`, `minmax()`, `repeat()`, `subgrid`, nombres `[linea]` | `none` |
| `grid-template-areas` | contenedor | mapa de strings | `none` |
| `grid-template` | contenedor | shorthand de areas + filas + columnas | — |
| `grid-auto-columns` / `grid-auto-rows` | contenedor | pistas implícitas | `auto` |
| `grid-auto-flow` | contenedor | `row`, `column`, `dense`, `row dense`, `column dense` | `row` |
| `grid` | contenedor | shorthand general (`rows / columns`) | — |
| `gap` / `row-gap` / `column-gap` | contenedor | longitud, `%`, `normal` | `normal` |
| `justify-items` / `align-items` | contenedor | `stretch`, `start`, `end`, `center`, `baseline`… | `stretch` |
| `justify-content` / `align-content` | contenedor | `start`, `end`, `center`, `space-between`, `space-around`, `space-evenly`, `stretch` | `normal` (= stretch) |
| `place-items` / `place-content` / `place-self` | contenedor / item | shorthands de ambos ejes | — |
| `grid-column` / `grid-row` | item | `inicio / fin`, `span n`, `-1`, nombres | `auto` |
| `grid-area` | item | nombre de área o shorthand `r-start / c-start / r-end / c-end` | `auto` |
| `justify-self` / `align-self` | item | `start`, `end`, `center`, `stretch`, `auto` | `auto` |

## Compatibilidad

Soporte **total en todos los navegadores actuales** (2026), incluyendo `subgrid` y la interpolación de pistas. Datos históricos: antes de 2017 era imprescindible el prefijo `-ms-` con sintaxis distinta (IE10/11: `-ms-grid-*`, sin `fr` real) — cualquier CSS `-ms-grid-` en herencia es legado eliminable. `grid-gap`/`grid-row-gap`/`grid-column-gap` son alias de la especificación vieja: funcionan, pero usar `gap`.

## Relacionadas

- [[Flexbox]] — organización en una dimensión (el compañero natural)
- [[Cuando usar GRID o Flexbox]] — guía de decisión con ejemplos lado a lado
- [[Medidas Relativas]] — `fr`, `rem`, `vh`/`dvh` y el resto de unidades de las pistas
- [[Responsive Web Design]] — cómo Grid se adapta (auto-fit, container queries)
- [[Tipos de Display]] — cómo nace el grid container
- [[Box Model]] — cómo cada celda entrega su contenido a los items
- [[Animaciones]] — por qué animar pistas es caro y cómo evitarlo
- [[TailwindCSS - Sintaxis Básica]] — la grilla expresada como utilidades (`grid-cols-*`, `col-span-*`)
