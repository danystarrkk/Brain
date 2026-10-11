---
aliases: ["Meta tags", "Metaetiquetas"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Estructurar Html]]"
  - "[[Responsive Web Design]]"
  - "[[Formatos Avif y Webp]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Element/meta"
---

# Etiquetas Meta

Las etiquetas `meta` son una especie de **configuración inicial** que el navegador (y los buscadores) interpretan para leer correctamente el resto del HTML. Todas van dentro del `<head>` del documento — ver [[Estructurar Html]].

```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="description" content="Página web de un Blog de Café">
```

## Las esenciales

| Meta | Qué hace |
|---|---|
| `<meta charset="UTF-8">` | Define la codificación de caracteres; sin ella, los acentos y ñ pueden mostrarse como caracteres raros. Debe ir en las primeras líneas del `<head>`. |
| `<meta name="viewport" content="width=device-width, initial-scale=1">` | Hace que el sitio se renderice al ancho real del dispositivo; **sin ella, los móviles muestran la página "encogida"**. Imprescindible para responsive — ver [[Responsive Web Design]]. |
| `<meta name="description" content="…">` | Descripción que los buscadores suelen mostrar en los resultados (SERP). No es directamente un factor de ranking, pero afecta el CTR: escribila como un anuncio de ~155 caracteres. |

## SEO y compartir

```html
<meta name="robots" content="index, follow">
<link rel="canonical" href="https://ejemplo.com/blog/cafe">

<meta property="og:title" content="Blog de Café">
<meta property="og:description" content="Notas sobre café de especialidad">
<meta property="og:image" content="https://ejemplo.com/img/portada.jpg">
<meta name="twitter:card" content="summary_large_image">
```

- **`robots`**: controla si la página se indexa (`index`/`noindex`) y si se siguen sus enlaces (`follow`/`nofollow`).
- **`canonical`**: declara la URL oficial del contenido; evita contenido duplicado cuando la misma página se sirve desde varias URLs.
- **Open Graph (`og:*`)** y **Twitter Cards**: configuran cómo se ve el enlace cuando alguien lo comparte en redes o mensajería — la imagen `og:image` suele ser la misma que se sirve en formatos modernos (ver [[Formatos Avif y Webp]]); ojo: no todos los crawlers de redes aceptan AVIF, conviene un JPG/PNG de respaldo.

## Otras comunes

```html
<meta name="author" content="Daniel">
<meta name="theme-color" content="#1a1a2e">
<meta http-equiv="refresh" content="5; url=nueva-pagina.html">
```

- **`theme-color`**: tiñe la barra del navegador en móviles/escritorio con el color de la marca.
- **`http-equiv="refresh"`**: redirección o refresco programado — **evitarlo** si es posible: rompe el botón "atrás", es malo para accesibilidad y SEO; una redirección real se hace en el servidor.

> [!warning] El charset va primero
> El `<meta charset>` debe aparecer dentro de los primeros 1024 bytes del documento (prácticamente: primeras líneas del `<head>`). Si aparece tarde, el navegador puede haber parseado parte del archivo con otra codificación y mostrar basura.

> [!tip] Las metas no son magia de SEO
> Una buena `description` y un `canonical` correctos ayudan, pero el posicionamiento real viene del contenido con buena semántica (ver [[Etiquetas]] y [[Estructurar Html]]) y de la velocidad del sitio (ver [[Preload]]).

## Relacionadas

- [[Html]] — anatomía del documento y el `head`
- [[Estructurar Html]] — estructura completa de una página
- [[Etiquetas]] — catálogo de etiquetas de contenido
- [[Responsive Web Design]] — la meta `viewport` y el diseño responsive
- [[Formatos Avif y Webp]] — servir imágenes modernas para compartir en redes
- [[Preload]] — otros `<link>` de configuración del `<head>`
