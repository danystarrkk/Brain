---
aliases: ["Etiquetas HTML", "Tags HTML"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Estructurar Html]]"
  - "[[Formularios]]"
  - "[[Formatos Avif y Webp]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Element"
---

# Etiquetas en HTML

En HTML tenemos varias etiquetas que podemos usar, y conviene separarlas por categorías según el propósito que cumplen. Una etiqueta es el elemento delimitado por signos menor/mayor que (`< >`) que le dice al navegador **qué significa** el contenido que envuelve — ver [[Html]] para la anatomía completa.

## Tres naturalezas de elemento (el detalle que explica el comportamiento)

Antes del catálogo, conviene distinguir tres familias porque **se comportan distinto ante el parser**:

| Familia | Ejemplos | Regla de cierre | Característica |
|---|---|---|---|
| **Normal** | `<p>`, `<div>`, `<article>` | Apertura + cierre obligatorios | Envuelven contenido que fluye |
| **Void** (vacío) | `<img>`, `<br>`, `<hr>`, `<meta>`, `<link>`, `<input>`, `<source>`, `<track>`, `<wbr>`, `<col>`, `<embed>`, `<param>`, `<area>` | **No tienen cierre**; escribir `</img>` es error | No envuelven contenido |
| **Raw text** (texto crudo) | `<script>`, `<style>`, `<title>`, `<textarea>` | Tienen cierre; **el contenido no se parsea como HTML** | El parser no interpreta etiquetas dentro (salvo la de cierre) |

> [!warning] Raw text: el parser se apaga dentro
> Dentro de `<script>` y `<style>`, las etiquetas **no son etiquetas**: `<p>` es texto. Por eso `document.write("<p>")` funciona pero `<!--` dentro de un script no es un comentario HTML (es el prefijo de `-->` comment legacy de JS). Y dentro de `<textarea>`, escribir `</textarea>` **cierra el campo** — no se puede mostrar literal sin escapar.

Además existe **contenido foráneo** (*foreign content*): `<svg>` y `<math>` insertan nombres de espacio XML dentro del documento HTML. Ahí el parser cambia de reglas: se admite auto-cierre (`<circle/>`), es sensible a mayúsculas (`viewBox`, no `viewbox`) y el contenido se mezcla con el DOM real (los elementos SVG son nodos con API propia — ver [[Document Object Model (DOM)]]).

```html
<svg viewBox="0 0 100 100" width="40" height="40" aria-hidden="true">
  <circle cx="50" cy="50" r="45" fill="teal"/>
</svg>
```

## Categorías de elementos según el modelo de contenido

El spec agrupa los elementos en categorías que determinan **qué puede contener cada uno**:

- **Flow content**: la mayoría del contenido de `<body>` (`p`, `div`, `ul`, `img`, `a`…).
- **Phrasing content**: el "inline" — lo que puede ir dentro de un `<p>` (`span`, `strong`, `em`, `img`, `a`, `br`…).
- **Interactive content**: elementos con estado/activación (`a`, `button`, `input`, `select`, `details`, `video`…).
- **Metadata content**: `<meta>`, `<link>`, `<title>`, `<script>` (casi todo en `<head>`).

Regla práctica: **dentro de un `<p>` solo cabe phrasing content** — meter un `<div>` dentro de un párrafo cierra el párrafo automáticamente.

## Texto

Tenemos diferentes etiquetas que cumplen propósitos específicos al momento de ingresar texto:

### Encabezados

- **`<h1>`–`<h6>`**: los *headings* (encabezados), del 1 al 6; mientras más alto el número, menor su jerarquía. Reglas:
  - Un solo `<h1>` por página; más de un `<h2>` sí, para separar secciones.
  - Sin saltos de nivel (`h1` → `h3` rompe la navegación por encabezados).
  - El nivel indica jerarquía, **no tamaño** — ver [[Estructurar Html]].

### Bloques de texto

- **`<p>`**: párrafo. No admite bloques dentro; los márgenes verticales colapsan (ver [[CSS Introducción]]).
- **`<br>`**: salto de línea (poco espacio semántico: para poesía sí, para separar párrafos no — para eso está `<p>`).
- **`<hr>`**: separador temático (cambio de tema, no meramente visual).
- **`<blockquote>`**: cita larga; `cite="url"` indica la fuente. Para citas cortas, `<q>` (con comillas automáticas según idioma) o `<cite>` para el título de la obra.
- **`<pre>`**: texto preformateado (espacios y saltos respetados) — la base de los bloques de código. Con `<code>`: `<pre><code>…</code></pre>`.
- **`<address>`**: datos de contacto del autor/duenio del artículo o sitio (no es "etiqueta de dirección" genérica).

