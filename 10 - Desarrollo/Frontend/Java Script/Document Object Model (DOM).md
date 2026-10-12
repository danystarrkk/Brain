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
  - "[[ReactJS - Fundamentos]]"
  - "[[Qué es una API]]"
fuente: "https://developer.mozilla.org/es/docs/Web/API/Document_Object_Model/Introduction"
---

# Document Object Model (DOM)

El **DOM** (*Document Object Model*) es la representación en memoria del documento HTML como un **árbol de objetos** que JavaScript puede leer y modificar. El navegador convierte el texto del HTML en este árbol, y cada etiqueta se convierte en un **nodo**. Una forma gráfica de verlo:

![[Pasted image 20250125081928.png]]

> [!info] El DOM no es el HTML
> El HTML es un archivo de texto; el DOM es una **estructura de datos** que el navegador construye a partir de ese texto. JavaScript no edita el archivo: edita el árbol, y el navegador vuelve a pintar la página. También puede existir un DOM para XML o para documentos que no son HTML. Ese mismo documento puede tener **varias representaciones simultáneas**: el HTML fuente (el texto), el DOM (árbol de nodos) y el CSSOM (árbol de estilos computados) — el navegador las cruza para pintar.

### `document` vs `window`

Dos objetos que se usan juntos todo el tiempo y conviene separar:

| Objeto | Qué representa | Ejemplos |
| ------ | -------------- | -------- |
| `window` | **La ventana/pestaña**: el contexto global del navegador (en navegador, todo objeto global cuelga de `window`) | `window.scrollY`, `window.innerWidth`, `window.addEventListener('resize')`, `window.localStorage` |
| `document` | **El documento** dentro de esa ventana: el árbol del DOM | `document.querySelector`, `document.body`, `document.title` |

`window.document` es el mismo objeto que el global `document`. Regla mnemotécnica: si se refiere a **la página en la ventana** (tamaño, scroll, historial) es `window`; si se refiere al **árbol de nodos** es `document`.

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

### Tipos de nodo

El `nodeType` numérico identifica cada tipo; `nodeName` da su nombre (los elementos usan mayúsculas HTML; el documento usa `"#document"`):

| `nodeType` | Constante | `nodeName` | Ejemplo |
| ---------- | --------- | ---------- | ------- |
| `1` | `ELEMENT_NODE` | el tag en MAYÚSCULAS (`DIV`) | `<div>`, `<p>` |
| `2` | `ATTRIBUTE_NODE` | el nombre del atributo | `class="card"` (accesible hoy vía `attributes`) |
| `3` | `TEXT_NODE` | `"#text"` | el texto entre etiquetas, **incluidos los saltos de línea** |
| `8` | `COMMENT_NODE` | `"#comment"` | `<!-- nota -->` |
| `9` | `DOCUMENT_NODE` | `"#document"` | el objeto `document` raíz |
| `10` | `DOCUMENT_TYPE_NODE` | `"html"` | la doctype `<!DOCTYPE html>` |
| `11` | `DOCUMENT_FRAGMENT_NODE` | `"#document-fragment"` | `DocumentFragment` (contenedor ligero) |

```js
const nodo = document.querySelector('h1');
console.log(nodo.nodeType); // 1 — ELEMENT_NODE
console.log(nodo.nodeName); // "H1" — el tag en mayúsculas
// etiqueta propia del elemento (solo existe en elementos HTML):
console.log(nodo.tagName);  // "H1" — misma información que nodeName en elementos
```

### Relaciones familiares: nodos vs elementos

Relaciones familiares que usa el DOM:

- **Padre** (`parentNode` / `parentElement`): el nodo que contiene a otro.
- **Hijos** (`children`): los nodos contenidos. `children` lista solo elementos; `childNodes` incluye también textos y saltos de línea.
- **Hermanos** (`nextElementSibling` / `previousElementSibling`): nodos al mismo nivel.

