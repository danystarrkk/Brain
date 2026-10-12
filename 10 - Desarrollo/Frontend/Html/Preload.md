---
tags:
  - desarrollo
  - frontend
  - preload
tipo: concepto
estado: terminado
---

# Preload

**Preload** le dice al navegador que **descargue ya, con prioridad alta, un recurso de la página actual** sin esperar a que el resto del documento lo descubra. Resuelve el problema del *waterfall* (cascada de descubrimiento): el navegador parsea el HTML, encuentra el CSS, lee el CSS, descubre las fuentes y las imágenes… cada paso espera al anterior. Con `rel="preload"` se salta directo a la descarga.

> [!info] La idea en una frase
> "Este recurso es crítico para lo que el usuario va a ver/interactuar apenas cargue: bajalo **ahora**, aunque todavía no sepas dónde se usa."

## Diagnóstico del waterfall: cuándo el preload realmente ayuda

El pipeline de carga sin hints se ve así (cada fila espera a la anterior):

| Paso | Recurso | Espera a |
|---|---|---|
| 1 | HTML document | — (conexión) |
| 2 | CSS descubierto en el HTML | Parsing del HTML |
| 3 | Fuentes descubiertas dentro del `@font-face` del CSS | Descarga + parseo del CSS |
| 4 | Imagen hero descubierta en el CSS/HTML tardío | Descarga del CSS / render |
| 5 | JS deferred | Parsing del HTML |

Cada eslabón suma latencia (red + parseo). Ejemplo concreto (típico de una landing en 4G de ~50 ms de RTT):

| Recurso | Sin preload (llega en) | Con preload | Por qué |
|---|---|---|---|
| `font.woff2` | ~900 ms (descubierta al parsear el CSS) | ~450 ms | Se descarga desde el paso 1 en paralelo |
| Imagen hero | ~800 ms (tras render del CSS) | ~400 ms | El `<link>` la descubre el parser del HTML |
| Módulo `api.js` | ~1200 ms (tras ejecutar `app.js`) | ~500 ms | `modulepreload` achata el grafo ESM |

El preload ataca **eslabones 3–5**: si sabés desde el HTML qué va a hacer falta, la descarga empieza en el paso 1.

> [!tip] Regla de diagnóstico
> Abrí DevTools → Network y mirá el **Initiator** del recurso crítico. Si dice *"Parser"* o *"Stylesheet"* tardío, y la request arranca 500 ms+ después del documento, es candidato a preload. Si arranca en el primer milisegundo junto al HTML, el preload no aporta nada.

## El patrón de dos pasos

El preload solo **adelanta la descarga**; después hay que usar realmente el recurso. El patrón clásico para CSS: declararlo dos veces, primero como preload y luego como hoja de estilo normal:

```html
<!-- 1. Precarga con prioridad alta -->
<link rel="preload" href="css/normalize.css" as="style">
<!-- 2. Uso real (la petición ya viene de la caché) -->
<link href="css/normalize.css" rel="stylesheet">
```

```html
<link rel="preload" href="css/style.css" as="style">
<link href="css/style.css" rel="stylesheet">
```

La segunda etiqueta no vuelve a descargar nada: el navegador reutiliza el archivo que la precarga ya trajo.

También se puede inyectar desde servidor o incluso desde JS (último recurso — el `<link>` en HTML es lo más temprano posible sin cabeceras):

```js
const l = document.createElement('link');
l.rel = 'preload';
l.as = 'font';
l.href = '/fonts/open-sans-400.woff2';
l.crossOrigin = 'anonymous';
document.head.appendChild(l);
```

Y por cabecera HTTP, que adelanta la hint **incluso antes de enviar el HTML**:

```http
Link: </css/style.css>; rel=preload; as=style
```

> [!warning] Precargar sin usar = recurso desperdiciado
> Si el recurso precargado no llega a consumirse en pocos segundos, los navegadores lo descartan y muestran un warning en consola (`The resource ... was preloaded using link preload but not used`), Lighthouse lo marca como oportunidad y el usuario pagó los bytes de más. Precargá solo recursos que **seguro** se usarán.

## `as`: qué se está precargando

