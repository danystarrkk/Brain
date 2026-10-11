---
aliases: ["Imágenes de fondo", "Background images"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Background]]"
  - "[[Formatos Avif y Webp]]"
  - "[[Lazy Loading]]"
  - "[[Degradados en CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/background-image"
---

# Imágenes CSS

Desde CSS podemos insertar imágenes, y el uso más común es como **fondo** de un elemento con `background-image`.

## `background-image` y la ruta

```css
body {
  background-image: url("ruta/de/la/imagen.jpg");
}
```

La ruta se resuelve **desde la ubicación de la hoja de CSS**, no desde el HTML: si el CSS vive en `css/estilos.css` y la imagen en `img/fondo.jpg`, la ruta correcta es `url("../img/fondo.jpg")` — hay que retroceder un nivel con `../`. Entre comillas simples o dobles; sin comillas solo si la ruta no tiene espacios.

> [!warning] `bacground-image` no existe
> El nombre de la propiedad es `background-image` (con "k" no hay nada en CSS). Un typo acá no tira error de sintaxis de la hoja entera, pero la imagen simplemente no aparece.

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
| `contain`                  | Muestra la imagen **completa** dentro del área (puede sobrar fondo) |
| `50%` / `300px`            | Tamaño explícito, admite cualquier unidad                         |

```css
.hero {
  background-image: url("hero.avif");
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center; /* casi siempre junto a cover */
}
```

> [!tip] `cover` + `center` es el dúo del hero
> `background-size: cover` casi siempre va con `background-position: center` para que el recorte sea parejo a ambos lados.

## La sombra del `background` shorthand

Todas las propiedades de fondo entran en un solo atajo:

```css
.hero {
  background: #111 url("hero.avif") center / cover no-repeat;
  /*           color  imagen           pos / tamaño  repetición */
}
```

> [!warning] El atajo resetea lo que no menciones
> Escribir `background:` sin los valores que necesites **borra los valores por defecto** posteriores (por ejemplo, `background-color` queda `transparent`). En proyectos serios se prefiere usar las propiedades largas.

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

## ¿Fondo (`background-image`) o `<img>`?

| Criterio             | `background-image`                       | `<img>`                                  |
| -------------------- | ---------------------------------------- | ---------------------------------------- |
| Semántica / SEO      | Decorativa                              | Contenido real                           |
| Accesibilidad        | Requiere aria si aporta información      | Tiene `alt` obligatorio                  |
| Responsive automático | `cover`/`contain` lo resuelve           | Necesitás `srcset` y `sizes`             |
| Lazy loading         | Por CSS (limitado)                      | Nativo con `loading="lazy"`              |

Regla: **si la imagen comunica algo, es `<img>`; si es decoración del diseño, es fondo.** Ver [[Lazy Loading]] para diferir la carga de imágenes.

> [!info] Fondos degradados conviven gratis
> `background-image` acepta degradados además de imágenes: `background: linear-gradient(...)` — ver [[Degradados en CSS]].

## Relacionadas

- [[Background]] — `background-position` en detalle y el resto de propiedades
- [[Formatos Avif y Webp]] — qué formato servir y cómo detectarlo
- [[Lazy Loading]] — no cargar imágenes hasta que se necesiten
- [[Degradados en CSS]] — pintar fondos sin archivos