Tabla completa de la familia — la distinción clave es **nodo** (cualquier tipo) vs **elemento** (solo `nodeType 1`):

| Camino | Versión "todo nodo" | Versión "solo elementos" |
| ------ | ------------------- | ------------------------ |
| Hijos | `childNodes` (NodeList viva) | `children` (HTMLCollection viva) |
| Padre | `parentNode` | `parentElement` (null si el padre es el documento) |
| Primer hijo | `firstChild` | `firstElementChild` |
| Último hijo | `lastChild` | `lastElementChild` |
| Siguiente hermano | `nextSibling` | `nextElementSibling` |
| Anterior hermano | `previousSibling` | `previousElementSibling` |

> [!warning] `childNodes` ve "fantasmas"
> Entre las etiquetas hay nodos de texto con los saltos de línea y la indentación. `div.childNodes` los lista como `#text`; `div.children` los ignora. Por eso el código casi siempre quiere las versiones de *elemento*. Si necesitás los textos reales (p. ej. para resaltar palabras), ahí `childNodes` es el correcto.

## El API de `document`

`document` es el objeto raíz y la puerta de entrada para todo:

```js
document.body;   // el <body>
document.head;   // el <head>
document.title;  // el título de la pestaña (escritura = cambiar la pestaña)
document.documentElement; // el <html>
document.location;        // URL actual (alias de window.location)
document.cookie;          // cookies como string (uso legacy; preferir APIs dedicadas)
```

Las tres operaciones fundamentales sobre el DOM son:

1. **Seleccionar** — `document.querySelector('.card')`, `getElementById`, `querySelectorAll`.
2. **Modificar** — contenido (`textContent`, `innerHTML`), atributos (`src`, `href`), estilos (`style`) y clases (`classList`).
3. **Crear/eliminar** — `createElement`, `appendChild`, `remove`.

Ver [[Acceder a Html]] para el detalle de cada una con ejemplos.

> [!warning] El DOM debe existir antes de tocarlo
> Si el script corre en el `<head>` sin `defer`, `querySelector` retorna `null` porque el navegador aún no construyó ese sector del árbol. Ver [[Java Script]] para las opciones de carga del `<script>`, y [[Eventos]] para el ciclo `DOMContentLoaded`/`load`.

## Parsing de HTML: tres formas de materializar nodos

Insertar HTML crudo en el árbol tiene tres rutas, con compromisos distintos:

| Método | Cómo funciona | Velocidad | Seguridad | Cuándo |
| ------ | ------------- | --------- | --------- | ------ |
| `document.createElement('p')` + propiedades | Construye nodos uno a uno vía API | Rápida en pocos nodos; sin parseo de texto | **Segura**: no interpreta HTML | Nodos sueltos, datos de usuario |
| `el.innerHTML = '<b>x</b>'` | Re-parsea el string **dentro** del subárbol: destruye y recrea los hijos | Cara en DOMs grandes (reescribe todo el subárbol) | **Peligrosa** con datos externos (XSS) | Plantillas confiables, una sola vez al cargar |
| `new DOMParser().parseFromString(str, 'text/html')` | Crea un **documento independiente** en memoria | Costosa pero aislada | No ejecuta scripts hasta insertar | Parsear fragmentos sin tocar el DOM vivo |

```js
// 1) API de nodos (recomendada)
const p = document.createElement('p');
p.textContent = 'hola';
document.body.append(p);

// 2) innerHTML — solo con HTML propio, jamás con datos de usuario
document.querySelector('.caja').innerHTML = '<b>hola</b>';

// 3) DOMParser — documento aislado que luego puedes importar
const doc = new DOMParser().parseFromString('<p>hola</p>', 'text/html');
const pImportado = document.importNode(doc.body.firstElementChild, true);
document.body.append(pImportado);
```

