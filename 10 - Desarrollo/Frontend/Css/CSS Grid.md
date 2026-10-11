---
aliases: ["Css Grid", "Grid CSS", "CSS grid layout"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Flexbox]]"
  - "[[Cuando usar GRID o Flexbox]]"
  - "[[Medidas Relativas]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_grid_layout"
---

# CSS Grid

CSS Grid permite definir la **ubicación y el tamaño** de los contenidos sobre una retícula de filas y columnas. El contenido se agrupa dentro de un área definida; en cierta forma es como una tabla de HTML, con la ventaja de mucho más flexibilidad y control sobre el diseño.

![[Pasted image 20250124170628.png]]

A diferencia de [[Flexbox]], que organiza en **una sola dimensión** (fila *o* columna), Grid trabaja en **dos dimensiones** a la vez. Ninguno es mejor que otro: se complementan y se pueden usar juntos en el mismo diseño (ver [[Cuando usar GRID o Flexbox]]).

## Base: filas, columnas y líneas

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* 3 columnas iguales */
  grid-template-rows: 200px 1fr;         /* 2 filas: una fija, una flexible */
  gap: 2rem;
}
```

- Las columnas se crean con `grid-template-columns`, las filas con `grid-template-rows`.
- Las **líneas** se numeran desde 1: la columna 2 empieza en la línea 2.
- `1fr` es una fracción del espacio libre — ver [[Medidas Relativas]].

## Column Span y Row Span

Un elemento puede ocupar **más de una pista** con `span`:

```css
.grid-4 .box:nth-child(1) {
  grid-column: 2 / span 2; /* empieza en la línea 2 y abarca 2 columnas (equivale a 2 / 4) */
}

.box-alfila {
  grid-row: span 2; /* ocupa 2 filas */
}
```

La sintaxis es `grid-column: inicio / fin` (línea inicial / línea final). También se aceptan los nombres de área con `grid-area`.

## Grid shorthand

`grid` abrevia filas y columnas en una línea, separadas por **barra** (filas primero, columnas después):

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

> [!warning] El shorthand `grid` reinicia todo
> Igual que con `background`, usar `grid` borra valores implícitos de otras sub-propiedades. En equipos mixtos, muchos prefieren la forma explícita (`grid-template-*`) para legibilidad.

## Grid autoflow y `dense`

Cuando un elemento ocupa más pistas de las que le corresponden por orden, el auto-placement deja **huecos vacíos**:

![[Pasted image 20250731183018.png]]

`grid-auto-flow: dense` le permite al navegador **rellenar esos huecos** con elementos posteriores (ojo: eso puede alterar el orden visual y de lectura):

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

Las opciones de `grid-auto-flow` son `row` (por defecto), `column` y `dense` (o `row dense` / `column dense`). En la práctica, `dense` es la que resuelve los huecos.

## Gap

`gap` separa las pistas **sin afectar los bordes** de las cajas (a diferencia de los márgenes, que colapsan y se acumulan):

```css
.grid-6 {
  display: grid;
  grid: repeat(2, 1fr) / repeat(3, 1fr);
  gap: 4rem 3rem; /* gap: filas columnas — también existen row-gap y column-gap */
}
```

> [!tip] `gap` también funciona en Flexbox
> Desde 2021 `gap` es compatible en flex containers en todos los navegadores modernos — ver [[Flexbox]].

## Grid areas

Nombrar las áreas deja el layout **legible como un diagrama**: el `grid-template-areas` dibuja literalmente el mapa de la página con strings, donde cada string es una fila:

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

Cada celda repetida une sus bloques. El nombre debe repetirse en **todas** las celdas que ocupa el área, y el mapa debe ser rectangular (no se admiten huecos).

## Grid template (shorthand de áreas)

`grid-template` combina áreas, tamaños de fila y columnas en una sola declaración:

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
    / 1fr 1fr 1fr; /* al final, tras la barra: tamaños de las columnas */
  gap: 2rem;
}
```

> [!warning] Limitación del shorthand
> En `grid-template` las columnas finales (`/ 1fr 1fr 1fr`) **no aceptan `repeat()`**. Si necesitás `repeat`, usá las propiedades explícitas.

## Alineación

Grid expone los mismos ejes de alineación que Flexbox, pero aplicados a la **retícula completa**:

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

| Propiedad                          | Alinea                                                     |
| ---------------------------------- | ---------------------------------------------------------- |
| `align-items` / `justify-items`    | Cada ítem dentro de su celda (vertical / horizontal)       |
| `align-content` / `justify-content`| Toda la retícula dentro del contenedor                     |
| `place-items` / `place-content`    | Shorthands que fijan ambos ejes a la vez                   |

## Autofill y autofit

Para grids **fluidos** que crean columnas según el ancho disponible:

```css
.grid-12 {
  display: grid;
  /* ambas líneas: la segunda pisa a la primera — se elige UNA */
  grid-template-columns: repeat(auto-fill, 200px);
  grid-template-columns: repeat(auto-fit, 200px);
}
```

- `auto-fill` genera tantas columnas de 200px como entren, **aunque queden vacías**.
- `auto-fit` genera las mismas pero **colapsa las vacías**, estirando las que tienen contenido.

Por sí solas con píxeles fijos no se estiran; para eso se combinan con `minmax()`:

```css
.grid-12 {
  display: grid;
  /* minmax(min, max): cada columna mide al menos 200px y hasta 1fr */
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
```

Este patrón (`repeat(auto-fit, minmax(X, 1fr))`) reemplaza en la mayoría de casos a los breakpoints de [[Responsive Web Design]] para galerías y tarjetas.

> [!tip] Container queries
> En CSS moderno, el grid puede adaptarse al **contenedor** y no solo a la pantalla: con `container-type: inline-size` en el padre y `@container` en las reglas hijas. Ideal para cards reutilizables en sidebars y main a la vez.

## Relacionadas

- [[Flexbox]] — organización en una dimensión
- [[Cuando usar GRID o Flexbox]] — guía de decisión
- [[Medidas Relativas]] — `fr` y unidades relativas
- [[Responsive Web Design]] — grids adaptables con media queries
- [[Tipos de Display]] — cómo nace el grid container