El atributo `as` es **obligatorio**: le indica al navegador el tipo de recurso, para asignarle la prioridad correcta, la cola de descarga y las cabeceras (`Sec-Fetch-Dest`) adecuadas.

| `as` | Recurso | `type` recomendado | `crossorigin` | Notas |
|---|---|---|---|---|
| `style` | CSS | `text/css` | No (same-origin) | El caso más común; usar con el patrón de dos pasos |
| `script` | JavaScript | `text/javascript` | Depende del uso final | Usar luego con `defer`/`async` — ver [[Html]] |
| `font` | Fuentes web | `font/woff2` | **Siempre** (ver trampa) | Incluso same-origin |
| `image` | Imágenes | `image/avif`, `image/webp`, `image/jpeg` | Si es CORS | Ideal para el LCP; combinar con `imagesrcset`/`imagesizes` |
| `fetch` | Datos vía fetch/XHR | `application/json`… | Si es CORS | APIs críticas del arranque |
| `document` | HTML | `text/html` | — | Navegaciones dentro de iframes |
| `worker` | Web Workers | — | `anonymous` | Workers necesarios ya al arrancar |
| `audio` / `video` / `track` / `embed` / `object` | Media | tipo MIME | Según origen | Poco usados; players específicos |

El `type` opcional tiene un poder extra: si el navegador **no** soporta ese MIME (p. ej. `type="image/avif"` en un navegador sin AVIF), **descarta la hint y no gasta bytes** — la forma correcta de precargar formatos negociados.

## Fuentes web: el caso clásico (y su trampa)

Las fuentes declaradas en `@font-face` dentro de un CSS se descubren tarde (cuando el navegador ya parseó el CSS). Precargarlas acelera el texto visible (un FOIT/FOUT más corto). Ejemplo con Google Fonts:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<link rel="preload" as="style"
      href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400;700&display=swap">
<link rel="stylesheet"
      href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400;700&display=swap">
```

Y cuando se conoce el archivo de fuente directamente (recomendado: autoalojar, ver [[Fuentes Externas]]), precargar el `.woff2`:

```html
<link rel="preload" as="font" type="font/woff2" crossorigin
      href="/fonts/open-sans-400.woff2">
```

> [!warning] `crossorigin` es obligatorio en fuentes
> Las fuentes web se solicitan **siempre** en modo CORS (`Sec-Fetch-Dest: font`, modo anónimo). Sin `crossorigin` en el preload, la petición del `@font-face` posterior **no reconoce la precarga** (la trata como otra request) y el navegador descarga el archivo **dos veces**. Incluso para fuentes del mismo origen hay que escribirlo. Mismo truco aplica a cualquier recurso que se pida con modo CORS.

> [!tip] `display=swap` y una sola familia
> Precargá como máximo **una** fuente (la crítica del body): cada precarga compite con el CSS y los JS principales. `display=swap` muestra texto de reserva mientras llega, así el render no queda bloqueado — ver [[Etiquetas Meta]] para el resto de configuración del `<head>`.

## Imágenes LCP con `imagesrcset`

Para la imagen que domina la primera pantalla, preload responsivo declarando variantes directamente en el `<link>`:

```html
<link rel="preload" as="image" type="image/avif"
      href="img/hero-800.avif"
      imagesrcset="img/hero-800.avif 800w, img/hero-1600.avif 1600w"
      imagesizes="100vw">
