---
aliases: ["Flexbox", "Modelo Flex", "Flexible Box Layout"]
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
  - "[[Especificidad]]"
  - "[[TailwindCSS - Sintaxis Básica]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_flexible_box_layout"
---

# Flexbox

**Flexbox** (*Flexible Box Layout*) es el modelo de layout **unidimensional** de CSS: organiza a sus hijos en un solo eje — fila *o* columna — con control fino sobre alineación, distribución del espacio y comportamiento ante el resize. Es el modelo de layout por defecto para componentes de interfaz (navs, toolbars, cards, formularios, centrados) y el fundamento sobre el que se apoyan frameworks como Tailwind (ver [[TailwindCSS - Sintaxis Básica]]).

> [!info] Por qué existe Flexbox
> Antes de Flexbox, los layouts se hacían con `float` (hack frágil, con clear-fix de por medio), `display: table` (semántica falsa) o posicionamiento absoluto (fuera del flujo). Flexbox nació (2012, estabilizado en 2015) para resolver **un problema concreto**: alinear y distribuir espacio entre elementos **dentro de una línea** sin romper el flujo del documento. Para dos dimensiones a la vez llegó [[CSS Grid]] en 2017 — ver [[Cuando usar GRID o Flexbox]] para elegir entre ambos.

## Modelo conceptual: ejes, líneas y roles

Todo lo que sigue se entiende con este vocabulario:

| Término | Significado |
|---|---|
| **Contenedor flex** | El elemento con `display: flex` o `display: inline-flex` |
| **Flex items** | Los **hijos directos** del contenedor (no los nietos) |
| **Eje principal** (*main axis*) | Dirección por la cual fluyen los items. Determinada por `flex-direction` |
| **Eje cruzado** (*cross axis*) | Siempre **perpendicular** al eje principal |
| **Main start / end** | Extremos del eje principal (por donde empieza y termina el flujo) |
| **Cross start / end** | Extremos del eje cruzado |
| **Main size** | El ancho (o alto) del item sobre el eje principal — lo que controlan `flex-basis`, `grow`, `shrink` |
| **Cross size** | El alto (o ancho) del item sobre el eje cruzado — lo que controlan `align-items` / `align-self` |
| **Línea** (*line*) | Cada fila o columna de items; con `wrap` pueden existir varias |

> [!tip] La regla que resuelve el 90% de la confusión
> **`justify-*` = eje principal. `align-*` = eje cruzado.** Da igual si el eje principal es horizontal o vertical: `justify-content: center` siempre centra *a lo largo de la dirección de flujo*, e `align-items: center` siempre centra *a lo ancho de ella*. Cambiás `flex-direction` y los roles se invierten solos.

## Activar el modelo

```css
.contenedor {
  display: flex;          /* bloque, ocupa su línea */
  /* display: inline-flex;  mismo modelo pero inline (fluye con el texto) */
}
```

Con eso el elemento se vuelve contenedor flex y **todos sus hijos directos** se convierten en flex items automáticamente — sin envolverlos en clases extra ni tocar el HTML. Los hijos pierden ciertos comportamientos por defecto (por ejemplo, colapsan su `width: auto` según su contenido, y los márgenes verticales de bloques ya no colapsan).

### Contenedor vs item: qué propiedades aplican dónde

| Propiedades de **contenedor** | Propiedades de **item** |
|---|---|
| `display`, `flex-direction`, `flex-wrap`, `flex-flow` | `flex` (`grow` `shrink` `basis`) |
| `justify-content`, `align-items`, `align-content` | `align-self`, `order` |
| `gap`, `row-gap`, `column-gap` | `min-width/height`, `max-width/height` (comportamiento especial) |

Poner `justify-content` en un hijo **no hace nada** — es una de las confusiones más frecuentes.

## `flex-direction`: definir el eje principal

```css
.contenedor { flex-direction: row | row-reverse | column | column-reverse; }
```

### `row` (por defecto)

Los hijos se colocan de izquierda a derecha, uno junto al otro:

![[Pasted image 20250124163705.png]]

### `column`

Los hijos se colocan de arriba hacia abajo:

![[Pasted image 20250124163736.png]]

Cuándo usar `column`: sidebars verticales, listas, layouts de altura completa (header + contenido + footer), tarjetas con contenido apilado.

