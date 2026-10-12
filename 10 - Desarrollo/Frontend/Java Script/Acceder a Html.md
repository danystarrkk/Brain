---
tags:
  - desarrollo
  - frontend
  - acceder-html
tipo: concepto
estado: terminado
---

# Acceder a HTML con JavaScript

Una vez que el navegador parsea el HTML, construye el **[[Document Object Model (DOM)|DOM]]**: un árbol de objetos JavaScript que representa la página. JavaScript no "edita el archivo HTML": **edita ese árbol**, y el navegador vuelve a pintar la página. Esta nota cubre las tres grandes tareas: **seleccionar**, **modificar** y **crear/eliminar** elementos.

> [!info] El script debe correr después del DOM
> Si el código usa `document.querySelector(...)` y el script carga en el `<head>`, el elemento todavía no existe y el resultado es `null`. Coloca el `<script>` antes del cierre de `</body>` o usa `defer` — ver [[Java Script]].

## El objeto `document`

`document` es la puerta de entrada al DOM y expone atajos muy útiles:

```js
document;            // todo el documento
document.head;       // el <head>
document.body;       // el <body>
document.domain;     // el dominio actual
document.all;        // todos los elementos (legacy, no usar en código nuevo)

document.forms;      // HTMLCollection con todos los <form>
document.forms[0];           // se accede como si fuera un arreglo
document.forms[0].id;        // id del formulario
document.forms[0].method;    // GET o POST
document.forms[0].action;    // a dónde se envía
document.forms[0].classList; // clases del formulario

document.links;      // todos los <a>
document.links[4].classList; // clases de un enlace específico
document.links[4].className; // la clase como string

document.images;     // todas las <img>
document.scripts;    // todos los <script>
```

Estas colecciones (`forms`, `links`, `images`, `scripts`) son `HTMLCollection` **vivas** — ver "Errores comunes" más abajo.

> [!warning] `document.all` es legacy
> Existe por compatibilidad, pero `typeof document.all` devuelve `"undefined"` (una peculiaridad histórica para no romper código que lo chequeaba como falsy). Usa `querySelectorAll` en su lugar.

## Seleccionar elementos

Hay dos generaciones de métodos. Los antiguos (`getElementById`, `getElementsBy*`) siguen siendo válidos, pero hoy se prefiere `querySelector`/`querySelectorAll`, que aceptan **cualquier selector CSS**.

### Tabla exhaustiva de selección

| Método | Selector | ¿Devuelve vivo? | ¿Lanza error? | Notas |
| ------ | -------- | ---------------- | ------------- | ----- |
| `getElementById('id')` | solo el id (sin `#`) | — | No (`null`) | El más rápido de todos; ID único |
| `getElementsByClassName('c')` | clase, **sin punto**; varias separadas por espacio = todas deben estar | **Sí** (HTMLCollection) | No (colección vacía) | Solo clases, no selectores compuestos |
| `getElementsByTagName('p')` | tag, **sin** `<` `>` | **Sí** (HTMLCollection) | No | `'*'` = todos los elementos |
| `getElementsByName('q')` | atributo `name` | **Sí** (NodeList) | No | Solo útil en formularios legacy |
| `querySelector('.sel')` | **cualquier selector CSS** | No (snapshot) | No (`null`) | Primera coincidencia en orden del documento |
| `querySelectorAll('.sel')` | **cualquier selector CSS** | No (NodeList estática) | No (NodeList vacía) | Coincidencias en orden del documento |
| `element.querySelector(sel)` | selector **restringido a los descendientes** | No | No | Equivale a buscar dentro del subárbol |

```js
// Forma antigua
const header = document.getElementsByClassName('header');
const hero = document.getElementsByClassName('hero');
const contenedores = document.getElementsByClassName('contenedor');

// Si la clase no existe: colección vacía (no null)
const noExiste = document.getElementsByClassName('no-existe');

// Por id: retorna UN elemento (o null); si hay varios ids iguales, el primero
const formulario = document.getElementById('formulario');
const formularioNoExiste = document.getElementById('no-exite'); // null

// Forma actual: cualquier selector CSS
const card = document.querySelector('.card');
const info = document.querySelector('.premium .info');
const navegacion = document.querySelector('nav');

// Selectores complejos de CSS funcionan igual
const segundoCard = document.querySelector('section.hospedaje .card:nth-child(2)');
const formularioComplejo = document.querySelector('.contenido-hero #formulario');

// Todas las coincidencias: retorna un NodeList
const cards = document.querySelectorAll('.card');

// selección acotada a un subárbol (más rápida que buscar en todo el documento)
const item = navegacion.querySelector('.activo');
```

