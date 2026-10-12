---
tags:
  - desarrollo
  - frontend
  - lazy-loading
tipo: concepto
estado: terminado
---

# Lazy Loading

**Lazy loading** (carga diferida) es la técnica de **postergar la descarga de recursos no críticos** hasta que el usuario realmente los necesita — normalmente, cuando el elemento se acerca a la zona visible de la página. En páginas con muchas imágenes o embeds, es una de las optimizaciones de rendimiento con mejor relación esfuerzo/impacto: reduce el peso inicial, el tiempo hasta la interactividad y el consumo de datos (decisivo en móviles y redes lentas).

> [!info] El navegador ya lo hace solo
> Desde 2020-2021, todos los navegadores modernos soportan lazy loading **nativo** con un simple atributo: no hace falta JavaScript ni librerías. Lo que sigue enseña a usarlo bien, cuándo *no* usarlo y qué hacer cuando la carga nativa no alcanza.

> [!warning] Lazy loading no es gratis
> Diferir una imagen la hace llegar **más tarde** (cuando el usuario scrollea) y puede **empeorar** el scroll en redes lentas si no se combina con reserva de espacio. La técnica correcta es: lazy para lo no visible, eager+prioritario para lo visible (ver `fetchpriority` más abajo), y dimensiones en **todas** las imágenes.

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

| Valor | Comportamiento | Equivalente práctico |
|---|---|---|
| `lazy` | Carga diferida: el navegador descarga el recurso cuando está a punto de entrar en el viewport | — |
| `eager` | Carga inmediata (comportamiento por defecto) | `loading` ausente |
| `auto` | El navegador decide con su heurística por defecto | En la práctica hoy = `eager` en Chromium |

### Cómo decide el navegador (heurísticas de distancia)

El navegador no espera a que el elemento esté a pocos píxeles del viewport: usa una **heurística de distancia** y carga con antelación para que la imagen aparezca sin esperar. Los números son internos y pueden cambiar, pero el comportamiento observable es:

| Navegador | Umbral observable |
|---|---|
| **Chromium** | Imágenes hasta ~**1250 px encima** del viewport y ~**3000 px debajo** entran en la cola; en conexiones lentas el umbral superior crece (~4000 px) para compensar la latencia |
| **Firefox** | Carga cuando el elemento **se aproxima al viewport** (umbral propio, no publicado como constante) |
| **Safari** | Desde 15.4: carga al acercarse al viewport; sin umbral documentado |

Detalles prácticos:

- Un elemento lazy **cerca del viewport inicial se carga igual, al arrancar**: el navegador no penaliza por "poco".
- Solo se aplica a recursos **fuera** de la zona visible; el lazy loading no afecta lo que ya se ve.
- El comportamiento exacto es una *hint*, no una garantía: el navegador manda. **Nunca** construyas lógica que asuma que una imagen lazy "no se ha descargado".
- `loading="lazy"` solo aplica a `<img>` e `<iframe>` con `src`. Imágenes de CSS (`background-image`) **no** lo soportan: ahí el diferido requiere JS o `content-visibility` — ver [[Imágenes CSS]].
- Las imágenes que entran al viewport **por resize de la ventana** también disparan la carga (el navegador reevalúa al cambiar el layout).

### La capa de datos: `Save-Data` y tipo de conexión

En Chromium, los usuarios pueden activar *Save-Data* (Ahorro de datos); el navegador envía el client hint `Save-Data: on` y tiende a ser más agresivo diferidiendo. El `Network Information API` permite lógica propia (no soportado en Firefox/Safari):

```js
const conn = navigator.connection;
if (conn?.saveData || ['slow-2g', '2g'].includes(conn?.effectiveType)) {
  document.querySelectorAll('img[data-hq]').forEach((img) => {
    img.src = img.dataset.lq;   // variante liviana en conexiones malas
  });
}
```

> [!tip] Combiná formato y diferido
> Lazy loading y formatos modernos son capas independientes que se potencian: diferir **y** servir AVIF/WebP multiplica el ahorro — ver [[Formatos Avif y Webp]].

## `width`/`height` y el salto de layout (CLS)

