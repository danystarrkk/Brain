---
tags:
  - desarrollo
  - frontend
  - background
tipo: concepto
estado: terminado
---

# Background

Las propiedades de `background` controlan el fondo de un elemento: color, imagen, degradados, repetición, tamaño, posición, recorte y modo de mezcla. Son **ocho sub-propiedades** que viven juntas en un shorthand — y entender cómo interactúan es la diferencia entre pelear con fondos y dominarlos. Esta nota cubre todas, con énfasis en `background-position`, la propiedad menos intuitive.

## El equipo completo

```css
.hero {
  background-color: #111827;
  background-image: url("hero.avif");
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center 30%;
  background-attachment: fixed;
}
```

| Sub-propiedad | Para qué sirve | Valores principales | Defecto |
|---|---|---|---|
| `background-color` | Color de fondo (por debajo de la imagen) | color — ver [[Colores CSS]] | `transparent` |
| `background-image` | Imagen o degradado (ver [[Degradados en CSS]]) | `url()`, `linear-gradient()`, `none` | `none` |
| `background-repeat` | Repetición de la imagen | `repeat`, `no-repeat`, `repeat-x`, `repeat-y`, `space`, `round` | `repeat` |
| `background-size` | Tamaño de la imagen | `auto`, `cover`, `contain`, longitudes, `%` | `auto` |
| `background-position` | Ubicación de la imagen | keywords, `%`, longitudes, 4 valores | `0% 0%` |
| `background-origin` | Caja de referencia de la posición | `padding-box`, `border-box`, `content-box` | `padding-box` |
| `background-clip` | Hasta dónde se dibuja | `border-box`, `padding-box`, `content-box`, `text` | `border-box` |
| `background-attachment` | Relación con el scroll | `scroll`, `fixed`, `local` | `scroll` |
| `background-blend-mode` | Mezcla entre capas de background | `normal`, `multiply`, `overlay`… | `normal` |
| `background` (shorthand) | Todas juntas en una línea | — | — |

> [!info] Color e imagen conviven, no compiten
> El `background-color` es la capa **siempre presente, al fondo de todo**; la imagen (o degradado) se pinta **encima**. Si la imagen falla o tiene transparencia, se ve el color. Por eso el patrón de legibilidad sobre foto es color de respaldo + overlay semitransparente, ver recetas más abajo.

## Ubicar la imagen: `background-position`

`background-position` recibe **1, 2 o 4 valores** y define desde qué punto se coloca la imagen de fondo:

```css
background-position: top;                              /* 1 valor: alineación */
background-position: top 50px;                        /* 2 valores: eje X e Y */
background-position: bottom 50px right 100px;         /* 4 valores: edge offset edge offset */
```

| Forma     | Ejemplo                              | Significado                                              |
| --------- | ------------------------------------ | -------------------------------------------------------- |
| 1 valor   | `center`, `top`, `left`              | Posición en ese eje; el otro queda centrado              |
| 2 valores | `top 50px` o `50% 50%`               | Eje Y y eje X (en ese orden para keywords mixtas)        |
| 4 valores | `bottom 50px right 100px`            | Borde vertical + desplazamiento, borde horizontal + desplazamiento |

Keywords válidas por eje: `left`, `center`, `right`, `top`, `bottom`. Combinaciones de dos keywords son equivalentes a porcentajes fijos:

| Keyword | % equivalente |
|---|---|
| `left` / `top` | `0%` |
| `center` | `50%` |
| `right` / `bottom` | `100%` |

> [!info] El punto de referencia se mueve hacia el lado contrario
> El valor que indicás es el **punto de anclaje**: `top` dice "pegá la parte superior de la imagen al borde superior del elemento", y los píxeles o porcentajes posteriores empujan la imagen **hacia el lado opuesto**. `right 100px` mueve la imagen 100px hacia la izquierda desde el borde derecho. Cualquier unidad sirve: `px`, `%`, `rem`, `vw`, `vh`, etc.

### Porcentajes: el comportamiento no obvio

Con porcentajes, `background-position: 100% 0%` equivale a `top right`, y el 50% central es `center`. La fórmula real es:

```
posición = (contenedor - imagen) × porcentaje
```

Un 20% alinea el 20% de la imagen con el 20% del contenedor — por eso `50%` centra exactamente. Consecuencias prácticas:

