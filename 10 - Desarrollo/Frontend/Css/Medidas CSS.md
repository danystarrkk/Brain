---
aliases: ["Unidades absolutas", "Medidas px, em y rem"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Medidas Relativas]]"
  - "[[Box Model]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/length"
---

# Medidas CSS

CSS tiene dos familias de medidas: **absolutas** (`px`, `pt`...) y **relativas** (`em`, `rem`, `%`, `vw`...). Las absolutas son fijas; las relativas se calculan en función de algo — por eso son las que permiten diseños adaptables y accesibles.

## Unidades absolutas: `px`

El píxel es la unidad más fácil e intuitiva: 1px es siempre 1px (en CSS, un píxel lógico). Mide bien para lo **pequeño y fijo**: bordes, sombras, radios, iconos.

```css
.boton {
  border: 1px solid #333;   /* borde fino: px está bien */
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}
```

> [!warning] El problema de los px para tipografía y espaciado
> Si todo está en px, la web **no respeta la configuración de tamaño de fuente del usuario** ni escala con el zoom: mal para accesibilidad (WCAG) y mal para adaptar el diseño. Para tamaños de texto y espacios usá unidades relativas.

## Unidades relativas: `em` y `rem`

| Unidad  | Relativa a                                        | Comportamiento                                            |
| ------- | ------------------------------------------------- | --------------------------------------------------------- |
| `em`    | El `font-size` **del propio elemento** (o del padre al heredarse) | Se acumula: anidado, 1em no es el mismo valor en dos niveles |
| `rem`   | El `font-size` del **elemento raíz** (`html`)    | Siempre el mismo valor en todo el documento               |

- **`em`**: ideal para componentes autocontenido (un padding que crece con el texto de su tarjeta), pero su acumulación lo hace impredecible en layouts grandes.
- **`rem`**: unidad recomendada para layout y tipografía: sin sorpresas anidadas y respeta el zoom/ tamaño de fuente del usuario (bueno para accesibilidad).

> [!info] La excepción donde `em` gana
> Para bordes y radios dentro de componentes escalables, `em` es mejor que `rem`: si el usuario amplía el texto de la tarjeta, el borde crece proporcionalmente.

## El truco del 62.5%: 1rem = 10px

Como `rem` es relativo al `font-size` de `html` (por defecto 16px), cambiarlo a `62.5%` (= 10px) hace que **1rem = 10px**, y los cálculos se vuelven triviales: un padding de 24px es `2.4rem`, un margen de 40px es `4rem`:

```css
html {
  box-sizing: border-box;
  font-size: 62.5%; /* 1rem = 10px */
}

*,
*::before,
*::after {
  box-sizing: inherit; /* hereda el border-box — ver [[Box Model]] */
}

body {
  font-size: 1.6rem; /* 16px: tamaño base de lectura "normal" */
}
```

Dos detalles de este setup:

- El `box-sizing: inherit` con `*` propaga el `border-box` a todo el árbol sin romper la herencia — patrón recomendado en [[Box Model]].
- `body` vuelve a `1.6rem` para que el texto de lectura siga midiendo los 16px reales.

> [!warning] Conocé el costo del 62.5%
> 1. `rem` deja de ser "el tamaño que eligió el usuario": si alguien configura fuentes grandes en el navegador, 1rem ya no sigue la escala original del navegador, solo la proporción. 2. Genera conversión mental constante entre px y rem. Es un trade-off válido y muy usado, pero no es gratis: muchos equipos hoy prefieren dejar `html` en 16px y usar valores "crudos" (`1.5rem`, `2.25rem`).

## Tabla resumen

| Unidad | Relativa a               | Uso recomendado                          |
| ------ | ------------------------ | ---------------------------------------- |
| `px`   | Nada (absoluta)          | Bordes, sombras, radios, iconos          |
| `em`   | Fuente del elemento      | Componentes escalables, tamaños locales  |
| `rem`  | Fuente raíz (`html`)     | Tipografía y espaciado de layout         |
| `%`    | Fuente/padre según contexto | Anchos y alturas de caja              |
| `vw/vh` | Viewport               | Ver [[Medidas Relativas]]                |

Para diseños fluidos modernos combiná estas unidades con `clamp()` — ver [[Responsive Web Design]].

## Relacionadas

- [[Medidas Relativas]] — `%`, `vh`, `vw` y las unidades del viewport
- [[Box Model]] — por qué `border-box` cambia cómo se calcula el tamaño
- [[Responsive Web Design]] — dónde entra cada unidad en un diseño adaptable