> [!warning] `innerHTML` es la puerta XSS clásica
> Si el string contiene datos del usuario (input, URL, respuesta de API), un `<img src=x onerror="...">` se ejecuta al insertarse. Regla: **texto → `textContent`; HTML dinámico → sanitizar o construir con `createElement`**. Detalle en [[Acceder a Html]].

## Recorrer el árbol

```js
// hacia arriba
elemento.parentElement;
elemento.closest('.card');      // ancestro más cercano que cumple el selector (o null)
elemento.closest('.card')?.classList.add('resaltado');

// hacia abajo y al lado
elemento.children;
elemento.firstElementChild;
elemento.nextElementSibling;

// preguntas
elemento.matches('.card.activa'); // ¿este elemento cumple el selector?
document.body.contains(elemento); // ¿está contenido en...?
```

`closest` y `matches` usan **selectores CSS** completos (clases, atributos, `:nth-child`…) y son la base de la delegación de eventos (ver [[Eventos]]). `parentElement.parentElement.parentElement` funciona pero `closest` comunica mejor la intención ("el ancestro `.card` más cercano").

## Mutar el árbol

Resumen de la API de inserción; el detalle con ejemplos vive en [[Acceder a Html]]:

| Operación | Método |
| --------- | ------ |
| Agregar al final | `padre.append(...nodos)` (acepta texto y varios) / `appendChild(nodo)` (solo un nodo) |
| Agregar al inicio | `padre.prepend(...nodos)` |
| Insertar antes/después de un nodo | `hermano.before(nodo)` / `hermano.after(nodo)` |
| Reemplazar | `padre.replaceChildren(...nodos)` / `nodo.replaceWith(otro)` |
| Eliminar | `nodo.remove()` / `padre.removeChild(nodo)` |
| Clonar | `nodo.cloneNode(true)` (deep) / `false` (solo el elemento) |
| Lote atómico | `DocumentFragment`: se llenan off-screen y se inserta de una |

> [!tip] `DocumentFragment` para inserts en lote
> Cada inserción individual invalida el layout. Un `DocumentFragment` es un nodo contenedor **efímero**: agregás todos los hijos ahí (sin tocar el árbol vivo) y al hacer `contenedor.append(fragment)` se mueven todos de una sola pasada. Es la forma barata de insertar 100 nodos sin 100 re-pintados.

## Atributos

Cada elemento expone sus atributos HTML como puente entre el marcado y el objeto:

```js
const img = document.querySelector('img');
img.getAttribute('src');   // valor textual del atributo
img.setAttribute('alt', 'foto');
img.removeAttribute('loading');
img.hasAttribute('lazy');

// colección viva de atributos del elemento
for (const attr of img.attributes) console.log(attr.name, attr.value);

// data-* accesible vía dataset (ver [[Acceder a Html]])
img.dataset.idProducto;    // ← <img data-id-producto="7">
```

Atributos vs propiedades: los **atributos** viven en el marcado (`getAttribute`), las **propiedades** son las claves del objeto elemento (`img.src`). Suelen duplicarse, pero no siempre coinciden: `getAttribute('checked')` refleja el HTML inicial, mientras que la propiedad `input.checked` refleja el estado actual del usuario.

## CSSOM: el puente con los estilos

El navegador construye además el **CSSOM** (*CSS Object Model*), el árbol de reglas CSS parseadas. JavaScript lo toca por tres vías:

```js
// 1) clases (recomendado — la lógica de estilos vive en el CSS)
el.classList.add('activo');

// 2) estilos en línea (camelCase: background-color → backgroundColor)
el.style.backgroundColor = 'red';
el.style.cssText = 'margin: 0; padding: 1rem';  // varios de golpe

// 3) estilo COMPUTADO: qué aplica realmente tras cascada, herencia y media queries
const cs = getComputedStyle(el);
console.log(cs.color, cs.display, cs.getPropertyValue('--tema'));
```