- Con imagen más chica que el contenedor, `100%` la pega al borde derecho/inferior; `0%` al izquierdo/superior.
- Con imagen **más grande** que el contenedor, el signo se invierte en la práctica: `100%` muestra el borde derecho de la imagen, `0%` el izquierdo — por eso para "focalizar" un rostro en un `cover` se usan valores como `center 25%`.
- Valores negativos o > 100% son válidos: permiten mover la imagen más allá de los bordes (útiles para sprite animations).

### El truco del fondo fijo centrado (parallax clásico)

```css
.parallax {
  background-image: url("paisaje.jpg");
  background-attachment: fixed;   /* la imagen no se mueve con el scroll */
  background-position: center;
  background-size: cover;
  background-repeat: no-repeat;
}
```

Con `fixed`, el sistema de coordenadas deja de ser el elemento y pasa a ser el **viewport**: `center` centra respecto de la pantalla, no del elemento. Ese cambio de marco es lo que produce el efecto de profundidad.

## `background-size`: control del escalado

```css
.foto {
  background-size: cover;         /* llena todo, recorta lo que sobre */
  background-size: contain;       /* cabe entera, puede dejar bordes */
  background-size: 50% auto;      /* ancho al 50%, alto proporcional */
  background-size: 300px 200px;   /* tamaño exacto (deforma si no mantén proporción) */
  background-size: 100% 100%;     /* estira hasta llenar la caja */
}
```

| Valor | Comportamiento | Efecto |
|---|---|---|
| `auto` (defecto) | tamaño intrínseco de la imagen | sin escalar |
| `cover` | escala para cubrir **toda** la caja, recortando lo que sobra | sin bordes, puede perder contenido en los extremos |
| `contain` | escala para que **quepa entera** | nada recortado, puede haber huecos |
| `%` | relativo a la caja de posición (no a la imagen) | `100% 100%` = estirar |
| longitudes | tamaño fijo | útil para sprites y tiles |

> [!tip] `cover` + `center 30%`: la fórmula de los heroes
> `cover` recorta, y el punto de recorte lo decide `background-position`. Para fotos de personas, el tercio superior suele ser donde está la cara: `background-position: center 30%` la conserva visible. La combinación `cover`/`contain` y cuándo elegir cada una para imágenes reales vive en [[Imágenes CSS]].

## `background-repeat`: tiles sin costuras

```css
background-repeat: repeat;      /* en ambos ejes (defecto) */
background-repeat: repeat-x;    /* solo horizontal */
background-repeat: repeat-y;    /* solo vertical */
background-repeat: space;       /* repite, pero distribuye el espacio sobrante entre copias (sin cortar) */
background-repeat: round;       /* escala las copias para que enteras cubran la caja (sin cortar) */
background-repeat: no-repeat;   /* una sola copia */
```

También admite dos valores (`repeat no-repeat` = eje X, eje Y). La diferencia sutil:

- `space`: las copias mantienen su tamaño; el **espacio vacío** se reparte entre ellas (puede dejar márgenes en los bordes).
- `round`: las copias se **redimensionan** (ligeras deformes) para que la última cierre justa contra el borde.

Para texturas y patrones, el secreto no es CSS sino el asset: un tile bien cortado de 200×200 que repita con `repeat` es invisible; un tile mal cortado produce costuras evidentes.

## `background-origin` vs `background-clip`: dos cajas distintas

Muchas confusiones vienen de mezclarlas:

- **`background-origin`**: desde qué caja se **posiciona** la imagen (su sistema de coordenadas).
- **`background-clip`**: hasta qué caja se **dibuja** (qué porción de la caja visible pinta).

```css
.borde-grosor {
  border: 12px dashed #3498db;
  background: url("textura.png") repeat;
  background-origin: content-box;  /* la textura arranca donde hay contenido */
  background-clip: content-box;    /* y no se cuela bajo el borde punteado */
}
```

Casos notables:

| Valor de `background-clip` | Efecto |
|---|---|
| `border-box` (defecto) | pinta bajo el borde (el borde semi-transparente deja ver el fondo) |
| `padding-box` | corta en el inicio del padding |
| `content-box` | solo detrás del contenido |
| `text` | **el fondo se convierte en relleno del texto** — requiere `color: transparent` |

```css
.titulo-degradado {
  background: linear-gradient(90deg, #3498db, #8b5cf6);
  -webkit-background-clip: text;   /* prefijo todavía necesario en Safari */
  background-clip: text;
  color: transparent;
}
```

## Capas múltiples: un stack de fondos

`background-image` (y las demás) aceptan **listas separadas por coma**: cada ítem es una capa. La primera capa de la lista se pinta **encima** de las demás.