### Énfasis e inline

- **`<strong>`**: énfasis de **importancia** (navegador muestra negrita; lectores de pantalla alteran la entonación).
- **`<em>`**: énfasis de **énfasis/acentuación** (cursiva semántica).
- **`<b>`**: negrita **sin** importancia semántica (palabra destacada de un índice, nombre de producto).
- **`<i>`**: cursiva sin énfasis (extranjerismo, nombre científico).
- **`<mark>`**: resaltado **relevante** en contexto (término buscado en resultados).
- **`<small>`**: letra chica (copyright, letra pequeña legítima — no lo uses para evadir letra mínima WCAG).
- **`<del>` / `<ins>`**: texto **eliminado/añadido** con `datetime` y `cite`; semántica de edición real.
- **`<s>`**: texto ya no preciso/ya no relevante (tachado decorativo).
- **`<u>`**: subrayado no-idiomático; hoy rara vez se usa (el subrayado choca con los enlaces).
- **`<abbr title="…">`**: abreviatura con expansión.
- **`<sub>`** (subíndice) y **`<sup>`** (superíndice) — fórmulas, notas al pie.
- **`<span>`**: etiqueta **en línea** genérica para aplicar estilos a un fragmento sin significado propio.
- **`<time datetime="…">`**: fecha/hora legible por máquina.
- **`<data value="…">`**: contenido con valor machine-readable cuando no hay atributo nativo.

```html
<p>Este es un <strong>párrafo</strong> con <em>énfasis</em>.</p>
<p><span class="resaltado">Este fragmento</span> tiene estilo propio.</p>
<p>Oferta hasta el <time datetime="2026-04-01">1 de abril</time>.</p>
```

> [!tip] `strong`/`em` vs `b`/`i`
> Visualmente son idénticos por defecto, semánticamente no. Si el énfasis **importa para entender** el texto (advertencia, clave): `strong`/`em`. Si es solo tipografía contextual: `b`/`i`. Nunca uses `b`/`i` por costumbre de "negrita/cursiva" — para eso está CSS.

## Enlaces

- **`<a>`** (*anchor*): el elemento de hipertexto; envuelve phrasing content y crea un enlace.

```html
<a href="https://ejemplo.com">Contenido como texto</a>
<a href="pagina.html"><img src="miniatura.jpg" alt="Miniatura"></a>
<a href="#seccion">Salto interno</a>
<a href="doc.pdf" download>Descargar PDF</a>
<a href="https://ejemplo.com" target="_blank" rel="noopener noreferrer">Nueva pestaña</a>
<a href="mailto:daniel@ejemplo.com">Escribir</a>
<a href="tel:+541140000000">Llamar</a>
```

- `href`: URL o archivo de destino (puede ser `#id`, `mailto:`, `tel:`, `http(s):`, ruta relativa o `data:`).
- `target="_blank"`: nueva pestaña — con `rel="noopener"` (seguridad: evita que la página abierta acceda a `window.opener`) o `noreferrer` si además no queremos enviar el referrer.
- `download`: fuerza descarga en vez de navegar.
- **`<a>` sin `href`** no es un enlace: no es enfocable ni navegable — no lo uses como botón (para acciones hay `<button>`).

## Imágenes y contenido embebido

Conforme se crea un proyecto web vamos a ver que las imágenes son importantes para el diseño:

- **`<img>`**: la etiqueta más usada para imágenes.

```html
<img src="imagen.jpg" alt="Texto alternativo" width="200" height="300" loading="lazy" decoding="async">
```

  - `src`: ruta de la imagen.
  - `alt`: texto alternativo, obligatorio para accesibilidad y SEO; describe la imagen si no carga o con lector de pantalla. `alt=""` vacío explícito = imagen decorativa (el lector la salta).
  - `width`/`height`: dimensiones declaradas en píxeles naturales — evitan saltos de layout mientras carga (ver [[Lazy Loading]]).
  - `srcset`/`sizes`: imágenes responsivas por resolución/ancho — ver [[Formatos Avif y Webp]].
  - `loading="lazy"`, `decoding="async"`, `fetchpriority="high|low"`: control de carga (ver [[Lazy Loading]] y [[Preload]]).
  - `map` + `<area>`: mapa de imagen (*image map*), legado prácticamente extinto.