| Fuente | Qué refleja | Rendimiento |
| ------ | ----------- | ----------- |
| `el.style` | Solo estilos **en línea** del elemento | Rápida (lectura de objeto) |
| `getComputedStyle(el)` | El valor **final** aplicado (cascada + herencia + media query) | Cara: fuerza resolución de estilos |
| `el.classList` | Solo el token de clase | Rápida |

`getComputedStyle` es la única forma de leer valores que vienen del CSS (p. ej. `--variables` o propiedades de media queries); leer `el.style.width` devuelve vacío si el ancho viene de una hoja de estilo.

## El módulo Events

El DOM es grande; el estándar lo divide en módulos:

| Módulo | Para qué sirve |
| ------ | -------------- |
| **Core** (`Node`, `Document`) | estructura del árbol, nodos, selección |
| **Events** | `addEventListener`, `click`, `input`... ver [[Eventos]] |
| **HTML** (`HTMLElement`) | propiedades específicas de las etiquetas HTML |
| **CSSOM** | leer/aplicar estilos: `getComputedStyle`, `style` |
| **XMLHttpRequest / Fetch** | comunicación con servidores — ver [[Qué es una API]] |

Cada módulo extiende a los anteriores: un `<div>` es un `HTMLElement` (módulo HTML), que es un `Element`, que es un `Node` (Core), que recibe eventos (Events).

## Shadow DOM: el árbol oculto

Dos conceptos que suelen confundirse con el DOM:

- **`innerHTML`**: getter/setter que trabaja con el HTML como texto. Útil para insertar bloques, pero peligroso con datos de usuario (XSS).
- **Shadow DOM**: árbol oculto que usan los *web components* (custom elements). No aparece en el DOM "normal" y permite encapsular estilos y comportamiento. Los nodos del shadow root no se alcanzan con `querySelector` del documento — se acceden por `element.shadowRoot`. Inspeccionar en DevTools: en la pestaña Elements se despliega como `#shadow-root`.

## DOM "virtual": cómo lo abstrae React

React (y Svelte, Vue…) no manipulan el DOM por nodo: mantienen una **copia ligera del árbol en memoria** (virtual DOM / VDOM), la comparan con la anterior (*diffing*) y aplican al DOM real solo el **mínimo de cambios** (*reconciliación*). Motivo: las operaciones de nodos disparan layout/pintado, y escribir en lote es más barato que escribir 50 veces y dejar que el navegador recalcule. Ver [[ReactJS - Fundamentos]] y, para el ciclo de limpieza de efectos sobre nodos, [[ReactJS - Efectos (useEffect)]]. Esta nota describe el DOM **nativo**, el que React termina tocando.

## Rendimiento del DOM

El navegador pinta en fases: *style* → *layout* (reflow: calcular geometría) → *paint* → *composite*. Leer geometría (`offsetHeight`, `getBoundingClientRect`) **fuerza** el layout si hay cambios de estilo pendientes — el patrón vicioso *layout thrashing*:

```js
// ❌ MAL: intercala escrituras y lecturas — el navegador recalcula en cada vuelta
const items = document.querySelectorAll('.item');
items.forEach((el) => {
  el.style.width = el.offsetHeight + 'px'; // reflow por iteración
});

// ✅ BIEN: primero TODAS las lecturas, después TODAS las escrituras
const alturas = items.map((el) => el.offsetHeight);
items.forEach((el, i) => { el.style.width = alturas[i] + 'px'; });

// ✅ MEJOR: agrupar la escritura en el siguiente frame
requestAnimationFrame(() => items.forEach((el, i) => { el.style.width = alturas[i] + 'px'; }));
```

Reglas de oro:

1. **Lee todo, escribe todo** (nunca alternes).
2. Agrupa escrituras con `requestAnimationFrame` para tocar el DOM justo antes del pintado.
3. Reduce el **tamaño del subárbol** que mutas: `DocumentFragment` e inserciones únicas en vez de N.
4. `innerHTML` destruye y recrea todo el subárbol — pierde foco, estado y listeners; solo para rebuilds totales.
5. Para mostrar/ocultar, prefiere clases (`classList.toggle`) sobre estilos en línea: una sola propiedad invalidada.
6. Animaciones: `transform`/`opacity` van por composición sin layout — ver [[Acceder a Html]] y guías de animación CSS.

