---
aliases: ["CSS Introducción", "Qué es CSS", "Introducción a CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-05-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Anatomía de CSS]]"
  - "[[Especificidad]]"
  - "[[Selectores en CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/What_is_CSS"
---

# CSS Introducción

**CSS** (*Cascading Style Sheets*, hojas de estilo en cascada) es el lenguaje encargado del aspecto visual: colores, tamaños, espacios, animaciones y adaptación a distintos dispositivos. Mientras [[Html]] se ocupa de la **estructura y el significado** del contenido, CSS se ocupa de cómo **se ve**.

Concretamente, CSS controla:

- Tamaños y tipos de fuentes
- Colores (texto, fondos, bordes)
- Espacios, márgenes y el [[Box Model|modelo de caja]]
- Posicionamiento y layouts (Flexbox, Grid)
- Adaptación a distintos dispositivos ([[Responsive Web Design]])
- Animaciones y transiciones

## ¿Por qué "en cascada"?

Se llama así porque cuando varias reglas compiten por el mismo elemento, el navegador aplica un orden de resolución — la **cascade**: primero decide qué reglas aplican según su origen e importancia, luego resuelve conflictos por **[[Especificidad|especificidad]]** del selector y, a igualdad, por **orden en el archivo** (gana la última declarada).

```css
p {
  color: blue;
}

p {
  color: red; /* gana: mismo selector y specificity, pero aparece después */
}
```

> [!info] Cascada ≠ "gana siempre el último"
> La última regla solo gana cuando la especificidad es la misma. `#titulo { color: red }` posterior a `p { color: blue }` no pinta de azul los párrafos con ese id: los selectores más específicos dominan — ver [[Especificidad]].

## Anatomía mínima

La sintaxis no es complicada, aunque al principio pueda resultar extraña:

```css
p {
  color: blue;
}
```

Un selector (`p`), un bloque entre llaves y declaraciones `propiedad: valor;`. La estructura completa, con pseudo-clases, comentarios y unidades, se detalla en [[Anatomía de CSS]].

## Cómo llega el CSS al navegador

Hay tres vías principales:

1. **Archivo externo** (la recomendada): un `<link>` en el `<head>` — ver [[Crear archivo para CSS]].
2. **Interno**: un bloque `<style>` en el HTML, útil para estilos críticos de una sola página.
3. **En línea**: el atributo `style="..."` en cada etiqueta — máxima especificidad, casi imposible de mantener; usar solo para ajustes puntuales o con JS.

```html
<head>
  <link href="css/styles.css" rel="stylesheet"> <!-- externo -->
  <style> h1 { color: teal; } </style>          <!-- interno -->
</head>
<body>
  <p style="color: gray;">En línea</p>
</body>
```

## Herramientas de la caja moderna

El CSS actual va mucho más allá de `color` y `margin`:

| Herramienta                    | Para qué                                            |
| ------------------------------ | --------------------------------------------------- |
| Custom properties (`--var`)    | Variables en runtime, temas — ver [[Colores CSS]]   |
| Flexbox y Grid                 | Layouts modernos — ver [[Flexbox]] y [[CSS Grid]]   |
| Funciones `clamp()`, `min()`, `max()` | Tamaños fluidos sin media queries             |
| Media queries y container queries | Adaptación a pantalla o contenedor              |
| `@layer` (cascade layers)      | Control explícito del orden de la cascada           |
| `:has()`                       | Selector "padre" según sus hijos                    |

> [!tip] Cómo se aprende bien CSS
> CSS no es difícil: es un sistema pequeño con reglas de resolución (cascada, especificidad, herencia). Lo que lo hace confuso al principio es no conocer esas reglas. Empezá por [[Selectores en CSS]], entendé [[Especificidad]] y practicá layouts con [[Flexbox]] y [[CSS Grid]].

## Relacionadas

- [[Anatomía de CSS]] — estructura detallada de una regla
- [[Selectores en CSS]] — cómo apuntar a los elementos
- [[Especificidad]] — qué regla gana en un conflicto
- [[Crear archivo para CSS]] — enlazar CSS desde el HTML
- [[Colores CSS]] — valores de color y variables