Definir `width` y `height` (o `aspect-ratio` en CSS) es lo que hace al lazy loading **seguro**: el navegador reserva el espacio del futuro antes de tener el archivo, y la página no salta cuando la imagen carga. Sin esas dimensiones, cada imagen diferida provoca **layout shift** al aparecer, lo que empeora la métrica CLS de Core Web Vitals.

```html
<!-- Reserva espacio con las dimensiones reales de la imagen -->
<img src="galeria/foto.jpg" alt="Foto" loading="lazy" width="300" height="300">
```

Cómo funciona la reserva: los navegadores **derivan `aspect-ratio` de los atributos `width`/`height`** (especificación HTML), así que aunque el CSS haga `img { width: 100%; height: auto }`, la proporción ya está y el espacio se calcula antes de la descarga.

```css
/* Alternativa moderna: declarar la proporción desde CSS */
img[data-lazy] {
  aspect-ratio: 3 / 2; /* 300 / 200 */
  object-fit: cover;
}
```

> [!warning] `loading="lazy"` sin dimensiones es contraproducente
> Ahorrás ancho de banda pero saboteás el CLS: la imagen "empuja" el contenido hacia abajo cuando aparece. Siempre `width`/`height` o `aspect-ratio`. Con `object-fit: cover` las dimensiones del atributo pueden no coincidir con las del archivo: usá la proporción del *layout*, no la del archivo.

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

- Solo aplica a iframes **con `src`**; un `<iframe>` de `about:blank` (inyectado por scripts) no aplica.
- En móvil el ahorro de un iframe lazy es enorme: un embed de YouTube carga ~1 MB de página+JS cuando se resuelve. La evolución de esta idea es el **patrón de fachada**: mostrar una miniatura + botón y montar el iframe recién al hacer clic (ej. librerías `lite-youtube-embed`), que ahorra incluso más que `loading="lazy"`.

## Cuándo NO usar lazy loading

La carga diferida perjudica los recursos que el usuario espera ver **ya**. Reglas:

- **No la uses en la imagen LCP** (el gran elemento visible al cargar, típicamente el banner principal): retrasarla retrasa el mayor indicador de percepción de velocidad del sitio. Para el hero, lo correcto es todo lo contrario: `fetchpriority="high"` (ver [[Preload]]).
- **No la uses en nada dentro del viewport inicial**: el navegador lo cargaría igual, pero ganás complejidad por nada.
- **No la uses en la primera imagen de un carrusel visible** ni en logos/iconsos de la nav.
- **Con cautela en galerías infinitas**: si el usuario hace scroll muy rápido, las imágenes llegan tarde; combiná con `fetchpriority="high"` en las primeras.
- **Cuidado con el Ctrl+F**: en algunos navegadores, contenido cuyo render está omitido (ver `content-visibility`) o imágenes nunca intersectadas no aparecen en la búsqueda del navegador hasta que se renderizan.

> [!tip] Regla práctica
> Lazy loading para **todo lo que está por debajo del fold** (galerías, embeds, thumbnails relacionados); carga normal o incluso prioritaria para el hero y el primer bloque visible.

## Lazy loading a medida con `IntersectionObserver`

La carga nativa no se puede personalizar (no controla cuándo ni qué disparar). Cuando necesites lógica propia — swaps de `src`, efectos al aparecer, imágenes dentro de componentes JS — se usa `IntersectionObserver`:

```html
<img class="lazy" data-src="galeria/foto.jpg" data-srcset="galeria/foto-320.jpg 320w, galeria/foto-640.jpg 640w"
     alt="Foto" width="300" height="300">
```

```js
// Implementación completa: swap de src/srcset, fallback sin IO, un solo disparo
const lazyImages = document.querySelectorAll('img.lazy');

if (!('IntersectionObserver' in window)) {
  // Sin soporte: carga todo de inmediato (degradación elegante)
  lazyImages.forEach((img) => {
    if (img.dataset.src) img.src = img.dataset.src;
    if (img.dataset.srcset) img.srcset = img.dataset.srcset;
  });
} else {
  const observer = new IntersectionObserver((entries, obs) => {
    for (const entry of entries) {
      if (!entry.isIntersecting) continue;
      const img = entry.target;
      if (img.dataset.srcset) img.srcset = img.dataset.srcset;  // srcset primero…
      if (img.dataset.src) img.src = img.dataset.src;           // …luego src: una sola descarga
      obs.unobserve(img);                                       // un solo disparo por imagen
    }
  }, {
    rootMargin: '200px 0px',   // empieza a cargar 200px antes de entrar
    threshold: 0               // basta con un píxel visible
  });

  lazyImages.forEach((img) => observer.observe(img));
}
```