## Errores comunes

1. **Tocar el DOM antes de que exista** → `TypeError: Cannot read properties of null`. Síntoma clásico: `document.querySelector` devuelve `null` porque el script corre en el `<head>` sin `defer`. Fix: `defer`, fin del `<body>`, o `DOMContentLoaded` (ver [[Java Script]]).
2. **Iterar `children` como array**: es una `HTMLCollection` viva sin `map`/`filter`. `[...el.children].filter(...)` o `querySelectorAll`.
3. **Buscar `#id` duplicado**: `getElementById` devuelve el primero (o ninguno); los IDs deben ser únicos — el HTML válido exige una sola definición.
4. **`childNodes` donde esperabas elementos**: los saltos de línea aparecen como `#text` y descuadran índices. Usá `children`.
5. **Leer estilos del CSS vía `el.style`**: solo ve lo inline. Para el valor real: `getComputedStyle`.
6. **Mezclar lecturas y escrituras en loops**: layout thrashing — el rendimiento se desploma. Separa fases.
7. **Esperar que el DOM de shadow root responda a `document.querySelector`**: los web components encapsulan su árbol interno.
8. **Persistir referencias a nodos eliminados**: si conservás un nodo tras `remove()`, sus listeners siguen vivos en memoria (fuga) aunque no esté en pantalla — soltar referencias o usar `AbortController` (ver [[Eventos]]).

> [!tip] Inspeccionar el DOM en DevTools
> En Chrome/Firefox, la pestaña **Elements** muestra el árbol del DOM *en vivo*: al cambiar clases o texto con JavaScript, el árbol se actualiza en tiempo real. También puedes editar nodos desde ahí mismo. La pestaña **Performance** graba reflows y repaints para detectar layout thrashing; **Console** acepta `console.table` y `$0` (elemento seleccionado).

## Cheatsheet — API del DOM

| Concepto | API | Detalle |
| -------- | --- | ------- |
| Raíz | `document`, `document.documentElement` | `document` cuelga de `window` |
| Seleccionar | `querySelector(All)`, `getElementById` | detalle en [[Acceder a Html]] |
| Tipo de nodo | `nodeType`, `nodeName`, `tagName` | 1 = elemento; tagName solo existe en elementos |
| Familia | `parentElement`, `children`, `siblings` | versiones `*Element*` = solo elementos |
| Crear | `createElement`, `cloneNode`, `createDocumentFragment` | fragment = inserts en lote |
| Insertar | `append`, `prepend`, `before`, `after`, `replaceWith` | ver [[Acceder a Html]] |
| Eliminar | `remove`, `removeChild` | `remove` es más limpio |
| Atributos | `getAttribute/setAttribute/hasAttribute/removeAttribute`, `dataset` | atributo ≠ propiedad |
| Estilos | `classList`, `style`, `getComputedStyle` | `style` solo inline |
| Eventos | `addEventListener` | ver [[Eventos]] |
| Parsear HTML | `DOMParser`, `innerHTML` | innerHTML = riesgo XSS |

## Relacionadas

- [[Acceder a Html]] — seleccionar, modificar, crear y eliminar nodos con ejemplos
- [[Eventos]] — el módulo de eventos del DOM: flujo, delegación, catálogo
- [[Html]] — el lenguaje que el DOM representa
- [[Etiquetas]] — los elementos que se vuelven nodos
- [[Java Script]] — cómo y cuándo se carga el script
- [[ReactJS - Fundamentos]] — React trabaja sobre una abstracción del DOM (Virtual DOM)
- [[ReactJS - Efectos (useEffect)]] — tocar nodos reales dentro del ciclo de React
- [[Qué es una API]] — el DOM como el mayor API del navegador
