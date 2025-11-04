---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Eventos en HTML

```jsx
console.log(1);

document.addEventListener('DOMContentLoaded', () => { // este es un evente el cual se ejecuta una vez el html es descargado y cargado
  console.log(2);
});

console.log(3);

const nav = document.querySelector('.navegacion');

// Eventos de Mouse

nav.addEventListener('click', () => { // realiza la accion al detectar el click
  console.log("click en nav");
})

nav.addEventListener('mouseenter', () => { // nos permite detectar cuando el mouse se encuentra sobre el elemento
  console.log('entrando en la navegación');
  nav.style.backgroundColor = 'transparent';
})

nav.addEventListener('mouseout', () => { // Nos permite detectar cuando el mouse sale del espacio o elemento que definimos
  console.log('Saliendo de la navegacion');
  nav.style.backgroundColor = 'white';
})

nav.addEventListener('mousedown', () => { // es similar a click pero detecta cuando la oprecion del click y click detecta el pulsar y soltar
  console.log("Has precionando");
})

nav.addEventListener('mouseup', () => { // es lo contrario a mousedown detecta el soltar del click
  console.log("Has solatdo el click");
})

nav.addEventListener('dblclick', () => { // Nos permite detectar cuando realizamos un doble click
  console.log("Has realizado un doble click");
})

const busqueda = document.querySelector('.busqueda');

busqueda.addEventListener('keydown', () => { // Detecta el pulsar de una tecla, el PULSAR no el pulsar y soltar
  console.log("Escribiendo...");
})

busqueda.addEventListener('keyup', () => { // Detecta el soltar de una tecla
  console.log("Soltar la tecla");
})

busqueda.addEventListener('blur', () => { // Nos permite detectar cuando el usuario sale de un campo, (muy usado en validaciones)
  console.log('validación');
})

busqueda.addEventListener('copy', () => { // Detecta cuando copiamos algun texto
  console.log('Copiar');
})

busqueda.addEventListener('paste', () => { // Detecta al pegar algo
  console.log('Copiar');
})

busqueda.addEventListener('cut', () => { // Detecta al momento de coortar texto
  console.log('Copiar');
})

// El principal y mas usado es "input", esto ya que realiza todo lo anterior con excepción del blur

busqueda.addEventListener('input', (evt) => { // podemos pasarle a al función el evento (evt)
  console.log(evt);
})

busqueda.addEventListener('input', (evt) => { // Nos permite ver sobre que elemento se esta trabajando
  console.log(evt.type);
})

busqueda.addEventListener('input', (evt) => { // Esto es ideal cuando realizamos busquedas con el objetivo de leer lo que el usuario escribe
  console.log(evt.target.value);
})

busqueda.addEventListener('input', (evt) => { // Detectamos cuando borramos por completo un campo
  if (evt.target.value === "") {
    console.log("fallo la validadción");
    
const formulario = document.querySelector('#formulario');

formulario.addEventListener('submit', validarFormulario);

function validarFormulario(evt) {
  evt.preventDefault(); // esto nos ayuda a prevenir la accion de un elemento por default
  console.log(evt.target.method); // .method nos ayuda a ver que metodo usa para enviar datos .action para ver la acción del submit
  // en este punto podemos comenzar a consumir una api
}

window.addEventListener('scroll', () => { // esta es una forma de detectar el scroll vertical
  const scrollPX = window.scrollY; // Esto nos permite registrar el scroll en vertical
  // console.log(scrollPX);
  const premium = document.querySelector('.premium');
  const ubicacion = premium.getBoundingClientRect();

  // console.log(ubicacion);

  if (ubicacion.top < 784) {
    console.log("El elemento Ya esta visible");
  } else {
    console.log("Aún no, da mas scroll");
  }
})

// Event Bubbling
// Es cuando precionamos en un evento pero este mismo evento se propaga por multiples partes dando otros resultados

const cardDiv = document.querySelector(".card");
const infoDiv = document.querySelector(".info");
const titulo = document.querySelector(".titulo");

cardDiv.addEventListener('click', (evt) => {
  evt.stopPropagation(); // Esta es una forma de deterner el Event Bubbling
  console.log("click en card");
})

infoDiv.addEventListener('click', (evt) => {
  evt.stopPropagation(); // Esta es una forma de deterner el Event Bubbling
  console.log("click en info");
})

titulo.addEventListener('click', (evt) => {
  evt.stopPropagation(); // Esta es una forma de deterner el Event Bubbling
  console.log("click en titulo");
})

// Delegation

const cardDiv = document.querySelector(".card");
cardDiv.addEventListener('click', (evt) => {
  console.log(evt.target); // Esta es una forma de identificar a donde le damos click
  console.log(evt.target.classList); // Nos permite identificar las clases de cada elemnto

  // Podmeos usarlo  de la siguiente manera para poder identificar elementos

  if (evt.target.classList.contains('titulo')) {
    console.log('click a titulo');
  }
  if (evt.target.classList.contains('precio')) {
    console.log('click a precio');
  }

  if (evt.target.classList.contains('categoria')) {
    console.log('click a categoria');
  }

})

// Evitar la propagación con contenido creado...
const parrafo1 = document.createElement('P');
parrafo1.textContent = 'Concierto';
parrafo1.classList.add('categoria');
parrafo1.classList.add('concierto');

// Segundo parrafo
const parrafo2 = document.createElement('P');
parrafo2.textContent = 'Concierto de Rock';
parrafo2.classList.add('titulo');

// 3er parrafo...
const parrafo3 = document.createElement('p');
parrafo3.textContent = '$800 por persona';
parrafo3.classList.add('precio');
parrafo3.onclick = nuevaFuncion; // cuando agregamos nuevas funciones no se debe poner los parentesis sino esta se ejecutara siempre,
// si deseamos pasar un valor a una función vamos a crear una funcion anonima que se encargue de eso
parrafo3.onclick = function () {
  nuevaFuncion(1);
}

// crear el div...
const info = document.createElement('div');
info.classList.add('info');
info.appendChild(parrafo1)
info.appendChild(parrafo2)
info.appendChild(parrafo3);

// Vamos a crear la imagen
const imagen = document.createElement('img');
imagen.src = 'img/hacer2.jpg';

// Crear el Card..
const contenedorCard = document.createElement('div');
contenedorCard.classList.add('contenedorCard');

// Vamos a asignar la imagen al card...
contenedorCard.appendChild(imagen);

// y el info
contenedorCard.appendChild(info);

// Insertarlo en el HTML...
const contenedor = document.querySelector('.hacer .contenedor-cards');
contenedor.appendChild(contenedorCard); // al inicio info

function nuevaFuncion(id) {
  console.log(`nueva funcion con el id de ${id}`);
}
```