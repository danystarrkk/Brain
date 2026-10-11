---
aliases: ["Cuando usar Grid o Flexbox", "Grid vs Flexbox"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Flexbox]]"
  - "[[CSS Grid]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_flexible_box_layout"
---

# Cuando usar GRID o Flexbox

No hay respuesta única: **no existe un modelo "mejor" que el otro**. La regla práctica es preguntar *¿cuántas dimensiones controla esta decisión?* y, sobre todo, recordar que **se pueden usar los dos juntos** en el mismo diseño: Grid para el esqueleto de la página y Flexbox para los detalles dentro de cada caja.

## Utilizá Flexbox

Flexbox organiza elementos en **una sola dimensión** (una fila o una columna). Es la herramienta para **alinear y distribuir** lo que está *dentro* de un contenedor, como una barra de navegación:

![[Pasted image 20250124170846.png]]

Casos típicos de Flexbox:

- Navbar con logo a la izquierda y enlaces a la derecha (`justify-content: space-between`).
- Botones centrados vertical y horizontalmente.
- Cards de igual altura en una fila.
- Apilar ítems con `gap` en una columna.

```css
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}
```

## Utilizá CSS Grid

Grid define el **layout general del sitio** en dos dimensiones: columnas, filas y áreas de la página.

![[Pasted image 20250124170917.png]]

Casos típicos de Grid:

- Estructura header / nav / contenido / sidebar / footer.
- Galerías de imágenes con columnas fluidas.
- Cualquier diseño donde un ítem debe cruzar varias filas o columnas.

```css
.pagina {
  display: grid;
  grid-template-areas:
    "header header"
    "nav    main"
    "footer footer";
  grid-template-columns: 240px 1fr;
  gap: 1.5rem;
}
```

Detalle completo en [[CSS Grid]].

## Tabla resumen

| Criterio                  | Flexbox                          | CSS Grid                         |
| ------------------------- | -------------------------------- | -------------------------------- |
| Dimensiones               | Una (fila o columna)             | Dos (filas y columnas)           |
| Preguntas típicas         | "¿Alineo estos ítems?"           | "¿Cómo reparto el espacio total?" |
| Control del contenido    | El contenido define el tamaño    | El contenedor define la retícula |
| Ejemplo                   | Navbar, botones, tags            | Layout de página, galerías       |

> [!tip] La combinación real
> En la práctica, la mayoría de sitios usan **Grid para el esqueleto** (dónde van las zonas) y **Flexbox dentro** de cada zona (alineación de listas, botones, etc.). No compiten: cada uno resuelve su nivel del diseño.

## ¿Y los floats?

Los `float` fueron el **único** mecanismo de layout multi-columna antes de Flexbox y Grid, y arrastran problemas (el contenedor colapsa, hay que limpiarlos con `clear` o clearfix). Hoy solo se justifican para envolver texto alrededor de imágenes — el caso original delimitado para el que fueron inventados.

> [!warning] Nunca más floats para layout
> Si ves `float: left` en un layout de columnas, es código heredado: reemplazalo por [[Flexbox]] o [[CSS Grid]].

## Relacionadas

- [[Flexbox]] — sintaxis y propiedades de una dimensión
- [[CSS Grid]] — sintaxis y propiedades de dos dimensiones
- [[Responsive Web Design]] — cómo se adaptan ambos modelos a pantallas chicas
- [[Tipos de Display]] — cómo activar cada contexto de formato