> [!warning] `row-reverse` / `column-reverse` y accesibilidad
> Invierte el orden **visual**, pero **no** el orden del DOM: para un lector de pantalla y para el tabulado con Tab, el primer elemento sigue siendo el primero. Si el orden visual importa para la lectura, **cambiá el orden en el HTML**, no con `*-reverse`. Usá `*-reverse` solo con fines puramente estéticos y nunca en contenido lineal.

## Alinear y distribuir: `justify-content` e `align-items`

### `justify-content` — sobre el eje principal

```css
.contenedor {
  justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
}
```

| Valor | Comportamiento | Visual |
|---|---|---|
| `flex-start` (defecto) | Todo al inicio; espacio sobrante al final | `AAAA....` |
| `flex-end` | Todo al final | `....AAAA` |
| `center` | Todo centrado | `..AAAA..` |
| `space-between` | Primer item al inicio, último al final, resto reparte el medio | `A..A..A` |
| `space-around` | Igual espacio a izq. y der. de **cada** item (medio al extremo = la mitad) | `.A.A.A.` |
| `space-evenly` | El **mismo** espacio en todos los huecos incluidos los extremos | `..A.A..` |

> [!warning] `space-between` con un solo hijo
> Si el contenedor tiene **un único item**, `space-between` lo empuja al inicio (no hay "entre qué" separar). Caso clásico: el logo de una nav desaparece a la izquierda cuando solo queda un elemento visible. Con `center` no pasa.

Además, CSS añade `safe`/`unsafe` para decidir si la alineación rota el scroll: `justify-content: safe center` — si el contenido no cabe, cae al `start` en vez de cortar el inicio.

### `align-items` — sobre el eje cruzado

```css
.contenedor {
  align-items: stretch | flex-start | flex-end | center | baseline;
}
```

| Valor | Comportamiento |
|---|---|
| `stretch` (defecto) | Los items estiran hasta llenar el eje cruzado — **solo si su `height`/`width` en ese eje es `auto`** |
| `flex-start` / `flex-end` | Tamaño según contenido, alineados al inicio/final |
| `center` | Centrados, tamaño según contenido |
| `baseline` | Alineados por la línea de base de su primera línea de texto — ideal para tamaños de fuente distintos |

### El truco del centrado perfecto

```css
.pantalla-completa {
  display: flex;
  justify-content: center;   /* centro en el eje principal */
  align-items: center;       /* centro en el eje cruzado */
  min-height: 100vh;         /* necesario: sin esto el contenedor mide su contenido */
}
```

Una línea para centrar cualquier cosa horizontal **y** verticalmente — el problema que durante 15 años se resolvía con `position: absolute` + `transform` + `top: 50%`.

> [!note] Centro con desbordes
> Si el contenido centrado es más grande que la pantalla, `justify-content: center` recorta **ambos** extremos (el inicio queda fuera del scroll). Alternativa robusta: `margin: auto` en el propio item — centra igual pero permite scrollear hasta el inicio.

## `gap`: separación correcta entre items

```css
.contenedor {
  gap: 1rem;              /* row-gap y column-gap a la vez */
  row-gap: 1rem;          /* solo horizontal cuando flex-direction: column */
  column-gap: 2rem;
}
```

- **No genera espacio en los extremos** — solo entre elementos. Por eso reemplazó a los márgenes: `.item { margin-right: 1rem }` dejaba un fantasma después del último item.
- `gap` en flexbox tiene **soporte total desde 2021** (Chrome 84, Firefox 63, Safari 14.1). En proyectos muy viejos todavía se ve el patrón con `margin-left` negativo — no replicarlo.
- Con `flex-wrap: wrap`, `row-gap` separa las filas generadas — sin hacks.

## `flex-wrap`: dejar que las líneas fluyan

```css
.contenedor {
  flex-wrap: nowrap | wrap | wrap-reverse;   /* defecto: nowrap */
  gap: 1rem;
}
```

- `nowrap` (defecto): todo intenta caber en **una línea** — los items se comprimen con `flex-shrink` hasta donde puedan. Con pocas palabras esto produce layouts deformes inesperados.
- `wrap`: cuando no hay espacio, el item salta a una línea nueva.
- `wrap-reverse`: igual pero las líneas apiladas de abajo hacia arriba (uso casi nulo, efectos visuales).

### Patrón: grid responsive con flex-wrap

Para tarjetas que se acomodan solas según el ancho disponible (el equivalente flex de `grid-template-columns: repeat(auto-fit, minmax(250px, 1fr))`):

