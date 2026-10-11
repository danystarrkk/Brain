---
aliases: ["Box Model", "Modelo de caja", "box-sizing"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Medidas CSS]]"
  - "[[Medidas Relativas]]"
  - "[[Tipos de Display]]"
  - "[[Especificidad]]"
  - "[[Flexbox]]"
  - "[[CSS Grid]]"
  - "[[Sombras CSS]]"
  - "[[Tipos de Posiciones]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/Box_model"
---

# Box Model

En CSS **todo es una caja**: párrafos, imágenes, botones, spans vacíos. El modelo de caja describe que el tamaño final de un elemento se calcula con 4 capas concéntricas:

1. **Contenido** — el texto o la imagen, con su `width`/`height`.
2. **Relleno (`padding`)** — espacio transparente dentro del borde.
3. **Borde (`border`)** — la línea que rodea el relleno.
4. **Margen (`margin`)** — espacio vacío *fuera* de la caja, que separa a otros elementos.

![[Pasted image 20250124165421.png]]

> [!info] El modelo completo de CSS2.1
> El estándar clásico suma dos capas más alrededor de las que el diagrama simplificado no muestra: el **área de línea** (*inline box*, el espacio que la letra ocupa incluyendo ascendentes/descendentes) y el **área de contenido** propiamente dicha. El diagrama útil para maquetar es el de 4 capas; el detalle tipográfico vive en propiedades como `line-height`.

## El problema del box-sizing por defecto

Por defecto (`content-box`), `width` y `height` afectan **solo al contenido**: padding y border se *suman* encima. Si te piden un elemento de 200px de ancho total, tenés que restar a mano el padding y el borde:

```css
/* Con content-box: 200px de ancho + 2×20px padding + 2×2px border = 244px reales */
.caja-mala {
  width: 200px;
  padding: 20px;
  border: 2px solid black;
}

/* Con border-box: width = ancho TOTAL visible */
.caja-buena {
  box-sizing: border-box;
  width: 200px; /* sigue midiendo 200px con todo el padding dentro */
  padding: 20px;
  border: 2px solid black;
}
```

| `box-sizing` | `width` mide | Ancho total resultante | Origen |
|---|---|---|---|
| `content-box` (defecto) | solo el contenido | `width + padding*2 + border*2` | CSS1, el histórico |
| `border-box` | contenido + padding + border | `= width` | propuesto por devs, estandarizado después |
| `padding-box` | contenido + padding (border aparte) | poco usado | intermedio raro |

> [!tip] Por qué `border-box` "ganó"
> Nadie piensa en tamaños de contenido: se piensa en tamaños **visibles** ("esta card mide 300px"). `border-box` hace que `width` sea el número que el diseñador escribió en el Figma, sin restar nada a mano. Casi todo el ecosistema (Tailwind, Bootstrap, resets) lo aplica globalmente.

## La solución canónica de `border-box`

