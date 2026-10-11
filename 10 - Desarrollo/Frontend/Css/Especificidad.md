---
aliases: ["Specificity", "Cascada y especificidad"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Selectores en CSS]]"
  - "[[Anatomía de CSS]]"
  - "[[Estilos para escribir CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/Specificity"
---

# Especificidad

La **especificidad** es la puntuación que asigna el navegador a cada selector para decidir **qué regla gana** cuando varias apuntan al mismo elemento. Mientras más "descriptivo" (específico) es el selector, más prioridad tiene.

## El ejemplo clásico

```css
h1.titulo span {
  color: green;
}

.titulo span {
  color: red;
}
```

La regla 1 apunta al `span` dentro de un `h1` que tenga la clase `titulo`; la regla 2, al `span` dentro de cualquier `titulo`. Por la cascada pura "gana la última", pero la regla 2 es **más genérica**: su especificidad es menor y **no puede** pisar a la primera. El `span` dentro de `h1.titulo` queda **verde**.

## Cómo se calcula

Cada selector se puntúa en 3 niveles (A, B, C), como un número de 3 dígitos:

| Nivel | Componente               | Ejemplos                          | Puntos |
| ----- | ------------------------ | --------------------------------- | ------ |
| A     | IDs (`#`)                | `#header`                         | 1–0–0  |
| B     | Clases, atributos, pseudo-clases | `.card`, `[type="text"]`, `:hover` | 0–1–0 |
| C     | Elementos, pseudo-elementos | `p`, `div`, `::before`          | 0–0–1  |

```css
p                { } /* 0–0–1 */
p span           { } /* 0–0–2 */
.titulo          { } /* 0–1–0 */
h1.titulo        { } /* 0–1–1 */
#nav .item a     { } /* 1–1–1 — gana a todo lo anterior */
```

Se comparan de izquierda a derecha: **quien gana en A no se discute**; solo si empatan A se mira B, y así.

> [!warning] Los números no son decimales
> `0–1–0` (una clase) le gana a `0–0–12` (doce elementos): primero se miran las clases. Y 2 clases (0–2–0) le ganan a 1 id... no: 1 id (1–0–0) gana a cualquier cantidad de clases. El orden de importancia es **ID > clase > elemento**.

## Casos especiales

- **`*` (universal)**: aporta 0 puntos.
- **`!important`**: no es especificidad, es una banda que salta toda la cascada. Ver abajo.
- **Selectores de contexto** (`.a .b`): se suman todas sus partes.
- **`:is()` / `:not()`**: aportan la especificidad de su argumento más alto; `:where()` aporta **0** — truco útil para bases de estilos sobreescribibles.

```css
:where(.reset) h1 { } /* 0–0–1: el :where() no suma */
```

## Herencia: el otro mecanismo

Confundir cascada con herencia es común. La **herencia** es el flujo de ciertas propiedades (`color`, `font-family`, `line-height`...) desde el padre a los hijos **sin selector de por medio**. No compite con la especificidad: es un canal distinto que solo aplica si ninguna regla (aunque sea genérica) define la propiedad en el hijo.

> [!tip] Escalar la especificidad a propósito
> Hay dos estrategias para ganar batallas sin `!important`:
> 1. **Subir la especificidad** del selector propio (`.nav .item.activo`).
> 2. **Bajar la de la competencia**, reorganizando el CSS.
> La opción moderna y más limpia son las **cascade layers** (`@layer base, componentes, utilidades;`), que resuelven el orden por capas declarado explícitamente, por encima de la especificidad.

## `!important`: usar con criterio

`!important` vence a cualquier especificidad, pero crea una escalada difícil de romper (para deshacer un `!important` necesitás otro). Reservalo para utilidades puntuales — por ejemplo, debug — y preferí reestructurar selectores. Ver [[Estilos para escribir CSS]] para metodologías que evitan este problema desde el diseño del naming.

## Inspeccionarlo en DevTools

Las DevTools muestran la puntuación de especificidad al hacer hover sobre un selector en la pestaña Styles, y en el panel Computed podés ver qué regla ganó para cada propiedad.

## Relacionadas

- [[Selectores en CSS]] — todos los selectores y su sintaxis
- [[Anatomía de CSS]] — estructura de las reglas
- [[Estilos para escribir CSS]] — BEM y compañía para mantener la especificidad plana
- [[Sintaxis de Sass]] — por qué la anidación de Sass dispara la especificidad