```

- `imagesrcset`/`imagesizes` del `<link>` funcionan igual que los atributos del `<img>`: el navegador elige la variante según el ancho del viewport y la densidad de píxeles — el mismo mecanismo que en [[Formatos Avif y Webp]].
- `type="image/avif"` + el `href` de respaldo hacen que navegadores sin AVIF descarten la hint sin bytes.
- **Coherencia obligatoria**: la variante precargada debe coincidir con la que `<picture>`/`<img>` elegirán después; si el `<img>` sirve WebP y el preload trajo AVIF, se descargan **dos veces** (y consola de warning).
- La combinación reina del LCP: imagen eager + preload con `imagesrcset` + `fetchpriority="high"` en el `<img>`. Nada de `loading="lazy"` en el hero — ver [[Lazy Loading]].

## `modulepreload`: ES modules

Los módulos ES se cargan encadenados (import → import → import), otro *waterfall* clásico. `modulepreload` adelanta módulos sabiendo que se usarán:

```html
<script type="module" src="/js/app.js"></script>
<link rel="modulepreload" href="/js/api.js">
<link rel="modulepreload" href="/js/componentes.js">
```

Detalles:

- A diferencia del preload normal, el navegador **ya conoce el grafo**: los `import` posteriores matchean la precarga y no re-piden (funciona como la caché de módulos).
- Bundlers modernos generan estas etiquetas automáticamente (`modulePreload` en Vite, `experiments.outputModule` en Rollup); en mano, precarga solo la rama profunda que de otro modo se descubriría tarde.
- Acepta `crossorigin` para módulos cross-origin (el default de los modules es CORS anónimo).

## `fetchpriority` vs preload

Dos palancas distintas que suelen combinarse:

| Mecanismo | ¿Cuándo descarga? | ¿Qué cambia? |
|---|---|---|
| `rel="preload"` | **Ya**, y en qué punto del parseo | Descubrimiento temprano |
| `fetchpriority="high"` (atributo en `<img>`, `<link>`, `<script>`) | Cuando el navegador ya lo descubrió | **Cola de prioridad** entre requests existentes |

```html
<img src="hero.jpg" alt="Hero" width="1600" height="900"
     fetchpriority="high" decoding="async">
