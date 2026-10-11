---
aliases: ["HTML", "Qué es HTML"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Estructurar Html]]"
  - "[[Etiquetas]]"
  - "[[Etiquetas Meta]]"
  - "[[CSS Introducción]]"
  - "[[Document Object Model (DOM)]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML"
---

# Qué es HTML?

**HTML** son las iniciales de *"Hypertext Markup Language"* (Lenguaje de Marcas de Hipertexto). Es el lenguaje con el que se construye el contenido de toda página web: define la estructura y el significado de los elementos (títulos, párrafos, imágenes, enlaces), pero **no su apariencia ni su comportamiento**.

Es un lenguaje fácil de aprender si se pone en práctica: es muy intuitivo, y a través de **etiquetas** escribimos el marcado del contenido:

```html
<p>        → Párrafos
<nav>      → Navegaciones
<header>   → Encabezados de sitio o de elemento
<main>     → Contenido principal
<footer>   → Pie de página o de elemento
```

> [!info] HTML5
> HTML5 fue un gran salto (semántica, multimedia, formularios enriquecidos, APIs del navegador), pero hoy en día basta con llamarlo **HTML**: todas las versiones modernas son en la práctica HTML5.

## Lenguaje de marcado, no de programación

La diferencia clave:

| | Lenguaje de marcado (HTML) | Lenguaje de programación (JS, Python…) |
|---|---|---|
| Propósito | **Describir** contenido y su significado | **Dar comportamiento**: lógica, cálculos, control |
| ¿Tiene lógica? | No: no hay variables, bucles ni condicionales | Sí |
| ¿Se ejecuta? | Se **interpreta y renderiza** el navegador | Se **ejecuta** paso a paso |

HTML describe el *qué*; el *cómo se ve* lo resuelve CSS ([[CSS Introducción]]) y el *cómo se comporta* JavaScript ([[Java Script]]).

## Anatomía: etiquetas y atributos

El HTML se compone de **etiquetas** (tags) que envuelven contenido:

```html
<p>Hola, mundo</p>
```

- **Etiqueta de apertura**: `<p>` · **Etiqueta de cierre**: `</p>` · **Contenido**: `Hola, mundo`.
- Muchas etiquetas aceptan **atributos** para configurarlas: `<img src="foto.jpg" alt="Foto">`.
- Algunas etiquetas son **vacías** (no tienen cierre): `<img>`, `<br>`, `<meta>`, `<hr>`.

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

- **`<!DOCTYPE html>`**: le dice al navegador que es HTML5 moderno (sin esto, los navegadores entran en modo de compatibilidad).
- **`head`**: información *meta* del sitio: idioma, título, descripción, codificación, enlaces a hojas de estilo CSS, scripts. Nada de esto se ve en la página — ver [[Etiquetas Meta]].
- **`body`**: todo lo que **se ve** en la página: textos, imágenes, videos, botones, animaciones.

> [!tip] El navegador siempre intenta arreglar tu HTML
> Los navegadores son tolerantes con errores (etiquetas sin cerrar, anidamiento inválido), pero "funcione" no significa "bien hecho": el navegador corrige por su cuenta y ese arreglo puede no ser el que esperabas. Validá contra el validator del W3C cuando dudes.

## Semántica: HTML moderno = HTML semántico

Las etiquetas semánticas cuentan al navegador (y a buscadores y lectores de pantalla) **qué significa** el contenido:

- `<h1>`–`<h6>`: niveles de encabezado (jerarquía del documento).
- `<header>`, `<nav>`, `<main>`, `<footer>`: regiones del sitio.
- `<section>`, `<article>`, `<aside>`: bloques de contenido con significado propio.
- `<form>`, `<label>`, `<table>`, `<figure>`: estructuras específicas — ver [[Formularios]] y [[Etiquetas]].

El opuesto son las etiquetas **presentacionales** (`<font>`, `<center>`, `<b>`, `<i>`), heredadas de HTML antiguo: están deprecadas y el estilo hoy se resuelve con CSS. Cuando ninguna etiqueta semántica aplica, se usan `<div>` (bloque) y `<span>` (en línea).

## Malas prácticas

HTML tolera malas prácticas y "funciona", pero eso no significa usarlas:

- Mezclar presentación con estructura (estilos en línea en vez de CSS).
- Omitir el `alt` de las imágenes o poner texto vacío sin motivo.
- Saltarse la jerarquía de encabezados (`<h1>` directo a `<h4>`).
- Usar `<div>`/`<span>` para todo en vez de etiquetas semánticas.

Los buscadores penalizan los sitios con malas prácticas (contenido sin estructura, texto en imágenes, metadatos ausentes), y las herramientas de accesibilidad directamente no pueden interpretar el contenido.

## El HTML y el navegador

El navegador parsea el HTML y construye el **Document Object Model (DOM)**: un árbol de objetos que representa la página y sobre el que actúan CSS y JavaScript — ver [[Document Object Model (DOM)]] y [[Acceder a Html]].

## Relacionadas

- [[Estructurar Html]] — la estructura completa y válida de un documento
- [[Etiquetas]] — catálogo de etiquetas por categoría
- [[Etiquetas Meta]] — configuración del `<head>`
- [[CSS Introducción]] — dar estilo a la estructura HTML
- [[Java Script]] — comportamiento dinámico de la página
- [[Document Object Model (DOM)]] — cómo el navegador convierte el HTML en un árbol de objetos
