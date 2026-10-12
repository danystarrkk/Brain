---
tags:
  - desarrollo
  - frontend
  - etiquetas-meta
tipo: concepto
estado: terminado
---

# Etiquetas Meta

Las etiquetas `meta` son una especie de **configuración inicial** que el navegador (y los buscadores) interpretan para leer correctamente el resto del HTML. Todas van dentro del `<head>` del documento — ver [[Estructurar Html]].

```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="description" content="Página web de un Blog de Café">
```

## Anatomía: las tres formas de `<meta>`

El elemento tiene tres sintaxis según el atributo que la distingue:

| Forma | Sintaxis | Uso |
|---|---|---|
| `charset` | `<meta charset="UTF-8">` | Codificación de caracteres |
| `name` + `content` | `<meta name="description" content="…">` | Metadatos con nombre estándar |
| `property` + `content` | `<meta property="og:title" content="…">` | Open Graph y otras taxonomías con prefijo |

`http-equiv` es una cuarta forma heredada del HTTP (ver más abajo). Reglas comunes: el charset **no** admite `content`; `name` y `property` no se mezclan; los valores van entre comillas dobles como todo HTML.

> [!tip] El navegador ignora lo que no entiende
> Una meta con nombre inventado no rompe nada, pero tampoco aporta: los robots y navegadores leen solo los nombres que conocen. No hay "metas secretas de posicionamiento" — cualquier vendor que te las venda está mintiendo.

## Las esenciales

| Meta | Qué hace |
|---|---|
| `<meta charset="UTF-8">` | Define la codificación de caracteres; sin ella, los acentos y ñ pueden mostrarse como caracteres raros. **Debe ir en las primeras líneas del `<head>`** — dentro de los primeros 1024 bytes del documento. |
| `<meta name="viewport" content="width=device-width, initial-scale=1">` | Hace que el sitio se renderice al ancho real del dispositivo; **sin ella, los móviles muestran la página "encogida"** a ~980px. Imprescindible para responsive — ver [[Responsive Web Design]]. |
| `<meta name="description" content="…">` | Descripción que los buscadores suelen mostrar en los resultados (SERP). No es directamente factor de ranking, pero afecta el CTR: escribila como un anuncio de ~155 caracteres. |
| `<title>` (en `<head>`, pero no es `meta`) | El título real de la SERP y de la pestaña — la etiqueta con más peso on-page para el título mostrado. |

### Charset: la regla de los 1024 bytes

> [!warning] El charset va primero
> El `<meta charset>` debe aparecer dentro de los **primeros 1024 bytes** del documento (prácticamente: primera línea del `<head>`). Si aparece tarde, el navegador puede haber parseado parte del archivo con otra codificación y mostrar basura. El parser detecta el charset por esta cascada: 1) header `Content-Type` del servidor → 2) autodetección byte-order mark (BOM) → 3) la meta dentro de los primeros 1024 bytes → 4) heurística. **UTF-8 sin BOM** es el estándar; servirlo también desde el servidor es la capa correcta.

### Viewport: desglose de los tokens

```
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

| Token | Qué controla | Valores típicos |
|---|---|---|
| `width` | Ancho del viewport virtual | `device-width` (el ancho físico), un número fijo (evitar) |
| `initial-scale` | Zoom inicial | `1` |
| `minimum-scale` / `maximum-scale` | Rango de zoom | **No bloquear**: fijar `maximum-scale=1` o `user-scalable=no` impide zoom y viola WCAG 1.4.4 |
| `viewport-fit` | Cómo cubre las "notches" | `cover` + safe-area insets con `env()` |
| `interactive-widget` | Dónde va el teclado virtual (Chrome) | `resizes-content` |

> [!warning] Prohibido bloquear el zoom
> `user-scalable=no` / `maximum-scale=1` rompen el zoom de usuarios con baja visión (WCAG 1.4.4 falla). Si tu diseño "se rompe con zoom", el problema es el diseño, no el usuario.

### Description: la miniatura publicitaria

```html
<meta name="description" content="Guía práctica de café en casa: molienda, extracción y limpieza. Recetas y errores comunes del espresso casero.">
```

- Longitud útil: **~120–158 caracteres** (Google trunca alrededor de 155–160 px).
- Cada página la suya; descripciones genéricas no ayudan al CTR.
- Ausente: los buscadores inventan el snippet a partir del contenido — a veces mejor, a veces peor.

## SEO avanzado: robots, canónica y más

```html
<meta name="robots" content="index, follow">
<link rel="canonical" href="https://ejemplo.com/blog/cafe">
<meta name="googlebot" content="noimageindex">
```

### `robots` (y sus hermanos)

| Valor del `content` | Efecto |
|---|---|
| `index` / `noindex` | ¿Indexar la página? (defecto: `index`) |
| `follow` / `nofollow` | ¿Seguir sus enlaces? (defecto: `follow`) |
| `noarchive` | No mostrar versión cacheada |
| `nosnippet` | No mostrar snippet ni rich result |
| `max-snippet:-1` | Sin límite de longitud del snippet |
| `max-image-preview:large` | Permitir previews grandes (rich results) |
| `noimageindex` | No indexar las imágenes de la página |
| `unavailable_after: 2026/12/31` | Excluir tras la fecha |

- `googlebot`, `bingbot`, etc. aceptan los mismos valores para directives por crawler (poco usados).
- `noindex` efectivo requiere que la página sea **rastreable** (un `nofollow` global o `robots.txt` bloqueando impide que el crawler lea la meta).
- Para desautorizar enlaces salientes a nivel global existe el header `X-Robots-Tag` — la meta no cubre eso.

### Canónica

- `<link rel="canonical">` (¡es `<link>`, no `<meta>`!) declara la URL oficial cuando la misma página se sirve desde varias URLs (parámetros de tracking, `http` vs `https`, `/ruta` vs `/ruta/`).
- Debe ser absoluta (`https://…`), autoreferente en la página canónica, y coherente con sitemap y redirecciones 301.

