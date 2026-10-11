---
aliases: ["Fuentes personalizadas", "Google Fonts"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Preload]]"
  - "[[Imágenes CSS]]"
fuente: "https://fonts.google.com/"
---

# Fuentes Externas

CSS permite usar cualquier tipografía, no solo las que el usuario tiene instaladas. Hay dos caminos: usar una **fuente del sistema** (ya presente en el equipo) o cargar una **fuente externa** desde un servidor.

## `font-family`: la pila de fuentes

`font-family` acepta una **lista** de fuentes separada por comas: el navegador usa la primera que encuentre disponible en el sistema y, si no hay ninguna, cae en el tipo genérico final:

```css
body {
  font-family: "Segoe UI", Roboto, Arial, Helvetica, sans-serif;
  /*           ↑ primero      ↑ si no está  ↑ fallbacks ↑ último recurso */
}
```

Con esto ya cambiamos a las fuentes predeterminadas que todos los navegadores/ sistemas tienen (Arial, Helvetica, Times...). Los tipos genéricos válidos son: `serif`, `sans-serif`, `mono`, `cursive`, `fantasy` y `system-ui`.

> [!tip] Aplicá la fuente en `body`
> Si vas a usar el mismo tipo de fuente en toda la web, declarala en `body` y se propagará por **herencia** a todo el documento; solo marcarás las excepciones.

## Cargar fuentes externas: Google Fonts

Para fuentes que nadie tiene instalada hay que descargarlas de algún lado. La fuente más usada es [Google Fonts](https://fonts.google.com/), con miles de tipografías gratuitas. El flujo:

1. Buscá la fuente (por ejemplo **Roboto**) y elegí los pesos/estilos que necesites.

   ![[Pasted image 20250124162423.png]]

2. La página te da el `<link>` para el HTML y el `font-family` para el CSS.

   ![[Pasted image 20250124162505.png]]

3. Agregá el `<link>` en el `<head>` **antes de tu CSS** (para no tener conflictos de cascada):

```html
<head>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link
    href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap"
    rel="stylesheet"
  />
  <link rel="stylesheet" href="css/estilos.css" />
</head>
```

4. Y usala en el CSS:

```css
body {
  font-family: "Roboto", sans-serif;
}
```

### Rendimiento: cada fuente externa es una ida y vuelta

> [!warning] Las fuentes externas cuestan tiempo de carga
> Cada fuente agrega una petición de red y puede provocar **FOIT/FOUT** (texto invisible o texto con fuente provisional parpadeando). Tres medidas obligatorias:
> - **`preconnect`** a `fonts.googleapis.com` y `fonts.gstatic.com` (abre la conexión TLS antes).
> - **`display=swap`** en la URL: muestra un fallback mientras carga la real.
> - Pedí **solo los pesos que uses** (400 y 700, no toda la familia): cada peso es un archivo aparte. Ver también [[Preload]].

## `@font-face`: la ruta manual

Detrás de Google Fonts está el at-rule `@font-face`, que también podés usar con tus propios archivos:

```css
@font-face {
  font-family: "MiFuente";
  src: url("../fuentes/mifuente.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

h1 {
  font-family: "MiFuente", sans-serif;
}
```

> [!info] Formato recomendado: WOFF2
> Hoy el estándar es **WOFF2**: mejor compresión que WOFF y lo soportan todos los navegadores modernos. Para legacy se sirven también WOFF como fallback.

## Buenas prácticas resumidas

| Práctica                              | Por qué                                        |
| ------------------------------------- | ---------------------------------------------- |
| Declarar la fuente en `body`          | Aprovecha la herencia                          |
| Usar `display=swap`                   | Evita texto invisible durante la carga         |
| Limitarse a 1 familia y 2-3 pesos     | Cada peso = un archivo más                      |
| Combinar con fuentes del sistema      | Fallback instantáneo y offline                 |
| Priorizar `sans-serif` para UI        | Legibilidad en pantallas pequeñas              |

## Relacionadas

- [[Html]] — dónde va el `<link>` dentro del `<head>`
- [[Preload]] — acelerar la llegada de recursos críticos como fuentes
- [[Imágenes CSS]] — el otro recurso externo típico de un diseño
