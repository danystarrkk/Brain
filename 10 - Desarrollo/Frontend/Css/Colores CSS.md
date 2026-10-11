---
aliases: ["Colores Css", "Valores de color", "Custom properties de color"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Degradados en CSS]]"
  - "[[Background]]"
  - "[[Animaciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/color_value"
---

# Colores CSS

CSS puede representar un mismo color de varias formas. Elegir el formato correcto afecta a la legibilidad, al tamaño del CSS y a qué tan fácil es mantener el diseño.

## Formatos de color

### 1. Por nombre (keyword)

![[Pasted image 20250124161916.png]]

Son palabras clave reservadas (`blue`, `rebeccapurple`, `tomato`...). Solo hay ~148 colores con nombre y la mayoría no sirven para una paleta de marca. **No se recomienda salvo para `black`, `white` y `transparent`**, donde son claros e innegociables.

### 2. Hexadecimal

![[Pasted image 20250124162011.png]]

Formato `#RGB` (3 dígitos) o `#RRGGBB` (6 dígitos), donde cada par es el valor 00–FF de rojo, verde y azul:

```css
.negro { color: #000000; }
.negro-corto { color: #000; }      /* equivalente a lo anterior */
.acento { color: #3498db; }
```

### 3. Funciones: `rgb()`, `hsl()`, `oklch()`

Además de RGB y HSL (la nota original los listaba como "HLS" — el nombre correcto es **HSL**: *Hue, Saturation, Lightness*), el estándar moderno agrega `oklch()` con mejor percepción:

```css
.rgb  { color: rgb(52 152 219 / 0.8); }   /* con alfa */
.rgba { color: rgba(52, 152, 219, 0.8); } /* sintaxis antigua, todavía válida */
.hsl  { color: hsl(204 70% 53%); }        /* matiz, saturación, luminosidad */
.oklch { color: oklch(0.65 0.17 237); }   /* espacio moderno, gama más amplia */
```

| Formato     | Ventaja                                      | Uso recomendado                          |
| ----------- | -------------------------------------------- | ---------------------------------------- |
| `#hex`      | Compacto, universal                          | Valores fijos conocidos                  |
| `rgb()`     | Cada canal es intuitivo con un picker        | Colores derivados de diseño              |
| `hsl()`     | Fácil variar luz/saturación para tonos       | Paletas, hovers, estados                 |
| `oklch()`   | Gama amplia, tonos consistentes al variar luz | Diseño moderno con propiedades `color()` |

> [!tip] El canal alfa va con `/`
> La sintaxis moderna de todas las funciones acepta el canal alfa con barra: `rgb(0 0 0 / 50%)`. Es más legible que la vieja coma final de `rgba()`.

## Paleta con custom properties

Para crear una paleta reutilizable se define un selector raíz `:root` (el elemento `<html>`, el más alto del árbol) y ahí viven las **custom properties** (variables CSS), que siempre empiezan con doble guion:

```css
:root {
  --blanco: #ffffff;
  --negro: #111827;
  --primario: #3498db;
  --exito: #2ecc71;
  --error: #e74c3c;
}
```

> [!warning] El punto y coma es obligatorio
> Un custom property sin `;` antes de `}` rompe **todas** las declaraciones posteriores del bloque. El ejemplo original de la nota tenía `--blanco: #ffffff` sin punto y coma — corregido arriba.

Luego se consumen con `var()`:

```css
.boton {
  background-color: var(--primario);
  color: var(--blanco);
  border-radius: 6px;
}
.boton:hover {
  background-color: var(--exito);
}
```

> [!info] Custom properties vs colores fijos
> Las variables viven en el navegador y se pueden cambiar en **runtime** (por JS, o por una clase distinta como `.tema-oscuro`). Eso las hace el mecanismo natural para **temas**: definí la paleta en `:root` y redefinila en otro scope.

```css
.tema-oscuro {
  --blanco: #e5e7eb;  /* redefinición: todos los var(--blanco) cambian */
  --negro: #0b1120;
}
```

## Buenas prácticas

- Centralizá los colores como custom properties en lugar de repetir hex sueltos por el CSS.
- Verificá el **contraste** texto/fondo (mínimo 4.5:1 para texto normal) — DevTools lo muestra en el panel de contrast.
- Elegí un sistema: hex para tokens, `hsl()`/`oklch()` para derivados dinámicos.

## Relacionadas

- [[Degradados en CSS]] — combinar colores en degradados
- [[Background]] — colores aplicados a fondos
- [[Animaciones]] — interpolar colores en keyframes
- [[Especificidad]] — cómo ganan las reglas que pintan el color
