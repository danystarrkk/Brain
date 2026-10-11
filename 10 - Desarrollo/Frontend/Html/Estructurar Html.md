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
fuente: "https://developer.mozilla.org/es/docs/Glossary/Semantic_element"
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

## Estructura semántica por regiones

HTML moderno da significado a cada región del documento. Esta es la estructura recomendada y cuándo usar cada etiqueta:

| Etiqueta | Cuándo usarla |
|---|---|
| `<header>` | Encabezado del sitio o de una sección (logo, título, menú) |
| `<nav>` | Grupo de enlaces de navegación |
| `<main>` | Contenido principal de la página — **solo uno** por archivo |
| `<section>` | Bloque de contenido que introduce un nuevo tema (su primer hijo suele ser un heading) |
| `<article>` | Contenido independiente y autocontenido (un post, una tarjeta) |
| `<aside>` | Contenido complementario al principal (barra lateral, nota al margen) |
| `<footer>` | Pie del sitio o de una sección (créditos, enlaces legales) |

La regla de oro: **usá la etiqueta semántica antes que `<div>`**. `<div>` es el último recurso cuando ninguna de las anteriores aplica — ver [[Etiquetas]] para el catálogo completo.

## Lineamientos de una estructura válida

- **Un solo `<h1>`** por página, y sin saltos jerárquicos (`<h1>` → `<h3>` sin pasar por `<h2>`).
- **Etiquetas siempre cerradas** (excepto las vacías: `<img>`, `<br>`, `<meta>`).
- **Anidamiento correcto**: no se pueden cruzar etiquetas (`<p><b></p></b>` es inválido).
- **Indentación consistente**: la estructura se lee a simple vista cuando la sangría refleja el anidamiento.
- Separar **estructura (HTML)**, **estilo (CSS)** y **comportamiento (JavaScript)** — cada uno en su archivo.

> [!warning] Los navegadores corrigen por su cuenta
> Si la estructura es inválida, el navegador intenta repararla automáticamente, y el resultado raras veces es el que esperabas (etiquetas "tragadas", estilos aplicados donde no toca). Validá el HTML en el [validator del W3C](https://validator.w3.org/) antes de culpar a CSS o JS de un problema de maquetación.

## Buena estructura = mejores resultados

- **SEO**: los buscadores usan la jerarquía de encabezados y las etiquetas semánticas para entender el contenido — un `<main>` claro le dice a Google cuál es el contenido real de la página.
- **Accesibilidad**: los lectores de pantalla navegan por regiones (`header`, `nav`, `main`); una estructura plana de `<div>`s es ilegible para ellos.
- **Responsive**: con una estructura por regiones, la maquetación responsive (ver [[Responsive Web Design]]) se aplica por zonas y no por hacks puntuales.
- **Mantenimiento**: quien abra el proyecto dentro de un año encuentra dónde hacer los cambios sin leer toda la página.

## Relacionadas

- [[Html]] — qué es HTML y anatomía de un documento
- [[Etiquetas]] — catálogo de etiquetas por categoría
- [[Etiquetas Meta]] — configuración del `<head>`
- [[CSS Introducción]] — estilos sobre la estructura
- [[Responsive Web Design]] — adaptar la estructura a cada dispositivo
