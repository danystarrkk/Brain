---
aliases: ["Event listeners", "Eventos del DOM"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Document Object Model (DOM)]]"
  - "[[Acceder a Html]]"
  - "[[Funciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/API/Event_target/addEventListener"
---

# Eventos

Un **evento** es cualquier notificación que el navegador lanza cuando algo pasa: un click, una tecla, un envío de formulario, el scroll... **Escuchar eventos** con `addEventListener` es lo que hace a una página *interactiva*: el código corre **reaccionando** al usuario, no en línea recta.

```js
console.log(1);

document.addEventListener('DOMContentLoaded', () => {
  console.log(2); // se ejecuta cuando el HTML terminó de cargarse
});

console.log(3);
// Orden en consola: 1, 3, 2 — el código normal no espera a los eventos
```

> [!info] `DOMContentLoaded`
> Es el evento que indica que el HTML ya está construido (el [[Document Object Model (DOM)|DOM]] existe). Si tu script carga en el `<head>`, escuchar este evento es la garantía de que `querySelector` no retornará `null`.

## Eventos de mouse

```js
const nav = document.querySelector('.navegacion');

nav.addEventListener('click', () => console.log('click en nav'));

nav.addEventListener('mouseenter', () => {
  console.log('entrando en la navegación');
  nav.style.backgroundColor = 'transparent';
});

nav.addEventListener('mouseout', () => {
  console.log('saliendo de la navegación');
  nav.style.backgroundColor = 'white';
});

nav.addEventListener('mousedown', () => console.log('Has presionado'));  // botón abajo
nav.addEventListener('mouseup', () => console.log('Has soltado el click')); // botón arriba
nav.addEventListener('dblclick', () => console.log('Doble click'));
```

| Evento | Cuándo dispara |
| ------ | -------------- |
| `click` | presionar **y** soltar sobre el elemento |
| `dblclick` | doble click |
| `mousedown` / `mouseup` | solo la parte de presionar / soltar |
| `mouseenter` / `mouseleave` | el cursor entra o sale del elemento (sin burbujear) |
| `mouseover` / `mouseout` | igual, pero **sí** burbujea y se dispara al cruzar hijos |

## Eventos de teclado y formulario

```js
const busqueda = document.querySelector('.busqueda');

busqueda.addEventListener('keydown', () => console.log('Tecla presionada')); // al pulsar
busqueda.addEventListener('keyup', () => console.log('Tecla soltada'));       // al soltar
busqueda.addEventListener('blur', () => console.log('Salió del campo'));      // muy usado en validaciones

busqueda.addEventListener('copy', () => console.log('Copiar'));
busqueda.addEventListener('paste', () => console.log('Pegar'));
busqueda.addEventListener('cut', () => console.log('Cortar'));
```

### El evento `input`: el más útil

El evento `input` se dispara **cada vez que el valor cambia** (teclear o borrar), así que cubre `keydown` y `keyup` a la vez:

```js
busqueda.addEventListener('input', (evt) => {
  console.log(evt.type);        // "input": el tipo de evento
  console.log(evt.target);       // el ELEMENTO que disparó el evento
  console.log(evt.target.value); // lo que el usuario lleva escrito

  if (evt.target.value === '') {
    console.log('El campo quedó vacío');
  }
});
```

`evt.target` es la clave para saber **qué elemento originó** el evento — ver "Delegación" más abajo.

### Envío de formularios

```js
const formulario = document.querySelector('#formulario');

formulario.addEventListener('submit', validarFormulario);

function validarFormulario(evt) {
  evt.preventDefault(); // detiene el envío tradicional (recarga de página)
  console.log(evt.target.method); // GET o POST
  console.log(evt.target.action); // a dónde se enviaría

  // en este punto puedes validar los campos o consumir una API
}
```

> [!warning] `preventDefault()` es casi obligatorio en `submit`
> Sin él, el navegador recarga la página y pierdes todo el trabajo de validación. Solo omítelo si de verdad quieres el envío nativo del formulario.

## Scroll

```js
window.addEventListener('scroll', () => {
  const scrollPX = window.scrollY; // píxeles scrolleados en vertical

  const premium = document.querySelector('.premium');
  const ubicacion = premium.getBoundingClientRect();

  if (ubicacion.top < 784) {
    console.log('El elemento ya está visible');
  } else {
    console.log('Aún no, da más scroll');
  }
});
```

