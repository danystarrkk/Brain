---
aliases: ["rel=preload", "Precarga de recursos"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Prefetch]]"
  - "[[Lazy Loading]]"
  - "[[Formatos Avif y Webp]]"
  - "[[Etiquetas Meta]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Reference/Elements/link"
---

# Preload

**Preload** le dice al navegador que **descargue ya, con prioridad alta, un recurso de la página actual** sin esperar a que el resto del documento lo descubra. Resuelve el problema del *waterfall* (cascada de descubrimiento): el navegador parsea el HTML, encuentra el CSS, lee el CSS, descubre las fuentes y las imágenes… cada paso espera al anterior. Con `rel="preload"` se salta directo a la descarga.

> [!info] La idea en una frase
> "Este recurso es crítico para lo que el usuario va a ver/interactuar apenas cargue: bajalo **ahora**, aunque todavía no sepas dónde se usa."

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

> [!warning] Precargar sin usar = recurso desperdiciado
> Si el recurso precargado no llega a consumirse en pocos segundos, los navegadores lo descartan y muestran un warning en consola (`The resource ... was preloaded using link preload but not used`), Lighthouse lo marca como oportunidad y el usuario pagó los bytes de más. Precargá solo recursos que **seguro** se usarán.

## `as`: qué se está precargando

El atributo `as` es **obligatorio**: le indica al navegador el tipo de recurso, para asignarle la prioridad correcta y la cola de descarga adecuada.

| `as` | Recurso | Detalle |
|---|---|---|
| `style` | CSS | El caso más común |
| `script` | JavaScript | Usar luego con `defer`/`async` — ver [[Html]] |
| `font` | Fuentes web | **Siempre requiere `crossorigin`**, aunque sea same-origin |
| `image` | Imágenes | Ideal para el LCP; combinar con `imagesrcset`/`imagesizes` |
| `fetch` | Datos vía fetch/XHR | APIs, JSON |
| `document` | HTML | Navegaciones dentro de iframes |

## Fuentes web: el caso clásico (y su trampa)

Las fuentes declaradas en `@font-face` dentro de un CSS se descubren tarde (cuando el navegador ya parseó el CSS). Precargarlas acelera el texto visible. Ejemplo con Google Fonts:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<link rel="preload" as="style"
      href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400;700&display=swap">
<link rel="stylesheet"
      href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400;700&display=swap">
```

Y cuando se conoce el archivo de fuente directamente, precargar el `.woff2`:

```html
<link rel="preload" as="font" type="font/woff2" crossorigin
      href="/fonts/open-sans-400.woff2">
```

> [!warning] `crossorigin` es obligatorio en fuentes
> Las fuentes web se solicitan **siempre** en modo CORS. Sin `crossorigin` en el preload, la petición del `@font-face` posterior no reconoce la precarga y el navegador descarga el archivo **dos veces**. Incluso para fuentes del mismo origen hay que escribirlo.

> [!tip] `display=swap` y una sola familia
> Precargá como máximo **una** fuente (la crítica del body): cada precarga compite con el CSS y el JS principales. `display=swap` muestra texto de reserva mientras llega, así el render no queda bloqueado — ver [[Etiquetas Meta]] para el resto de configuración del `<head>`.

## Imágenes LCP con `imagesrcset`

Para la imagen que domina la primera pantalla, preload responsivo declarando variantes directamente en el `<link>`:

```html
<link rel="preload" as="image"
      href="img/hero-800.jpg"
      imagesrcset="img/hero-800.jpg 800w, img/hero-1600.jpg 1600w"
      imagesizes="100vw">
```

`imagesrcset`/`imagesizes` del `<link>` funcionan igual que los atributos del `<img>`: el navegador elige la variante según el ancho del viewport y la densidad de píxeles — el mismo mecanismo que en [[Formatos Avif y Webp]].

## `modulepreload`: ES modules

Los módulos ES se cargan encadenados (import → import → import), otro *waterfall* clásico. `modulepreload` adelanta módulos sabiendo que se usarán:

```html
<script type="module" src="/js/app.js"></script>
<link rel="modulepreload" href="/js/api.js">
<link rel="modulepreload" href="/js/componentes.js">
```

## Preload vs Prefetch vs Lazy loading

| Técnica | Página | Prioridad | Para qué |
|---|---|---|---|
| **Preload** (`rel="preload"`) | La **actual** | Alta | Recursos críticos descubiertos tarde |
| **Prefetch** (`rel="prefetch"`) | Una **futura** | Baja | Anticipar la siguiente navegación — ver [[Prefetch]] |
| **Lazy loading** (`loading="lazy"`) | La actual | Baja | Diferir recursos no críticos — ver [[Lazy Loading]] |

## Cuándo usarlo (y cuándo no)

**Sí:**

- Fuentes críticas declaradas en `@font-face` de CSS.
- Imagen LCP del hero.
- CSS/JS crítico descubierto tarde (generado por JS, incluido dinámicamente).
- Cualquier recurso sin el cual la primera pantalla queda incompleta.

**No:**

- Recursos que ya están al inicio del `<head>` en `<link>` directo: el navegador los descubre igual de rápido.
- "Todo lo que se ve en la página": las precargas se pelean entre sí y con el render; priorizá.
- Recursos diferibles: para eso está el [[Lazy Loading]].

> [!tip] Cómo verificar el efecto
> DevTools → pestaña Network: si el recurso figura como *"Initiator: preload"* y sube al principio de la lista, funciona. Lighthouse penaliza precargas que no se usan y precargas de baja prioridad mal declaradas.

## Relacionadas

- [[Prefetch]] — anticipar la *siguiente* navegación en tiempo ocioso
- [[Lazy Loading]] — lo contrario: diferir lo no crítico
- [[Formatos Avif y Webp]] — variantes responsivas con `imagesrcset`/`imagesizes`
- [[Etiquetas Meta]] — el resto de configuración del `<head>`
- [[Html]] — el elemento `<link>` y sus atributos