> [!tip] Una diferencia clave: vivas vs snapshot
> `HTMLCollection` es **viva**: si agregas un elemento con esa clase, la colección crece sola. `NodeList` de `querySelectorAll` es **estática**: no refleja cambios posteriores. Por eso `cards[0]` puede ser `undefined` si la lista quedó vacía: siempre verifica antes de usar. Para iterar, `querySelectorAll` da un snapshot predecible — `[...nodelist].map(...)` — mientras que con colecciones vivas conviene convertir a array **al momento de recorrer** (ver [[Iteradores]]).
>
> Excepción viva: `element.children` y `element.classList` también se actualizan solos; `querySelectorAll` nunca.

### Especificidad y rendimiento de la selección

- **Velocidad relativa**: `getElementById` (índice interno por id) > `querySelector` con selector simple > `querySelectorAll` con selectores complejos > `getElementsBy*` en documentos enormes (la colección viva tiene costo de mantenimiento). En la práctica, todas son rápidas; el costo real aparece al llamarlas **dentro de bucles o handlers de alta frecuencia** — cachea la referencia.
- **`querySelector` toma la primera coincidencia en orden del documento**, no la más específica según CSS. `querySelector('div .x')` no aplica reglas de especificidad: solo busca.
- Selectores válidos: cualquier CSS moderno — combinadores (`>`, `+`, `~`), pseudoclases (`:nth-child`, `:not()`, `:checked`, `:focus`), atributos (`[href^="https"]`). Lo que no sirve: pseudoelementos (`::before`) — no son elementos.

### `closest` y `matches`: preguntas al árbol

```js
const enlace = document.querySelector('a.descarga');

// ¿El elemento (o un ancestro) cumple el selector?
const boton = enlace.closest('button');          // ancestro más cercano <button> (o null)
console.log(enlace.matches('a[href$=".pdf"]'));   // true — el propio elemento cumple
console.log(boton?.contains(enlace));            // true — jerarquía
```

| Método | Pregunta que responde | Recorre |
| ------- | --------------------- | ------- |
| `matches(sel)` | "¿**Este** elemento cumple el selector?" | Ninguno |
| `closest(sel)` | "¿Yo o algún ancestro cumple?" | hacia arriba hasta `document` |
| `parentElement.closest(sel)` | Lo mismo pero saltándose el propio | hacia arriba |

`closest` es la pieza central de la delegación de eventos (ver [[Eventos]]): el click llega al hijo más profundo, pero `evt.target.closest('.card')` identifica la tarjeta completa.

## Dataset: atributos `data-*`

Los atributos `data-*` son el canal oficial para pegar datos de la aplicación al marcado:

```html
<button class="btn-agregar" data-id-producto="42" data-categoria="monitores">
  Agregar al carrito
</button>
```

```js
const btn = document.querySelector('.btn-agregar');

// data-id-producto → dataset.idProducto (conversión a camelCase)
console.log(btn.dataset.idProducto);   // "42" — ¡string siempre!
btn.dataset.precio = "300";            // escribe data-precio="300"

// leer todos los datos de una vez
console.log(btn.dataset);              // DOMStringMap { idProducto: "42", ... }

// uso real: identificar el origen en delegación
btn.closest('ul').addEventListener('click', (evt) => {
  const id = evt.target.dataset.idProducto;
  if (id) agregarAlCarrito(Number(id)); // ¡convertir el tipo!
});
```

> [!warning] `dataset` entrega strings
> Todo valor de `data-*` es texto: `dataset.contador = 5` guarda `"5"`. Convertí explícitamente (`Number`, `JSON.parse`) antes de comparar o calcular.

## Clases: `classList` y `className`

```js
const card = document.querySelector('.card');

card.classList.add('nueva-clase', 'segunda-clase'); // agrega una o varias
card.classList.remove('nueva-clase');               // elimina una o varias
card.classList.toggle('activa');                    // agrega si falta, quita si existe
card.classList.toggle('activa', true);              // segundo arg = forzar estado
card.classList.contains('activa');                  // verifica (true/false)
card.classList.replace('vieja', 'nueva');           // renombra
console.log(card.className);                        // la clase COMPLETA como string
console.log(card.classList.length);                 // número de clases
```

