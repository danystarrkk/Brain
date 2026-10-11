---
aliases: ["WebP", "AVIF", "Picture"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Etiquetas]]"
  - "[[Lazy Loading]]"
  - "[[Imágenes CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Element/picture"
---

# Formatos AVIF y WebP

En nuestras páginas podemos servir imágenes en formatos modernos (**AVIF** y **WebP**), que comprimen mucho mejor que el clásico JPG/PNG sin pérdida visible de calidad. Como no todos los navegadores los soportan, la etiqueta **`<picture>`** nos permite declarar varias versiones y dejar que el navegador elija la que soporte, con JPG como respaldo universal.

| Formato | Compresión | Transparencia | Soporte (2026) |
|---|---|---|---|
| JPG | Base | No | Universal |
| PNG | Pesado | Sí | Universal |
| **WebP** | ~25–35% mejor que JPG | Sí | Todos los navegadores modernos |
| **AVIF** | ~50% mejor que JPG | Sí | Todos salvo IE/Edge viejo |

## Carga por formato (negociación progresiva)

El navegador recorre los `<source>` **de arriba hacia abajo** y usa el primero cuyo `type` soporte; el `<img>` final es el respaldo obligatorio:

```html
<picture>
  <source srcset="img/blog1.avif" type="image/avif">
  <source srcset="img/blog1.webp" type="image/webp">
  <img loading="lazy" src="img/blog1.jpg" alt="Imagen del blog" width="800" height="600">
</picture>
```

> [!warning] `loading="lazy"` va en el `<img>`, no en el `<source>`
> El atributo `loading` solo es válido en `<img>` e `<iframe>`. En los `<source>` no tiene efecto: el navegador decide la fuente y aplica el lazy loading sobre el `<img>` final — ver [[Lazy Loading]]. Tampoco existe `lazyalt`: el atributo del texto alternativo es `alt`.

## Múltiples tamaños (responsive art direction + resolution switching)

Cuando servimos la misma imagen en diferentes tamaños según la pantalla, usamos `srcset` con **descriptores de ancho** (`w`) junto al atributo `sizes`:

```html
<picture>
  <!-- Formato AVIF para los navegadores que lo soporten -->
  <source
    type="image/avif"
    sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 640px"
    srcset="img/imagen-640.avif 640w,
            img/imagen-1280.avif 1280w,
            img/imagen-1920.avif 1920w">
  <!-- Formato WebP como segunda opción -->
  <source
    type="image/webp"
    sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 640px"
    srcset="img/imagen-640.webp 640w,
            img/imagen-1280.webp 1280w,
            img/imagen-1920.webp 1920w">
  <!-- JPG como respaldo -->
  <img
    src="img/imagen-640.jpg"
    srcset="img/imagen-640.jpg 640w,
            img/imagen-1280.jpg 1280w,
            img/imagen-1920.jpg 1920w"
    sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 640px"
    loading="lazy"
    decoding="async"
    alt="Imagen del blog"
    width="640"
    height="384">
</picture>
```

Cómo lee esto el navegador:

1. Fija la **anchura renderizada** real según `sizes` (por ejemplo, en un móvil de 600px: `100vw` → 600px).
2. Elige del `srcset` el archivo con el descriptor `w` más adecuado (con una densidad de píxeles de 1 o 2, según la pantalla retina).
3. Repite el proceso por `source` hasta encontrar un `type` soportado; si ninguno coincide, usa el `<img>`.

> [!tip] `sizes` describe el *render*, no la pantalla
> `sizes="1920w, 1280w, 640w"` no es válido: los descriptores `w` van en el `srcset`, y `sizes` declara **cuánto espacio ocupa la imagen en el layout** usando media queries (`(max-width: 640px) 100vw, 640px`). Un `sizes` incorrecto hace que el navegador elija mal el archivo.

## Art direction con `media`

`<picture>` también permite servir **recortes distintos** según el dispositivo (por ejemplo, una panorámica en escritorio y una cuadrada en móvil):

```html
<picture>
  <source media="(max-width: 640px)" srcset="img/banner-cuadrado.webp">
  <img src="img/banner-panoramico.jpg" alt="Banner principal" width="1200" height="400">
</picture>
```

## Buenas prácticas

- **Generá todas las variantes** (AVIF → WebP → JPG) a partir del mismo original, con herramientas como Squoosh o `cwebp`/`avifenc`.
- Declará siempre `width` y `height` (o `aspect-ratio`) para evitar saltos de layout mientras carga — ver [[Lazy Loading]].
- Para imágenes **criticas (LCP)**, evitá `loading="lazy"`: cargalas eager para que aparezcan lo antes posible.
- Las imágenes de fondo de CSS no usan `<picture>`: ahí la negociación se resuelve con `image-set()` y fallbacks — ver [[Imágenes CSS]].

## Relacionadas

- [[Html]] — etiquetas y atributos generales
- [[Etiquetas]] — sección de imágenes
- [[Lazy Loading]] — carga diferida de imágenes con `<picture>`
- [[Imágenes CSS]] — imágenes desde hojas de estilo
