---
aliases: ["Iconos Material", "Material Symbols"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Fuentes Externas]]"
  - "[[Html]]"
fuente: "https://fonts.google.com/icons"
---

# Material Icons

Los **Material Icons** (hoy renombrados **Material Symbols**) son el set de iconos oficial de Google, gratuito y de código abierto. La forma clásica de usarlos es como **fuente tipográfica**: cada icono es un glifo de una fuente, y el nombre del icono se escribe como texto. Como cualquier fuente externa, se carga con un `<link>` en el HTML — ver [[Fuentes Externas]].

## 1. Cargar la fuente en el HTML

Agregamos la referencia en el `<head>`, **antes que nuestro CSS**, para no tener conflictos:

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <!-- Fuente de los iconos -->
  <link
    href="https://fonts.googleapis.com/icon?family=Material+Icons"
    rel="stylesheet"
  />
  <link rel="stylesheet" href="css/estilos.css" />
  <title>Material Icons</title>
</head>
```

## 2. Definir la clase base en el CSS

La hoja de Google no trae estilos: solo el glifo. Definimos la clase `material-icons` que configura la fuente y neutraliza propiedades heredadas que distorsionan los glifos (interlineado, espaciado, mayúsculas...):

```css
.material-icons {
  font-family: "Material Icons";
  font-weight: normal;
  font-style: normal;
  font-size: 24px; /* Tamaño preferido de icono */
  display: inline-block;
  line-height: 1;
  text-transform: none;
  letter-spacing: normal;
  word-wrap: normal;
  white-space: nowrap;
  direction: ltr;

  /* Soporte para todos los navegadores WebKit */
  -webkit-font-smoothing: antialiased;
  /* Soporte para Safari y Chrome */
  text-rendering: optimizeLegibility;
  /* Soporte para Firefox */
  -moz-osx-font-smoothing: grayscale;
  /* Soporte para navegadores legacy */
  font-feature-settings: "liga";
}
```

> [!info] ¿Por qué `font-feature-settings: "liga"`?
> Los iconos se activan mediante **ligaduras** (ligatures): el navegador "fusiona" el texto del nombre del icono en un solo glifo. Sin las ligaduras activas, verías el nombre en lugar del icono.

## 3. Usarlos en el HTML

Solo hay que escribir el nombre del icono como contenido del elemento y asignarle la clase `material-icons`. Los nombres se encuentran en la [galería oficial](https://fonts.google.com/icons):

```html
<i class="material-icons">home</i>
<i class="material-icons">search</i>
<i class="material-icons">settings</i>
<span class="material-icons">delete</span>
```

```html
<!-- Con atributos de acceso -->
<i class="material-icons" aria-hidden="true">home</i>
<button>
  <i class="material-icons" aria-hidden="true">delete</i> Eliminar
</button>
```

## Ajustar tamaño y color: son texto

Al ser una fuente, el icono se estila **como cualquier texto**:

```css
.material-icons.md-18 { font-size: 18px; }
.material-icons.md-24 { font-size: 24px; } /* defecto */
.material-icons.md-36 { font-size: 36px; }
.material-icons.md-48 { font-size: 48px; }

.material-icons.rojo { color: #e74c3c; }
.material-icons.alinear-derecha { float: right; }
```

> [!tip] Iconos decorativos: `aria-hidden`
> Si el icono acompaña a un texto visible (como el botón de arriba), marcalo con `aria-hidden="true"` para que los lectores de pantalla no lo deletreen. Si el icono **es** el único contenido del botón, necesitás `aria-label` en el botón y `aria-hidden` en el icono.

## Versión moderna: Material Symbols

Google unificó el set en **Material Symbols**, que además permite elegir **peso, grado y filled** variables:

```html
<link
  href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@24,400,0,0"
  rel="stylesheet"
/>

<i class="material-symbols-outlined">home</i>
```

```css
.material-symbols-outlined {
  font-family: "Material Symbols Outlined";
  font-variation-settings: "FILL" 1, "wght" 500, "GRAD" 0, "opsz" 24;
}
```

| Variable | Efecto                              |
| -------- | ----------------------------------- |
| `FILL`   | 0 = contorno, 1 = relleno           |
| `wght`   | Grosor (100–700)                    |
| `GRAD`   | Gradación óptica                    |
| `opsz`   | Tamaño óptico (16–48)               |

La alternativa a todo lo anterior es servir los iconos como **SVG inline** o sprite: cero peticiones extra y permite `currentColor`, pero pierde la comodidad de cambiar de ícono editando solo el texto.

## Relacionadas

- [[Fuentes Externas]] — el mismo patrón de carga de fuentes externas
- [[Html]] — ubicación correcta de los `<link>` en el documento