## Open Graph (og:) y Twitter Cards

Configuran cómo se ve el enlace cuando alguien lo comparte en redes o mensajería:

```html
<!-- Open Graph (Facebook, LinkedIn, WhatsApp, Telegram…) -->
<meta property="og:type" content="website">
<meta property="og:url" content="https://ejemplo.com/blog/cafe">
<meta property="og:title" content="Blog de Café">
<meta property="og:description" content="Notas sobre café de especialidad">
<meta property="og:image" content="https://ejemplo.com/img/portada.jpg">
<meta property="og:image:alt" content="Taza de café sobre mesa de madera">
<meta property="og:locale" content="es_AR">
<meta property="og:site_name" content="Blog de Café">

<!-- Twitter/X Cards -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@usuario">
<meta name="twitter:title" content="Blog de Café">
<meta name="twitter:description" content="Notas sobre café de especialidad">
<meta name="twitter:image" content="https://ejemplo.com/img/portada.jpg">
```

| Propiedad | Valores / notas |
|---|---|
| `og:type` | `website`, `article`, `product`, `video`… (con `article` se añaden `article:author`, `article:published_time`) |
| `og:url` | URL canónica del objeto |
| `og:image` | Debe ser **absoluta**; recomendado 1200×630px; acepta varias etiquetas para variantes |
| `og:image:alt` | Texto alternativo de la imagen — accesibilidad en redes |
| `twitter:card` | `summary` (imagen chica ~144×144) o `summary_large_image` (1200×630) |
| `twitter:site` / `twitter:creator` | Cuentas atribuidas |

- Fallback: si falta una propiedad Twitter, se usa el equivalente OG — por eso el set OG bien hecho alimenta ambas plataformas.
- La imagen `og:image` suele ser la misma que se sirve en formatos modernos (ver [[Formatos Avif y Webp]]); **ojo**: no todos los crawlers de redes aceptan AVIF, conviene un JPG/PNG de respaldo (o servir JPG específicamente en og:image).
- Validación: Facebook Sharing Debugger, Card Validator de X/LinkedIn Post Inspector.

## Otras comunes

```html
<meta name="author" content="Daniel">
<meta name="generator" content="WordPress 6.x">
<meta name="theme-color" content="#1a1a2e" media="(prefers-color-scheme: dark)">
<meta name="color-scheme" content="light dark">
<meta name="referrer" content="strict-origin-when-cross-origin">
<meta http-equiv="refresh" content="5; url=nueva-pagina.html">
```

- **`theme-color`**: tiñe la barra del navegador en móviles/escritorio con el color de la marca; admite `media` para variantes claro/oscuro.
- **`color-scheme`**: le dice al navegador qué esquemas acepta la página — permite que `color-scheme: dark` haga scrollbars y widgets nativos oscuros (ver [[Responsive Web Design]]).
- **`referrer`**: política de Referrer-Policy por página (alternativa al header).
- **`http-equiv="refresh"`**: redirección o refresco programado — **evitarlo** si es posible: rompe el botón "atrás", es malo para accesibilidad (cambios inesperados, WCAG 2.2.1) y SEO; una redirección real se hace en el servidor con 301.

### `http-equiv`: los únicos usos legítimos hoy

`http-equiv` simula un header HTTP en el documento. Sus usos reales se reducen a:

| Valor | Uso | Alternativa mejor |
|---|---|---|
| `refresh` | Refresco/redirección (ver arriba) | HTTP 301 / JS controlado |
| `X-UA-Compatible` | Forzar modo IE (legacy) | `<!DOCTYPE html>` solo |
| `Content-Type` | Charset heredado | `<meta charset>` |

