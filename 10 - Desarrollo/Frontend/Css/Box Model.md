---
aliases: ["Box Model", "Modelo de caja", "box-sizing"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Medidas CSS]]"
  - "[[Medidas Relativas]]"
  - "[[Tipos de Display]]"
  - "[[Especificidad]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/Box_model"
---

# Box Model

En CSS **todo es una caja**: párrafos, imágenes, botones, spans vacíos. El modelo de caja describe que el tamaño final de un elemento se calcula con 4 capas concéntricas:

1. **Contenido** — el texto o la imagen, con su `width`/`height`.
2. **Relleno (`padding`)** — espacio transparente dentro del borde.
3. **Borde (`border`)** — la línea que rodea el relleno.
4. **Margen (`margin`)** — espacio vacío *fuera* de la caja, que separa a otros elementos.

![[Pasted image 20250124165421.png]]

## El problema del box-sizing por defecto

Por defecto (`content-box`), `width` y `height` afectan **solo al contenido**: padding y border se *suman* encima. Si te piden un elemento de 200px de ancho total, tenés que restar a mano el padding y el borde:

```css
/* Con content-box: 200px de ancho + 2×20px padding + 2×2px border = 244px reales */
.caja-mala {
  width: 200px;
  padding: 20px;
  border: 2px solid black;
}

/* Con border-box: width = ancho TOTAL visible */
.caja-buena {
  box-sizing: border-box;
  width: 200px; /* sigue midiendo 200px con todo el padding dentro */
  padding: 20px;
  border: 2px solid black;
}
```

## La solución canónica de `border-box`

Gracias a un desarrollador de Chrome (Paul Irish) que publicó lo [siguiente](https://www.paulirish.com/2012/box-sizing-border-box-ftw/), se puede aplicar `border-box` a **todo el documento** de una sola vez:

```css
html {
  box-sizing: border-box;
}
*,
*::before,
*::after {
  box-sizing: inherit;
}
```

Desglose del truco:

- Se declara `border-box` en `html` (la raíz del documento).
- El selector universal `*` selecciona **todos** los elementos, y con `box-sizing: inherit` cada uno toma el valor de su padre — así todo el árbol queda en `border-box` sin repetir la propiedad.
- Las comas separan selectores distintos a los que se aplican **las mismas declaraciones**.
- `::before` y `::after` son **pseudo-elementos**: cajas generadas por CSS que no existen en el HTML pero que también necesitan el `border-box`.

> [!tip] Práctica moderna
> Hoy casi todo framework (Tailwind, Bootstrap) y la mayoría de resets incluyen este snippet. Si usás [[Normalize]] o un reset moderno, revisá que ya lo traiga.

## Cómo se calcula el ancho total

```css
/* content-box */
.ancho-total = width + padding*2 + border*2

/* border-box */
.ancho-total = width   /* el padding y el border viven dentro */
```

El `margin` **nunca** cuenta para el ancho del elemento: es espacio entre cajas. Además, los márgenes verticales de dos bloques adyacentes **se colapsan** (no se suman: quedan el mayor de los dos). El margin no colapsa dentro de flex o grid containers — ver [[Flexbox]] y [[CSS Grid]].

## Inspeccionarlo en DevTools

Las DevTools de Chrome/Firefox muestran el diagrama del box model con los 4 valores en vivo. Es la forma más rápida de responder "¿por qué esto se sale de su contenedor?".

> [!warning] Padding con porcentaje
> `padding` en % se calcula **siempre sobre el ancho del padre**, nunca sobre la altura — útil para el truco de la caja con proporción fija (`padding-top: 50%` = altura la mitad del ancho).

## Relacionadas

- [[Medidas CSS]] — unidades para expresar tamaños de caja
- [[Medidas Relativas]] — por qué `rem` suele ser mejor que `px`
- [[Flexbox]] — distribución de cajas en una dimensión
- [[CSS Grid]] — distribución de cajas en dos dimensiones
- [[Tipos de Display]] — cómo `display` cambia el tipo de caja y su flow
