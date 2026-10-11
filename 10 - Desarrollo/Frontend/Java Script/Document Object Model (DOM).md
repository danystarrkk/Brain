---
aliases: ["DOM", "Document Object Model"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Acceder a Html]]"
  - "[[Eventos]]"
  - "[[Html]]"
  - "[[Java Script]]"
fuente: "https://developer.mozilla.org/es/docs/Web/API/Document_Object_Model/Introduction"
---

# Document Object Model (DOM)

El **DOM** (*Document Object Model*) es la representación en memoria del documento HTML como un **árbol de objetos** que JavaScript puede leer y modificar. El navegador convierte el texto del HTML en este árbol, y cada etiqueta se convierte en un **nodo**. Una forma gráfica de verlo:

![[Pasted image 20250125081928.png]]

> [!info] El DOM no es el HTML
> El HTML es un archivo de texto; el DOM es una **estructura de datos** que el navegador construye a partir de ese texto. JavaScript no edita el archivo: edita el árbol, y el navegador vuelve a pintar la página. También puede existir un DOM para XML o para documentos que no son HTML.

## El árbol de nodos

Cada etiqueta HTML es un **nodo elemento**; el texto dentro de ellos es un nodo de texto; los comentarios, un nodo comentario. El nodo raíz es `document`, y dentro está `html` (el raíz del contenido), que cuelga de `head` y `body`:

```
document
└── html
    ├── head
    │   ├── title
    │   └── meta
    └── body
        ├── h1
        └── div.card
            ├── img
            └── div.info
                └── p.titulo
```

Relaciones familiares que usa el DOM:

- **Padre** (`parentNode` / `parentElement`): el nodo que contiene a otro.
- **Hijos** (`children`): los nodos contenidos. `children` lista solo elementos; `childNodes` incluye también textos y saltos de línea.
- **Hermanos** (`nextElementSibling` / `previousElementSibling`): nodos al mismo nivel.

## El API de `document`

`document` es el objeto raíz y la puerta de entrada para todo:

```js
document.body;   // el <body>
document.head;   // el <head>
document.title;  // el título de la pestaña
```

Las tres operaciones fundamentales sobre el DOM son:

1. **Seleccionar** — `document.querySelector('.card')`, `getElementById`, `querySelectorAll`.
2. **Modificar** — contenido (`textContent`, `innerHTML`), atributos (`src`, `href`), estilos (`style`) y clases (`classList`).
3. **Crear/eliminar** — `createElement`, `appendChild`, `remove`.

Ver [[Acceder a Html]] para el detalle de cada una con ejemplos.

> [!warning] El DOM debe existir antes de tocarlo
> Si el script corre en el `<head>` sin `defer`, `querySelector` retorna `null` porque el navegador aún no construyó ese sector del árbol. Ver [[Java Script]] para las opciones de carga del `<script>`.

## La API `innerHTML` y el *shadow DOM*

Dos conceptos que suelen confundirse con el DOM:

- **`innerHTML`**: getter/setter que trabaja con el HTML como texto. Útil para insertar bloques, pero peligroso con datos de usuario (XSS).
- **Shadow DOM**: árbol oculto que usan los *web components* (custom elements). No aparece en el DOM "normal" y permite encapsular estilos y comportamiento.

## Módulos relacionados con el DOM

El DOM es grande; el estándar lo divide en módulos:

| Módulo | Para qué sirve |
| ------ | -------------- |
| **Core** (`Node`, `Document`) | estructura del árbol, nodos, selección |
| **Events** | `addEventListener`, `click`, `input`... ver [[Eventos]] |
| **HTML** (`HTMLElement`) | propiedades específicas de las etiquetas HTML |
| **CSSOM** | leer/aplicar estilos: `getComputedStyle`, `style` |
| **XMLHttpRequest / Fetch** | comunicación con servidores — ver [[Qué es una API]] |

> [!tip] Inspeccionar el DOM en DevTools
> En Chrome/Firefox, la pestaña **Elements** muestra el árbol del DOM *en vivo*: al cambiar clases o texto con JavaScript, el árbol se actualiza en tiempo real. También puedes editar nodos desde ahí mismo.

## Relacionadas

- [[Acceder a Html]] — seleccionar, modificar, crear y eliminar nodos
- [[Eventos]] — el módulo de eventos del DOM
- [[Html]] — el lenguaje que el DOM representa
- [[Java Script]] — cómo y cuándo se carga el script
- [[ReactJS - Fundamentos]] — React trabaja sobre una abstracción del DOM (Virtual DOM)