```css
.overlay {
  background-image:
    linear-gradient(rgb(0 0 0 / 0.55), rgb(0 0 0 / 0.55)),  /* encima: oscurece */
    url("hero.avif");                                       /* debajo: la foto */
  background-size: cover, cover;
  background-position: center, center;
  background-repeat: no-repeat, no-repeat;
}
```

Reglas del sistema de capas:

- Cada capa puede ser `url()`, un gradiente o `none`.
- `background-color` **no es una capa**: es una sola capa final única detrás de todas.
- Las listas de `background-size`, `position`, `repeat`, `origin`, `clip`, `attachment` y `blend-mode` se emparejan por posición con las capas de `background-image`; si faltan valores, se repite la última secuencia.
- El z-order es fijo: la lista define la profundidad, no se puede cambiar (a diferencia de `box-shadow` o `z-index`).

## `background-attachment`: scroll, fixed y local

| Valor | Comportamiento |
|---|---|
| `scroll` (defecto) | la imagen se queda anclada al elemento: scrollea con la página |
| `fixed` | anclada al viewport: efecto parallax, sin moverse con el scroll |
| `local` | anclada al **contenido** del elemento: si el elemento scrollea internamente, la imagen scrollea con él |

`local` es el valor olvidado: en un panel con `overflow: auto`, `background-attachment: local` hace que el fondo acompañe al texto al hacer scroll interno — como un documento de papel.

> [!warning] `background-attachment: fixed` en móvil
> En iOS y muchos móviles `fixed` no funciona (se comporta como `scroll`) porque rompe el rendimiento del scroll. Usá parallax con cuidado o con JavaScript controlado.

Además `fixed` + `cover` tiene un efecto secundario: como el sistema de coordenadas es el viewport, la imagen cubre la **pantalla completa**, no el elemento — en elementos bajos se ve una porción diferente de la foto.

## `background-blend-mode`: mezclar capas de fondo

```css
.textura-marca {
  background-image:
    url("ruido.png"),
    linear-gradient(135deg, #3498db, #8b5cf6);
  background-blend-mode: overlay, normal;
}
```

`background-blend-mode` mezcla **las capas de background entre sí y con el background-color** (mismos modos que `mix-blend-mode`: `multiply`, `screen`, `overlay`, `luminosity`…). Diferencia clave con `mix-blend-mode`: este último mezcla el elemento **entero** con lo que hay detrás en la página (ver [[Sombras CSS]]/[[Degradados en CSS]] para patrones combinados); el de background no sale de la caja del elemento.

## El shorthand y sus trampas

```css
/* shorthand equivalente a todas las anteriores */
.hero {
  background: #111827 url("hero.avif") no-repeat fixed center 30% / cover;
}
```

- El orden importa: color primero, luego imagen, luego repeat/attachment/position en ese orden canónico, y **`position / size`** separados por barra.
- **Cuidado con `background` shorthand**: reinicia **todas** las sub-propiedades que no se mencionen (por ejemplo, `background-repeat` vuelve a `repeat`, `background-attachment` a `scroll`). Si vas a usar shorthand, declara todo lo que necesites en esa misma línea.
- Shorthands parciales: `background-position` y `background-size` también pueden venir solos como listas de capas; el shorthand completo es el único que resetea todo.
- `background: none` borra también el color si estaba en otra declaración del mismo bloque/regla.

## Recetas prácticas

### 1. Hero con overlay legible (el patrón de siempre)

```css
.hero {
  min-height: 70vh;
  display: grid;
  place-items: center;
  color: white;
  background-image:
    linear-gradient(rgb(17 24 39 / 0.6), rgb(17 24 39 / 0.6)),
    url("hero.avif");
  background-size: cover;
  background-position: center 35%;
  background-repeat: no-repeat;
}
```

### 2. Textura sutil de ruido con SVG data-URI (cero requests)

```css
.ruido {
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='120' height='120'%3E%3Cfilter id='n'%3E%3CfeTurbulence baseFrequency='0.8'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.08'/%3E%3C/svg%3E");
}
```

### 3. Patrón de cuadrícula tipo paper

```css
.cuadricula {
  background-image:
    linear-gradient(#e5e7eb 1px, transparent 1px),
    linear-gradient(90deg, #e5e7eb 1px, transparent 1px);
  background-size: 24px 24px;
}
```

### 4. Hover que "enfoca" un fondo

```css
.thumb {
  background: url("miniatura.avif") center / cover no-repeat;
  transition: background-size 0.4s ease;  /* cuidado: escalar imagen es pintado, no barato */
}
```

## Formatos de imagen recomendados