Escribir todo el CSS desde JavaScript deja el código gigante e inmantenible. La práctica recomendada es **agregar y quitar clases** y dejar el estilo en el CSS:

```js
// ❌ estilo imperativo disperso
card.style.backgroundColor = '#ff4444';
card.style.transform = 'scale(1.02)';
// ✅ intención como clase; el CSS decide cómo se ve
card.classList.add('resaltado');
```

`className` reemplaza **toda** la cadena de clases (sobrescribe) — solo se usa en SVG o código legacy; `classList` es la API correcta.

## Leer y modificar contenido

```js
const encabezado = document.querySelector('.contenido-hero h1');

console.log(encabezado.innerText);   // solo el texto visible (oculta lo que esté con CSS)
console.log(encabezado.textContent); // todo el texto, aunque esté oculto
console.log(encabezado.innerHTML);   // el HTML interno completo

// Modificar
encabezado.textContent = 'Nuevo heading';

// Acceso encadenado (con optional chaining para no explotar si algo falta)
const texto = document.querySelector('.contenido-hero h1')?.textContent;
```

| Propiedad | Incluye etiquetas HTML | Respeta estilos/visibility | Rendimiento |
| --------- | ---------------------- | --------------------------- | ----------- |
| `innerText` | No | **Sí** (solo lo visible) | **Cara**: fuerza reflow para calcular lo que se ve |
| `textContent` | No | No (todo el texto) | Rápida (solo recorre nodos de texto) |
| `innerHTML` | **Sí** (getter y setter) | No | Setter cara: re-parsea y reconstruye el subárbol |

> [!warning] Cuidado con `innerHTML` con datos de usuario
> Asignar HTML desde entradas no sanitizadas (formularios, APIs) es la puerta clásica a ataques **XSS**: `<img src=x onerror="robacion()">` se ejecuta al insertarse. Para texto plano usa `textContent`; solo usa `innerHTML` con HTML que tú mismo construiste.

Regla de decisión: **mostrar datos del usuario → `textContent`**; leer lo que se ve a ojo de gato (p. ej. para scraping de widgets) → `innerText`; insertar plantilla propia ya construida → `innerHTML` (una sola vez, no en loops).

## Atributos vs propiedades

| | Atributo (`data-*`, `src`, `class`...) | Propiedad (`el.src`, `el.value`...) |
| --- | --- | --- |
| Dónde vive | En el marcado inicial (y su reflejo en el objeto) | En el objeto elemento |
| API | `getAttribute/setAttribute/removeAttribute/hasAttribute` | Asignación directa: `el.src = ...` |
| Tipo | **Siempre string** | El tipo real (boolean, number, function) |
| Estado actual | El valor del HTML | El valor **vivo** (lo que el usuario cambió) |

```js
const check = document.querySelector('#oferta');

check.setAttribute('checked', '');      // marca el atributo HTML
console.log(check.getAttribute('checked')); // ""
console.log(check.checked);             // true — la PROPIEDAD refleja el estado
check.checked = false;                  // cambia el estado real
console.log(check.hasAttribute('checked')); // true aún así: el atributo sigue

// recorrer todos los atributos del elemento
const img = document.querySelector('img');
for (const { name, value } of img.attributes) console.log(name, '=', value);
```

> [!info] El caso `checked`/`value` explica la regla
> El **atributo** `checked` es el estado inicial del HTML; la **propiedad** `checked` es el estado actual (muta con cada click del usuario). Para leer/ escribir estado de formularios: **siempre la propiedad** (`input.value`, `input.checked`). Ver [[Formularios]].

## Crear, clonar e insertar elementos

```js
// Crear desde cero
const enlace = document.createElement('a');
enlace.textContent = 'Nuevo enlace';
enlace.href = '#';

const navegacion = document.querySelector('.navegacion');
navegacion.appendChild(enlace);                     // al final (API clásica)
navegacion.insertBefore(enlace, navegacion.children[1]); // antes de un hijo concreto
```

La API moderna (2018+) unifica inserción con *strings* y nodos:

| Método | Inserta | Notas |
| ------ | ------- | ----- |
| `padre.append(nodo, 'texto', ...)` | Al final | Acepta nodos **y** strings, varios de golpe |
| `padre.prepend(nodo)` | Al inicio | Igual versatilidad |
| `hermano.before(nodo)` | Antes del hermano | No requiere el padre |
| `hermano.after(nodo)` | Después del hermano | |
| `nodo.replaceWith(otro)` | Reemplaza al nodo | |
| `padre.replaceChildren(...nodos)` | Limpia y reemplaza todos los hijos | Útil para "render" total |
| `nodo.remove()` | Se elimina a sí mismo | Más limpio que `removeChild` |
| `nodo.cloneNode(true)` | Copia (deep `true`, shallow `false`) | No copia listeners ni propiedades JS |

