---
aliases: ["Display", "Display block e inline"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Flexbox]]"
  - "[[CSS Grid]]"
  - "[[Box Model]]"
  - "[[Tipos de Posiciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/display"
---

# Tipos de Display

La propiedad `display` define **cómo se comporta un elemento en el flujo de la página**: si ocupa todo el ancho, si va al lado de otro, si deja de existir visualmente. Cada etiqueta HTML trae un display **por defecto**, y solo hace falta cambiarlo cuando el diseño lo exige.

## Los dos displays por defecto: block e inline

### Display block

Ocupa **todo el ancho disponible** y se coloca uno debajo del otro, como ladrillos apilados:

![[Pasted image 20250124163054.png]]

Los tienen los headings (`h1`–`h6`), párrafos `p`, `div`, listas `ul`/`ol`, secciones... Son el default de casi todo el HTML estructural.

Características:

- Aceptan `width`, `height`, `margin` y `padding` en **todos** los lados (ojo: los márgenes verticales entre bloques **se colapsan** — toman el mayor, no se suman; detalle en [[Box Model]]).

### Display inline

Los elementos se colocan **uno a la derecha del otro**, fluyendo con el texto:

![[Pasted image 20250124163114.png]]

Los tienen los enlaces `a`, `span`, `strong`, `em`, `img`, etc.

Características:

- **Ignoran `width`/`height`**: su tamaño lo decide el contenido.
- Los márgenes y paddings verticales **no empujan** a los demás elementos (pintan, pero no desplazan).

> [!warning] El error clásico
> Ponerle `width: 200px` a un `<a>` "no funciona" — no es un bug: los inline no aceptan ancho. Solución: `display: inline-block`.

## La pieza intermedia: `inline-block`

Combina ambos comportamientos: fluye en línea como un inline, pero acepta `width`/`height` y paddings completos como un block. Era *la* solución para tarjetas en fila antes de Flexbox, y sigue siendo útil para cosas puntuales:

```css
.badge {
  display: inline-block;
  width: 12px;
  height: 12px;
}
```

## El display moderno: flex y grid

Los displays de layout **son hijos de esta misma familia**: reemplazan al `inline-block` (y a los float) para construir diseños.

```css
.nav { display: flex; }    /* hijos en fila/columna — ver [[Flexbox]] */
.grilla { display: grid; } /* hijos en filas y columnas — ver [[CSS Grid]] */
```

| `display`       | Comportamiento                                        |
| --------------- | ----------------------------------------------------- |
| `block`         | Ocupa todo el ancho, apilado                         |
| `inline`        | Fluye con el texto, tamaño por contenido             |
| `inline-block`  | Fluye en línea pero acepta dimensiones               |
| `flex`          | Contenedor flex (layout unidimensional)               |
| `grid`          | Contenedor grid (layout bidimensional)                |
| `contents`      | El elemento "desaparece": sus hijos pasan al flujo del abuelo |
| `none`          | Se oculta y **no ocupa espacio**                      |

## `display: none` vs `visibility: hidden`

| Propiedad            | Se ve | Ocupa espacio | Existe para lectores de pantalla |
| -------------------- | ----- | ------------- | -------------------------------- |
| `display: none`      | No    | No            | No                               |
| `visibility: hidden` | No    | **Sí**        | No                               |

Para animar la aparición/ocultamiento de un elemento hoy se prefiere `opacity` + `translate` (ver [[Animaciones]]), que además respeta `prefers-reduced-motion`.

> [!tip] No cambies el display "por si acaso"
> No es necesario modificar los displays de las etiquetas siempre y cuando no sea necesario quitar o reemplazar el que viene por defecto: cada cambio de `display` rompe el flujo heredado y debe justificarse en el diseño.

## Relacionadas

- [[Flexbox]] — `display: flex` en profundidad
- [[CSS Grid]] — `display: grid` en profundidad
- [[Box Model]] — cómo dimensionar cada tipo de display
- [[Tipos de Posiciones]] — sacar elementos del flujo con `position`