```

Preload sin prioridad sigue siendo mejor que nada; prioridad sin preload ayuda si el recurso se descubre "temprano-ish". Para el LCP: ambos.

## Preload vs Prefetch vs Lazy loading

| Técnica | Página | Prioridad | Para qué |
|---|---|---|---|
| **Preload** (`rel="preload"`) | La **actual** | Alta | Recursos críticos descubiertos tarde |
| **Prefetch** (`rel="prefetch"` / Speculation Rules) | Una **futura** | Baja | Anticipar la siguiente navegación — ver [[Prefetch]] |
| **Lazy loading** (`loading="lazy"`) | La actual | Baja | Diferir recursos no críticos — ver [[Lazy Loading]] |

## Cuándo usarlo (y cuándo no)

**Sí:**

- Fuentes críticas declaradas en `@font-face` de CSS (siempre con `crossorigin`).
- Imagen LCP del hero (con `imagesrcset` si es responsive).
- CSS/JS crítico descubierto tarde (generado por JS, incluido dinámicamente, o descubierto tras el parseo de otro CSS).
- Módulos profundos del grafo ESM (`modulepreload`).
- Cualquier recurso sin el cual la primera pantalla queda incompleta.

**No:**

- Recursos que ya están al inicio del `<head>` en `<link>` directo: el navegador los descubre igual de rápido.
- "Todo lo que se ve en la página": las precargas se pelean entre sí y con el render; priorizá (ver sobre-preload abajo).
- Recursos diferibles: para eso está el [[Lazy Loading]].
- Fuentes decorativas o solo-weight-italic de segunda página.

### El sobre-preload: más hints, peor LCP

Precargar en exceso invierte la prioridad contra el propio render:

- Cada preload entra en la cola con prioridad alta y **compite con el HTML/CSS del render** y con la imagen LCP en sí.
- En móvil (ancho de banda y CPU limitados), 5–6 preloads simultáneos pueden **retrasar** el LCP que pretendían acelerar.
- Señal de alerta: Lighthouse "Preload Largest Contentful Paint image" *junto con* "Eliminate critical resources" / requests de prioridad alta al inicio.
- Presupuesto sano: **1–2 preloads por página** (la fuente crítica y/o el hero). Si necesitás más, casi siempre hay un descubrimiento tardío real que arreglar en el orden del `<head>`.

## Verificación con DevTools

1. **Network**: la request precargada aparece al inicio de la waterfall; en la columna *Initiator* figura `preload` (o el `<link>`).
2. **Priority**: "High" para `as="style"/"script"/"font"/"image"` precargados — confirma que la cola es la esperada.
3. **Consola**: el mensaje *"preloaded but not used within a few seconds"* denuncia hints huérfanos: o el recurso no se usa, o la segunda etiqueta del patrón de dos pasos apunta a otra URL.
4. **Lighthouse**: las auditorías "Preload Key Requests" y "LCP" detectan tanto candidatos faltantes como excesos.
5. **Doble descarga**: si la waterfall muestra **dos** requests al mismo archivo (o a variantes distintas del mismo recurso), el preload no matchea el uso final (tipicamente `crossorigin` o formato distintos).

## Errores comunes

1. **Precargar sin `as`**: la hint se ignora (consola la reporta); `as` es obligatorio.
2. **`as="font"` sin `crossorigin`** (aunque sea same-origin): descarga doble — la trampa más famosa del preload.
3. **Precargar un recurso que no se consume**: warning + bytes perdidos + penalización de Lighthouse.
4. **Mismatch de formato/variante** entre el preload de imagen y el `<picture>` final: doble descarga. Usar `imagesrcset` + `type` coherentes.
5. **Preload de todo**: sobre-preload = competencia de ancho de banda contra el LCP. Máximo 1–2.
6. **Precargar recursos ya tempranos**: el `<link rel="stylesheet">` en el primer bloque del `<head>` no necesita preload; el descubrimiento es inmediato.
7. **Olvidar `defer` en el script precargado**: el preload trae bytes con prioridad alta, pero un `<script>` sin `defer` bloquea el parseo igual — el hint acelera la red, no cambia la semántica de ejecución.
8. **Preload en páginas con `loading="lazy"` generalizado**: si casi todo es diferido, no hay waterfall que arreglar; el preload solo añade ruido.

## Accesibilidad

- Un preload mal apuntado puede **retrasar** el primer render (compitiendo con el CSS que pinta la página): en términos de UX eso afecta a todos los usuarios, y más a quienes van lento. Precarga con criterio = accesibilidad de rendimiento.
- Las fuentes precargadas reducen el parpadeo de texto (FOUT/FOIT); combinar con `font-display: swap` para que el texto de reserva sea visible de inmediato — ver [[Fuentes Externas]].
- El `<link rel="preload">` no toca el árbol de accesibilidad ni el DOM visible: es 100% hint de red.

## Rendimiento y compatibilidad

- `rel="preload"`: universal en navegadores actuales (Chrome 50+, Firefox 57+, Safari 13+).
- `imagesrcset`/`imagesizes` en `<link>`: llegaron más tarde y de forma despareja (Safari/Chrome modernos ✔; verificar soporte antes de depender — el `href` simple es el respaldo siempre disponible).
- `modulepreload`: Chromium 66+, Firefox 115+, Safari 15.4+.
- `fetchpriority`: Chromium 73+; llegó a Firefox/Safari en versiones recientes — degradación segura donde falta (solo se ignora la prioridad).
- La cabecera `Link` de preload funciona donde la etiqueta funciona, y antes: es la ventaja de aplicarla en el servidor (nginx: `add_header Link ...`).

## Cheatsheet

| Pieza | Sintaxis | Función |
|---|---|---|
| Preload base | `<link rel="preload" href="…" as="style">` | Descarga inmediata, prioridad alta |
| Fuentes | `as="font" type="font/woff2" crossorigin` | Evita la doble descarga |
| Imagen LCP | `as="image" imagesrcset="…" imagesizes="…"` | Variante responsive correcta |
| Formato negociado | `type="image/avif"` | Descarta hint si no hay soporte |
| Módulos | `<link rel="modulepreload" href="…">` | Achata el grafo ESM |
| Prioridad sin hint | `fetchpriority="high"` en `<img>`/`<script>` | Sube en la cola existente |
| Por servidor | `Link: </x.css>; rel=preload; as=style` | Antes que el HTML |
| Señal de error | consola: *"preloaded but not used"* | Quitar la hint o arreglar el uso |

## Relacionadas

- [[Prefetch]] — anticipar la *siguiente* navegación en tiempo ocioso
- [[Lazy Loading]] — lo contrario: diferir lo no crítico
- [[Formatos Avif y Webp]] — variantes responsivas con `imagesrcset`/`imagesizes`
- [[Fuentes Externas]] — `@font-face`, `font-display` y el auto-alojamiento que hace viable el preload de fuentes
- [[Etiquetas Meta]] — el resto de configuración del `<head>`
- [[Html]] — el elemento `<link>` y sus atributos
