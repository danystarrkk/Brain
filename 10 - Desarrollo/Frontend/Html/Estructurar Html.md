---
aliases: ["Estructura HTML", "Estructurar HTML"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Etiquetas]]"
  - "[[Etiquetas Meta]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/Guide/HTML/Using_HTML_sections_and_outlines"
---

# Estructurar HTML

Nuestros sitios web deberán tener una **estructura válida**, que siga ciertos lineamientos que HTML nos ha dado para crear código que siga estándares. Además de hacer el código más fácil de leer e identificar (saber dónde realizar cambios en el futuro), una buena estructura mejora el posicionamiento en buscadores y la accesibilidad.

Una forma gráfica de verlo es:

![[Pasted image 20250124203631.png]]

## El esqueleto de todo documento

Toda página HTML arranca con la misma estructura base — ver [[Html]] para el detalle de cada parte:

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="Descripción del sitio para buscadores">
    <title>Mi página Web</title>
    <link rel="stylesheet" href="css/style.css">
  </head>
  <body>
    <header>
      <nav><!-- menú de navegación --></nav>
    </header>

    <main>
      <h1>Título principal de la página</h1>
      <section><!-- sección de contenido --></section>
    </main>

    <footer><!-- pie de página --></footer>
  </body>
</html>
```

> [!info] `head` vs `body`
> En el `head` vive lo que el navegador *interpreta* pero no muestra (metadatos — ver [[Etiquetas Meta]]). En el `body` vive todo lo visible.

## Elementos de seccionado: las regiones del documento

HTML5 introduce los **elementos de seccionado**: divs con significado que partitionan la página en regiones comprensibles para humanos, máquinas y lectores de pantalla. Este es el catálogo completo con criterios de uso:

| Etiqueta | Cuándo usarla | Equivale a `role` ARIA | ¿Requiere heading? |
|---|---|---|---|
| `<header>` | Encabezado del sitio **o de un elemento** (artículo, sección): logo, título, metadatos | `banner` (si es hijo de `<body>`) / `generic` | No |
| `<nav>` | Grupo **principal** de enlaces de navegación | `navigation` | No |
| `<main>` | Contenido único de la página; **exactamente uno** por documento y no visible en otros lugares | `main` | No, pero su primer hijo suele ser el `<h1>` |
| `<article>` | Contenido independiente y autocontenido: un post, un comentario, una tarjeta que tendría sentido fuera de la página | `article` | Recomendado (`<h1>`/`<h2>` interno) |
| `<section>` | Bloque de contenido que introduce un **nuevo tema**, típicamente encabezado por un heading | `region` (solo si tiene heading) | **Sí** — sin heading no es `<section>`, es `<div>` |
| `<aside>` | Contenido complementario: barra lateral, nota al margen, anuncios relacionados | `complementary` | No |
| `<footer>` | Pie del sitio o de una sección/artículo: créditos, autoría, enlaces legales, fecha | `contentinfo` (si es hijo de `<body>`) | No |

### La pregunta discriminatoria: `section` vs `div` vs `article`

> [!tip] Test de las tres preguntas
> 1. ¿Tiene **heading** propio y un tema nuevo? → `<section>`.
> 2. ¿Tiene sentido **por sí solo** (se podría syndicar/comentar)? → `<article>`.
> 3. ¿No cumple ninguna de las anteriores y solo agrupa para estilos/layout? → `<div>`. Usar `<div>` no es pecado cuando realmente no hay significado.

Errores típicos: poner `<section>` en todo (viola el requisito de heading) y llenar la página de `<div>` (pierde toda la semántica).

### Anidamiento correcto y duplicidad de regiones

- Los elementos de seccionado **se anidan libremente**: `<article>` puede contener `<section>`, y un `<article>` puede tener su propio `<header>` y `<footer>` (de elemento, no de documento).
- **No se duplican regiones de documento**: un solo `<main>` visible por página; varios `<nav>` sí (menú principal, migas de pan, paginación) pero cada uno debe distinguirse (por ejemplo con `aria-label="Menú principal"`).
- `<header>`/`<footer>` como hijos directos de `<body>` son de **documento** (banner/contentinfo); anidados dentro de `<article>` o `<section>` son de ese elemento.

```html
<article>
  <header>
    <h2>Café etíope</h2>
    <p>Publicado el 12/03 por Daniel</p>
  </header>
  <p>El perfil de este café…</p>
  <footer>
    <p>Categoría: Orígenes</p>
  </footer>