Gracias a un desarrollador de Chrome (Paul Irish) que publicó lo [siguiente](https://www.paulirish.com/2012/box-sizing-border-box-ftw/), se puede aplicar `border-box` a **todo el documento** de una sola vez:

```css
html {
  box-sizing: border-box;
}
*,
*::before,
*::after {
  box-sizing: inherit;
}
```

Desglose del truco:

- Se declara `border-box` en `html` (la raíz del documento).
- El selector universal `*` selecciona **todos** los elementos, y con `box-sizing: inherit` cada uno toma el valor de su padre — así todo el árbol queda en `border-box` sin repetir la propiedad.
- Las comas separan selectores distintos a los que se aplican **las mismas declaraciones** — ver [[Anatomía de CSS]].
- `::before` y `::after` son **pseudo-elementos**: cajas generadas por CSS que no existen en el HTML pero que también necesitan el `border-box`.

> [!tip] Práctica moderna
> Hoy casi todo framework (Tailwind, Bootstrap) y la mayoría de resets incluyen este snippet. Si usás [[Normalize]] o un reset moderno, revisá que ya lo traiga. La ventaja del `inherit` es poder volver a `content-box` en un subárbol con una sola regla: `.email { box-sizing: content-box; }` — útil porque los editores de email asumen `content-box`.

## Cómo se calcula el ancho total

```css
/* content-box */
.ancho-total = width + padding*2 + border*2

/* border-box */
.ancho-total = width   /* el padding y el border viven dentro */
```

El `margin` **nunca** cuenta para el ancho del elemento: es espacio entre cajas. Con `border-box`, si sumás padding que supera el `width`, el contenido queda con ancho 0 o negativo (se recorta a 0): `width: 100px; padding: 80px;` deja 20px... y con más padding, cero visible.

## Anchos y alturas: `auto`, keywords y sizing keywords

El `width`/`height` no son solo números. Los valores de sizing determinan **quién gana** la pelea entre contenido y caja:

| Valor | Significado | Dónde brilla |
|---|---|---|
| `auto` | el contenido decide (bloques: llenan la línea; inline: su tamaño intrínseco) | comportamiento por defecto |
| `min-content` | el ancho mínimo sin romper el contenido (la palabra más larga) | sidebars, grid tracks |
| `max-content` | el ancho del contenido sin envolver líneas | texto en una sola línea |
| `fit-content` | `min(max(min-content, disponible), max-content)` | el ancho "natural" con límite |
| `%` | relativo a la caja del **padre** (contenedor de bloque) | layouts fluidos |
| `length` (`px`, `rem`…) | tamaño fijo | — |

```css
.titulo-largo {
  width: fit-content;          /* que no estire la caja más que su texto */
  max-width: 100%;             /* pero que nunca desborde el padre */
}
```

> [!warning] El % de `height` necesita padre con alto definido
> `height: 50%` funciona **solo** si el padre tiene un `height` explícito (no `auto`). En bloques, el alto fluye desde el contenido hacia abajo, no desde el padre hacia abajo. Alternativas modernas: `aspect-ratio`, `min-height: 100vh`, o el sistema de *container queries*.

Los valores intrínsecos (`min-content`…) son el fundamento de los tracks de [[CSS Grid]] (`minmax()`, `auto-fit`) y del `min-width: 0` de [[Flexbox]] — ver "Errores comunes" más abajo.

## `padding`: el relleno

```css
.caja {
  padding: 1rem 2rem;            /* shorthand como margin */
  padding-inline: 2rem;          /* lógico: según dirección de escritura */
}
```

Reglas y particularidades:

- **Los % en padding se calculan SIEMPRE sobre el ancho del padre**, nunca sobre la altura — útil para el truco de la caja con proporción fija (`padding-top: 50%` = altura la mitad del ancho, el *ratio box* histórico pre-`aspect-ratio`).
- `padding` **no admite valores negativos** (a diferencia del margin).
- El color del padding es el del `background` del elemento (el fondo se dibuja debajo del contenido *y* del padding, hasta el borde) — ver [[Background]].
- Un `<div>` vacío con padding visible dibuja una caja; un `<span>` inline con padding horizontal **no empuja** a los vecinos (los inline usan el modelo de línea) — ver [[Tipos de Display]].

## `border`: el borde

```css
.boton {
  border: 2px solid #3498db;        /* ancho estilo color (shorthand) */
  border-width: 1px 0 1px 0;        /* por caras */
  border-style: dashed;             /* solid, dotted, dashed, double, groove… */
  border-radius: 8px;               /* esquinas (2 valores: H / V) */
}
```

- `border-style` **tiene valor inicial `none`**: un `border-width` sin estilo no dibuja nada — la mitad de los "no me sale el borde" son esto.
- Los % en `border-radius` son elipses relativas a cada dimensión de la caja; el slash separa radio horizontal y vertical: `border-radius: 50% / 25%`.
- `border-image` permite nine-slice con imágenes — potente pero de uso raro y con trampas de compatibilidad.
- **El borde ocupa espacio**: cambiar `border: 1px` → `border: 2px` en hover desplaza el layout. Recetas anti-salto en "Errores comunes".

### `outline`: el borde que NO ocupa espacio

```css
:focus-visible {
  outline: 2px solid #3498db;
  outline-offset: 2px;   /* separación respecto del borde */
}
```

El `outline` se dibuja **encima** de todo, sin afectar layout ni scroll. Es el mecanismo canónico para el **foco accesible** (nunca uses `outline: none` sin reemplazo) — ver "Accesibilidad".

## `margin`: el margen (y sus rarezas)

```css
.elemento { margin: 0 auto; }        /* centrado horizontal de bloque */
.hijo      { margin-inline-start: auto; } /* empuja al final en flex/grid */
```

- **Centrado con `auto`**: en un bloque con ancho definido, `margin-left/right: auto` divide el espacio sobrante por igual. En [[Flexbox]] y [[CSS Grid]], `margin: auto` en el **item** centra dentro del track.
- **Valores negativos válidos**: `margin: -1rem` superpone cajas — truco clásico de *full-bleed* y de corrección de colapsado.
- Los % en margin son sobre el **ancho del padre** (igual que padding).

### El margin collapse (colapsado de márgenes)

Los márgenes verticales de bloques adyacentes **no se suman**: quedan el **mayor** de los dos. Ocurre en tres situaciones:

1. Adyacentes en vertical (hermanos): entre el `margin-bottom` de uno y el `margin-top` del otro → queda el mayor.
2. Padre e hijo vacíos (sin border/padding entre medio): los márgenes se fusionan a través.
3. Vacíos consecutivos: se colapsan todos en uno.

```css
/* Colapsado clásico: la "distancia real" entre p1 y p2 es 24px, no 40px */
p { margin-block: 16px; }
.vacio { margin-block: 24px; }   /* hermano vacío: fusiona con el del otro lado */
```

> [!info] Dónde NO colapsan los márgenes
> Ni en contenedores flex ni grid (los items viven en un contexto de formateo de flujo que lo impide) — ver [[Flexbox]] y [[CSS Grid]]. Ni cuando hay `padding`, `border` o contenido entre los márgenes. Por eso el "clearfix de márgenes" de siempre era: `overflow: hidden` en el padre.

## `overflow`: qué pasa cuando el contenido no cabe

```css
.panel {
  overflow: auto;        /* scroll solo cuando hace falta */
  scrollbar-gutter: stable; /* reserva la canaleta: evita el salto de layout */
}
```

| Valor | Comportamiento |
|---|---|
| `visible` (defecto) | el contenido desborda visible |
| `hidden` | se **recorta** sin scroll — y el elemento se vuelve contexto de scroll para JS |
| `clip` | recorta de verdad (sin scrollbar programática, sin contexto de scroll) |
| `auto` | scroll según necesidad |
| `scroll` | scroll siempre (aunque no haga falta) |
| `clip` + `overflow-clip-margin` | permite un margen visible antes de recortar |

También admite dos valores (`overflow-x`, `overflow-y`); **si uno de los dos no es `visible`, el otro se computa a `auto`** — causa misteriosa de scrollbars verticales no pedidas.

```css
.truncate {
  overflow: hidden;
  text-overflow: ellipsis;   /* "…" final */
  white-space: nowrap;       /* sin esto no hay una sola línea que recortar */
}
```

`overflow: hidden` en un ancestor **recorta** también a los descendientes posicionados (`position: absolute` con ancestor scroller) y a las sombras cercanas — la caja no crece para acomodar hijos flotantes; los recorta. `overflow: auto/scroll` crea un *scrolling box* al que se puede scrollear con teclado **solo si tiene foco** — ver accesibilidad.

## Inspeccionarlo en DevTools

Las DevTools de Chrome/Firefox muestran el diagrama del box model con los 4 valores en vivo. Es la forma más rápida de responder "¿por qué esto se sale de su contenedor?". El *computed* tab resuelve dudas de sizing: `width: auto` computado a 738.4px, `height: 0%` computado a `auto`, etc. Atajo: hover sobre el nodo en *Elements* resalta la caja sobre la página.

## Recetas prácticas

### 1. Caja con proporción fija (thumbnails 16:9 pre-`aspect-ratio` y su versión moderna)

```css
/* Moderna (recomendada) */
.thumb {
  aspect-ratio: 16 / 9;
  width: 100%;
  background: url("thumb.avif") center / cover no-repeat;
}

/* Clásica: padding-top como ratio (para contextos viejos o emails) */
.thumb-legacy {
  height: 0;
  padding-top: 56.25%;          /* 9/16 = 0.5625 */
  position: relative;
}
.thumb-legacy > * {
  position: absolute;
  inset: 0;
}
```

### 2. Tarjeta con borde que no mueve el layout en hover

```css
/* Estrategia A: el borde transparente de reserva */
.card-hover {
  border: 2px solid transparent;
  transition: border-color 0.2s ease;
}
.card-hover:hover { border-color: #3498db; }

/* Estrategia B: box-shadow como "borde" (no ocupa espacio, ver Sombras CSS) */
.card-shadow:hover { box-shadow: 0 0 0 2px #3498db; }
```

### 3. Full-bleed (romper el contenedor de medidas centradas)

```css
/* dentro de un contenedor max-width centrado, este bloque llega a los bordes */
.full-bleed {
  width: 100vw;
  margin-left: calc(50% - 50vw);
}
/* cuidado con scrollbars: 100vw incluye la canaleta → preferible
   width: 100% + margin con calc, o overflow-x: hidden en body */
```

### 4. Separación sin colapsar: gap donde aplica

En contexto [[Flexbox]]/[[CSS Grid]] usá `gap` en vez de márgenes entre items: no colapsa, no deja fantasma tras el último, y con `wrap` sigue funcionando.

## Accesibilidad

- **`outline: none` sin reemplazo es un bug de accesibilidad**: el foco de teclado deja de verse. Regla: eliminá el outline por defecto solo si ponés otro indicador (`box-shadow`, `border-color`, fondo) con contraste suficiente.
- **Áreas con `overflow: auto/scroll`** deben ser alcanzables con Tab (`tabindex="0"`) si contienen contenido scrolleable — un panel con scroll de solo ratón es una trampa para teclado y lectores de pantalla.
- **Contraste del borde**: si el borde comunica estado (input válido/inválido), no dependas solo del color — agregá ícono o texto ([[Colores CSS]]).
- Zonas táctiles: padding generoso en botones (mínimo ~44×44px) es padding de la caja, no del contenido.

## Rendimiento

- Cambiar `width`/`height`/`padding`/`margin` en transiciones **fuerza layout cada frame**; para hover/entradas usá `transform`/`opacity` (ver [[Animaciones]]).
- `overflow: hidden` no es gratis: convierte al elemento en contexto de scroll y puede habilitar rutas de repintado. Usarlo para truncar, no como "arreglo mágico" de layouts rotos.
- `100vw`/`100vh` en móvil: `vh` no contempla la barra del navegador (solucionado parcialmente con `dvh`/`svh`/`lvh` — ver [[Medidas Relativas]]).

## Errores comunes

1. **Restar padding a mano** olvidando `border-box` → cajas 40px más anchas de lo pedido.
2. **`border-width` sin `border-style`** → no se dibuja (el estilo por defecto es `none`).
3. **Hover con borde que salta el layout** → reservá el borde transparente desde el reposo o usá `box-shadow`.
4. **`height: 50%` con padre `auto`** → se computa a `auto`, no al 50%.
5. **Confundir padding con margin**: el padding es *dentro* (y pinta con el fondo); el margin separa *entre* cajas (y colapsa).
6. **`overflow: hidden` esperando que el hijo crezca el padre**: el hijo flotante/absoluto no estira al ancestor — lo tapa.
7. **Flex item que no desborda con scroll**: el `min-width: auto` por defecto impide el recorte — `min-width: 0` (bug #1 de Flexbox, ver [[Flexbox]]).
8. **`box-shadow` o `outline` recortados por un ancestor con `overflow: hidden`**: la sombra vive fuera de la caja; el recorte se la lleva.
9. **Márgenes que "no funcionan" entre hermanos**: es el colapsado — no se suman; usá `gap` o padding en el contenedor.
10. **`width: 100%` + `padding` sin `border-box`** → desborde lateral garantizado (el 100% no incluye el padding nuevo).

## Cheatsheet — propiedades del modelo de caja

| Propiedad | Valores | Defecto |
|---|---|---|
| `box-sizing` | `content-box`, `border-box` | `content-box` |
| `width` / `height` | `auto`, longitudes, `%`, `min-content`, `max-content`, `fit-content` | `auto` |
| `min-width` / `min-height` | longitudes, `%`, `auto` | `auto` |
| `max-width` / `max-height` | longitudes, `%`, `none` | `none` |
| `aspect-ratio` | `auto`, `n / m` | `auto` |
| `padding` (+ `-block/-inline`, por caras) | longitudes, `%` (sobre ancho del padre) | `0` |
| `border` (shorthand) | `estilo ancho color` por caras | `medium none currentColor` |
| `border-style` | `solid`, `dashed`, `dotted`, `double`, `groove`, `ridge`, `inset`, `outset`, `none`, `hidden` | `none` |
| `border-width` | longitudes por caras | `medium` |
| `border-radius` | longitudes, `%`, syntax `H / V` | `0` |
| `outline` (shorthand) | estilo, ancho, color | `auto` (varía) |
| `outline-offset` | longitudes (negativas válidas) | `0` |
| `margin` (+ lógico) | longitudes, `%`, `auto`; negativos válidos | `0` |
| `overflow` (+ `-x`, `-y`) | `visible`, `hidden`, `clip`, `auto`, `scroll` | `visible` |
| `overflow-clip-margin` | longitud (con `clip`) | `0` |
| `scrollbar-gutter` | `auto`, `stable` | `auto` |
| `overscroll-behavior` | `auto`, `contain`, `none` | `auto` |

## Relacionadas

- [[Medidas CSS]] — unidades para expresar tamaños de caja
- [[Medidas Relativas]] — por qué `rem` suele ser mejor que `px`
- [[Flexbox]] — distribución de cajas en una dimensión
- [[CSS Grid]] — distribución de cajas en dos dimensiones
- [[Tipos de Display]] — cómo `display` cambia el tipo de caja y su flow
- [[Tipos de Posiciones]] — cómo `position` saca la caja del flujo normal
- [[Sombras CSS]] — pintar fuera de la caja sin ocupar espacio
- [[Background]] — hasta dónde pinta el fondo dentro de la caja
- [[Anatomía de CSS]] — la sintaxis con la que se escriben todas estas reglas
