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

> [!info] Etiquetas vacías
> Algunas etiquetas no tienen contenido ni cierre porque no envuelven nada: `<img>`, `<br>`, `<hr>`, `<meta>`, `<input>`. Se llaman **void elements** y escribir `</img>` sería un error.

## Texto

Tenemos diferentes etiquetas que cumplen propósitos específicos al momento de ingresar texto:

- **`<p>`**: muy común, se usa para todos los párrafos.
- **`<h1>`, `<h2>`, …, `<h6>`**: los llamados *headings* (encabezados), del 1 al 6; mientras más alto el número, menor es su importancia en el contenido. Los headings siguen reglas:
  - Solo podemos tener un `<h1>` por página.
  - Podemos usar más de un `<h2>` por página, y con ayuda de estos separaremos por secciones.
- **`<span>`**: etiqueta **en línea** (inline) que se usa para aplicar estilos a un fragmento dentro de otros elementos, sin romper el flujo del texto.
- **`<strong>`** y **`<em>`**: énfasis semántico (negrita fuerte y cursiva fuerte); el navegador los renderiza en negrita/cursiva, pero además indican *importancia* a buscadores y lectores de pantalla. `<b>` e `<i>` son puramente visuales.

```html
<p>Este es un <strong>párrafo</strong> con <em>énfasis</em>.</p>
<p><span class="resaltado">Este fragmento</span> tiene estilo propio.</p>
```

## Imágenes

Conforme se crea un proyecto web vamos a ver que las imágenes son importantes para el diseño. Algunas de las etiquetas que usamos para añadir imágenes a la web son:

- **`<img>`**: es la etiqueta más usada para agregar imágenes. Para eso hacemos uso de los **atributos** de HTML — un ejemplo:

```html
<img src="imagen.jpg" alt="Texto alternativo" width="200" height="300">
```

  - `src`: ruta de la imagen.
  - `alt`: texto alternativo, obligatorio para accesibilidad y SEO: describe la imagen si esta no carga o si se usa un lector de pantalla.
  - `width`/`height`: dimensiones declaradas — evitan saltos de layout mientras carga (ver [[Lazy Loading]]).

- **`<figure>` / `<figcaption>`**: agrupan una imagen (o contenido similar) con su pie de foto, como bloque semántico:

```html
<figure>
  <img src="diagrama.png" alt="Diagrama de arquitectura">
  <figcaption>Arquitectura del sistema</figcaption>
</figure>
```

Para servir imágenes en formatos modernos con respaldo (AVIF/WebP/JPG) se usa `<picture>`, que veremos en [[Formatos Avif y Webp]].

## Enlaces

Algunos sitios (como los de marketing) solo requieren una página para funcionar, pero cuando los proyectos aumentan de tamaño vamos a necesitar más de una página, y para eso agregamos enlaces:

- **`<a>`** (*anchor*): con esta etiqueta agregamos nuestros enlaces. Al igual que en las imágenes, hacemos uso de los atributos — un ejemplo:

```html
<a href="https://ejemplo.com">Contenido como texto</a>
<a href="pagina.html"><img src="miniatura.jpg" alt="Miniatura"></a>
```

  - `href`: URL o archivo de destino.
  - `target="_blank"`: abre en una pestaña nueva — si lo usás, agregá también `rel="noopener"` por seguridad.
  - `href="#seccion"`: enlace interno a un `id` de la misma página.

## Estructura

Para estructurar nuestra web vamos a tener las siguientes etiquetas y casos de uso (detalle completo en [[Estructurar Html]]):

| Etiqueta | Cuándo usarla |
|---|---|
| `<header>` | Parte superior de un sitio o de un elemento |
| `<footer>` | Parte inferior de un sitio o de un elemento |
| `<nav>` | Grupo de enlaces o navegación |
| `<main>` | Contenido principal del archivo HTML; solo uno por archivo |
| `<section>` | Cuando el primer hijo es un heading e introduce una nueva sección |
| `<aside>` | El contenido que acompaña al principal |
| `<div>` | Cuando ninguno de los anteriores aplica (bloque genérico) |

## Listas

Tenemos dos formas de crear listas en HTML: ordenadas y no ordenadas:

- **`<ul>`**: crea un listado **no ordenado** (con viñetas).
- **`<ol>`**: crea un listado **ordenado** (numerado). Acepta `start` y `type` para controlar la numeración.
- **`<li>`**: crea cada elemento de una lista.

Ejemplo:

```html
<ul>
  <li>Esta es</li>
  <li>Una</li>
  <li>Lista</li>
  <li>No ordenada</li>
</ul>

<ol>
  <li>Esta es</li>
  <li>Una</li>
  <li>Lista</li>
  <li>Ordenada</li>
</ol>
```

> [!tip] Listas de navegación
> Los menús de sitios son listas de enlaces: `<nav><ul><li><a href="...">…` — es el patrón semántico estándar y el que esperan buscadores y lectores de pantalla.

## Tablas

Para datos tabulares existe `<table>`, con `<thead>`, `<tbody>`, `<tr>` (fila), `<th>` (celda de encabezado) y `<td>` (celda de datos):

```html
<table>
  <thead>
    <tr><th>Formato</th><th>Compresión</th></tr>
  </thead>
  <tbody>
    <tr><td>AVIF</td><td>Excelente</td></tr>
    <tr><td>WebP</td><td>Muy buena</td></tr>
  </tbody>
</table>
```

No se usan tablas para maquetar páginas — para eso está CSS (ver [[CSS Introducción]]).

## Formularios

Las etiquetas de formulario (`<form>`, `<input>`, `<label>`, `<select>`, `<textarea>`) tienen categoría propia: ver [[Formularios]].

## Comentarios

En HTML también podemos usar comentarios; hay que comentar lo justo y lo necesario:

```html
<!-- Texto del comentario -->
```

Los comentarios no se muestran en la página ni afectan el renderizado, pero son visibles en el código fuente: no guardes datos sensibles en ellos.

## Relacionadas

- [[Html]] — qué es HTML y anatomía de las etiquetas
- [[Estructurar Html]] — estructura semántica de un documento
- [[Etiquetas Meta]] — etiquetas de configuración del `<head>`
- [[Formularios]] — etiquetas de formulario
- [[Formatos Avif y Webp]] — `<picture>` para imágenes modernas
- [[CSS Introducción]] — dar estilo a las etiquetas con selectores
- [[Document Object Model (DOM)]] — cómo cada etiqueta se convierte en un nodo del árbol