Por qué este patrón y no `scroll` con mediciones:

- `IntersectionObserver` corre **fuera del hilo principal de layout**: no dispara reflow por cada scroll ni consume el budget de INP (la métrica de interactividad de Core Web Vitals).
- Antes de `IntersectionObserver` se usaba el evento `scroll` con mediciones manuales: obsolete, caro y con jank.
- `rootMargin` "adelanta" el disparo, igual que la heurística de la carga nativa.
- `unobserve` libera el observer de cada imagen ya cargada.
- Para interacciones al aparecer (fade-in, conteadores) el mismo observer sirve cambiando `entry.target` por el efecto en vez del `src`.

> [!warning] El swap de `src` manual tiene un costo propio
> Cambiar `img.src` en runtime invalida y re-descarga; si la imagen ya estaba en viewport y el JS tarda, el usuario ve el hueco más tiempo que con carga nativa. Regla: **si la carga nativa te alcanza, no reimplementes** — usala; el observer es para lógica que la nativa no puede expresar.

## `content-visibility: auto` (CSS)

Para bloques completos (tarjetas, listas largas) existe lazy loading **a nivel de render en CSS**: el navegador omite layout y pintado de secciones fuera del viewport y las procesa cuando se acercan:

```css
.tarjeta {
  content-visibility: auto;
  contain-intrinsic-size: auto 320px; /* tamaño estimado mientras no se renderiza */
}
```

Puntos clave del modelo:

| Aspecto | `content-visibility: auto` | `content-visibility: hidden` |
|---|---|---|
| Render fuera de pantalla | Omitido | Omitido |
| Se puede llegar scrolleando | **Sí** (se renderiza al acercarse) | No (hasta que se cambie a `visible`) |
| Búsqueda en página (Ctrl+F) | Funciona (el navegador la considera) | No encuentra el contenido |
| `contain-intrinsic-size` | Reserva tamaño estimado; `auto <length>` recuerda el último tamaño real | Igual |

- **No es lazy loading de red**: `content-visibility: auto` **no** difiere la descarga de imágenes o scripts de adentro del bloque — si necesitás diferir bytes, seguí usando `loading="lazy"` (o JS). Difiera *render*, no *red*.
- `contain-intrinsic-size` reserva un tamaño aproximado para evitar el mismo salto de layout del `width`/`height` — ver [[Imágenes CSS]] para cómo encaja con el resto de técnicas de imagen.
- Soporte: Chromium 85+, Firefox 125+, Safari 18+. Donde no hay soporte, el contenido simplemente se renderiza normal: degradación segura.

## Interacción con preload y preconnect

Son palancas de **dirección opuesta** y se pisan si se mezclan sin criterio:

| Técnica | Cuándo carga | Prioridad | Uso típico |
|---|---|---|---|
| **Preload** (`rel="preload"`) | **Ya**, al arrancar la página | Alta | Recurso crítico descubierto tarde (fuente, hero) — ver [[Preload]] |
| **Preconnect** (`rel="preconnect"`) | Abre conexión (DNS+TCP+TLS) | — | Origen del que sabés que vendrán recursos lazy (embeds) |
| **Lazy loading** (`loading="lazy"`) | **Después**, al acercarse al viewport | Baja/media | Imágenes y embeds por debajo del fold |

Errores de combinación típicos:

- **Preload + lazy en el mismo recurso**: contradicción directa — el preload lo baja ya con prioridad alta y el atributo no tiene efecto. Elegí una.
- **Preconnect al CDN de las imágenes lazy**: sí tiene sentido — la conexión queda lista cuando el usuario llegue al fold y la imagen lazy dispare su petición.
- **Preload de una imagen below-the-fold "por si acaso"**: roba ancho de banda al LCP. No.

## Matices móviles