```css
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}

.cards > * {
  flex: 1 1 250px;   /* base 250px; crece y se encoge para llenar la fila */
}
```

Con 4 items y espacio para 3 por fila: los 3 primeros se reparten la fila, el cuarto salta sola a la segunda fila y **se estira al 100%** (como es el único, `flex-grow` lo hincha). Para que el último item **no** se estire deforme, la receta clásica es añadir un centinela invisible:

```css
.cards::after {
  content: "";
  flex: 1 0 250px;   /* ocupa el hueco sobrante y evita que el último crezca */
}
```

O directamente usar `max-width` con `margin: auto`… o migrar a `auto-fit` de [[CSS Grid]], que resuelve esto de forma nativa.

## Tamaño de los items: `flex-basis`, `flex-grow`, `flex-shrink`

Las tres propiedades que gobiernan cómo reparte el contenedor **el espacio disponible en el eje principal**.

### `flex-basis`: el punto de partida

Define el **tamaño inicial** del item en el eje principal **antes** de que actúen crecimiento o contracción:

```css
.box {
  flex-basis: 33.3%;   /* tamaño base: un tercio del contenedor */
  /* admite px, rem, %, vw... o auto (mide su contenido) */
}
```

Si hay espacio libre, `flex-grow` decide quién se lo lleva; si falta espacio, `flex-shrink` decide quién cede. Pero **el punto de partida es `flex-basis`**.

### `flex-grow`: reparto del sobrante

Factor de crecimiento. Por defecto `0`: el item **no** crece, se queda en su `flex-basis` (o en su contenido si es `auto`). Cuando hay espacio libre, se reparte **proporcionalmente** entre los items con `grow > 0`:

```css
.caja-1 { flex-grow: 3; }   /* recibe 3/6 del espacio libre */
.caja-2 { flex-grow: 2; }   /* 2/6 */
.caja-3 { flex-grow: 1; }   /* 1/6 */
```

![[Pasted image 20250729213842.png]]

> [!tip] El reparto es proporcional al **sobrante**, no absoluto
> `flex-grow: 3` no significa "mide 3px más": la caja recibe **3 partes de las 6 totales del espacio libre**. Si el contenedor sobra 600px, la caja 1 gana 300px, la segunda 200, la tercera 100. Y solo se reparte lo que sobra *después* de los `flex-basis`.

### `flex-shrink`: reparto del déficit

El espejo: cuando **no cabe** todo, cada item cede espacio proporcional a su factor. Por defecto es `1`; `0` significa *"nunca me encogo"*:

```css
.fijo       { flex-shrink: 0; }              /* mantiene su tamaño pase lo que pase */
.elastico-2 { flex-shrink: 2; }              /* cede el doble de rápido que los demás */
```

> [!warning] El caso típico del shrink descontrolado
> Imágenes o tablas dentro de un flex `nowrap` se deforman porque heredan `flex-shrink: 1`. Solución: `flex-shrink: 0` en el elemento fijo, o `max-width: 100%` + `height: auto` en las imágenes (ver [[Imágenes CSS]]). Para texto largo que no debe encoger: `min-width: 0` es casi obligatorio, ver "errores comunes" más abajo.

### Atajo `flex`

Las tres se escriben juntas: **`flex: grow shrink basis`**:

```css
.box {
  /* versión larga */
  flex-grow: 1;
  flex-shrink: 1;
  flex-basis: 0%;

  /* versión corta — exactamente equivalente */
  flex: 1 1 0%;
}
```

| Atajo | Expandido | Significado | Uso típico |
|---|---|---|---|
| `flex: 1` | `1 1 0%` | todos miden 0 y reparten el total | columnas iguales, sidebar adaptable |
| `flex: auto` | `1 1 auto` | tamaño base = contenido, crece si sobra | nav con items que respiran |
| `flex: none` | `0 0 auto` | ni crece ni encoge | logos, badges, elementos fijos |
| `flex: initial` | `0 1 auto` | valores por defecto del navegador | volver al estado inicial |

> [!warning] `flex: 1` vs `flex: auto` — la diferencia que rompe layouts
> `flex: 1` pone `flex-basis: **0%**`: el contenido inicial **no** cuenta, todo se reparte a partes iguales. `flex: auto` usa `flex-basis: **auto**`: cada item arranca con su tamaño de contenido y solo el sobrante se reparte. Con textos de distinta longitud: `flex: 1` iguala todo; `flex: auto` respeta el contenido. Elegir uno u otro cambia el diseño por completo.

