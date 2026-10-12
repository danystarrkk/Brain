---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Imágenes CSS

Desde CSS podemos insertar imágenes, y el uso más común es como **fondo** de un elemento con `background-image`. Pero el sistema de fondos es completo: **capas múltiples**, control de repetición, origen de coordenadas, anclaje al scroll y selección por resolución. Esta nota es la referencia del subsistema entero `background-*`.

> [!info] El modelo mental: capas apiladas
> `background-*` no pinta "una imagen": pinta **una o varias capas**, de la primera (arriba) a la última (abajo), sobre el `background-color` final. Cada capa puede ser imagen, degradado (ver [[Degradados en CSS]]) o SVG. Casi todas las propiedades `background-*` aceptan **una lista de valores, uno por capa** — el orden importa.
> El sistema de capas explica por qué `background: #111 url(a.jpg) url(b.jpg)` es válido: a.jpg tapa a b.jpg.

## `background-image` y la ruta

```css
body {
  background-image: url("ruta/de/la/imagen.jpg");
}
```

La ruta se resuelve **desde la ubicación de la hoja de CSS**, no desde el HTML: si el CSS vive en `css/estilos.css` y la imagen en `img/fondo.jpg`, la ruta correcta es `url("../img/fondo.jpg")` — hay que retroceder un nivel con `../`. Entre comillas simples o dobles; sin comillas solo si la ruta no tiene espacios. Misma regla que para las fuentes en `@font-face` (ver [[Fuentes Externas]]).

> [!warning] `bacground-image` no existe
> El nombre de la propiedad es `background-image` (con "k" no hay nada en CSS). Un typo acá no tira error de sintaxis de la hoja entera, pero la imagen simplemente no aparece.

Valores admitidos: `url()`, `image-set()`, `linear-gradient(...)` y compañía, `element()` (Firefox), `cross-fade()`, o `none`.

## Repetición y tamaño: las dos propiedades clave

Por defecto, la imagen de fondo **se repite** en mosaico (tile) y mide lo que mide. Si no cubre todo el espacio, aparece el patrón repetido; si la repetición sobra y el tamaño no cubre, se ven los bordes. Las dos propiedades lo corrigen:

```css
body {
  background-image: url("ruta/de/la/imagen.jpg");
  background-repeat: no-repeat; /* por defecto: repeat — evita el mosaico */
  background-size: cover;       /* estira hasta cubrir todo el área */
}
```

| Valor de `background-size` | Comportamiento                                                    |
| -------------------------- | ----------------------------------------------------------------- |
| `auto` (defecto)           | Tamaño natural de la imagen                                       |
| `cover`                    | Cubre **todo** el área manteniendo proporción (recorta lo que sobra) |
| `contain`                  | Muestra la imagen **completa dentro** del área (puede sobrar fondo) |
| `50%` / `300px`            | Tamaño explícito, admite cualquier unidad                         |
| `100% 50%`                 | Dos valores: ancho y alto por separado (deforma si se fuerza)      |
| `auto 200px`               | Un valor `auto` conserva la proporción original                   |

Cómo decide `cover`/`contain`: comparan el **aspect ratio** de la imagen con el del área. `cover` escala hasta que el lado más corto cubre (recorta el excedente); `contain` escala hasta que el lado más largo cabe (deja bandas en el lado corto). Por eso el dúo clásico:

```css
.hero {
  background-image: url("hero.avif");
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center; /* casi siempre junto a cover */
}
```

> [!tip] `cover` + `center` es el dúo del hero
> `background-size: cover` casi siempre va con `background-position: center` para que el recorte sea parejo a ambos lados. Para retratos, subí el foco: `background-position: center 30%` (la cara no se corta).

## Repetición fina: `background-repeat`

| Valor | Comportamiento |
|---|---|
| `repeat` (defecto) | Mosaico en ambas direcciones; el tile se **recorta** en los bordes si no cabe exacto |
| `no-repeat` | Una sola copia |
| `repeat-x` / `repeat-y` | Solo horizontal / solo vertical |
| `space` | Copias completas repartidas con espacio **igual** entre ellas (sin recortar) |
| `round` | Las copias se **escalan** para que enteras llenen el área (deforma levemente) |

`space` y `round` solo tienen efecto cuando `background-size` no es `auto`… en la práctica se usan con tiles de patrones:

```css
.raya {
  background-image: url("raya-8px.png");
  background-size: 8px 8px;
  background-repeat: round; /* el patrón llena sin cortar jamás */
}
```

## Posicionamiento: `background-position`

Acepta keywords (`left top`, `center`, `bottom right`), longitudes, porcentajes y mezclas; con dos capas, una posición por capa:

