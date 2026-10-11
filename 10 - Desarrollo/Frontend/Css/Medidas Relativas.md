---
aliases: ["Unidades relativas", "Unidades de viewport"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Medidas CSS]]"
  - "[[Responsive Web Design]]"
  - "[[Flexbox]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/length#viewport-percentage_lengths"
---

# Medidas Relativas

Las medidas relativas se calculan en función de algo que **no es el píxel fijo**: la ventana (viewport), el contenido de texto o la fuente raíz. Las unidades de viewport (`vh`, `vw`...) son las que permiten que un elemento se adapte al tamaño de la pantalla sin media queries.

## Unidades de viewport

Imaginá la pantalla del navegador como el 100% de referencia:

| Unidad  | Relativa a                                   | Ejemplo práctico                            |
| ------- | -------------------------------------------- | ------------------------------------------- |
| `vh`    | **Alto** de la ventana (1vh = 1% del alto)   | Secciones a pantalla completa: `100vh`      |
| `vw`    | **Ancho** de la ventana (1vw = 1% del ancho) | Barras o texto que escala con la pantalla   |
| `vmin`  | El **menor** de los dos (ancho o alto)       | Elementos circulares adaptables             |
| `vmax`  | El **mayor** de los dos                      | Fondos o marcos que nunca deben cortarse    |

```css
.seccion-hero {
  height: 100vh;                /* ocupa exactamente la altura de la ventana */
  width: 50vw;                  /* la mitad del ancho de la ventana */
}

.logo-adaptable {
  /* Se ajusta al lado más pequeño de la pantalla */
  width: 30vmin;
  height: 30vmin;
}
```

Ejemplo clásico de centrado con estas medidas:

```css
.centrado-en-pantalla {
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
```

> [!warning] `100vh` en móviles: el problema de la barra de direcciones
> En móviles, `100vh` mide la altura **incluyendo** la barra del navegador, que aparece y desaparece al hacer scroll: una sección `100vh` queda "más alta que la pantalla" y tapa contenido. La solución moderna son las unidades dinámicas:
>
> | Unidad  | Significado                                        |
> | ------- | -------------------------------------------------- |
> | `svh`   | Small: la altura mínima (con barra visible)        |
> | `lvh`   | Large: la altura máxima (con barra oculta)         |
> | `dvh`   | Dynamic: sigue el valor actual mientras scrolleás  |
>
> Para secciones a pantalla completa en móvil, usá **`100dvh`**.

## Otras unidades relativas

| Unidad | Relativa a                          | Uso típico                                    |
| ------ | ----------------------------------- | --------------------------------------------- |
| `%`    | El mismo eje del **contenedor**     | Anchos/alturas de cajas fluidas               |
| `ch`   | Ancho del carácter "0" de la fuente | Medidas de texto: `max-width: 65ch` (párrafo legible) |
| `ex`   | Altura de la "x" minúscula          | Alineación fina de texto                      |
| `rem`  | Fuente raíz del documento           | Ver [[Medidas CSS]]                           |
| `em`   | Fuente del elemento                 | Ver [[Medidas CSS]]                           |

```css
.article p {
  max-width: 65ch; /* ~60-75 caracteres por línea: zona cómoda de lectura */
}

.tarjeta {
  width: 80%; /* 80% del ancho de su contenedor, no de la pantalla */
}
```

> [!info] `%` vs `vw`: ¿del contenedor o de la pantalla?
> `80%` es relativo al **contenedor padre**; `80vw` es relativo a la **ventana**, sin importar dónde esté el elemento. Para componentes reutilizables casi siempre querés `%`.

## Combinadas con `clamp()`: diseño fluido sin breakpoints

La forma moderna de aprovechar estas unidades es dejar de escribir valores fijos:

```css
h1 {
  font-size: clamp(2rem, 5vw + 1rem, 4rem);
  /*             mín   preferido (fluido)  máx */
}
```

El texto crece y encoge con la pantalla dentro de un rango, sin media queries — ver [[Responsive Web Design]].

## Relacionadas

- [[Medidas CSS]] — `px`, `em` y `rem`: la otra mitad de las unidades
- [[Responsive Web Design]] — media queries y `clamp()` en contexto
- [[Flexbox]] — modelos de layout que usan estas medidas en sus hijos