```js
// insertAdjacentHTML: texto HTML en 4 posiciones respecto al elemento
// ⚠️ MISMA advertencia que innerHTML: nunca con datos de usuario
elemento.insertAdjacentHTML('beforeend', '<li>Nuevo ítem</li>');
// posiciones: 'beforebegin' | 'afterbegin' | 'beforeend' | 'afterend'
```

| Posición `insertAdjacentHTML` | Queda... |
| ----------------------------- | -------- |
| `beforebegin` | hermano antes del elemento (solo si tiene padre) |
| `afterbegin` | como primer hijo |
| `beforeend` | como último hijo |
| `afterend` | hermano después del elemento |

> [!warning] Insertar texto HTML = XSS potencial
> `innerHTML` e `insertAdjacentHTML` interpretan etiquetas. Si el string incluye datos de usuario, escapá primero (`el.textContent` como setter escapa solo) o construí con `createElement`.

Ejemplo práctico: construir una "card" completa y agregarla al DOM:

```js
const parrafo1 = document.createElement('p');
parrafo1.textContent = 'Concierto';
parrafo1.classList.add('categoria', 'concierto');

const parrafo2 = document.createElement('p');
parrafo2.textContent = 'Concierto de ROCK';
parrafo2.classList.add('titulo');

const parrafo3 = document.createElement('p');
parrafo3.textContent = '$400';
parrafo3.classList.add('precio');

const info = document.createElement('div');
info.classList.add('info');
info.append(parrafo1, parrafo2, parrafo3); // append acepta varios hijos

const imagen = document.createElement('img');
imagen.src = 'img/hacer2.jpg';

const card = document.createElement('div');
card.classList.add('card');
card.append(imagen, info);

document.querySelector('.hacer .contenedor-cards').append(card);
```

Para muchos nodos de una vez, llena un `DocumentFragment` y insertalo completo — una sola invalidación de layout (ver [[Document Object Model (DOM)]]).

## Recorrer el árbol (traversing)

```js
const navegacion = document.querySelector('.navegacion');

console.log(navegacion.childNodes); // TODOS los nodos: incluye los saltos de línea como #text
console.log(navegacion.children);   // solo elementos HTML (los enlaces)
console.log(navegacion.children[0]);           // se accede como arreglo
console.log(navegacion.children[0].nodeName);  // nombre del nodo, p. ej. "A"
console.log(navegacion.children[0].nodeType);  // 1 = ELEMENT_NODE
```

De hijo a padre y entre hermanos:

```js
const card = document.querySelector('.card');

// Hacia arriba
console.log(card.parentNode);      // cualquier nodo padre
console.log(card.parentElement);   // solo elementos HTML (forma recomendada)
console.log(card.parentElement.parentElement); // se puede encadenar

// Entre hermanos
console.log(card.nextElementSibling);
console.log(card.previousElementSibling);

// Primer/último hijo de un contenedor
console.log(navegacion.firstElementChild);
console.log(navegacion.lastElementChild);

// Descender por niveles: "traversing"
console.log(card.children[1].children[1].textContent);
card.children[1].children[1].textContent = 'Hola muchachos';
```

`children` es una colección **viva** sin `map`/`filter`: `[...card.children].filter(...)` cuando necesitás métodos de [[Arreglos]].

## Acceso a formularios

```js
const form = document.querySelector('#registro');

// elementos por nombre (colección viva)
form.elements.email.value;
form.elements['nombre-completo'].value; // ids/ nombres con guiones requieren corchetes

// propiedades directas del form
console.log(form.method, form.action, form.elements.length);

// todos los campos de golpe (incluye campos sin name si se pasa true)
console.log([...form.elements].map((el) => el.name));

// integración moderna: FormData
const datos = new FormData(form);          // name → value de todo el form
console.log(datos.get('email'));
console.log(Object.fromEntries(new FormData(form))); // → objeto plano
```

Los detalles de validación y envío están en [[Formularios]] y [[Eventos]].

## Ejemplo práctico: botón que despliega el footer

Este ejemplo combina selección, clases y un evento (ver [[Eventos]] para el detalle):