```css
.banner {
  background-image: url("marca.svg");
  background-position: right 1rem top 1rem; /* offset desde dos bordes */
}

.dos-capas {
  background-image: url("arbol.png"), url("cielo.jpg");
  background-position: bottom 2rem center, center; /* primero = capa de arriba */
  background-repeat: no-repeat, no-repeat;
}
```

### La matemática del porcentaje (clave mental)

Un `background-position: X%` **no** es "X% desde la izquierda": es *colocar el punto X% de la imagen sobre el punto X% del área*. Fórmula exacta del desplazamiento:

```
offset = (área − imagen) × porcentaje
```

- `0%` → imagen pegada al inicio; `100%` → pegada al final; `50%` → centrada (por eso `center` = `50% 50%`).
- Con imagen **más grande** que el área, el signo se invierte: `100%` muestra la esquina inferior derecha, y `background-position: 100% 100%` es el pan del parallax clásico.
- Con longitudes, en cambio, el valor es un offset directo desde el origen (top-left): `background-position: 20px 40px`, incluidos negativos para "empujar" la imagen fuera del área.

> [!info] `background-origin` define desde dónde se miden las coordenadas
> Por defecto las posiciones parten del **padding-box** (no del borde). `background-origin: content-box` las re-basa al área de contenido; `border-box` al borde. Casi nunca se toca, pero explica por qué un `background-position: 0 0` no pega la imagen al borde exterior si hay `border` de 10px.

## Capas múltiples: el recurso oculto

Todas las propiedades aceptan listas; la primera capa es la más alta. Regla de oro: **cada capa debe declarar todas sus propiedades** (repetición, posición, tamaño) — si una lista es más corta, se **ciclan** los valores (como `box-shadow`):

```css
.ticket {
  background-image:
    url("sello.png"),                 /* arriba */
    linear-gradient(#fff, #eee),      /* abajo */
    url("papel.jpg");                 /* base */
  background-repeat: no-repeat, no-repeat, repeat;
  background-position: right bottom, center, center;
  background-size: 120px, cover, cover;
}
```

El orden de capas es el z-index del fondo: sin numeración, solo el orden de la lista.

## `background-attachment`: scroll, fixed y local

| Valor | Referencia del movimiento | Caso de uso |
|---|---|---|
| `scroll` (defecto) | La imagen queda fija respecto **del elemento**: al scrollear la página, sale con él | Default |
| `fixed` | La imagen se ancla a la **viewport**: el elemento se mueve y la imagen queda — efecto parallax | Heroes, texturas de fondo |
| `local` | La imagen scrollea con el **contenido del propio elemento** (div con scroll interno) | Paneles con overflow propio |

```css
.parallax {
  background-image: url("montana.jpg");
  background-attachment: fixed;
  background-size: cover;
}
```

> [!warning] `background-attachment: fixed` es tóxico en móvil
> En iOS Safari y Android el fixed provoca repaints costosos en cada scroll (y comportamiento errático al redimensionar). Receta moderna: `background-attachment: fixed` solo en `@media (min-width: 1024px) and (hover: hover)`, o lograr el efecto con `transform` + `position: sticky` (ver [[Efectos de Scroll]] y [[Animaciones]]).

## La sombra del `background` shorthand

Todas las propiedades de fondo entran en un solo atajo (orden libre, con `/` entre posición y tamaño):

```css
.hero {
  background: #111 url("hero.avif") center / cover no-repeat;
  /*           color  imagen           pos / tamaño  repetición */
}
```

> [!warning] El atajo resetea lo que no menciones
> Escribir `background:` sin los valores que necesites **borra los valores por defecto** posteriores (por ejemplo, `background-color` queda `transparent`). En proyectos serios se prefiere usar las propiedades largas.

`background` (color+imagen) vs `background-image` (solo imagen): el primero resetea `background-position/size/repeat/attachment/color`; por eso `background: url(x.jpg)` destruye un `background-size: cover` previo. Gramatura del atajo completo: `[color] [position] / [size] [repeat] [attachment] [origin] [clip]`.

## `background-clip`: hasta dónde pinta

`border-box` (defecto), `padding-box`, `content-box`, `text` (¡el texto se rellena con el fondo!). Los usos modernos:

```css
.titulo-degradado {
  background: linear-gradient(90deg, #6366f1, #ec4899);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent; /* el degradado tiñe las letras */
}
```

```css
/* Receta "borde de degradado": padding-box recorta el relleno */
.card-borde {
  border: 1px solid transparent;
  background:
    linear-gradient(#fff, #fff) padding-box,
    linear-gradient(135deg, #6366f1, #ec4899) border-box;
}
```

Ver [[Sombras CSS]] y [[Degradados en CSS]] para el resto del arsenal de bordes.

## Formatos: usá formatos modernos

