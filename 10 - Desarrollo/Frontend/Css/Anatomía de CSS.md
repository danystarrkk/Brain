---
aliases: ["Anatomía CSS", "Sintaxis básica de CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[CSS Introducción]]"
  - "[[Selectores en CSS]]"
  - "[[Especificidad]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/Getting_started"
---

# Anatomía de CSS

Toda regla de CSS se compone de un **selector** (qué elemento afecta) y un bloque de **declaraciones** (qué cambia). Entender esta estructura es el cimiento para leer y escribir cualquier hoja de estilo.

## La estructura básica

```css
p {
  color: blue;
}
```

- **`p`** es el **selector**: apunta a todas las etiquetas `<p>` del HTML (ver [[Etiquetas]] y [[Selectores en CSS]] para la lista completa de selectores).
- **`color: blue;`** es una **declaración**, formada por:
  - **propiedad** (`color`): qué aspecto queremos controlar.
  - **valor** (`blue`): cómo queda ese aspecto.
- Cada declaración termina en **punto y coma `;`**. La única declaración sin `;` es la última *antes* de la llave de cierre, pero escribirla siempre es mejor hábito: agregar una nueva declaración después ya no obliga a recordar el `;` anterior.

```css
/* Varias declaraciones en un mismo bloque */
.card {
  color: #1a1a2e;   /* texto */
  padding: 1rem;     /* espacio interior */
  border-radius: 8px; /* esquinas redondeadas */
}
```

## Selector y bloque

```
.card { color: blue; }
└─┬─┘ └────────┬────────┘
selector    bloque de declaraciones
```

El selector puede ser simple (`p`), combinado (`.card .titulo`) o incluir pseudo-clases y pseudo-elementos (`a:hover`, `p::first-line`). Entre selectores se usan **comas** para aplicar las mismas declaraciones a varios elementos a la vez:

```css
h1, h2, h3 {
  font-family: Georgia, serif;
}
```

## Propiedades: kebab-case y alias de JS

Las propiedades CSS usan **guion medio** (`background-color`, `font-size`). Al llevarlas a JavaScript hay que usar corchetes o camelCase:

```js
elemento.style.backgroundColor = 'blue'; // JS convierte a background-color
```

> [!info] La última declaración gana
> Si dos reglas con el mismo selector declaran la misma propiedad, se aplica **la última** que aparezca en la hoja de estilo. Este es el punto de partida de la cascada y la [[Especificidad]]: reglas más específicas ganan aunque aparezcan antes.

## Valores: keywords, números con unidad y funciones

```css
.box {
  color: blue;              /* keyword */
  width: 300px;             /* número + unidad */
  opacity: 0.8;             /* número puro (sin unidad) */
  margin: 0 auto;           /* una keyword + una función */
  width: calc(100% - 40px); /* función: calc() */
}
```

> [!warning] Unidad obligatoria
> Las longitudes necesitan unidad (`10px`, `2rem`, `50%`). **Excepción: `0`** — `margin: 0` es válido, pero `margin: 0px` también lo es. Unidades relativas como `rem`, `em` o `vw` se detallan en [[Medidas CSS]] y [[Medidas Relativas]].

## Comentarios

```css
/* Comentario de una o varias líneas */
// Los comentarios de dos barras NO existen en CSS puro
```

Los comentarios de dos barras (`//`) solo funcionan en preprocesadores como Sass — ver [[Sintaxis de Sass]].

## Múltiples valores en una propiedad

Muchas propiedades aceptan de 1 a 4 valores que se leen como caras de un reloj (`arriba derecha abajo izquierda`):

```css
padding: 1rem 2rem 1rem 2rem;  /* 4 valores */
padding: 1rem 2rem;            /* 2 valores: vertical | horizontal */
padding: 1rem;                 /* 1 valor: los 4 lados */
```

## Relacionadas

- [[CSS Introducción]] — qué es CSS y para qué sirve
- [[Selectores en CSS]] — todos los selectores disponibles
- [[Especificidad]] — qué regla gana cuando compiten varias
- [[Medidas CSS]] — unidades absolutas de medida
- [[Medidas Relativas]] — `rem`, `em`, `%`, `vh`, `vw`
