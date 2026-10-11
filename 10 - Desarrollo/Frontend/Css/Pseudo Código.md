---
aliases: ["Pseudo-clases", "Selectores pseudo"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Selectores en CSS]]"
  - "[[Especificidad]]"
  - "[[Anatomía de CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Learn/CSS/Building_blocks/Selectors/Pseudo-classes_and_pseudo-elements"
---

# Pseudo Código

Los selectores **pseudo** ("pseudocódigo" de CSS) seleccionan elementos **sin necesidad de clases ni IDs extra en el HTML**: se definen en el momento y según el estado o la utilidad que se necesite. Se dividen en dos familias que se distinguen por la cantidad de dos puntos:

| Familia          | Sintaxis  | Selecciona                                            |
| ---------------- | --------- | ----------------------------------------------------- |
| **Pseudo-clase** | `:estado` | Un elemento en un **estado** o de un **tipo** concreto |
| **Pseudo-elemento** | `::parte` | Una **parte virtual** del elemento (no existe en el HTML) |

## Pseudo-clases: un solo `:`

Estilizan estados dinámicos o categorías de elementos. El ejemplo clásico: un enlace que cambia **al haber entrado ya una vez** o al recibir foco:

![[Pasted image 20250124164638.png]]

En la imagen, `.navegacion` es la clase contenedora, `a` el enlace dentro, y con `:focus` definimos la pseudo-clase. Dentro de una pseudo-clase podemos declarar **tantas propiedades como haga falta**: solo es cuestión de acomodar todo a gusto.

```css
.navegacion a {
  color: #333;
}

.navegacion a:hover {
  color: #0066cc; /* el mouse está sobre el enlace */
}

.navegacion a:focus {
  color: #e67e22; /* el enlace tiene foco de teclado */
  outline: 2px solid #e67e22;
}

.navegacion a:visited {
  color: #7f5af0; /* enlace ya visitado */
}
```

### Las pseudo-clases de estado más usadas

| Pseudo-clase     | Significado                                   |
| ---------------- | --------------------------------------------- |
| `:hover`         | Cursor sobre el elemento                      |
| `:focus`         | Elemento enfocado (teclado/click)             |
| `:focus-visible` | Focalizado **solo por teclado** (mejor UX: no muestra outline al click) |
| `:focus-within`  | El elemento o alguno de sus hijos tiene foco  |
| `:visited`       | Enlace ya visitado (solo aplica a `<a>`)      |
| `:checked`       | Checkbox/radio marcado                        |
| `:disabled`      | Control deshabilitado                         |
| `:first-child`, `:last-child`, `:nth-child(n)` | Posición entre hermanos      |
| `:not(selector)` | Todo lo que **no** coincide con el selector   |
| `:is()` / `:where()` | Agrupar selectores con reglas de especificidad distintas |

```css
/* Filas alternas de una tabla */
.tabla tbody tr:nth-child(even) {
  background: #f5f5f5;
}

/* Todo ítem menos el primero */
.menu li:not(:first-child) {
  margin-left: 1rem;
}

/* El contenedor se resalta cuando adentro hay un input inválido */
.formulario:has(input:invalid) {
  border-color: red;
}
```

> [!tip] `:hover` no existe en pantallas táctiles
> En móvil no hay cursor: lo que se estiliza con `:hover` se activa con el toque y "se queda pegado". Duplicá esos estilos con `:focus-visible` o diseñá estados que no escondan información.

## Pseudo-elementos: dos puntos dobles `::`

No seleccionan un estado, sino una **parte inexistente en el HTML** del elemento. El ejemplo rey: contenido decorativo antes/después del elemento:

```css
.advertencia::before {
  content: "⚠ ";
  font-size: 1.2em;
}

.enlace-exterior::after {
  content: " ↗";           /* añade un glifo al final del enlace */
  font-size: 0.85em;
}

input::placeholder {
  color: #999;             /* el texto de ayuda del input */
}

.parrafo::first-letter {
  font-size: 3rem;         /* capitular */
  float: left;
}
```

> [!warning] `content` solo existe en pseudo-elementos
> `::before` y `::after` **ignoran sus estilos si no tienen `content`** (aunque sea vacío `""`). Y ese contenido es decorativo: los lectores de pantalla no lo leen por defecto, así que nunca pongas ahí información importante.

## Especificidad

Las pseudo-clases se puntúan como **clases** (0–1–0) y los pseudo-elementos como **elementos** (0–0–1) — detalle completo en [[Especificidad]].

## Relacionadas

- [[Selectores en CSS]] — el resto de selectores y cómo se combinan
- [[Especificidad]] — cómo puntúa el navegador estos selectores
- [[Anatomía de CSS]] — la estructura de la regla donde viven
