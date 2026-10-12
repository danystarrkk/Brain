---
tags:
  - desarrollo
  - frontend
  - formatos-webp
tipo: concepto
estado: terminado
---

# Formatos AVIF y WebP

En nuestras páginas podemos servir imágenes en formatos modernos (**AVIF** y **WebP**), que comprimen mucho mejor que el clásico JPG/PNG sin pérdida visible de calidad. Como no todos los navegadores los soportan, la etiqueta **`<picture>`** nos permite declarar varias versiones y dejar que el navegador elija la que soporte, con JPG como respaldo universal.

> [!info] Por qué esto importa tanto
> Las imágenes suelen ser el **mayor peso de una página** (frecuentemente 50% o más de los bytes descargados) y dominan el LCP (*Largest Contentful Paint*), la métrica de percepción de velocidad más importante de las [Core Web Vitals](https://web.dev/vitals/). Pasar de JPG a AVIF puede recortar 40–60% del peso de una fotografía: eso se traduce directo en LCP menor, menos datos móviles y mejor posicionamiento en búsqueda.

## Modelos de compresión: qué hace a cada formato lo que es

Para elegir bien hay que entender *por qué* cada formato pesa lo que pesa:

| Formato | Modelo | Tipo | Notas del formato |
|---|---|---|---|
| **PNG** | Deflate (LZ77 + Huffman) sobre píxeles sin comprimir | Solo **sin pérdida** | Perfecto para texto/line-art; terrible para fotos (no aprovecha redundancia espacial perceptual) |
| **JPEG** | DCT (transformada de coseno discreta) + cuantización + color YCbCr 4:2:0 | Solo **con pérdida** | Sin alfa, sin animación; calidad >90 desperdicia bytes (ruido de alta frecuencia) |
| **WebP** | Predicción intra-frame (basada en VP8) +, en animación, predicción entre frames | **Con y sin pérdida** | 8 bits por canal; lossless rinde mal en fotos complejas |
| **AVIF** | Codificador **AV1** adaptado a imágenes: predicción intra, múltiples planes, superresolución sintética de grano | **Con y sin pérdida** | 8/10/12 bits, HDR y gamut amplio; la contrapartida es el **coste de codificación** (10–100× más lento que WebP) |

Detalles que explican el comportamiento práctico:

- **AVIF con pérdida** aprovecha herramientas de AV1 pensadas para vídeo (referencia intra múltiple, restauración de bordes, *grain synthesis*): por eso logra tamaños que WebP no alcanza en fotos.
- **AVIF sin pérdida** suele ser *peor* que PNG en imágenes de poca entropía (iconos, texto): su pérdidaless está optimizado para fotos.
- **WebP sin pérdida** es buena en capturas de UI e iconos; su versión con pérdida ya supera cómodamente a JPEG.
- La **codificación AVIF es cara**: un `avifenc` con `--speed 6` puede tardar segundos por imagen. En build time no importa; en un CMS en vivo sí.

## Tabla comparativa completa

| Característica | JPEG | PNG | GIF | WebP | **AVIF** | JPEG XL |
|---|---|---|---|---|---|---|
| Compresión con pérdida | ✔ | ✖ | ✖ (indexada) | ✔ | ✔ (mejor) | ✔ |
| Compresión sin pérdida | ✖ | ✔ | ✖ | ✔ | ✔ | ✔ |
| Transparencia (alfa) | ✖ | ✔ | ✔ (1 bit) | ✔ | ✔ | ✔ |
| Animación | ✖ | ✖ | ✔ | ✔ | ✔ | ✔ |
| HDR / 10 bits | ✖ | Parcial (16b) | ✖ | ✖ (8b) | ✔ | ✔ |
| Tamaño relativo (foto, calidad equivalente) | ~100% (base) | 200–400% | — | ~65–75% | **~40–60%** | ~50–60% |
| Streaming progresivo | ✔ (entrelazado) | ✔ (Adam7) | ✖ | ✔ (demux de frames) | ✔ | ✔ |
| Soporte (2026) | Universal | Universal | Universal | Universal (desde sept. 2022) | Todos salvo Edge viejo/IE | Parcial (Safari 17+; resto a bandera) |

> [!tip] Regla de decisión
> **Fotos y banners → AVIF** (con WebP de respaldo). **Iconos, capturas de UI, line-art → WebP sin pérdida o PNG**. **Animaciones cortas → WebP animado** (o AVIF animado cuando el soporte lo permita). JPEG solo como *fallback* universal de `<picture>` — nadie debería subir solo JPEG en 2026.

## Soporte real por navegador (lo que hay que saber)

| Formato | Chrome | Edge | Firefox | Safari |
|---|---|---|---|---|
| WebP | 32+ | 18+ | 65+ | 14+ (2020) |
| AVIF | 85+ | **121+** (2024; el viejo Edge/IE no) | 93+ | **16+** (2022, macOS Ventura / iOS 16) |

Es decir: desde ~2023 el 95%+ de usuarios reales soporta AVIF y el 99%+ soporta WebP. El `<picture>` con fallback JPG sigue siendo obligatorio **por cobertura larga** (kioscos, enterprise viejo, bots), no por el usuario medio.

> [!warning] Soporte ≠ corrección
> Un navegador puede *declarar* soporte por MIME y aun así fallar si el servidor envía el archivo con un `Content-Type` equivocado (por ejemplo `application/octet-stream` o el de un error 404 HTML). El decoder rechaza el recurso y queda un hueco. Ver "Errores comunes".

## Negociación con `<picture>`: mecánica exacta

```html
<picture>
  <source srcset="img/blog1.avif" type="image/avif">
  <source srcset="img/blog1.webp" type="image/webp">
  <img loading="lazy" src="img/blog1.jpg" alt="Imagen del blog" width="800" height="600">
</picture>
```

Cómo decide el navegador, paso a paso:

1. Recorre los `<source>` **de arriba hacia abajo** y toma el **primero** cuya `type` sea un MIME que declare soportado (`image/avif`, `image/webp`, `image/jpeg`…). El orden importa: **AVIF antes que WebP**, porque el primer match gana.
2. El chequeo de `type` es **puro sniffing de MIME**: el navegador compara el string contra su lista de tipos soportados **sin descargar nada**. No "prueba" el archivo.
3. Si ningún `<source>` coincide (navegador sin soporte, o tipos vacíos), usa el `<img>`: es el **respaldo obligatorio** — sin `<img>` la imagen no se muestra en navegadores viejos.
4. Una vez elegido el `<source>`, su `srcset`/`sizes` gobiernan la selección de variante (ver más abajo).

> [!warning] `loading="lazy"` va en el `<img>`, no en el `<source>`
> El atributo `loading` solo es válido en `<img>` e `<iframe>`. En los `<source>` no tiene efecto: el navegador decide la fuente y aplica el lazy loading sobre el `<img>` final — ver [[Lazy Loading]]. Tampoco existe `lazyalt`: el atributo del texto alternativo es `alt`.

### El lado del servidor: `Content-Type` y caching

La respuesta del servidor debe traer el MIME correcto:

```http
HTTP/1.1 200 OK
Content-Type: image/avif
Cache-Control: public, max-age=31536000, immutable
```

- `Content-Type` equivocado → decode fallido (síntoma: icono roto o espacio vacío sin error 4xx).
- Cache con hash en el nombre (`hero.a1b2c3.avif`) + `immutable` = la capa de caching perfecta; los formatos modernos no cambian esto, solo lo potencian.
- En CDNs con *image optimization on the fly* (Cloudflare Polish/Imagen, Fastly, Cloudinary) la negociación se hace en el borde: el CDN mira la cabecera `Accept` del navegador y sirve AVIF/WebP/JPG automáticamente, sin `<picture>` en el HTML.

## `srcset` / `sizes`: anatomía de una selección responsive

Para servir la misma imagen en diferentes tamaños (***resolution switching***) usamos `srcset` con **descriptores de ancho** (`w`) junto al atributo `sizes`:

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
2. Multiplica por la **densidad de píxeles** del dispositivo (1× en escritorio, 2× o 3× en retina): 600px × 2 = 1200px *necesarios*.
3. Elige del `srcset` el descriptor `w` **más pequeño que cubra esa necesidad** (si todos son menores, el más grande): con candidatos 640/1280/1920 → elige `1280w`.
4. Repite el proceso por `source` hasta encontrar un `type` soportado; si ninguno coincide, usa el `<img>`.

Variantes de notación en `srcset`:

| Notación | Significado | Uso |
|---|---|---|
| `img-640.jpg 640w` | descriptor de **ancho físico** | Responsive con `sizes` — la opción recomendada |
| `img-1x.jpg 1x, img-2x.jpg 2x` | descriptor de **densidad** | Cuando el layout es siempre el mismo ancho |
| `img.jpg` (sin descriptor) | única opción | Equivalente a `src` simple |

> [!tip] `sizes` describe el *render*, no la pantalla
> `sizes="1920w, 1280w, 640w"` no es válido: los descriptores `w` van en el `srcset`, y `sizes` declara **cuánto espacio ocupa la imagen en el layout** usando media queries (`(max-width: 640px) 100vw, 640px`). Si `sizes` falta con descriptores `w`, el navegador asume **`100vw`** — funciona en heroes, pero en thumbnails esto elige archivos gigantes sin que lo notes. Un `sizes` incorrecto hace que el navegador elija mal el archivo.

## Art direction vs resolution switching

Son **dos problemas distintos** que `<picture>` resuelve con atributos distintos:

| Problema | Pregunta | Mecanismo | Ejemplo |
|---|---|---|---|
| **Resolution switching** | "Misma imagen, distinto tamaño de archivo" | `srcset` + `sizes` en `<img>` o `<source>` | Thumbnail 320px en móvil, full 1920px en desktop |
| **Art direction** | "Imagen distinta según el dispositivo" | `media` en `<source>` | Panorámica 3:1 en escritorio, recorte cuadrado en móvil |

Art direction con `media` — los `<source>` con `media` también se evalúan de arriba hacia abajo, del más específico al más general:

```html
<picture>
  <source media="(max-width: 640px)" srcset="img/banner-cuadrado.webp">
  <img src="img/banner-panoramico.jpg" alt="Banner principal" width="1200" height="400">
</picture>
```

Ambos se pueden combinar (formato × media × tamaño) — el ejemplo completo sería un `<source media>` con `type` y `srcset` a la vez. La regla de orden se mantiene: **lo más específico primero**.

## Flujo de trabajo de codificación

Generá **todas las variantes a partir del mismo original de máxima calidad** (nunca "JPG → AVIF": cada transcodificación pierde calidad; el original debe ser PNG sin pérdida o TIFF/JPG de calidad máxima).

| Herramienta | Comando típico | Cuándo usarla |
|---|---|---|
| **Squoosh** (app web, squoosh.app) | GUI con comparación lado a lado | Ajuste visual de calidad; one-off |
| **cwebp** (libwebp) | `cwebp -q 80 -m 6 hero.jpg -o hero.webp` | Builds/scripts; `-q 75–85` es el rango útil |
| **avifenc** (libavif) | `avifenc -s 6 -q 60 hero.jpg hero.avif` | Máximo control; `-q 50–60` equivale visualmente a JPG `q75`; `-s` (speed) sube = más rápido y peor |
| **sharp** (Node) | `sharp('hero.jpg').avif({ quality: 55, effort: 4 })` | Pipelines Node/bundlers; `effort` 0–9 |
| **ImageMagick** | `convert hero.jpg -quality 60 hero.avif` | Cuando ya está instalado |
| **Vips / ffmpeg** | `vips avifsave hero.jpg hero.avif` | Volúmenes grandes |

Guías de calidad prácticas:

- **WebP lossy**: `-q 80` rinde visualmente equivalente a JPEG `-quality 75` con ~25% menos bytes. Por debajo de 70 los artefactos se ven en texto/ bordes.
- **AVIF**: `-q 50` es el punto dulce foto; `-q 30` tolerable en thumbnails; `-q 70+` solo si el cliente exige inspección pixel a pixel.
- **Iconos/line-art**: WebP `-lossless` o directamente SVG si el arte lo permite (ver [[Imágenes CSS]] para uso desde CSS).
- Dimensiones: servir **exactamente** el tamaño renderizado; un "1920px para un thumbnail de 300px" anula cualquier ahorro del formato.

### Autómata de build (receta)

```js
// pipeline con sharp: un original → todas las variantes
import sharp from 'sharp';

const widths = [640, 1280, 1920];
for (const w of widths) {
  const img = sharp('src/hero.jpg').resize({ width: w });
  await img.webp({ quality: 80 }).toFile(`dist/hero-${w}.webp`);
  await img.avif({ quality: 55, effort: 4 }).toFile(`dist/hero-${w}.avif`);
  await img.jpeg({ quality: 75, mozjpeg: true }).toFile(`dist/hero-${w}.jpg`);
}
```

## LCP: dónde los formatos tocan las Core Web Vitals

- **LCP**: si el elemento más grande del primer viewport es una imagen (casi siempre lo es en landings), su tamaño decide el LCP. AVIF/WebP reducen sus bytes → descarga más rápida → LCP menor. En imágenes LCP combiná formato moderno con carga *eager* + `fetchpriority="high"`, **nunca** lazy — ver [[Lazy Loading]].
- **CLS**: reservá el espacio con `width`/`height` (los navegadores derivan `aspect-ratio` de esos atributos) — también en imágenes dentro de `<picture>`.
- **Descubrimiento tardío**: si la imagen LCP está dentro de `<picture>`, el preload tiene que coincidir con la variante que el navegador va a elegir; para eso existe `imagesrcset` en [[Preload]].
- Ancho de banda: menos bytes = menos competencia entre peticiones, lo que también beneficia a CSS/JS críticos.

> [!warning] Negociación + lazy loading mal combinados
> Un `<source type="image/avif">` solo sirve si el navegador elige ese `<source>` **antes** de que la imagen se ponga lazy/necesaria. En la práctica funciona (la selección es instantánea al llegar al viewport), pero si además usás preload, el `<link rel="preload">` debe apuntar al **mismo formato** que eligirá `<picture>` — o descargarás dos veces.

## Despliegue en CDN

- **Negociación en el origen** (con `<picture>`): el HTML lista variantes; el CDN solo cachea. Máximo control, HTML más pesado.
- **Negociación en el borde**: el CDN (Cloudinary, imgix, Cloudflare Images + Polish, Fastly Image Optimizer) recibe un original y sirve el formato óptimo mirando `Accept: image/avif,image/webp,...`. El HTML queda limpio; dependés de un tercero.
- `Vary: Accept` es necesario si el origen sirve formatos distintos según cabecera — sin él, un proxy puede servir AVIF a un navegador que no lo soporta (y viceversa, sirviendo JPG a quien esperaba AVIF).
- Compresión de texto (Brotli/Gzip) no aplica a imágenes: los formatos modernos *son* la compresión de imágenes.

## Accesibilidad

- **`alt` es obligatorio** en el `<img>` final de `<picture>`, aunque la imagen tarde en cargar. El `<source>` **nunca** lleva `alt` (ni ningún atributo de contenido): la imagen es un solo contenido con una sola descripción, sin importar qué variante se sirva.
- Si la imagen es puramente decorativa: `alt=""` (vacío, para que el lector de pantalla la salte).
- No poner texto en la imagen como única vía de información: los formatos no cambian eso, pero un hero AVIF con texto incrustado sigue siendo inaccesible.

## Rendimiento

- Cachearo con `Cache-Control` largo + nombres hasheados: los formatos modernos valen más si el navegador no vuelve a pedirlos.
- No generes variantes que nadie usa: cada `srcset` duplicado ×3 formatos = 9 archivos por imagen; limitá los anchos a los breakpoints reales del diseño.
- El peso del HTML crece con cada `<source>`: es el costo de la negociación; en páginas con muchas imágenes valora CDN en el borde.

## Errores comunes

1. **`loading="lazy"` o `alt` en el `<source>`**: no hacen nada / no son válidos. Van en el `<img>`.
2. **`type` equivocado o servidor con MIME genérico**: el `<source type="image/avif">` pasa el sniffing, pero si el servidor responde `Content-Type: application/octet-stream`, el decoder falla. Verificá la cabecera real (DevTools → Network → cabeceras).
3. **WebP antes que AVIF en el `<source>`**: el primer match gana; un navegador con ambos soportes se quedaría con WebP. AVIF primero.
4. **`srcset` con `w` y sin `sizes`** en imágenes que no son full-bleed: el default `100vw` hace que el navegador descargue el archivo gigante en un thumbnail.
5. **Transcodificar de JPG a AVIF**: acumula artefactos dobles. Siempre desde el original sin pérdida.
6. **Olvidar el `<img>` de respaldo**: navegadores sin soporte (y algunos crawlers) muestran un hueco.
7. **`sizes` con descriptores `w`** (`sizes="640w"`): sintaxis inválida; `sizes` es lista de media queries + longitudes.
8. **Preload de un formato que `<picture>` no elige**: descarga duplicada — usar `imagesrcset`/`imagesizes` y `type` en el preload, ver [[Preload]].
9. **Subir fotos de 4000px para un slider de 800px**: ningún formato salva eso; redimensioná antes de codificar.

## Compatibilidad

- WebP: soporte universal desde septiembre 2022 (última pieza: Firefox 65, ya de 2019). AVIF: Chrome 85+ (2020), Firefox 93+ (2021), Safari 16+ (2022), Edge recién en 121 (2024) — por eso el fallback WebP/JPG sigue teniendo sentido.
- JPEG XL: soportado en Safari 17+, sin soporte en Chromium/Firefox por ahora: **no adoptar** como estrategia principal.
- HEIC (formato de Apple): **nunca** sirvas HEIC en web; los navegadores no lo decodifican. Exportá a AVIF/WebP.
- `<picture>` y `srcset`/`sizes` tienen soporte total desde 2017–2020 en todos los navegadores: no requieren polyfill.

## Cheatsheet

| Elemento/atributo | Función | Valor clave |
|---|---|---|
| `<picture>` | Contenedor de negociación | — |
| `<source type>` | Filtro por MIME soportado (sin descargar) | `image/avif`, `image/webp` |
| `<source media>` | Art direction | media query |
| `<source srcset>` + `sizes` | Selección de variante por ancho | descriptores `w` |
| `<img src/srcset/sizes>` | Respaldo obligatorio + selector final | — |
| `width`/`height` | Reserva de espacio (CLS ≈ 0) | px reales del layout |
| `loading` / `decoding` | Lazy / decodificación asíncrona | ver [[Lazy Loading]] |
| `Content-Type` (servidor) | MIME correcto | `image/avif` / `image/webp` |
| cwebp / avifenc / sharp | Codificación | WebP `q80`, AVIF `q55` |

## Relacionadas

- [[Html]] — etiquetas y atributos generales
- [[Etiquetas]] — sección de imágenes
- [[Lazy Loading]] — carga diferida de imágenes con `<picture>`
- [[Preload]] — `imagesrcset`/`imagesizes` para el LCP dentro de `<picture>`
- [[Imágenes CSS]] — `image-set()` para imágenes de fondo (sin `<picture>`)
- [[Modernizr]] — detección de soporte AVIF/WebP (hoy innecesaria: `<picture>` lo resuelve sin JS)