- **`<picture>`**: contenedor para formatos modernos con respaldo (AVIF/WebP/JPG) y variantes por ancho — ver [[Formatos Avif y Webp]].
- **`<figure>` / `<figcaption>`**: imagen (u otro contenido: código, video, tabla) con su pie de foto, como bloque semántico:

```html
<figure>
  <img src="diagrama.png" alt="Diagrama de arquitectura">
  <figcaption>Arquitectura del sistema</figcaption>
</figure>
```

- **`<video>` / `<audio>`**: multimedia nativa con `controls`, `poster`, `muted`, `loop`, `autoplay` (solo funciona con `muted` en navegadores actuales) y fuentes múltiples:

```html
<video controls poster="portada.jpg" width="640" preload="metadata">
  <source src="clase.webm" type="video/webm">
  <source src="clase.mp4" type="video/mp4">
  <track kind="subtitles" src="es.vtt" srclang="es" label="Español" default>
  Tu navegador no soporta video HTML5.
</video>
```

  `<track>` añade subtítulos/captions (VTT) — la pista `default` es la que se muestra; los captions son requisito WCAG para video con audio.

- **`<iframe>`**: documento embebido; hoy se usa con `sandbox`, `allow` y `loading="lazy"` (embeds de video/mapas):

```html
<iframe src="https://ejemplo.com/embed" title="Mapa de la tienda"
        loading="lazy" sandbox="allow-scripts allow-same-origin"></iframe>
```

  El `title` es obligatorio para accesibilidad (es el nombre accesible del marco).

- **`<embed>` / `<object>`**: contenido externo/plugins — legado (Flash); hoy prácticamente solo `<object>` para PDFs con fallback.
- **`<canvas>`**: lienzo de píxeles pintado por JS; sin fallback semántico (ver [[Acceder a Html]] para dibujar).
- **`<svg>`**: gráficos vectoriales embebidos (inline o por `<img src="x.svg">`); ver arriba sobre contenido foráneo.

## Estructura semántica

Para estructurar nuestra web (detalle completo en [[Estructurar Html]]):

| Etiqueta | Cuándo usarla |
|---|---|
| `<header>` | Parte superior de un sitio o de un elemento |
| `<footer>` | Parte inferior de un sitio o de un elemento |
| `<nav>` | Grupo de enlaces o navegación |
| `<main>` | Contenido principal del archivo HTML; solo uno por archivo |
| `<section>` | Bloque con tema propio encabezado por un heading |
| `<article>` | Contenido independiente y autocontenido |
| `<aside>` | El contenido que acompaña al principal |
| `<div>` | Cuando ninguno de los anteriores aplica (bloque genérico) |

## Listas

Tenemos dos formas de crear listas en HTML: ordenadas y no ordenadas:

- **`<ul>`**: listado **no ordenado** (viñetas).
- **`<ol>`**: listado **ordenado** (numerado). Atributos: `start="5"`, `reversed`, `type="a|A|i|I|1"` (el tipo visual hoy mejor con CSS `list-style-type`).
- **`<li>`**: cada elemento de una lista.
- **`<dl>` / `<dt>` / `<dd>`**: lista de **definiciones** — término y descripción (glosarios, metadatos de features, FAQ).

```html
<ul>
  <li>Esta es</li>
  <li>Una lista no ordenada</li>
</ul>

<ol start="3" reversed>
  <li>Contador descendente desde 3</li>
</ol>

<dl>
  <dt>Espresso</dt>
  <dd>Café extraído con presión</dd>
</dl>
```

> [!tip] Listas de navegación
> Los menús de sitios son listas de enlaces: `<nav><ul><li><a href="...">…` — el patrón semántico estándar que esperan buscadores y lectores de pantalla. CSS elimina las viñetas cuando son decorativas (ver [[CSS Introducción]]).

## Tablas: el modelo completo

Para datos tabulares existe `<table>`. El modelo completo tiene más piezas de las que suelen verse:

| Elemento | Rol |
|---|---|
| `<table>` | Contenedor de la tabla |
| `<caption>` | Título de la tabla (un solo uno, primer hijo) |
| `<thead>` / `<tbody>` / `<tfoot>` | Grupos de filas: cabecera, cuerpo, pie (en este orden de aparición lógica; `tfoot` puede ir al final) |
| `<tr>` | Fila |
| `<th>` | Celda de **encabezado** — `scope="col"` (columna) o `scope="row"` (fila) declara qué encabeza; `headers="id"` para tablas complejas |
| `<td>` | Celda de datos |
| `<colgroup>` / `<col>` | Columnas: define columnas **antes** de las filas para estilarlas juntas (`<col span="2">`) |

