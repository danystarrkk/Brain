---
tags:
  - desarrollo
  - frontend
  - html
tipo: concepto
estado: terminado
---

# Qué es HTML?

**HTML** son las iniciales de *"Hypertext Markup Language"* (Lenguaje de Marcas de Hipertexto). Es el lenguaje con el que se construye el contenido de toda página web: define la **estructura y el significado** de los elementos (títulos, párrafos, imágenes, enlaces), pero **no su apariencia ni su comportamiento**.

Es un lenguaje fácil de aprender si se pone en práctica: es muy intuitivo, y a través de **etiquetas** escribimos el marcado del contenido:

```html
<p>        → Párrafos
<nav>      → Navegaciones
<header>   → Encabezados de sitio o de elemento
<main>     → Contenido principal
<footer>   → Pie de página o de elemento
```

> [!info] HTML5
> HTML5 fue un gran salto (semántica, multimedia, formularios enriquecidos, APIs del navegador), pero hoy en día basta con llamarlo **HTML**: todas las versiones modernas son en la práctica HTML5. El estándar lo mantiene el [WHATWG](https://whatwg.org/) como *HTML Living Standard* — ya no hay "HTML6": HTML es un estándar vivo que se actualiza continuamente.

## Lenguaje de marcado, no de programación

La diferencia clave:

| | Lenguaje de marcado (HTML) | Lenguaje de programación (JS, Python…) |
|---|---|---|
| Propósito | **Describir** contenido y su significado | **Dar comportamiento**: lógica, cálculos, control |
| ¿Tiene lógica? | No: no hay variables, bucles ni condicionales | Sí |
| ¿Se ejecuta? | Se **interpreta y renderiza** el navegador | Se **ejecuta** paso a paso |
| Resultado | Un **árbol de elementos** (el DOM) | Funciones que manipulan ese árbol |

HTML describe el *qué*; el *cómo se ve* lo resuelve CSS ([[CSS Introducción]]) y el *cómo se comporta* JavaScript ([[Java Script]]). Esta separación (estructura / estilo / comportamiento) es el principio organizador de toda la web moderna.

### El "hipertexto" del nombre

El hipertexto es el contenido conectado por **enlaces**: `<a href="...">` apunta a otro documento y `<a href="#id">` salta a un lugar del mismo documento. Sin enlaces, una página HTML sería un documento aislado; con ellos, la web es una red de documentos — el "WWW" de Tim Berners-Lee (1989), quien inventó HTML precisamente para que los científicos pudieran enlazar sus papers.

## Anatomía: elementos, etiquetas y atributos

El HTML se compone de **elementos**, delimitados por etiquetas (tags):

```html
<p>Hola, mundo</p>
```

- **Etiqueta de apertura**: `<p>` · **Etiqueta de cierre**: `</p>` · **Contenido**: `Hola, mundo` · **Elemento**: todo el conjunto.
- Muchas etiquetas aceptan **atributos** para configurarlas: `<img src="foto.jpg" alt="Foto">`.
- Algunas etiquetas son **vacías** (no tienen cierre porque no envuelven nada): `<img>`, `<br>`, `<meta>`, `<hr>`, `<input>`.

### Reglas de sintaxis de los atributos

```html
<input type="text" required maxlength="40" data-campo="nombre">
```

| Regla | Detalle |
|---|---|
| Nombre minúsculas | `maxlength`, no `maxLength` (el HTML es case-insensitive para nombres, pero la convención y el XHTML legacy exigen minúsculas) |
| Valores entre comillas | Se aceptan comillas dobles (estándar), simples o incluso sin comillas si no hay espacios; **usar siempre comillas dobles** |
| Booleanos | `required`, `disabled`, `checked`: se escriben **solo con el nombre**; su mera presencia los activa (`required="false"` también los activa — bug histórico conocido) |
| Valores `true`/`false` | Casi ningún atributo acepta `="false"` para desactivar: se **omite el atributo** |
| Atributos `data-*` | Espacio de nombres libre para guardar datos en el DOM, legibles con JS: `el.dataset.campo` (ver [[Acceder a Html]]) |
| Atributos globales | `id`, `class`, `style`, `title`, `hidden`, `tabindex`, `role`, `aria-*` funcionan en **todos** los elementos |

> [!warning] `class` no es `className` en el HTML
> En el marcado se escribe `class="card"` (una sola clase puede tener varias palabras: `class="card card--big"`). `className` es el nombre del atributo en el DOM de JavaScript, donde `class` es palabra reservada.

## Estructura básica de un documento

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mi página Web</title>
    <link rel="stylesheet" href="css/style.css">
  </head>
  <body>
    <p>Contenido de la página</p>
  </body>
</html>
```

- **`<!DOCTYPE html>`**: le dice al navegador que es HTML5 moderno — ver "Quirks mode" más abajo; es la línea más importante del documento.
- **`html`**: raíz del documento; su atributo `lang="es"` declara el idioma (crítico para lectores de pantalla y traducción).
- **`head`**: información *meta* del sitio: idioma, título, descripción, codificación, enlaces a hojas de estilo CSS, scripts. Nada de esto se ve en la página — ver [[Etiquetas Meta]].
- **`body`**: todo lo que **se ve** en la página: textos, imágenes, videos, botones, animaciones.

La estructura semántica completa por regiones (header/nav/main/aside/footer) se trata en [[Estructurar Html]].

> [!tip] El navegador siempre intenta arreglar tu HTML
> Los navegadores son tolerantes con errores (etiquetas sin cerrar, anidamiento inválido), pero "funcione" no significa "bien hecho": el navegador corrige por su cuenta y ese arreglo puede no ser el que esperabas. Validá contra el [validator del W3C](https://validator.w3.org/) cuando dudes.

## El modelo de análisis: cómo el navegador convierte texto en página

El navegador no "pinta" el archivo: ejecuta un **parser** (analizador) que recorre el texto byte a byte y construye el **DOM** — un árbol de objetos en memoria sobre el que actúan CSS y JavaScript. El proceso completo:

### Pipeline de renderizado (simplificado)

```
HTML en texto
   │  parser HTML tokeniza y construye…
   ▼
DOM  (árbol de elementos)  ←── también: árbol de accesibilidad (a11y tree)
   │  + CSS parseado → CSSOM
   ▼
Árbol de render (layout tree: cajas visibles)
   │  layout: se calcula geometría de cada caja
   ▼
Paint (pintura) → Composite (composición de capas)
```

CSS y JS bloquean o modifican este pipeline: un `<script>` sin `defer` **detiene el parseo del DOM** (por eso conviene cargar los scripts al final o con `defer` — ver [[Preload]] y [[Prefetch]] para las estrategias de carga).

### Reglas de construcción del árbol que explican bugs raros

El parser HTML no es un parser de gramática libre: es un **autómata con reglas de recuperación de errores**. Algunas consecuencias prácticas:

1. **Los `<p>` no pueden anidarse**: `<p>uno<p>dos</p>` cierra el primero automáticamente y produce dos párrafos hermanos.
2. **Tablas "tragadas"**: si dentro de `<table>` aparece contenido que no es celda (`<table><div>hola</div>`), el navegador **eleva el div fuera de la tabla** antes de renderizarlo.
3. **`<li>` fuera de lista**: se corrige insertándolo dentro de una `<ul>` implícita.
4. **Etiquetas desconocidas** (como `<foo>`) se crean como elementos HTML genéricos (inline por defecto) — se pueden estilar con CSS y leer con JS.
5. **`<html>`, `<head>` y `<body>` implícitos**: si los omitís, el parser los inventa.

> [!tip] Mental modelo: el DOM es el contrato
> Todo lo que ve (CSS), todo lo que programa (JS), todo lo que escucha un lector de pantalla y todo lo que indexa un buscador pasa por el DOM. Escribir HTML válido es escribir un buen contrato para cuatro consumidores distintos a la vez.

### Quirks mode, almost-standards y standards mode

La presencia (o ausencia) de `<!DOCTYPE html>` decide en qué **modo** parsea el navegador:

| Modo | Trigger | Comportamiento |
|---|---|---|
| **Standards mode** | `<!DOCTYPE html>` (cualquier doctype moderno) | Cajas según especificación, `box-sizing: content-box`, `%` de altura según contenido |
| **Almost-standards mode** | Doctypes antiguos con DTD (XHTML 1.0 estricto/transitional) | Casi estándar; solo difiere en el manejo de imágenes en celdas de tabla |
| **Quirks mode** | **Sin doctype** o doctypes basura | Reproduce bugs de Netscape/IE de los 90: `box-sizing: border-box` retro, alturas `%` rotas, márgenes de fuente diferentes |

Síntoma típico de quirks mode: le pones `height: 100%` al `body` y no hace nada, o el layout "se ve distinto" sin razón CSS. Diagnóstico rápido: `document.compatMode` en la consola devuelve `"CSS1Compat"` (standards) o `"BackCompat"` (quirks). **Siempre** empezá el documento con `<!DOCTYPE html>`.

## Codificación de caracteres

El HTML es texto, y el texto necesita una codificación para convertirse en bytes:

- **UTF-8** es el estándar desde hace más de una década: cubre todos los idiomas, los emojis y es compatible con ASCII.
- Se declara con `<meta charset="UTF-8">` como **primera etiqueta del `<head>`** — debe estar dentro de los primeros 1024 bytes del archivo, si no el navegador puede parsear parte del documento con otra codificación y los acentos se vuelven basura (`Ã©` en vez de `é`) — ver [[Etiquetas Meta]].
- Servir el archivo con el header `Content-Type: text/html; charset=UTF-8` es la capa correcta cuando controlás el servidor; la meta es el respaldo.

> [!warning] Caracteres reservados en texto
> `<`, `>` y `&` son del lenguaje. Para mostrarlos como texto se usan **entidades**: `&lt;`, `&gt;`, `&amp;`. Caso especial clásico: `<script>alert("1 < 2")</script>` — dentro de `<script>` no se pueden usar entidades, hay que escapar con `\` en JS o construir el string dinámicamente.

## Semántica: HTML moderno = HTML semántico

Las etiquetas semánticas cuentan al navegador (y a buscadores y lectores de pantalla) **qué significa** el contenido:

- `<h1>`–`<h6>`: niveles de encabezado (jerarquía del documento).
- `<header>`, `<nav>`, `<main>`, `<footer>`: regiones del sitio.
- `<section>`, `<article>`, `<aside>`: bloques de contenido con significado propio.
- `<form>`, `<label>`, `<table>`, `<figure>`: estructuras específicas — ver [[Formularios]] y [[Etiquetas]].

El opuesto son las etiquetas **presentacionales** (`<font>`, `<center>`, `<b>`, `<i>`), heredadas de HTML antiguo: están deprecadas y el estilo hoy se resuelve con CSS. Cuando ninguna etiqueta semántica aplica, se usan `<div>` (bloque) y `<span>` (en línea).

> [!info] El valor real de la semántica
> Una `<div>` y un `<button>` renderizan igual con el CSS correcto, pero para el navegador la diferencia es enorme: el `<button>` viene con rol de botón, activación con Enter/Espacio, foco en el tabulado y nombre accesible. La semántica es **comportamiento gratis** que una `<div>` tendrías que reimplementar con JS y ARIA (y peor).

## El ecosistema moderno alrededor de HTML

HTML no vive solo en archivos `.html`:

- **Frameworks de componentes** (React, Vue, Svelte) generan HTML: en React, el JSX es azúcar sintáctica sobre `createElement` — ver [[ReactJS - Fundamentos]]. La semántica sigue importando igual porque el DOM resultante es el que consume el navegador.
- **Tailwind y utilitarios** aplican estilos vía `class` sin tocar la estructura — ver [[TailwindCSS - Instalación y Configuración]].
- **Progressive Web Apps**: el mismo HTML + APIs del navegador (service workers, notificaciones) dan apps instalables — ver [[Qué es una API]].
- **Server-side rendering / Static Site Generators**: Astro, Next, Hugo etc. producen HTML final; el navegador recibe el mismo documento que escribirías a mano.
- **Contenido dinámico**: el DOM se modifica en vivo con JS (ver [[Document Object Model (DOM)]] y [[Acceder a Html]]).

> [!tip] HTML es la base común de todo
> Frameworks van y vienen; el navegador siempre termina recibiendo HTML + CSS + JS. Dominar el HTML puro es lo que te permite depurar cualquier framework, porque el DOM final que recibe el navegador es idéntico al que escribirías a mano.

## Breve historia: por qué "HTML5" fue un punto de inflexión

| Etapa | Año | Qué aportó | Estado hoy |
|---|---|---|---|
| HTML 1.0/2.0 | 1993–1995 | Encabezados, párrafos, listas, enlaces, imágenes | Base del lenguaje, vigente |
| HTML 3.2 / 4.01 | 1997–1999 | Tablas, formularios, presentación (`font`, `center`), marcos | Presentación movida a CSS |
| XHTML 1.0/1.1 | 2000 | HTML "estricto" con sintaxis XML | Abandonado; XHTML 2.0 nunca se implementó |
| HTML5 (WHATWG) | 2008–2014+ | Semántica, multimedia, canvas, formularios ricos, APIs del navegador | **Estándar vivo actual** |

La lección de XHTML: intentar que la web fuera "perfecta y estricta" fracasó; HTML5 triunfó precisamente **tolerando errores** — el parser recupera, no rechaza. Es un diseño deliberado, no un descuido.

## Herramientas de verificación

| Herramienta | Para qué |
|---|---|
| [Validator del W3C](https://validator.w3.org/) | Errores de sintaxis y especificación |
| DevTools → pestaña **Elements** | El DOM final real (tras las correcciones del parser) — compararlo con el archivo revela qué "arregló" el navegador |
| DevTools → **Accessibility** | El árbol de accesibilidad: roles y nombres accesibles |
| `document.compatMode` en consola | Confirmar standards vs quirks mode |
| Lighthouse / axe | Auditorías de semántica y accesibilidad automáticas |

> [!tip] `view-source:` vs Elements
> `view-source:` muestra el archivo tal cual llegó; la pestaña Elements muestra el DOM ya corregido. Cuando algo "no está en mi HTML pero se ve en la página", la diferencia entre ambas vistas es la respuesta: el parser lo movió o lo creó.

## Malas prácticas

HTML tolera malas prácticas y "funciona", pero eso no significa usarlas:

- Mezclar presentación con estructura (estilos en línea en vez de CSS).
- Omitir el `alt` de las imágenes o poner texto vacío sin motivo.
- Saltarse la jerarquía de encabezados (`<h1>` directo a `<h4>`).
- Usar `<div>`/`<span>` para todo en vez de etiquetas semánticas.
- Atributos obsoletos heredados de HTML4: `align`, `bgcolor`, `border`, `width` en tablas, `name` en iframes — todos reemplazados por CSS.

Los buscadores penalizan los sitios con malas prácticas (contenido sin estructura, texto en imágenes, metadatos ausentes), y las herramientas de accesibilidad directamente no pueden interpretar el contenido.

## Errores comunes

1. **Olvidar el `<!DOCTYPE html>`** → quirks mode y layouts impredecibles. Síntoma: `height: 100%` no funciona.
2. **Charset tarde o ausente** → `ñ`, acentos y emojis como caracteres raros. Va al inicio del `<head>`.
3. **Etiquetas de cierre escritas al revés** (`<b>negrita</b` o `<option>…<option>` sin barra) → el parser recupera como puede y "traga" contenido posterior.
4. **`id` duplicados** → el `for` de labels y los anclajes `#id` apuntan al primero que encuentre el navegador.
5. **HTML en minúsculas inconsistentes** — funcional, pero ilegible y fuente de bugs al copiar de fuentes que respetan mayúsculas.
6. **Contenido importante dentro de `<script>`** → los buscadores no lo indexan y hay que esperar el parseo del JS.
7. **Atributos booleanos con valor** (`required="false"`) → siguen activos; se omiten para desactivar.

## Rendimiento

- **El tamaño del HTML importa menos que su estructura**: un HTML de 100 KB bien construido (pocos nodos redundantes, imágenes con `width`/`height`) renderiza antes que uno de 50 KB con 12 `<div>` por card.
- **HTML crítico bloquea render**: el parser se detiene en `<script>` sin `defer`; el CSS bloquea el render — ver [[Preload]] para hints de carga y [[Lazy Loading]] para imágenes diferidas.
- **Anidar profundamente cuesta**: cada nivel profundo de `<div>`s añade nodos al árbol de render; la semántica profunda (nav > ul > li > a) es necesaria, la profundidad decorativa no.

## Accesibilidad

- `lang="es"` en `<html>`: sin él, los lectores de pantalla anuncian el texto en el idioma equivocado (o lo intentan adivinar).
- Cada `<img>` necesita `alt` (o `alt=""` explícito si es decorativa — omitirlo hace que el lector lea la URL).
- El orden del DOM es el orden de lectura y de tabulado: no uses CSS para reordenar contenido.
- Semántica = navegación por regiones: `<main>`, `<nav>` y `<header>` son saltos directos en los lectores de pantalla.

## Cheatsheet del documento mínimo

| Pieza | Rol | ¿Obligatoria? |
|---|---|---|
| `<!DOCTYPE html>` | Activa standards mode | **Sí** |
| `<html lang="es">` | Raíz + idioma | Sí (y `lang` es requisito WCAG 3.1.1) |
| `<meta charset="UTF-8">` | Codificación | Sí (dentro de los primeros 1024 bytes) |
| `<meta name="viewport" …>` | Escala en móviles | Sí para responsive |
| `<title>` | Título de pestaña / SERP | Sí (WCAG 2.4.2) |
| `<head>` / `<body>` | Meta / contenido | Técnicamente omitibles (el parser los crea) |

## Relacionadas

- [[Estructurar Html]] — la estructura completa y válida de un documento
- [[Etiquetas]] — catálogo de etiquetas por categoría
- [[Etiquetas Meta]] — configuración del `<head>`
- [[CSS Introducción]] — dar estilo a la estructura HTML
- [[Java Script]] — comportamiento dinámico de la página
- [[Document Object Model (DOM)]] — cómo el navegador convierte el HTML en un árbol de objetos
- [[Acceder a Html]] — manipular el DOM desde JavaScript
- [[Formularios]] — la base de la interacción con el usuario