</article>
```

## Jerarquía de encabezados y el "outline" del documento

Los `<h1>`–`<h6>` dibujan el esqueleto lógico del contenido:

- **Un solo `<h1>`** por página: el título principal (casi siempre el mismo que el `<title>`).
- **Sin saltos** entre niveles: `<h1>` → `<h3>` sin pasar por `<h2>` rompe la navegación por encabezados de los lectores de pantalla (donde cada página es una "lista de temas" navegable).
- Los niveles indican **jerarquía, no tamaño**: el tamaño lo pone CSS — ver [[CSS Introducción]].

> [!warning] El algoritmo de outline de HTML5 está muerto
> La especificación definió un algoritmo donde `<section>` "reseteaba" la jerarquía (permitiendo `<h1>` en cada sección), pero **ningún navegador lo implementó** ni los lectores de pantalla lo aplican. La regla práctica y actual es simple: jerarquía plana de `h1` → `h2` → `h3` por niveles reales, ignorando ese algoritmo. No diseñes estructuras que dependan de él.

### Miga de pan (breadcrumb) y navegación por landmarks

```html
<nav aria-label="Miga de pan">
  <ol>
    <li><a href="/">Inicio</a></li>
    <li><a href="/blog">Blog</a></li>
    <li aria-current="page">Café etíope</li>
  </ol>
</nav>
```

`aria-label` distingue varios `<nav>`; `aria-current="page"` marca la página actual (estilo + semántica).

## El árbol de accesibilidad: por qué la estructura no es cosmética

Cada navegador construye, a partir del DOM, un segundo árbol: el **árbol de accesibilidad** (a11y tree), que es lo que consumen los lectores de pantalla, el reconocimiento de voz y las herramientas de automatización (axe, Lighthouse). El árbol de accesibilidad expone por cada nodo: **rol** (role), **nombre accesible** (accessible name) y **propiedades/estado**.

| HTML escrito | Árbol de accesibilidad resultante | Con `<div>` genérico |
|---|---|---|
| `<nav>` | Rol `navigation`, navegable por landmark | Rol `generic`, invisible como región |
| `<main>` | Rol `main`, salto directo | Ninguno |
| `<h2>` | Encabezado nivel 2, listado en el "índice" del lector | Texto plano |
| `<button>` | Botón, activable con Enter/Espacio | Genérico (habría que reimplementar todo con JS + ARIA) |

> [!info] Un `<div>` no aparece como región, y una página de divs es "ciega"
> El lector de pantalla navega por landmarks (regiones), encabezados y listas. Una estructura plana de `<div>`s obliga a recorrer la página línea por línea — la diferencia de productividad para un usuario ciego es de minutos por página. La semántica **es** la accesibilidad estructural (WCAG 1.3.1).

Herramientas para verificarlo: pestaña "Accessibility" de DevTools (Chrome/Firefox), que muestra el árbol completo con roles y nombres accesibles.

## Plantillas de esqueleto completas

### 1. Landing / sitio de marketing

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Café Aurora — Café de especialidad</title>
  <meta name="description" content="Café de especialidad tostado en Argentina. Envíos a todo el país.">
  <link rel="canonical" href="https://cafeaurora.com/">
  <meta property="og:title" content="Café Aurora">
  <meta property="og:image" content="https://cafeaurora.com/img/portada.jpg">
  <link rel="stylesheet" href="css/style.css">
</head>
<body>
  <a class="skip-link" href="#contenido">Saltar al contenido</a>

  <header>
    <nav aria-label="Menú principal">
      <ul>
        <li><a href="/">Inicio</a></li>
        <li><a href="/cafes">Cafés</a></li>
        <li><a href="/contacto">Contacto</a></li>
      </ul>
    </nav>
  </header>

  <main id="contenido">
    <h1>Café Aurora</h1>
    <section aria-labelledby="hero-titulo">
      <h2 id="hero-titulo">Tostado de especialidad</h2>
      <p>Granos seleccionados de origen único…</p>
    </section>
    <section aria-labelledby="productos-titulo">
      <h2 id="productos-titulo">Nuestros cafés</h2>
      <article><h3>Etiopía Yirgacheffe</h3><p>…</p></article>
      <article><h3>Colombia Huila</h3><p>…</p></article>
    </section>
  </main>

  <aside><!-- contenido complementario, si lo hay --></aside>

  <footer>
    <p>© 2026 Café Aurora — <a href="/privacidad">Privacidad</a></p>
  </footer>
</body>
</html>
```