`getBoundingClientRect().top` es la distancia entre el borde superior del elemento y el borde superior de la ventana: cuando se vuelve negativa, el elemento entró en pantalla.

> [!tip] Rendimiento en `scroll`
> El evento `scroll` dispara a alta frecuencia. En proyectos grandes, limita su ejecución (*throttle*) o usa `IntersectionObserver`, que avisa cuando un elemento entra en pantalla sin recalcular en cada frame.

## Event Bubbling (propagación)

Cuando haces click en un hijo, el evento **sube** por el árbol: primero ejecuta el listener del hijo, luego del padre, luego del abuelo... eso es el *bubbling*:

```js
const cardDiv = document.querySelector('.card');
const infoDiv = document.querySelector('.info');
const titulo = document.querySelector('.titulo');

cardDiv.addEventListener('click', () => console.log('click en card'));
infoDiv.addEventListener('click', (evt) => {
  evt.stopPropagation(); // detiene la propagación hacia arriba
  console.log('click en info');
});
titulo.addEventListener('click', (evt) => {
  evt.stopPropagation();
  console.log('click en titulo');
});
```

> [!warning] `stopPropagation` con cuidado
> Detener la propagación en muchos niveles puede "ahogar" listeners legítimos del padre. Prefiere diseñar el flujo y usar `stopPropagation` solo donde hace falta.

## Delegación de eventos

En vez de poner un listener en cada hijo, se pone **uno solo en el padre** y se identifica el origen con `evt.target`. Es más eficiente y, sobre todo, **funciona con elementos que se crean dinámicamente**:

```js
const cardDiv = document.querySelector('.card');

cardDiv.addEventListener('click', (evt) => {
  console.log(evt.target);
  console.log(evt.target.classList);

  if (evt.target.classList.contains('titulo')) console.log('click en título');
  if (evt.target.classList.contains('precio')) console.log('click en precio');
  if (evt.target.classList.contains('categoria')) console.log('click en categoría');
});
```

## Asignando handlers a elementos creados

```js
const parrafo1 = document.createElement('p');
parrafo1.textContent = 'Concierto';
parrafo1.classList.add('categoria', 'concierto');

const parrafo2 = document.createElement('p');
parrafo2.textContent = 'Concierto de Rock';
parrafo2.classList.add('titulo');

const parrafo3 = document.createElement('p');
parrafo3.textContent = '$800 por persona';
parrafo3.classList.add('precio');

// Asignar la función SIN paréntesis: si escribes nuevaFuncion(), se ejecuta
// en el momento de asignar y el listener nunca se registra.
parrafo3.onclick = nuevaFuncion;

// Para pasar argumentos, se envuelve en una función anónima/arrow
parrafo3.onclick = () => nuevaFuncion(1);

function nuevaFuncion(id) {
  console.log(`nueva función con el id ${id}`);
}

// Componer el card y agregarlo al DOM
const info = document.createElement('div');
info.classList.add('info');
info.append(parrafo1, parrafo2, parrafo3);

const imagen = document.createElement('img');
imagen.src = 'img/hacer2.jpg';

const contenedorCard = document.createElement('div');
contenedorCard.classList.add('contenedorCard');
contenedorCard.append(imagen, info);

document.querySelector('.hacer .contenedor-cards').appendChild(contenedorCard);
```

> [!tip] `onclick` vs `addEventListener`
> `elemento.onclick = fn` solo admite **un** handler por evento (la última asignación pisa a la anterior). `addEventListener` admite varios y permite opciones (`once`, `capture`). Prefiere `addEventListener`; el `onclick` se ve sobre todo en HTML inline.

Ver [[Acceder a Html]] para `createElement`/`append` y el resto de manipulación del DOM.

## Relacionadas

- [[Document Object Model (DOM)]] — el árbol por el que viajan los eventos
- [[Acceder a Html]] — crear los elementos a los que se les agregan listeners
- [[Funciones]] — callbacks: la función que recibe `addEventListener`
- [[Qué es una API]] — qué ocurre cuando el evento dispara un `fetch`
