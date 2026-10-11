---
aliases: ["Feature Detection", "Detección de features"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Formatos Avif y Webp]]"
  - "[[CSS Introducción]]"
  - "[[Java Script]]"
fuente: "https://modernizr.com/"
---

# Modernizr

**Modernizr** es una librería de JavaScript para **detección de compatibilidad** (*feature detection*): analiza el navegador en el que se está ejecutando la página y expone, tanto en CSS como en JavaScript, qué capacidades soporta (WebP, AVIF, `localStorage`, `intersectionobserver`, etc.) para poder mostrar una versión adecuada de la página a cada navegador.

> [!info] El principio por encima de la herramienta
> Lo importante no es Modernizr en sí, sino el patrón: **detectar capacidades, no identificar navegadores**. La librería es solo una forma conveniente de aplicar ese patrón.

## Detectar capacidades, no navegadores

La tentación clásica es adivinar el navegador por su *user agent*; eso es frágil (el string miente, hay forks, se modifica, y cada versión nueva rompe la lógica). Lo correcto es preguntar **"¿sabe el navegador hacer esto?"**:

```js
// ❌ Sniffing de navegador: frágil y siempre desactualizado
if (navigator.userAgent.includes('Trident')) { /* asumir IE viejo */ }

// ✅ Feature detection: responde a lo que importa
if ('IntersectionObserver' in window) { /* usar API moderna */ }
```

Modernizr lleva esa idea a escala: en lugar de escribir decenas de comprobaciones a mano, generás un build con las que te interesan.

## Crear un build personalizado

1. Ir a [modernizr.com](https://modernizr.com/) → *Build*.
2. Marcar solo los tests necesarios (cargar todos es innecesario y engorda el bundle).
3. Descargar el `modernizr.js` generado y guardarlo en el proyecto, por ejemplo en `js/modernizr.js`.

Luego se carga en el HTML:

```html
<script src="js/modernizr.js"></script>
```

> [!warning] Va en el `<head>`, no antes de cerrar `body`
> La regla general de rendimiento dice que los scripts que no son urgentes van al final del `<body>`. Modernizr es la excepción: la librería **agrega clases al elemento `<html>` que el CSS usa para decidir estilos**. Si carga tarde, el usuario ve primero los estilos equivocados y luego el cambio (un *flash of unstyled content*). El render-blocking acá es intencional y vale la pena. Ver [[Html]] para la anatomía del documento.

Al ejecutarse, Modernizr reemplaza la clase `no-js` del elemento `<html>` por `js` (la convención es escribir `class="no-js"` en la etiqueta `<html>` desde el HTML): sirve como red de seguridad para el caso de JavaScript deshabilitado.

## Las clases en `<html>`

Al cargar, Modernizr agrega al elemento raíz una clase por cada test: `.webp` si el navegador soporta WebP, `.no-webp` si no. Eso permite resolver la compatibilidad **enteramente desde el CSS**, sirviendo una imagen u otra:

```css
.webp .header {
  background-image: url(../img/banner.webp);
}

.no-webp .header {
  background-image: url(../img/banner.jpg);
}
```

- Los prefijos de estado son siempre el nombre del test y su negación (`audio`, `no-audio`, `canvas`, `no-canvas`).
- El patrón completo se llama **progressive enhancement**: todos reciben una versión funcional; quien soporte más, ve la mejor.

## La API de JavaScript

Los mismos tests quedan disponibles como booleanos en el objeto global `Modernizr`, para lógica condicional en scripts:

```js
if (Modernizr.webp) {
  cargarBanner('./img/banner.webp');
} else {
  cargarBanner('./img/banner.jpg');
}
```

## Alternativas nativas hoy (2026)

Modernizr sigue vivo, pero gran parte de su terreno fue cubierto por estándares nativos:

| Necesidad | Alternativa nativa |
|---|---|
| Detectar soporte de una **propiedad CSS** | At-rule `@supports` |
| Detectar una **API de JS** | `'api' in window` / `typeof api === 'function'` |
| Servir imágenes en formatos modernos | `<picture>` + `type`, **sin JS** — ver [[Formatos Avif y Webp]] |
| Detectar soporte de AVIF/WebP vía canvas | Test propio de canvas en 3 líneas |

```css
/* Detección de features en CSS puro, sin librerías */
@supports (aspect-ratio: 16 / 9) {
  .banner { aspect-ratio: 16 / 9; }
}
```

> [!tip] Cuándo sigue valiendo la pena Modernizr
> Cuando hay que mantener navegadores viejos muy extendidos (entornos corporativos, kioscos, dispositivos heredados) y conviene centralizar los tests en un solo archivo con un API uniforme. En un proyecto solo evergreen, `@supports` + `<picture>` + comprobaciones puntuales de JS cubren el 95% de los casos.

## Buenas prácticas

- **Build mínimo**: marcá solo los tests que realmente usás en CSS o JS.
- Combiná la detección con entrega progresiva: el HTML y el contenido deben funcionar aunque los estilos alternos no carguen.
- Para formatos de imagen, preferí la solución declarativa (`<picture>`) sobre la detección con JS: funciona incluso sin JavaScript — ver [[Formatos Avif y Webp]].
- No mezcles detección de features con *polyfills*: Modernizr **detecta**, no **agrega** soporte; la carga de polyfills es otra capa (hoy casi siempre innecesaria).

## Relacionadas

- [[Html]] — dónde vive el `<script>` y las clases en `<html>`
- [[Formatos Avif y Webp]] — fallback de imágenes sin necesidad de detección con JS
- [[CSS Introducción]] — estilos alternativos resueltos con las clases de Modernizr
- [[Java Script]] — feature detection a mano con el objeto global