Detalles que valen oro:

- **Skip link** (`Saltar al contenido`): primer elemento enfocable, permite a quien navega con Tab saltarse la nav. Sin él, cada visita exige recorrer el menú entero.
- Cada `<section>` con `aria-labelledby` apuntando a su heading: la región queda nombrada en el árbol de accesibilidad.

### 2. Blog / artículo

```html
<body>
  <header><nav><!-- menú --></nav></header>
  <main>
    <article>
      <header>
        <h1>Cómo tostar en casa</h1>
        <p><time datetime="2026-03-12">12 de marzo, 2026</time> · Daniel</p>
      </header>
      <section>
        <h2>Elegir el grano</h2>
        <p>…</p>
      </section>
      <section>
        <h2>Temperaturas</h2>
        <p>…</p>
      </section>
      <aside>
        <h2>Equipamiento</h2>
        <p>Lista recomendada…</p>
      </aside>
    </article>
  </main>
  <footer><!-- pie --></footer>
</body>
```

`<time datetime="…">` da a máquinas la fecha ISO exacta aunque el texto visible sea humano; los buscadores y los lectores de pantalla lo agradecen.

### 3. Layout de aplicación (sidebar + panel)

```html
<body>
  <header>
    <nav aria-label="Principal"><!-- íconos/enlaces principales --></nav>
  </header>
  <main>
    <h1>Panel de pedidos</h1>
    <section><h2>Pendientes</h2><!-- tabla o lista --></section>
  </main>
  <aside aria-label="Filtros">
    <h2>Filtros</h2><form><!-- controles --></form>
  </aside>
</body>
```

Para maquetar las regiones (sidebar/columnas) se usan [[Tipos de Display]] — Flexbox o [[CSS Grid]] — sobre estas etiquetas semánticas, no al revés.

## Lineamientos de una estructura válida

- **Un solo `<h1>`** por página, y sin saltos jerárquicos (`<h1>` → `<h3>` sin pasar por `<h2>`).
- **Etiquetas siempre cerradas** (excepto las vacías: `<img>`, `<br>`, `<meta>`).
- **Anidamiento correcto**: no se pueden cruzar etiquetas (`<p><b></p></b>` es inválido).
- **Indentación consistente**: la estructura se lee a simple vista cuando la sangría refleja el anidamiento.
- Separar **estructura (HTML)**, **estilo (CSS)** y **comportamiento (JavaScript)** — cada uno en su archivo.
- **`id` únicos** en toda la página (anclajes, labels, `aria-labelledby` dependen de ello).
- Elementos **interactivos reales** en vez de divs: `<button>` para acciones, `<a>` para navegar — ver [[Etiquetas]].