> [!info] Casi todo `http-equiv` es legado
> Los headers de verdad (CSP, HSTS, charset, referrer) se configuran en el servidor. `http-equiv` sobrevive solo donde no controlás headers (hosting estático gratuito); aún así, `refresh` y `X-UA-Compatible` en código nuevo son señal de deuda.

## Metadatos estructurados: JSON-LD

Los buscadores modernos entienden **entidades** (recetas, productos, eventos, FAQs) gracias a datos estructurados. El formato recomendado por Google es **JSON-LD**: un bloque `<script type="application/ld+json">` con datos en [[Formato Json]] siguiendo el vocabulario [schema.org](https://schema.org/):

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Cómo tostar café en casa",
  "author": { "@type": "Person", "name": "Daniel" },
  "datePublished": "2026-03-12",
  "image": "https://ejemplo.com/img/cafe.jpg"
}
</script>
```

- Se coloca en el `<head>` o al final del `<body>`; no afecta el render (no es ejecutable).
- Debe **coincidir con el contenido visible** de la página (inconsistencia = penalización de rich results).
- Tipos útiles: `Article`, `BreadcrumbList`, `Product`, `Recipe`, `FAQPage`, `LocalBusiness`, `Organization`, `Person`, `VideoObject`. Validar con el [Rich Results Test](https://search.google.com/test/rich-results) de Google.
- Las microdata (`itemprop`) y RDFa son alternativas más verbales; JSON-LD es el estándar de facto.

## Meta por frameworks

React y otros emiten estas metas desde el framework, no a mano:

- **React**: `<Helmet>` o su equivalente (react-helmet-async) para `title`, `description`, OG por ruta — ver [[ReactJS - Fundamentos]].
- **Next.js / Astro / Nuxt**: frontmatter o APIs `defineOgMeta` que generan el `<head>` en el render del servidor.

En SSR/SSG las metas llegan en el HTML inicial (bien para crawlers); en CSR puro los buscadores ven el DOM renderizado — Google renderiza JS, pero otros crawlers no siempre.

## Las metas no son magia de SEO

> [!tip] Jerarquía real del SEO on-page
> 1. Contenido con buena semántica y encabezados claros (ver [[Etiquetas]] y [[Estructurar Html]]).
> 2. Velocidad y Core Web Vitals (ver [[Preload]], [[Prefetch]], [[Lazy Loading]]).
> 3. Enlaces (internos y externos).
> 4. Recién ahí: title, description y metas — que **amplifican** lo anterior pero no lo reemplazan.

## Errores comunes

1. **Charset después del CSS o de scripts** → puede exceder los 1024 bytes y provocar basura de acentuación.
2. **`og:image` relativa** → los scrapers no la resuelven; siempre URL absoluta.
3. **`noindex` en staging y olvidarlo en producción** → sitio invisible; clásico de deploys.
4. **Description idéntica en todo el sitio** → Google la ignora como duplicada y genera snippets pobremente.
5. **Canonical apuntando a otra URL** → la página deja de competir por sus propias palabras clave.
6. **`maximum-scale=1`** por diseño "a prueba de zoom" → infracción WCAG.
7. **Poner `keywords`** (`<meta name="keywords">`): los buscadores la ignoran desde 2009; es ruido.

## Cheatsheet

| Meta | Forma | Destinatario |
|---|---|---|
| charset | `<meta charset="UTF-8">` | Navegador (primeros 1024 bytes) |
| viewport | `name="viewport"` | Navegador móvil |
| description | `name="description"` | Buscadores → snippet/CTR |
| robots | `name="robots"` | Crawlers (index/nofollow…) |
| canonical | `<link rel="canonical">` | Crawlers (URL oficial) |
| og:* | `property="og:…"` | Redes sociales |
| twitter:* | `name="twitter:…"` | Twitter/X |
| theme-color | `name="theme-color"` | UI del navegador |
| color-scheme | `name="color-scheme"` | Navegador (claro/oscuro) |
| refresh | `http-equiv="refresh"` | Evitar (usar 301) |
| JSON-LD | `<script type="application/ld+json">` | Buscadores (entidades) |

## Relacionadas

- [[Html]] — anatomía del documento y el `head`
- [[Estructurar Html]] — estructura completa de una página
- [[Etiquetas]] — catálogo de etiquetas de contenido
- [[Responsive Web Design]] — la meta `viewport` y el diseño responsive
- [[Formatos Avif y Webp]] — servir imágenes modernas para compartir en redes
- [[Formato Json]] — el formato del JSON-LD estructurado
- [[Preload]] — otros `<link>` de configuración del `<head>`
- [[Lazy Loading]] — hints de carga que complementan a las metas