Elegí formatos comprimidos: **WebP** y **AVIF** ofrecen mucha mejor calidad/ peso que JPG o PNG, con soporte universal hoy en día. Detalles y `image-set()` para respaldo por formato en [[Formatos Avif y Webp]].

```css
.banner {
  background-image: image-set(
    url("banner.avif") type("image/avif"),
    url("banner.webp") type("image/webp"),
    url("banner.jpg") type("image/jpeg")
  );
}
```

### `image-set()`: una capa, varias resoluciones o formatos

El equivalente de `srcset` pero para fondos. Acepta `type()` (negociar formato) y `resolution` (negociar densidad de píxeles):

```css
.logo {
  background-image: image-set(
    url("logo@1x.png") 1x,
    url("logo@2x.png") 2x,       /* pantallas retina */
    url("logo@3x.avif") 3x type("image/avif")
  );
}
```

Unidades de resolución para `image-set` y media queries: `dppx` (1 = 1 píxel CSS por píxel físico), `dpi` (96 = 1dppx), `dpcm`. Prefijo histórico `-webkit-image-set` para Safari viejo; sin `type()` no negocia formato — mejor usar el sin prefijo con `type()` explícito, degradando a la primera entrada compatible.

## SVG en fondos: los dos gotchas

```css
.icono-fondo {
  background-image: url("icono.svg");          /* SVG externo: dimensionado por background-size */
  background-size: 24px 24px;
}

/* SVG inline/data-URI puede heredar currentColor... */
.icono-hereda {
  background-image: url('data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" fill="currentColor" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10"/></svg>');
}
```

1. **El `#` del color se URL-encripta**: un SVG con `fill="#ff0000"` en `url()` debe ir como `%23ff0000`; si no, el navegador corta la ruta en `#` (lo trata como fragmento) y la imagen no carga — síntoma clásico: funciona en el editor, no en el navegador.
2. **`currentColor` no funciona en SVG externo**: el SVG de fondo no hereda el contexto del documento. Solo funciona en SVG **inline** (data-URI con `fill="currentColor"`, o inline en el HTML). Para iconos dinámicos, mejor `<img>`/inline SVG (ver [[Html]]).

## ¿Fondo (`background-image`) o `<img>`?

| Criterio             | `background-image`                       | `<img>`                                  |
| -------------------- | ---------------------------------------- | ---------------------------------------- |
| Semántica / SEO      | Decorativa                              | Contenido real                           |
| Accesibilidad        | Requiere aria si aporta información      | Tiene `alt` obligatorio                  |
| Responsive automático | `cover`/`contain` lo resuelve           | Necesitás `srcset` y `sizes`             |
| Lazy loading         | Por CSS (limitado)                      | Nativo con `loading="lazy"`              |
| Art direction        | Cambiar la imagen con media queries      | `<picture>` + `<source media>`           |
| LCP                  | El fondo **no** se prioriza como LCP candidate fácil | Sí se puede preloading (`fetchpriority="high"`) |

Regla: **si la imagen comunica algo, es `<img>`; si es decoración del diseño, es fondo.** Ver [[Lazy Loading]] para diferir la carga de imágenes.

Para `<img>` ya dentro del flujo, el equivalente de `cover`/`contain` son `object-fit` y `object-position`:

```css
.avatar {
  width: 96px;
  height: 96px;
  object-fit: cover;      /* idéntico a background-size: cover */
  object-position: top;   /* idéntico a background-position */
  border-radius: 50%;
}
```

> [!info] Fondos degradados conviven gratis
> `background-image` acepta degradados además de imágenes: `background: linear-gradient(...)` — ver [[Degradados en CSS]].

## Recetas

### Patrón de textura sutil sobre color

```css
.panel {
  background-color: #0f172a;
  background-image: url("ruido-512.png");
  background-repeat: repeat;
  background-size: 512px;
  opacity: 1; /* la textura va en su propia capa con low-contrast */
}
```

### Overlay oscuro legible (imagen + velo, en una propiedad)

```css
.hero-legible {
  background-image:
    linear-gradient(rgba(0, 0, 0, 0.55), rgba(0, 0, 0, 0.55)),
    url("hero.avif");
  background-size: cover, cover;
  background-position: center, center;
  background-repeat: no-repeat;
}
```

### Patrón de líneas 1px sin imágenes (degradado tile)

```css
.reticula {
  background-image:
    linear-gradient(#e5e7eb 1px, transparent 1px),
    linear-gradient(90deg, #e5e7eb 1px, transparent 1px);
  background-size: 24px 24px;
}
```

## Accesibilidad