> [!warning] Los navegadores corrigen por su cuenta
> Si la estructura es inválida, el navegador intenta repararla automáticamente, y el resultado raras veces es el que esperabas (etiquetas "tragadas", estilos aplicados donde no toca). Validá el HTML en el [validator del W3C](https://validator.w3.org/) antes de culpar a CSS o JS de un problema de maquetación.

## Estructura, SEO y metadatos

Los buscadores no "ven" la página: leen el árbol. La estructura es señal directa:

- **`<title>` + `<h1>` coherentes**: el título de la SERP sale del `<title>` (ver [[Etiquetas Meta]]); el `<h1>` debe describir lo mismo.
- **Jerarquía de headings = índice del contenido**: Google usa los `h2`/`h3` para entender subtemas y rich snippets.
- **`<main>` claro** le dice al crawler cuál es el contenido real y cuál es chrome de navegación.
- **Datos estructurados** (JSON-LD) se inyectan en el `<head>` pero describen entidades que el HTML semántico ya refleja — la coherencia entre ambos importa.
- **Enlaces con `<a href>` reales** (no click handlers en divs): los crawlers siguen hrefs, no eventos.

## Estructura y responsive

Una estructura por regiones permite maquetar por zonas: el `<header>` colapsa a hamburguesa, el `<main>` mantiene el flujo de texto, el `<aside>` pasa debajo en pantallas angostas. Ver [[Responsive Web Design]] para media queries y [[Tipos de Display]] para los modelos de layout.

## Errores comunes

1. **`<section>` sin heading**: no aporta nada y confunde; usá `<div>`.
2. **Saltos de nivel de encabezado** (`h1` → `h4`): rompe la navegación por headings de los lectores de pantalla.
3. **Varios `<main>`** (por ejemplo uno oculto con `display:none` para móvil): HTML exige uno solo y visible.
4. **`<h1>` por cada sección** (por el algoritmo de outline muerto): la jerarquía real queda plana y confusa.
5. **Skip link que no se puede enfocar**: el `href="#id"` debe apuntar a un elemento enfocable o con `tabindex="-1"`.
6. **Nav con solo `<div>`s y JS**: para buscadores no es navegable; para teclado no hay foco.
7. **Contenido complementario dentro de `<main>`**: si está dentro de main, no es complementario — movelo a `<aside>` hermano.

## Accesibilidad: checklist de estructura

- [ ] `lang="es"` en `<html>`.
- [ ] Un `<main>` por página; skip link como primer foco.
- [ ] `<h1>` único; jerarquía sin saltos.
- [ ] Cada `<nav>` diferenciado con `aria-label` si hay más de uno.
- [ ] Las `<section>` tienen heading (directo o vinculado con `aria-labelledby`).
- [ ] Orden de tabulado = orden visual (el DOM manda — ver [[Flexbox]] sobre `order`).
- [ ] Verificación con el árbol de accesibilidad de DevTools y un recorrido real con Tab.

## Rendimiento

- La semántica no cuesta bytes: `<main>` pesa lo mismo que `<div>` pero habilita saltos y regiones.
- Estructuras profundas con muchos wrappers redundantes añaden nodos al DOM y al árbol de render; cada `<div>` decorativo es un nodo que CSS y JS tendrán que recorrer.
- Los `<link rel="preload"/"prefetch">` de optimización se declaran en el `<head>` — ver [[Preload]] y [[Prefetch]].

## Relacionadas

- [[Html]] — qué es HTML y anatomía de un documento
- [[Etiquetas]] — catálogo de etiquetas por categoría
- [[Etiquetas Meta]] — configuración del `<head>` (title, description, OG)
- [[CSS Introducción]] — estilos sobre la estructura
- [[Tipos de Display]] — maquetar las regiones con Flexbox y Grid
- [[Responsive Web Design]] — adaptar la estructura a cada dispositivo
- [[Document Object Model (DOM)]] — el árbol que resulta del parseo
- [[Formularios]] — estructura de formularios dentro del layout