```html
<table>
  <caption>Compresión de formatos de imagen</caption>
  <colgroup>
    <col>          <!-- columna 1 -->
    <col span="2"> <!-- columnas 2 y 3 juntas -->
  </colgroup>
  <thead>
    <tr><th scope="col">Formato</th><th scope="col">Compresión</th><th scope="col">Transparencia</th></tr>
  </thead>
  <tbody>
    <tr><th scope="row">AVIF</th><td>Excelente</td><td>Sí</td></tr>
    <tr><th scope="row">WebP</th><td>Muy buena</td><td>Sí</td></tr>
  </tbody>
  <tfoot>
    <tr><td colspan="3">Datos medidos con 1280px de ancho</td></tr>
  </tfoot>
</table>
```

Puntos técnicos que importan:

- **`scope` en los `th`** es lo que permite a un lector de pantalla anunciar "Formato, fila AVIF, Compresión: Excelente" al recorrer celdas — sin él, la tabla es un galimatías. `colspan`/`rowspan` combinan celdas.
- **`<caption>`** es el título accesible de la tabla (alternativa: `aria-labelledby`).
- Tablas **anidadas** son válidas pero complican la lectura accesible; prefieras una tabla con secciones.
- No se usan tablas para **maquetar** páginas — para eso están Flexbox y [[CSS Grid]].

## Formularios

Las etiquetas de formulario (`<form>`, `<input>`, `<label>`, `<select>`, `<textarea>`, `<fieldset>`, `<legend>`, `<button>`, `<datalist>`, `<output>`, `<progress>`, `<meter>`) tienen categoría propia: ver [[Formularios]].

## Interactivos nativos

HTML trae widgets interactivos sin JS:

- **`<details>` / `<summary>`**: acordeón nativo. `<summary>` es el disparador visible (el primer hijo de `<details>`); `open` lo muestra expandido.

```html
<details>
  <summary>¿Cómo se calcula el envío?</summary>
  <p>Según peso y destino…</p>
</details>
```

  Accesible gratis: Enter/Espacio lo alternan; sin él, un acordeón con divs necesita JS + ARIA live.

- **`<dialog>`**: diálogo modal/no-modal nativo. `showModal()` lo abre como modal (con backdrop, foco atrapado, `::backdrop` estilizable, cierra con Esc); `show()` lo abre no-modal.

```html
<dialog id="confirmar">
  <h2>¿Eliminar café?</h2>
  <form method="dialog">
    <button value="si">Sí</button>
    <button value="no" autofocus>No</button>
  </form>
</dialog>
<script>
  const dlg = document.getElementById('confirmar');
  dlg.showModal();
  dlg.addEventListener('close', () => console.log(dlg.returnValue));
</script>
```

  `method="dialog"` hace que el botón cierre el diálogo y exponga su `value` en `close` — el patrón más limpio para confirmaciones (ver [[Eventos]]).

## Metadata

- **`<title>`**: título del documento (pestaña, SERP, nombre accesible de la ventana).
- **`<meta>`**: metadatos — charset, viewport, description… referencia exhaustiva en [[Etiquetas Meta]].
- **`<link>`**: relaciones externas — CSS (`rel="stylesheet"`), favicon (`rel="icon"`), canónica, preload/prefetch (ver [[Preload]] y [[Prefetch]]).
- **`<style>`**, **`<script>`**: CSS y JS embebidos; conviene sacarlos a archivos externos (caché) — ver [[Crear archivo para CSS]].

## Obsoletas pero vistas "en la wild"

HTML4 trajo etiquetas presentacionales que **siguen apareciendo en código viejo**. Todas son válidas sintácticamente pero su contenido ahora lo resuelve CSS:

| Obsoleta/deprecada | Alternativa moderna |
|---|---|
| `<font color="red">` | CSS `color` |
| `<center>` | `text-align: center` |
| `<big>` | `font-size` |
| `<strike>`, `<tt>`, `<acronym>` | `<s>`, `<code>`, `<abbr>` |
| `<frame>`, `<frameset>`, `<noframes>` | `<iframe>` o layouts CSS |
| `<applet>` | `<embed>`/`<canvas>` (Flash murió) |
| `<marquee>`, `<blink>` | CSS Animaciones ([[Animaciones]]) |
| `<bgsound>`, `<basefont>`, `<dir>` | `<audio>`, CSS, `<ul>` |