## `align-content`, `align-self` y `order`

### `align-content` (contenedor, solo con `wrap`)

Agrupa las **líneas** generadas cuando hay varias filas/columnas: `stretch` (defecto), `flex-start`, `flex-end`, `center`, `space-between`, `space-around`. Sin `wrap` no hace nada.

### `align-self` (item)

Sobreescribe `align-items` para **un solo item**:

```css
.item-destacado { align-self: flex-start; }  /* el resto sigue con stretch */
```

### `order` (item)

Reordena visualmente los items (defecto `0`; los menores van primero):

```css
.item-nuevo { order: -1; }   /* aparece primero aunque sea el último en el HTML */
```

> [!warning] `order` también rompe la relación con el teclado
> Igual que `*-reverse`: `order` cambia el orden visual **sin** cambiar el orden del DOM. Si alguien navega con Tab, saltará en un orden que no coincide con lo que ve. Regla: decoración sí, contenido navegable no.

### Posicionamiento absoluto dentro de un flex item

Un hijo con `position: absolute` **sale del flujo flex** pero su caja de referencia es el flex item que tiene `position: relative` (y se posiciona respecto de sus bordes de contenido). Patrón habitual: badge superpuesto en una esquina de card:

```css
.card { position: relative; display: flex; flex-direction: column; }
.card__badge { position: absolute; top: 0.5rem; right: 0.5rem; }
```

## Recetas prácticas

### 1. Barra de nav completa (izquierda-centro-derecha)

```css
.nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem;
}
```

Con `space-between` + `gap`, el logo queda pegado a la izquierda, el menú al medio y las acciones a la derecha — sin márgenes mágicos ni centrados negativos.

### 2. Sidebar + contenido a pantalla completa

```css
.layout {
  display: flex;
  min-height: 100vh;          /* el contenedor mide la ventana, no su contenido */
}

.sidebar { flex: 0 0 260px; } /* ancho fijo, no encoge */
.contenido {
  flex: 1;                    /* toma el resto */
  min-width: 0;               /* permite scroll horizontal interno */
}
```

`min-width: 0` es **obligatorio** aquí: sin él, el contenido largo (código, tablas) fuerza el ancho del item y rompe el layout en lugar de hacer scroll.

### 3. Sticky footer (footer pegado al fondo, pero no flotando)

```css
body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

main { flex: 1; }   /* el contenido empuja el footer hacia abajo siempre */
```

### 4. Media object (avatar al lado del texto — patrón de redes sociales)

```css
.media { display: flex; gap: 1rem; }
.media__avatar { flex: 0 0 48px; }        /* siempre 48px */
.media__body   { flex: 1; min-width: 0; } /* el texto fluye y scrollea */
```

### 5. Botones de un formulario alineados al fondo

```css
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 1.5rem;
}
/* Variante: acciones repartidas a izq y der con 3 hijos:
   flex-start | space-between(center) | flex-end — el truco de 3 elementos */
```

## Accesibilidad

- **El orden del DOM es ley**: los lectores de pantalla y el tabulado recorren el DOM, no la posición visual. `order` y `*-reverse` solo para efectos decorativos.
- **`align-items: center` + texto largo**: centrar verticalmente bloques de texto grande puede cortar el inicio en pantallas bajas; verificar con zoom al 200%.
- **`gap` como separación**: da un target táctil real entre botones (mínimo 8px recomendado) a diferencia de márgenes colapsados.
- Contraste y foco visibles no dependen de Flexbox, pero un layout centrado con `order` rara vez mantiene una secuencia de foco coherente — revisar con Tab después de reordenar.

## Rendimiento

- Flexbox es un modelo de **layout normal**: recalcular tamaños/posiciones de muchos items en cada resize/scroll puede disparar *reflows*. En la práctica, layouts de interfaz son triviales para el navegador; el costo real aparece con **animar** `width`/`height`/`top` (dispara layout cada frame). Para animar, usá `transform` y `opacity` que van por composición (ver [[Animaciones]]).
- **Flexbox vs Grid vs Float vs Table**:

| Modelo | Dimensiones | Uso típico | Estado hoy |
|---|---|---|---|
| `float` | 1 (hack) | texto en columns, layouts antiguos | Legado; solo flotar imágenes en texto |
| `display: table` | 2 (limitado) | layouts antiguos | Legado |
| **Flexbox** | **1** | componentes, navs, centrados, filas | **Estándar** |
| **CSS Grid** | **2** | grillas de página, layouts completos | **Estándar** |