- **Fondos decorativos**: el `background-image` no tiene alt; si transmite información, o es `<img>`, o el elemento necesita `role="img"` + `aria-label`.
- **Contraste del texto sobre imagen**: la imagen puede cambiar; el velo (`linear-gradient` encima) fija el contraste — ver [[Colores CSS]] para los ratios.
- **`prefers-reduced-motion`**: el parallax con `background-attachment: fixed` mareá; apagalo:

```css
@media (prefers-reduced-motion: reduce) {
  .parallax { background-attachment: scroll; }
}
```

- Zoom de texto (200%): `background-size: cover` escala bien; los fondos en px fijo no.

## Rendimiento

- Fondos grandes sin comprimir son el clásico recurso fantasma: 4 MB de hero en JPG 4000×2000 — serví AVIF/WebP responsive con `image-set` (ver [[Formatos Avif y Webp]]).
- Animar `background-position` dispara **repaint** en cada frame; si el efecto es de desplazamiento, preferí mover un pseudo-elemento con `transform` (composición, ver [[Animaciones]]).
- `background-attachment: fixed` en móvil = repaint de la capa entera en cada scroll (ver advertencia arriba).
- Fondos no participan bien del cache de fuentes de CLS/LCP: si la imagen **es** el contenido del primer viewport, usá `<img fetchpriority="high">` en vez de fondo.

## Errores comunes

1. **Ruta relativa al HTML**: la ruta de `url()` se resuelve desde el CSS — el `../` más famoso del oficio.
2. **Typo `bacground-image`**: silencioso, sin imagen y sin error.
3. **`background:` borrando el size/position anteriores**: usar propiedades largas.
4. **Listas de capas desfasadas**: si hay 3 imágenes, cada propiedad acepta hasta 3 valores; si cicla alguna, revisá que todas las listas tengan la misma longitud.
5. **`#` sin codificar en SVG data-URI**: corta la ruta; usar `%23`.
6. **`cover` en retratos sin `background-position` ajustado**: recorta la cara; subir el foco.
7. **`fixed` en iOS**: jank garantizado; media query de escritorio o transform.
8. **Esperar `contain` "llenando"**: `contain` deja bandas — eso es diseño, no bug; si no querés bandas, es `cover` con recorte.

## Cheatsheet — propiedades de fondo

| Propiedad | Valores clave | Defecto | Nota |
|---|---|---|---|
| `background-image` | `url()`, `image-set()`, gradientes, `none` | `none` | Lista = capas; primera arriba |
| `background-color` | color | `transparent` | Queda debajo de todas las capas |
| `background-repeat` | `repeat`, `no-repeat`, `repeat-x/y`, `space`, `round` | `repeat` | — |
| `background-size` | `auto`, `cover`, `contain`, largo, %, dos valores | `auto` | `cover` recorta, `contain` bandas |
| `background-position` | keywords, %, largo, mixto (`right 1rem top`) | `0% 0%` | % = (área − imagen) × % |
| `background-origin` | `padding-box`, `content-box`, `border-box` | `padding-box` | Origen de posiciones |
| `background-clip` | `border-box`, `padding-box`, `content-box`, `text` | `border-box` | `text` tiñe letras |
| `background-attachment` | `scroll`, `fixed`, `local` | `scroll` | `fixed` = parallax, mal en móvil |
| `background` (shorthand) | color + todo lo anterior (`pos / size`) | — | **Resetea lo no mencionado** |
| `image-set()` | `url() 1x`, `type("image/avif")` | — | Equivalente a `srcset` en fondos |
| Unidades de resolución | `dppx`, `dpi`, `dpcm` | `1dppx = 96dpi` | Media queries e `image-set` |

## Compatibilidad

El subsistema `background-*` completo (capas múltiples, `cover/contain`, `origin/clip`) es universal desde 2015+. Datos para leer CSS viejo: IE8 no tenía `cover/contain` ni capas múltiples; `-webkit-image-set` es el prefijo histórico de Safari; `background-clip: text` necesita `-webkit-` todavía en algunos motores (dejar ambas líneas). `image-set` con `type()` se estabilizó en 2021–2023 en los tres motores.

## Relacionadas

- [[Background]] — `background-position` en detalle y el resto de propiedades
- [[Degradados en CSS]] — pintar fondos sin archivos
- [[Formatos Avif y Webp]] — qué formato servir y cómo detectarlo
- [[Lazy Loading]] — no cargar imágenes hasta que se necesiten
- [[Animaciones]] — por qué animar `background-position` es caro y `transform` no
- [[Efectos de Scroll]] — parallax sin `background-attachment: fixed`
- [[Box Model]] — padding-box/border-box/content-box: las cajas que miden el fondo
- [[Fuentes Externas]] — mismo patrón de `url()` relativa al CSS
- [[Html]] — cuándo la imagen merece ser `<img>` con `srcset`