No las uses en código nuevo; si las ves, no hace falta migrarlas de urgencia (el navegador las trata como elementos inline genéricos), pero son deuda técnica.

## Comentarios

En HTML también podemos usar comentarios; hay que comentar lo justo y lo necesario:

```html
<!-- Texto del comentario -->
```

No se muestran ni afectan el renderizado, pero son visibles en el código fuente: no guardes datos sensibles en ellos. No se anidan y un `-->` extra cierra el comentario. JS tiene su propia sintaxis (`//`, `/* */`) — mezclarlas produce errores clásicos.

## Cheatsheet por categoría

| Categoría | Elementos |
|---|---|
| Texto (bloque) | `p`, `h1–h6`, `br`, `hr`, `blockquote`, `pre`, `address`, `figure`, `figcaption` |
| Texto (inline) | `span`, `strong`, `em`, `b`, `i`, `mark`, `small`, `s`, `del`, `ins`, `u`, `sub`, `sup`, `abbr`, `time`, `data`, `q`, `cite`, `code`, `kbd`, `samp`, `var` |
| Enlaces | `a` |
| Imágenes / media | `img`, `picture`, `source`, `video`, `audio`, `track`, `canvas`, `svg`, `iframe`, `embed`, `object` |
| Estructura | `header`, `nav`, `main`, `section`, `article`, `aside`, `footer`, `div`, `hr` |
| Listas | `ul`, `ol`, `li`, `dl`, `dt`, `dd` |
| Tablas | `table`, `caption`, `thead`, `tbody`, `tfoot`, `tr`, `th`, `td`, `colgroup`, `col` |
| Formularios | `form`, `input`, `label`, `select`, `option`, `optgroup`, `textarea`, `button`, `fieldset`, `legend`, `datalist`, `output`, `progress`, `meter` |
| Interactivos | `details`, `summary`, `dialog` |
| Metadata | `title`, `meta`, `link`, `style`, `script`, `base` |
| Edición/deprecadas | `font`, `center`, `big`, `strike`, `marquee`, `frameset`… (evitar) |

## Errores comunes

1. **`</img>` y cierres en void elements**: no existen; el HTML5 los ignora o marca error.
2. **Divs dentro de `<p>`**: cierra el párrafo y parte el layout en dos.
3. **Tablas para maquetar**: destruye semántica y accesibilidad; usá [[CSS Grid]]/Flexbox (ver [[Tipos de Display]]).
4. **`th` sin `scope`**: la tabla es accesible solo a medias.
5. **`target="_blank"` sin `rel="noopener"`**: la página destino puede redirigir a la original (tabnabbing).
6. **Listas con `<div>` de viñetas en vez de `<li>`**: pierdes la semántica de lista y la numeración nativa.
7. **`autoplay` con sonido**: bloqueado por los navegadores; además es antipattern de accesibilidad (WCAG 1.4.2 — audio que suena solo necesita control de pausa).
8. **Mezclar sintaxis dentro de raw text elements**: comentarios HTML dentro de `<script>` no son comentarios JS.

## Accesibilidad

- Cada `alt` describe la función/contenido de la imagen; imágenes-enlace llevan alt con la acción ("Ir a inicio", no "logo.png").
- `<iframe>` y `<canvas>` necesitan nombre accesible (`title` / contenido fallback).
- Tablas con `caption` + `scope`; tablas de **layout** no existen.
- `kbd`, `samp` y `var` dan contexto semántico en documentación técnica.
- `<dialog>` con `showModal()` aporta foco atrapado y cierre con Esc gratis — reimplementarlo con divs es más código y más errores.

## Relacionadas

- [[Html]] — qué es HTML y anatomía de las etiquetas
- [[Estructurar Html]] — estructura semántica de un documento
- [[Etiquetas Meta]] — etiquetas de configuración del `<head>`
- [[Formularios]] — etiquetas de formulario
- [[Formatos Avif y Webp]] — `<picture>` para imágenes modernas
- [[Lazy Loading]] — `loading="lazy"` y control de carga de imágenes
- [[CSS Introducción]] — dar estilo a las etiquetas con selectores
- [[Tipos de Display]] — cómo cada etiqueta se comporta en el modelo de cajas
- [[Document Object Model (DOM)]] — cómo cada etiqueta se convierte en un nodo del árbol