```js
const btnFlotante = document.querySelector('.btn-flotante');
const footer = document.querySelector('footer');

btnFlotante.addEventListener('click', mostrarFooter);

function mostrarFooter() {
  if (footer.classList.contains('activo')) {
    footer.classList.remove('activo');
    this.classList.remove('activo'); // `this` es el botón que disparó el evento
    this.textContent = 'Idioma y Moneda';
  } else {
    footer.classList.add('activo');
    this.classList.add('activo');
    this.textContent = 'X Cerrar';
  }
}
```

> [!tip] `this` solo en funciones normales
> `this` funciona aquí porque la función fue declarada con `function`. En una arrow function no hay `this` propio: usa `evt.currentTarget` en su lugar.

## Errores comunes

1. **`querySelector` devuelve `null`**: el script corrió antes de que el elemento existiera (script en `<head>` sin `defer`), o el selector no coincide (typo, clase que solo aplica en desktop...). Síntoma: `TypeError: Cannot read properties of null`. Fix: `defer`/DOMContentLoad y `?.` como red de seguridad — ver [[Java Script]].
2. **Colección viva que "crece" durante la iteración**: `document.getElementsByClassName('x')` y `document.links` se actualizan solas; si mientras recorres agregás/quitas nodos con esa clase, cambian los índices y te saltás elementos. `[...coleccion]` antes de mutar — ver [[Iteradores]].
3. **Esperar `map`/`filter` en `NodeList`/`HTMLCollection`**: no existen (salvo `forEach` en NodeList). `[...lista].map(...)` o `Array.from(lista)`.
4. **Snapshot obsoleto de `querySelectorAll`**: la lista no refleja nodos creados después. Re-ejecuta la selección tras mutar el DOM, o delega con listeners en ancestros.
5. **XSS vía `innerHTML`/`insertAdjacentHTML`** con datos externos: el ataque clásico. `textContent` para texto; sanitizador dedicado si necesitás HTML de usuario.
6. **Modificar atributo `checked`/`value` esperando cambio de estado**: los atributos son estado inicial; las **propiedades** (`el.checked`, `el.value`) son el estado vivo.
7. **`innerText` en loops**: cada lectura fuerza reflow. `textContent` es órdenes de magnitud más barata y suele dar el mismo resultado.
8. **`className` como setter**: borra todas las clases menos la nueva. `classList.add` en su lugar.

## Cheatsheet — acceso y manipulación

| Tarea | API | Nota clave |
| ----- | --- | ---------- |
| 1 elemento | `querySelector` / `getElementById` | `null` si no hay — verificar |
| N elementos | `querySelectorAll` (snapshot) / `getElementsBy*` (vivo) | vivas vs snapshot |
| Dentro de un subárbol | `elemento.querySelector(sel)` | más rápido, acotado |
| ¿Cumple el selector? | `el.matches(sel)` / `el.closest(sel)` | propio / ancestros |
| Texto | `textContent` (rápido) / `innerText` (visible, caro) | setter escapa HTML |
| HTML | `innerHTML` | setter = parseo + riesgo XSS |
| Datos | `el.dataset.clave` | siempre strings |
| Clases | `classList.add/remove/toggle/contains` | `className` sobrescribe |
| Atributos | `getAttribute/setAttribute/hasAttribute` | strings; estado inicial |
| Estado vivo | `el.value`, `el.checked`, `el.disabled` | propiedades, no atributos |
| Crear | `document.createElement` | seguro, sin parseo |
| Insertar | `append/prepend/before/after/replaceWith` | nodos o strings |
| Insertar HTML | `insertAdjacentHTML` | XSS: solo HTML propio |
| Clonar | `cloneNode(true)` | no copia listeners |
| Eliminar | `el.remove()` | limpia también listeners si no se reusan |
| Formularios | `form.elements`, `new FormData(form)` | ver [[Formularios]] |

## Relacionadas

- [[Document Object Model (DOM)]] — qué es el árbol del DOM, tipos de nodo y rendimiento
- [[Eventos]] — cómo reaccionar a interacciones del usuario (incluye delegación con `closest`)
- [[Java Script]] — cómo cargar el script para que el DOM exista al ejecutarse
- [[Html]] — la estructura que el DOM representa
- [[Etiquetas]] — los elementos que se seleccionan y modifican
- [[Formularios]] — validación y acceso a campos
- [[Iteradores]] — recorrer colecciones del DOM (vivas vs snapshot)
- [[Arreglos]] — `[...querySelectorAll]` para usar métodos de arreglo
