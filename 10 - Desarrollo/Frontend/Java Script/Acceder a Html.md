---
aliases: ["Seleccionar elementos del DOM", "Manipular el DOM"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Document Object Model (DOM)]]"
  - "[[Eventos]]"
  - "[[Html]]"
fuente: "https://developer.mozilla.org/es/docs/Web/API/Document/querySelector"
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

> [!warning] `document.all` es legacy
> Existe por compatibilidad, pero `typeof document.all` devuelve `"undefined"` (una peculiaridad histórica). Usa `querySelectorAll` en su lugar.

## Seleccionar elementos

Hay dos generaciones de métodos. Los antiguos (`getElementById`, `getElementsBy*`) siguen siendo válidos, pero hoy se prefiere `querySelector`/`querySelectorAll`, que aceptan **cualquier selector CSS**.

### Forma antigua

```js
// Por clase: retorna una HTMLCollection (viva, se actualiza sola)
const header = document.getElementsByClassName('header');
const hero = document.getElementsByClassName('hero');
const contenedores = document.getElementsByClassName('contenedor');

// Si la clase no existe: colección vacía (no null)
const noExiste = document.getElementsByClassName('no-existe');

// Por id: retorna UN elemento (o null); si hay varios ids iguales, el primero
const formulario = document.getElementById('formulario');
const formularioNoExiste = document.getElementById('no-exite'); // null
```

### Forma actual: `querySelector` y `querySelectorAll`

```js
// El primero que coincida con el selector CSS (o null)
const card = document.querySelector('.card');
const info = document.querySelector('.premium .info');
const navegacion = document.querySelector('nav');

// Selectores complejos de CSS funcionan igual
const segundoCard = document.querySelector('section.hospedaje .card:nth-child(2)');
const formulario = document.querySelector('.contenido-hero #formulario');

// Todas las coincidencias: retorna un NodeList
const cards = document.querySelectorAll('.card');
```

| Método | Selector | Retorna | ¿Colección viva? |
| ------ | -------- | ------- | ---------------- |
| `getElementById` | solo id | 1 elemento o `null` | — |
| `getElementsByClassName` | clase | `HTMLCollection` | Sí |
| `querySelector` | CSS completo | 1 elemento o `null` | — |
| `querySelectorAll` | CSS completo | `NodeList` | No (copia fija) |

> [!tip] Una diferencia clave
> `HTMLCollection` es **viva**: si agregas un elemento con esa clase, la colección crece sola. `NodeList` de `querySelectorAll` es **estática**: no refleja cambios posteriores. Por eso `cards[0]` puede ser `undefined` si la lista quedó vacía: siempre verifica antes de usar.

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

| Propiedad | Incluye etiquetas HTML | Respeta estilos/visibilidad |
| --------- | ---------------------- | --------------------------- |
| `innerText` | No | Sí (solo lo visible) |
| `textContent` | No | No (todo el texto) |
| `innerHTML` | **Sí** | No |

> [!warning] Cuidado con `innerHTML` con datos de usuario
> Asignar HTML desde entradas no sanitizadas (formularios, APIs) es la puerta clásica a ataques **XSS**. Para texto plano usa `textContent`; solo usa `innerHTML` con HTML que tú mismo construiste.

## Modificar atributos, estilos y clases

```js
// Atributos directamente como propiedades
const imagen = document.querySelector('.card img');
imagen.src = 'img/hacer2.jpg';
imagen.alt = 'Nuevo concierto';

// Estilos en línea: camelCase en vez de guion (background-color -> backgroundColor)
const encabezado = document.querySelector('h1');
encabezado.style.backgroundColor = 'red';
encabezado.style.fontFamily = 'Arial';
encabezado.style.textTransform = 'uppercase';
```

Escribir todo el CSS desde JavaScript deja el código gigante e inmantenible. La práctica recomendada es **agregar y quitar clases** y dejar el estilo en el CSS:

```js
const card = document.querySelector('.card');

card.classList.add('nueva-clase', 'segunda-clase'); // agrega una o varias
card.classList.remove('nueva-clase');               // elimina una o varias
card.classList.toggle('activa');                    // agrega si falta, quita si existe
card.classList.contains('activa');                  // verifica (true/false)
```

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

## Crear y eliminar elementos

```js
// Eliminar
const primerEnlace = document.querySelector('a');
primerEnlace.remove();                              // el propio elemento se borra
navegacion.removeChild(navegacion.children[2]);     // desde el padre

// Crear desde cero
const enlace = document.createElement('a');
enlace.textContent = 'Nuevo enlace';
enlace.href = '#';

const navegacion = document.querySelector('.navegacion');
navegacion.appendChild(enlace);                     // al final
navegacion.insertBefore(enlace, navegacion.children[1]); // antes de un hijo concreto
```

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

document.querySelector('.hacer .contenedor-cards').appendChild(card);
```

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

## Relacionadas

- [[Document Object Model (DOM)]] — qué es el árbol del DOM
- [[Eventos]] — cómo reaccionar a interacciones del usuario
- [[Java Script]] — cómo cargar el script para que el DOM exista al ejecutarse
- [[Html]] — la estructura que el DOM representa