- En móvil el ancho de banda suele ser el cuello de botella: el ahorro de diferir 20 fotos below-the-fold es la diferencia entre LCP de 2s y de 5s.
- Chromium en redes 2G/lentas agranda el umbral de precarga (carga *más tarde*, con más margen), lo que puede producir "huecos" visibles al scrollear rápido: `rootMargin` es tu herramienta si implementás tu propio observer.
- Lazy loading nativo **no respeta `Save-Data` de forma especial** (es siempre lazy igual); la capa Save-Data aplica a *qué* servir, no a *cuándo*.
- Iframes lazy de mapas/video son la mayor victoria móvil: un embed sin resolver ocupa ~0 KB; con `loading="lazy"` el navegador solo lo monta si el usuario llega hasta él.

## Buenas prácticas

- Medí antes y después (Lighthouse, DevTools → Network): el lazy loading en una página con 3 imágenes no cambia nada.
- Serví las variantes modernas en formato ligero (AVIF/WebP) además de diferirlas — ver [[Formatos Avif y Webp]]: son capas independientes que se potencian.
- No dupliques estrategias: si el navegador ya hace lazy loading nativo, no cargues una librería (`lazysizes`) salvo necesidad real de personalización. El atributo `loading` se puede usar *dentro* de librerías (lazysizes lo detecta), pero la necesidad real es rara.
- El `alt` sigue siendo obligatorio aunque la imagen tarde en cargar.
- En SPA: las imágenes insertadas por JS después del render **no** disparan la heurística del navegador de forma fiable en todos los motores; si el insertado es masivo, evaluá `loading="lazy"` en el markup generado por el framework (React: `<img loading="lazy">` funciona directo).

## Errores comunes

1. **Lazy en la imagen LCP**: empeora la métrica que más pesa. Hero = eager + `fetchpriority="high"`.
2. **Sin `width`/`height`**: CLS disparado por cada imagen que aparece. Siempre dimensiones.
3. **Reimplementar con observer lo que la nativa hace**: código extra, posible doble lógica, y la nativa ya cubre la heurística de distancia en los 3 motores.
4. **Poner `loading` en `<source>` de `<picture>`**: no es válido; va en el `<img>` — ver [[Formatos Avif y Webp]].
5. **`rootMargin` negativo** en el observer propio: dispara tarde (recién dentro del viewport) y el usuario ve el hueco.
6. **Asumir que "lazy = no descargado"**: cerca del fold, el navegador la cargó igual al arrancar. La semántica es *hint*, no promesa.
7. **Olvidar el fallback sin `IntersectionObserver`**: en navegadores viejos no se carga nada. El `if` de fallback del ejemplo de arriba es obligatorio.

## Compatibilidad

- `loading="lazy"`: Chrome 77+ (imágenes e iframes), Firefox 75+, **Safari 15.4+** (2022). Los navegadores anteriores ignoran el atributo y cargan todo: degradación segura.
- `IntersectionObserver`: universal desde ~2019.
- `content-visibility`: ver sección propia (Safari llegó en 18).

## Cheatsheet

| Mecanismo | Dónde | Difiere | Requiere JS | Nota clave |
|---|---|---|---|---|
| `loading="lazy"` | `<img>`, `<iframe>` | Descarga | No | Heurística de distancia del navegador |
| `loading="eager"`/`auto` | igual | — (carga ya) | No | `auto` ≈ eager hoy |
| `fetchpriority="high"` | `<img>` etc. | — (sube prioridad) | No | Para el hero LCP |
| `IntersectionObserver` + `data-src` | cualquier recurso | Descarga, a medida | **Sí** | `rootMargin` adelanta el disparo |
| `content-visibility: auto` | bloques de layout | **Render**, no red | No | `contain-intrinsic-size` reserva espacio |
| Fachada de embed | iframes de video/mapas | Montaje del iframe | Sí (o click nativo) | Mayor ahorro móvil |

## Relacionadas

- [[Html]] — el atributo `loading` y los elementos que lo soportan
- [[Formatos Avif y Webp]] — complementar carga diferida con formatos compactos
- [[Preload]] — lo contrario: forzar carga temprana de un recurso crítico
- [[Prefetch]] — anticipar la *siguiente* navegación
- [[Imágenes CSS]] — lazy loading desde estilos con `content-visibility`
- [[Document Object Model (DOM)]] — cómo se insertan las imágenes diferidas desde JS