Para fondos pesados, usá formatos modernos como AVIF o WebP en el `background-image` — ver [[Formatos Avif y Webp]] y [[Imágenes CSS]] para la estrategia completa con imágenes. Dado que el fondo no es contenido, la estrategia de calidad comprimida (calidad ~70, dimensiones 2× del display) aplica con más fuerza todavía.

## Accesibilidad y modo de color forzado

- Un `background-image` **decorativo** está bien; si la imagen portaba **información**, estás duplicando contenido no-textual — proveé alternativa.
- **Contraste**: el texto sobre foto necesita overlay o sombra (mínimo 4.5:1, ver [[Colores CSS]]). DevTools muestra el ratio en el panel de *contrast*.
- **Forced colors mode** (Windows high contrast, `forced-colors: active`): el navegador **elimina los `background-image`** y fuerza colores del sistema. No dependas de fondos para transmitir estados; los bordes y el texto sobreviven, las fotos no.

```css
@media (forced-colors: active) {
  .hero { border: 1px solid CanvasText; } /* asegurar algún marcador visible */
}
```

## Rendimiento

- Imágenes de fondo **grandes y pocas** rinden mejor que muchas chicas: cada `url()` es una descarga; cada capa, un repintado.
- `background-attachment: fixed` fuerza repintado de la página en cada scroll en varios navegadores — en móviles es directamente ignorado.
- Animar `background-position` o `background-size` es **pintado por frame** (aceptable en pocos elementos); para mover fondos, a veces conviene convertirlos a elemento y animar `transform` — ver [[Animaciones]].
- Los degradados grandes con `blur` o `background-blend-mode` en elementos full-screen pueden costar frames en dispositivos bajos; medí con la pestaña *Performance*.

## Errores comunes

1. **`background-repeat: repeat` por defecto**: una foto de hero sin `no-repeat` se ve tileada. Síntoma clásico: "la imagen se repite".
2. **Olvidar `background-size: cover`**: la imagen se dibuja a tamaño intrínseco y "no llena".
3. **`cover` vs `contain` intercambiados**: `contain` deja huecos (esperabas llenar); `cover` recorta (esperabas ver todo).
4. **Posicionar sin saber la fórmula de %**: creer que `30%` mueve "30px"; en realidad alinea punto de imagen y punto de caja a ese %.
5. **Shorthand al final que resetea todo**: escribir las sub-propiedades bonitas y cerrar con `background: url(...)` — pum, todo lo anterior a `initial`.
6. **Ruta de `url()` resuelta contra el CSS**: `url("img.png")` busca relativo al archivo CSS, no al HTML — ver [[Anatomía de CSS]].
7. **Imagen de fondo con texto encima sin overlay**: fallo de contraste garantizado.
8. **`fixed` esperando parallax en iOS**: se comporta como `scroll` — no es bug tuyo.
9. **Capas sin alinear listas**: 2 imágenes pero 1 `background-size` — la capa 2 hereda la repetición de la lista y "se ve raro".

## Cheatsheet — propiedades de background

| Propiedad | Valores | Defecto |
|---|---|---|
| `background-color` | cualquier color | `transparent` |
| `background-image` | `none`, `url()`, gradiente, lista por capas | `none` |
| `background-repeat` | `repeat`, `repeat-x`, `repeat-y`, `no-repeat`, `space`, `round` (+ 2 valores) | `repeat` |
| `background-size` | `auto`, `cover`, `contain`, longitudes, `%` (lista por capas) | `auto` |
| `background-position` | keywords, `%`, longitudes, sintaxis de 4 valores (lista por capas) | `0% 0%` |
| `background-origin` | `padding-box`, `border-box`, `content-box` | `padding-box` |
| `background-clip` | `border-box`, `padding-box`, `content-box`, `text` | `border-box` |
| `background-attachment` | `scroll`, `fixed`, `local` | `scroll` |
| `background-blend-mode` | modos de mezcla (`normal`, `multiply`, `screen`…) | `normal` |
| `background` | shorthand (respetar orden: color image repeat attachment position/size) | — |

## Relacionadas

- [[Degradados en CSS]] — `linear-gradient()` y `radial-gradient()` como fondo
- [[Imágenes CSS]] — etiqueta `img` vs fondo CSS
- [[Formatos Avif y Webp]] — formatos comprimidos para fondos
- [[Colores CSS]] — el `background-color` y todos los formatos de color
- [[Animaciones]] — combinar fondos con animación
- [[Box Model]] — de qué caja cuelga el fondo (padding-box/border-box)
