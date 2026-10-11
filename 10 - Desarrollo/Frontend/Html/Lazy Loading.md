---
aliases: ["Carga diferida", "loading lazy"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Formatos Avif y Webp]]"
  - "[[Preload]]"
  - "[[Imágenes CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/Performance/Lazy_loading"
---

# Lazy Loading

**Lazy loading** (carga diferida) es la técnica de **postergar la descarga de recursos no críticos** hasta que el usuario realmente los necesita — normalmente, cuando el elemento se acerca a la zona visible de la página. En páginas con muchas imágenes o embeds, es una de las optimizaciones de rendimiento con mejor relación esfuerzo/impacto: reduce el peso inicial, el tiempo hasta la interactividad y el consumo de datos (decisivo en móviles y redes lentas).

> [!info] El navegador ya lo hace solo
> Desde 2020-2021, todos los navegadores modernos soportan lazy loading **nativo** con un simple atributo: no hace falta JavaScript ni librerías.

## Carga nativa: `loading="lazy"`

En imágenes y iframes, un solo atributo activa la carga diferida:

```html
<img
  src="galeria/foto.jpg"
  alt="Foto de la galería"
  loading="lazy"
  width="300"
  height="300">
```

El atributo `loading` acepta tres valores:

| Valor | Comportamiento |
|---|---|
| `lazy` | Carga diferida: el navegador descarga el recurso cuando está a punto de entrar en el viewport |
| `eager` | Carga inmediata (este es el comportamiento por defecto) |
| `auto` | El navegador decide (su heurística por defecto) |

### Cómo decide el navegador

El navegador no espera a que el elemento esté a pocos píxeles del viewport: usa una **heurística de distancia** (del orden de 1250 px en Chromium) y carga con antelación para que la imagen aparezca sin esperar. Detalles prácticos:

- Un elemento lazy **cerca del viewport inicial se carga igual, al arrancar**: el navegador no penaliza por "poco".
- Solo se aplica a recursos **fuera** de la zona visible; el lazy loading no afecta lo que ya se ve.
- El comportamiento exacto es una *hint*, no una garantía: el navegador manda.

## `width`/`height` y el salto de layout (CLS)

Definir `width` y `height` (o `aspect-ratio` en CSS) es lo que hace al lazy loading **seguro**: el navegador reserva el espacio del futuro antes de tener el archivo, y la página no salta cuando la imagen carga. Sin esas dimensiones, cada imagen diferida provoca **layout shift** al aparecer, lo que empeora la métrica CLS de Core Web Vitals.

```html
<!-- Reserva espacio con las dimensiones reales de la imagen -->
<img src="galeria/foto.jpg" alt="Foto" loading="lazy" width="300" height="300">
```

```css
/* Alternativa moderna: declarar la proporción desde CSS */
img[data-lazy] {
  aspect-ratio: 3 / 2; /* 300 / 200 */
  object-fit: cover;
}
```

> [!warning] `loading="lazy"` sin dimensiones es contraproducente
> Ahorrás ancho de banda pero saboteás el CLS: la imagen "empuja" el contenido hacia abajo cuando aparece. Siempre `width`/`height` o `aspect-ratio`.

## Lazy loading de iframes

El mismo atributo funciona en `<iframe>` — ideal para embeds de video o mapas fuera de la primera pantalla:

```html
<iframe
  src="https://www.youtube-nocookie.com/embed/VIDEO"
  title="Video explicativo"
  loading="lazy"
  width="560"
  height="315"></iframe>
```

## Cuándo NO usar lazy loading

La carga diferida perjudica los recursos que el usuario espera ver **ya**. Reglas:

- **No la uses en la imagen LCP** (el gran elemento visible al cargar, típicamente el banner principal): retrasarla retrasa el mayor indicador de percepción de velocidad del sitio.
- **No la uses en nada dentro del viewport inicial**: el navegador lo cargaría igual, pero ganás complejidad por nada.
- **Con cautela en galerías infinitas**: si el usuario hace scroll muy rápido, las imágenes llegan tarde; combiná con `fetchpriority="high"` en las primeras.

> [!tip] Regla práctica
> Lazy loading para **todo lo que está por debajo del fold** (galerías, embeds, thumbnails relacionados); carga normal o incluso prioritaria para el hero y el primer bloque visible.

## Lazy loading a medida con `IntersectionObserver`

La carga nativa no se puede personalizar (no controla cuándo ni qué disparar). Cuando necesites lógica propia — swaps de `src`, efectos al aparecer, imágenes dentro de componentes JS — se usa `IntersectionObserver`:

```html
<img class="lazy" data-src="galeria/foto.jpg" alt="Foto" width="300" height="300">
```

```js
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;      // recién ahora se dispara la descarga
      observer.unobserve(img);        // un solo disparo por imagen
    }
  }
}, { rootMargin: '200px' });          // empieza a cargar 200px antes de entrar

document.querySelectorAll('img.lazy').forEach((img) => observer.observe(img));
```

- `rootMargin` "adelanta" el disparo, igual que la heurística de la carga nativa.
- `unobserve` libera el observer de cada imagen ya cargada.
- Antes de `IntersectionObserver` se usaba el evento `scroll` con mediciones manuales: obsolete, caro y con jank.

## `content-visibility: auto` (CSS)

Para bloques completos (tarjetas, listas largas) existe lazy loading **a nivel de render en CSS**: el navegador omite layout y pintado de secciones fuera del viewport y las procesa cuando se acercan:

```css
.tarjeta {
  content-visibility: auto;
  contain-intrinsic-size: auto 320px; /* tamaño estimado mientras no se renderiza */
}
```

`contain-intrinsic-size` reserva un tamaño aproximado para evitar el mismo salto de layout del `width`/`height` — ver [[Imágenes CSS]] para cómo encaja con el resto de técnicas de imagen.

## Lazy loading vs preload

Son estrategias opuestas y complementarias:

| Técnica | Cuándo carga | Prioridad | Uso típico |
|---|---|---|---|
| **Preload** (`rel="preload"`) | **Ya**, al arrancar la página | Alta | Recurso crítico descubierto tarde (fuente, hero) |
| **Lazy loading** (`loading="lazy"`) | **Después**, al acercarse al viewport | Baja/media | Imágenes y embeds por debajo del fold |

Detalle de la precarga en [[Preload]] y su prima futurista [[Prefetch]].

## Buenas prácticas

- Medí antes y después (Lighthouse, DevTools → Network): el lazy loading en una página con 3 imágenes no cambia nada.
- Serví las variantes modernas en formato ligero (AVIF/WebP) además de diferirlas — ver [[Formatos Avif y Webp]]: son capas independientes que se potencian.
- No dupliques estrategias: si el navegador ya hace lazy loading nativo, no cargues una librería (`lazysizes`) salvo necesidad real de personalización.
- El `alt` sigue siendo obligatorio aunque la imagen tarde en cargar.

## Relacionadas

- [[Html]] — el atributo `loading` y los elementos que lo soportan
- [[Formatos Avif y Webp]] — complementar carga diferida con formatos compactos
- [[Preload]] — lo contrario: forzar carga temprana de un recurso crítico
- [[Prefetch]] — anticipar la *siguiente* navegación
- [[Imágenes CSS]] — lazy loading desde estilos con `content-visibility`
