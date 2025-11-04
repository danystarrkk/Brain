---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Acceder a elementos del HTML con document

```jsx
let elemento;

elemento = document; // nos permite seleccionar todo el documento html
elemento = document.all; // Nos permite seleccionar todos los elementos que componen el html
elemento = document.head; // Nos permite seleccionar la parte del heade del HTML
elemento = document.body; // Nos permite seleccionar todos los elementos de un body
elemento = document.domain; // Nos permite seleccionar el dominio actual
elemento = document.forms; // Nos permite ver cuantos formularios tenemos
elemento = document.forms[0]; // Al listar en una especie de arreglo el HTMLCollection podemos acceder como tal
elemento = document.forms[0].id; // Acedemos al id del formulario
elemento = document.forms[0].method; // Nos permite acceder al metodo GET/POST
elemento = document.forms[0].classList; // Listar las clases
elemento = document.forms[0].action;

elemento = document.links; // Podemos listar todos los enlaces
elemento = document.links[4]; // Accedemos al elmento especifico
elemento = document.links[4].classList; // clases para ese elemento en forma de arreglo
elemento = document.links[4].className; // Nombre de la clase en forma de string

elemento = document.images; // selecciona todas las imagenes del documento
elemento = document.scripts; // Lista los scripts de todo el documento

console.log(elemento);

// Seleccionar elemenots por su clase (Forma mas Antigua)

const header = document.getElementsByClassName("header"); // Nos permite seleccionar una clase
console.log(header);

const hero = document.getElementsByClassName("hero");
console.log(hero);

// Si las calses exiten mas de 1 vez
const contenedores = document.getElementsByClassName("contenedor");
console.log(contenedores);

// Si una clase no existe, retorna como vacío
const noExiste = document.getElementsByClassName("no-existe");
console.log(noExiste);

// Selecicionando mediante id, este solo retorna maximo un elemento por lo que si se repite retorna el primero
const formulario = document.getElementById("formulario");
console.log(formulario);

// Si un id no existe, nos retorna vasio
const formularioNoExiste = document.getElementById("no-exite");
console.log(formularioNoExiste);

// QuerySelector (Forma Actual)

const card = document.querySelector(".card"); // Solo retorna el primero que encuentre por lo que retorna como máximo un elemento
console.log(card);

// Podemos tener selectores especificos como en css
const info = document.querySelector(".premium .info");
console.log(info);

// Seleccionar el segundo card de hospedaje
const segundoCard = document.querySelector(
  "section.hospedaje .card:nth-child(2)"
); // con ayuda de la sintaxis de css podemos seleccionar el segundo card
console.log(segundoCard);

// Tambien podemos seleccionar ID
const formulario = document.querySelector(".contenido-hero #formulario"); // Seleccionamos el id que especificamos
console.log(formulario);

// Seleccionar Elementos HTML
const navegacion = document.querySelector("nav"); // podemos seleccionar mediante las etiquetas los elementos html
console.log(navegacion);

// QuerySelectorall para selecionar multiples Elementos (Forma Actual)
// La forma de seleccionar es igual a QuerySelctorall con la diferencia que retorna todas las coincidencias

const card = document.querySelectorAll("card"); // A los elementos de un HTML los conocemos como NODOS por eso retorna un NODELiST
console.log(card);

// En el caso de los ID que no deveria pero si tenemos 2 iguales vamos a ver que selecciona todos de igual forma
// todo lo demos como la no existencia de una clase o id va a retornar un NODELIST pero vacio.

const encabezado = document.querySelector(".contenido-hero h1");

// Ver el contenido del elemento selecionado
console.log(encabezado.innerText); // Forma1 (trae solo el texto), en caso de ser texto oculto no lo muestra (css - visibility: hidden;)
console.log(encabezado.textContent); // Forma2 (trae solo el texto), muestra el texto sin importar si esta oculto
console.log(encabezado.innerHTML); // Forma3 (Trae netamente el html)
console.log(encabezado);
// Podemos seleccionar directamente el contenido con chaiding (encadenamiento opcional)
const encabezado2 = document.querySelector(".contenido-hero h1").textContent;
console.log(encabezado2);

// Editar contenido sobre la información del elmento selecicionado
document.querySelector(".contenido-hero h1").textContent = "nuevoHeading";

// Seleccionar y modificar una imagen

const imagen = document.querySelector(".card img");
console.log(imagen);

// Modificando la imagen

imagen.src = "img/hacer2.jpg";

// Modificando o cambiando el css Con javaScript

const encabezado = document.querySelector("h1");
console.log(encabezado);

// La sintaxis seria por ejemplo justify-content -> justifyContent

// Modificando el CSS
encabezado.style.backgroundColor = "red"; // Modificamos el background el h1
encabezado.style.fontFamily = "Arial";
encabezado.style.textTransform = "uppercase";

// Esta forma de escribir JavaScript Puede llevar a dejar le JS muy grande y para evitar escribir el código de esta forma podemos agregar o quitar clases

const card = document.querySelector(".card");
console.log(card.classList);
// Agregando una clase a en card
card.classList.add("nueva-clase", "segunda-clase"); // podemos agregar desde 1 a n clases
card.classList.remove("nueva-clase"); // Podemos eliminar uno o mas de una clase

const navegacion = document.querySelector(".navegacion");
console.log(navegacion);
// Cuanod expandimos la navegación vamos a encontrarnos enlaces, a estos enlaces los conoceremos como nodos en JavaScript
console.log(navegacion.childNodes); // Nos permite retornar los espacios en blanco como elementos representados por text y los elementos por a

// Tenemos una alternativa para listar solo elementos reales osea solo los enlaces
console.log(navegacion.children); // Solo lista los elementos, en este caso los enlaces
// con children nosotros se nos permite seleccionar los elementos como si fueran un array
console.log(navegacion.children[0]);
console.log(navegacion.children[0].nodeName); // Vamos a tener dos separaciones los que es le NodeName y el NodeType
console.log(navegacion.children[0].nodeType); // Vamos a tener dos separaciones los que es le NodeName y el NodeType

// Los tipos de Nodos podemos encontrarlos en la documentacion y de forma especifica el 1 es de ELEMENT_NODE (Etiqueta HTML)

const card = document.querySelector(".card");
console.log(card.children[1].children[1].textContent); // con ayuda de children realizamos traversing the DOM para entrar en un elemento

card.children[1].children[1].textContent = "Hola Muchahos"; // Modificando mediante el Traversing the DOM

console.log(card.children[0]); // Mediante traversl the DOM identificamos el documento
card.children[0].src = "img/hacer2.jpg"; // Modificamos el documento mediante Traversal the DOM

// Traversing The DOM (Hijo al Padre)

console.log(card.parentNode); // obtenemos el padre del elemento seleccionado
console.log(card.parentElement); //Verifica por elementos validos por HTML, forma recomandada
console.log(card.parentElement.parentElement.parentElement); // podemos concatenar los parentElement que se requieran para recores hacia el padre del elemento

// Traversing The DOM (Saltar entre los elementos igual o Hermanos)

console.log(card.nextElementSibling); // Nos permite saltar al siguiente elemento desde el elemento seleccionado

const ultimoCard = document.querySelector(".card:nth-child(4)");
console.log(ultimoCard);

console.log(ultimoCard.previousElementSibling); // Nos permite dirigirnos al elemento anterior desde el elemento selecionado

// podemos tambien de una forma mas simplificada ubicar el ultimo o primer elemento de nuestro html
console.log(navegacion.firstElementChild); // nos permite acceder al primer elementos osea al primer enlace de nuestra navegación
console.log(navegacion.lastElementChild); // Nos permite acceder al ultimo elemento, osea al ultimo enlace de nuestra navegación

const primerEnlace = document.querySelector("a");
primerEnlace.remove(); // Nos permite remover elementos del html por si mismo

// Eliminar desde el padre

const navegacion = document.querySelector(".navegacion");
console.log(navegacion.children);

navegacion.removeChild(navegacion.children[2]); // eliminamos de forma referenciada los elmentos de nuestro html

// Generar HTML desde el JavaScript

const enlace = document.createElement("a");

// Agregando el Texto
enlace.textContent = "Nuevo enlace";

// Agregando el Href
enlace.href = "#";

// Seleccionar la navegación
const navegacion = document.querySelector(".navegacion");

// Colocando el enlace al final
navegacion.appendChild(enlace); // esta forma coloca al final del NODELIST por ende al final

// Colocando un enlace en posiciones especificas
console.log(navegacion.children);
navegacion.insertBefore(enlace, navegacion.children[1]); // de esta forma colocamos el elemento antes de

console.log(enlace);

// creando un Card mas en la Web

const parrafo1 = document.createElement("p");
parrafo1.textContent = "Concierto";
parrafo1.classList.add("categoria", "concierto");

const parrafo2 = document.createElement("p");
parrafo2.textContent = "Concierto de ROCK";
parrafo2.classList.add("titulo");

const parrafo3 = document.createElement("p");
parrafo3.textContent = "$400";
parrafo3.classList.add("Precio");

// Crear div con la clase de info

const info = document.createElement("div");
info.classList.add("info");

info.appendChild(parrafo1);
info.appendChild(parrafo2);
info.appendChild(parrafo3);

// Crear la imagen
const imagen = document.createElement("img");
imagen.src = "img/hacer2.jpg";

// Crear el Card
const card = document.createElement("div");
card.classList.add("card");
card.appendChild(imagen);

// Asignar info
card.appendChild(info);

console.log(card);

// Agregando al html
const contenedorCard = document.querySelector(".hacer .contenedor-cards");
contenedorCard.appendChild(card);
const btnFlotante = document.querySelector(".btn-flotante");
const footer = document.querySelector("footer");

// Registro evento
btnFlotante.addEventListener("click", mostrarFooter); // primero se le pasa el posible evento a recivir, y despues se le puede pasar una función o una arrowfuncion

function mostrarFooter() {
  if (footer.classList.contains("activo")) {
    // el .contains nos permite verificar si tenemos la clase en el html que especificamos
    footer.classList.remove("activo");
    this.classList.remove("activo"); // podemos usar this ya que el objeto que se usa para mandar a llamar la función es btnFlotante que es el mismo que
    // queremos modificar por lo tanto si podemos usar this
    this.textContent = "Idioma y Moneda";
  } else {
    footer.classList.add("activo");
    this.classList.add("activo");
    this.textContent = "X Cerrar";
  }
}

// Ejmplo practico del uso del DOM

const btnFlotante = document.querySelector(".btn-flotante");
const footer = document.querySelector("footer");

// Registro evento
btnFlotante.addEventListener("click", mostrarFooter); // primero se le pasa el posible evento a recivir, y despues se le puede pasar una función o una arrowfuncion

function mostrarFooter() {
  if (footer.classList.contains("activo")) {
    // el .contains nos permite verificar si tenemos la clase en el html que especificamos
    footer.classList.remove("activo");
    this.classList.remove("activo"); // podemos usar this ya que el objeto que se usa para mandar a llamar la función es btnFlotante que es el mismo que
    // queremos modificar por lo tanto si podemos usar this
    this.textContent = "Idioma y Moneda";
  } else {
    footer.classList.add("activo");
    this.classList.add("activo");
    this.textContent = "X Cerrar";
  }
}
```