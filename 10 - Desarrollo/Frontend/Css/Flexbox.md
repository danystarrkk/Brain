---
aliases: ["Flexbox", "Modelo Flex"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Cuando usar GRID o Flexbox]]"
  - "[[CSS Grid]]"
  - "[[Tipos de Display]]"
  - "[[CSS Introducción]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_flexible_box_layout"
---

# Flexbox

Flexbox (**Flexible Box Layout**) es un modelo de layout **unidimensional**: organiza los elementos en un solo eje (fila *o* columna). A cambio de renunciar al control de las dos dimensiones a la vez, ofrece un control finísimo de la alineación y el reparto del espacio disponible.

> [!info] Un eje, dos roles
> Un contenedor flex tiene un **eje principal** (main axis, por defecto la fila) y un **eje cruzado** (cross axis, perpendicular). Las propiedades `justify-*` trabajan sobre el eje principal y las `align-*` sobre el cruzado. Entender esto evita el 90% de la confusión con Flexbox.

## Activar el modelo

```css
.contenedor {
  display: flex;
}
```

Eso convierte al elemento en contenedor flex y a **sus hijos directos** en items flex. No hace falta envolver nada en clases extra: cualquier hijo directo pasa a ser flex item.

## `flex-direction`: fila o columna

El valor por defecto es `row`. Los valores posibles son `row`, `row-reverse`, `column` y `column-reverse`; definen cuál es el eje principal y hacia dónde crece.

### `row` (por defecto)

Los hijos se colocan de izquierda a derecha, uno junto al otro:

![[Pasted image 20250124163705.png]]

### `column`

Los hijos se colocan de arriba hacia abajo:

![[Pasted image 20250124163736.png]]

> [!warning] `*-reverse` con cuidado
> `row-reverse` invierte el orden visual pero **no** el orden del DOM: para un lector de pantalla (y para el tabulado con teclado) el primer elemento sigue siendo el primero. Usalo solo con fines puramente visuales.

## Alinear: `justify-content` e `align-items`

```css
.contenedor {
  display: flex;
  justify-content: center;   /* eje principal: start | center | end | space-between | space-around | space-evenly */
  align-items: center;       /* eje cruzado: start | center | end | stretch (defecto) */
  min-height: 100vh;         /* truco clásico para centrar en pantalla completa */
}
```

Centrar horizontal *y* verticalmente un hijo es una línea con Flexbox:

```css
.contenedor {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

## `gap`: separación sin trucos

Desde que `gap` es soportada en flexbox, **no se usan márgenes negativos ni `justify-content: space-between` con un fantasma vacío** para separar items:

```css
.contenedor {
  display: flex;
  gap: 1rem;        /* separación uniforme entre items */
  column-gap: 2rem; /* o por eje */
  row-gap: 1rem;
}
```

La ventaja sobre los márgenes es que **no agrega espacio en los extremos**, solo entre elementos.

## `flex-wrap`: dejar que las filas fluyan

Por defecto los items intentan caber todos en una sola línea y se comprimen. `flex-wrap: wrap` permite crear una línea nueva cuando no hay espacio, sin que se deforme el diseño:

```css
.contenedor {
  display: flex;
  flex-wrap: wrap; /* por defecto: nowrap */
  gap: 1rem;
}
```

Con `wrap` conviene definir un ancho base (por ejemplo `flex-basis: 33.3%`) para que sepas cuántos items caben por fila.

## Tamaño de los items: `flex-basis`, `flex-grow` y `flex-shrink`

Estas tres propiedades controlan cómo reparte el contenedor el espacio disponible.

### `flex-basis`

Define el **tamaño inicial** del item en el eje principal, antes de que crezca o se contraiga:

```css
.d-flex-10 .box {
  flex-basis: 33.3%;   /* tamaño base: un tercio del contenedor */
}
```

Admite cualquier unidad (`px`, `%`, `rem`...). Es el punto de partida desde el cual actúan `flex-grow` y `flex-shrink`.

### `flex-grow`: factor de crecimiento

Controla qué proporción del **espacio libre** recibe cada item. Por defecto es `0` (nadie crece: el item toma el ancho de su contenido). Si todos tienen `flex-grow: 1`, el espacio sobrante se reparte **por igual**:

```css
.d-flex-11 .box {
  flex-grow: 1; /* las 3 cajas reparten el espacio sobrante a partes iguales */
}
```

Su valor real es cuando los factores **no** son iguales: el sobrante se reparte proporcionalmente al factor, no "de un pixel en un pixel":

```css
.d-flex-11 .box:nth-child(1) { flex-grow: 3; } /* 3/6 del sobrante */
.d-flex-11 .box:nth-child(2) { flex-grow: 2; } /* 2/6 */
.d-flex-11 .box:nth-child(3) { flex-grow: 1; } /* 1/6 */
```

Visualmente:

![[Pasted image 20250729213842.png]]

> [!tip] El reparto es proporcional, no absoluto
> Con `flex-grow: 3` la caja 1 no mide "3 px más": recibe **3 partes de las 6 totales** del espacio libre. Siempre se calcula sobre el espacio sobrante del contenedor, no sobre un valor fijo.

### `flex-shrink`: factor de contracción

El opuesto: controla a qué velocidad se encoge cada item cuando **no cabe** todo. Por defecto es `1`; `0` significa "nunca me encogo":

```css
.d-flex-12 .box:nth-child(3) {
  flex-shrink: 2; /* la 3ra caja se contrae el doble de rápido que las demás */
}
```

## Atajo `flex`

Las tres propiedades se escriben juntas con la sintaxis `flex: grow shrink basis`:

```css
.d-flex-13 .box {
  /* versión larga */
  flex-basis: 33.3%;
  flex-grow: 1;
  flex-shrink: 0;

  /* versión corta — idéntica */
  flex: 1 0 33.3%;
}
```

Atajos comunes:

| Atajo        | Equivalente expandido            | Uso típico                          |
| ------------ | -------------------------------- | ----------------------------------- |
| `flex: 1`    | `1 1 0%`                         | columnas iguales que reparten todo  |
| `flex: auto` | `1 1 auto`                       | crecen pero respetan su contenido   |
| `flex: none` | `0 0 auto`                       | tamaño fijo, no crece ni se encoge  |

> [!warning] Orden y valores del atajo
> Es `grow shrink basis` (aunque en la práctica `flex: 1` también funciona con solo un número). Ojo: `flex: 1` pone `flex-basis: 0%`, **no** respeta el contenido inicial; si querés que el contenido mande, usá `flex: auto`.

## Errores comunes

- **Medir los márgenes de los hijos**: con Flexbox el espacio se hace con `gap`, no con márgenes.
- **Olvidar `min-height`/`min-width`**: un item flex por defecto no se encoge por debajo de su contenido (`min-height: auto`). En layouts de altura completa casi siempre hay que poner `min-height: 0` en los hijos para permitir scroll interno.
- **Confundir unidimensional con bidimensional**: si necesitás controlar filas *y* columnas a la vez, eso es [[CSS Grid]]. Regla práctica en [[Cuando usar GRID o Flexbox]].

## Relacionadas

- [[Cuando usar GRID o Flexbox]] — cuándo elige uno u otro
- [[CSS Grid]] — el modelo bidimensional complementario
- [[Tipos de Display]] — cómo `display: flex` entra en la familia de displays
- [[CSS Introducción]] — fundamentos del lenguaje