## Errores comunes

1. **Márgenes en los items en vez de `gap`**: deja fantasmas en los extremos y complica los `*-reverse`. Gap siempre.
2. **Olvidar `min-width: 0` / `min-height: 0`**: el valor inicial de `min-width/auto` en un flex item es su tamaño mínimo de contenido — por eso una columna de texto largo **no hace scroll**, rompe el layout. Es el bug #1 de Flexbox. Fórmula: *si el hijo tiene contenido largo y debe desbordarse con scroll, poné `min-width/min-height: 0` en el item*.
3. **Esperar `stretch` con `height` definido**: `align-items: stretch` solo estira items cuyo tamaño en el eje cruzado es `auto`. Si le pusiste `height: 200px`, ese item **no** se estira.
4. **`flex: 1` cuando querías `flex: auto`** (o viceversa) — ver la advertencia de la sección del atajo.
5. **Poner propiedades de item en el contenedor** (`justify-content` en un hijo, `flex-grow` en el padre): simplemente no hacen nada.
6. **Contador mágico de porcentajes**: `flex: 0 0 33.333%` + `gap` **no caben** (33.33%×3 + gaps > 100%). Con gaps, la fórmula segura es `flex: 1 1 0` con `gap` natural, o calcular el ancho como `(100% - gaps) / n`.
7. **Imágenes que no respetan su caja**: en flex, a las imágenes siempre `max-width: 100%; height: auto;` (ver [[Imágenes CSS]]).
8. **`justify-content: space-between` con 1 o 2 items donde esperabas centrado** — ver la advertencia de la sección de `justify-content`.

## Cheatsheet — todas las propiedades

| Propiedad | Dónde | Valores | Defecto |
|---|---|---|---|
| `display` | contenedor | `flex`, `inline-flex` | `block` |
| `flex-direction` | contenedor | `row`, `row-reverse`, `column`, `column-reverse` | `row` |
| `flex-wrap` | contenedor | `nowrap`, `wrap`, `wrap-reverse` | `nowrap` |
| `flex-flow` | contenedor | shorthand de direction + wrap | — |
| `justify-content` | contenedor | `flex-start`, `flex-end`, `center`, `space-between`, `space-around`, `space-evenly`, `safe/unsafe` | `flex-start` |
| `align-items` | contenedor | `stretch`, `flex-start`, `flex-end`, `center`, `baseline` | `stretch` |
| `align-content` | contenedor (con wrap) | `stretch`, `flex-start`, `flex-end`, `center`, `space-between`, `space-around` | `stretch` |
| `gap` / `row-gap` / `column-gap` | contenedor | longitud, `%`, `normal` | `normal` |
| `flex` | item | shorthand `grow shrink basis` o `1/none/auto/initial` | `0 1 auto` |
| `flex-grow` | item | número ≥ 0 | `0` |
| `flex-shrink` | item | número ≥ 0 | `1` |
| `flex-basis` | item | `auto`, longitud, `%` | `auto` |
| `align-self` | item | igual que align-items + `auto` | `auto` |
| `order` | item | entero | `0` |

## Compatibilidad

Soporte **total en todos los navegadores actuales** (2026). Datos históricos útiles al leer código viejo: IE10 soportaba Flexbox solo con prefijo `-ms-` y una sintaxis antigua (`-ms-flex-pack`); Safari 8 usaba `-webkit-`. Cualquier regla `-webkit-box` / `-ms-flex*` en CSS heredado es legado — puede eliminarse (ver también [[Normalize]] para baselines de reset).

## Relacionadas

- [[Cuando usar GRID o Flexbox]] — la decisión entre unidimensional y bidimensional
- [[CSS Grid]] — el complemento para dos dimensiones
- [[Tipos de Display]] — `display: flex` dentro de la familia de displays
- [[CSS Introducción]] — cascada y fundamentos
- [[Especificidad]] — por qué las utilidades de flex ganan o pierden
- [[TailwindCSS - Sintaxis Básica]] — cómo se expresan estas propiedades como utilidades (`flex-*`, `justify-*`, `items-*`, `gap-*`)
- [[Animaciones]] — por qué animar `width` es caro y `transform` no
- [[Imágenes CSS]] — patrones de imagen dentro de flex
